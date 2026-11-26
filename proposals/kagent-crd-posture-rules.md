# Proposal: Posture rules for Kagent CRDs (`api.kagent.dev`)

## Summary

Add seven Rego rules and seven controls (`C-0323`–`C-0329`) to `regolibrary` covering
insecure configuration of Kagent's custom resources, extending the existing
**Agent Runtime Hardening** framework (`frameworks/agent-runtime-hardening.json`)
that gathers them together with the existing Agent Sandbox controls. Kagent is a
CNCF Sandbox project for Kubernetes-native AI agent orchestration; its CRDs carry
the credentials, TLS posture, and tool-trust decisions for every agent in the
cluster, and nothing in `regolibrary` looks at them today.

This is the Kagent-specific continuation of
[`agent-sandbox-crd-scanning.md`](agent-sandbox-crd-scanning.md). That proposal
established the general case — agent-runtime custom resources are posture objects
and belong in the scan — and delivered rules against the upstream Agent Sandbox
API groups (`agents.x-k8s.io`, `extensions.agents.x-k8s.io`). It did not cover any
concrete agent platform's own API group. This proposal does that for `api.kagent.dev`,
against fields verified in the shipped CRD schemas at Kagent v1.0.x (pinned at
`v1.0.1`).

No new sensing, no runtime work, no admission work. Static posture only, on
capability Kubescape already has.

## Motivation

Kagent's resources sit directly on the trust boundary between a language model,
external MCP tool servers, and the Kubernetes API:

- A `ModelConfig` or `ModelProviderConfig` decides whether the provider API key
  comes from a `Secret`, `actorJwt`, or passes through caller tokens
  (`spec.tokenPassthrough`).
- A `RemoteMCPServer` decides whether the agent trusts an upstream tool server's
  certificate at all (`spec.tls.disableVerify`), and whether it authenticates to it.
- A `Harness` or `SandboxTemplate` decides container environment variables (avoiding
  embedded credentials in `spec.env`).
- An `EgressPolicy` decides which external domains/origins an agent's sandboxed
  execution may reach (`spec.destinations[].origin`).
- A `ModelConfig`, `RemoteMCPServer`, or `ModelProviderConfig` decides which
  namespaces may attach it as a tool or model provider (`spec.allowedNamespaces`).

Each of these is a checkable field with an unambiguous insecure value, and none of
them is visible on the `Deployment` the Kagent controller renders. A cluster can
pass every existing workload control while running an agent that disables TLS
verification against an MCP server and forwards caller tokens to an LLM provider.

The value is also cumulative: this is the first control set in `regolibrary`
targeting a specific agent framework's API, and it establishes the pattern (rule
naming, schema-aware matching, framework placement) for the agent platforms that
follow.

## Goals

- Seven rules under `rules/kagent-*`, each with unit test fixtures that pass in
  `testrunner`.
- Seven controls, `C-0323`–`C-0329`, in an allocated contiguous block.
- Extend the existing framework `frameworks/agent-runtime-hardening.json`
  ("Agent Runtime Hardening") with the seven new controls alongside the existing
  Agent Sandbox controls.
- End-to-end verification: a real `kubescape scan` against a kind cluster running
  Kagent v1.0.x, with a deliberately-insecure CR set and a secure CR set.

## Non-goals

### Pod security on Kagent CRs

`Agent`, `Harness`, and `SandboxTemplate` embed pod-shaped configuration or workload
templates (`spec.workload.*`, `image`, `command`, `args`, etc.). Writing controls
against those fields is **explicitly out of scope**, because the Kagent controller
renders them into a `Deployment` or `Pod` that `regolibrary` already scans — `C-0009`
(resource limits), `C-0016` (privilege escalation), `C-0017` (read-only root
filesystem), `C-0048` (hostPath), `C-0237` (image provenance) and their siblings all
fire on the rendered workload.

Duplicating them on the CR produces two findings for one misconfiguration and two
places to keep remediation text correct. The dividing line for this proposal is
therefore: **a rule earns its place only if the misconfiguration is invisible on
the generated Deployment.** All seven rules below meet that test.

This does cost the shift-left signal in file-scan mode, where no `Deployment`
exists to scan. That is a known and accepted gap; if it proves painful in practice
the right fix is a follow-up that scans the CR's workload block in `file` scope
only, not a duplicate control in `cluster` scope.

### Admission control

Rejecting an insecure `Agent` or `ModelConfig` at admission time is the natural
next step and is deliberately deferred. `agent-sandbox-crd-scanning.md` opened
that thread and [`cel-admission-rules.md`](cel-admission-rules.md) is the place it
belongs. The controls proposed here are written so that CEL/VAP variants can be
derived from them later without changing the rule semantics.

### Surfacing findings through `kubescape mcpserver`

Letting a Kagent agent ask Kubescape about its own posture is a Kubescape-side
change and is tracked separately. It would couple this `regolibrary` PR to work in
another repository for no benefit to the rules themselves.

## Correcting the source design

This work originates in a January 2025 internal design document proposing fifteen
Kagent controls. That document is now twenty-two months old and the Kagent API
has moved substantially. The proposal below is a re-derivation against the shipped
CRD schemas, and the drift is recorded here because it is the reason the control
list changed shape.

| The 2025 document assumed | Reality in the shipped v1.0.x CRDs |
|---|---|
| `executeCodeBlocks: true` on `Agent` (its highest-severity control, 9/9) | **The field does not exist anywhere in the CRDs.** Sandboxed execution posture is now expressed via `EgressPolicy.spec.destinations[].origin`, which denies egress by default when unset or empty |
| An `MCPServer` CRD | **No such CRD.** The live resource is `RemoteMCPServer` (an upstream URL); `ToolServer` was removed in v1.0.x |
| `Agent` carries pod fields directly | Pod and container configuration is distributed across `Harness` and `SandboxTemplate` |
| Four pod-security controls | Out of scope per the non-goal above — they duplicate existing workload controls |
| — | Fields the document predates: `ModelConfig.spec.tokenPassthrough`, `spec.actorJwt`, `spec.allowedNamespaces` (a Gateway-API-style object, not a list), and `EgressPolicy` destinations |

Four of the fifteen proposed controls therefore have no field to check or no
resource to match, and four more needed their paths reworked. Seven survive
re-derivation against Kagent v1.0.x, with `kagent-toolserver-local-exec` dropped
following the removal of `ToolServer` and stdio MCP transport.

### A field that looks like a security control and is not

An earlier draft of this proposal added a ninth control requiring
`isolateSessions: true` on `Agent.spec.declarative.tools[]` entries, on the
reasoning that shared tool sessions leak state between users. That control has
been **dropped**, for two independent reasons:

- It targeted the wrong tool type. The field is documented "Only valid when Type is
  Agent" and the CRD's CEL rejects it on `McpServer` tools, so the rule described a
  manifest the API would refuse.
- More fundamentally, the field is not a security boundary. It controls whether
  calls this agent makes to a **sub-agent** reuse one A2A `context_id`, and its
  documented purpose is correctness under parallel fan-out: without it, N parallel
  calls in one turn collapse into a single shared sub-agent session. The sessions
  in question all belong to the same calling agent, so there is no cross-tenant
  boundary to breach.

It is recorded here because "field name contains *isolate*" is exactly the kind of
inference that produces a plausible-looking control with no threat behind it, and
the next person reading the schema for rule candidates will hit the same field.

## Proposal

### Rules

Rules live at `rules/kagent-<aspect>/` and follow the `agent-sandbox-*` family's
conventions exactly: `package armo_builtins`, `import rego.v1`,
`deny contains msga if`, `object.get(resource, [...], <explicit default>)` so an
omitted field cannot error the rule, a local `resource_name()` helper, and
`failedPaths` naming the real field path so the CLI can point at it.

They are grouped in three tiers by implementation order. All seven are in scope for
this work; the tiers describe sequence, not priority cuts.

#### Tier A — credentials and transport

These are mechanical: the insecure value is explicit in the schema and there is no
judgment call about what "bad" means.

**`kagent-model-credential-source` → `C-0323`**
A `ModelConfig` or `ModelProviderConfig` must not hold its provider credential in a
way that bypasses Kubernetes secret management. Two distinct failures:

- **`spec.tokenPassthrough == true`**, on `ModelConfig` (or within
  `ModelProviderConfig.spec.modelConfigTemplate.auth`). This forwards the bearer
  token from an inbound A2A request to the LLM provider as the API key, making
  every caller's token a provider credential and removing the secret-management
  boundary entirely. Unconditional failure. Note that CRD CEL validation already
  rejects `tokenPassthrough` for `Gemini`, `GeminiVertexAI` and `AnthropicVertexAI`.
- **An API-key provider with no `spec.apiKeySecret`.**

The second branch is **provider-aware**, because `spec.provider` spans eleven values
and only some of them authenticate with an API key:

| `spec.provider` | Credential model | Verdict when `apiKeySecret` absent |
|---|---|---|
| `Anthropic`, `OpenAI`, `AzureOpenAI`, `Gemini`, `Mistral`, `SAPAICore` | API key (`SAPAICore` exchanges it at `authUrl`) | **Fail** |
| `Bedrock` | AWS SDK credential chain — sub-object carries only `region` | Pass |
| `Foundry` | Azure workload identity / `DefaultAzureCredential` — sub-object carries `endpoint`, `apiVersion` | Pass |
| `GeminiVertexAI`, `AnthropicVertexAI` | GCP ADC — sub-object carries `projectID`, `location` | Pass |
| `Ollama` | Typically unauthenticated — sub-object carries `host` | Pass |

The ambient-identity providers carry no credential field anywhere in their
sub-objects, which is the schema-level evidence that they are meant to inherit an
identity from the pod. Failing them for a missing `apiKeySecret` would mark the
*more* secure configuration — workload identity over a long-lived static key — as
the insecure one.

`tokenPassthrough`, `apiKeySecret`, and `actorJwt` are mutually exclusive under the
CRD's own CEL validation (at most one may be set). `actorJwt` represents workload
identity token generation and passes.

**SAPAICore handling (#18 §4):** In the Kagent CRD schema, `spec.apiKeySecret` is a
string holding the Secret name, and `spec.apiKeySecretKey` is a string holding the key
name. For standard API key providers (e.g. `OpenAI`), CEL requires both fields to be
set. However, `SAPAICore` uses a Kubernetes Secret with fixed keys (`client_id` and
`client_secret`) exchanged against SAP AI Core's OAuth `authUrl`. The Kagent schema
and CEL explicitly exempt `SAPAICore` from requiring `apiKeySecretKey`. Therefore,
for `SAPAICore`, the rule requires only `spec.apiKeySecret`; absence of
`spec.apiKeySecretKey` must **pass**.

An unauthenticated `Ollama` endpoint is a real posture question, but it is a
property of the endpoint rather than of this resource's credential handling, and
`C-0326` already covers the plaintext-transport half of it. Left out deliberately
rather than folded in here.

**`kagent-tls-verification-disabled` → `C-0324`**
Fails on `spec.tls.disableVerify == true` on `ModelConfig` or `RemoteMCPServer`.
The CRD's own field documentation says this is for development only and that
production deployments must use proper certificates; the control makes that
statement checkable.

In Kagent v1.0.x, `tls` carries only `disableVerify` (`caCertSecretRef` and
`disableSystemCAs` from older schemas were removed). On `RemoteMCPServer`, the CRD's
CEL validation strictly forbids setting `tls` when the URL uses the `http://` scheme.

**`kagent-inline-credentials` → `C-0325`**
Credential material must be referenced from Kubernetes Secrets, not embedded inline.
Fails on:
- `RemoteMCPServer.spec.headersFrom[].value` set where `.valueFrom` (which requires
  `type: Secret`, `name`, and `key`) should be used, for a header whose name indicates
  a credential (`Authorization`, `*-Api-Key`, `*-Token`, `Cookie`, and similar).
- `ModelConfig.spec.azureOpenAI.azureAdToken` set inline as a literal string.
- `ModelConfig.spec.defaultHeaders` (a `map[string]string`) containing keys that
  match credential-name patterns.
- `Harness.spec.env[].value` matching credential-name patterns.
- `SandboxTemplate.spec.env[].value` matching credential-name patterns.

`Harness` and `SandboxTemplate` environment entries (`spec.env[]`) have only `name`
and `value` fields; they do not have a `valueFrom` reference to a Secret. Any
credential placed there is inline by definition.

**Credential matching predicate (#18 §4):** Field or environment variable names are
evaluated case-insensitively against the following patterns:
- Exact matches or substrings: `AUTHORIZATION`, `API_KEY`, `APIKEY`, `TOKEN`, `SECRET`,
  `PASSWORD`, `COOKIE`.
- Values that are empty (`""`) pass (no leak). Ordinary non-credential headers
  (e.g., `User-Agent`, `Content-Type`, `Accept`) pass.

#### Tier B — the MCP trust boundary

**`kagent-plaintext-upstream-endpoint` → `C-0326`**
Fails when an upstream URL or egress origin uses the unencrypted `http://` scheme.
Agent traffic to an MCP server carries tool arguments and results — frequently the
most sensitive content in the system — and to a model provider it carries prompts and
credentials.

The URL lives across several resources and provider sub-objects:
- `RemoteMCPServer.spec.url`
- `ModelProviderConfig.spec.endpoint`
- `ModelConfig.spec.{openAI,anthropic,gemini,mistral}.baseUrl`
- `ModelConfig.spec.azureOpenAI.azureEndpoint`
- `ModelConfig.spec.foundry.endpoint`
- `ModelConfig.spec.sapAICore.{baseUrl,authUrl}`
- `ModelConfig.spec.ollama.host` (when prefixed with `http://`)
- `EgressPolicy.spec.destinations[].origin` (the CRD schema pattern allows `http://`
  origins; fails when an origin starts with `http://`)

Note: `foundry.endpointFrom` reads from a ConfigMap and cannot be judged from the CR
alone, so it is excluded. `ToolServer` from 0.10.x is removed and not matched.

**`kagent-mcp-server-unauthenticated` → `C-0327`**
Fails when a `RemoteMCPServer` declares no authentication mechanism. An unauthenticated
MCP endpoint is a tool surface that anything reachable in the network may invoke.

In Kagent v1.0.x, a `RemoteMCPServer` is considered authenticated if **any** of the
following is present:
- `spec.tokenPassthrough == true` (forwards inbound caller token),
- `spec.actorJwt` is configured (generates caller identity token),
- An authentication header is declared in `spec.headersFrom[]` (either via literal
  `value` or via referenced `valueFrom` with `type: Secret`).

**Authentication header predicate (#18 §4):** A header in `headersFrom[]` qualifies
as an authentication header if its `name` (compared case-insensitively) matches
`Authorization`, contains `API-KEY` / `APIKEY`, or ends with `TOKEN`.

The control fails only when **none** of these three mechanisms is present.

*Known false-positive class:* an in-cluster MCP server fronted by a service mesh
with mTLS needs no application-level auth header. The control's remediation text
names this case and points at risk acceptance rather than pretending it cannot
happen.

**`kagent-cross-namespace-reference-scope` → `C-0328`**
Fails when cross-namespace reference scope is overly permissive on resources that hold
credentials or provide tool models:
- `ModelConfig.spec.allowedNamespaces`
- `RemoteMCPServer.spec.allowedNamespaces`
- `ModelProviderConfig.spec.modelConfigTemplate.allowedNamespaces`

(Note: In v1.0.x, `allowedNamespaces` sits on the resources that hold credentials and
templates, rather than on `Agent`).

`allowedNamespaces` follows the Gateway API cross-namespace attachment pattern: it is
an **object**, not a list, with `from` taking `All`, `Same`, or `Selector`.

The verdicts follow:
- **`from: All`** — **Fail**. Any agent in any namespace may attach this resource.
- **field omitted, or `from: Same`** — **Pass**. `Same` is the schema default, and it
  is the secure value.
- **`from: Selector`** —
  - **`selector: {}` (or selector with empty matchLabels and matchExpressions)** —
    **Fail**. In Kubernetes label selector semantics, an empty selector `{}` selects
    *all* objects in the collection (here, all namespaces). Permitting `selector: {}`
    would be an immediate bypass of `from: All`.
  - **`selector` with populated `matchLabels` or `matchExpressions`** — **Pass**.
    A bounded, deliberate grant to specific namespaces.

#### Tier C — the agent escape surface

**`kagent-sandbox-egress-allowlist` → `C-0329`**
In Kagent v1.0.x, sandbox network egress is defined via `EgressPolicy` resources
referenced by `Agent.spec.egressPolicyRefs` or `SandboxTemplate.spec.egressPolicyRefs`.
`EgressPolicy.spec.destinations[].origin` defines permitted outbound origins.

The Kagent baseline operates on a default-deny model: when no destinations are
defined or `EgressPolicy` is absent, outbound network access is blocked. Therefore,
**absence or empty destinations is a pass**. The failure mode is an overly broad
wildcard origin.

The CRD's regex pattern already rejects `https://*`, `https://*.com`, `https://*.io`,
`https://*.*.example.com`, IP literals, and uppercase hostnames. It does, however,
syntactically allow wildcards directly over multi-label public suffixes (e.g.
`https://*.co.uk`, `https://*.github.io`, `https://*.compute.amazonaws.com`).

**Deterministic wildcard predicate (#18 §4):**
- Fails when the wildcard pattern `*.` is placed directly above a known multi-label
  public suffix.
- To avoid maintaining a massive external Public Suffix List (PSL) snapshot in Rego,
  the rule embeds a short, high-risk list of multi-label public suffixes in the
  control's default inputs (e.g. `co.uk`, `gov.uk`, `com.au`, `github.io`, `gitlab.io`,
  `azurewebsites.net`, `s3.amazonaws.com`).
- Wildcards covering organizational subdomains (e.g., `https://*.corp.example.com`,
  `https://*.internal.net`) **pass**.

#### Dropped control

**`kagent-toolserver-local-exec` (`C-0317` in earlier draft) → Dropped**
In the 0.10.x line, `ToolServer` supported a `stdio` transport where tool invocation
ran local commands inside the container. In Kagent v1.0.x, `ToolServer` has been
completely removed; all MCP tool interactions occur over network protocols (`SSE` or
`STREAMABLE_HTTP` on `RemoteMCPServer`). As agreed in #18, this control is dropped,
leaving an active block of seven controls (`C-0323`–`C-0329`).

### The served-version matrix

Two facts drive everything below:

1. **A rule fires only on the kinds and versions its `match` block names.** A rule
   matching the wrong group or version silently never fires, and **nothing in the
   test harness catches that**: fixtures exercise the rule body, not the match
   block.
2. **These are alpha APIs whose field paths move between releases.** The 0.10.x
   line served `kagent.dev` kinds at `v1alpha1`–`v1alpha3`. Kagent v1.0.0 replaced
   them with a new group (`api.kagent.dev`) and a different set of kinds.

#### Baseline

The supported baseline is **Kagent v1.0.x**, pinned at
[`v1.0.1`](https://github.com/kagent-dev/kagent/releases/tag/v1.0.1)
(`558e3d761d8ed2b0d23bfef1867a9d1915210121`). Every kind, version and field path
in this section was read from the CRD chart at that commit,
[`helm/kagent-crds/templates/`](https://github.com/kagent-dev/kagent/tree/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates).
CEL rules quoted below come from the same files.

The 0.10.x `kagent.dev` kinds are out of scope. Supporting them as well would mean
every `match` block names both groups, and the per-version path fan-out in the
previous revision of this section would come back.

#### Served versions, per kind

All eight kinds are in group `api.kagent.dev`, namespaced, and served and stored
only as `v1alpha3`. Nothing in the v1.0.1 chart serves `kagent.dev`.

| Kind | CRD | What matters here |
|---|---|---|
| `Agent` | [`api.kagent.dev_agents.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_agents.yaml) | `spec` is only `template` (inline or by reference), `harnessRef` and `egressPolicyRefs` |
| `AgentTemplate` | [`api.kagent.dev_agenttemplates.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_agenttemplates.yaml) | prompts, tools (`mcp`, `subAgent`), skills, plugins |
| `EgressPolicy` | [`api.kagent.dev_egresspolicies.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_egresspolicies.yaml) | new; `destinations[].origin`, and optional gateway-injected `authentication` |
| `Harness` | [`api.kagent.dev_harnesses.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_harnesses.yaml) | `kagent`, `claude`, `codex` or `byo`; `env`, `workload`, `substrate` |
| `ModelConfig` | [`api.kagent.dev_modelconfigs.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_modelconfigs.yaml) | 11 providers; `Mistral` is new; required `spec.model` |
| `ModelProviderConfig` | [`api.kagent.dev_modelproviderconfigs.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_modelproviderconfigs.yaml) | `modelConfigTemplate` generates `ModelConfig`s |
| `RemoteMCPServer` | [`api.kagent.dev_remotemcpservers.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_remotemcpservers.yaml) | `protocol` is `SSE` or `STREAMABLE_HTTP`; required `spec.description`, `spec.url` |
| `SandboxTemplate` | [`api.kagent.dev_sandboxtemplates.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_sandboxtemplates.yaml) | `workload.image` (the schema requires a digest), `env`, `egressPolicyRefs`, `substrate` |

`SandboxAgent`, `ToolServer`, `AgentHarness` and `Memory` are not shipped at
v1.0.1, so no rule names them.

#### Per-control kind and field paths

Every `match` block names `api.kagent.dev` / `v1alpha3` and the kinds below. With a
single served version there is no version fan-out; what changed is which kinds
carry each field.

| Control | Kinds and field paths | Notes against the previous revision |
|---|---|---|
| `C-0323` model-credential-source | `ModelConfig`: `spec.apiKeySecret`, `spec.apiKeySecretKey`, `spec.tokenPassthrough`, `spec.actorJwt`. `ModelProviderConfig`: `spec.modelConfigTemplate.auth` | `tokenPassthrough` unconditionally fails. `actorJwt` passes. `spec.apiKeySecret` and `spec.apiKeySecretKey` are string fields. CEL permits omitting `apiKeySecretKey` for `SAPAICore`; rule passes if `apiKeySecret` is present. `Mistral` requires `apiKeySecret` |
| `C-0324` tls-verification-disabled | `ModelConfig.spec.tls.disableVerify`, `RemoteMCPServer.spec.tls.disableVerify` | `tls` carries only `disableVerify`. RemoteMCPServer CEL forbids `tls` on `http://` URLs |
| `C-0325` inline-credentials | `RemoteMCPServer.spec.headersFrom[].value`, `ModelConfig.spec.azureOpenAI.azureAdToken`, `ModelConfig.spec.defaultHeaders`, `Harness.spec.env[].value`, `SandboxTemplate.spec.env[].value` | Evaluates case-insensitive credential patterns against header and env names. See below |
| `C-0326` plaintext-upstream-endpoint | `RemoteMCPServer.spec.url`, `ModelProviderConfig.spec.endpoint`, `ModelConfig.spec.{openAI,anthropic,gemini,mistral}.baseUrl`, `ModelConfig.spec.azureOpenAI.azureEndpoint`, `ModelConfig.spec.foundry.endpoint`, `ModelConfig.spec.sapAICore.{baseUrl,authUrl}`, `ModelConfig.spec.ollama.host`, `EgressPolicy.spec.destinations[].origin` | Fails on `http://` prefix. `foundry.endpointFrom` reads a ConfigMap and is skipped |
| `C-0327` mcp-server-unauthenticated | `RemoteMCPServer` | Fails when none of: authentication header in `headersFrom` (via literal `value` or `valueFrom`), `tokenPassthrough == true`, or `actorJwt` is present |
| `C-0328` cross-namespace-reference-scope | `ModelConfig.spec.allowedNamespaces`, `RemoteMCPServer.spec.allowedNamespaces`, `ModelProviderConfig.spec.modelConfigTemplate.allowedNamespaces` | Fails on `from: All` and on `from: Selector` with empty `selector: {}` |
| `C-0329` sandbox-egress-allowlist | `EgressPolicy.spec.destinations[].origin` | Destinations add origins to a deny baseline. Fails when wildcard `*.` is placed directly over a multi-label public suffix |

#### `C-0325` paths

The full list of inline credential paths on v1.0.1 is:

| Kind | Path | Shape |
|---|---|---|
| `RemoteMCPServer` | `spec.headersFrom[].value` | literal string beside `valueFrom`; only the literal is inline. (`valueFrom` requires `{type, name, key}`) |
| `ModelConfig` | `spec.azureOpenAI.azureAdToken` | literal string |
| `ModelConfig` | `spec.defaultHeaders` | `map[string]string` |
| `Harness` | `spec.env[].value` | items have only `name` and a required `value` |
| `SandboxTemplate` | `spec.env[].value` | items have only `name` and a required `value` |

`Harness` and `SandboxTemplate` env items cannot reference a Secret, so a credential
there is inline by construction. The rule evaluates `name` case-insensitively against
credential patterns (`AUTHORIZATION`, `API_KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `COOKIE`).

#### `C-0329` and the origin pattern

The `origin` pattern already rejects `https://*`, `https://*.com`, `https://*.io`,
`https://*.*.example.com`, IP literals and uppercase hosts. It accepts
`https://*.co.uk` and `https://*.github.io`. The deterministic predicate flags
wildcards directly over known multi-label public suffixes using a curated list
provided in the control inputs (`co.uk`, `gov.uk`, `com.au`, `github.io`, `gitlab.io`,
`azurewebsites.net`, `s3.amazonaws.com`).

### Controls

Seven control JSONs under `controls/`, following the shape the agent-sandbox family
established: `category: {name: "Workload"}`, `scanningScope: {matches: ["cluster",
"file"]}`, `controlTypeTags: ["security"]`, and a `long_description` that explains
why the field matters rather than restating the check.

The controls are numbered in an allocated contiguous block:
- `C-0323`: `kagent-model-credential-source`
- `C-0324`: `kagent-tls-verification-disabled`
- `C-0325`: `kagent-inline-credentials`
- `C-0326`: `kagent-plaintext-upstream-endpoint`
- `C-0327`: `kagent-mcp-server-unauthenticated`
- `C-0328`: `kagent-cross-namespace-reference-scope`
- `C-0329`: `kagent-sandbox-egress-allowlist`

`C-0311`–`C-0317` are already occupied in `regolibrary` by Agent Sandbox controls,
and the highest existing control ID is `C-0322`. `C-0323`–`C-0329` is clean,
contiguous, and unallocated.

#### Control metadata and base scores

To ensure external contributors do not need access to internal historical design documents,
the exact metadata, scoring, and remediation definitions for all seven controls are published below:

| Control ID | Name | Base Score | Rule Name | Description | Remediation |
|---|---|---|---|---|---|
| `C-0323` | Kagent model credential source | **9.0** | `kagent-model-credential-source` | Ensure model provider credentials reference a Kubernetes Secret rather than passing caller tokens or omitting secrets. | Disable `spec.tokenPassthrough` and configure `spec.apiKeySecret` to reference a Kubernetes Secret containing the provider credential. |
| `C-0324` | Kagent TLS verification disabled | **8.0** | `kagent-tls-verification-disabled` | Ensure TLS certificate verification is enforced for connections to model providers and remote MCP tool servers. | Set `spec.tls.disableVerify` to `false` or omit the `tls` configuration on `ModelConfig` and `RemoteMCPServer` resources. |
| `C-0325` | Kagent inline credentials | **7.0** | `kagent-inline-credentials` | Ensure credentials and API tokens are not embedded inline in headers, default configs, or container environments. | Replace inline string values with `valueFrom.secretKeyRef` in `headersFrom`, and avoid embedding sensitive tokens in `defaultHeaders` or `spec.env`. |
| `C-0326` | Kagent plaintext upstream endpoint | **7.0** | `kagent-plaintext-upstream-endpoint` | Ensure all upstream endpoints (model providers, MCP servers, and egress origins) use encrypted HTTPS schemes. | Replace `http://` URLs with `https://` across `RemoteMCPServer.spec.url`, `ModelConfig` provider base URLs, and `EgressPolicy.spec.destinations`. |
| `C-0327` | Kagent MCP server unauthenticated | **8.0** | `kagent-mcp-server-unauthenticated` | Ensure remote MCP servers declare an authentication mechanism before tool execution is permitted. | Provide an authentication header in `spec.headersFrom`, or enable `spec.tokenPassthrough` or `spec.actorJwt` on `RemoteMCPServer`. |
| `C-0328` | Kagent cross-namespace reference scope | **7.0** | `kagent-cross-namespace-reference-scope` | Ensure cross-namespace access to model configs and tool servers is restricted to bounded selectors or same namespace. | Set `spec.allowedNamespaces.from` to `Same` (default), or use `Selector` with specific `matchLabels` rather than `All` or empty `{}`. |
| `C-0329` | Kagent sandbox egress allowlist | **8.0** | `kagent-sandbox-egress-allowlist` | Ensure sandboxed agent execution does not permit overly broad wildcard egress over public multi-label suffixes. | Restrict `EgressPolicy.spec.destinations[].origin` to specific organizational domains or fully qualified subdomains. |

### Framework

Extend the existing `frameworks/agent-runtime-hardening.json` ("Agent Runtime
Hardening") by adding `C-0323`–`C-0329` to its `activeControls` list. This gives
operators one framework that answers "is the agent infrastructure in this cluster
configured safely", across both upstream Agent Sandbox resources and Kagent's own.

`allcontrols`, `armobest` and `devopsbest` are **not** modified. Adding Kagent
controls to `armobest` would change compliance scores for every user, and for the
large majority who do not run Kagent the controls pass vacuously — noise in the
control list for no signal.

Operators run the scan via:
```sh
kubescape scan framework AgentRuntimeHardening
```

## Verification

### Contributor workflow and CI verification

Contributors implementing these rules in `regolibrary` should follow the standard
development and verification workflow matching repository CI:

1. **Rule execution and unit testing:**
   ```sh
   cd testrunner
   go test -v -run TestSingleRule -rule kagent-model-credential-source
   ```
2. **Rego linting:**
   ```sh
   regal lint --format github rules/kagent-*
   ```
3. **Control ID duplicate and schema validation:**
   ```sh
   python ./scripts/validations.py
   ```
4. **Framework subsection generation:**
   ```sh
   python ./scripts/generate_subsections_ids.py
   ```
5. **Regolibrary artifact bundling:**
   ```sh
   OUTPUT=release python ./scripts/export.py
   ```
6. **Local Kubescape validation with `--use-from`:**
   ```sh
   kubescape scan framework AgentRuntimeHardening --use-from release/agent-runtime-hardening.json
   ```

### Unit tests and testrunner fixtures

Each rule ships fixtures under `rules/<name>/test/`, one directory per case named
`success-*` or `failed-*`, run through `regolibrary/testrunner`:

All fixtures must validate against the pinned Kagent v1.0.1 CRD schemas
(`api.kagent.dev/v1alpha3`). Required test cases:

- **`C-0323` (credential source):**
  - `failed-token-passthrough`: `ModelConfig` with `spec.tokenPassthrough: true`.
  - `failed-missing-api-key`: `ModelConfig` for `OpenAI` / `Mistral` without `apiKeySecret`.
  - `success-api-key-secret`: `ModelConfig` with valid `spec.apiKeySecret` and `spec.apiKeySecretKey`.
  - `success-ambient-identity`: `ModelConfig` for `Bedrock` / `Foundry` without `apiKeySecret`.
  - `success-actor-jwt`: `ModelConfig` with `spec.actorJwt` set.
  - `success-sapaicore`: `ModelConfig` for `SAPAICore` with `apiKeySecret` present and `apiKeySecretKey` omitted (#18 §4).
- **`C-0324` (TLS verification):**
  - `failed-disable-verify`: `ModelConfig` or `RemoteMCPServer` with `spec.tls.disableVerify: true`.
  - `success-tls-default`: `spec.tls` omitted or `disableVerify: false`.
- **`C-0325` (inline credentials):**
  - `failed-headers-from`: `RemoteMCPServer` with `spec.headersFrom[].value` set for `Authorization`.
  - `failed-azure-ad-token`: `ModelConfig` with literal `spec.azureOpenAI.azureAdToken`.
  - `failed-default-headers`: `ModelConfig` with `spec.defaultHeaders` containing `X-API-Key`.
  - `failed-env-credential`: `Harness` or `SandboxTemplate` with `spec.env[].name` matching `*_API_KEY`.
  - `success-referenced`: `RemoteMCPServer` using `valueFrom: {type: Secret, name: mcp-auth-secret, key: token}`, `Harness` with non-credential `spec.env`.
- **`C-0326` (plaintext endpoints):**
  - `failed-remote-mcp`: `RemoteMCPServer` with `http://` URL.
  - `failed-provider-endpoint`: `ModelConfig` with `http://` in `baseUrl` or `authUrl`.
  - `failed-egress-origin`: `EgressPolicy` with `http://` origin.
  - `success-https`: All URLs and origins using `https://`.
- **`C-0327` (unauthenticated MCP):**
  - `failed-unauthenticated`: `RemoteMCPServer` without `headersFrom`, `tokenPassthrough`, or `actorJwt`.
  - `success-headers`: `RemoteMCPServer` with `Authorization` header in `headersFrom`.
  - `success-tokens`: `RemoteMCPServer` with `tokenPassthrough: true` or `actorJwt`.
- **`C-0328` (cross-namespace scope):**
  - `failed-from-all`: `ModelConfig` or `RemoteMCPServer` with `from: All`.
  - `failed-empty-selector`: `from: Selector` with `selector: {}` or empty match rules (#18 §4).
  - `success-default`: `allowedNamespaces` omitted or `from: Same`.
  - `success-bounded-selector`: `from: Selector` with populated `matchLabels`.
- **`C-0329` (egress allowlist):**
  - `failed-public-suffix`: `EgressPolicy` with `https://*.co.uk` or `https://*.github.io`.
  - `success-empty`: Destinations omitted or empty (secure default-deny).
  - `success-subdomain`: Specific domain wildcard `https://*.api.example.com`.

### End to end

1. kind cluster, Kagent v1.0.x installed from its Helm chart:
   ```sh
   helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
     --version 1.0.1 --namespace kagent --create-namespace
   helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
     --version 1.0.1 --namespace kagent
   ```
2. Export the local `regolibrary` bundle:
   ```sh
   OUTPUT=release python scripts/export.py
   ```
3. Run `kubescape` against the local framework export using `--use-from`:
   ```sh
   kubescape scan framework AgentRuntimeHardening --use-from release/agent-runtime-hardening.json
   ```
4. Apply the insecure CR manifest set and secure CR manifest set (`apiVersion: api.kagent.dev/v1alpha3`).
5. Assert each control fires exactly once on the insecure set and not at all on the secure set.

### Schema-valid E2E test manifests

To decouple fixture verification from chart installation variances, two canonical manifest sets
are provided below. Each resource conforms strictly to the Kagent v1.0.1 OpenAPI v3 schemas
(including required `spec.model` on `ModelConfig`, required `spec.description` on `RemoteMCPServer`,
and `{type, name, key}` on `valueFrom`). Each resource in the insecure set trips **exactly one**
target control, ensuring clean, unambiguous assertions.

#### Insecure manifest set (`insecure-kagent-crs.yaml`)

```yaml
---
# Trips C-0323: model credential source (tokenPassthrough == true)
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-insecure-c0323
  namespace: kagent
spec:
  provider: OpenAI
  model: "gpt-4o"
  openAI:
    baseUrl: "https://api.openai.com/v1"
  tokenPassthrough: true
---
# Trips C-0324: TLS verification disabled
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-insecure-c0324
  namespace: kagent
spec:
  provider: Bedrock
  model: "anthropic.claude-v2"
  bedrock:
    region: us-east-1
  tls:
    disableVerify: true
---
# Trips C-0325: inline credentials
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-insecure-c0325
  namespace: kagent
spec:
  description: "Test remote MCP server with inline credentials"
  url: "https://tools.example.com/mcp"
  protocol: SSE
  headersFrom:
    - name: Authorization
      value: "Bearer hardcoded-inline-token"
---
# Trips C-0326: plaintext upstream endpoint
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-insecure-c0326
  namespace: kagent
spec:
  description: "Test remote MCP server with plaintext URL"
  url: "http://tools.example.com/mcp"
  protocol: SSE
  tokenPassthrough: true
---
# Trips C-0327: MCP server unauthenticated
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-insecure-c0327
  namespace: kagent
spec:
  description: "Test unauthenticated remote MCP server"
  url: "https://tools.example.com/mcp"
  protocol: SSE
---
# Trips C-0328: cross-namespace reference scope (from: All)
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-insecure-c0328
  namespace: kagent
spec:
  provider: Bedrock
  model: "anthropic.claude-v2"
  bedrock:
    region: us-east-1
  allowedNamespaces:
    from: All
---
# Trips C-0329: sandbox egress allowlist (wildcard over public suffix)
apiVersion: api.kagent.dev/v1alpha3
kind: EgressPolicy
metadata:
  name: test-insecure-c0329
  namespace: kagent
spec:
  destinations:
    - origin: "https://*.co.uk"
```

#### Secure manifest set (`secure-kagent-crs.yaml`)

```yaml
---
# Passes C-0323: references Secret for OpenAI
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-secure-c0323
  namespace: kagent
spec:
  provider: OpenAI
  model: "gpt-4o"
  openAI:
    baseUrl: "https://api.openai.com/v1"
  apiKeySecret: "openai-secret"
  apiKeySecretKey: "api-key"
---
# Passes C-0324: TLS verification enabled
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-secure-c0324
  namespace: kagent
spec:
  provider: Bedrock
  model: "anthropic.claude-v2"
  bedrock:
    region: us-east-1
  tls:
    disableVerify: false
---
# Passes C-0325: references Secret via valueFrom (type, name, key)
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-secure-c0325
  namespace: kagent
spec:
  description: "Test remote MCP server with referenced credentials"
  url: "https://tools.example.com/mcp"
  protocol: SSE
  headersFrom:
    - name: Authorization
      valueFrom:
        type: Secret
        name: mcp-auth-secret
        key: token
---
# Passes C-0326: HTTPS endpoint
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-secure-c0326
  namespace: kagent
spec:
  description: "Test remote MCP server with HTTPS endpoint"
  url: "https://tools.example.com/mcp"
  protocol: SSE
  tokenPassthrough: true
---
# Passes C-0327: authenticated via tokenPassthrough
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: test-secure-c0327
  namespace: kagent
spec:
  description: "Test authenticated remote MCP server"
  url: "https://tools.example.com/mcp"
  protocol: SSE
  tokenPassthrough: true
---
# Passes C-0328: bounded selector
apiVersion: api.kagent.dev/v1alpha3
kind: ModelConfig
metadata:
  name: test-secure-c0328
  namespace: kagent
spec:
  provider: Bedrock
  model: "anthropic.claude-v2"
  bedrock:
    region: us-east-1
  allowedNamespaces:
    from: Selector
    selector:
      matchLabels:
        environment: production
---
# Passes C-0329: specific organizational domain
apiVersion: api.kagent.dev/v1alpha3
kind: EgressPolicy
metadata:
  name: test-secure-c0329
  namespace: kagent
spec:
  destinations:
    - origin: "https://*.api.corp.example.com"
```

### Expected findings matrix

A default Kagent v1.0.1 Helm install (`helm/kagent`) deploys three custom resources:
1. `ModelConfig` `default-model-config` (from `modelconfig.yaml`): configures the default OpenAI provider referencing Secret `kagent-openai`.
2. `RemoteMCPServer` `kagent-tool-server` (from `toolserver-kagent.yaml`): configures the built-in tool server over HTTP without auth.
3. `RemoteMCPServer` `kagent-api` (from `toolserver-api.yaml`): configures controller API tools over HTTP without auth.

The matrix below differentiates the stock install baseline from the test manifest suites:

| Control ID | Stock Helm Install Findings (`default-model-config`, `kagent-tool-server`, `kagent-api`) | Insecure Manifest Set | Secure Manifest Set |
|---|---|---|---|
| `C-0323` | **Pass** (`default-model-config` correctly references `apiKeySecret: kagent-openai`) | **Fail** (`test-insecure-c0323`) | Pass |
| `C-0324` | **Pass** (`tls` omitted on all default resources; verification enabled by default) | **Fail** (`test-insecure-c0324`) | Pass |
| `C-0325` | **Pass** (no inline credentials in headers or env) | **Fail** (`test-insecure-c0325`) | Pass |
| `C-0326` | **Fail (2 findings)** (`kagent-tool-server` and `kagent-api` both configure plaintext `http://` URLs) | **Fail** (`test-insecure-c0326`) | Pass |
| `C-0327` | **Fail (2 findings)** (`kagent-tool-server` and `kagent-api` declare no authentication mechanism) | **Fail** (`test-insecure-c0327`) | Pass |
| `C-0328` | **Pass** (`allowedNamespaces` omitted on all default resources, securely defaulting to `Same`) | **Fail** (`test-insecure-c0328`) | Pass |
| `C-0329` | **Pass** (the chart creates no `EgressPolicy`, so there is nothing to evaluate) | **Fail** (`test-insecure-c0329`) | Pass |

### Expected result on a default install, stated up front

A stock Kagent v1.0.x Helm install is expected to **fail two controls**, producing a total of
**four findings** across two default `RemoteMCPServer` objects created by the chart:

1. `kagent-tool-server` (`helm/kagent/templates/toolserver-kagent.yaml`):
```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: kagent-tool-server
  namespace: kagent
spec:
  url: "http://kagent-tools.kagent.svc.cluster.local:8084/mcp"
  timeout: 30s
  sseReadTimeout: 5m0s
  description: "Official KAgent tool server"
```

2. `kagent-api` (`helm/kagent/templates/toolserver-api.yaml`):
```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: kagent-api
  namespace: kagent
spec:
  description: "Kagent API tools for Sessions, checkpoints, and sandboxes"
  protocol: STREAMABLE_HTTP
  url: "http://kagent-controller.kagent.svc.cluster.local:8083/mcp"
  timeout: 30s
  sseReadTimeout: 5m0s
```

Both resources specify plaintext `http://` URLs and declare no authentication mechanisms (`headersFrom`,
`tokenPassthrough`, or `actorJwt`), tripping:
- **`C-0326`** (plaintext upstream endpoint) — 2 findings
- **`C-0327`** (MCP server without authentication) — 2 findings

In contrast, the default `ModelConfig` resource (`default-model-config`) safely references `kagent-openai`
and passes `C-0323`. Neither resource trips `C-0328` because `allowedNamespaces` is omitted, which defaults
to `from: Same`.

These four findings are true positives reflecting Kagent's permissive chart defaults. They provide clear
rationale for upstream chart hardening PRs, and reviewers should expect this non-clean baseline in E2E testing.

## Alternatives considered

**Extend the existing agent-sandbox controls instead of allocating new IDs.**
Adding the Kagent rule names to `C-0301`/`C-0303`/`C-0304`/`C-0309`'s `rulesNames`
and their match blocks avoids ID coordination entirely. Rejected: it puts two
unrelated APIs under one control, so remediation text has to describe both and a
finding no longer tells you which resource shape to fix. The IDs are cheap; the
conflation is not.

**Add the controls to `armobest` and `allcontrols`, as the agent-sandbox family
did.** Rejected for now, on the reasoning in the Framework section. Revisit with
real-cluster evidence.

**Cover pod security on the CRs too, for shift-left signal.** Rejected on the
duplicate-finding reasoning in the non-goals. A `file`-scope-only variant is the
better answer if the gap proves painful.

**Implement all fifteen controls from the 2025 document as written.** Not
possible — four have no field or no resource to match. Re-deriving against the
shipped schemas is the only honest option.

**Add a control resolving each resource's ServiceAccount and cross-checking its
real RBAC bindings.** Rejected: `serviceAccountName` lives in the workload/pod
block and is already checked on rendered workloads, while RBAC objects are scanned
directly by existing controls.

## Resolved decisions (#18)

- **Baseline:** Kagent v1.0.x only (`api.kagent.dev/v1alpha3`). Legacy 0.10.x
  `kagent.dev` schemas are excluded from the match blocks to avoid multi-group fan-out.
- **Control allocation:** `C-0323` to `C-0329` (seven controls). `C-0317` is dropped
  as `ToolServer` and stdio MCP transport are not shipped in v1.0.x.
- **Framework:** Extend the existing `frameworks/agent-runtime-hardening.json`
  ("Agent Runtime Hardening"). No changes to `allcontrols`, `armobest` or `devopsbest`.
- **Policy predicates:**
  - `C-0323`: `SAPAICore` passes with `apiKeySecret` even if `apiKeySecretKey` is absent.
  - `C-0325`: Evaluates case-insensitive credential patterns on header and env names.
  - `C-0327`: Authenticated via `tokenPassthrough`, `actorJwt`, or auth header in `headersFrom`.
  - `C-0328`: `from: Selector` with empty `selector: {}` fails.
  - `C-0329`: Evaluates wildcard origins against a curated list of multi-label public
    suffixes in the control inputs.
- **Local testing:** Verified using `kubescape scan framework AgentRuntimeHardening --use-from <exported-bundle.json>`.

## Changes since the first review

Recorded so reviewers can see what moved rather than re-reading the whole document.

### Round one

- **`C-0315` rewritten.** `allowedNamespaces` is a Gateway-API-style object with
  `from: All|Same|Selector`, not a list whose bad value is `*`.
- **`C-0317` (tool session isolation) dropped.** `isolateSessions` is valid only on
  `Agent`-type tools and is a fan-out flag, not a tenant boundary.
- **`C-0310` made provider-aware.** Ambient providers (`Bedrock`, `Foundry`, etc.)
  inherit pod identity and pass without `apiKeySecret`.

### Round two

- Enumerated multi-version path shapes across 0.10.x alpha schemas (`Agent`,
  `SandboxAgent`, `ModelConfig`).

### Round three: Kagent v1.0.x baseline

- Rebuilt the served-version matrix against the Kagent v1.0.1 CRD chart
  (`558e3d761d8ed2b0d23bfef1867a9d1915210121`). All kinds moved to
  `api.kagent.dev/v1alpha3`.
- `SandboxAgent` and `ToolServer` removed. `EgressPolicy`, `Harness`,
  `AgentTemplate` and `SandboxTemplate` introduced.

### Round four: full alignment with Kagent v1.0.x and maintainer decisions (#18, #40)

- **Control allocation & framework integration:** Allocated `C-0323`–`C-0329` in a
  free contiguous block. Dropped `C-0317` (`kagent-toolserver-local-exec`). Extended
  the existing `frameworks/agent-runtime-hardening.json`.
- **Control metadata & base scores:** Published the complete metadata matrix
  (base scores, descriptions, and remediations) for `C-0323` through `C-0329`.
- **Rule descriptions and field paths:** Updated all seven rule descriptions to
  match the v1.0.1 schemas and field paths.
- **Explicit policy predicates (#18 §4):**
  - Codified `SAPAICore` handling under `C-0323` (`apiKeySecretKey` omitted is valid).
  - Defined header and env credential-name matching patterns under `C-0325`.
  - Defined MCP server authentication mechanisms under `C-0327`.
  - Codified verdict for `from: Selector` with empty `selector: {}` as **Fail** under `C-0328`.
  - Codified deterministic public-suffix wildcard check with curated list under `C-0329`.
- **Schema-valid fixtures and E2E manifests:** All 16 manifests in this document (the 14
  test manifests and the 2 stock examples) are admitted by `kubectl-validate` against the
  Kagent v1.0.1 CRDs, OpenAPI schema and CEL rules (required `model` on `ModelConfig`,
  required `description` on `RemoteMCPServer`, and `{type, name, key}` on `valueFrom`).
- **Stock install verification:** Verified the exact custom resources rendered by
  stock `helm/kagent` (`default-model-config`, `kagent-tool-server`, and `kagent-api`)
  and documented the expected 4 findings in the findings matrix.

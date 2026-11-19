# Proposal: Posture rules for Kagent CRDs (`kagent.dev`)

## Summary

Add eight Rego rules and eight controls to `regolibrary` covering insecure
configuration of Kagent's custom resources, and a new **Agent Runtime Hardening**
framework (`agentic`) that gathers them together with the existing Agent Sandbox
controls. Kagent is a CNCF Sandbox project for Kubernetes-native AI agent
orchestration; its CRDs carry the credentials, TLS posture, and tool-trust
decisions for every agent in the cluster, and nothing in `regolibrary` looks at
them today.

This is the Kagent-specific continuation of
[`agent-sandbox-crd-scanning.md`](agent-sandbox-crd-scanning.md). That proposal
established the general case — agent-runtime custom resources are posture objects
and belong in the scan — and delivered rules against the upstream Agent Sandbox
API groups (`agents.x-k8s.io`, `extensions.agents.x-k8s.io`). It did not cover any
concrete agent platform's own API group. This proposal does that for `kagent.dev`,
against fields verified in the shipped CRD schemas.

No new sensing, no runtime work, no admission work. Static posture only, on
capability Kubescape already has.

## Motivation

Kagent's resources sit directly on the trust boundary between a language model,
external MCP tool servers, and the Kubernetes API:

- A `ModelConfig` decides whether the provider API key comes from a `Secret` or is
  passed through from whatever bearer token arrived on an inbound A2A request.
- A `RemoteMCPServer` decides whether the agent trusts an upstream tool server's
  certificate at all, and whether it authenticates to it.
- A `ToolServer` decides whether tool invocation means an HTTP call or a local
  command executing in the cluster.
- An `Agent` decides which domains its sandboxed execution may reach, and which
  namespaces may attach it as a tool.

Each of these is a checkable field with an unambiguous insecure value, and none of
them is visible on the `Deployment` the Kagent controller renders. A cluster can
pass every existing workload control while running an agent that disables TLS
verification against an MCP server and forwards caller tokens to an LLM provider.

The value is also cumulative: this is the first control set in `regolibrary`
targeting a specific agent framework's API, and it establishes the pattern (rule
naming, multi-version matching, framework placement) for the agent platforms that
follow.

## Goals

- Eight rules under `rules/kagent-*`, each with unit test fixtures that pass in
  `testrunner`.
- Eight controls, `C-0310`–`C-0317`, in a free contiguous block.
- A new framework `agentic.json` ("Agent Runtime Hardening") containing the eight
  new controls plus the existing `C-0297`, `C-0301`, `C-0303`, `C-0304`, `C-0309`.
- End-to-end verification: a real `kubescape scan` against a kind cluster running
  Kagent, with a deliberately-insecure CR set and a secure CR set.

## Non-goals

### Pod security on Kagent CRs

`Agent` and `SandboxAgent` embed pod-shaped configuration under
`spec.byo.deployment.*` and `spec.declarative.deployment.*`: `securityContext`,
`podSecurityContext`, `volumes`, `resources`, `imageRegistry`, `extraContainers`,
`serviceAccountName`. Writing controls against those fields is **explicitly out of
scope**, because the Kagent controller renders them into a `Deployment` that
`regolibrary` already scans — `C-0009` (resource limits), `C-0016` (privilege
escalation), `C-0017` (read-only root filesystem), `C-0048` (hostPath),
`C-0237` (image provenance) and their siblings all fire on the rendered workload.

Duplicating them on the CR produces two findings for one misconfiguration and two
places to keep remediation text correct. The dividing line for this proposal is
therefore: **a rule earns its place only if the misconfiguration is invisible on
the generated Deployment.** All eight rules below meet that test.

This does cost the shift-left signal in file-scan mode, where no `Deployment`
exists to scan. That is a known and accepted gap; if it proves painful in practice
the right fix is a follow-up that scans the CR's deployment block in `file` scope
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
Kagent controls. That document is now twenty months old and the `kagent.dev` API
has moved substantially. The proposal below is a re-derivation against the shipped
CRD schemas, and the drift is recorded here because it is the reason the control
list changed shape.

| The 2025 document assumed | Reality in the shipped CRDs |
|---|---|
| `executeCodeBlocks: true` on `Agent` (its highest-severity control, 9/9) | **The field does not exist anywhere in the CRDs.** Sandboxed execution posture is now expressed as `Agent.spec.sandbox.network.allowedDomains`, which denies egress by default when unset or empty |
| An `MCPServer` CRD | **No such CRD.** The two live resources are `RemoteMCPServer` (an upstream URL) and `ToolServer` (a tool server config with `stdio` / `sse` / `streamableHttp` variants) |
| `Agent` carries pod fields directly | Nested under `spec.byo.*` **and** `spec.declarative.*`, with or without an intervening `deployment` level depending on the served version, and the same shape repeated on `SandboxAgent` — see the served-version matrix |
| Four pod-security controls | Out of scope per the non-goal above — they duplicate existing workload controls |
| — | Fields the document predates: `ModelConfig.spec.apiKeyPassthrough`, `tls.disableSystemCAs`, `spec.allowedNamespaces` (a Gateway-API-style object, not a list) |

Four of the fifteen proposed controls therefore have no field to check or no
resource to match, and four more needed their paths reworked. Eight survive
re-derivation, and one — `kagent-toolserver-local-exec` — is new, standing in for
the intent behind the dead `executeCodeBlocks` control.

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

They are grouped in three tiers by implementation order. All eight are in scope for
this work; the tiers describe sequence, not priority cuts.

#### Tier A — credentials and transport

These are mechanical: the insecure value is explicit in the schema and there is no
judgment call about what "bad" means.

**`kagent-model-credential-source` → `C-0310`**
A `ModelConfig` must not hold its provider credential in a way that bypasses
Kubernetes secret management. Two distinct failures:

- **`spec.apiKeyPassthrough == true`**, for any provider. This forwards the bearer
  token from an inbound A2A request to the LLM provider as the API key, making
  every caller's token a provider credential and removing the secret-management
  boundary entirely. Unconditional failure.
- **An API-key provider with no `spec.apiKeySecret` + `spec.apiKeySecretKey`.**

The second branch is **provider-aware**, because `spec.provider` spans ten values
and only some of them authenticate with an API key at all:

| `spec.provider` | Credential model | Verdict when `apiKeySecret` absent |
|---|---|---|
| `Anthropic`, `OpenAI`, `AzureOpenAI`, `Gemini`, `SAPAICore` | API key (`SAPAICore` exchanges it at `authUrl`) | **Fail** |
| `Bedrock` | AWS SDK credential chain — sub-object carries only `region` | Pass |
| `Foundry` | Azure workload identity / `DefaultAzureCredential` — sub-object carries `endpoint`, `apiVersion` | Pass |
| `GeminiVertexAI`, `AnthropicVertexAI` | GCP ADC — sub-object carries `projectID`, `location` | Pass |
| `Ollama` | Typically unauthenticated — sub-object carries `host` | Pass |

The ambient-identity providers carry no credential field anywhere in their
sub-objects, which is the schema-level evidence that they are meant to inherit an
identity from the pod. Failing them for a missing `apiKeySecret` would mark the
*more* secure configuration — workload identity over a long-lived static key — as
the insecure one.

`apiKeyPassthrough` and `apiKeySecret` are mutually exclusive under the CRD's own
CEL validation, so the two branches cannot both fire on one object.

On **`ModelConfig/v1alpha1`** the field is named **`apiKeySecretRef`**, not
`apiKeySecret`, and there is no `apiKeyPassthrough` at all — so only the second
branch applies there, reading a different path. That version's `provider` enum is
also shorter. See the served-version matrix below.

An unauthenticated `Ollama` endpoint is a real posture question, but it is a
property of the endpoint rather than of this resource's credential handling, and
`C-0313` already covers the plaintext-transport half of it. Left out deliberately
rather than folded in here.

**`kagent-tls-verification-disabled` → `C-0311`**
Fails on `spec.tls.disableVerify == true` on `ModelConfig` or `RemoteMCPServer`.
The CRD's own field documentation says this is for development only and that
production deployments must use proper certificates; the control makes that
statement checkable. `spec.tls.disableSystemCAs == true` needs no rule of its own:
the CRD's CEL requires it to be accompanied by *either* a `caCertSecretRef` *or*
`disableVerify` (a trust-nothing config would reject every upstream). The
`caCertSecretRef` case is a legitimate strict-trust configuration and passes; the
`disableVerify` case is already caught by this control's main branch.

`ModelConfig/v1alpha1` has no `tls` field, so this control matches only `v1alpha2`
and `v1alpha3` of that kind.

**`kagent-inline-credentials` → `C-0312`**
Credential material must be referenced, not embedded. Fails on:
- `RemoteMCPServer.spec.headersFrom[].value` set where `.valueFrom` should be used,
  for a header whose name indicates a credential (`Authorization`, `*-Api-Key`,
  `*-Token`, `Cookie`, and similar),
- `ToolServer.spec.config.{sse,streamableHttp}.headers` — an
  `x-kubernetes-preserve-unknown-fields` map, so anything inline here is a literal
  value in the manifest,
- `deployment.env[].value` matching credential-name patterns, on both the `byo` and
  `declarative` paths.

This is the one rule in scope whose field paths differ between served versions —
the `deployment` nesting level is present in some and absent in others, across four
distinct shapes on `Agent` and `SandboxAgent`. It handles them through an explicit
path list rather than a duplicated rule body or an assumed nesting depth. The four
shapes are enumerated in the served-version matrix below, and each needs its own
fixture.

#### Tier B — the MCP trust boundary

**`kagent-plaintext-upstream-endpoint` → `C-0313`**
Fails when an upstream URL uses the `http://` scheme. Agent traffic to an MCP
server carries tool arguments and results — frequently the most sensitive content
in the system — and to a model provider it carries prompts and credentials.

The URL does not live in one place. Paths to cover:

- `RemoteMCPServer.spec.url`
- `ModelProviderConfig.spec.endpoint`
- `ToolServer.spec.config.sse.url`, `ToolServer.spec.config.streamableHttp.url`
- `ModelConfig` **provider sub-objects**, which each carry their own base URL:
  `openAI.baseUrl`, `azureOpenAI.azureEndpoint`, `foundry.endpoint`,
  `sapAICore.baseUrl`, `sapAICore.authUrl`, `ollama.host`

The `ModelConfig` paths were missing from an earlier draft of this proposal, which
named only the first three bullets. A self-hosted or gateway-fronted model endpoint
reached over plaintext is the same finding as a plaintext MCP server, and it is
configured on a resource this control set already matches.

**`kagent-mcp-server-unauthenticated` → `C-0314`**
Fails when a `RemoteMCPServer` declares no authentication header in
`spec.headersFrom`. An unauthenticated MCP endpoint is a tool surface anything
that can reach it may drive.

*Known false-positive class:* an in-cluster MCP server fronted by a service mesh
with mTLS needs no application-level auth header. The control's remediation text
names this case and points at risk acceptance rather than pretending it cannot
happen. The alternative — inspecting mesh configuration from a CRD rule — is not
something Rego over a single object can do.

**`kagent-cross-namespace-reference-scope` → `C-0315`**
Fails when `spec.allowedNamespaces.from == "All"` on `Agent`, **`SandboxAgent`**
or `RemoteMCPServer`. `SandboxAgent` embeds the same spec shape and carries the
field in both its served versions; `Agent` carries it in `v1alpha2` only.

`allowedNamespaces` follows the Gateway API cross-namespace attachment pattern: it
is an **object**, not a list, with `from` taking `All`, `Same`, or `Selector`, and
it governs which namespaces may **reference this resource as a tool** — not which
namespaces the resource watches.

The verdicts follow from that shape:

- **`from: All`** — fail. Any agent in any namespace may attach this resource as a
  tool, so a tool server holding production credentials is reachable from a
  scratch namespace.
- **field omitted, or `from: Same`** — pass. `Same` is the schema default, and it
  is the secure value.
- **`from: Selector`** — pass. A deliberate, bounded grant; the CRD's own CEL
  already requires a `selector` alongside it.

This is the correction of an earlier draft of this proposal, which treated the
field as a list whose insecure value was `*` and treated absence as a failure.
Both were wrong: absence is the secure default, and `*` is not a value the schema
accepts. The rule as drafted would have failed every secure resource and passed
every genuinely open one.

#### Tier C — the agent escape surface

These are the rules that describe the actual agent trust boundary, and each
requires the proposal to take a position on what the secure default is.

**`kagent-sandbox-egress-allowlist` → `C-0316`**
`spec.sandbox.network.allowedDomains` on `Agent` and **`SandboxAgent`** governs
what sandboxed execution may contact. The field is identical in all three served
shapes that have it; `Agent/v1alpha1` has no `sandbox` block. The API denies egress by default when the field is unset or empty, so
**absence is a pass** — the failure mode is a permissive allowlist. Fails on `*`,
on a bare-TLD wildcard (`*.com`, `*.io`), and on `*.` entries broad enough to be
equivalent to open egress. Named domains and specific subdomain wildcards pass.

This rule carries the intent of the 2025 document's `executeCodeBlocks` control:
model-generated code will run, and the question that matters is where it can
reach.

**`kagent-toolserver-local-exec` → `C-0317`**
Fails when `ToolServer.spec.config.stdio` is present. The `stdio` transport means
tool invocation runs a command in the tool server's container, driven by
model-selected arguments — a local execution path where the `sse` and
`streamableHttp` variants are network calls to a bounded surface. Legitimate uses
exist, which is why this is a control to justify rather than a hard failure; the
remediation text says so.

### The served-version matrix

Two facts drive everything below:

1. **A rule fires only on the kinds and versions its `match` block names.** A rule
   matching the wrong group or version silently never fires, and **nothing in the
   test harness catches that**: fixtures exercise the rule body, not the match
   block.
2. **These are alpha APIs whose field paths move between releases.** The 0.10.x
   line served `kagent.dev` kinds at `v1alpha1`–`v1alpha3`. Kagent v1.0.0 replaced
   them with a new group and a different set of kinds.

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
| `ModelConfig` | [`api.kagent.dev_modelconfigs.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_modelconfigs.yaml) | 11 providers; `Mistral` is new |
| `ModelProviderConfig` | [`api.kagent.dev_modelproviderconfigs.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_modelproviderconfigs.yaml) | `modelConfigTemplate` generates `ModelConfig`s |
| `RemoteMCPServer` | [`api.kagent.dev_remotemcpservers.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_remotemcpservers.yaml) | `protocol` is `SSE` or `STREAMABLE_HTTP` |
| `SandboxTemplate` | [`api.kagent.dev_sandboxtemplates.yaml`](https://github.com/kagent-dev/kagent/blob/558e3d761d8ed2b0d23bfef1867a9d1915210121/helm/kagent-crds/templates/api.kagent.dev_sandboxtemplates.yaml) | `workload.image` (the schema requires a digest), `env`, `egressPolicyRefs`, `substrate` |

`SandboxAgent`, `ToolServer`, `AgentHarness` and `Memory` are not shipped at
v1.0.1, so no rule names them.

#### Per-control kind and field paths

Every `match` block names `api.kagent.dev` / `v1alpha3` and the kinds below. With a
single served version there is no version fan-out; what changed is which kinds
carry each field.

| Control | Kinds and field paths | Notes against the previous revision |
|---|---|---|
| `C-0310` model-credential-source | `ModelConfig`: `spec.apiKeySecret`, `spec.tokenPassthrough`, `spec.actorJwt`. `ModelProviderConfig`: `spec.modelConfigTemplate.auth` | `apiKeyPassthrough` is now `tokenPassthrough`. `actorJwt` is a third credential mode, and CEL allows at most one of `actorJwt`, `apiKeySecret`, `tokenPassthrough`; `actorJwt` passes. CEL already rejects `tokenPassthrough` for `Gemini`, `GeminiVertexAI` and `AnthropicVertexAI`. `Mistral` takes an API key, so it fails without a secret. The SAPAICore case from #18 §4 still holds: `apiKeySecretKey` is unused for it |
| `C-0311` tls-verification-disabled | `ModelConfig.spec.tls.disableVerify`, `RemoteMCPServer.spec.tls.disableVerify` | `tls` now has only `disableVerify`; `caCertSecretRef` and `disableSystemCAs` are gone. RemoteMCPServer CEL forbids `tls` on an `http://` URL |
| `C-0312` inline-credentials | `RemoteMCPServer.spec.headersFrom[].value`, `ModelConfig.spec.azureOpenAI.azureAdToken`, `ModelConfig.spec.defaultHeaders`, `Harness.spec.env[].value`, `SandboxTemplate.spec.env[].value` | See below |
| `C-0313` plaintext-upstream-endpoint | `RemoteMCPServer.spec.url`, `ModelProviderConfig.spec.endpoint`, `ModelConfig.spec.{openAI,anthropic,gemini,mistral}.baseUrl`, `ModelConfig.spec.azureOpenAI.azureEndpoint`, `ModelConfig.spec.foundry.endpoint`, `ModelConfig.spec.sapAICore.{baseUrl,authUrl}`, `ModelConfig.spec.ollama.host`, `EgressPolicy.spec.destinations[].origin` | `anthropic`, `gemini` and `mistral` base URLs are new to the list, and so is `origin`, whose pattern accepts `http://`. `foundry.endpointFrom` reads a ConfigMap and cannot be judged from the object alone |
| `C-0314` mcp-server-unauthenticated | `RemoteMCPServer` | A server is authenticated by a credential header in `headersFrom`, by `tokenPassthrough`, or by `actorJwt`; CEL bars an `Authorization` header alongside either of the last two. The control fails only when none is present. Which header names count is still open (#18 §4) |
| `C-0315` cross-namespace-reference-scope | `ModelConfig.spec.allowedNamespaces`, `RemoteMCPServer.spec.allowedNamespaces`, `ModelProviderConfig.spec.modelConfigTemplate.allowedNamespaces` | `Agent` no longer has the field; it now sits on the resources that hold credentials. CEL only requires `selector` to be present, so `from: Selector` with `selector: {}` is accepted and matches every namespace. The verdict for that case is still open (#18 §4) |
| `C-0316` sandbox-egress-allowlist | `EgressPolicy.spec.destinations[].origin`, reached through `Agent.spec.egressPolicyRefs` and `SandboxTemplate.spec.egressPolicyRefs` | `sandbox.network.allowedDomains` is gone. Destinations add origins to a deny baseline. See below for what the schema already rejects |
| `C-0317` toolserver-local-exec | none | `ToolServer` was removed and no v1.0.1 kind has a stdio MCP transport. The nearest surface is a `byo` `Harness` with `spec.workload.command`, which runs a digest-pinned image. **Proposed: drop this control**, leaving seven |

#### `C-0312` paths

The four env path shapes of the previous revision, and the `ToolServer` header and
stdio-env cases, are gone. The full list on v1.0.1 is:

| Kind | Path | Shape |
|---|---|---|
| `RemoteMCPServer` | `spec.headersFrom[].value` | literal string beside `valueFrom`; only the literal is inline |
| `ModelConfig` | `spec.azureOpenAI.azureAdToken` | literal string |
| `ModelConfig` | `spec.defaultHeaders` | `map[string]string`, new in this list |
| `Harness` | `spec.env[].value` | items have only `name` and a required `value` |
| `SandboxTemplate` | `spec.env[].value` | items have only `name` and a required `value` |

`Harness` and `SandboxTemplate` env items cannot reference a Secret, so a credential
there is inline by construction. The name-matching rules #18 §4 asks for decide
which of those entries the control flags.

#### `C-0316` and the origin pattern

The `origin` pattern already rejects `https://*`, `https://*.com`, `https://*.io`,
`https://*.*.example.com`, IP literals and uppercase hosts. It accepts
`https://*.co.uk` and `https://*.github.io`. The only broad case left is a wildcard
directly over a public suffix, which gives the deterministic predicate #18 asks
for: fail when the labels after `*.` are a public suffix. Whether that list is a
pinned Public Suffix List snapshot or a short fixed list is open.

### Controls

Eight control JSONs under `controls/`, following the shape the agent-sandbox family
established: `category: {name: "Workload"}`, `scanningScope: {matches: ["cluster",
"file"]}`, `controlTypeTags: ["security"]`, and a `long_description` that explains
why the field matters rather than restating the check. Base scores carry over from
the 2025 document where the control survived re-derivation.

`C-0310`–`C-0317` is free and contiguous: the highest control ID currently in
`regolibrary` is `C-0309`, allocated to the agent-sandbox family.

### Framework

A new `frameworks/agentic.json`, **Agent Runtime Hardening**, containing the eight
new controls together with the existing agent-runtime controls `C-0297`, `C-0301`,
`C-0303`, `C-0304` and `C-0309`. This gives operators one framework that answers
"is the agent infrastructure in this cluster configured safely", across both the
upstream Agent Sandbox resources and Kagent's own.

`allcontrols`, `armobest` and `devopsbest` are **not** modified. Adding eight
controls to `armobest` would change compliance scores for every user, and for the
large majority who do not run Kagent the controls pass vacuously — noise in the
control list for no signal. Promoting them into the broader frameworks is a
reasonable follow-up once the rules have run against real clusters, and is easier
to argue with that evidence in hand.

## Verification

### Unit

Each rule ships fixtures under `rules/<name>/test/`, one directory per case named
`success-*` or `failed-*`, run through `regolibrary/testrunner`:

```sh
cd testrunner && go test -run TestSingleRule -rule kagent-model-credential-source
```

Every rule gets at minimum:

- an **omitted-field** case, asserting the correct verdict when the field is absent
  — a pass where the API's default is secure (`sandbox.network.allowedDomains`,
  `tls.disableVerify`, `allowedNamespaces`), a failure where absence is itself the
  problem (`apiKeySecret`, for an API-key provider only),
- an **explicit-insecure** case,
- an **explicit-secure** case.

The omitted-field distinction is where the agent-sandbox family's subtlety lives —
`C-0309` exists in the shape it does specifically to avoid a false positive on a
field whose absence is safe — and it is the case most likely to be got wrong. It is
also where the first draft of this proposal got `C-0315` backwards, so the fixture
set is the mechanism that would have caught it: an omitted-`allowedNamespaces`
success case fails loudly against a rule that treats absence as insecure.

`C-0310` additionally needs one fixture **per provider class** — at minimum an
API-key provider without `apiKeySecret` (fail), a `Bedrock` or `Foundry` config
without one (pass), and an `apiKeyPassthrough: true` case (fail regardless of
provider).

### Fixtures per served shape, not per rule

A fixture set that covers only the newest schema is the single easiest way to ship
a rule with a silent false negative, because **the match block is untested**: the
harness feeds the rule body an object directly, so a rule that would never be
selected on a real cluster still passes its unit tests. The version matrix is
therefore a fixture requirement, not just implementation guidance.

Each rule needs at least one fixture **per distinct shape** in its row of the
per-control coverage table:

- **`C-0312` — four env fixtures minimum**, one per path shape:
  `Agent/v1alpha1` (`spec.deployment.env`), the v1alpha2 shape
  (`spec.{declarative,byo}.deployment.env`), and `SandboxAgent/v1alpha3`
  (`spec.{declarative,byo}.env`), plus the `RemoteMCPServer`, `ToolServer` and
  `azureAdToken` paths.
- **`C-0310` — both field names**: `apiKeySecret` on v1alpha2/v1alpha3 and
  `apiKeySecretRef` on v1alpha1.
- **`C-0315` and `C-0316` — a `SandboxAgent` fixture each**, not only an `Agent`
  one, since the resources share the field and differ only in kind.

Where a kind/version genuinely lacks the field a rule targets — `Agent/v1alpha1`
for `C-0315` and `C-0316`, `ModelConfig/v1alpha1` for `C-0311` — the correct
treatment is to leave it out of the match block, and to note it in the control's
`long_description` so a later reader does not "fix" the omission by adding a
version where the rule can only ever pass vacuously.

### End to end

1. kind cluster, Kagent installed from its Helm chart.
2. `kubescape` built from source, pointed at the local `regolibrary` bundle rather
   than the released one.
3. Apply an insecure CR set (one CR per rule, each tripping exactly one control)
   and a secure CR set.
4. Assert each control fires exactly once on the insecure set and not at all on the
   secure set.
5. `kubescape scan framework agentic` on the whole cluster, reviewing every finding
   for false positives.

### Expected result on a default install, stated up front

A stock Kagent Helm install is expected to **fail two controls**, both on a single
object. The chart auto-creates a `RemoteMCPServer` for its own built-in tool
server:

```yaml
apiVersion: kagent.dev/v1alpha3
kind: RemoteMCPServer
spec:
  url: "http://<tools-service>.<namespace>:8084/mcp"
  timeout: 30s
  sseReadTimeout: 5m0s
  description: "Official KAgent tool server"
```

No `headersFrom` and a plaintext `http://` URL, so it trips `C-0313` (plaintext
upstream endpoint) and `C-0314` (MCP server without authentication).

It does **not** trip `C-0315`: `allowedNamespaces` is omitted, which defaults to
`from: Same` and is the secure value. This is worth stating explicitly because an
earlier draft of this proposal predicted a `C-0315` failure here — that prediction
was an artifact of the same misreading corrected under `C-0315` above, and it
disappears once the rule matches the schema.

A plain install ships no demo `Agent`, so nothing else in the default set is
expected to fail.

These are true positives, not rules to tune away: the chart's defaults are
permissive, which is the same observation the 2025 document made about empty
security contexts and disabled NetworkPolicies. The in-cluster plaintext URL is the
most arguable of the two — see the false-positive note under `C-0314` — and is
the one most likely to end in a documented risk acceptance rather than a chart
change.

Those findings are a deliverable. They become input to the Kagent pod-hardening
work and to an upstream PR proposing tighter chart defaults. The rules encode the
secure posture; where Kagent's defaults do not yet meet it, the gap is the finding.

Reviewers should expect a non-clean E2E report and read it as intended behaviour.

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
real RBAC bindings.** Raised in review, and the underlying point is sound:
privilege inheritance is the path an attacker actually walks, and a resource can
look perfectly scoped in its own spec while running under a ServiceAccount bound to
`cluster-admin`. Not adopted here, for three reasons:

- `serviceAccountName` lives in the `deployment` block, which is out of scope
  precisely because it survives onto the rendered Deployment — where existing
  RBAC and privilege controls already see it.
- Over-permissive bindings, `cluster-admin` among them, are already covered by
  existing `regolibrary` controls that scan RBAC objects directly.
- Resolving a ServiceAccount to its effective permissions means joining across
  `ServiceAccount`, `RoleBinding` and `ClusterRoleBinding` objects. That is a
  materially different rule shape from anything in this set, and it belongs in the
  RBAC control family rather than bolted onto a CRD-field family.

What would be genuinely additive is a control asserting that an agent's *claimed*
tool scope matches its *granted* permissions. That needs a definition of claimed
scope that the CRDs do not currently express, and is a better fit for a follow-up
proposal than a tenth rule here.

## Open questions

- Confirmation from `regolibrary` maintainers that `C-0310`–`C-0317` is theirs to
  take, and that `agentic` is the right framework name and filename.
- Whether the framework should also be offered under a name that signals scope more
  loudly (`agent-runtime-hardening`) given `agentic` is short and broad.
- `kagent.dev` resources are all `v1alpha*`. Alpha APIs change, and this proposal
  is itself a record of what that costs. Worth deciding whether rules should be
  gated on an observed API version and how the family gets re-validated when Kagent
  cuts a beta.
- The mechanics of pointing a locally built `kubescape` at a locally built
  `regolibrary` bundle for step 2 of the E2E, and whether the CI bundle needs a
  release before `kubescape` picks the rules up.

## Changes since the first review

Recorded so reviewers can see what moved rather than re-reading the whole document.

**Blocking schema mismatches, all confirmed against the shipped CRDs and fixed:**

- **`C-0315` rewritten.** `allowedNamespaces` is a Gateway-API-style object with
  `from: All|Same|Selector`, not a list whose bad value is `*`. The original rule
  would have failed the secure default and missed the actual open value. Renamed to
  `kagent-cross-namespace-reference-scope`, since it governs cross-namespace tool
  *attachment*, not namespace watching.
- **`C-0317` (tool session isolation) dropped**, taking the set from nine controls
  to eight and the block to `C-0310`–`C-0317`. `isolateSessions` is valid only on
  `Agent`-type tools (CEL rejects it on `McpServer`) and is a parallel-fan-out
  correctness flag, not a tenant boundary. Rationale kept in the document rather
  than deleted, because the field reads like a security control and will tempt the
  next reader.
- **`C-0310` made provider-aware** and renamed to `kagent-model-credential-source`.
  `Bedrock`, `Foundry`, `GeminiVertexAI`, `AnthropicVertexAI` and `Ollama` carry no
  credential field in their sub-objects and inherit an ambient identity; a blanket
  `apiKeySecret` requirement marked those configurations insecure for using
  workload identity over a static key. `apiKeyPassthrough` remains an
  unconditional failure.

**Smaller corrections from review:**

- `disableSystemCAs` requires `caCertSecretRef` **or** `disableVerify`, not
  `caCertSecretRef` alone. Stated as an AND before; corrected under `C-0311`.
- The default-install prediction was wrong in both directions and is now grounded
  in the chart's own `RemoteMCPServer` template: it fails `C-0313` and `C-0314`
  (plaintext URL, no auth headers), and **passes** `C-0315`, where the first draft
  predicted a failure.
- Added a rejected-alternatives entry for the ServiceAccount/RBAC cross-check
  proposed in review.

The served-version matrix and the absence of `executeCodeBlocks` from all ten CRDs
were independently verified in review against live schemas on a kind cluster, and
both hold as documented.

### Round two

One further blocker: the served-version matrix omitted `Agent` and `SandboxAgent`
entirely, which is the same class of error as round one — reasoning from the newest
schema instead of enumerating what the API actually serves. Every claim below was
verified against the shipped CRDs.

- **`Agent` serves `v1alpha1` and `v1alpha2` only; `SandboxAgent` serves `v1alpha2`
  and `v1alpha3`.** The two kinds overlap only at `v1alpha2`, and an earlier draft
  of this document described `Agent` as though it served `v1alpha3`.
- **`C-0312` has four env path shapes, not one.** `spec.deployment.env` on
  `Agent/v1alpha1`; `spec.{declarative,byo}.deployment.env` on the v1alpha2 shape;
  `spec.{declarative,byo}.env` on `SandboxAgent/v1alpha3`, where the `deployment`
  nesting level disappears. The previous text named only the middle shape, so an
  implementation following it would have missed inline credentials on half the
  served shapes.
- **`C-0315` and `C-0316` now name `SandboxAgent`.** It embeds the same spec shape
  including `allowedNamespaces` and `sandbox.network.allowedDomains`, so naming
  only `Agent` skipped two served resources carrying the same insecure values.

Three further faults of the same class, found while building the matrix and not
raised in review:

- **`ModelConfig/v1alpha1` names the field `apiKeySecretRef`, not `apiKeySecret`**,
  and has no `apiKeyPassthrough`. `C-0310` reads a different path there, and its
  provider enum is seven values rather than ten.
- **`ModelConfig/v1alpha1` has no `tls` field**, so `C-0311` must match only
  `v1alpha2` and `v1alpha3` of that kind.
- **`C-0313` was missing `ModelConfig`'s own endpoints.** Provider sub-objects each
  carry a base URL (`openAI.baseUrl`, `azureOpenAI.azureEndpoint`,
  `foundry.endpoint`, `sapAICore.baseUrl`/`authUrl`, `ollama.host`); a plaintext
  model endpoint is the same finding as a plaintext MCP server. Also added
  `azureOpenAI.azureAdToken` to `C-0312` as an inline-credential path.

`Agent/v1alpha1` additionally has neither `allowedNamespaces` nor a `sandbox`
block, so `C-0315` and `C-0316` deliberately exclude it rather than matching a
version where they could only pass vacuously.

The version matrix is now stated as a per-control kind × version table with the
version-specific paths spelled out, and fixtures are required per served shape
rather than per rule — since a rule whose match block omits a version still passes
its unit tests, that requirement is the only thing standing between this design and
a silent false negative.

Also fixed the "three"/"two" default-install typo.

### Round three: Kagent v1.0.x baseline

Kagent v1.0.0 (Nov 9, 2026) moved every kind to `api.kagent.dev/v1alpha3` and
changed the set of kinds, so the served-version matrix is re-derived from the
v1.0.1 CRD chart at `558e3d761d8ed2b0d23bfef1867a9d1915210121` (#18 §2 and §3).
The main effects:

- One group and one version, so the per-version fan-out and the four `C-0312` env
  shapes are gone.
- `SandboxAgent` and `ToolServer` are not shipped. `EgressPolicy`, `Harness`,
  `AgentTemplate` and `SandboxTemplate` are new.
- `C-0312` gains `ModelConfig.spec.defaultHeaders` and the `Harness` and
  `SandboxTemplate` env lists. `C-0313` gains the `anthropic`, `gemini` and
  `mistral` base URLs and `EgressPolicy` origins.
- `C-0315` now reads `allowedNamespaces` on `ModelConfig`, `RemoteMCPServer` and
  `ModelProviderConfig`, not `Agent`.
- `C-0316` reads `EgressPolicy` origins instead of
  `sandbox.network.allowedDomains`.
- `C-0317` has no target at v1.0.1 and is proposed for removal.

Only the matrix section is updated in this round. The rule descriptions, fixtures
and E2E example above still describe the 0.10.x schemas, and follow once the
baseline is agreed. Control IDs and framework membership (#18 §1) are unchanged
here.

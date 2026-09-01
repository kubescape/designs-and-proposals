# Proposal: Posture rules for Kagent CRDs (`kagent.dev`)

## Summary

Add nine Rego rules and nine controls to `regolibrary` covering insecure
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
- An `Agent` decides which namespaces it watches, which domains its sandboxed
  execution may reach, and whether tool sessions are isolated from one another.

Each of these is a checkable field with an unambiguous insecure value, and none of
them is visible on the `Deployment` the Kagent controller renders. A cluster can
pass every existing workload control while running an agent that disables TLS
verification against an MCP server and forwards caller tokens to an LLM provider.

The value is also cumulative: this is the first control set in `regolibrary`
targeting a specific agent framework's API, and it establishes the pattern (rule
naming, multi-version matching, framework placement) for the agent platforms that
follow.

## Goals

- Nine rules under `rules/kagent-*`, each with unit test fixtures that pass in
  `testrunner`.
- Nine controls, `C-0310`–`C-0318`, in a free contiguous block.
- A new framework `agentic.json` ("Agent Runtime Hardening") containing the nine
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
the generated Deployment.** All nine rules below meet that test.

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
| `Agent` carries pod fields directly | Nested under `spec.byo.deployment.*` **and** `spec.declarative.deployment.*`, with the same shape repeated on `SandboxAgent` |
| Four pod-security controls | Out of scope per the non-goal above — they duplicate existing workload controls |
| — | Fields the document predates: `ModelConfig.spec.apiKeyPassthrough`, `tls.disableSystemCAs`, `declarative.tools[].isolateSessions`, `spec.allowedNamespaces` |

Four of the fifteen proposed controls therefore have no field to check or no
resource to match, and four more needed their paths reworked. Nine survive
re-derivation, and one — `kagent-toolserver-local-exec` — is new, standing in for
the intent behind the dead `executeCodeBlocks` control.

## Proposal

### Rules

Rules live at `rules/kagent-<aspect>/` and follow the `agent-sandbox-*` family's
conventions exactly: `package armo_builtins`, `import rego.v1`,
`deny contains msga if`, `object.get(resource, [...], <explicit default>)` so an
omitted field cannot error the rule, a local `resource_name()` helper, and
`failedPaths` naming the real field path so the CLI can point at it.

They are grouped in three tiers by implementation order. All nine are in scope for
this work; the tiers describe sequence, not priority cuts.

#### Tier A — credentials and transport

These are mechanical: the insecure value is explicit in the schema and there is no
judgment call about what "bad" means.

**`kagent-model-api-key-secret` → `C-0310`**
A `ModelConfig` must source its provider credential from
`spec.apiKeySecret` + `spec.apiKeySecretKey`. Fails when neither is set, and fails
when `spec.apiKeyPassthrough` is `true` — that forwards the bearer token from an
inbound A2A request to the LLM provider as the API key, which makes every caller's
token a provider credential and removes the secret-management boundary entirely.
The two are mutually exclusive in the API, so the rule reports whichever applies.

**`kagent-tls-verification-disabled` → `C-0311`**
Fails on `spec.tls.disableVerify == true` on `ModelConfig` or `RemoteMCPServer`.
The CRD's own field documentation says this is for development only and that
production deployments must use proper certificates; the control makes that
statement checkable. `spec.tls.disableSystemCAs == true` without a
`caCertSecretRef` is rejected by the CRD's own CEL validation and needs no rule;
`disableSystemCAs` *with* a custom CA is a legitimate strict-trust configuration
and passes.

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

This is the one rule in scope that touches the dual deployment paths, and it
handles them through a shared path list rather than a duplicated rule body.

#### Tier B — the MCP trust boundary

**`kagent-plaintext-upstream-endpoint` → `C-0313`**
Fails when an upstream URL uses the `http://` scheme:
`RemoteMCPServer.spec.url`, `ModelProviderConfig.spec.endpoint`, and the URL inside
`ToolServer.spec.config.sse` / `.streamableHttp`. Agent traffic to an MCP server
carries tool arguments and results — frequently the most sensitive content in the
system — and to a model provider it carries prompts and credentials.

**`kagent-mcp-server-unauthenticated` → `C-0314`**
Fails when a `RemoteMCPServer` declares no authentication header in
`spec.headersFrom`. An unauthenticated MCP endpoint is a tool surface anything
that can reach it may drive.

*Known false-positive class:* an in-cluster MCP server fronted by a service mesh
with mTLS needs no application-level auth header. The control's remediation text
names this case and points at risk acceptance rather than pretending it cannot
happen. The alternative — inspecting mesh configuration from a CRD rule — is not
something Rego over a single object can do.

**`kagent-crd-namespace-scope` → `C-0315`**
Fails when `spec.allowedNamespaces` is unset or contains `*` on `Agent` or
`RemoteMCPServer`. An agent that watches every namespace is an agent whose blast
radius is the cluster, and the field exists precisely so operators can bound it.

#### Tier C — the agent escape surface

These are the rules that describe the actual agent trust boundary, and each
requires the proposal to take a position on what the secure default is.

**`kagent-sandbox-egress-allowlist` → `C-0316`**
`Agent.spec.sandbox.network.allowedDomains` governs what sandboxed execution may
contact. The API denies egress by default when the field is unset or empty, so
**absence is a pass** — the failure mode is a permissive allowlist. Fails on `*`,
on a bare-TLD wildcard (`*.com`, `*.io`), and on `*.` entries broad enough to be
equivalent to open egress. Named domains and specific subdomain wildcards pass.

This rule carries the intent of the 2025 document's `executeCodeBlocks` control:
model-generated code will run, and the question that matters is where it can
reach.

**`kagent-tool-session-isolation` → `C-0317`**
Fails when an `Agent.spec.declarative.tools[]` entry of MCP-server type does not
set `isolateSessions: true`. Without isolation, state a tool accumulates in one
session is reachable from the next, which is a cross-tenant leak in any agent
serving more than one user.

**`kagent-toolserver-local-exec` → `C-0318`**
Fails when `ToolServer.spec.config.stdio` is present. The `stdio` transport means
tool invocation runs a command in the tool server's container, driven by
model-selected arguments — a local execution path where the `sse` and
`streamableHttp` variants are network calls to a bounded surface. Legitimate uses
exist, which is why this is a control to justify rather than a hard failure; the
remediation text says so.

### Two hazards specific to these CRDs

**Multi-version `match` blocks.** `ModelConfig` serves `v1alpha1`, `v1alpha2` and
`v1alpha3`; `RemoteMCPServer` serves `v1alpha2` and `v1alpha3`; `ToolServer` serves
`v1alpha1` only. Every `rule.metadata.json` `match` entry must list all served
versions of the resource it targets. A rule matching only the newest version
silently never fires on a cluster storing an older one, and nothing in the test
harness catches that — the fixtures are what the test feeds the rule, not what the
match block selects.

**Dual deployment paths.** Anything reading pod-shaped fields must read both
`spec.byo.deployment.*` and `spec.declarative.deployment.*`, and the identical
shape on `SandboxAgent`. Within this proposal's scope only `C-0312` is affected,
but the helper it introduces is the pattern for any later work that crosses into
the deployment block.

### Controls

Nine control JSONs under `controls/`, following the shape the agent-sandbox family
established: `category: {name: "Workload"}`, `scanningScope: {matches: ["cluster",
"file"]}`, `controlTypeTags: ["security"]`, and a `long_description` that explains
why the field matters rather than restating the check. Base scores carry over from
the 2025 document where the control survived re-derivation.

`C-0310`–`C-0318` is free and contiguous: the highest control ID currently in
`regolibrary` is `C-0309`, allocated to the agent-sandbox family.

### Framework

A new `frameworks/agentic.json`, **Agent Runtime Hardening**, containing the nine
new controls together with the existing agent-runtime controls `C-0297`, `C-0301`,
`C-0303`, `C-0304` and `C-0309`. This gives operators one framework that answers
"is the agent infrastructure in this cluster configured safely", across both the
upstream Agent Sandbox resources and Kagent's own.

`allcontrols`, `armobest` and `devopsbest` are **not** modified. Adding nine
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
cd testrunner && go test -run TestSingleRule -rule kagent-model-api-key-secret
```

Every rule gets at minimum:

- an **omitted-field** case, asserting the correct verdict when the field is absent
  — a pass where the API's default is secure (`sandbox.network.allowedDomains`,
  `tls.disableVerify`), a failure where absence is itself the problem
  (`apiKeySecret`, `allowedNamespaces`),
- an **explicit-insecure** case,
- an **explicit-secure** case.

The omitted-field distinction is where the agent-sandbox family's subtlety lives —
`C-0309` exists in the shape it does specifically to avoid a false positive on a
field whose absence is safe — and it is the case most likely to be got wrong.

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

A stock Kagent Helm install is expected to **fail** `C-0314` (MCP servers without
authentication headers) and `C-0315` (cluster-wide namespace scope). This is a
true positive, not a rule to tune away: the chart's defaults are permissive, which
is the same observation the 2025 document made about empty security contexts and
disabled NetworkPolicies.

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

## Open questions

- Confirmation from `regolibrary` maintainers that `C-0310`–`C-0318` is theirs to
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

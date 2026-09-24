# Proposal: Attack-Path Analysis in the Kubescape CLI (`kubescape scan attack-paths`)

- **Status:** Draft – discussion
- **Author:** Mohit ([@mohityadav8](https://github.com/mohityadav8))
- **Date:** 2026-09-19
- **Scope:** [`kubescape`](https://github.com/kubescape/kubescape) CLI (`core/pkg/*`, `cmd/scan`, `core/pkg/resultshandling/printer`)
- **Related work:** [`vex-ingestion`](./vex-ingestion.md), [`evidence-of-finding`](./evidence-of-finding.md), [`repository-scan-contracts`](./repository-scan-contracts.md)

## 1. Summary

The `kubescape` repository already contains four static analysis engines, each answering one isolated question about a cluster:

| Engine | Package | Question it answers |
|---|---|---|
| External exposure | `core/pkg/exposure` | Is this Service reachable from outside the cluster, and by what mechanism? |
| Network reachability | `core/pkg/networkpolicy` | Can pod A reach pod B, and which policy decides? |
| RBAC escalation | `core/pkg/rbacgraph` | If this identity is compromised, what else (including cluster-admin) becomes reachable? |
| Vulnerability × exposure | `core/pkg/vulnexposure` | Which CVEs sit on workloads that are actually reachable? |

Today these are consumed **only** by the MCP server (`cmd/mcpserver/*`), and only one question at a time. `kubescape scan` does not use any of them, and nothing chains them.

This proposal adds one CLI command, `kubescape scan attack-paths`, that runs all four engines over a **single resource-collection pass**, joins their outputs on workload identity, and reports ranked, end-to-end paths such as:

```
Internet → Ingress → Service → Deployment (Critical CVE) → ServiceAccount → RBAC escalation → cluster-admin
```

The work is mostly integration. The engines, their tests and their `FromResources` adapters exist. Three pieces are genuinely missing and are the new code (see §3.2).

## 2. Motivation

Every engine above is deliberately an **upper-bound static model**, and each documents that trust model in its package comment. Used alone, each produces a finding that is hard to prioritise:

- A critical CVE on a workload nobody can reach is a different risk from the same CVE on a workload behind a public Ingress.
- A ServiceAccount that can `create pods` is a routine finding until you notice that a *publicly reachable, vulnerable* workload runs as it.
- An RBAC escalation path is only urgent if something an attacker can reach holds the starting identity.

`vulnexposure` already proves the value of joining two engines: its package comment states that a fixable critical CVE on a workload open to the whole cluster "is a materially different risk than the same CVE on a workload no NetworkPolicy lets anything reach". This proposal generalises that idea across all four engines.

Two further gaps motivate doing it in the CLI rather than leaving it in MCP:

1. **CI and PR-time use.** The static engines work from manifests, so a chained check could gate a pull request (`--fail-on-path`). MCP cannot do this.
2. **Discoverability.** Users who never run an MCP client cannot reach this capability at all.

### Relationship to existing attack-track output

`--print-attack-tree` (via `core/pkg/resourcesprioritization`) maps *failed controls* onto predefined attack-track definitions. It is control-driven, not graph-driven, and it does not consult reachability, exposure or RBAC chaining. This proposal does not replace it (see Non-goals and §8).

## 3. Proposed design

### 3.1 What already exists (verified against `kubescape` master)

- Each of `exposure`, `networkpolicy` and `rbacgraph` exposes a `FromResources(map[string]workloadinterface.IMetadata)` adapter that already consumes the scanner's own generic resource map, skipping (and reporting) objects that fail to decode rather than aborting.
- `exposure.Index.ServiceExposure(ServiceRef)` returns `[]ExposurePath` plus an `unclear` flag.
- `networkpolicy.Index` provides `Reaches` and `IngressExposure(Endpoint)`.
- `rbacgraph.Index.AnalyzeEscalation(Subject)` returns `EscalationResult{ClusterAdmin, Reached, Unbounded, Truncated}`.
- `vulnexposure.Correlate(idx, endpoints, vulnsByWorkload, minSeverity)` returns findings and a `skipped` list.
- The MCP vulnerability tool (`cmd/mcpserver/vulnerability_exposure.go`) already resolves a workload to a `networkpolicy.Endpoint` via a pod-template-labels resolver covering Pod, Deployment, StatefulSet, DaemonSet, ReplicaSet, Job and CronJob.
- The printer registry (`printer.AllFormats`, `IPrinter`) is the extension point for output formats.

### 3.2 What is missing (the new code)

I searched the non-test Go code for these and found no existing implementation:

1. **Service → backing workload.** `exposure` stops at the Service: it reports *that* a Service is reachable but never resolves *which workload* sits behind it. Nothing matches a Service's `spec.selector` to pod-template labels.
2. **Workload → ServiceAccount.** No non-test code outside the anonymiser reads a workload's `spec.template.spec.serviceAccountName` or `automountServiceAccountToken` for graph purposes. `rbacgraph` documents the pod-creation edge but nothing links a *running workload* to its identity.
3. **A join layer.** Nothing composes the four engines' outputs into one graph.

The pod-template resolver in the MCP vulnerability tool is a useful starting point for (2), but it lives in `cmd/mcpserver` and would need to move to a shared package.

### 3.3 Architecture

```
 live cluster / manifests
        │  ONE collection pass (existing resourcehandler)
        ▼
  map[string]IMetadata
        │
        ├─ exposure.FromResources        ─┐
        ├─ networkpolicy.FromResources    ├─ existing adapters
        ├─ rbacgraph.FromResources        │
        └─ vulnexposure (+ VulnManifests)─┘
                    │
                    ▼
        NEW core/pkg/attackpath
          ├─ inputs.go     single call to all four adapters, aggregated warnings
          ├─ workload.go   Service→workload, workload→ServiceAccount (§3.2)
          ├─ model.go      Node / Edge / Path types
          ├─ builder.go    join engines into a directed graph
          ├─ search.go     bounded, deterministic path search
          └─ score.go      path scoring (documented, auditable)
                    │
                    ▼
        NEW printer/v2 attack-path printer  →  pretty | json | sarif | markdown
                    │
                    ▼
        NEW cmd/scan/attackpaths.go  (flags, exit code)
```

### 3.4 Data model (sketch)

```go
type NodeKind string // Internet, Service, Workload, ServiceAccount, RBACSubject, CVE, ClusterAdmin

type EdgeKind string // exposes, network-reach, runs-as, escalates, vulnerable

type Edge struct {
    From, To NodeID
    Kind     EdgeKind
    Evidence string // e.g. Ingress name, policy name, RBAC verb, CVE id
    Certain  bool   // false when the source engine returned Unknown/unclear/Truncated
}
```

`Certain` is load-bearing. The engines deliberately return `Unknown`, `unclear` and `Truncated` in places (a named container port, an IP-block peer, a cross-namespace route reference, a search that hit its safety bound). The path layer must **propagate** that uncertainty and never silently promote it to "reachable" or demote it to "safe".

### 3.5 Evaluating the network hop

`networkpolicy.IngressExposure` reports the widest source class a workload's rules admit (open, external-cidr, any-namespace, restricted). It is not a verdict for a specific upstream, so the Service → workload hop is evaluated against the **actual source of the exposure path**:

| Exposure path | Source endpoint | Rule |
|---|---|---|
| Ingress / Gateway route | The ingress-controller pod | `Reaches(controllerEndpoint, workloadEndpoint)` when the controller can be identified in the collected resources; otherwise the edge is emitted with `Certain=false`. |
| LoadBalancer / NodePort | An external IP | `Certain=false` in v1. `networkpolicy.Endpoint` represents a pod, not an external source (`core/pkg/networkpolicy/index.go`), so `Reaches` cannot yet distinguish an external IP from a cluster pod: an empty `namespaceSelector` is treated as an unconditional match (`match.go`) and would wrongly admit Internet traffic. Until an external-aware matcher exists that explicitly excludes pod/namespace-selector rules for external sources (with a regression test covering this case), every LB/NodePort hop is reported as `Certain=false`, never promoted on port or CIDR grounds alone. |

A workload restricted to the ingress-controller namespace must therefore be reported as reachable from the Internet through that Ingress, not as unreachable. LB/NodePort paths remain `Certain=false` until the external-aware matcher above exists; `--fail-on-path` CI runs will therefore treat most of these edges as uncertain by default, and the gate must decide explicitly whether uncertain edges count (see §9).

### 3.6 CLI surface

```bash
kubescape scan attack-paths [flags]
```

Indicative flags (final set to be settled in review):

| Flag | Purpose |
|---|---|
| `--from` | Start node: `internet` (default) or `<namespace>/<kind>/<name>` |
| `--to` | Sink: `cluster-admin` (default) |
| `--max-depth` | Search bound |
| `--min-severity` | Minimum CVE severity to include |
| `--with-vulns` | Include `VulnerabilityManifest` data when present |
| `--fail-on-path` | Exit non-zero when a matching path exists (CI gate) |
| `-f/--format` | `pretty`, `json`, `sarif`, `markdown` |

Existing flags (`--include-namespaces`, `--exclude-namespaces`, `--kube-context`, `--hide`, `--encrypt`, `--exceptions`) are reused, not redefined. Local manifest scanning is supported, since the engines are static; `ipBlock` peers then remain `Unknown`, which is already their documented behaviour.

### 3.7 Example output

```
PATH #1  score 9.6 (CRITICAL)
  [Internet] --(Ingress shop/web-ing, host shop.example.com)--> Service shop/web
      --(no NetworkPolicy, ingress OPEN)--> Deployment shop/web
          finding: CVE-XXXX-YYYY (Critical, fix available)
      runs as ServiceAccount shop/web-sa
          --(create pods + assign-serviceaccount)--> ServiceAccount kube-system/deployer
              --(bind verb)--> ClusterRole cluster-admin
  ==> CLUSTER-ADMIN REACHABLE FROM INTERNET IN 3 HOPS 
```

  A hop is one edge between two nodes in the path; the example above has 6 edges. Final wording for the "N HOPS" summary line is to be settled in review.

## 4. Goals

- One command that reports ranked end-to-end paths from a single collection pass.
- Reuse the four existing engines and their adapters; add only the three missing pieces in §3.2.
- Preserve uncertainty end to end (`unknown`, `unclear`, `truncated` stay visible).
- Deterministic output: the same input yields byte-identical results. (`rbacgraph` already sorts by resource ID because map iteration order would otherwise reorder reported paths; the join layer must follow the same discipline.)
- Usable as a CI gate via an exit code.
- Honour `--hide` and `--encrypt`, since paths expose namespace, ServiceAccount, host and CVE names.

## 5. Non-goals

- **No runtime data.** This stays a static model with the same trust model the engines already document. Reported paths are an upper bound of what specs allow.
- **No new escalation primitives** in v1. Only chaining what `rbacgraph` already models.
- **No admission-control modelling** (PodSecurity, Gatekeeper, Kyverno) in v1. Those would only narrow edges, never widen them, so omitting them errs toward over-reporting, not under-reporting. See §9.
- **Not replacing `--print-attack-tree`** or changing any of the four existing MCP tools' names or output.
- **No auto-remediation.** Fix suggestions, if any, are text/JSON only.

## 6. Phased delivery

Each phase is independently mergeable and ships standalone value.

| Phase | Scope | Risk |
|---|---|---|
| **1** | Shared input adapter (one call to all four `FromResources`); Service→workload and workload→ServiceAccount resolvers, moving the pod-template resolver out of `cmd/mcpserver` into a shared package. **Library only, no CLI.** | Low–medium: additive; the MCP tools are refactored to call the shared code with unchanged output. |
| **2** | Graph builder, bounded deterministic path search, scoring. Still no CLI flag. | Medium: scoring is subjective, so ship the formula documented and unit-tested. |
| **3** | `kubescape scan attack-paths` with `pretty` and `json` output, `--fail-on-path`, docs. **First user-visible release.** | Medium: new command surface. |
| **4** | SARIF and Markdown output; `--hide` / `--encrypt` support; smoke tests. | Medium: privacy handling must be proven, not assumed. |
| **5** | "Highest-leverage fix" suggestion (which single object breaks the most paths); exceptions / accepted-risk support. | Higher: needs a stable path fingerprint. |
| **6** | Multi-cluster rollup via `core/pkg/fleet`; incremental caching if profiling shows it is needed. | Separate follow-up. |

A user gets a useful CLI at phase 3. Phases 1–2 are pure library work and can be reviewed without any new user-facing surface.

## 7. Reliability bar

Because the output will be read as a statement about attack reachability, the implementation MUST:

1. **Never present "no path found" as "safe".** Print the count of `unknown` and `truncated` results alongside any negative result.
2. **Never drop an unresolved edge silently.** A workload whose pod template cannot be resolved, or a Service with no selector, is reported as skipped, not guessed (the existing `vulnexposure` `skipped` return sets this precedent).
3. **Be deterministic** across runs.
4. **Bound the search.** Enforce a depth limit and a path-count cap; when the cap is hit, set an explicit `truncated` flag rather than returning a partial list as if complete.
5. **Have golden-file tests** covering: a fully resolved path; each engine's "unknown" outcome; `automountServiceAccountToken: false` at both pod and ServiceAccount level **and** no projected `serviceAccountToken` volume (no `runs-as` edge); `automountServiceAccountToken: false` **with** a projected `serviceAccountToken` volume (`runs-as` edge present, since the token is still mounted); a selector-less/headless Service (no backend edge); and each output format. The pod-level value overrides the ServiceAccount-level value.

## 8. Alternatives considered

- **Keep it MCP-only.** Rejected: it excludes CI/PR use and everyone who does not run an MCP client, and it leaves each engine answering one question at a time.
- **Extend `--print-attack-tree`.** Rejected as the primary path: that feature is driven by predefined attack tracks over failed controls, which is a different model from a reachability graph. Converging the two is a reasonable later discussion, but merging them now would couple a stable feature to a new one.
- **A standalone tool or separate repo.** Rejected: the engines live in `kubescape`, and their `FromResources` adapters are already shaped for the scanner's resource map. Extracting them would add an interface boundary for no gain.
- **Runtime/eBPF-based reachability.** Out of scope; it would need node-agent data and is a different proposal.

## 9. Open questions

- **Command name.** `attack-paths` vs `attack-graph` vs another. Needs a maintainer decision.
- **Where should the shared pod-template resolver live?** Moving it out of `cmd/mcpserver` touches existing MCP code. Is a new `core/pkg/attackpath` package acceptable, or should it live next to `networkpolicy`?
- **Vulnerability data dependency.** `vulnexposure` needs `VulnerabilityManifest` objects from an in-cluster scanner. Proposed behaviour: paths still appear without a CVE hop (exposed workload → ServiceAccount → cluster-admin), but score lower than an otherwise identical path with an RCE-class CVE, because there is no known entry vector. The output states clearly when CVE data was unavailable. Is that acceptable?
- **Uncertain edges and `--fail-on-path`.** Should a path containing `Certain=false` edges trip the CI gate by default, or only with an opt-in flag such as `--fail-on-uncertain`? This is especially relevant to LB/NodePort paths, which are `Certain=false` in v1 (see §3.5).
- **External-aware reachability matching.** `networkpolicy.Reaches` currently only matches pod/namespace selectors and has no concept of an external source; building one (so an empty `namespaceSelector` doesn't wrongly admit Internet IPs) is a prerequisite for ever marking an LB/NodePort hop `Certain=true`. Which phase should own this: part of Phase 2 (graph builder), or a dedicated follow-up once the core command ships?
- **Interaction with VEX ingestion.** If [`vex-ingestion`](./vex-ingestion.md) lands, vendor `not_affected` statements would suppress CVEs before they reach this layer. This proposal reads whatever the vulnerability manifests contain, so it should benefit automatically, but the provenance of suppressed findings needs a decision (should a suppressed CVE appear as a de-emphasised hop, or not at all?).
- **Scoring.** Which factors and weights? Proposed starting factors: maximum CVE severity on the path, fix availability, exposure level, hop count, sink type, edge certainty. The formula should be documented and its factors emitted in JSON so the ranking is auditable.
- **Path explosion.** What default depth and path-count caps are reasonable for large clusters, given the known cost concerns in `docs/optimization-plan.md`?
- **Admission-aware pruning.** The repo already contains `vapreconcile` and `mapreconcile`. Should a later phase use them to prune edges an admission policy would block?
- **Exceptions.** Should accepted-risk exceptions reuse the existing exceptions machinery with a stable path fingerprint, and if so, in which phase?

## 10. Prior art and references

- `kubescape/core/pkg/exposure`, `networkpolicy`, `rbacgraph`, `vulnexposure` (package comments document each engine's trust model and non-goals)
- `kubescape/cmd/mcpserver` (current sole consumer of the four engines)
- `kubescape/core/pkg/resourcesprioritization` and `core/pkg/resultshandling/printer/v2/attacktracks.go` (existing control-driven attack-track output)
- `kubescape/core/pkg/fleet` (existing multi-context rollup)
- [`vex-ingestion`](./vex-ingestion.md), [`evidence-of-finding`](./evidence-of-finding.md) in this repository
- PaloAltoNetworks `rbac-police` and `kubiscan`, cited in `rbacgraph`'s own package comment as the source of the RBAC escalation primitives

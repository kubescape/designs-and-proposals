# Native Multi-Cluster Fleet Posture Aggregation for Kubescape

| | |
|---|---|
| Status | Draft – discussion |
| Author | adityashinde1525@gmail.com |
| Date | 2026-07-02 |
| Related work | [kubescape/kubescape#2004: k8s client is a global singleton](https://github.com/kubescape/kubescape/issues/2004) |

## 1. Summary

Kubescape scans exactly one cluster per invocation. There is no first-class way
to scan a set of clusters and reason about them **together**: no type in the
codebase sits above a single `PostureReport`, and the Kubernetes client is a
process-global singleton ([#2004](https://github.com/kubescape/kubescape/issues/2004)),
so even running two scans in one process is unsafe today. Teams that operate
more than one cluster (prod/staging/DR, per-region, per-tenant) fall back to
shell loops over `KUBECONFIG`, the ARMO SaaS platform, or bespoke CI glue that
stitches JSON reports together by hand.

This proposal adds a `kubescape scan fleet` command that scans multiple
kubeconfig contexts **sequentially** through the existing, unmodified
single-cluster scan path, then aggregates the per-cluster `PostureReport`
results into a new `FleetReport` type. `FleetReport` carries a cross-cluster
**control matrix** (control × cluster status grid) and **drift detection** that
flags controls whose result diverges from a chosen baseline cluster. The change
is purely additive: no existing `PostureReport` field, printer, or public API is
modified. Phase 1 is deliberately sequential-only: concurrency is blocked on
#2004 and is explicitly deferred to Phase 2, which makes Phase 1 a small,
self-contained, mergeable foundation and the whole effort a good fit for an LFX
mentorship term.

## 2. Motivation

### The problem

Running one cluster is the exception, not the rule. Almost every real Kubescape
user has at least a production and a non-production cluster, and platform teams
routinely run tens of them across regions and accounts. Kubescape gives each of
those clusters an excellent individual report and then stops. The questions an
operator actually asks are cross-cluster:

- *Which clusters fail C-0016 (`allowPrivilegeEscalation`)?*
- *Staging passes this control but prod fails it: where did we drift?*
- *What is our fleet-wide compliance trend, not just this one cluster's?*

Nothing in the codebase can answer these, and there are two concrete reasons.

**(a) No type exists above `PostureReport`.** The scan entry point
`core/core/scan.go:183`,
`func (ks *Kubescape) Scan(scanInfo *cautils.ScanInfo) (*resultshandling.ResultsHandler, error)`,
returns a handler whose `GetResults()` yields a single
`*reporthandlingv2.PostureReport` (`core/pkg/resultshandling/results.go:74`).
That is the top of the type hierarchy. There is no `FleetReport`, no list of
reports, no aggregation surface. Everything downstream (printers under
`core/pkg/resultshandling/printer/v2/`, the compliance score, `ScanCoverage`)
is single-cluster by construction.

**(b) The Kubernetes client is a global singleton
([#2004](https://github.com/kubescape/kubescape/issues/2004)).** The active
context is selected once, globally, at CLI start:
`cmd/root.go:61` calls `k8sinterface.SetClusterContextName(rootInfo.KubeContext)`
(the flag is defined at `cmd/root.go:92`). From then on the scan pipeline reads a
process-global config: `core/core/scan.go:254`
(`k8sinterface.GetK8sConfig()`), `core/core/initutils.go:31`
(`k8sinterface.NewKubernetesApi()`), plus getters like
`core/cautils/getter/crdcontrolinputs.go:36` and the host-sensor handler. Because
that state is global, you cannot hold two live cluster clients in one process,
and you certainly cannot scan two clusters concurrently. This is exactly why
Phase 1 is sequential: a sequential loop re-points the global to one context,
runs a **complete** scan, and only then moves to the next context, never
holding two clients at once.

### Current workarounds (and why they hurt)

- **`KUBECONFIG`-per-scan shell loops.** `for ctx in ...; do kubescape scan
  --kube-context "$ctx" --format json -o "$ctx.json"; done`, then a `jq` script
  to merge. Every user reinvents the merge, the drift logic, and the exit-code
  policy; none of it is tested or shared, and the per-cluster JSON files have no
  common envelope.
- **ARMO Platform.** The commercial SaaS does aggregate multi-cluster posture,
  but it requires sending data off-cluster to a hosted backend. Air-gapped,
  privacy-sensitive, and "just the OSS CLI" users are excluded. Fleet
  aggregation should not be a SaaS-only capability.
- **Custom CI glue.** Teams wire multi-cluster matrices into GitLab/GitHub
  Actions pipelines with hand-rolled scripts. Fragile, opaque, and duplicated
  across every org.

### Who is affected

Platform/SRE teams running more than one cluster; security teams that must
report fleet-wide compliance for an audit; and anyone running Kubescape in CI
who wants a single pass/fail gate across an environment rather than N
independent ones. All of them today either pay for SaaS or maintain private
glue.

## 3. Design

### 3.1 Overview

```
kubescape scan fleet --contexts prod,staging,dr --baseline staging
        │
cmd/scan/fleet.go  (new subcommand: parse --contexts/--baseline, reuse scan flags)
        │
core/pkg/fleet/orchestrator.go
   Orchestrator.Run(ctx, baseScanInfo):
     for each context (SEQUENTIAL):
        k8sinterface.SetClusterContextName(context)   // re-point the global
        scanInfo := baseScanInfo.clone(context)
        rh, err := scan(scanInfo)                      // EXISTING core.Scan, unmodified
        record ClusterResult{report: rh.GetResults(), coverage, err}
     buildControlMatrix(results)
     detectDrift(results, baseline)
        │
core/pkg/fleet/report.go  → FleetReport
        │
core/pkg/resultshandling/printer/v2/fleetprinter.go  → pretty table / JSON
```

The single-cluster scan path (`core.Scan`, resource collection, OPA evaluation,
`ScanCoverage`, compliance scoring) is called **as-is**. The fleet layer is a
pure orchestrator-plus-aggregator wrapped around it.

### 3.2 New package: `core/pkg/fleet`

A new, self-contained package holds all fleet logic so nothing leaks into the
single-cluster hot path.

```
core/pkg/fleet/
  orchestrator.go     // sequential scan loop, per-cluster error isolation
  report.go           // FleetReport + aggregation (matrix, drift) constructors
  types.go            // FleetReport, ClusterResult, FleetControlMatrix, DriftedControl
  orchestrator_test.go
  report_test.go
```

### 3.3 New types

All additive; none touches `reporthandlingv2.PostureReport`, which is embedded
by value/pointer and never mutated.

```go
// FleetReport is the top-level aggregate: one entry per scanned context plus
// the cross-cluster views derived from them.
type FleetReport struct {
    Metadata      FleetMetadata      `json:"metadata"`
    Clusters      []ClusterResult    `json:"clusters"`
    ControlMatrix FleetControlMatrix `json:"controlMatrix"`
    Drift         []DriftedControl   `json:"drift,omitempty"`
}

type FleetMetadata struct {
    GeneratedAt      time.Time `json:"generatedAt"`
    KubescapeVersion string    `json:"kubescapeVersion"`
    Contexts         []string  `json:"contexts"`
    Baseline         string    `json:"baseline,omitempty"` // clusterID used for drift
}

// ClusterResult wraps one cluster's existing PostureReport with fleet-level
// status. A cluster that could not be scanned still gets a row (Status set,
// Report nil) so it is visible rather than silently dropped.
type ClusterResult struct {
    ClusterID       string                          `json:"clusterID"` // context name in Phase 1
    Context         string                          `json:"context"`
    Status          ClusterScanStatus               `json:"status"`    // scanned | unreachable | error
    Error           string                          `json:"error,omitempty"`
    ComplianceScore float32                          `json:"complianceScore"`
    Coverage        cautils.ScanCoverage            `json:"coverage"`
    Duration        string                          `json:"duration"`
    Report          *reporthandlingv2.PostureReport `json:"report,omitempty"`
}

type ClusterScanStatus string

const (
    ClusterScanned     ClusterScanStatus = "scanned"
    ClusterUnreachable ClusterScanStatus = "unreachable"
    ClusterError       ClusterScanStatus = "error"
)

// FleetControlMatrix is the control × cluster grid: for each control, its
// status in every scanned cluster.
type FleetControlMatrix struct {
    Controls []FleetControlRow `json:"controls"`
}

type FleetControlRow struct {
    ControlID string                       `json:"controlID"`
    Name      string                       `json:"name"`
    Severity  string                       `json:"severity"`
    ByCluster map[string]ControlStatusCell `json:"byCluster"` // clusterID -> cell
}

type ControlStatusCell struct {
    Status          string  `json:"status"` // passed|failed|skipped|irrelevant|notEvaluated
    FailedResources int     `json:"failedResources"`
    ComplianceScore float32 `json:"complianceScore"`
}

// DriftedControl records a control whose status in one or more clusters differs
// from the baseline cluster's status for that control.
type DriftedControl struct {
    ControlID      string            `json:"controlID"`
    Name           string            `json:"name"`
    BaselineStatus string            `json:"baselineStatus"`
    ClusterStatus  map[string]string `json:"clusterStatus"` // clusterID -> divergent status
    Confidence     string            `json:"confidence"`    // "high" | "low" -- low when either cluster's ScanCoverage.Degraded is true
}
```

### 3.4 CLI command

`kubescape scan fleet` is a new subcommand under the existing `scan` command
(`cmd/scan/`), so it inherits the full scan flag set (frameworks, `--format`,
`--severity-threshold`, exceptions, etc.) via the shared `scanInfo`.

New flags:

| Flag | Meaning | Default |
|---|---|---|
| `--contexts` | Comma-separated kubeconfig context names to scan | **required, no default** |
| `--baseline` | Context whose results are the drift reference; if set, it must be in `--contexts` or is automatically added to the scan set | first **explicitly listed** context in `--contexts` |

`--contexts` is required. There is deliberately no "scan every context in the
kubeconfig" default. A host scan deploys a DaemonSet (the host sensor) into each
scanned cluster to collect node-level data, so scanning every context in the
active kubeconfig by default would deploy workloads into every cluster that
kubeconfig can reach, including production or third-party clusters the user never
intended to touch. Requiring an explicit context list makes the blast radius of
a fleet scan something the user states on purpose, not something inferred from
whatever happens to be in the kubeconfig.

```bash
# Scan three named contexts, drift-compare against staging
kubescape scan fleet --contexts prod,staging,dr --baseline staging

# Scan two named contexts with the NSA framework, JSON out
kubescape scan fleet --contexts prod,staging --framework nsa --format json -o fleet.json
```

`--contexts` is validated against the kubeconfig up front; unknown names are
rejected before any scan starts. Reachability is **not** pre-checked: an
unreachable-at-scan-time cluster is handled per-row (§3.6).

**Baseline resolution** (validated up front, before any scan starts):

- If `--baseline` is set, it must either already be present in `--contexts`, or
  it is **automatically added** to the scan set (so drift always has a scanned
  baseline to compare against). A baseline that names a context absent from the
  kubeconfig is rejected.
- If `--baseline` is not set, it defaults to the **first explicitly listed
  context in `--contexts`**, not the kubeconfig's current-context and not
  kubeconfig ordering, so drift is deterministic and independent of kubeconfig
  file order.
- If the resolved baseline cluster's scan fails (`Status != scanned`), the fleet
  report is still produced but **without drift detection**, and a warning is
  surfaced: `baseline cluster <name> could not be scanned -- drift detection
  skipped`. The control matrix and per-cluster summary rows are unaffected.
- The resolved baseline value, whether explicitly set via `--baseline` or
  defaulted to the first context in `--contexts`, is written to
  `FleetMetadata.Baseline` in the output. Downstream consumers read that field to
  identify which context was used as the drift reference, rather than having to
  re-derive it from the flags.

### 3.5 Orchestrator

```go
type Orchestrator struct {
    contexts []string
    baseline string
    // scan is the injection seam for tests; defaults to core.NewKubescape(ctx).Scan
    scan func(*cautils.ScanInfo) (*resultshandling.ResultsHandler, error)
}

func (o *Orchestrator) Run(ctx context.Context, base *cautils.ScanInfo) (*FleetReport, error)
```

`Run` loops over `o.contexts` **strictly sequentially**:

1. `k8sinterface.SetClusterContextName(context)`: re-point the process-global
   client at this context (the same call `cmd/root.go:61` already makes once).
2. Clone `base` into a per-cluster `ScanInfo` carrying that context and a
   distinct in-memory output target (fleet aggregation reads the report object
   directly; it does not shell out to files).
3. Call `o.scan(scanInfo)`: the **unmodified** `core.Scan`.
4. On success: `rh.GetResults()` gives the `*PostureReport`; record a
   `ClusterResult{Status: scanned, ...}` including its `ScanCoverage`.
5. On error: record `ClusterResult{Status: unreachable|error, Error: msg}` and
   **continue**: one bad cluster never aborts the fleet (§3.6).

Only after the loop completes does `Run` build the control matrix and drift
list. When building both, the aggregator **iterates only over `ClusterResult`
entries whose `Status == scanned` and whose `Report != nil`**; entries for failed
clusters (`Status != scanned`, `Report == nil`) are skipped so their nil
`Report.SummaryDetails.Controls` is never dereferenced:

- **Control matrix** (`report.go`): for each scanned cluster, iterate
  `ClusterResult.Report.SummaryDetails.Controls`, indexed by control ID. Failed
  clusters contribute no cells; their column renders as `-`/`unreachable`
  (§3.6).
- **Drift** (`report.go`): compare each control's status per scanned cluster
  against the baseline cluster's cell. Failed clusters are not compared and do
  not count toward the drift set. Each `DriftedControl` carries a `Confidence`
  field: it is set to `"low"` when **either** the baseline cluster or the
  compared cluster has `ScanCoverage.Degraded == true` (e.g. RBAC blocked a
  resource type, so the divergence may be a coverage artifact rather than a real
  drift), and `"high"` otherwise. The fleet printer renders low-confidence drift
  entries with a visual indicator (a `~` marker and a `low-confidence` note) so
  operators can tell a solid divergence from one that needs corroboration.

A failed cluster still appears in the fleet report's per-cluster **summary row**
with its `Status` and `Error` message, so operators see the gap; it simply does
not contribute to the control matrix or the drift calculation. Sequential
execution is a hard invariant in Phase 1, enforced by the loop and asserted in
tests; there is no goroutine, no `errgroup`, no shared mutable cross-cluster
state.

### 3.6 Per-cluster error isolation

A cluster can be unreachable (VPN down, expired credentials, API-server
timeout), or its scan can fail partway. In all cases the orchestrator captures
the error into that `ClusterResult`, sets `Status` to `unreachable` or `error`,
leaves `Report` nil, and proceeds to the next context. The fleet scan's own exit
is governed by policy (e.g. any cluster failing the compliance gate, or any
unreachable cluster), but the **scan itself does not abort**. In the matrix and
drift views, a degraded cluster's column shows `-`/`unreachable` rather than
being dropped, so operators see the gap instead of a falsely-complete fleet.

### 3.7 Integration points with existing code

| Area | File(s) | Change |
|---|---|---|
| CLI | `cmd/scan/fleet.go` (new); small registration in `cmd/scan/scan.go` | Add the `fleet` subcommand; parse `--contexts`/`--baseline`; build `Orchestrator`. |
| Scan core | `core/core/scan.go` | **None.** `Scan` is called unmodified per cluster. |
| Context switch | `cmd/root.go` / `k8sinterface.SetClusterContextName` | Reused as-is; the orchestrator calls the same setter per iteration. |
| Aggregate types | `core/pkg/fleet/` (new package) | All new. |
| Coverage | `core/cautils/scancoverage.go` | Reused read-only; each `ClusterResult` carries the cluster's `ScanCoverage`/`Degraded` so comparability is visible (§4). |
| Printing | `core/pkg/resultshandling/printer/v2/fleetprinter.go` (new) | New printer for the matrix + drift; JSON is `json.Marshal(FleetReport)`. Existing printers untouched. |

### 3.8 Example terminal output

```
Kubescape Fleet Posture Report - 3 contexts (baseline: staging)

CLUSTER     STATUS        COMPLIANCE   COVERAGE   FAILED CONTROLS
prod        scanned            72%        100%    18
staging     scanned            81%        100%    11
dr          unreachable          -          -     -   (context deadline exceeded)

Control matrix (● pass  ✗ fail  ○ skipped  - n/a):

CONTROL   NAME                              SEV     prod  staging  dr
C-0016    Allow privilege escalation        High     ✗      ●      -
C-0017    Immutable container filesystem    Med      ✗      ✗      -
C-0038    Host PID/IPC privileges           High     ●      ●      -
...

Drift vs. baseline "staging" (2 controls diverge):
  C-0016    staging=passed   prod=failed
  C-0260  ~ staging=passed   prod=failed   (low-confidence: prod coverage degraded)

2 of 3 clusters scanned; 1 unreachable. Fleet gate: FAIL (prod < threshold).
```

## 4. Open questions for maintainer input

1. **Schema location.** Should `FleetReport` and its sub-types live in this repo
   (`core/pkg/fleet`) or in `opa-utils` (`reporthandling/`) next to
   `PostureReport`? In-repo is faster to iterate and keeps Phase 1 self-contained;
   `opa-utils` is the right long-term home if the operator or other consumers
   ever need to produce/consume `FleetReport`. Proposed: start in-repo, promote
   to `opa-utils` in Phase 2 if a second consumer appears.
2. **Cluster identity.** Phase 1 keys clusters by **context name**: simple,
   local, no API call. But context names are arbitrary and not stable across
   machines/kubeconfigs, which weakens drift/history correlation. Should we
   instead (or additionally) key on a cluster UID (e.g. the `kube-system`
   namespace UID, which Kubescape can already read)? Proposed: context name in
   Phase 1, add an optional stable cluster UID in Phase 2 for history.
3. **Coverage comparability gate.** Drift between two clusters is only meaningful
   if both were scanned with comparable coverage. If `prod` has
   `ScanCoverage.Degraded = true` (e.g. RBAC blocked a resource type), a
   "drift" on a control that depends on that resource may be an artifact, not a
   real divergence. Should drift detection **suppress or flag** comparisons where
   either side is degraded? Proposed: still report the drift but annotate it as
   `low-confidence` when either cell's coverage is degraded.
4. **Scope vs. ARMO boundary.** How far should the OSS fleet view go before it
   overlaps the commercial platform? This proposal draws the line at
   **local, stateless, single-run aggregation** (no history, no storage, no
   SaaS). Fleet history/trends and an operator-side fleet CRD are called out as
   Phase 2+ and may be where the OSS/commercial boundary should sit. Maintainer
   guidance wanted on where to stop.
5. **Cluster state isolation between sequential scans. RESOLVED.** The earlier
   open question was whether looping the existing single-cluster scan path over
   several contexts in one process is enough to produce correct per-cluster
   reports. It is not. Sequential invocation alone leaves process-global state
   (the active context, the cached Kubernetes config, and the resource map) set
   from the first cluster, so later scans read stale state and can describe the
   wrong cluster. The resolution is explicit per-cluster state reset, not just
   sequential invocation: an enter/leave helper resets that state around each
   cluster scan. This is the chosen approach and is specified as PR 1 in the
   Prerequisites section (§6).

## 5. Alternatives considered

- **A `--contexts` flag on the existing `scan` command instead of a `fleet`
  subcommand.** Rejected for Phase 1: `scan` returns a single
  `ResultsHandler`/`PostureReport`, and every printer, exit-code path, and
  `-o` file assumes one cluster. Overloading it to sometimes mean "many
  clusters, different return type" would either break that contract or bolt a
  second return shape onto a hot path. A dedicated subcommand keeps the
  single-cluster surface pristine and gives the aggregate its own printer and
  exit policy. (This mirrors the framing in the merged CLI-cluster-operations
  proposal, which fenced fleet orchestration off as a non-goal precisely because
  it needs its own surface.)
- **Parallel scanning in Phase 1.** Rejected: it is *unsafe today*. The k8s
  client is a process-global singleton (#2004); two concurrent scans on
  different contexts would race on the same global config/clientset and produce
  cross-contaminated results. Concurrency is not a performance nicety we skipped;
  it is blocked until #2004 is resolved, and Phase 2 is explicitly gated on it.
- **An external aggregation script (the status quo, blessed).** Rejected as the
  *product* answer: shipping a `jq` recipe leaves drift logic, the report
  envelope, coverage-comparability handling, and the fleet exit gate untested and
  duplicated in every user's CI. The value of this feature is precisely that
  those become first-class, tested, and shared.

## 6. Prerequisites

Two isolation fixes must land before the fleet command is built. Both address the
same underlying fact: a Kubescape process carries mutable global state that is set
once at CLI start and never reset. The single-cluster CLI never noticed, because
it scans one cluster and exits. A sequential fleet loop reuses that state across
clusters in one process, so without these fixes the second and later scans read
state left behind by the first and can report on the wrong cluster.

### 6.1 PR 1: client config and resource map isolation (this proposal's author)

After a single-cluster scan, three pieces of process-global state remain set to
the first cluster:

- The active context name, set by `k8sinterface.SetClusterContextName`
  (`cmd/root.go:61`).
- The cached Kubernetes client config returned by `k8sinterface.GetK8sConfig`
  (`core/core/scan.go:254`), which memoizes the config for the selected context.
- The resource map built by `InitializeMapResources`, which maps resources to API
  groups off the connected cluster's discovery API.

A second scan in the same process inherits all three. Re-pointing the context
with `SetClusterContextName` alone is not enough: the cached config and the
resource map still describe the first cluster, so the second scan can silently
collect and report on the wrong one.

This PR adds an enter/leave helper that resets those three pieces of state around
each cluster scan. On enter it sets the target context and clears the cached
config and the resource map, so the scan rebuilds them against the correct
cluster. On leave it returns the process to a clean baseline, so the next
iteration starts from a known state rather than from whatever the previous scan
left behind.

The leave/reset operation is registered with `defer` before the scan runs, not
called after it returns. This matters because a scan can fail partway: if the
reset ran only on the success path, a scan error would leave the cached config,
the resource map, and the active context still attached to the failed cluster and
poison the next iteration. Deferring the reset guarantees it runs whether the scan
succeeds, returns an error, or panics.

After the full fleet run completes, the helper also restores the original
kubeconfig context that was active before the fleet command ran, so the fleet
command does not leave the process (or a reused client) pointed at the last
scanned cluster.

The PR includes a two-kind-cluster integration test: it scans two kind clusters
in a row within one process and asserts that the second report describes the
second cluster (its nodes, namespaces, or cluster identity), not the first. This
test fails on `main` today, where the second report still describes the first
cluster. That failure proves the isolation gap exists independently of the fleet
feature: it is a latent bug in running two scans in one process, and the fleet
command is simply the first caller that does so. The fleet command cannot produce
correct per-cluster reports until this test passes.

### 6.2 PR 2: PolicyHandler singleton isolation (issue #2005)

`PolicyHandler` holds its own process-level state that is likewise not reset
between scans. This is tracked separately as
[issue #2005](https://github.com/kubescape/kubescape/issues/2005) and is **out of
scope for this proposal**. It is named here only so the full isolation surface is
visible: the fleet command depends on it landing, but the work and its review
belong to that issue, not this one.

## 7. Implementation plan

### Phase 1: foundational, mergeable slice (this proposal)

| Milestone | Deliverable |
|---|---|
| P1.1 | `core/pkg/fleet/types.go`: `FleetReport`, `ClusterResult`, `FleetControlMatrix`, `DriftedControl` + JSON tags. |
| P1.2 | `orchestrator.go`: sequential `Run` loop with per-cluster error isolation and the `scan` injection seam. |
| P1.3 | `report.go`: control-matrix and drift constructors from `[]ClusterResult`. |
| P1.4 | `cmd/scan/fleet.go`: subcommand, `--contexts`/`--baseline`, kubeconfig validation, reuse of scan flags. |
| P1.5 | `printer/v2/fleetprinter.go`: pretty matrix/drift table + JSON marshal. |
| P1.6 | Integration tests: multi-context orchestration with a **stubbed `scan`** (no live cluster), one-cluster-unreachable degradation, drift detection, matrix construction, JSON golden file. |

Phase 1 is intentionally the smallest thing that is independently useful:
sequential scan + aggregate + print, no new dependencies, no SaaS, no schema
changes elsewhere.

### Phase 2: explicitly deferred

- **Concurrency**, gated on [#2004](https://github.com/kubescape/kubescape/issues/2004).
  Once the k8s client is no longer a global singleton, replace the sequential
  loop with a bounded worker pool. No `FleetReport` schema change expected.
- **Operator-side fleet CRD** so an in-cluster operator can publish fleet posture.
- **Fleet history / trends** (drift over time), which requires a storage/identity
  story (open question 2) and is where the OSS/commercial boundary likely sits.

These are named here only to scope them **out** of Phase 1.

## 8. Estimated scope

- **New files:** ~6 - `core/pkg/fleet/{types,orchestrator,report}.go`,
  `cmd/scan/fleet.go`, `core/pkg/resultshandling/printer/v2/fleetprinter.go`,
  plus tests (`orchestrator_test.go`, `report_test.go`, printer golden test).
- **Modified files:** ~1 - `cmd/scan/scan.go` (register the subcommand). No
  changes to `core/core/scan.go`, `PostureReport`, or any existing printer.
- **New types:** 8 (`FleetReport`, `FleetMetadata`, `ClusterResult`,
  `ClusterScanStatus`, `FleetControlMatrix`, `FleetControlRow`,
  `ControlStatusCell`, `DriftedControl`).
- **Test approach:** the orchestrator's `scan` seam lets tests inject canned
  `PostureReport`s, so the entire matrix/drift/degradation logic is unit-tested
  with **no live cluster**. A single opt-in integration test can exercise two
  real contexts (e.g. two kind clusters) behind an env guard.
- **Rough size:** ~700-1000 lines including tests. Additive, low blast radius:
  if the `fleet` package were deleted, the single-cluster CLI is byte-for-byte
  unchanged.
- **Mentorship fit:** the Phase 1 milestones (P1.1-P1.6) are cleanly separable,
  each independently reviewable, and the whole slice is self-contained with a
  clear "done." That structure, plus the honest Phase 2 deferral gated on a
  named upstream issue, makes this a good **LFX mentorship term** project.

## 9. Impact

**What becomes possible**

- A single `kubescape scan fleet` run answers "which clusters fail control X"
  and "where did prod drift from staging" without any external tooling.
- One fleet-wide pass/fail gate for CI instead of N independent per-cluster gates
  stitched together by hand.
- A shareable, tested `FleetReport` JSON envelope that downstream tools
  (dashboards, GitOps checks) can consume with a stable shape.

**What workarounds are eliminated**

- Per-org `KUBECONFIG` shell loops + `jq` merge scripts.
- Reliance on the ARMO SaaS purely to see multi-cluster posture (OSS/air-gapped
  users get a local equivalent for the aggregation view).

**What downstream work this unblocks**

- Gives concurrency (Phase 2) a concrete, already-shipped consumer, adding
  weight to resolving [#2004](https://github.com/kubescape/kubescape/issues/2004).
- Establishes the first type above `PostureReport`, which an operator fleet CRD
  and fleet-history features can build on without re-litigating the schema.

## 10. References

- [kubescape/kubescape#2004: k8s client is a global singleton](https://github.com/kubescape/kubescape/issues/2004) (the concurrency blocker; motivates sequential-only Phase 1)
- Scan entry point reused unmodified: `core/core/scan.go:183` - `func (ks *Kubescape) Scan(scanInfo *cautils.ScanInfo) (*resultshandling.ResultsHandler, error)`
- Single-cluster report accessor: `core/pkg/resultshandling/results.go:74` - `GetResults() *reporthandlingv2.PostureReport`
- Global context selection reused per iteration: `cmd/root.go:61` - `k8sinterface.SetClusterContextName(...)`; flag at `cmd/root.go:92`
- Process-global client reads (why concurrency is unsafe): `core/core/scan.go:254`, `core/core/initutils.go:31`, `core/cautils/getter/crdcontrolinputs.go:36`
- Coverage type reused read-only: `core/cautils/scancoverage.go` - `ScanCoverage` / `Degraded` / `BuildScanCoverage`
- Existing printers left untouched: `core/pkg/resultshandling/printer/v2/`
- [Kubescape getting-started docs](https://kubescape.io/docs/) (CLI scan usage the fleet command layers on top of)
- Prior-art framing for a CLI feature proposal in this repo: [`proposals/cli-cluster-operations.md`](https://github.com/kubescape/designs-and-proposals/blob/main/proposals/cli-cluster-operations.md) (which lists cross-cluster/fleet orchestration as an explicit non-goal - this proposal picks up exactly that thread)

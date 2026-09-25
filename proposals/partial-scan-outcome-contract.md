# Proposal: Explicit Completion and Partial-Result Contract for Posture Scans

**Status:** Draft  
**Scope:** Kubescape core scan API, OPA processors, CLI, HTTP handlers, and report submission  
**Author:** @ye11oc4t  
**Related work:** [kubescape/kubescape#3242](https://github.com/kubescape/kubescape/pull/3242), [kubescape/kubescape#3243](https://github.com/kubescape/kubescape/pull/3243), [kubescape/kubescape#1987](https://github.com/kubescape/kubescape/issues/1987), [kubescape/kubescape#1992](https://github.com/kubescape/kubescape/pull/1992), [kubescape/kubescape#2207](https://github.com/kubescape/kubescape/issues/2207), [kubescape/kubescape#2419](https://github.com/kubescape/kubescape/issues/2419), [kubescape/kubescape#2813](https://github.com/kubescape/kubescape/issues/2813), [kubescape/kubescape#2814](https://github.com/kubescape/kubescape/issues/2814), [kubescape/kubescape#2815](https://github.com/kubescape/kubescape/pull/2815), [kubescape/kubescape#2960](https://github.com/kubescape/kubescape/issues/2960), [kubescape/kubescape#2965](https://github.com/kubescape/kubescape/pull/2965), [kubescape/kubescape#2010](https://github.com/kubescape/kubescape/issues/2010)

## 1. Summary

Kubescape does not currently have an explicit contract for a posture scan that produced useful evaluation results but did not complete successfully.

The ambiguity is visible at the OPA boundary:

- the eager processor finalizes accumulated results before returning an evaluation error;
- the streaming processor can return after one or more scopes without running final coverage, summary, score, and resource-merge steps;
- `Kubescape.Scan` returns before its normal `ResultsHandler.SetData` point;
- built-in callers treat every `Scan` error as fatal and skip `HandleResults`; and
- `ResultsHandler.SetScanError` supports “print available output, then fail” for combined image scans, but does not expose a typed programmatic posture outcome.

A one-line change that always returns or prints the current `scanData` is unsafe. Some failed sessions contain coherent, finalized partial results. Others contain mutable processor state whose zero-value or stale derived fields can look complete when serialized.

This proposal introduces an explicit scan outcome with three completion states:

- **complete**: every required stage completed;
- **partial**: trustworthy results exist, finalization ran under defined partial semantics, and incompleteness is represented in the artifact; and
- **failed**: no trustworthy posture result exists and no posture artifact should be serialized.

The existing `Scan` method remains a compatibility wrapper. New callers receive a typed outcome without having to invoke a printer or infer completion from `(handler, error)`, coverage, or zero-value summaries.

## 2. Motivation

### 2.1 Useful results and successful completion are different facts

An evaluator can successfully process some resources or scopes before another rule, scope, producer, or downstream stage fails. Discarding all earlier results loses useful posture information. Reporting the accumulated state as a successful scan is worse: users and downstream systems may treat incomplete coverage and provisional scores as authoritative.

Kubescape therefore needs to represent two independent facts:

1. whether trustworthy posture results exist; and
2. whether the scan completed every required stage.

An `error` alone describes neither fact. A non-nil handler alone does not establish that its derived fields are internally consistent.

### 2.2 Eager and streaming processors expose different failure artifacts

Using one Pod and one syntactically invalid Rego rule on current master produces the following behavior:

| Path | Processing error | `ResourcesResult` | Control summary | Resource summary | Coverage |
|---|---:|---|---|---|---|
| Eager `ProcessRulesListener` | yes | one skipped result | skipped | one skipped | 100%, `degraded=false` |
| `ProcessWithStreaming` | yes | one skipped result | empty status | all counters zero | zero-value totals, `degraded=false` |
| Public `Kubescape.Scan` | yes | session not attached | unavailable | unavailable | unavailable |

The eager path performs post-processing after `Process` returns: it rebuilds coverage, aggregates resource results, marks controls skipped, calculates scores, and reweights them before returning the processing error.

The streaming path returns directly from several pre-finalization points, including:

- malformed or missing resident-batch handoff;
- resident or namespace evaluation error;
- producer error after earlier batches succeeded;
- cancellation;
- duplicate resident batch; and
- whole-cluster-control error.

At those points, earlier scopes may be valid, the current scope may contain skipped results, and later scopes may never have arrived. The current API does not distinguish these states.

### 2.3 Coverage does not encode completion

The eager example reports 100% coverage even though evaluation returned an error. This is not necessarily a coverage bug: `BuildScanCoverage` intentionally filters resource-keyed evaluation skips out of Kubernetes GVR coverage.

Coverage describes a particular dimension of collection and evaluation. It is not a scan-completion signal. A partial artifact must carry completion metadata independently of coverage and policy degradation fields.

### 2.4 Existing precedents solve only caller-specific cases

Combined image scanning already uses `ResultsHandler.SetScanError` to print available image results and then return failure. That establishes a useful CLI behavior—available output and a non-zero exit are compatible—but it does not define posture finalization or expose a typed library contract.

Earlier OPA error work established fail-closed evaluation and preservation of skipped resources. Reviews also identified caller-boundary result loss. None of that defines a cross-caller contract for eager and streaming posture scans.

## 3. Goals and non-goals

### 3.1 Goals

- Distinguish complete, finalized-partial, and failed-without-results outcomes through a typed API.
- Preserve the primary error, including cancellation, deadline, and evaluator identity through `errors.Is` and `errors.As`.
- Define when accumulated processor state is safe to finalize and serialize.
- Make finalization common and idempotent across eager and streaming paths.
- Ensure serialized partial artifacts identify themselves deterministically as partial.
- Mark final versus provisional coverage and scores explicitly.
- Record completed and unprocessed work when that information is known.
- Keep CLI, HTTP, library, and reporter behavior consistent with the same outcome.
- Preserve exactly-once writer closure on success, partial failure, and fatal failure.

### 3.2 Non-goals

- Ignoring invalid policy rules.
- Treating skipped or unevaluated resources as passing.
- Changing collection-memory bounds or resource partitioning.
- Automatically retrying invalid policies or failed external calls.
- Redefining Kubernetes GVR coverage to include every processing failure.
- Uploading partial results through the current complete-report protocol.
- Guaranteeing a partial artifact for every error.

## 4. Terminology and invariants

### 4.1 Completion states

**Complete** means all stages required to produce the posture result for the selected scan mode ran successfully. Controls may legitimately be skipped or resources may be excluded; those modeled states do not make the scan partial. Rendering, persisting, transporting, and submitting the resulting artifact happen after scan completion and do not change this state.

**Partial** means at least one trustworthy evaluation result exists, the processor reached a legal partial-finalization point, derived fields were rebuilt under defined semantics, and the artifact records incomplete work. The primary scan error remains non-nil.

**Failed** means no trustworthy posture result exists, or accumulated state cannot legally be finalized. The result handler is absent from the public outcome and no posture report is serialized.

### 4.2 Required invariants

1. `complete` implies `Err == nil`, where `Err` is exclusively the scan-production error returned by `ScanWithOutcome`.
2. `partial` implies `Results != nil` and `Err != nil`.
3. `failed` implies `Results == nil` and `Err != nil`.
4. Only `complete` and `partial` may reach result rendering.
5. Only `complete` may reach the existing report-submission path.
6. Every exposed `partial` result has passed the common finalizer.
7. Finalization never changes the primary error.
8. Calling the finalizer more than once produces the same observable report.
9. Zero-value summaries are never presented as final merely because serialization succeeded.
10. Render, persistence, transport, and submission errors are represented as artifact-delivery outcomes; they never rewrite `ScanCompletion` or `ScanOutcome.Err`.

## 5. Proposed API

### 5.1 Typed outcome

One backwards-compatible API shape is:

```go
type ScanCompletion string

const (
    ScanComplete ScanCompletion = "complete"
    ScanPartial  ScanCompletion = "partial"
    ScanFailed   ScanCompletion = "failed"
)

type ScanStage string

const (
    ScanStageInitialization      ScanStage = "initialization"
    ScanStageResourceCollection  ScanStage = "resource-collection"
    ScanStagePolicyEvaluation    ScanStage = "policy-evaluation"
    ScanStageFinalization        ScanStage = "finalization"
    ScanStageEnrichment          ScanStage = "enrichment"
)

type ScanOutcome struct {
    Results     *resultshandling.ResultsHandler
    Completion  ScanCompletion
    FailedStage ScanStage
    Err         error
}

func (ks *Kubescape) ScanWithOutcome(
    scanInfo *cautils.ScanInfo,
    policyIdentifiers []cautils.PolicyIdentifier,
) ScanOutcome
```

`FailedStage` is empty for `complete` and identifies the primary scan-production failure for `partial` and `failed`. Output and submission are intentionally not `ScanStage` values because they operate on an already-produced result.

The exact package placement and scan-stage granularity are open design questions. The normative requirement is that completion be inspectable without invoking a printer and without parsing an error string.

### 5.2 Compatibility wrapper

The existing method remains source-compatible:

```go
func (ks *Kubescape) Scan(
    scanInfo *cautils.ScanInfo,
    policyIdentifiers []cautils.PolicyIdentifier,
) error {
    outcome := ks.ScanWithOutcome(scanInfo, policyIdentifiers)
    return outcome.Err
}
```

Existing callers retain their current fail-on-error behavior until migrated. The compatibility wrapper must not silently print, submit, or otherwise consume a partial result.

### 5.3 Error identity

`ScanOutcome.Err` contains the original primary error or an error that unwraps to it. In particular:

- `errors.Is(err, context.Canceled)` remains true;
- `errors.Is(err, context.DeadlineExceeded)` remains true; and
- `errors.As` continues to identify typed evaluator and producer errors.

A typed `PartialScanError` may be provided as a convenience, but it is not a substitute for `ScanOutcome.Completion`. Callers must not need to extract a result handler from an error.

### 5.4 Artifact-delivery outcome

Artifact delivery is orthogonal to scan completion. A shared caller-level representation may use the following shape:

```go
type ArtifactDeliveryStatus string

const (
    DeliveryNotAttempted ArtifactDeliveryStatus = "not-attempted"
    DeliverySucceeded    ArtifactDeliveryStatus = "succeeded"
    DeliveryFailed       ArtifactDeliveryStatus = "failed"
    DeliverySkipped      ArtifactDeliveryStatus = "skipped"
)

type ArtifactDeliveryStage string

const (
    DeliveryStageRender  ArtifactDeliveryStage = "render"
    DeliveryStagePersist ArtifactDeliveryStage = "persist"
    DeliveryStageRespond ArtifactDeliveryStage = "respond"
    DeliveryStageSubmit  ArtifactDeliveryStage = "submit"
)

type ArtifactDeliveryOutcome struct {
    Stage  ArtifactDeliveryStage
    Status ArtifactDeliveryStatus
    Err    error
}
```

This type is not returned by `ScanWithOutcome`: the scan API cannot know which delivery operations a CLI, HTTP handler, or embedding library will attempt. Callers retain the `ScanOutcome` and record each attempted or deliberately skipped delivery operation separately.

The combinations are unambiguous:

| Situation | Scan outcome | Delivery outcome |
|---|---|---|
| Complete scan, writer succeeds | `complete`, `Err=nil` | `render/succeeded` |
| Complete scan, writer fails | `complete`, `Err=nil` | `render/failed`, writer error |
| Partial scan, partial artifact is written | `partial`, evaluator error | `render/succeeded` |
| Partial scan, writer also fails | `partial`, evaluator error | `render/failed`, writer error |
| Complete scan, upload fails | `complete`, `Err=nil` | `submit/failed`, reporter error |
| Partial scan under the current reporter protocol | `partial`, scan error | `submit/skipped`, no delivery error |

Threshold or policy-gate rejection is another caller-level operation result, not a scan-completion or delivery state. A caller may aggregate these errors for its return value, for example with `errors.Join`, but must keep their typed sources separately inspectable.

## 6. Processing and finalization model

### 6.1 Common finalizer

Eager and streaming processors should call one common finalization operation that:

1. freezes or snapshots accepted evaluation state;
2. rebuilds scan coverage from the work known to the processor;
3. aggregates per-resource results;
4. marks controls and resources that cannot be evaluated under the partial state;
5. calculates summaries;
6. calculates and reweights scores only when their required inputs are known; and
7. records whether derived values are final or provisional.

The finalizer must be idempotent. Repeated calls must not duplicate resources, double-count summaries, reweight scores twice, or change completion metadata.

### 6.2 Legal partial-finalization points

Finalization is legal only when the processor can identify a stable set of accepted work. At minimum, streaming processing must track:

- whether the resident batch was accepted;
- namespaces or other scopes accepted and completed;
- the current scope, if accepted but incomplete;
- scopes expected but not received, when knowable;
- producer completion or failure; and
- whether whole-cluster controls started and completed.

Examples:

| Failure point | Result |
|---|---|
| Policy load fails before evaluation state exists | failed |
| Producer fails before any batch is accepted | failed |
| Malformed initial resident handoff with no accepted state | failed |
| Namespace evaluation fails after resident or earlier namespaces completed | partial, if accepted state can be frozen and finalized |
| Producer fails after completed scopes | partial |
| Cancellation after completed scopes | partial, if finalization does not require blocked or canceled work |
| Finalizer itself cannot produce internally consistent summaries | failed |

The table is a contract boundary, not an instruction to recover from every error. When legality cannot be established, the safe outcome is `failed`.

### 6.3 Eager processor

The eager processor already performs most required finalization after evaluation failure. It should use the common finalizer, attach explicit partial metadata, and return the original evaluation error.

Existing coverage behavior may remain unchanged, but completion metadata must prevent a 100% GVR coverage value from implying successful scan completion.

### 6.4 Streaming processor

The streaming processor must not expose its mutable accumulator directly. On an eligible error after accepted work:

1. cancel and join the producer according to the existing lifecycle contract;
2. stop accepting new batches;
3. snapshot completed/accepted scopes;
4. invoke the common finalizer once;
5. attach partial metadata; and
6. return the finalized result with the original error.

If producer shutdown or state freezing fails in a way that makes the snapshot untrustworthy, the outcome is `failed`.

## 7. Report representation

### 7.1 Additive completion metadata

Machine-readable posture output gains an additive metadata block. A representative shape is:

```json
{
  "scanCompletion": {
    "status": "partial",
    "failedStage": "policy-evaluation",
    "reasons": [
      {
        "code": "OPA_EVALUATION_ERROR",
        "message": "policy evaluation failed"
      }
    ],
    "completedScopes": ["resident", "namespace/default"],
    "unprocessedScopes": ["namespace/payments"],
    "scores": "provisional",
    "coverage": "provisional",
    "skippedScanStages": ["enrichment"]
  }
}
```

Field names may follow the owning report package's conventions. The semantics are normative:

- every serialized partial posture report contains `scanCompletion.status: partial`;
- partial reports identify the failing scan stage;
- reason codes are stable and messages are sanitized;
- completed and unprocessed work are included only when known;
- score and coverage finality are explicit; and
- skipped scan-production stages are distinguished from stages that ran and found nothing.

For v1 wire compatibility, newly produced **complete** posture reports omit the `scanCompletion` block. A **failed** scan produces no posture report. Therefore only a partial artifact introduces this additive block in v1, and any artifact carrying it is self-identifying.

Absence of the block in an artifact from an unknown or legacy producer means “completion was not explicitly encoded”; it must not be treated as universal proof of completion without producer/version or operation context. The programmatic `ScanOutcome.Completion` is always explicit for new API callers. A future report-schema version may encode `status: complete`, but that is not part of this v1 contract.

### 7.2 Determinism and sanitization

JSON, YAML, and programmatic representations must be deterministic for the same accepted work and failure. Scope lists, reason lists, and skipped-stage lists use stable ordering.

Failure metadata must not include raw Rego source, resource contents, credentials, network endpoints with secrets, or unbounded external error text. The original error remains available to the in-process caller; serialized metadata uses stable codes and bounded sanitized messages.

### 7.3 Do not overload collection-gap fields

OPA evaluation and processor failures must not be represented as failed Kubernetes GVR pulls. `FailedGVRPulls` and policy-degradation fields retain their existing meanings.

Completion metadata may reference those existing gaps, but it is a separate concept. Collection coverage, policy-input degradation, evaluation completion, and artifact delivery are different dimensions.

### 7.4 Scores and coverage

Partial finalization may calculate scores or coverage for completed work if consumers need those values, but it must label them provisional unless every input required for the complete-scan value is known.

Threshold evaluation must not replace the primary scan error. If a partial result also violates a configured threshold, the scan still fails primarily because processing did not complete. Threshold details may be attached as secondary diagnostics.

## 8. Caller behavior

### 8.1 CLI

- `complete`: render normally and use existing threshold exit behavior.
- `partial`: render the finalized partial result, clearly label it partial, and exit non-zero with the primary processing error.
- `failed`: print the error only; do not render a posture report.

Rendering records a separate delivery outcome. A render failure after `complete` leaves the scan complete and exits non-zero for the delivery error. If rendering a `partial` result also fails, the CLI reports the scan error as primary and the delivery error as secondary while preserving both for `errors.Is`/`errors.As`. A threshold failure ranks after scan and delivery errors and never replaces either. Writer closure remains exactly once.

### 8.2 HTTP

HTTP callers preserve a canonical partial artifact together with typed scan and delivery status. The transport mapping must make it impossible to confuse “the scan completed,” “a partial scan was finalized,” and “artifact persistence or response delivery failed.”

The precise status-code and response-envelope choice belongs to the HTTP API owner, but the response must expose:

- completion state;
- primary failure stage and stable reason code; and
- the artifact or a stable reference to it when completion is partial; and
- a separate delivery failure when persistence or response writing did not succeed.

### 8.3 Library

Library callers use `ScanWithOutcome` and can branch on `Completion`. No printer invocation, error-string parsing, or zero-value-field inspection is required.

### 8.4 Reporter and submission

The current reporter protocol accepts complete scans. Until a future protocol explicitly represents partial reports, a partial artifact must not be uploaded as a complete report. Skipping submission for this reason records `submit/skipped`, not a scan error.

Local serialization and remote submission are separate decisions: the CLI may write a partial JSON artifact while the reporter skips submission. Conversely, a submission failure after a complete scan records `submit/failed` while the underlying scan remains `complete`.

## 9. Lifecycle and ownership

### 9.1 Writer closure

The component that creates an output writer owns its closure. Exactly-once closure must hold for:

- pre-result fatal failure;
- partial finalization;
- partial rendering failure;
- successful rendering;
- threshold failure after rendering; and
- cancellation and deadlines.

Finalization does not close output writers. Rendering does not close processor inputs. Closing or flushing a writer may change the render/persist delivery outcome but never the scan outcome. These lifecycle boundaries should be independently testable.

### 9.2 Producer cancellation

Streaming failures must preserve the cancellation and join guarantees established by the producer-leak fixes. Partial finalization must not reintroduce a producer goroutine leak or block the processor while the producer attempts to send another batch.

## 10. Compatibility and rollout

### Phase 0 — preserve an already-finalized eager session

The bounded caller-boundary fixes in #3242 and #3243 may preserve an eager session that is already finalized while retaining the processing error. They do not automatically print or submit partial output and do not define the final public contract.

### Phase 1 — centralize idempotent processor finalization

- Extract common coverage, aggregation, skip, summary, and score finalization.
- Define legal and illegal partial-finalization points.
- Make repeated finalization harmless.
- Track resident acceptance and completed namespace scopes in streaming mode.
- Preserve cancellation and producer shutdown behavior.

### Phase 2 — add typed completion metadata

- Add `ScanOutcome` or the agreed equivalent.
- Add the additive report representation.
- Mark scores and coverage final or provisional.
- Mark unprocessed work without converting failures into passes.
- Preserve existing `Scan` as a compatibility wrapper.
- Require the completion block on partial artifacts and omit it on complete v1 artifacts.

### Phase 3 — integrate callers

- CLI renders finalized partial results and exits non-zero.
- HTTP preserves a canonical partial artifact and typed failure status.
- Library callers can distinguish partial from fatal without printing.
- Reporter rejects or skips partial artifacts under the complete-report protocol.
- Callers record render, persistence, response, and submission outcomes separately from scan completion.

### Phase 4 — parity and failure injection

Exercise eager and streaming paths with:

- invalid Rego;
- cancellation and deadlines;
- producer failure before resident input;
- producer failure after resident or namespace success;
- namespace failure after earlier namespace success;
- duplicate or malformed resident batch;
- whole-cluster-control failure;
- finalizer failure; and
- output failure.

Run lifecycle and failure-injection tests under `-race`.

## 11. Alternatives considered

### 11.1 Always attach or print the current scan data

Rejected. Mutable streaming state can contain zero-value summaries and incomplete merges that serialize successfully and appear authoritative.

### 11.2 Return only a typed `PartialScanError`

Potentially useful as a convenience, but insufficient as the primary contract. It encourages hiding result ownership inside an error and does not naturally represent complete and failed outcomes with the same invariants.

### 11.3 Infer completion from handler presence

Rejected. A handler may exist before finalization, and the current public scan path may fail before attaching an otherwise finalized session.

### 11.4 Infer completion from coverage or degraded policy state

Rejected. Coverage intentionally models collection/evaluation dimensions that do not include every processing failure. Policy degradation has a different meaning. The eager invalid-Rego example demonstrates that coverage can be 100% while the scan is incomplete.

### 11.5 Treat all processing errors as fatal and discard results

Safe but unnecessarily lossy. It prevents CLI users and incident responders from using already-completed posture results and does not align with the established “render available output, then fail” behavior used by combined image scans.

### 11.6 Submit partial artifacts with the existing reporter

Rejected. A backend that does not understand completion metadata may index or compare the artifact as a complete scan. Submission requires an explicitly versioned protocol change outside the initial scope.

## 12. Testing strategy

### 12.1 Outcome matrix

Tests assert all three invariants:

| Scenario | Completion | Results | Error |
|---|---|---:|---:|
| Successful eager scan | complete | yes | no |
| Successful streaming scan | complete | yes | no |
| Eager evaluation failure after accepted resource | partial | yes | yes |
| Streaming namespace failure after completed scope | partial | yes | yes |
| Producer failure after completed scope | partial | yes | yes |
| Policy-load failure before evaluation | failed | no | yes |
| Producer failure before accepted batch | failed | no | yes |
| Illegal or failed finalization | failed | no | yes |

### 12.2 Finalizer properties

- Calling finalization twice produces byte-equivalent completion metadata and summaries.
- Resource results are neither duplicated nor dropped.
- Score reweighting occurs at most once.
- Completed and unprocessed scopes have deterministic order.
- A provisional value is never labeled final.

### 12.3 Error and lifecycle properties

- `errors.Is` works for cancellation and deadline errors.
- `errors.As` works for evaluator and producer errors.
- Scan, delivery, and threshold errors retain separately inspectable typed identities when combined.
- Producer goroutines terminate on every streaming return path.
- Writers close exactly once.
- Partial artifacts never reach the complete-report submitter.

### 12.4 Serialization

Golden tests cover JSON and YAML complete, partial, and legacy-compatible output. Complete v1 output omits `scanCompletion`; partial output requires it and includes stable status, scan stage, reason code, finality flags, and known scope information without leaking raw failure contents; failed scans produce no posture artifact. Programmatic tests require an explicit completion enum for every `ScanWithOutcome` return.

## 13. Acceptance criteria

- [ ] A typed API distinguishes complete, finalized-partial, and failed-without-results.
- [ ] Eager evaluation errors preserve results and original errors without reporting completion.
- [ ] Streaming errors after completed scopes run a defined finalizer and never expose default summaries as final.
- [ ] Pre-result failures remain fatal and are not serialized as posture reports.
- [ ] Partial metadata is deterministic in JSON, YAML, and programmatic output.
- [ ] CLI emits partial output only with a non-zero exit; thresholds do not replace the primary error.
- [ ] HTTP exposes both the artifact and typed failure status.
- [ ] Partial output is not uploaded as a complete report.
- [ ] Render, persistence, HTTP response, and submission failures remain separate from `ScanCompletion` and `ScanOutcome.Err`.
- [ ] Complete v1 reports omit `scanCompletion`; partial reports require `scanCompletion.status: partial`; failed scans emit no posture report.
- [ ] Writer closure remains exactly once across fatal, partial, and success paths.
- [ ] `errors.Is` and `errors.As` work for cancellation, deadline, evaluator, and producer errors.
- [ ] Eager and streaming failure-injection tests pass under `-race`.

## 14. Open questions

1. Should the public type be `ScanOutcome`, or should an existing result type gain completion fields?
2. Which package owns `ScanCompletion` and `ScanStage` so that core, CLI, HTTP, and report serialization do not create dependency cycles?
3. Which score and coverage fields are meaningful for completed scopes, and should provisional numeric values be emitted by default?
4. What is the minimum deterministic scope identity required for streaming reports without increasing collection-memory bounds?
5. Should `ArtifactDeliveryOutcome` be a shared public type or a small caller-local interface with normative semantics?
6. What HTTP status and envelope best preserve both a partial artifact and independent delivery failure?
7. Which failure reason codes belong in the stable report schema for the first version?

Implementation beyond the bounded Phase 0 caller fix should wait for agreement on these API and serialization questions.

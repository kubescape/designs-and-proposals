# Proposal: Hour-of-Day Learning Coverage for Container Profiles

**Status:** Proposed
**Author:** Matthias Bertschy (matthias.bertschy@gmail.com)
**Date:** 2026-10-29

## Summary

Today a `ContainerProfile` is completed when **one container** of the workload either learns
without interruption for the full learning period (`maxSniffingTimePerContainer`, default 24h) or
exits cleanly. Long-running workloads whose containers rarely live that long — nightly
scale-to-zero, spot-node churn, frequent rollouts — stay in `Learning` forever, even though storage
has already merged many hours of behavior from their containers into the profile.

This proposal lets **storage** complete such a profile once, for every moment of the UTC day, the
workload has either been learned at that time of day or has never run at that time of day — no
matter how many containers it took. Storage already holds the merged profile, the report chains of
every container, and a Kubernetes client, so it has everything needed. node-agent only adds one
annotation (`kubescape.io/learning-start`); its learning timer is unchanged. When storage completes
a profile, containers still learning are stopped through the existing "already completed" path.

The rule keeps the guarantee that motivates the 24h default: **every time of day at which the
workload ran has been observed**, so daily jobs inside the workload (a nightly cleanup, a cron
thread, a batch window) are in the profile. A single container learning 24h without interruption
is a special case of the rule, so the result is never weaker than today.

## Motivation

The 24h learning period exists so that a profile sees a full daily cycle. Requiring that cycle to
come from one uninterrupted container is stricter than needed, and in common deployment patterns
it can never be met:

- **Scheduled scale-to-zero.** Non-production namespaces are often scaled down at night
  (e.g. 06:00–20:00 UTC up, 0 replicas overnight and on weekends). No container lives more than
  14h, so no profile ever completes.
- **Spot / autoscaled nodes.** Nodes live minutes to hours; containers move with them.
- **Frequent rollouts.** Each rollout creates a new ReplicaSet and a new profile; replicas also come
  and go with the HPA.

Data from one anonymized production user (9 clusters, 15 days, ~540k container learning runs,
taken from node-agent's `container.profile.learning` trace spans):

| Workload pattern | ReplicaSets | Longest single-container learning | Share of time the newest ReplicaSet showed Completed (24h rule) |
|---|---|---|---|
| Stage namespaces, nightly scale-to-zero | 72 | exactly 14.0h | 0% |
| Production, spot nodes + frequent rollouts | 89 | < 24h for most ReplicaSets | 40–80% |

The obvious alternative — **summing** learned time across containers until it reaches 24h — does
**not** preserve the daily-cycle guarantee. In the same data, of the 71 ReplicaSets whose summed
learning reached 24h, all 31 stage ReplicaSets had never been observed between 20:00 and 06:00
(median 10.2h of the day missing): three days of 08:00–16:00 add up to 24h without ever seeing
02:00. In that case the night happened to be empty (0 replicas), but a summing rule cannot tell
"ran unobserved" from "did not run". This proposal makes that distinction explicit.

## Goals

- Complete a profile once every time of day at which the workload ran has been observed, whether
  by one container or several.
- Keep the change in storage, with node-agent's learning logic untouched.
- Be conservative under missing information: when in doubt, keep learning.
- Be backward compatible in both directions of version skew, with no new completion protocol.

## Non-goals

- Weekly or monthly behavior. Like today, only the daily cycle is guaranteed.
- Carrying learning across ReplicaSets (a new pod template means a new profile, as today). See
  Open questions.
- Changing what is learned, how profiles are merged, or how `failed` / `too_large` are handled.
  Coverage completion only applies to a profile whose status is `learning`.
- Changing completion after the full learning period or on a clean container exit (exit code 0),
  which is how run-to-completion workloads (Jobs) complete today.
- Changing behavior when a learning period **shorter** than 24h is in effect (cluster-wide or via
  the `kubescape.io/max-sniffing-time` pod label): the user has explicitly opted out of daily
  coverage, and the current single-container rule stays.

## Background: current state

- **node-agent** (`pkg/containerprofilemanager/v1`):
  - When it starts watching a container, `addContainer` arms
    `time.AfterFunc(calculateSniffingTime(container), handleContainerMaxTime)`.
    `calculateSniffingTime` returns exactly `maxSniffingTimePerContainer` (never jittered), or the
    value of the `kubescape.io/max-sniffing-time` pod label. The effective value is reported on
    every chunk as the learning-period label (`objectcache.GetLabels`).
  - Every `updateDataPeriod` (default 10m) it sends a time-series `ContainerProfile` chunk. Each
    chunk carries a series ID (one per watched container), `ReportTimestamp`,
    `PreviousReportTimestamp`, and a completion annotation: `full` when node-agent watched the
    container from its start, `partial` otherwise (for example, after a node-agent restart).
  - A chunk with nothing new is **skipped**, and the chain links only to reports actually sent. So
    two consecutive linked reports prove the container was watched continuously between them,
    including idle time.
  - It sends a final report (`forceSend`):
    - when the timer expires, with status `completed`;
    - when the container exits, with status `completed` for exit code 0 and `failed` otherwise.
  - When storage rejects a report because the profile is already completed (`ObjectCompletedError`),
    node-agent marks the container completed and stops learning it. This is how other replicas
    stop today once one replica completes the profile.
- **storage** (`pkg/registry/file/containerprofile_processor.go`):
  - Time-series chunks are recorded in the `time_series` table and consolidated periodically into
    the profile. `consolidateContinuousTimeSeries` collapses a linked chain into one row, keeping
    only its latest `ReportTimestamp`.
  - `updateProfileStatus` marks the profile `completed` from a single series with no gap
    (`PreviousReportTimestamp` back to zero) whose last status is `completed`.
  - `PreSave` rejects reports for a profile that is already `completed`/`full`. Consolidation of a
    frozen profile reclaims leftover rows.
  - Series with no report for `2 × maxSniffingTime` are expired and finalized as
    `completed`/`partial`. Workloads that keep starting new containers never reach this.
  - storage runs an in-cluster `kubernetes.Interface` client (`pkg/apiserver/apiserver.go`).

So for a long-running workload, which never exits cleanly, a profile completes only when one
container learns for the full `T`.

## Proposal

### Definitions

All times are UTC. A **time of day** is a point in `[00:00, 24:00)`; projecting an absolute time
onto the time of day means taking it modulo 24h. UTC avoids DST ambiguity; behavior that follows
local time still recurs at a fixed UTC time of day, except across a DST change.

For a **profile key** (the existing `ContainerProfile` identity: workload instance, i.e.
ReplicaSet, StatefulSet, DaemonSet or Job, plus container name), storage collects three kinds of
**absolute-time** evidence:

- **Running intervals `R`**: when a container of that name, in any pod owned by the workload
  instance, was running. They come from the pods' `containerStatuses` (`state.running.startedAt`,
  `state.terminated.startedAt`/`finishedAt`, and `lastState` for restarts), so a container in a
  terminating pod counts as running until it actually finishes. These timestamps have one-second
  precision, so running intervals are rounded **outward** to whole seconds.
- **Unknown periods `U`**: when storage's pod informer was not synced and watching. They are treated
  as running.
- **Learning segments `L`**: when node-agent was verifiably watching such a container, taken from
  the report chains (see "Learning segments" below).

Evidence about an absolute time `t` is **settled** once the informer has been continuously synced
from `t` until now and `now - t` exceeds a settle delay (e.g. 10 minutes, well above kubelet's
status-update latency). Settling is asymmetric. Evidence that the container **ran**, or that its
state was unknown, is used immediately, because it can only make the result more conservative.
Evidence that it **did not run** is used only once settled, because a status update still in flight
could contradict it.

From the evidence, storage maintains three **time-of-day sets** per profile key, each a list of
disjoint intervals over `[00:00, 24:00)` with millisecond endpoints:

| Set | Contains time of day `τ` when, on some day, … | Rounding |
|---|---|---|
| `Learned` | some container of the profile key was inside a learning segment at `τ` (projection of `L`). | inward |
| `Occupied` | the container was running, or its state unknown, at `τ` (projection of `R ∪ U`, used immediately). | outward |
| `Vacant` | the state was settled and known, and the container was not running at `τ`. | inward |

The time of day that is **covered** is

```
Covered = Learned ∪ (Vacant \ Occupied_eval)
Occupied_eval = Occupied ∪ projection of [now - settle delay, now]
```

So a time of day is covered either because the workload was watched at that exact time on some
day, or because it has **never** been seen running at that time and has been seen not running there
at least once. The recent, not yet settled window counts as occupied when evaluating. Vacancy seen
on an older day therefore can't cover a time of day at which the workload may have just run, before
its pod status reached storage. That window can still be covered through `Learned`, typically from
earlier days.

Every set is a union over days, so it only grows. `Learned` does not depend on `R` at all, so
running evidence that arrives late can never invalidate it. Late evidence can only grow `Occupied`,
which can only shrink `Covered`. The sets are bounded: past a cap (e.g. 512 intervals), `Occupied`
merges its closest neighbors (it can only grow), and `Learned` and `Vacant` drop their shortest
intervals (they can only shrink). Either way the result is conservative.

### Learning segments

A **learning segment** is a maximal interval during which one series is verifiably watched:

- If the series' first report links back to nothing, it starts at the series' **watch start**, the
  new `kubescape.io/learning-start` annotation (see "node-agent change"). For a `full` series,
  node-agent caught the container at creation, before its process ran, so the segment starts at the
  container's start (`startedAt`, rounded down) when that is earlier. Without this, every container
  start would leave a short unlearned gap. A workload that scales up at the same time every day
  would leave that gap at the same time of day, and never complete.
- After a broken link, it starts at the `ReportTimestamp` of the first report after the gap: data
  between a missing report and its successor is lost, so that time isn't credited.
- It is extended to the `ReportTimestamp` of every report that links to its predecessor.
- When a series ends with its container's exit, node-agent watched the container until it
  finished. So its last segment ends at the container's `finishedAt` (rounded up) when that is
  later than the final `ReportTimestamp`. Like the start rule, this keeps a container watched for
  its whole life from leaving sub-second unlearned slivers.
- A broken link closes a segment.

Because the collapsed row keeps only the chain's latest `ReportTimestamp`, storage persists, per
active series, the current segment's `[start, end]` and the last linked `ReportTimestamp` across
consolidation passes. Reports consolidated in different passes (at 00:10, 00:20 and 00:30, say)
therefore extend one 30-minute segment instead of producing three disjoint fragments.

### Completion rule

For a profile key in status `learning` whose effective learning period `T` (from the chunks'
learning-period label) is at least 24h, storage completes the profile when:

1. `Covered` is the whole day `[00:00, 24:00)`, **and**
2. at least one contributing series was a full observation (completion `full`), so start-up
   behavior is in the profile, **and**
3. for `T > 24h`, the total learned wall-clock time (union of learning segments, not their sum) is
   at least `T`.

Completing means setting the consolidated profile to `completed`/`full`, derived from these
conditions rather than copied from any one series. The existing completion paths (a series
completing after the full `T`, a clean exit, expiry) are unchanged. Whichever comes first
completes the profile.

### Storage changes

1. **Learning segments.** Read `kubescape.io/learning-start` and the completion annotation from
   chunks, and build and persist segments as described above.
2. **Running intervals.** A pod informer (one per cluster, metadata and `status.containerStatuses`
   only) records `R` per workload instance and container name, using each pod's `ownerReferences`
   to find the instance. This works the same way for ReplicaSets, StatefulSets, DaemonSets and
   Jobs, and includes terminating pods. Informer gaps (storage restart, watch expiry before resync)
   are recorded as `U`.
3. **Time-of-day sets.** storage keeps the three sets per profile key in its own metadata table:
   - when a segment is extended, its projection is added to `Learned`;
   - when running or unknown evidence arrives, its projection is added to `Occupied`;
   - once absolute time becomes settled, the projection of its known not-running part is added to
     `Vacant`.

   Only recent absolute-time evidence (segments and running intervals not yet projected and
   settled) needs to be kept, so storage doesn't keep an unbounded history.
4. **Learned-time union (only for `T > 24h`).** `L` is also kept as a list of disjoint
   `[start, end)` ranges. Overlapping or touching ranges are coalesced, so overlapping replicas
   aren't double-counted (00:00–12:00 plus 06:00–18:00 is 18h, not 24h). It is updated in the
   **same transaction** that deletes the consolidated time-series rows, so a crash can neither drop
   nor double-count an interval. Past a cap (e.g. 256 ranges) the shortest ranges are dropped, which
   can only under-count.
5. **Evaluation and completion.** During each consolidation pass, for every profile key in
   `learning` with `T ≥ 24h`, storage evaluates the completion rule. When it holds, storage sets the
   consolidated profile to `completed`/`full`. From then on:
   - `PreSave` rejects new reports, and node-agent stops each container still learning on its next
     report (`ObjectCompletedError`), as it does today when another replica completes first;
   - consolidation treats the profile as frozen and reclaims leftover rows, as today.
6. **Visibility** (annotations on the consolidated profile, informational):
   - `kubescape.io/coverage-ratio`: share of the day covered, e.g. `0.93`;
   - `kubescape.io/completion-source: coverage` when this rule completed the profile.

### node-agent change

Add `kubescape.io/learning-start` (the time node-agent started watching the container, RFC 3339
with sub-second precision) to every chunk. That is the only change: the learning timer, the final
reports and the handling of `ObjectCompletedError` stay as they are.

The annotation is needed because storage can't derive the watch start on its own. A chunk
identifies the workload instance and container name, but not the pod, so storage can't match it to
a pod's `startedAt`.

### Why this guarantees the daily cycle

Take any time of day `τ` at which the workload runs a daily job. When storage completes a profile
through coverage, `τ ∈ Covered`, so either:

- `τ ∈ Learned`: on some day, a container of this exact pod template was being watched without a
  gap **at that exact time of day**. If the job ran then, its execs, file opens, syscalls and
  network peers were recorded. Watching a neighboring moment is not enough: observing
  02:00:00–02:00:10 says nothing about 02:00:45; or
- `τ ∈ Vacant \ Occupied_eval`: on **no** day did the container run at `τ` (or was its state
  unknown there), including the recent unsettled window, and on some settled day it was seen not
  running there. So the job has never run at that time of day for this workload instance. Empty
  evidence can't hide a time of day at which the workload ran unobserved: one unlearned running
  occurrence puts `τ` in `Occupied` for good. Only `Learned` can then cover it.

A single container learning 24h puts the whole day in `Learned`, so the rule is a strict
generalization of today's. It relaxes exactly one requirement: that the whole day be observed by
the *same* container on *consecutive* hours.

What it does **not** guarantee, same as today:

- behavior that does not recur daily;
- a job that runs only on one replica that wasn't being watched while another replica was.
  `Learned` asks that at every time of day *some* running replica was watched, not every replica.
  Replicas of one pod template run the same code, but in-process leader election can confine a
  job to one replica. Open question below.

### Worked examples

- **Nightly scale-to-zero (06:00–20:00 UTC up), pod informer synced throughout.**
  - Day 1: a container starts at 06:00:20, is watched for its whole life, and stops at 20:00:05.
    `Learned` gets 06:00:20–20:00:06 (start and end rounded outward, like `Occupied`). Nothing runs
    overnight, so as the night settles, 20:00:06 onwards goes into `Vacant` and never into
    `Occupied`.
  - Day 2: a new container starts at 06:00:15 and is learned from there. By ~06:10:15 the night up
    to 06:00:15 has settled as vacant, so the whole day is covered. The next consolidation pass
    completes the profile, and the container stops learning on its next report. Today it never
    completes.
- **Spot churn, replicas overlapping around the clock.** Time of day goes into `Learned` as soon
  as some replica is watched at that time on some day. Typically the whole day is learned within
  ~24h of wall-clock time, instead of needing one container to survive 24h.
- **Single long-lived container.** The whole day becomes `Learned` at the same moment its 24h
  timer fires, so it completes exactly as today, through whichever path runs first.
- **Ran unobserved once.** The workload ran at 02:00 on one night, on a node where node-agent was
  down, and was empty at 02:00 on every other night. 02:00 is in `Occupied`, so the vacant nights
  can't cover it. The profile waits until a container is watched at 02:00, or until today's 24h
  rule completes it.

## Compatibility and rollout

- **Feature flag in storage only**, default off at first: `containerProfileCoverage.enabled`.
- **Old node-agent + new storage:** chunks have no `learning-start`, so segments start at the first
  report. Every container start then leaves a gap of up to `initialDelay` (2 minutes by default)
  that is never learned, and workloads that start at the same time every day won't complete through
  coverage. They complete as today. Nothing completes wrongly.
- **New node-agent + old storage:** the annotation is ignored. As today.
- **Downgrading storage:** profiles already completed stay completed (old storage honors
  `completed`/`full`). Coverage evidence is no longer collected, and the rest behaves as today.
  No new status or protocol exists that old storage could misread.
- No change to the `ContainerProfile` spec, and no new status value. Only new annotations.

## Alternatives considered

1. **Sum learned time across containers.** Rejected: it doesn't preserve the daily cycle (see
   Motivation, 31/31 stage ReplicaSets would have completed without ever observing the night).
2. **Whole-slot accounting (quarter-hours learned or empty as a whole).** An earlier draft.
   Rejected: a workload that scales at 06:01 leaves the 06:00–06:15 slot partly learned and partly
   empty forever, so it never completes. And OR-ing "empty" occurrences lets an empty night cover a
   time of day at which the workload ran unobserved on another night.
3. **Per-minute bitmaps with set-only bits.** An earlier draft. Rejected: watching 10 seconds of a
   minute marked the whole minute learned, and a set-only bit couldn't be cleared when late running
   evidence contradicted it.
4. **Workload status for emptiness (`status.replicas == 0`).** Rejected: the field doesn't exist on
   DaemonSet or Job status, and on ReplicaSets it excludes terminating pods whose containers are
   still running.
5. **node-agent shortens its learning timer from storage-published coverage, and storage gates the
   resulting completion.** The previous revision of this proposal, which kept node-agent as the
   only component that completes profiles. Rejected: it needed a lot of extra machinery to be safe.
   - a published per-minute coverage view with a freshness check;
   - a new final status that storage without the proposal would treat as `learning`, so that
     downgrades couldn't complete profiles unchecked;
   - a gate cutoff at the final report, with dated evidence to reconstruct coverage as of that
     cutoff (the completing container keeps running after it stops learning);
   - a minimum timer covering the settle delay.

   A container stopped by a shortened timer also couldn't recover if the gate then rejected the
   completion. Storage-side completion gets the same guarantee through the existing
   `ObjectCompletedError` path, and containers keep learning until the profile is really complete.
6. **Shorter learning period via the `kubescape.io/max-sniffing-time` label.** Already available
   and useful when the user accepts losing the daily guarantee. It can't help workloads that need
   the guarantee.

## Open questions

1. **Pod informer cost.** A cluster-wide pod informer in storage (status only, with a transform
   that drops everything else) adds memory and apiserver watch load. Could node-agent provide the
   running intervals for the pods on its node instead? That would lose pods on nodes without a
   healthy node-agent, which the informer marks as running but unobserved.
2. **Leader-elected jobs.** Should `Learned` require that *every* running replica was watched at a
   time of day? That's stricter, makes `Learned` depend on `R` (so on settled time only), and
   delays completion whenever one node's node-agent lags.
3. **Carry-over across ReplicaSets with an identical image.** A rollout that changes only
   configuration or labels currently resets learning. Should the evidence carry over when every
   container's image digest is unchanged?
4. **Evidence aging.** `Occupied` is permanent, so one unobserved night can delay coverage
   completion until today's rule kicks in. Should `Occupied` and `Vacant` evidence older than some
   window (e.g. 7 days) be dropped?
5. **Idle-heartbeat reports.** An idle container extends its segment only when its next report or
   its final report arrives, which delays (never falsifies) coverage. Should node-agent send an
   empty linked report at least every few minutes? That would be a second, optional node-agent
   change.
6. **Evaluation cadence.** Is the consolidation pass the right place to evaluate, or should
   evaluation be triggered when a profile's evidence changes?

## Testing

- **storage unit tests**:
  - segment construction across consolidation passes, split rows, broken links, and the
    start and end rules for full series;
  - time-of-day set evaluation: fragments inside a minute, transitions, terminating pods, informer
    gaps, the asymmetric settle delay and the unsettled window, late `L` and late `R` evidence, the
    interval cap;
  - the `Occupied` block on the vacancy path;
  - completion: the three conditions, `T < 24h` and non-`learning` profiles left alone,
    `completed`/`full` derivation, and rejection of later reports via `PreSave`;
  - chunks from old node-agents without `learning-start`.
- **node-agent unit test**: chunks carry `kubescape.io/learning-start`.
- **Component test**: a Deployment scaled 1→0→1 on a short schedule, with an artificially shortened
  day (day length configurable in tests). The profile must complete through coverage only after
  the zero-replica window has settled as vacant, must not complete if a replica ran unobserved
  during it, and the running container must stop learning after completion.
- **Telemetry**: add `coverage` attributes (learned, vacant, occupied-unlearned and uncovered
  seconds of day) to the existing `container.profile.learning` span, so the effect can be measured
  in the field the same way the motivation data was.

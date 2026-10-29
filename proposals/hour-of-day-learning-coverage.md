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

This proposal changes **how long node-agent learns a new container**, not who completes the
profile. Storage records, for every moment of the UTC day, whether the workload has been learned
at that time of day, or has never run at that time of day. When node-agent starts watching a
container, it reads that record and learns only until the remaining uncovered times of day have
passed. Completion still happens the way it does today: node-agent sends the container's last
time-series profile with status `completed`.

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
- Keep node-agent as the only component that declares a profile `completed`, through the existing
  final time-series report, and keep storage's existing continuity check on that report.
- Be conservative under missing information: when in doubt, keep learning.
- Be backward compatible in both directions of version skew.

## Non-goals

- Weekly or monthly behavior. Like today, only the daily cycle is guaranteed.
- Carrying learning across ReplicaSets (a new pod template means a new profile, as today). See
  Open questions.
- Changing what is learned, how profiles are merged, or how `failed` / `too_large` are handled.
- Changing completion on a clean container exit (exit code 0), which is how run-to-completion
  workloads (Jobs) complete today.
- Changing behavior when a learning period **shorter** than 24h is configured (cluster-wide or via
  the `kubescape.io/max-sniffing-time` pod label): the user has explicitly opted out of daily
  coverage, and the current single-container rule stays.

## Background: current state

- **node-agent** (`pkg/containerprofilemanager/v1`):
  - When it starts watching a container, `addContainer` arms
    `time.AfterFunc(calculateSniffingTime(container), handleContainerMaxTime)`.
    `calculateSniffingTime` returns exactly `maxSniffingTimePerContainer` (never jittered,
    see kubescape/node-agent#1060), or the value of the `kubescape.io/max-sniffing-time` pod label.
  - Every `updateDataPeriod` (default 10m) it sends a time-series `ContainerProfile` chunk. Each
    chunk carries a series ID (one per watched container), `ReportTimestamp` and
    `PreviousReportTimestamp`.
  - A chunk with nothing new is **skipped**, and the chain links only to reports actually sent. So
    two consecutive linked reports prove the container was watched continuously between them,
    including idle time.
  - It sends a final report (`forceSend`):
    - when the timer expires, with status `completed`;
    - when the container exits, with status `completed` for exit code 0 and `failed` otherwise
      (`removeContainer`, via `objectcache.GetTerminationExitCode`).
  - When storage answers with "already completed", node-agent stops learning that container.
- **storage** (`pkg/registry/file/containerprofile_processor.go`):
  - Time-series chunks are recorded in the `time_series` table and consolidated periodically into
    the profile. `consolidateContinuousTimeSeries` collapses a linked chain into one row.
  - `updateProfileStatus` marks the profile `completed` from a single series with no gap
    (`PreviousReportTimestamp` back to zero) whose last status is `completed`. It does not check
    how long that series learned. `SetCompletedStatus` copies `full`/`partial` from that series.
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
status-update latency). `R` and `U` are only used for settled time.

From the evidence, storage maintains three **time-of-day sets**, each a list of disjoint intervals
over `[00:00, 24:00)` with millisecond endpoints:

| Set | Contains time of day `τ` when, on some day, … | Rounding |
|---|---|---|
| `Learned` | some container of the profile key was inside a learning segment at `τ` (projection of `L`). | inward |
| `Occupied` | the container was running, or its state unknown, at `τ` (projection of settled `R ∪ U`). | outward |
| `Vacant` | the state was settled and known, and the container was not running at `τ`. | inward |

The time of day that is **covered** is

```
Covered = Learned ∪ (Vacant \ Occupied)
```

So a time of day is covered either because the workload was watched at that exact time on some
day, or because it has **never** been seen running at that time and has been seen not running there
at least once.

Every set is a union over days, so it only grows. `Learned` does not depend on `R` at all, so
running evidence that arrives late can never invalidate it. Late evidence can only grow `Occupied`,
which can only shrink `Covered`, and the completion gate re-evaluates `Covered` when it runs. The
sets are bounded: past a cap (e.g. 512 intervals), `Occupied` merges its closest neighbors (it can
only grow), and `Learned` and `Vacant` drop their shortest intervals (they can only shrink). Either
way the result is conservative.

### Completion rule

For an effective learning period `T ≥ 24h`, a profile may be completed **through coverage** when:

1. `Covered` is the whole day `[00:00, 24:00)`, evaluated on settled evidence, **and**
2. at least one contributing series was a full observation (node-agent watched the container from
   its start, completion `full`), so start-up behavior is in the profile, **and**
3. for `T > 24h`, the total learned wall-clock time (union of learning segments, not their sum) is
   at least `T`.

For `T < 24h` nothing changes. Completion on timer expiry after the full `T`, and on a clean exit,
also stay exactly as today.

### Learning segments

A **learning segment** is a maximal interval during which one series is verifiably watched:

- If the series' first report links back to nothing, it starts at the series' **watch start**, a
  new `kubescape.io/learning-start` annotation that node-agent puts on every chunk. For a `full`
  series, node-agent caught the container at creation, before its process ran, so the segment
  starts at the container's start (`state.running.startedAt`, rounded down) when that is earlier.
  Otherwise a container that is watched from its very start would leave a sub-second unlearned gap
  at every start.
- After a broken link, it starts at the `ReportTimestamp` of the first report after the gap: data
  between a missing report and its successor is lost, so that time isn't credited.
- It is extended to the `ReportTimestamp` of every report that links to its predecessor.
- A broken link closes it.

Today the collapsed row keeps only the chain's latest `ReportTimestamp`. storage therefore persists,
per active series, the current segment's `[start, end]` and the last linked `ReportTimestamp`
across consolidation passes. This lets reports consolidated in different passes (at 00:10, 00:20
and 00:30, say) extend one 30-minute segment instead of producing three disjoint fragments.

### Storage changes

1. **Record watch start.** Read `kubescape.io/learning-start` from chunks, and persist segment
   boundaries per series as described above.
2. **Running intervals.** A pod informer (one per cluster, metadata and `status.containerStatuses`
   only) records `R` per workload instance and container name, using each pod's `ownerReferences`
   to find the instance. This works the same way for ReplicaSets, StatefulSets, DaemonSets and
   Jobs, and includes terminating pods. Informer gaps (storage restart, watch expiry before resync)
   are recorded as unknown.
3. **Time-of-day evaluation.** storage keeps `R`, `U` and `L` in absolute time for a bounded
   look-back window (e.g. `2 × T`) in its own metadata table. When a segment is extended, it adds
   its projection to `Learned` straight away. Once absolute time becomes settled, it adds that
   time's projections to `Occupied` and `Vacant`. Evidence that arrives late, for example the next
   report of an idle container, still extends `Learned` for the time it now covers.
4. **Learned-time union (only for `T > 24h`).** `L` is kept as a list of disjoint `[start, end)`
   ranges in the same metadata table. Overlapping or touching ranges are coalesced, so overlapping
   replicas aren't double-counted (00:00–12:00 plus 06:00–18:00 is 18h, not 24h). It is updated in
   the **same transaction** that deletes the consolidated time-series rows, so a crash can neither
   drop nor double-count an interval. Past a cap (e.g. 256 ranges) the shortest ranges are dropped,
   which can only under-count.
5. **Exposed on the consolidated profile metadata** (annotations, readable without the spec):
   - `kubescape.io/covered-minutes`: a conservative per-minute view of `Covered`, used only to
     size node-agent's timer. Bit `m` is set when the whole minute `[m, m + 1min)` lies inside
     `Covered`. The 1440 bits = 180 raw bytes are encoded with standard base64 (RFC 4648 §4, with
     padding), which is always exactly 240 characters with no `=`. Minute `m` is bit
     `7 - (m mod 8)` of byte `m / 8`, most significant bit first: minute 0 (00:00–00:01) is the
     high bit of byte 0.
   - `kubescape.io/coverage-evaluated-at`: when storage last evaluated `Covered` (RFC 3339);
   - `kubescape.io/full-observed: "true"` once condition 2 holds;
   - `kubescape.io/learned-seconds`: size of the learned-time union (informational).

   The time-of-day sets stay internal to storage. The completion gate uses them exactly, never the
   per-minute view.
6. **Completion gate.** A container whose timer was shortened by this proposal ends with a **new
   final status, `coverage-completed`**, instead of `completed`. Every other final report
   (`completed` after the full `T`, `completed` on a clean exit, `failed`) is handled exactly as
   today, and so are reports from node-agents that don't implement this proposal.

   For a `coverage-completed` report, storage merges the data first, re-evaluates the sets, and
   completes the profile **only if** all three conditions of the completion rule hold on settled
   evidence. Otherwise the data is kept and the profile stays `learning`.

   This closes the case where a `partial` series (a container found after a node-agent restart)
   fills the last gaps: its data counts, but it cannot complete the profile until some full
   observation exists. It also makes the protocol safe against storage that doesn't implement it
   (see "Compatibility and rollout").
7. **Aggregate completion state.** When a `coverage-completed` report passes the gate, storage sets
   the aggregate profile to `completed`/`full`. That state is derived from `full-observed` and
   coverage, not copied from the final series as `SetCompletedStatus` does today. A gate-passing
   completion delivered by a `partial` series still freezes consolidation and rejects further
   reports, like any `completed`/`full` profile. The series' own row keeps its `partial`
   completion.

### node-agent changes

1. **Annotate chunks** with `kubescape.io/learning-start`.
2. **Compute the timer from coverage.** In `calculateSniffingTime`, when the effective `T ≥ 24h`
   and the profile key's consolidated profile exists and is still learning, read
   the coverage annotations with a metadata-only read. If `coverage-evaluated-at` is missing or
   older than a freshness bound (e.g. 3 × the consolidation interval), the coverage is ignored and
   the timer is `T`. Otherwise:

   ```
   if !fullObserved and this container is partial:
       timer = T                                   # it cannot satisfy condition 2
   else:
       uncovered = minutes of day not in covered
       deadline  = end of the last uncovered minute, walking forward over the next 24h from the
                   current minute (this container is learned from its watch start, so the
                   current minute can be learned too)
       if uncovered is empty: deadline = now + floor   # floor: e.g. 2 × updateDataPeriod, to
                                                       # send at least one linked report
       if T > 24h: deadline = max(deadline, now + (T - learnedSeconds))
       timer = min(deadline - now, T)              # never longer than today
   final status on expiry = (timer < T) ? coverage-completed : completed
   ```

   A container that is a full observation sets condition 2 itself, so for it `fullObserved` does
   not matter. If the profile does not exist or can't be read, the timer is `T` (today's behavior).
3. **Optional refresh.** Re-read the annotations periodically (e.g. hourly) and shorten the timer if
   other replicas covered the remaining minutes. Without it, completion still happens: the
   longest-running replica's own timer completes the profile, and the others stop on "already
   completed".

If storage's gate rejects a `coverage-completed` report (for example because new running evidence entered `Occupied` in
the meantime), node-agent has already stopped learning that container. The next container to start
reads the updated coverage and learns through the remaining minutes. An optional improvement is for
node-agent to re-arm instead of stopping when the gate rejects. Because `covered-minutes` is
conservative (a minute is set only when it is entirely covered), node-agent's timer never ends
before the exact gate could pass, except when new `Occupied` evidence arrives in the meantime.

### Why this guarantees the daily cycle

Take any time of day `τ` at which the workload runs a daily job. When the profile completes through
coverage, `τ ∈ Covered` on settled evidence, so either:

- `τ ∈ Learned`: on some day, a container of this exact pod template was being watched without a
  gap **at that exact time of day**. If the job ran then, its execs, file opens, syscalls and
  network peers were recorded. Watching a neighboring moment is not enough: observing
  02:00:00–02:00:10 says nothing about 02:00:45; or
- `τ ∈ Vacant \ Occupied`: on **no** settled day did the container run at `τ` (or was its state
  unknown there), and on some day it was seen not running there. So the job has never run at that
  time of day for this workload instance. Empty evidence can't hide a time of day at which the
  workload ran unobserved: one unlearned running occurrence puts `τ` in `Occupied` for good. Only
  `Learned` can then cover it.

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
  - Day 1: a container starts at 06:00:20 and is watched from its start until it stops at 20:00:05.
    `Learned` gets 06:00:20–20:00:05. Nothing runs overnight, so 20:00:06–06:00:19 goes into
    `Vacant` (inward rounding loses at most a second at each end) and never into `Occupied`.
    `Occupied` gets 06:00:20–20:00:05 (rounded outward), which is all learned.
  - Day 2: the new container starts at 06:00:15, a few seconds earlier, and is learned from there.
    The few seconds between 06:00:15 and 06:00:20 were vacant on day 1 and are learned on day 2,
    so the whole day is covered. After the settle delay, a full `covered-minutes` is published, the
    next timer is the floor, and the profile completes around 06:30 with `coverage-completed`.
    Today it never completes.
- **Spot churn, replicas overlapping around the clock.** Time of day goes into `Learned` as soon
  as some replica is watched at that time on some day. Typically the whole day is learned within
  ~24h of wall-clock time, instead of needing one container to survive 24h.
- **Single long-lived container.** Nothing is covered at its start, so the timer is `T` and it ends
  with `completed`. That is exactly today's behavior.
- **Ran unobserved once.** The workload ran at 02:00 on one night, on a node where node-agent was
  down, and was empty at 02:00 on every other night. 02:00 is in `Occupied`, so the vacant nights
  can't cover it. The profile waits until a container is watched at 02:00.

## Compatibility and rollout

- **Feature flag on both sides**, default off at first: node-agent
  `learningCoverage.enabled`, storage `containerProfileCoverage.enabled`.
- **Old node-agent + new storage:** reports never carry `coverage-completed`, so storage applies
  today's rule, and the node-agent timer is `T`, as today.
- **New node-agent + old storage:** there is no coverage annotation, so the timer is `T` and the
  container ends with `completed`. As today.
- **Downgrading storage (or turning its flag off) while node-agents still have shortened timers
  armed:** old storage doesn't know `coverage-completed`. Its `updateProfileStatus` treats any
  status other than `completed` and `failed` as learning (the `default` branch), so such reports
  are merged as data and **cannot** complete a profile without the gate. Coverage annotations left
  behind by newer storage go stale: `coverage-evaluated-at` stops advancing, and node-agent falls
  back to `T` once it is past the freshness bound. The worst case during a downgrade is that some
  containers stop learning early and their profiles complete later. A profile can never complete
  wrongly. No drain or reset step is required.
- **Mixed node-agents in one cluster:** each container uses its own node-agent's rule. The
  coverage evidence is shared, and completion is still first-completed-wins.
- No change to the `ContainerProfile` spec. Only new annotations and one new final status value.

## Alternatives considered

1. **Sum learned time across containers.** Rejected: it doesn't preserve the daily cycle (see
   Motivation, 31/31 stage ReplicaSets would have completed without ever observing the night).
2. **Whole-slot accounting (e.g. quarter-hours learned or empty as a whole).** An earlier draft of
   this proposal. Rejected: a workload that scales at 06:01 leaves the 06:00–06:15 slot partly
   learned and partly empty forever, so it never completes. Separately, OR-ing "empty" occurrences
   lets an empty night cover a time of day at which the workload ran unobserved on another night.
   Exact time-of-day sets plus `Occupied` fix both.
3. **Per-minute bitmaps with set-only bits.** Another earlier draft. Rejected: watching 10 seconds
   of a minute marked the whole minute learned, and a set-only bit couldn't be cleared when late
   running evidence contradicted it. Exact time-of-day sets, with `Learned` independent of `R` and
   `R` used only once settled, avoid both. The per-minute view survives only as a conservative
   input to node-agent's timer.
4. **Workload status for emptiness (`status.replicas == 0`).** Rejected: the field doesn't exist on
   DaemonSet or Job status, and on ReplicaSets it excludes terminating pods whose containers are
   still running. Container running intervals from pod status work for every owner kind and
   include terminating pods.
5. **Storage stamps `completed` on its own when `Covered` is the whole day.** It works, but it adds a
   second completion trigger that has to be reconciled with node-agent's in-memory state. Keeping
   node-agent as the only completer reuses the existing path. storage only gates it and derives
   the aggregate state.
6. **Shorter learning period via the `kubescape.io/max-sniffing-time` label.** Already available
   and useful when the user accepts losing the daily guarantee. It can't help workloads that need
   the guarantee.

## Open questions

1. **Metadata-only read.** What is the cheapest way for node-agent to read the annotations when a
   container starts: a dedicated subresource, a field selector, or the profile cache it already
   maintains? It should be one read per container start, not per report.
2. **Pod informer cost.** A cluster-wide pod informer in storage (status only, with a transform
   that drops everything else) adds memory and apiserver watch load. Could node-agent provide the
   running intervals for the pods on its node instead? That would lose pods on nodes without a
   healthy node-agent, which the informer marks as running but unobserved.
3. **Leader-elected jobs.** Should `Learned` require that *every* running replica was watched at a
   time of day? That's stricter, makes `Learned` depend on `R` again (so only on settled time), and
   delays completion whenever one node's node-agent lags.
4. **Carry-over across ReplicaSets with an identical image.** A rollout that changes only
   configuration or labels currently resets learning. Should the evidence carry over when every
   container's image digest is unchanged?
5. **Evidence aging.** `Occupied` is permanent, so one unobserved night can delay completion
   indefinitely if the workload never runs at that time again. Should `Occupied` and `Vacant`
   evidence older than some window (e.g. 7 days) be dropped?
6. **Idle-heartbeat reports.** An idle container extends its segment only when its next report or
   its final report arrives. Should node-agent send an empty linked report at least every few
   minutes, so `Learned` fills sooner for other replicas?
7. **Re-arm on gate rejection.** Should node-agent keep learning when storage rejects a
   `coverage-completed` report, rather than relying on the next container?

## Testing

- **storage unit tests**:
  - segment construction across consolidation passes, split rows and broken links;
  - time-of-day set evaluation (fragments inside a minute, transitions, terminating pods, informer
    gaps, the settle delay, late `L` and late `R` evidence, the interval cap);
  - the `Occupied` block on the vacancy path;
  - the completion gate for `coverage-completed`, other final statuses and legacy reports, and the
    aggregate `completed`/`full` derivation for a `partial` final series;
  - the conservative `covered-minutes` view and its encoding.
- **Downgrade test**: a `coverage-completed` report sent to storage without this proposal leaves
  the profile `learning`.
- **node-agent unit tests**: timer computation, table-driven over `covered-minutes`, current time, `T`
  and full/partial cases, plus the old-storage fallback.
- **Component test**: a Deployment scaled 1→0→1 on a short schedule, with an artificially shortened
  day (minute length and day length configurable in tests). The profile must complete only after
  the zero-replica window has been recorded as vacant, and must not complete if a replica ran
  unobserved during it.
- **Telemetry**: add `coverage` attributes (learned, vacant, occupied-unlearned and uncovered
  seconds of day) to the existing `container.profile.learning` span, so the effect can be measured
  in the field the same way the motivation data was.

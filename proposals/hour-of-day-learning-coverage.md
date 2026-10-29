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
profile. Storage records, minute by minute over the UTC day, whether the workload has been learned
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

All times are UTC. A **minute of day** `m` is one of the 1440 minutes `[m, m + 1min)` of the UTC
day. UTC avoids DST ambiguity; behavior that follows local time still recurs at a fixed UTC minute,
except across a DST change.

For a **profile key** (the existing `ContainerProfile` identity: workload instance, i.e.
ReplicaSet, StatefulSet, DaemonSet or Job, plus container name), storage tracks two kinds of
**absolute-time** evidence:

- **Running intervals `R`**: when a container of that name, in any pod owned by the workload
  instance, was running. They come from the pods' `containerStatuses` (`state.running.startedAt`,
  `state.terminated.startedAt`/`finishedAt`, and `lastState` for restarts), so a container in a
  terminating pod counts as running until it actually finishes. Periods in which storage's pod
  informer was not synced and watching are **unknown**, and are treated as running.
- **Learning segments `L`**: when node-agent was verifiably watching such a container, taken from
  the report chains (see "Learning segments" below).

From these, storage maintains three monotonic (set-only) bitmaps over the 1440 minutes of day:

| Bitmap | Bit `m` is set when, on some day, … |
|---|---|
| `learned` | the container was running at some moment of minute `m`, **and** every running moment of that minute lies inside a learning segment (`R ∩ m ≠ ∅` and `R ∩ m ⊆ L`). |
| `occupied` | the container was running, or its state was unknown, at some moment of minute `m`. |
| `vacant` | the state was known for the whole minute `m`, and the container was not running at any moment of it. |

A minute of day is **covered** when

```
covered[m] = learned[m] OR (NOT occupied[m] AND vacant[m])
```

That is, it is covered either because the workload was learned at that time of day, or because it
has **never** been seen running there and has been seen not running there at least once.

Evaluating `R ∩ m ⊆ L` on exact timestamps, not on whole minutes, means a container that starts at
06:01:30 and is learned from 06:01:30 still produces a learned minute 06:01. Coverage does not
depend on transitions lining up with any boundary.

### Completion rule

For an effective learning period `T ≥ 24h`, a profile may be completed **through coverage** when:

1. all 1440 minutes of day are covered, **and**
2. at least one contributing series was a full observation (node-agent watched the container from
   its start, completion `full`), so start-up behavior is in the profile, **and**
3. for `T > 24h`, the total learned wall-clock time (union of learning segments, not their sum) is
   at least `T`.

For `T < 24h` nothing changes. Completion on timer expiry after the full `T`, and on a clean exit,
also stay exactly as today.

### Learning segments

A **learning segment** is a maximal interval during which one series is verifiably watched:

- It starts at the series' **watch start** (a new `kubescape.io/learning-start` annotation that
  node-agent puts on every chunk) if the series' first report links back to nothing. Otherwise it
  starts at the `ReportTimestamp` of the first report after a broken link: data between a missing
  report and its successor is lost, so that time isn't credited.
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
3. **Bitmap evaluation.** storage keeps `R`, the unknown periods and `L` in absolute time for a
   bounded look-back window (e.g. `2 × T`) in its own metadata table. Whenever a segment is extended
   or `R` changes, it re-evaluates the affected absolute minutes and ORs the results into
   `learned`, `occupied` and `vacant`. Bits are only ever set. Evidence that arrives late, for
   example the next report of an idle container, can still set `learned` for minutes it now
   covers.
4. **Learned-time union (only for `T > 24h`).** `L` is kept as a list of disjoint `[start, end)`
   ranges in the same metadata table. Overlapping or touching ranges are coalesced, so overlapping
   replicas aren't double-counted (00:00–12:00 plus 06:00–18:00 is 18h, not 24h). It is updated in
   the **same transaction** that deletes the consolidated time-series rows, so a crash can neither
   drop nor double-count an interval. Past a cap (e.g. 256 ranges) the shortest ranges are dropped,
   which can only under-count.
5. **Exposed on the consolidated profile metadata** (annotations, readable without the spec):
   - `kubescape.io/covered-minutes`: the `covered` bitmap, 1440 bits = 180 raw bytes, encoded with
     standard base64 (RFC 4648 §4, with padding), which is always exactly 240 characters with no
     `=`. Minute `m` is bit `7 - (m mod 8)` of byte `m / 8`, most significant bit first: minute 0
     (00:00–00:01) is the high bit of byte 0.
   - `kubescape.io/full-observed: "true"` once condition 2 holds;
   - `kubescape.io/learned-seconds`: size of the learned-time union (informational).

   The three underlying bitmaps stay internal to storage.
6. **Completion gate for coverage completions.** node-agent tags every final report with
   `kubescape.io/completion-reason`: `max-time` (the timer ran the full `T`), `coverage` (a timer
   shortened by this proposal), or `exit` (clean exit).
   - `max-time` and `exit` reports, and **legacy reports** without the annotation, are handled
     exactly as today.
   - A `coverage` report is merged first. storage then re-evaluates the bitmaps, and completes the
     profile **only if** all three conditions of the completion rule hold. Otherwise the data is
     kept and the profile stays `learning`.

   This closes the case where a `partial` series (a container found after a node-agent restart)
   fills the last minutes: its data counts, but it cannot complete the profile until some full
   observation exists.
7. **Aggregate completion state.** When a `coverage` completion passes the gate, storage sets the
   aggregate profile to `completed`/`full`. That state is derived from `full-observed` and
   coverage, not copied from the final series as `SetCompletedStatus` does today. A gate-passing
   completion delivered by a `partial` series still freezes consolidation and rejects further
   reports, like any `completed`/`full` profile. The series' own row keeps its `partial`
   completion.

### node-agent changes

1. **Annotate chunks** with `kubescape.io/learning-start`, and final reports with
   `kubescape.io/completion-reason`.
2. **Compute the timer from coverage.** In `calculateSniffingTime`, when the effective `T ≥ 24h`
   and the profile key's consolidated profile exists and is still learning, read
   `covered-minutes`, `full-observed` and `learned-seconds` with a metadata-only read, then:

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
   reason = (timer < T) ? coverage : max-time
   ```

   A container that is a full observation sets condition 2 itself, so for it `fullObserved` does
   not matter. If the profile does not exist, can't be read, or has no `covered-minutes` (old
   storage), the timer is `T` (today's behavior).
3. **Optional refresh.** Re-read the annotations periodically (e.g. hourly) and shorten the timer if
   other replicas covered the remaining minutes. Without it, completion still happens: the
   longest-running replica's own timer completes the profile, and the others stop on "already
   completed".

If storage's gate rejects a `coverage` completion (for example because a minute became occupied in
the meantime), node-agent has already stopped learning that container. The next container to start
reads the updated bitmap and learns through the remaining minutes. An optional improvement is for
node-agent to re-arm instead of stopping when the gate rejects.

### Why this guarantees the daily cycle

Take any time of day `t` at which the workload runs a daily job, inside minute `m`. When the
profile completes through coverage, `covered[m]` holds, so either:

- `learned[m]`: on some day, the container was running during `m` and every running moment of `m`
  was watched without a gap. If the job ran that day, its execs, file opens, syscalls and network
  peers were recorded; or
- `NOT occupied[m] AND vacant[m]`: on **no** day did the container run at minute `m`, or was its
  state unknown there, so the job has never run at that time of day for this workload instance.
  Empty evidence can't hide a time of day at which the workload ran unobserved: one unlearned
  running occurrence sets `occupied[m]` and blocks the vacancy path for that minute forever. Only
  `learned[m]` can then cover it.

A single container learning 24h sets `learned` for all 1440 minutes, so the rule is a strict
generalization of today's. It relaxes exactly one requirement: that the whole day be observed by
the *same* container on *consecutive* hours.

What it does **not** guarantee, same as today:

- behavior that does not recur daily;
- a job that runs only on one replica that wasn't being watched while another replica was.
  `learned[m]` asks that at every moment *some* running replica was watched, not every replica.
  Replicas of one pod template run the same code, but in-process leader election can confine a
  job to one replica. Open question below.

### Worked examples

- **Nightly scale-to-zero (06:00–20:00 UTC up), pod informer synced throughout.**
  - Day 1: a container runs and is learned ~06:00:20–20:00:05. Minutes 06:00–20:00 become
    `learned`. Nothing runs overnight, so minutes 20:01–05:59 become `vacant` and never `occupied`.
    The two boundary minutes are learned, because their running part is fully learned.
  - Day 2: a new container starts at ~06:00, reads a full `covered-minutes`, arms the floor timer,
    and completes the profile by ~06:20 with reason `coverage`. Today it never completes.
- **Spot churn, replicas overlapping around the clock.** Minutes become `learned` as soon as, on
  some day, every running moment of that minute was watched by some replica. Typically the day is
  covered within ~24h of wall-clock time, instead of needing one container to survive 24h.
- **Single long-lived container.** Nothing is covered at its start, so the timer is `T` and the
  reason is `max-time`. That is exactly today's behavior.
- **Ran unobserved once.** The workload ran at 02:00 on one night, on a node where node-agent was
  down, and was empty at 02:00 on every other night. `occupied[02:00]` is set, so the vacant
  nights can't cover it. The profile waits until a container is learned at 02:00.

## Compatibility and rollout

- **Feature flag on both sides**, default off at first: node-agent
  `learningCoverage.enabled`, storage `containerProfileCoverage.enabled`.
- **Old node-agent + new storage:** reports carry no `completion-reason`. storage applies today's
  rule, and the node-agent timer is `T`, as today.
- **New node-agent + old storage:** no `covered-minutes` annotation, so the timer is `T` and the
  reason is `max-time`. Old storage ignores the new annotations. As today.
- **Mixed node-agents in one cluster:** each container uses its own node-agent's rule. The
  bitmaps are shared, and completion is still first-completed-wins.
- No change to the `ContainerProfile` spec. Only new annotations.

## Alternatives considered

1. **Sum learned time across containers.** Rejected: it doesn't preserve the daily cycle (see
   Motivation, 31/31 stage ReplicaSets would have completed without ever observing the night).
2. **Whole-slot accounting (e.g. quarter-hours learned or empty as a whole).** An earlier draft of
   this proposal. Rejected: a workload that scales at 06:01 leaves the 06:00–06:15 slot partly
   learned and partly empty forever, so it never completes. Separately, OR-ing "empty" occurrences
   lets an empty night cover a time of day at which the workload ran unobserved on another night.
   Exact intervals plus the `occupied` bitmap fix both.
3. **Workload status for emptiness (`status.replicas == 0`).** Rejected: the field doesn't exist on
   DaemonSet or Job status, and on ReplicaSets it excludes terminating pods whose containers are
   still running. Container running intervals from pod status work for every owner kind and
   include terminating pods.
4. **Storage stamps `completed` on its own when the bitmap is full.** It works, but it adds a
   second completion trigger that has to be reconciled with node-agent's in-memory state. Keeping
   node-agent as the only completer reuses the existing path. storage only gates it and derives
   the aggregate state.
5. **Shorter learning period via the `kubescape.io/max-sniffing-time` label.** Already available
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
3. **Leader-elected jobs.** Should `learned[m]` require that *every* running replica was watched
   during `m`? That's stricter, and it delays completion whenever one node's node-agent lags.
4. **Carry-over across ReplicaSets with an identical image.** A rollout that changes only
   configuration or labels currently resets learning. Should the evidence carry over when every
   container's image digest is unchanged?
5. **Evidence aging.** `occupied` is permanent, so one unobserved night can delay completion
   indefinitely if the workload never runs at that time again. Should `occupied` and `vacant`
   occurrences older than some window (e.g. 7 days) be dropped?
6. **Idle-heartbeat reports.** An idle container extends its segment only when its next report or
   its final report arrives. Should node-agent send an empty linked report at least every few
   minutes, so `learned` fills sooner for other replicas?
7. **Re-arm on gate rejection.** Should node-agent keep learning when storage rejects a
   `coverage` completion, rather than relying on the next container?

## Testing

- **storage unit tests**:
  - segment construction across consolidation passes, split rows and broken links;
  - bitmap evaluation on exact timestamps (transitions inside a minute, terminating pods, informer
    gaps, late evidence);
  - the `occupied` block on the vacancy path;
  - the completion gate for each `completion-reason`, legacy reports, and the aggregate
    `completed`/`full` derivation for a `partial` final series;
  - the encoding of `covered-minutes`.
- **node-agent unit tests**: timer computation, table-driven over the bitmap, current time, `T`
  and full/partial cases, plus the old-storage fallback.
- **Component test**: a Deployment scaled 1→0→1 on a short schedule, with an artificially shortened
  day (minute length and day length configurable in tests). The profile must complete only after
  the zero-replica window has been recorded as vacant, and must not complete if a replica ran
  unobserved during it.
- **Telemetry**: add `coverage` attributes (learned, vacant, occupied-unlearned and uncovered
  minute counts) to the existing `container.profile.learning` span, so the effect can be measured
  in the field the same way the motivation data was.

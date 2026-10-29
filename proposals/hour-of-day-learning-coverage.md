# Proposal: Hour-of-Day Learning Coverage for Container Profiles

**Status:** Proposed
**Author:** Matthias Bertschy (matthias.bertschy@gmail.com)
**Date:** 2026-10-29

## Summary

Today a `ContainerProfile` is completed only when **one container** of the workload learns without
interruption for the full learning period (`maxSniffingTimePerContainer`, default 24h). Workloads
whose containers rarely live that long — nightly scale-to-zero, spot-node churn, frequent rollouts —
stay in `Learning` forever, even though storage has already merged many hours of behavior from
their containers into the profile.

This proposal changes **how long node-agent learns a new container**, not who completes the
profile. Storage records which quarter-hours of the UTC day the workload has been observed in
(by any of its containers), plus the quarter-hours in which the workload had no running replicas.
When node-agent starts watching a container, it reads that record and learns only until the
remaining unobserved times of day have passed. Completion still happens the way it does today:
node-agent sends the container's last time-series profile with status `completed`.

The rule keeps the guarantee that motivates the 24h default: **every time of day in which the
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

- Complete a profile once every time of day in which the workload ran has been observed, whether
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
  - On container exit and on timer expiry it sends a final report (`forceSend`) with status
    `completed` (timer) or the terminal status.
  - When storage answers with "already completed", node-agent stops learning that container.
- **storage** (`pkg/registry/file/containerprofile_processor.go`):
  - Time-series chunks are recorded in the `time_series` table and consolidated periodically into
    the profile. `consolidateContinuousTimeSeries` collapses a linked chain into one row.
  - `updateProfileStatus` marks the profile `completed` only from a single series with no gap
    (`PreviousReportTimestamp` back to zero) whose last status is `completed`.
  - Series with no report for `2 × maxSniffingTime` are expired and finalized as
    `completed`/`partial`. Workloads that keep starting new containers never reach this.
  - storage runs an in-cluster `kubernetes.Interface` client (`pkg/apiserver/apiserver.go`).

## Proposal

### Definitions

- **Slot**: one of the 96 quarter-hours of the UTC day (00:00–00:15 is slot 0). UTC avoids DST
  ambiguity; behavior that follows local time still recurs at a fixed UTC slot, except across a DST
  change.
- **Profile key**: the existing `ContainerProfile` identity (workload instance, i.e. ReplicaSet or
  StatefulSet/DaemonSet/Job, + container name).
- **Learned slot**: a slot that lies entirely within a *covered interval* of some series of the
  profile key. A covered interval runs from the series' watch start to its latest linked report,
  or between two linked reports.
- **Empty slot**: a slot during which the profile's workload instance had **zero** pods for the
  entire slot, according to the per-kind signal defined in "Empty-slot accounting" below.
- **Covered slot** = learned slot OR empty slot.

### Completion rule

For an effective learning period `T ≥ 24h`, a profile may be completed when:

1. every one of the 96 slots is covered, **and**
2. at least one contributing series was a full observation (node-agent watched the container from
   its start, completion `full`), so start-up behavior is in the profile, **and**
3. for `T > 24h`, the total learned wall-clock time (union over series, not sum) is at least `T`.

For `T < 24h` nothing changes.

Completion still happens through node-agent's final report. The rule only changes the timer.

### Storage changes

1. **Watch start per series.** node-agent adds a `kubescape.io/learning-start` annotation (the
   time it started watching the container) to every chunk. storage records it per series. Today the
   collapsed row loses the chain's first timestamp.
2. **Learned-slot accounting.** At consolidation, for every collapsed continuous row of a series,
   storage ORs the slots fully inside `[learning-start or last accounted report, ReportTimestamp]`
   into the profile's learned mask. Slots are half-open: slot `i` is
   `[i × 15m, (i + 1) × 15m)`.
3. **Learned-time union (only needed for `T > 24h`).** storage keeps the exact union of covered
   intervals as a list of disjoint `[start, end)` ranges in its own metadata table, next to the
   profile, not in an annotation. Each new interval is merged into the list (overlapping or touching
   ranges are coalesced). The list is updated **in the same transaction** that deletes the
   consolidated time-series rows, so a crash can neither drop nor double-count an interval. Summing
   independent intervals would double-count overlapping replicas (00:00–12:00 and 06:00–18:00 are
   18h, not 24h). The list is bounded: past a cap (e.g. 256 ranges) the shortest ranges are dropped,
   which can only under-count.
4. **Empty-slot accounting.** One informer per kind (per cluster, not per profile) records, per
   workload instance, the intervals in which it had no pods:

   | Owner kind | Empty when |
   |---|---|
   | ReplicaSet | `status.replicas == 0` |
   | StatefulSet | `status.replicas == 0` |
   | DaemonSet | `status.currentNumberScheduled == 0` and `status.numberMisscheduled == 0` |
   | Job | `status.active == 0` (and the Job not yet started, or finished) |
   | anything else | never empty |

   `status.replicas` counts pods that exist, including pending ones, so a slot counts as empty only
   when no pod of that workload instance existed. A slot counts as empty only if the informer was
   synced and watching for the **whole** slot. A missing field, an unknown kind, or a gap in the
   informer (storage restart, resync) leaves the slot not empty. At consolidation storage ORs the
   fully empty slots into the profile's empty mask.
5. **Exposed on the consolidated profile metadata** (annotations, so they're readable without the
   spec):
   - `kubescape.io/learned-slots` and `kubescape.io/empty-slots`: each 96-bit mask is 12 raw bytes,
     encoded with standard base64 (RFC 4648 §4, with padding), which is always exactly 16
     characters with no `=`. Slot `i` (UTC `[i × 15m, (i + 1) × 15m)`) is bit `7 - (i mod 8)` of
     byte `i / 8`, i.e. most significant bit first: slot 0 (00:00–00:15) is the high bit of byte 0.
   - `kubescape.io/learned-seconds`: the size of the learned-time union (informational; the exact
     union lives in storage's metadata table);
   - `kubescape.io/full-observed: "true"` once condition 2 holds.
6. **Mandatory completion gate.** When a final `completed` report arrives from a series that
   carries `kubescape.io/learning-start` (i.e. from a node-agent that implements this proposal), and
   the series learned for less than `T`, storage completes the profile **only if** all three
   conditions of the completion rule hold *after* merging that report: masks full, `full-observed`
   set, and for `T > 24h` the union at least `T`. Otherwise the report's data is merged and the
   profile stays `learning`.

   This closes the case where a `partial` series (a container found after a node-agent restart)
   fills the last slots: its report is merged, but cannot complete the profile until some full
   observation exists.

   **Legacy reports** without `kubescape.io/learning-start` come from node-agents that don't
   implement this proposal and always learn for the full `T`. storage applies today's rule to them
   unchanged, so mixed and old clusters behave as they do now.

### node-agent changes

1. **Annotate chunks** with `kubescape.io/learning-start`.
2. **Compute the timer from coverage.** In `calculateSniffingTime`, when the effective `T ≥ 24h`
   and the profile key's consolidated profile exists and is still learning, read its annotations
   (a metadata-only read) and compute:

   ```
   if !fullObserved and this container is partial:
       timer = T                                   # it cannot satisfy condition 2
   else:
       uncovered = all slots not in (learned | empty)
       deadline  = end of the last uncovered slot, walking forward over the next 24h from the
                   first slot with slotStart >= watchStart (slots are half-open; a slot that
                   started before watchStart is not fully observable and maps to tomorrow's
                   occurrence)
       if uncovered is empty: deadline = now + floor   # floor: e.g. 2 × updateDataPeriod, to
                                                       # send at least one linked report
       if T > 24h: deadline = max(deadline, now + (T - learnedSeconds))
       timer = min(deadline - now, T)              # never longer than today

   A container that is a full observation sets condition 2 itself, so for it fullObserved does not
   matter.
   ```

   If the profile does not exist, can't be read, or has no masks (old storage), use `T` (today's
   behavior).
3. **Optional refresh.** Re-read the masks periodically (e.g. hourly) and shorten the timer if
   other replicas covered the remaining slots. Without it, completion still happens: the
   longest-running replica's own timer completes the profile, and the others stop on "already
   completed".

Containers that node-agent found already running (after a node-agent restart) produce `partial`
series. They contribute learned slots, but cannot satisfy condition 2 on their own.

### Why this guarantees the daily cycle

Take any time of day `t` at which the workload runs a daily job. On the day the profile completes,
the slot containing `t` is covered. Either:
- it is **learned**: some container of this exact pod template was watched without a gap through
  that whole slot, so if the job ran, its execs, file opens, syscalls and network peers were
  recorded; or
- it is **empty**: no pod of this workload instance existed during that slot, so there was nothing
  to run.

A single container learning 24h covers all 96 slots, so the rule is a strict generalization of
today's. It relaxes exactly one requirement: that the 96 slots be observed by the *same* container
on *consecutive* hours.

What it does **not** guarantee, same as today:
- behavior that does not recur daily;
- a job that runs only on some replica not being watched while another replica covers that slot.
  Replicas of one pod template run the same code, but in-process leader election can confine a
  job to one replica. Open question below.

### Worked examples

- **Nightly scale-to-zero (06:00–20:00 UTC up).**
  - Day 1: a container learns 06:00–20:00, so 56 slots are learned. At 20:00 replicas go to 0 and
    the 40 night slots become empty by 06:00.
  - Day 2: a new container starts at 06:00, reads full masks, arms the floor timer, and completes
    the profile by ~06:20. Today it never completes.
- **Spot churn, replicas overlapping around the clock.** Learned slots fill as soon as the union of
  replicas has been watched through every slot, typically within ~24h of wall-clock time instead of
  needing one container to survive 24h.
- **Single long-lived container.** Masks are empty at start, so the timer is 24h. That is exactly
  today's behavior.
- **Night never observed (replicas exist at night, but on nodes where node-agent was down).**
  Night slots stay uncovered. New containers keep a timer that runs through the next night, so the
  rule stays conservative.

## Compatibility and rollout

- **Feature flag on both sides**, default off at first: node-agent
  `learningCoverage.enabled`, storage `containerProfileCoverage.enabled`.
- **Old node-agent + new storage:** masks are written but ignored. The timer is `T`, as today.
- **New node-agent + old storage:** no annotations, so the timer is `T`, as today.
- **Mixed node-agents in one cluster:** each container uses its own node-agent's rule. The masks
  are shared, and completion is still the existing first-completed-wins path.
- No change to the `ContainerProfile` spec. Only new annotations.

## Alternatives considered

1. **Sum learned time across containers.** Rejected: it doesn't preserve the daily cycle (see
   Motivation, 31/31 stage ReplicaSets would have completed without ever observing the night).
2. **Storage stamps `completed` on its own when the masks are full.** It works, but it adds a
   second completion path that has to be reconciled with `failed`, `partial` / `full` and with
   node-agent's in-memory state. Keeping node-agent as the only completer reuses the existing,
   tested path.
3. **Shorter learning period via the `kubescape.io/max-sniffing-time` label.** Already available
   and useful when the user accepts losing the daily guarantee. It can't help workloads that need
   the guarantee.
4. **Per-hour (24 slots) instead of quarter-hour (96).** Hour slots are cheaper but lose up to
   two hours at each container boundary. Quarter-hours match the 10-minute report period.

## Open questions

1. **Metadata-only read.** What is the cheapest way for node-agent to read the four annotations
   when a container starts: a dedicated subresource, a field selector, or the profile cache it
   already maintains? It should be one read per container start, not per report.
2. **Empty-slot signals.** Are the per-kind signals in "Empty-slot accounting" right? A DaemonSet
   is never really "scaled to zero", so it may be simpler to treat DaemonSets as never empty.
   Would deriving empty intervals from the pods owned by each workload instance be more robust than
   reading workload status?
3. **Leader-elected jobs.** Should a slot count as learned only when *all* running replicas were
   watched during it? That's stricter, and it needs per-slot replica counts.
4. **Carry-over across ReplicaSets with an identical image.** A rollout that changes only
   configuration or labels currently resets learning. Should the masks carry over when every
   container's image digest is unchanged?
5. **Mask aging.** Should learned slots expire (e.g. after 7 days without completion) so that an
   observation from a very different day doesn't fill a slot indefinitely?
6. **Idle-heartbeat reports.** An idle container only credits its slots when its next report or its
   final report arrives. Should node-agent send an empty linked report at least once per slot, so
   the masks fill sooner for other replicas?

## Testing

- **storage unit tests**: slot accounting from collapsed chains, including chains that span
  consolidation passes, split rows and missing links; empty-slot accounting from replica-count
  intervals; annotations.
- **node-agent unit tests**: timer computation, table-driven over mask, current time, `T` and
  full/partial cases, including the "slot already in progress" boundary and old-storage fallback.
- **Component test**: a Deployment scaled 1→0→1 on a short schedule, with an artificially shortened
  day (slot length and day length configurable in tests). The profile must complete only after the
  zero-replica window has been recorded as empty.
- **Telemetry**: add a `coverage` attribute (learned/empty/uncovered slot counts) to the existing
  `container.profile.learning` span, so the effect can be measured in the field the same way the
  motivation data was.

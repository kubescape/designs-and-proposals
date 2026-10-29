# Proposal: Hour-of-Day Learning Coverage for Container Profiles

**Status:** Proposed
**Author:** Matthias Bertschy (matthias.bertschy@gmail.com)
**Date:** 2026-10-29

## Summary

Today a `ContainerProfile` is completed when **one container** of the workload either learns
without interruption for the full learning period (`maxSniffingTimePerContainer`, default 24h) or
exits cleanly. Always-on workloads whose individual containers rarely live that long — spot-node
churn, autoscaling, frequent rollouts — stay in `Learning` forever, even though their replicas,
taken together, have been watched around the clock.

This proposal lets **storage** complete such a profile once **every time of day has been watched**
by some container of the workload, on some day. Storage already holds the merged profile and the
report chains of every container, so it can tell exactly when each container was watched. node-agent
adds two annotations and a heartbeat report; its learning timer is unchanged. When storage completes
a profile, containers still learning are stopped through the existing "already completed" path.

The guarantee is the one the 24h default exists for: **every time of day has been observed**, so
daily jobs inside the workload (a nightly cleanup, a cron thread, a batch window) are in the
profile. A single container learning 24h without interruption is a special case, so the result is
never weaker than today.

Workloads that don't run at all hours (for example, scaled to zero at night) can't have every time
of day watched, so they don't complete through this rule. For them, the per-workload
`kubescape.io/max-sniffing-time` label is the right tool, and it loses nothing, because the hours
it skips are hours in which the workload isn't running.

## Motivation

The 24h learning period exists so that a profile sees a full daily cycle. Requiring that cycle to
come from one uninterrupted container is stricter than needed for always-on workloads:

- **Spot / autoscaled nodes.** Nodes live minutes to hours; containers move with them.
- **Frequent rollouts and HPA.** Replicas come and go many times a day.

Data from one anonymized production user (9 clusters, 15 days, ~540k container learning runs,
taken from node-agent's `container.profile.learning` trace spans):

| Workload pattern | ReplicaSets | Longest single-container learning | Share of time the newest ReplicaSet showed Completed (24h rule) |
|---|---|---|---|
| Production, spot nodes + frequent rollouts | 89 | < 24h for most ReplicaSets | 40–80% |
| Stage namespaces, nightly scale-to-zero | 72 | exactly 14.0h | 0% |

For the production workloads, replicas overlap around the clock. Of the 40 production
ReplicaSets whose containers together had learned for 24h, 39 had been watched at every time of
day, give or take one quarter-hour (the resolution of the analysis); one had a 4.75h gap. These are
exactly the workloads this proposal targets.

The stage workloads only run 06:00–20:00 UTC. No rule based on observation can watch them at 03:00.
The user can set the learning period to match their daily uptime instead (e.g. `12h`), which is
what they are already being advised to do.

**Summing** learned time across containers until it reaches 24h would complete both kinds, but
that loses the daily-cycle guarantee. In the same data, all 31 stage ReplicaSets whose summed
learning reached 24h had never been observed between 20:00 and 06:00: three days of 08:00–16:00 add
up to 24h without ever seeing 02:00. Here the night happened to be empty, but a summing rule can't
know that, and the same rule would complete an always-on workload that was watched only during
office hours.

## Goals

- Complete a profile once every time of day has been watched, whether by one container or several.
- Keep the guarantee of today's rule: each time of day observed at least once.
- Keep the decision in storage, with node-agent's learning timer and completion paths untouched.
- Need no new data sources: only the report chains storage already receives.
- Be backward compatible in both directions of version skew.

## Non-goals

- **Workloads that don't run at all hours** (scale-to-zero, office-hours-only). They keep today's
  rule. Users are expected to tune the learning period for them with the
  `kubescape.io/max-sniffing-time` pod label. See "Alternatives considered" for the vacancy-based
  extension that would cover them automatically, and why it isn't part of this proposal.
- Weekly or monthly behavior. Like today, only the daily cycle is guaranteed.
- Carrying learning across ReplicaSets (a new pod template means a new profile, as today).
- Changing what is learned, how profiles are merged, or how `failed` / `too_large` are handled.
- Changing completion after the full learning period, on a clean container exit (exit code 0), or
  on expiry.
- Changing behavior when a learning period **shorter** than 24h is in effect: the user has
  explicitly opted out of daily coverage, and the current single-container rule stays.

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

So for a long-running workload, which never exits cleanly, a profile completes only when one
container learns for the full `T`.

## Proposal

### Definitions

All times are UTC. A **time of day** is a point in `[00:00, 24:00)`; projecting an absolute time
onto the time of day means taking it modulo 24h. UTC avoids DST ambiguity; behavior that follows
local time still recurs at a fixed UTC time of day, except across a DST change.

A **profile key** is the existing `ContainerProfile` identity: workload instance (ReplicaSet,
StatefulSet, DaemonSet or Job) plus container name.

A **learning segment** is a maximal absolute-time interval during which one series (one watched
container) was verifiably watched (see "Learning segments").

For each profile key, storage keeps one **time-of-day set**:

```
Learned = projection onto the time of day of all learning segments of the profile key
```

It is a list of disjoint intervals over `[00:00, 24:00)` with millisecond endpoints, rounded
**inward** and never padded. It is a union over days, so it only grows. Past a cap (e.g. 512
intervals) its shortest intervals are dropped, which can only shrink it, so it stays conservative.

### Learning segments

- **Start.** If the series' first report links back to nothing, the segment starts at the series'
  **watch start** (`kubescape.io/learning-start`, see "node-agent changes"). After a broken link,
  it starts at the `ReportTimestamp` of the first report after the gap: data between a missing
  report and its successor is lost, so that time isn't credited.
- **Extension.** It is extended to the `ReportTimestamp` of every report that links to its
  predecessor. Heartbeat reports count (see "node-agent changes").
- **End.** A broken link closes it. When the series ends with the container's exit, it ends at the
  **verified runtime exit time** (`kubescape.io/container-exited-at`), not when node-agent handled
  the exit notification: after the exit nothing was running in that container, so that time must
  not count as watched. If no verified exit time is available, it ends at the last report sent
  before the exit.
- **No rounding.** Segment bounds are always node-agent's verified observation bounds. If the
  annotations are missing (older node-agent), the segment starts at the first report, and the
  uncertain start stays unlearned.

Because the collapsed row keeps only the chain's latest `ReportTimestamp`, storage persists, per
active series, the current segment's `[start, end]` and the last linked `ReportTimestamp` across
consolidation passes. Reports consolidated in different passes (at 00:10, 00:20 and 00:30, say)
therefore extend one 30-minute segment instead of producing three disjoint fragments.

### Completion rule

**Effective `T`.** Every chunk carries the effective learning period as the learning-period label.
storage records it per profile key the first time it sees it, and **freezes** it. Chunks of one
profile key normally agree, because the pod-label override is part of the pod template, and a new
template means a new profile key. They can disagree only if `maxSniffingTimePerContainer` changes
cluster-wide while a profile is learning. storage then keeps the **largest** value seen (more
evidence required, never less) and doesn't lower it. If no chunk of the profile key carries the
label, `T` is unknown and coverage completion doesn't apply to that profile.

For a profile key in status `learning` whose frozen effective `T` is at least 24h, storage
completes the profile when:

1. `Learned` is the whole day `[00:00, 24:00)`, **and**
2. at least one contributing series is a **received full observation**: completion `full`, **and**
   storage received and merged its initial report (the one whose `PreviousReportTimestamp` is
   empty). A `full` series whose initial report was lost doesn't count, because its start-up data
   never reached the profile. This ensures start-up behavior is in the profile, **and**
3. for `T > 24h`, the total learned wall-clock time (union of learning segments, not their sum) is
   at least `T`.

Completing means setting the consolidated profile to `completed`/`full`, derived from these
conditions rather than copied from any one series.

**Interaction with the existing completion paths.** These stay exactly as they are today:
- a series that learned for the full `T` and ends `completed` completes the profile as today;
- a clean exit (exit code 0) completes it with the series' own completion, as today;
- expiry (no report for `2 × maxSniffingTime`) finalizes it as `completed`/`partial`, as today.

Whichever path comes first completes the profile. Coverage only applies while the profile is
`learning`. It does not promote a `completed`/`partial` profile to `full`.

### Storage changes

1. **Learning segments.** Read `kubescape.io/learning-start`, `kubescape.io/container-exited-at`
   and the completion annotation from chunks, and build and persist segments as described above.
   A heartbeat chunk (`kubescape.io/heartbeat: "true"`) extends the segment like any linked
   report. Its row is recorded with `hasData=false` and its payload is not stored or merged, since
   it carries no new behavior.
2. **`Learned` set.** When a segment is extended, add its projection to the profile key's
   `Learned` set, kept in storage's own metadata table.
3. **Learned-time union (only for `T > 24h`).** Learning segments are also kept as a list of
   disjoint absolute `[start, end)` ranges. Overlapping or touching ranges are coalesced, so
   overlapping replicas aren't double-counted (00:00–12:00 plus 06:00–18:00 is 18h, not 24h). It is
   updated in the **same transaction** that deletes the consolidated time-series rows, so a crash
   can neither drop nor double-count an interval. Past a cap (e.g. 256 ranges) the shortest ranges
   are dropped, which can only under-count.
4. **Evaluation and completion.** storage evaluates the completion rule on **every consolidation
   pass**, for every profile key in `learning` with `T ≥ 24h`. That is where segments are extended
   and rows are deleted anyway, so the evidence is current and no extra trigger is needed. When the
   rule holds, storage sets the consolidated profile to `completed`/`full`. From then on:
   - `PreSave` rejects new reports, and node-agent stops each container still learning on its next
     report (`ObjectCompletedError`), as it does today when another replica completes first;
   - consolidation treats the profile as frozen and reclaims leftover rows, as today.
5. **Visibility** (annotations on the consolidated profile):
   - `kubescape.io/coverage-ratio`: the share of the day in `Learned`, e.g. `0.93`. This is the
     learning **progress** consumers should show for a profile in `learning`. "Elapsed time /
     learning period" is no longer meaningful once a profile can complete before `T` has elapsed;
   - `kubescape.io/completion-source: coverage` when this rule completed the profile.

No informers and no new RBAC are needed: everything comes from the report chains storage already
receives.

### node-agent changes

1. **Watch and exit bounds.**
   - Add `kubescape.io/learning-start` to every chunk: when node-agent started watching the
     container, RFC 3339 with sub-second precision.
   - On the final report after an exit, add `kubescape.io/container-exited-at`: the **verified**
     exit time from the container runtime (the CRI `ContainerStatus.finished_at`, nanosecond
     precision), when node-agent can obtain it.

   storage can't derive these: the first report arrives `initialDelay` (2 minutes by default) after
   the watch starts, and the final report arrives after the exit was handled, not at the exit.
2. **Heartbeat reports.** Today a chunk with nothing new is skipped. An idle container then extends
   its learning segment only when its next real report, or its final report, arrives, which can
   delay coverage by hours. node-agent sends a **heartbeat chunk** for a container that is still
   learning when no report has been sent for it for `learningHeartbeatInterval` (default
   `3 × updateDataPeriod`, i.e. 30m; `0` disables it). A multiple of the report period keeps the
   write cost low: at most 2 heartbeats per hour per idle learning container. Coverage then lags
   real time by at most the interval, which can only delay completion, never falsify it.
   - It links into the chain like any report (`PreviousReportTimestamp` set to the last sent report,
     a new `ReportTimestamp`), and carries `kubescape.io/heartbeat: "true"`.
   - Its spec follows the existing `stitchChunk` shape (`queue/containerprofile_split.go`): it
     carries only `SeccompProfile`, `ImageID` and `ImageTag`. Storage merges those three by
     assignment, so an empty spec would zero them on the merged profile in storage that doesn't
     know about heartbeats.

   It stops when learning stops (timer, exit, or `ObjectCompletedError`).

The learning timer, the final reports and the handling of `ObjectCompletedError` stay as they are.

### Why this keeps the daily-cycle guarantee

Take any time of day `τ` at which the workload runs a daily job. When storage completes a profile
through coverage, `τ ∈ Learned`: on some day, a container of this exact pod template was being
watched without a gap **at that exact time of day**. If the job ran then, its execs, file opens,
syscalls and network peers were recorded. Watching a neighboring moment is not enough: observing
02:00:00–02:00:10 says nothing about 02:00:45.

This is the same guarantee today's rule gives. A single container learning 24h watches each time
of day exactly once, on one day, and says nothing about runs on other days. The new rule relaxes
only one requirement: that the whole day be watched by the *same* container on *consecutive*
hours.

What it does **not** guarantee, same as today:

- behavior that does not recur daily;
- a job that runs only on one replica that wasn't being watched while another replica was.
  `Learned` asks that at every time of day *some* replica was watched, not every replica.
  Replicas of one pod template run the same code, but in-process leader election can confine a
  job to one replica. Open question below.

### Worked examples

- **Spot churn, replicas overlapping around the clock.** No single container lives 24h, but at
  every moment some replica is watched. Within roughly a day of wall-clock time, `Learned` covers
  the whole day, and the next consolidation pass completes the profile. The containers still
  learning stop on their next report. Today it often never completes.
- **Frequent rollouts.** Each rollout is a new pod template and a new profile key, so it starts
  from an empty `Learned`, as today. A ReplicaSet that lives at least a day completes within about
  a day, whatever its pods do. A ReplicaSet replaced every few hours still never completes, and the
  label is the tool for it.
- **Single long-lived container.** Its segment covers the whole day just as its 24h timer fires,
  so it completes as today, through whichever path runs first.
- **Nightly scale-to-zero (06:00–20:00 UTC up).** `Learned` never covers 20:00–06:00, so coverage
  never completes it. It behaves as today, and the user sets
  `kubescape.io/max-sniffing-time: "12h"` on its pod template. That loses nothing, because the
  workload doesn't run during the hours it skips.

## Guidance for workloads that don't run at all hours

Documentation should state plainly:

- Coverage completion needs the workload to be running (and watched) at every time of day. Always-on
  workloads get it automatically, however often their containers churn.
- Workloads that are scaled to zero on a schedule, or run only part of the day, should set
  `kubescape.io/max-sniffing-time` on their pod template to a value that fits within one daily run,
  e.g. `12h` for a 06:00–20:00 schedule. Setting the label changes the pod template, so it triggers
  one rollout and learning starts once more from there.
- `kubescape.io/coverage-ratio` shows how much of the day has been watched, which makes it easy to
  spot workloads stuck below `1.0` because they don't run around the clock.

## Compatibility and rollout

- **Feature flags**, default off at first: storage `containerProfileCoverage.enabled`, and node-agent
  `learningHeartbeatInterval` (0 = off). The new chunk annotations are always sent; they are
  harmless on their own.
- **Old node-agent + new storage:** no `learning-start` or `container-exited-at`. Segments start at
  the first report and end at the last report before an exit, so a few minutes around each
  container start stay unlearned. For always-on workloads with churn, other replicas usually cover
  those times on other days, so coverage still works; otherwise the profile completes as today.
  Nothing completes wrongly.
- **New node-agent + old storage:** the annotations are ignored. Heartbeat chunks are merged as
  ordinary rows. Because they carry `SeccompProfile`, `ImageID` and `ImageTag`, they change
  nothing, and they cost one small write per idle container per interval. Otherwise as today.
- **Downgrading storage:** profiles already completed stay completed (old storage honors
  `completed`/`full`). Coverage is no longer evaluated, and the rest behaves as today. No new
  status or protocol exists that old storage could misread.
- **Consumers that assume completion at start + `T`.** A profile can now complete before its
  learning period has elapsed. UIs and backends that compute learning progress as elapsed time over
  the learning period should switch to `kubescape.io/coverage-ratio`. Anything scheduled from an
  expected completion time should key off the profile's actual `completed` status instead.
- No change to the `ContainerProfile` spec, and no new status value. Only new annotations.

## Alternatives considered

1. **Sum learned time across containers.** Rejected: it doesn't preserve the daily cycle (see
   Motivation).
2. **Vacancy-based coverage.** An earlier revision of this proposal also credited times of day at
   which the workload provably *never ran*, so that scale-to-zero workloads could complete
   automatically: `Covered = Learned ∪ (Vacant \ Occupied)`. Storage can't learn that a workload
   didn't run from node-agent, because the runs that matter are the ones node-agent didn't see. So
   that design needed independent evidence from the Kubernetes API:
   - a cluster-wide pod informer, plus node and owner informers;
   - per-container-run records matched by container ID, with exact start times from node-agent;
   - unknown periods for informer gaps, not-`Ready` nodes and `restartCount` jumps;
   - a settle delay before trusting vacancy, and a reconciliation window for late evidence.

   Rejected for now: a large amount of machinery and watch load, to automate something that the
   owners of a scale-to-zero schedule can express with one label, at no loss. It can be revisited
   as an extension if label tuning proves impractical.
3. **node-agent shortens its learning timer from storage-published coverage.** Rejected: to be safe
   it needed a new final status, a freshness check, a completion cutoff with dated evidence, and a
   timer floor, and a container stopped early couldn't recover if storage then rejected the
   completion. Storage-side completion reuses the existing `ObjectCompletedError` path instead.
4. **Whole-slot or per-minute accounting.** Earlier drafts used quarter-hour slots or per-minute
   bits. Rejected: watching 10 seconds of a minute would mark the whole minute, and slot boundaries
   that don't line up with container starts never fill. Exact intervals avoid both.
5. **Only the `kubescape.io/max-sniffing-time` label.** It already exists and is right for
   workloads that don't run at all hours. For always-on workloads with churn it is the wrong
   trade-off: it gives up observing parts of the day that the workload does run in.

## Open questions

1. **Leader-elected jobs.** Should `Learned` require that *every* running replica was watched at a
   time of day? That needs to know which replicas were running, which brings back the evidence
   sources of Alternative 2.
2. **Carry-over across ReplicaSets with an identical image.** A rollout that changes only
   configuration or labels currently resets learning. Should `Learned` carry over when every
   container's image digest is unchanged?
3. **Upgrading expired profiles.** Should coverage be allowed to promote a profile that expiry
   finalized as `completed`/`partial` to `completed`/`full` once `Learned` becomes the whole day?
4. **Exit time availability.** Can node-agent always obtain the runtime's exit time (CRI
   `ContainerStatus.finished_at`) for exited containers? Without it, each exit leaves the time
   between the last report and the exit unlearned. That is safe, but slows coverage for workloads
   whose containers exit often.

## Testing

- **storage unit tests**:
  - segment construction across consolidation passes, split rows and broken links, heartbeat chunks
    (extending without merging), start at `learning-start`, end at `container-exited-at` or at the
    last report before the exit;
  - the `Learned` set: projection across midnight, exact millisecond bounds with no padding, the
    interval cap;
  - effective `T`: frozen on first sight, raised on conflicting labels, unknown without the label;
  - the received-full-observation condition, with a `full` series whose initial report is lost;
  - completion: the three conditions, `T < 24h` and non-`learning` profiles left alone,
    `completed`/`full` derivation, rejection of later reports via `PreSave`, and the existing
    paths winning when they come first;
  - chunks from old node-agents without the new annotations.
- **node-agent unit tests**: chunks carry `learning-start`; final reports after an exit carry
  `container-exited-at` when the runtime provides it; an idle learning container sends a linked
  heartbeat after `learningHeartbeatInterval`, in the stitch shape, and stops sending them once
  learning stops.
- **Component test**: a Deployment with replicas restarted on a short schedule so that no single
  container lives a full (shortened) day, while together they cover it. The profile must complete
  through coverage, and the running containers must stop learning afterwards. A second Deployment
  scaled to zero for part of the shortened day must **not** complete through coverage.
- **Telemetry**: add a `coverage` attribute (learned seconds of day) to the existing
  `container.profile.learning` span, so the effect can be measured in the field the same way the
  motivation data was.

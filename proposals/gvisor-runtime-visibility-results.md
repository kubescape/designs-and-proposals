# Results: gVisor runtime visibility experiment

- **Status:** Implemented as an opt-in source; results recorded
- **Author:** Daksh Pathak (<daksh.pathak.ug24@nsut.ac.in>)
- **Design:** [gVisor runtime visibility experiment](gvisor-runtime-visibility.md) ([#23](https://github.com/kubescape/designs-and-proposals/pull/23))
- **Related work:** [Agent Sandbox CRD posture scanning](agent-sandbox-crd-scanning.md)

This closes out the experiment proposed in #23. It records what was built,
what the live trials showed, how the original open questions came out, and
what is still unproven. Every number below comes from a trial report in the
linked pull request; none of them is a delivery guarantee.

## What was built

All of it is opt-in and off by default.

| Piece | Where | Pull request |
|---|---|---|
| Receiver for the `runsc` remote sink, with `container/start` decoding | node-agent | [#1024](https://github.com/kubescape/node-agent/pull/1024), [#1052](https://github.com/kubescape/node-agent/pull/1052) |
| Runtime-verified start observations, exported as OTLP `RuntimeObservation` | node-agent | [#1069](https://github.com/kubescape/node-agent/pull/1069) |
| Collector health and local event loss accounting (`RuntimeSourceHealth`) | node-agent | [#1072](https://github.com/kubescape/node-agent/pull/1072) |
| Inventory-verified container stop observations | node-agent | [#1094](https://github.com/kubescape/node-agent/pull/1094) |
| Helm deployment for start collection, then stop observations | helm-charts | [#994](https://github.com/kubescape/helm-charts/pull/994), [#1026](https://github.com/kubescape/helm-charts/pull/1026) |
| Mapping of the Agent Runtime Hardening controls to OWASP LLM Top 10 (2025) and MITRE ATLAS | regolibrary | [#940](https://github.com/kubescape/regolibrary/pull/940) |

Each exported record is marked `source=gvisor_trace` with
`identity_status=verified`, and its container, sandbox and Pod identity is
joined to the node's CRI and Kubernetes inventory before export. Events whose
identity cannot be joined stay out of workload findings. Arguments and working
directories reach the receiver, as the design warned, and are discarded before
anything is queued, logged or exported.

## What the trials showed

All trials ran on one self-managed Linux ARM64 VM (Ubuntu 24.04, K3s, runsc
`release-20261019.0` with systrap). None ran on managed GKE or x86_64.

| Question | Result | Report |
|---|---|---|
| Does host-side collection see inside a gVisor sandbox? | In two fresh ordinary/gVisor pairs, the ordinary Pod's profile contained the target exec and the gVisor Pod produced no matching profile. The cause was not isolated to kernel hooks. | [#1053](https://github.com/kubescape/node-agent/pull/1053) |
| Do starts arrive? | 20/20, 19/20, then 20/20 in the first runs. The 19/20 run is unexplained and was kept. A reproduced receiver defect (a timed read returning `EINTR` closed valid connections) was fixed, and two further runs gave 20/20 each. | [#1051](https://github.com/kubescape/node-agent/pull/1051), [fix](https://github.com/kubescape/node-agent/pull/1052) |
| Does the real node-agent export verified starts? | 20 of 20 repeated starts and 2 of 2 overlapping workloads exported; an excluded namespace, a forged unknown ID and an ordinary runtime exported none. | [#1069](https://github.com/kubescape/node-agent/pull/1069) |
| Is there a stop event? | There is no `container/stop` point. `sentry/task_exit` on the container's initial thread group maps to the CRI exit code. A child's exit and a closed socket are not stops. | [#1081](https://github.com/kubescape/node-agent/pull/1081) |
| Does the stop mapping hold up? | 48 of 48 terminations across three runs matched CRI exit codes. A Sentry kill and a receiver restart lose exits. | [#1091](https://github.com/kubescape/node-agent/pull/1091) |
| Does it work through Helm? | With `nodeAgent.gVisorStart.stopObservations=true`, 7 starts and 7 stops matched CRI exit codes (5, 7, 21, 22, 137, 137, 143). An exec session's exit produced no stop. Controls exported nothing. Turning the option off removed stops and kept starts. | [#1029](https://github.com/kubescape/helm-charts/pull/1029) |
| What does a `connect` record mean? | See below. | [#1095](https://github.com/kubescape/node-agent/pull/1095) |

### Network visibility

The egress study compared connect-point records with host conntrack and
packet captures.

- A blocking TCP `connect` that returns 0 corresponds to an established,
  delivered connection.
- A non-blocking `connect` returns `EINPROGRESS` whatever the outcome, so the
  record alone says nothing about success.
- A UDP `connect` sends nothing.
- UDP `sendto`, musl DNS lookups and traffic after a disconnect are not seen.
- A NetworkPolicy reject shows ICMP and creates no conntrack entry.

A connect record is therefore an attempt. It is not exported as egress, and
the source does not claim to cover network traffic.

## Answers to the open questions in #23

- **Can the target `runsc` build emit a trustworthy terminal event?** Not as a
  named container stop. The initial task's `sentry/task_exit`, matched by
  container ID, thread group and thread group start time and deduplicated
  against `exit_notify_parent`, matched CRI in every trial above.
- **Can identity be runtime-verified without the `runsc` control root?** Yes.
  The receiver joins claimed IDs to the local CRI and Kubernetes inventory and
  never touches the control root. Allowing the lookup after the container has
  exited, and while its Pod is deleting, was needed for stops.
- **Who owns the single `Default` trace session if another monitor exists?**
  Not tested. The trials used a node with no other monitor and never used
  `--force`.
- **Which managed GKE modes expose a supported way to configure the trace
  session and socket?** Not tested. Nothing here supports a GKE claim.
- **Which exporter path carries these events?** The existing OTLP log path,
  as `RuntimeObservation` and `RuntimeSourceHealth` records with their own
  source label.

## What this does not establish

- No per-session or per-actor attribution. A record identifies a container and
  its Pod, not the agent session inside it.
- No full in-sandbox enforcement or syscall monitoring.
- No complete delivery guarantee. The earlier 19/20 run was never explained,
  and a Sentry kill or a receiver restart loses stop events. The health
  counters report local loss; they cannot report events that never arrived.
- No managed GKE support, and no multi-monitor behaviour.
- One ARM64 installation. No x86_64, upgrade matrix or overhead measurement.
- The privacy checks cover a literal synthetic marker and a fixed export
  contract, not every possible encoding of arbitrary credentials.

## Remaining work

- Decide whether the unexplained 19/20 start miss needs a new, longer repeat.
- Test `Default` session ownership against another monitor, and a managed GKE
  node, before claiming either.
- Network egress remains a separate question. The connect point can support
  attempt-level records only, and any such event needs its own review.
- Whether the Agent Runtime ActorTemplate checks go further is a maintainer
  decision and is not part of this work.

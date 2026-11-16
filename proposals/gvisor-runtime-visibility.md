# Proposal: First runtime signals from gVisor sandboxes

- **Status:** Experiment completed; see [results](gvisor-runtime-visibility-results.md)
- **Author:** Daksh Pathak (<daksh.pathak.ug24@nsut.ac.in>)
- **Related work:** [Agent Sandbox CRD posture scanning](agent-sandbox-crd-scanning.md)
- **Related runtime design:** [designs-and-proposals#14](https://github.com/kubescape/designs-and-proposals/pull/14)
- **Initial implementation target:** [kubescape/node-agent](https://github.com/kubescape/node-agent)

This document is based on published gVisor interfaces and the existing
Kubescape architecture. It is kept as written when the experiment was
proposed. The trials were run afterwards, and their results are in
[gvisor-runtime-visibility-results.md](gvisor-runtime-visibility-results.md).

## Summary

Kubescape can assess the configuration of Agent Sandbox resources, but its
node-agent cannot infer a sandboxed application's system calls from host kernel
eBPF events in the same way it can for an ordinary container. gVisor handles
most application system calls in its userspace Sentry. A host event may describe
the Sentry or its network boundary without describing the application action
that caused it.

This proposal defines an experiment to find one useful, honest runtime signal
without changing gVisor's Sentry. The leading candidate is gVisor's existing
runtime monitoring interface: a `runsc` trace session can send selected trace
points to an external process over a Unix domain socket. We will first prove
that `container/start` can be received, safely decoded, and matched to a known
Kubernetes container. A stop signal and outbound network data remain separate
questions to test. A successful first integration would expose only events for
which their source and identity are verifiable.

The experiment starts on a self-managed Linux node where we can configure
`runsc`. Managed GKE Agent Sandbox is an intended deployment target, but its
runtime configuration and socket access must be checked separately before we
claim support there. A design result that establishes this limit is still useful;
it prevents a local prototype from being mistaken for a deployable GKE feature.

## Why this is separate from posture scanning

The earlier [CRD posture proposal](agent-sandbox-crd-scanning.md) deliberately
stopped at configuration and admission. Those checks answer whether the
sandbox was *defined* with the expected isolation, egress, and resource limits.
They cannot answer whether a particular sandbox started, whether a source
failed, or what traffic crossed the sandbox boundary. The two kinds of evidence
should remain distinct in Kubescape's reports.

### Relationship to the existing runtime proposal

[PR #14](https://github.com/kubescape/designs-and-proposals/pull/14) already
proposes gVisor's SecCheck remote sink for node-agent and reports a working
proof of concept. This proposal is a narrower experiment companion, not a
replacement design or a second receiver to deploy alongside it. It asks what
can be established first from a verified start event, what sensitive fields
the receiver must handle, and which stop and egress claims need separate
evidence. PR #14 considers a broader mapping of syscall trace points into
node-agent events; this proposal does not commit to that mapping.

If PR #14 becomes the implementation design, its proof of concept is the
starting point for the tests and privacy gates below. The two PRs should
converge before node-agent integration, especially because gVisor currently
permits only one `Default` trace session per sandbox. The earlier maintainer
approval on PR #14 was [dismissed for LFX timing](https://github.com/kubescape/designs-and-proposals/pull/14#issuecomment-5324063216),
not because its technical approach was rejected.

The runtime gap is specific. gVisor's [networking guide](https://gvisor.dev/docs/user_guide/networking/)
describes TCP state and packet assembly inside the Sentry's netstack. Its
[runtime monitoring guide](https://gvisor.dev/docs/user_guide/runtimemonitor/)
describes trace points produced there and streamed to an external monitoring
process. This is a better starting point for in-sandbox actions than labeling a
host syscall from the Sentry process as an application syscall.

## Goals

1. Reproduce the host eBPF visibility gap on one known gVisor sandbox and
   record exactly what the node-agent sees at the host boundary.
2. Compare the available signal sources using the same start, stop, and egress
   workload, with their setup costs and failure modes recorded.
3. Prove or reject a minimal `runsc` remote trace-point receiver for
   `container/start`, with a trustworthy container identity and observable
   message loss.
4. Define the smallest event contract needed to feed a verified signal into
   node-agent without presenting it as a kernel eBPF observation.
5. Establish whether that receiver can run on a self-managed Kubernetes node
   and whether equivalent access is available on managed GKE Agent Sandbox.

## Non-goals

- Per-session or per-actor attribution. A Kubernetes container or sandbox ID
  does not identify the individual agent session running inside it.
- Full syscall monitoring, enforcement, or a replacement for node-agent's
  existing eBPF sources.
- Treating a `connect` syscall, a packet, and a completed outbound request as
  equivalent events.
- Running a monitor inside the untrusted application sandbox.
- Changing gVisor's Sentry code or requiring a private gVisor fork.
- Enabling tracing for every gVisor workload by default.

## Signal candidates

| Source | What it could establish | Main limitation | Decision for first experiment |
|---|---|---|---|
| gVisor trace points via the `remote` sink | Selected Sentry events, including documented `container/start`; potentially syscall context | Requires `runsc` trace configuration and a listener reachable through a Unix socket; the protocol and input are untrusted | **Lead candidate** |
| `runsc --strace` and debug logs | Human-readable application syscall logs useful for comparison | Debug output is file-oriented and intended for troubleshooting; log rotation, volume, parsing stability, and early-event capture need proof | Diagnostic baseline, not the first production feed |
| `runsc` metrics | Sandbox presence, running state, counters, and source health | gVisor says these primarily describe its internals, not workload behavior; polling cannot prove every start/stop transition | Corroboration and health only |
| Host process and network events | Sentry process lifecycle and packets visible outside the sandbox | Does not reveal the guest process or establish which in-sandbox action caused a packet; attribution depends on runtime inventory | Fallback boundary signal |
| Seccomp or audit information at the host | Possible record of a host-side Sentry action | Host seccomp is not a direct stream of guest syscalls; no supported per-workload forwarding contract has been established | Research only unless the experiment finds a documented interface |

gVisor documents `--strace` in its [debugging guide](https://gvisor.dev/docs/user_guide/debugging/),
`runsc export-metrics` and `runsc metric-server` in its
[observability guide](https://gvisor.dev/docs/user_guide/observability/), and
the remote trace session in its
[SecCheck overview](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/README.md).
The observability guide explicitly distinguishes internal metrics from
workload monitoring. These are documented capabilities, not results we have
already measured in Kubescape.

## Proposed experiment

### 1. Establish a reproducible baseline

Use a Linux test node with containerd and a pinned `runsc` release. Record the
Linux, containerd, `runsc`, and node-agent versions and the OCI runtime root.
Run one ordinary container and one gVisor container with the same small
workload: start, make a single outbound connection to a controlled endpoint,
then exit. Capture the container runtime's ground-truth IDs and timestamps.

Run node-agent's current process and network collection against both. Compare
its events with the runtime's record. A host event is kept as a host event
unless the container mapping is demonstrated; Sentry PIDs are not relabeled as
guest PIDs. Repeat with two concurrent gVisor sandboxes to expose accidental
cross-sandbox attribution.

### 2. Test the gVisor remote trace sink

Start from gVisor's documented [remote sink protocol](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/sinks/remote/README.md)
and its example receiver. The sink uses a Unix `SOCK_SEQPACKET` connection and
sends protobuf messages after a handshake. One monitoring process can accept
connections from multiple sandboxes; the receiver must keep each connection's
state separate.

For the first run, configure only `container/start` and the minimum context
needed to evaluate identity. A session shaped like this is the test input;
the exact point fields and runtime flag support will be checked against the
pinned `runsc` build with `runsc trace metadata`:

```json
{
  "trace_session": {
    "name": "Default",
    "points": [{ "name": "container/start" }],
    "sinks": [{
      "name": "remote",
      "config": { "endpoint": "/run/kubescape/gvisor-events.sock" }
    }]
  }
}
```

Selecting only `container/start` does **not** mean the receiver sees only an
ID. In gVisor's [root and child container start paths](https://github.com/google/gvisor/blob/8a2c5049262ca84ea9c0981ac82eed02110d4ba7/runsc/boot/loader.go),
the event always includes `Args` and `Cwd`. Only `Env` is guarded by an
optional-field setting. Arguments and working directories can contain
credentials, so the receiver must treat the full incoming message as
sensitive even though the proposed output keeps only verified identity and
event metadata. If the requirement is to avoid receiving these fields at all,
`container/start` is not an acceptable source and the experiment must choose
a different point or signal.

Configure this through `runsc --pod-init-config` so the session is present
before the application starts. Attaching later with `runsc trace create` is a
useful recovery test but cannot prove that early events were captured. The
runtime must be able to reach the host socket path; a socket mounted into the
application container is neither required nor desirable. The node-agent or a
small receiver must listen before the sandbox starts.

We must not overwrite another monitor's trace session. The current gVisor
documentation describes a single session named `Default`. The experiment will
record what happens when another consumer already owns it and will not use
`--force` in normal operation. Restart recovery is allowed only when ownership
of the session is established.

### 3. Determine what “stop” means

The documented `container/start` point is the initial positive event. A remote
connection closing only proves that the trace transport ended. It may mean a
normal stop, a crashed Sentry, a disconnected receiver, or a configuration
change. The experiment will inspect the pinned build's trace metadata for a
usable terminal point and compare it with containerd/CRI lifecycle records.

Until that comparison is done, the proposed output is a verified start event
and a separate `source_disconnected` health event. A `sandbox_stopped` event
will be emitted only if its terminal source and identity can be verified. No
synthetic stop event will be inferred from an idle socket timeout.

### 4. Check network meaning separately

Issue a known outbound request to a controlled endpoint, a failed connection,
and a DNS lookup from the sandbox. Compare candidate trace points from
`runsc trace metadata`, `--strace` output, and host packet observations. The
result should state which of these was actually observed: connection attempt,
successful connection, DNS resolution, or packet egress. Record destination
address and port only when present in the selected event and avoid attributing
DNS names to packets by timing alone.

gVisor's netstack performs connection state management inside the Sentry, so a
host packet can be useful boundary evidence without being a guest `connect`
event. Network egress integration is a follow-up only if the observed signal
has stable semantics and verifiable sandbox identity.

## Proposed node-agent boundary

The first receiver should translate a narrow set of verified gVisor events
into an internal event type, then use node-agent's existing event handling and
export paths where their semantics fit. Before adding a general “external
event source” interface, implementation should inspect the current watcher
and tracer extension points. The interface belongs in a separate PR only if
the real receiver needs it.

Each accepted event should carry:

| Field | Meaning |
|---|---|
| `source` | `gvisor_trace`, so it cannot be mistaken for a host eBPF event |
| `kind` | A specific verified action, initially `container_started` |
| `observed_at` | Receiver timestamp; source timestamp retained separately if supplied |
| `node` | The node running the receiver |
| `sandbox_id` and `container_id` | Values from the trace and/or runtime, marked by origin and used only after cross-checking |
| `identity_status` | `verified` or `unresolved`; unresolved events stay out of workload-specific findings |
| `loss` | Per-connection drops or sequence gaps when the protocol exposes them |

The receiver must join trace identity to the local container runtime inventory
before attaching Kubernetes Pod metadata. A claimed container ID inside a
message is untrusted. If the join fails, the event can inform source health but
must not be attributed to a Pod, Sandbox CRD, WorkerPool, or actor. There is no
per-session identity in this design.

The first cut will expose an opt-in source with its own health counters. It
will not turn gVisor events into existing syscall events merely because they
have similar names. Any mapping into a profile or alert type requires a
separate review of event meaning and downstream assumptions.

## Security and failure behavior

The [remote protocol documentation](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/sinks/remote/README.md)
explicitly treats Sentry input as untrusted. The receiver therefore needs
hard limits on frame and decoded message size, bounded per-sandbox queues,
timeouts for handshake and reads, a maximum number of concurrent connections,
and validation before allocating from untrusted lengths. Unknown message
types or fields must not crash the listener. One noisy sandbox must not block
others.

The socket should have node-local ownership and restrictive permissions.
Runtime setup must not grant node-agent broad access to `runsc`'s control root
or to all sandbox sockets simply to receive events. If reconnection requires
control-root access, that permission is a separate deployment and security
decision, not an implicit part of the initial listener. The first trace
session does not request the optional environment field, extra context
fields, or network points. It nevertheless receives `Args` and `Cwd` in every
`container/start` message, including child containers. The receiver must
discard both before logging, queuing, retaining experiment artifacts, or
exporting an event. Error paths must not print or retain raw messages. Debug
and strace logs used for comparison must be checked and sanitized before
they are saved or shared. These controls limit retention and onward
disclosure; they do not prevent receipt over the socket.

The sink has retry and backoff settings. gVisor warns that excessive retries
can delay application execution. The experiment will test a slow consumer and
a disconnected listener, record dropped-message counters and startup impact,
and choose a bounded failure behavior before recommending any default. Source
failure must be visible as degraded telemetry, never interpreted as “nothing
happened.”

## Test matrix and decision gates

| Test | Evidence to record | Pass condition for a lifecycle prototype |
|---|---|---|
| Single sandbox start | Runtime ID, trace event, receiver log, timestamp | Trace arrives with an ID that can be joined to the runtime record |
| Twenty repeated starts | Expected versus received starts | 20/20 starts arrive in a healthy run with no incorrect attribution; a miss blocks lifecycle integration until explained |
| Two concurrent sandboxes | Distinct connections and IDs | No event crosses identities or queues |
| Normal exit and forced termination | Trace point, socket closure, CRI state | A stop is emitted only when a terminal source is independently verified |
| Receiver absent, slow, and restarted | Startup result, drop counter, retry delay, recovery | Bounded resource use and explicit degraded state; no silent claim of complete coverage |
| Malformed and oversized frames | Receiver errors and memory use | No crash, unbounded allocation, or effect on another connection |
| Synthetic secret in root and child container argv and cwd | Canary in the test fixture; receiver logs, errors, queued events, metrics, exports, and retained receiver output | The receiver necessarily ingests the raw fields, but the canary appears nowhere in its retained or forwarded output; the fixture itself contains a harmless canary by design |
| Controlled outbound, refused outbound, DNS | Trace fields and host packet comparison | Event name describes only the behavior actually observed |
| Kubernetes node and managed GKE | Required runtime flags, socket path, permissions | Deployment instructions name only environments where setup was demonstrated |

The experiment will retain a small, reproducible workload, sanitized receiver
output, version list, and a table of expected versus observed events. If the
remote sink cannot satisfy identity or reliability requirements, the design will
record that result and evaluate host boundary signals as a narrower fallback.
It will not build a general node-agent adapter on an unverified source.

## Deployment boundary

The initial prototype targets self-managed `runsc` with containerd. The
[gVisor setup guide](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/README.md)
documents setting `pod-init-config` for `containerd-shim-runsc-v1`. A later
Helm change could mount a node-local receiver socket and pass opt-in node-agent
configuration, but Helm alone cannot alter a host's containerd or managed
GKE `runsc` configuration. Those steps need separate, supported installation
instructions and a security review.

Google's [Agent Sandbox installation guide](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/how-install-agent-sandbox)
shows gVisor-backed Sandbox resources, while its
[GKE Sandbox guidance](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods)
describes managed node pools. Neither is evidence that users can set
`--pod-init-config` on those nodes. The GKE trial must answer that question;
the self-managed result must not be advertised as working on GKE by default.

## Suggested implementation sequence

1. Land this design, then publish the reproducible experiment record with the
   first local trial, including any negative results.
2. Add a narrowly scoped gVisor receiver and lifecycle proof of concept in
   `node-agent`, including hostile-input and identity tests. Reuse existing
   extension points when possible.
3. Route verified events through the appropriate node-agent exporter with a
   distinct source label and documented semantics.
4. Add reconnect, drop, and resource-limit behavior, then consider network
   egress only if the controlled tests establish a reliable meaning.
5. Add opt-in Helm wiring only for an environment whose runtime setup has
   been demonstrated. Keep the default disabled.

This sequence describes dependencies, not a promise that each line needs a
separate PR. The term's minimum runtime deliverable is a design backed by a
working proof of concept for one signal source. Network visibility and managed
GKE deployment are stretch outcomes if the evidence supports them.

## Open questions for review

- Can the target `runsc` build emit a trustworthy terminal event, or should
  stop come only from CRI lifecycle data?
- Can the receiver obtain a runtime-verified sandbox/container identity
  without access to the `runsc` control root?
- Who owns the single `Default` trace session if another monitor, such as
  Falco, is already configured on the node?
- Which managed GKE modes, if any, expose a supported way to configure the
  trace session and node-local socket?
- Which node-agent event or exporter path can carry a `container_started`
  event without implying that it came from eBPF or from an individual actor?

## References

- [gVisor runtime monitoring](https://gvisor.dev/docs/user_guide/runtimemonitor/)
- [gVisor SecCheck points, sessions, and sink configuration](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/README.md)
- [gVisor remote sink protocol and security considerations](https://github.com/google/gvisor/blob/master/pkg/sentry/seccheck/sinks/remote/README.md)
- [gVisor root and child `container/start` event construction](https://github.com/google/gvisor/blob/8a2c5049262ca84ea9c0981ac82eed02110d4ba7/runsc/boot/loader.go)
- [gVisor debugging and strace logs](https://gvisor.dev/docs/user_guide/debugging/)
- [gVisor observability and metrics](https://gvisor.dev/docs/user_guide/observability/)
- [gVisor networking](https://gvisor.dev/docs/user_guide/networking/)
- [Kubescape node-agent architecture](https://github.com/kubescape/node-agent/blob/main/README.md)

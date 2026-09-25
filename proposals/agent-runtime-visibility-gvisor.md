# Proposal: Runtime visibility for gVisor-sandboxed agents (node-agent)

## Summary

Give the node-agent eyes inside a gVisor sandbox. Today the node-agent's runtime
signals come from eBPF on the **host** kernel; a gVisor-isolated agent actor
services its syscalls in the **Sentry** (userspace) and is therefore a black box
to those probes. This proposal recovers per-actor runtime signals by consuming
gVisor's `seccheck` **remote sink** and normalizing each trace point into the
node-agent's existing event shape (`exec`, `network`, `open`, `fork`, `exit`,
`ptrace`, …), so existing runtime rules fire on sandboxed actors with no rule
changes. It is the runtime half (Part B) of kubescape/kubescape#2557 and the
companion to the static-posture work in
[#10](https://github.com/kubescape/designs-and-proposals/pull/10). A working
proof-of-concept is implemented and exercised end-to-end.

## Motivation

GKE Agent Sandbox runs model-generated, untrusted code *because* gVisor contains
it. Static posture (the "Agent Runtime Hardening" framework) verifies the actor
is configured to use gVisor and to deny egress — it says "the door is locked."
But once the actor is running, the thing we most want to watch — what that
untrusted code actually does — is exactly what the current host-kernel sensor
cannot see, because those syscalls never reach the host kernel. The issue states it directly: "gVisor intercepts syscalls in userspace (the
Sentry), so the node-agent's kernel-level eBPF hooks can't see inside a
gVisor-isolated actor the way they see a normal container." Runtime visibility is the "someone is climbing through the window
anyway" signal that posture alone cannot provide.

## Goals

- A node-agent signal source that observes gVisor-sandboxed actor behavior.
- Normalization of that signal into the node-agent event types the detection
  engine already consumes, so existing rules apply unchanged.
- Per-actor attribution (sandbox/container id → Kubernetes workload).
- Graceful, explicit handling of gVisor's single-session constraint.
- A working proof-of-concept for at least one signal source (the issue's minimum
  Part B deliverable), not a paper design.

## Non-goals

- No changes to the static posture controls or the framework (that is #10 and the
  regolibrary work, e.g. regolibrary#789).
- No claim to see *inside* model reasoning; this is OS-level behavior of the actor
  process.
- Not a full production node-agent integration in this proposal — the tracer seam
  and multi-consumer sink broker are described and scoped, not fully built.

## Candidate signal sources

Three ways to recover visibility; lead with one, keep a second as fallback.

**1. Sentry `seccheck` remote sink (recommended primary; independently
identified as the strongest candidate in
[node-agent#894](https://github.com/kubescape/node-agent/issues/894)).**
gVisor's first-class
trace subsystem streams structured protobuf trace points over a `SOCK_SEQPACKET`
UDS to an external monitor. Points carry rich context (container id, thread
id/tgid, process name, cwd, credentials, timestamps) and per-syscall detail (e.g.
`connect()` carries the destination sockaddr; `execve()` carries argv, resolved
binary path, even a binary SHA-256). Crucially, `runsc trace create` attaches a
session to an **already-running** sandbox — so a DaemonSet sensor can attach when
it starts, not only at pod launch. Cost is low (protobuf + an 8-byte header).

**2. `runsc --strace` / debug-log export (dev-only fallback).** Available with no
extra socket, but it is a text debug log, not a stable interface: lossy under
load, expensive, and format-unstable. Fine as a stop-gap; a poor production base.

**3. Host-boundary signals (complementary, always-available).** Some behavior
does cross the host boundary and is visible today: egress at the CNI/conntrack
layer (the same signal C-0301 governs statically) and the `runsc` Sentry
processes' own host lifecycle. Coarse, but requires no session — the natural
fallback when source 1 is unavailable, and an independent cross-check.

| Dimension | seccheck (1) | strace (2) | host-boundary (3) |
|---|---|---|---|
| Granularity | per-syscall, structured | per-syscall, text | coarse |
| Stable interface | handshake-versioned wire protocol; point schemas **not** stability-guaranteed | no | yes |
| Dynamic attach to running sandbox | yes | partial | n/a |
| Trusts the Sentry | yes | yes | no |
| Session contention | yes (single `Default`) | no | no |
| Production-suitable | **yes (primary)** | dev only | **yes (fallback)** |

A note on schema stability: the trace-point protos are not a stable public
API — Falco removed its gVisor engine in v0.44.0 (May 2026) rather than keep
absorbing this drift. The design mitigates it by isolating all decoding behind
bindings generated from vendored `.proto` files (with recorded provenance),
versioning the decode, and keeping the `runsc` metric-server endpoint as a
stable-but-coarse alternative. The connect-time handshake additionally lets the
collector negotiate versions and skip-and-count unknown point types rather than
fail.

## Proposal

Run a monitoring process on each node that owns the seccheck `Default` session,
decodes the point stream, normalizes it, and feeds the node-agent detection
pipeline. Normalization mapping:

| seccheck message | node-agent `EventType` |
|---|---|
| `EXECVE`, `SENTRY_EXEC` | `exec` |
| `CONNECT` / `SOCKET` / `BIND` / `ACCEPT` / `LISTEN` | `network` |
| `OPEN` | `open` |
| `SENTRY_CLONE` / `CLONE` / `FORK` | `fork` |
| `SENTRY_TASK_EXIT` / `EXIT_NOTIFY_PARENT` | `exit` |
| `PTRACE` | `ptrace` |
| `CONTAINER_START` | (metadata: sandbox/container correlation) |

Names are shorthand for the wire `MESSAGE_*` types (e.g. `MESSAGE_SYSCALL_CONNECT`,
`MESSAGE_CONTAINER_START`).

## The `Default`-session constraint (and fallback)

gVisor currently supports exactly one trace session **per sandbox**, and it
must be named `Default` — `pkg/sentry/seccheck/config.go` enforces
`only a single "Default" session is supported`, with the comment "When multiple
sessions are supported, this can be removed." Collection itself needs no
broker: the remote sink already allows "a single process to monitor all
sandboxes in the machine" (sinks/remote README). The scarce resource is each
sandbox's one session — a downstream sharing problem, not a collection problem.
Consequences and mitigations:

1. **Contention** — if the platform or another tool already installed `Default`,
   node-agent's attach fails; it must detect this rather than assume ownership.
2. **Graceful degradation** — on "session already exists," fall back to
   host-boundary signals (source 3) for that sandbox, log an actionable event,
   and raise a posture finding that runtime visibility is unavailable on the node
   (itself security-relevant).
3. **Sharing** — where node-agent can own the session, run a per-node **sink
   broker**: node-agent holds the one session and re-publishes the normalized
   stream to multiple internal consumers, so "one session" ≠ "one consumer."
4. **Upstream** — gVisor's own comments anticipate multi-session support; an
   upstream contribution to name/multiplex sessions removes the constraint at the
   root. Worth a gVisor issue referencing this use case.

## Proof of concept (implemented)

A companion PoC implements source 1 end-to-end:

- A **collector** binds the `SOCK_SEQPACKET` UDS (Go standard library only),
  performs the seccheck handshake, decodes points against gVisor's **real**
  protobuf schemas, normalizes them, and emits JSON.
- A **cluster-free E2E** (`make e2e`): a faithful `fakesentry` replays a realistic
  agent-sandbox session over the **real** wire protocol (handshake + header +
  gVisor protobufs). The collector decodes all points and maps them to node-agent
  events — including recovering an external egress destination from a raw
  `connect()` sockaddr and attributing every event to the sandbox container id,
  i.e. signals invisible to host eBPF.
- A **real-gVisor path** (`runsc trace create` against a running
  `--runtime=runsc` container) using the identical collector code.
- Unit tests over decode + mapping.

- Prior-art note: gVisor ships `tools/tracereplay` for generic save/replay of
  remote-sink sessions. `fakesentry` differs in what it *asserts* — the
  seccheck-to-node-agent normalization contract (wire message to decode to expected
  exec/network/open semantics) — and `tracereplay` can complement it later by
  capturing real-session fixtures.

No Kubernetes cluster is required to see it work; the real-gVisor path needs only
Docker + runsc on any Linux box (no GKE).

## OWASP LLM / MITRE ATLAS mapping

Runtime visibility supports detection of, and posture controls mitigate,
the following (OWASP Top 10 for LLM Applications 2025; MITRE ATLAS):

| Control / capability | OWASP LLM (2025) | MITRE ATLAS |
|---|---|---|
| C-0297 hardened runtime class (isolation) | LLM06 Excessive Agency; LLM05 Improper Output Handling | Execution / Defense-Evasion tactics; limits blast radius of AML.T0053 (AI Agent Tool Invocation) |
| C-0301 egress default-deny | LLM02 Sensitive Information Disclosure; LLM06 Excessive Agency | Exfiltration tactic; AML.T0024 (Exfiltration via AI Inference API); constrains exfiltration reached via AML.T0053 (AI Agent Tool Invocation) |
| Runtime visibility (this proposal) | LLM06 Excessive Agency; LLM02 Sensitive Information Disclosure | Detection across Execution / Exfiltration; observes AML.T0051 (LLM Prompt Injection) downstream effects |

(Technique IDs are from the current ATLAS matrix; tactic-level mapping is given
where a technique does not map 1:1. Refinement welcome.)

## Security considerations

The collector sits on a trust boundary: the *channel* is Sentry-side, but the
message *content* originates in the sandboxed workload — the adversary in
gVisor's threat model. The remote-sink documentation is explicit: "the
monitoring process must validate and never trust input received from the
Sentry because it can be controlled by a malicious user. All fields must have
hard coded size limits."

- **Hard-coded field size caps** on every string/bytes field (argv, paths,
  process names) — implemented in the PoC collector.
- **Unknown points/versions tolerated:** the connect-time handshake negotiates
  compatibility; unknown message types are skipped and counted, never fatal.
- **Dropped-message accounting:** the wire header's control fields carry drop
  counts; the collector surfaces them as metrics so shedding is observable,
  not silent.
- **Backpressure without touching the actor's hot path:** bounded
  per-connection buffers with shed-and-count behavior; the sensor is
  out-of-band by construction and never blocks the Sentry.
- **Per-sandbox connection isolation:** one hostile or excessively chatty
  actor must not starve other sandboxes' streams.
- **Sensitive payloads:** full-argv capture and execve binary hashing are
  opt-in; recovered fields should be maskable consistent with PII policy
  before persistence or export.

## node-agent integration path

1. Wrap the collector as a node-agent `containerwatcher` tracer, started when a
   gVisor runtime class is detected on the node.
2. Translate the normalized event into node-agent's concrete event structs and
   feed the existing `eventreporters` / rule-manager path.
3. Correlate the seccheck `container_id` to the pod/actor via node-agent's
   existing container→workload mapping, seeded by `CONTAINER_START`.
4. Derive higher-level signals where cheap: sensitive-file access from `open`
   paths, and domain enrichment by correlating recovered `connect()`
   destinations (a sockaddr carries IP/port/family, not the originating
   hostname) with node-agent's existing DNS tracer and `pkg/dnsmanager`
   address-to-domain cache, where that relationship is observable. Query-level
   analytics (DGA entropy, DNS exfiltration) require the DNS trace points
   themselves and are future work.

## Alternatives considered

- **Wait for eBPF-in-gVisor.** Rejected: no host-kernel eBPF path exists for
  Sentry-serviced syscalls; seccheck is the supported mechanism today.
- **Parse the strace debug log.** Rejected as a production base (source 2 above).
- **Host-boundary signals only.** Insufficient granularity for per-actor
  detection; retained as the fallback, not the primary.

## Open questions

- Session-ownership policy vs. GKE-managed tooling that may want `Default`
  (needs a real GKE Agent Sandbox cluster to settle; the PoC needs none).
- Default point/field selection vs. cost (e.g. execve binary hashing is powerful
  but expensive — likely opt-in).
- Standalone node-agent capability vs. folding under the CADR design (mirrors the
  same open question in #10). Recommendation: land standalone, reference from CADR.

## References

- Issue: kubescape/kubescape#2557 (Part B)
- Companion static-posture proposal: designs-and-proposals#10
- gVisor seccheck: `pkg/sentry/seccheck/` (config.go session restriction); `sinks/remote` README (untrusted-input guidance); `tools/tracereplay`
- Prior art: kubescape/node-agent#894 — identifies seccheck and the
  single-session limitation (note-only); this proposal and its PoC are the
  executable design over that observation.
- Companion PoC: [gvisor-visibility-poc](https://github.com/yellow-forrest/gvisor-visibility-poc) (collector, cluster-free E2E, runsc path)

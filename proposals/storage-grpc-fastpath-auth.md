# Proposal: gRPC Fast-Path Authentication and Authorization for kubescape/storage

**Status:** Proposed
**Author:** Matthias Bertschy (matthias.bertschy@gmail.com)
**Date:** 2026-08-31

## Summary

`node-agent` currently sends `ContainerProfile` and SBOM (`SBOMSyft`/`SBOMSyftFiltered`) data into
`kubescape/storage` exclusively through the kube-apiserver aggregation hop (REST). This proposal
adds an additive gRPC ingestion path that lets `node-agent` send the same data directly to
`kubescape/storage`, skipping that hop for efficiency — reusing the existing `kubescape/backend`
[`StorageService` proto](https://github.com/kubescape/backend/blob/main/pkg/client/v1/proto/storage_service.proto)
wire format unchanged, since it already defines the right shape of RPCs
(`SendContainerProfile`, `PutSBOMStream`, etc.) for this kind of data.

**This proposal covers only the authentication, authorization, and admission model for the new
listener** — how `kubescape/storage` decides whether to trust and accept a write arriving over
gRPC instead of REST. It does not propose any change to the `StorageService` proto itself, any
change to `node-agent`, or any change to how `ContainerProfile`/SBOM objects are keyed in storage.

## Motivation

- The kube-apiserver aggregation hop adds REST/JSON marshaling and an extra network hop for what is
  often high-volume, latency-sensitive data (per-container runtime profiles, SBOMs).
- `kubescape/backend` already defines a gRPC service for exactly this data shape, used today for
  cloud upload. Reusing that same wire format for in-cluster delivery avoids inventing a second
  protocol and a second client implementation in `node-agent`.
- The existing REST aggregated API must remain the interface for kubectl, controllers, and any
  other consumer doing normal Kubernetes operations against these resources — this is an additive
  path, not a replacement.

## Goals

- Add a gRPC listener to `kubescape/storage` that accepts writes in the existing
  `StorageService` wire format.
- Authenticate `node-agent` instances without standing up new PKI infrastructure.
- Authorize writes with real, live RBAC parity to the REST path — never weaker than what REST
  enforces today, including future RBAC changes.
- Preserve every write-path safety guarantee the REST path already has: per-key locking,
  key derivation, and namespace-lifecycle admission.

## Non-goals

- No changes to the `kubescape/backend` `StorageService` proto or wire format.
- No changes to `node-agent` — this proposal is scoped to the storage-side listener only; when and
  how `node-agent` adopts the fast path is a separate decision.
- No change to how `ContainerProfile`/SBOM objects are keyed in storage. As a direct consequence,
  this proposal does **not** deliver key-level, per-node write scoping — see "What this does not
  guarantee" below.
- No design for open-ended per-node rate limiting — only a basic per-object size cap, matching
  what REST already enforces, is in scope.

## Background: current state

- `kubescape/storage` is an aggregated Kubernetes APIServer. It runs with no in-process authorizer
  (`RecommendedOptions.Authorization` is nil) — RBAC for these resources is enforced entirely
  upstream, by kube-apiserver, before the request is proxied to this component.
- `ContainerProfile` and both SBOM types are namespace-scoped custom resources. Neither the object
  names nor their metadata carry a pod identifier — profiles are workload-derived, SBOMs are
  image-derived.
- The default storage key for a `ContainerProfile` (Kubernetes-mode deployments) has no node/host
  segment. Two nodes running the same workload can legitimately write toward the same key today.
  Introducing node-level scoping at the key level would require a storage-layout migration, which
  is explicitly out of scope here.

## Proposed design

### Identity

`node-agent` authenticates using its existing kubelet-rotated, audience-bound projected
ServiceAccount token (a dedicated audience, e.g. `storage-grpc-fastpath`), validated by a
purpose-built `TokenReview`-based authenticator in `kubescape/storage`. This is a new authenticator
instance, not a reuse of the REST path's delegating authenticator — that one has an
always-succeeding anonymous fallback and doesn't check token audience, both of which would defeat
the purpose here.

A client-certificate identity scheme (short-lived certs issued via the built-in Kubernetes CSR API,
`certificates.k8s.io/v1`, `kube-apiserver-client` signer — no new CA required) is a viable
alternative that removes the token path's dependency on kube-apiserver at connection time. This
proposal defers it to a later phase, conditional on real measurement showing that dependency is a
problem in practice (e.g. during a `node-agent` DaemonSet rollout, when every instance reconnects
at once).

### Authorization

Every write is authorized with a live, per-write `SubjectAccessReview`-backed check — using
`k8s.io/apiserver`'s built-in `authorizerfactory.DelegatingAuthorizerConfig`, the same mechanism
kube-apiserver's own RBAC authorizer is built from — scoped to the resource, namespace, and verb of
the specific write. This gives the gRPC path the same live RBAC guarantee REST has today, including
any future RBAC change, rather than a static, go-stale-able allowlist.

### Write path

The gRPC handler does not reconstruct storage keys or call storage internals directly. It calls
into the same `rest.Storage` object `kubescape/storage` already constructs per resource for the
REST path. This makes identical key derivation, per-key locking, and post-write processing
guaranteed by construction — there is exactly one write path per resource type, reachable by two
transports, not two implementations that have to be kept in sync.

A small, shared helper (covered by a differential test against the REST HTTP path) replicates the
few request-handling steps that live above that shared object in REST's HTTP layer: namespace-
context injection, the existing `NamespaceLifecycle` admission check, and clearing of any
caller-supplied system metadata fields — so the gRPC path can't silently skip a protection the REST
path has.

### Optional per-write attribution

`node-agent` may additionally stamp its own node identity on a `ContainerProfile` write; the server
verifies the stamp matches the authenticated connection's identity. This catches a compromised or
misconfigured node lying about which node produced a write — useful for downstream per-node
analytics or auditing — but it is explicitly **not** a namespace or workload ownership boundary;
the authorization check above is what enforces that. Whether this is worth building for the first
version is left open (see Open Questions).

### What this does not guarantee

Because the default `ContainerProfile` storage key carries no node segment, this design cannot
restrict *which specific namespace or workload* a given node may write to beyond whatever RBAC
already grants `node-agent`'s ServiceAccount on the REST path today. If that RBAC grant is broad
today, this proposal does not narrow it — it only ensures the new gRPC entry point doesn't fall
*below* that existing bar. Real key-level, per-node scoping would require the storage-layout
migration this proposal explicitly does not take on.

## Rollout

1. **Phase 0 — decide, don't code.** Confirm `node-agent`'s current RBAC scope for these resource
   types, and decide whether the optional attribution check (above) is worth building given that
   baseline.
2. **Phase 1.** Ship authentication, RBAC-equivalent authorization, and admission parity with REST,
   behind a feature flag defaulting off.
3. **Phase 2 — measure before trusting.** Instrument connection-time and per-write latency/failure
   rates under real conditions: a `node-agent` DaemonSet rollout, and, if it can be done safely, a
   simulated kube-apiserver brownout.
4. **Phase 3 — conditional.** Promote authentication to the client-certificate scheme if Phase 2
   shows the token path's dependency on kube-apiserver is measurably a problem.

## Alternatives considered

- **A static allowlist (e.g. a hardcoded ServiceAccount identity) instead of a live authorization
  check.** Rejected: it goes stale the moment RBAC changes and would make the gRPC path weaker than
  REST, which we treat as a hard requirement, not a nice-to-have.
- **Reconstructing storage keys independently inside the gRPC handler**, to avoid depending on the
  REST-facing storage object. Rejected: two independent key-derivation implementations risk
  silently diverging, which would defeat the per-key write locking the storage layer relies on for
  correctness.
- **Replaying the full in-process HTTP handler chain** (synthesizing an `http.Request` and calling
  into the existing HTTP handler) instead of calling the shared storage object directly. This would
  eliminate any need to separately replicate handler-layer behavior, but reintroduces the
  serialization/HTTP overhead the fast path exists to avoid — rejected as self-defeating.
- **Client-certificate identity from the start**, instead of a token-based Phase 1. Not rejected,
  deferred: it removes a per-connection dependency on kube-apiserver, but requires new CSR-approval
  operational tooling that isn't justified without evidence the token path's dependency is actually
  a problem at the scale this is meant to help with.

## Open questions

- What is `node-agent`'s actual RBAC scope for `ContainerProfile`/SBOM writes today (any namespace,
  or already narrower)? This determines how much additional value the optional attribution check
  provides.
- Should the attribution check ship in Phase 1, or wait until there's a concrete downstream consumer
  for "which node actually produced this write"?
- What `SubjectAccessReview` result caching duration is acceptable — trading control-plane call
  volume against how quickly an RBAC revocation takes effect on this path?
- Does the minimum supported Kubernetes version reliably populate the node-identity claim on
  projected service account tokens that the optional attribution check would rely on?
- Does the size-cap/backpressure mechanism this component already applies to REST writes need to be
  reimplemented for the gRPC path, or can it be shared?

## References

- [`kubescape/backend` `StorageService` proto](https://github.com/kubescape/backend/blob/main/pkg/client/v1/proto/storage_service.proto) — the wire format this proposal reuses unchanged.

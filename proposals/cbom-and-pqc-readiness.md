# Proposal: Cryptographic Bill of Materials (CBOM) and post-quantum cryptography readiness

- **Status:** Proposed
- **Author:** Ben Hirschberg ([@slashben](https://github.com/slashben))
- **Date:** 2026-11-05
- **Revised:** 2026-11-08, after review by [@matthyx](https://github.com/matthyx) and
  [@entlein](https://github.com/entlein): the CBOM and the crypto profile are now separate objects for capability
  and behavior, connection states are defined, and the kernel TLS statement is corrected
- **Scope:** [`regolibrary`](https://github.com/kubescape/regolibrary) (new controls and framework),
  [`kubescape`](https://github.com/kubescape/kubescape) (CLI output, host-sensor data),
  [`kubevuln`](https://github.com/kubescape/kubevuln) (image crypto cataloging, CBOM generation),
  [`storage`](https://github.com/kubescape/storage) (new CBOM and crypto-profile kinds),
  [`node-agent`](https://github.com/kubescape/node-agent) (handshake capture),
  [`helm-charts`](https://github.com/kubescape/helm-charts) (feature flags)
- **Related:** [`verified-oci-sbom-reuse.md`](verified-oci-sbom-reuse.md) (SBOM pipeline this builds on),
  [`vex-ingestion.md`](vex-ingestion.md) (same "suppress what is not reachable" pattern applied to CVEs),
  [`evidence-of-finding.md`](evidence-of-finding.md) (how findings carry evidence)

## 1. Summary

**In short:** Governments now require organizations to inventory where quantum-vulnerable cryptography is used and to
migrate it on fixed deadlines. The first required step everywhere is an automated cryptographic inventory, and in the
US that inventory must feed a Cryptographic Bill of Materials (CBOM). Kubescape already has the three surfaces such an
inventory needs: the cluster API (certificates, control-plane flags, mesh configuration), image contents (via the
Syft SBOM pipeline in kubevuln) and runtime observation (node-agent's eBPF layer). No open-source Kubernetes-native tool
combines them today. This proposal adds post-quantum cryptography (PQC) readiness to Kubescape in three phases:
posture controls, CBOM output and runtime-verified crypto profiles. The third phase is the differentiator. It turns
"RSA is present in this image" into "this pod negotiated a classical key exchange with that peer in production",
which is the same noise-reduction pattern Kubescape already uses for CVEs with application profiles.

**Technically:**

1. **Phase 1, posture.** New Rego controls in regolibrary that check control-plane and client Go versions, TLS
   minimum version, service-mesh PQC settings and the algorithms of TLS Secrets, webhook CA bundles and the
   service-account signing key. A `pqc-readiness` framework maps them to NIST IR 8547, OMB M-26-15, the EU roadmap
   and the UK NCSC timeline. Image packages are mapped to PQC capability by version (OpenSSL, Go, JDK, liboqs).
2. **Phase 2, CBOM.** kubevuln gains a crypto cataloger that parses certificates, keys and crypto configuration from
   image filesystems and emits CycloneDX `cryptographic-asset` components. Cluster-level assets come from the API.
   Storage persists the result as a new kind next to `SBOMSyft`. The CBOM records what an image *can* do. Detectors
   are upstreamed to Syft or embedded from PQCA's cbomkit-theia rather than forked privately.
3. **Phase 3, runtime verification.** node-agent parses TLS ClientHello/ServerHello and SSH KEXINIT messages on the
   wire, which are plaintext even in TLS 1.3, and aggregates offered and selected key-exchange groups, protocol
   versions and cipher suites per workload and port into a crypto profile. The crypto profile records what a workload
   *does*. Joining the two gives "relevant crypto", the crypto that is both present and used, and classical-only
   sessions crossing a trust boundary become findings.

## 2. Motivation

### The need is real and dated

The regulatory picture changed in 2026. The list below is what drives buyers; the dates set the urgency.

| Jurisdiction | Instrument | What it asks for |
|---|---|---|
| US | EO 14412 (June 22, 2026) | Inventory of high-value assets; PQC key establishment by Dec 31, 2030 and signatures by Dec 31, 2031; CISA and NIST to define minimum CBOM elements "for automated assessment" within 270 days (around March 2027); FAR rule extending PQC FIPS compliance to contractors by end of 2030 |
| US | OMB M-26-15 (June 24, 2026) | Migration plan within 120 days stating "methodologies and automated tools used for the cryptographic inventory"; the inventory "should populate a central Cryptographic Bill of Materials (CBOM)" |
| US | NIST IR 8547 final (Oct 27, 2026) | RSA, ECDSA, EdDSA, DH, ECDH and MQV deprecated after 2030 at all strengths and disallowed after 2035 |
| US | CNSA 2.0 | National security systems: new acquisitions compliant from Jan 1, 2027; exclusive PQC use for networking equipment by 2030, servers and cloud services by 2033, everything by 2035 |
| EU | Coordinated Implementation Roadmap v1.1 (June 2025) | National roadmaps and inventory by end of 2026; high-risk use cases and critical infrastructure migrated by end of 2030; everything feasible by 2035; hybrid schemes recommended |
| EU | DORA RTS (EU) 2024/1774 | Crypto-agility provisions (Art. 6(4), recital 9 names quantum), a certificate register (Art. 7(4)), tracking of third-party and open-source libraries (Art. 10(2)(d)) |
| EU | NIS2 amendment proposal COM(2026) 13 | PQC transition policy in national cybersecurity strategies |
| UK | NCSC guidance (Mar 2025) | Discovery and migration plan by 2028; high-priority migrations by 2031; full migration by 2035 |
| DE | BMI/BMDS/BSI joint guideline (Oct 2026) | First inventory by end of 2026; high-risk cases by end of 2030 |

Only the US rules name a CBOM explicitly. The EU, UK and DORA require inventory, discovery and crypto-agility without
naming the format. In practice buyers in all of these markets ask for the same four things:

- an inventory of where RSA and ECC are used, with an owner;
- evidence of what is negotiated on the wire (for example X25519MLKEM768 versus plain X25519);
- a machine-readable export, for which CycloneDX CBOM is the de facto format;
- a prioritized migration backlog.

Industry is moving faster than regulation. Microsoft moved its Quantum Safe Program target from 2033 to 2029 in
June 2026. Go 1.24, Kubernetes v1.33, OpenSSL 3.5, Linkerd 2.19 and Istio 1.27 all ship hybrid PQC key exchange, so
the question for a cluster owner is no longer "can I" but "where am I not".

### The competitive window is open but closing

- **Wiz** sells "Wiz for PQC Readiness" (GA): crypto inventory from cloud config, host config files, code, runtime
  sensor and DSPM, exported as CBOM.
- **Trivy 0.76** ships an experimental `--scanners crypto` that inventories X.509 certificates and keys in images
  as CycloneDX 1.7 cryptographic assets. Its own design discussion says it reports presence, "not that the asset is
  actively used".
- **PQCA cbomkit-theia** (Linux Foundation, Go, Apache-2.0) scans images and directories for certificates, keys,
  Java security config, OpenSSL config and secrets.
- **cdxgen 13.5** adds certificate identification to CBOMs.
- **Syft** has only an open proposal (issue #5190) for X.509 inventory in CycloneDX `cryptoProperties`.
- **SandboxAQ AQtive Guard** and **Keyfactor AgileSec** correlate network, application and filesystem analyzers, but
  they are proprietary and enterprise-wide, not Kubernetes-native.
- **Linkerd** exports negotiated TLS parameters as metrics, and **Riptides** reports the negotiated group per
  connection, but only for their own data plane.

Nobody owns the open, Kubernetes-native, workload-correlated position. Kubescape is one of the few projects that can
take it with existing machinery.

### Why Kubescape fits

| Kubescape component | PQC role | Effort | Risk |
|---|---|---|---|
| regolibrary controls, CLI, host-sensor | API and control-plane posture, version-skew checks | Low | Low |
| kubevuln (Syft) | Library capability mapping, CBOM generation | Medium | Medium (noise) |
| node-agent (eBPF) | Handshake capture, per-workload crypto profile | Medium-high | Medium (kernel and Go variance, overhead) |
| storage | Persist CBOM and crypto profile kinds | Low | Low |

The project also has a credibility reason to do this. Control C-0142 today recommends `aescbc` as an acceptable
encryption-at-rest provider, which the Kubernetes documentation rates "Weak". Fixing that is not PQC work, but a
cryptography framework that ships with a stale recommendation will not be trusted.

## 3. Words used in this proposal

| Term | Meaning |
|---|---|
| **PQC** | Post-quantum cryptography: algorithms believed secure against quantum computers. NIST standardized ML-KEM (FIPS 203, key establishment), ML-DSA (FIPS 204) and SLH-DSA (FIPS 205, both signatures) in August 2024. |
| **Quantum-vulnerable** | Asymmetric primitives broken by Shor's algorithm: RSA, ECDSA, EdDSA, DH, ECDH. Symmetric ciphers and hashes face only Grover's quadratic speedup and are low priority. |
| **Hybrid key exchange** | A classical and a PQC key exchange combined, such as X25519MLKEM768 (TLS group 0x11EC). This is the goal state through 2030, not a weakness. |
| **Harvest now, decrypt later** | Recording encrypted traffic today to decrypt once a quantum computer exists. This makes key exchange urgent; signatures cannot be abused retroactively. |
| **CBOM** | Cryptographic Bill of Materials. In CycloneDX 1.6+, components of type `cryptographic-asset` with `cryptoProperties` describing an algorithm, certificate, protocol or key material, linked to software components by dependencies. In this proposal the CBOM describes capability: what an image contains and can negotiate. |
| **Crypto profile** | The per-workload runtime record of observed handshakes proposed here. It describes behavior, analogous to the existing container profile for syscalls, files and network. |
| **Relevant crypto** | The join of the two: CBOM assets that the crypto profile shows in use. The crypto counterpart of the relevant (filtered) SBOM. |
| **Effective state** | The protocol, group and cipher suite a connection actually ended up with after any retries or fallbacks. |

## 4. Goals and non-goals

**Goals**

- Give a cluster owner a prioritized answer to "where does this cluster still rely on quantum-vulnerable cryptography,
  and which of those uses are exposed to harvest-now-decrypt-later".
- Produce a CycloneDX CBOM per image and per cluster that maps to the CISA/NIST minimum elements once they are
  published.
- Correlate static findings with runtime evidence so that users see "negotiated in production", not "present in a
  file".
- Keep every detector vendor-neutral and upstreamable (Syft, cbomkit-theia, PQCA).

**Non-goals (for the versions described here)**

- Decrypting or inspecting application payloads. Only handshake metadata is parsed.
- Source-code scanning for crypto API usage (sonar-cryptography already covers Java and Python).
- A "quantum risk score". Buyers pay for inventory and evidence.
- Enforcement. Blocking classical handshakes is a later question for the prevention track.
- In-process crypto that never touches a socket (JWT signing, envelope encryption, KMS SDK calls). Attributed through
  the image CBOM only.

### Design principle: capability and behavior are separate objects

Kubescape already separates what an image contains from what a workload does. kubevuln produces an SBOM per image,
node-agent learns a container profile per workload, and joining them gives the relevant SBOM that CVE prioritization
uses. This proposal follows the same split for cryptography:

| | Software (today) | Cryptography (proposed) |
|---|---|---|
| Capability: what is in the image | SBOM from kubevuln | **`CryptoBOM`** from kubevuln (Phase 2) |
| Behavior: what the workload does | Container profile from node-agent | **`CryptoProfile`** from node-agent (Phase 3) |
| Join | Relevant SBOM | **Relevant crypto** |

The objects differ in key, lifecycle and consumer:

- **`CryptoBOM`** is keyed by image digest and shared by every workload that runs the image, like `SBOMSyft`. It is
  static, has no learning period and is the source for CycloneDX export.
- **`CryptoProfile`** is keyed by workload and port. It has a learning period, it changes as traffic changes and it
  carries connection states that a static document cannot express (section 7).
- **Relevant crypto** is computed, not stored as a flag on the CBOM. The same image can run in two workloads where only
  one of them negotiates classical key exchange, so "observed" is a property of the workload, not of the image.

A third layer sits between them. What a workload negotiates also depends on deployment configuration that is not in
the image: `GODEBUG` and other environment variables, mounted `openssl.cnf`, mesh settings. The Phase 1 controls cover
that layer. The CBOM therefore states what an image is able to negotiate. It does not predict what it will negotiate.

## 5. Phase 1: posture controls and version mapping (about one quarter)

Everything here is Rego against objects the CLI and operator already collect, plus host-sensor data that already
exists for the CIS controls.

### New controls

| Control | Input | Finding and severity |
|---|---|---|
| Control-plane Go version supports PQC KEX | kube-apiserver, kubelet and etcd binaries built with Go 1.24+ (buildinfo survives stripping) | Below 1.24: no hybrid key exchange possible. Medium. etcd has its own Go version and must be checked separately |
| Client and server Go version skew | Go 1.23 offers only the draft X25519Kyber768Draft00; Go 1.24 offers X25519MLKEM768; they do not interoperate and silently fall back to X25519 | A 1.24 client against a 1.23 server, or the reverse, negotiates classical. Medium |
| TLS minimum version | `--tls-min-version` on apiserver, kubelet, etcd | Below TLS 1.3 blocks hybrid KEX. Medium. TLS 1.2 has no standard hybrid path, so report "requires upgrade", not "misconfigured" |
| Custom cipher suites | `--tls-cipher-suites` | Informational only. TLS 1.3 suites are symmetric and not a PQC concern. Do not raise alarms |
| Service mesh PQC | Istio `COMPLIANCE_POLICY=pqc` or `meshConfig.tlsDefaults.ecdhCurves` / `meshMTLS.ecdhCurves`; Linkerd version 2.19+ | Mesh without hybrid groups: Medium. Istio's `pqc` policy is hybrid-only and breaks TLS 1.2 clients, so the remediation recommends additive group lists |
| TLS Secret inventory | `kubernetes.io/tls` Secrets, Ingress `spec.tls`, Gateway listeners: parse PEM for key algorithm, size, signature algorithm, expiry, issuer | Informational with expiry, so certificate rotation lines up with the 2030 signature deadline. Not a failure: ML-DSA certificates are not available in cert-manager or the Go stack yet |
| cert-manager issuers | `Certificate.spec.privateKey.algorithm` | Inventory and ownership |
| Webhook and APIService trust anchors | Decoded `caBundle` | Trust-anchor algorithm and lifetime |
| Service-account signing key | `/openid/v1/jwks` `alg` (typically RS256) | Signature algorithm of the cluster identity root. Informational |
| Encryption at rest | EncryptionConfiguration providers | Symmetric, so low PQC risk. KMS v2 plugin transport is TLS and is checked like any other endpoint |
| C-0142 fix | existing control | Stop recommending `aescbc`; prefer KMS v2, `secretbox` or `aesgcm` with rotation |

### Library capability mapping

kubevuln already produces a Syft SBOM per image. A small table maps package name and version to PQC capability and
becomes a field on the vulnerability-manifest pipeline's package view:

| Library | PQC key exchange in TLS | PQC signatures | Notes |
|---|---|---|---|
| OpenSSL 3.5.0+ | X25519MLKEM768 default key share | ML-DSA, SLH-DSA in default provider | Below 3.5 needs oqs-provider; `Groups` in openssl.cnf can disable |
| Go stdlib 1.24+ | X25519MLKEM768 default | None in crypto/tls | Read from Go buildinfo; `CurvePreferences` in code overrides |
| aws-lc, rustls | Hybrid ML-KEM supported | Varies | Check features |
| JDK 24+ | ML-KEM and ML-DSA primitives (JEP 496, 497) | Primitives yes | JSSE hybrid TLS defaults unverified |
| liboqs, oqs-provider | Yes | Yes | Signals intentional PQC |

### Framework

A `pqc-readiness` framework in regolibrary groups the controls above and carries references to IR 8547, M-26-15, the
EU roadmap and the NCSC timeline the same way the CIS and SOC 2 frameworks carry their section references.

**Severity policy for the whole proposal.** "Classical key exchange on the wire" is High for confidentiality-sensitive
data. "RSA certificate" is Informational until ML-DSA support exists in the common tool chain. If every cluster fails,
users mute the framework.

## 6. Phase 2: CBOM output (about two quarters)

### Image CBOM in kubevuln

kubevuln adds a crypto cataloging step after Syft runs. It does three things:

1. Tags known crypto packages from the SBOM with a capability record (the table above).
2. Parses certificates, private keys and crypto configuration files from the image filesystem: PEM and DER
   certificates, key files, `openssl.cnf`, `java.security`, SSH host keys and daemon config.
3. Emits CycloneDX 1.6/1.7 `cryptographic-asset` components with `algorithmProperties`,
   `certificateProperties`, `protocolProperties` or `relatedCryptoMaterialProperties`, and `dependsOn` links from the
   software component to the algorithm it implements or uses.

Detectors are not written from scratch. The preferred order is: contribute an X.509 and key cataloger to Syft
(issue #5190 is open and the maintainers are receptive), otherwise embed cbomkit-theia's plugins as a library. A private
fork is the fallback, not the plan.

**Noise control is a design requirement, not a later fix.** One demo image produced 8,947 crypto assets, 1,454 of
them trust-store certificates. System CA trust stores are inventoried in the CBOM but collapsed to one entry per store
and excluded from findings by default. Trivy's own design discussion says static crypto scanning reports presence, not
use, and this proposal does not claim otherwise until Phase 3 lands.

### Cluster CBOM

The CLI and operator assemble a cluster-level CBOM from the Phase 1 API inventory: TLS Secrets, trust anchors, mesh
configuration, control-plane protocol settings. Each asset carries the Kubernetes object that owns it so the inventory
has an owner, which is what DORA and M-26-15 ask for.

### Storage

A new kind in `storage`, provisionally `CryptoBOM` in `spdx.softwarecomposition.kubescape.io/v1beta1`, stored next
to `SBOMSyft` with the same image-keyed naming and the same retention. It holds capability only and is never updated
from runtime data. Phase 3 adds `CryptoProfile` for behavior (section 7). Whether the CBOM embeds the CycloneDX document
or normalizes it into typed fields is an open question (section 9).

### Standards tracking

CISA and NIST are due to publish minimum CBOM elements around March 2027. The output format conforms to them when they
appear. Until then CycloneDX 1.7 is the target, and SPDX 3.0 is not, because it has no first-class crypto model.

## 7. Phase 3: runtime crypto profile and relevant crypto (two to three quarters)

### What is observable

TLS 1.2 and 1.3 ClientHello messages carry `supported_groups`, `key_share`, cipher suites, `signature_algorithms`,
ALPN and SNI in plaintext. ServerHello carries the selected version, cipher suite and `key_share` group. That is enough
to classify every connection as hybrid-PQ or classical, per pod and per peer. SSH KEXINIT is plaintext too, and
OpenSSH negotiates `mlkem768x25519-sha256` or `sntrup761x25519-sha512` when both sides support it.

What is not observable: in TLS 1.3 the certificate and CertificateVerify messages are encrypted, so the wire cannot
reveal the signature algorithm in use. That is correlated from the Phase 1 and 2 inventories instead.

### Design for node-agent

- **Primary method:** a TC or socket-filter program that parses the first packets of each flow for ClientHello,
  ServerHello and KEXINIT. It is library-agnostic (OpenSSL, BoringSSL, Go, rustls, Java all look the same on the
  wire), low overhead because only handshake bytes are inspected, and keyed to a container through the enrichment
  node-agent already does for the network manager. stolostron/tls-probe (Red Hat ecosystem, Rust, TC classifiers)
  is prior art that does exactly this passive capture.
- **Secondary method, opt-in only:** OpenSSL uprobes (`SSL_get_negotiated_group`,
  `SSL_get_peer_signature_type_nid`) and Go `crypto/tls` hooks for cases where the wire is unreadable (QUIC) or where
  the TLS 1.3 signature algorithm is needed. Go uprobes are fragile across versions and stripped binaries, so they are
  never on by default.
- **Aggregation:** into a `CryptoProfile` kind keyed by `workload||namespace||selector||port`, the same granularity the
  network neighborhood uses for peers. Keying on port keeps different TLS versions on different ports of one workload
  apart. The object follows the container-profile lifecycle (learning window, completion, merge into the
  workload-level object). Fields per port and peer: protocol, version, offered groups, selected group, cipher suite,
  signature algorithms offered, SNI, peer identity, direction, session count and connection state (below).
- **Link to the CBOM:** node-agent already knows the executable behind each socket. Each profile entry carries the
  `bom-ref` of the CBOM component (library or binary) that made the handshake, so behavior points back to capability.
- **Relevant crypto:** the join of `CryptoBOM` and `CryptoProfile`, computed when findings are produced. A
  classical-only session to an egress or cross-trust-boundary peer becomes a High finding. Repeated hybrid sessions
  mark the workload and port as migrated. Capability that is never used stays in the inventory and is not a finding.

### Connection states

A connection does not always have one clean answer. The profile records the state explicitly instead of guessing:

| State | When | What the profile records |
|---|---|---|
| `negotiated` | A full handshake was seen | Offered groups, selected group, version and cipher suite of the completed handshake |
| `downgraded` | The effective state is weaker than what one side offered | Every attempt and the final result. Example: the client offers X25519MLKEM768, the server answers with a HelloRetryRequest for X25519 and the session completes classical. Another example: a middlebox drops the larger ClientHello and the client reconnects without PQC groups |
| `indeterminate` | The connection existed before the tracer started, or the handshake was not captured | Peer, port and connection age. No crypto claim is made |

**How the effective state is chosen.** The effective state is the selected group and version of the last handshake
that completed. Attempts are correlated in two ways: within one flow (HelloRetryRequest) and across flows to the same
peer and port inside a short window (client reconnect fallback). A `downgraded` entry is reported even when the final
result is hybrid on another attempt. Fallback paths are exactly what an attacker or a broken middlebox exercises.

**Long-lived connections.** Connections opened before node-agent started stay `indeterminate` until they reconnect on
their own. TLS 1.3 has no renegotiation, and `KeyUpdate` does not change the group, so a "forced re-handshake" means
closing the connection. That is disruptive and is not done by default. Whether to offer it as an explicit opt-in
action is an open question (section 9). The profile reports the share of `indeterminate` connections so users can
judge coverage.

**Nested tunnels.** The parser sees the outermost handshake on the interface it watches. For TLS inside a WireGuard
or IPsec tunnel, or TLS inside TLS, the profile records the outer envelope and marks the entry `envelope-only`. Inner
handshakes are visible only where they are plaintext at another capture point, for example inside the pod network
namespace for a sidecar mesh. For harvest-now-decrypt-later the outer envelope is what a passive recorder sees, so it
is the right first answer, but it does not prove anything about the inner layer.

### Coverage gaps to design around

- **Fragmented ClientHellos.** ML-KEM-768 adds 1,184 bytes and pushes the ClientHello past 1,500 bytes. The parser
  must reassemble across segments or it will systematically miss the exact sessions it wants to see.
- **QUIC/HTTP3.** Initial packets are encrypted with keys derived from the public connection ID. Decryptable without
  secrets, but it needs userspace work. Deferred.
- **Sidecar and ambient meshes.** The application speaks plaintext and the proxy does TLS. Sessions are attributed
  to the proxy, and the mesh configuration from Phase 1 is the authoritative answer for the application.
- **Kernel TLS offload** is not a gap. The handshake runs in userspace and is visible on the wire before the keys
  move into the kernel, so the group and cipher suite are captured as usual. Only the records after the handshake are
  handled by the kernel, and they are not needed.
- **In-process crypto** that never touches a socket is out of scope for the wire parser and is covered by the CBOM.

### Why this phase is the point

Runtime evidence turns "RSA present" into "pod X negotiated classical X25519 with external peer Y 4,200 times in the
last 24 hours". That is the harvest-now-decrypt-later exposure M-26-15 asks agencies to prioritize, and it is the same
false-positive reduction Kubescape already sells with application profiles and reachability for CVEs. Static CBOM
output is noisy regardless of tooling; runtime correlation should ship before the project markets CBOM accuracy.

## 8. Pitfalls this design must respect

- **Hybrid is the goal state.** Never flag X25519MLKEM768 as "uses vulnerable X25519".
- **TLS 1.2 cannot be fixed by configuration.** There is no standard hybrid path. Report "requires upgrade".
- **Crypto agility is about where the choice is made.** Record whether an algorithm comes from a library default or
  explicit configuration. Users need to know what they can change.
- **Expect breakage.** Larger key shares break some middleboxes. Pair every "enable hybrid" remediation with a test
  path rather than a flag flip.
- **Severity discipline.** Signatures are Informational until the ecosystem can issue PQC certificates. Key exchange is
  the finding that matters now.

## 9. Open questions for review

1. **`CryptoBOM` storage shape.** Embed the CycloneDX JSON (simplest, round-trips cleanly), or normalize into typed
   fields the way `SBOMSyft` does? Typed fields make Rego and the UI easier; embedding keeps the standard intact for
   export. Either way, customers and regulators expect one CBOM file, so export merges the `CryptoProfile` into the
   CBOM as CycloneDX evidence on the matching components while storage keeps the two objects separate.
2. **Where the cluster CBOM is assembled.** In the CLI for one-shot scans, in the operator for continuous mode, or
   both with a shared package?
3. **Upstream first or embed first.** Contributing to Syft is the right long-term answer but gates Phase 2 on another
   project's review cycle. Embedding cbomkit-theia gets output sooner. The proposal leans toward starting with
   cbomkit-theia plugins behind an interface, and upstreaming to Syft in parallel.
4. **Sharing the peer index with the network neighborhood.** Review settled on a separate `CryptoProfile` kind with
   the network neighborhood's key (`workload||namespace||selector||port`). Crypto fields would bloat the neighborhood
   object, and the two complete learning at different times. Still open: should the two share one peer-resolution
   index in node-agent so peer identity is resolved once?
5. **Cilium.** Its PQC status for mutual auth, WireGuard and IPsec was not verified. It needs testing before the mesh
   control covers it.
6. **Forced re-handshake.** Should users be able to close long-lived `indeterminate` connections so they reconnect
   and get verified, for example as an explicit operator action per workload? Or should verification always wait for
   organic reconnects?

## 10. Suggested sequence and exit criteria

| Phase | Deliverable | Exit criterion |
|---|---|---|
| 1 | Controls, framework, library map, C-0142 fix | `kubescape scan framework pqc-readiness` runs on a v1.32 and a v1.33 cluster and the Go-version and skew controls produce the expected results |
| 2 | kubevuln crypto cataloger, image-keyed `CryptoBOM` kind, CLI export | A CycloneDX 1.7 CBOM for a sample image validates against the schema; trust stores are collapsed; the same image scanned by Trivy's crypto scanner yields a comparable asset count after collapsing |
| 3 | node-agent handshake parser, `CryptoProfile` kind, relevant-crypto join, merged CBOM export | A cluster with one Go 1.23 workload and one Go 1.24 workload shows classical versus hybrid sessions correctly, including fragmented ClientHellos and a HelloRetryRequest downgrade; connections opened before the agent started are `indeterminate`; one image run as two workloads yields different relevant crypto; measured overhead stays under the existing network-manager budget |

## 11. Caveats on the research behind this proposal

Several 2026 facts (the EO number, the IR 8547 finalization date, Trivy and cdxgen versions, Wiz GA) come mainly from
secondary write-ups and vendor blogs. The M-26-15 text was confirmed from the whitehouse.gov PDF. The Federal Register
lists the order as EO 14412, while day-one coverage called it EO 14409, a number already used by an AI order. Reconfirm
before any external messaging. Not verified: Cilium PQC status, JDK JSSE hybrid TLS defaults, cert-manager ML-DSA plans
and whether Hubble or Tetragon ship any TLS-group reporting.

## 12. References

- NIST IR 8547, *Transition to Post-Quantum Cryptography Standards*
- FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA)
- OMB M-26-15, *Execution of the Migration to Post-Quantum Cryptography*
- EO 14412, *Securing the Nation Against Advanced Cryptographic Attacks*
- NIS Cooperation Group, *Coordinated Implementation Roadmap for the Transition to Post-Quantum Cryptography*, v1.1
- Commission Delegated Regulation (EU) 2024/1774 (DORA RTS), Articles 6, 7 and 10
- UK NCSC, *Timelines for migration to post-quantum cryptography*
- CycloneDX 1.6/1.7 specification, cryptographic-asset component type
- PQCA CBOMkit: cbomkit-theia, sonar-cryptography
- Trivy design discussion #10933 and issue #10962 (crypto scanner)
- Syft issue #5190 (X.509 inventory proposal)
- Kubernetes blog, July 18, 2025: *Post-Quantum Cryptography in Kubernetes*
- Go 1.24 release notes, `crypto/mlkem` and default X25519MLKEM768
- Istio `COMPLIANCE_POLICY=pqc` (1.27); Linkerd 2.19 release notes
- stolostron/tls-probe

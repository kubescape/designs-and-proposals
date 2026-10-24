# Verified OCI-attached SBOM reuse in kubevuln

Status: proposed; implementation decisions below require maintainer review.

Tracking issue: https://github.com/kubescape/designs-and-proposals/issues/22

This document moves the proposal and its historical experiment from the issue into a reviewable file. The experiment output below was recorded on 4 October 2026 against the pinned revision; it is not a claim that tests were rerun for this documentation change.

## Summary

Propose adding support for kubevuln to reuse a verified, compatible Syft SBOM published alongside an immutable container image in an OCI registry. On an eligible cache miss, kubevuln would retrieve and verify the SBOM instead of downloading image layers and running the cataloguer again. Grype would still perform vulnerability matching using kubevuln's database and configuration.

This is an opt-in optimization for explicitly approved build pipelines. A signature establishes who signed a document; it does not establish that the document lists every package. The experiment below demonstrates both the potential saving and that limitation.

This issue proposes design and implementation scope. The attached experiment is a feasibility test, not a production implementation.

## Problem and existing integration points

Inspected kubevuln at [`c68de0013294ea8ac46738b906d38feb8820e7c1`](https://github.com/kubescape/kubevuln/tree/c68de0013294ea8ac46738b906d38feb8820e7c1).

- [`getOrCreateSBOM`](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/core/services/scan.go#L1390) already provides stored-SBOM reuse and process-local singleflight. On a miss, it invokes `SBOMCreator.CreateSBOM`. This proposal targets that miss, including a new cluster with no stored SBOM; it does not claim every scan currently recatalogues an image.
- [`scanEmbeddedSBOMs`](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/adapters/v1/syft.go#L410) adds an embedded-SBOM cataloguer after image source acquisition. It does not provide the registry-referrer acquisition path proposed here.
- [`domain.SBOM`](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/core/domain/sbom.go), [`domainToSyft`](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/adapters/v1/domain_to_syft.go), and [`GrypeAdapter.ScanSBOM`](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/adapters/v1/grype.go#L470) provide an existing downstream path to reuse.

For organizations already generating SBOMs in controlled build pipelines, this could reduce registry traffic and cataloguing work on cold caches. It is separate from [external VEX ingestion](https://github.com/kubescape/kubevuln/issues/387): an SBOM describes packages, whereas VEX changes the interpretation of vulnerability findings.

## Proposed first version

1. **Resolve the actual image subject.** Use the immutable platform-specific manifest digest and OS/architecture/variant. Do not bind eligibility to a mutable tag or assume an index-wide SBOM describes every child image.
2. **Discover an attached SBOM.** Use OCI 1.1 referrers in the image repository, with bounded candidates, response size, and time. Reuse registry credentials, TLS/proxy configuration, and cancellation handling. Unsupported discovery falls back to normal generation.
3. **Verify trust and binding.** Start with configured public keys scoped to approved repositories/build producers. Use maintained Sigstore/in-toto libraries to verify an agreed standard signed-attestation profile. Check signed subject, artifact content digest, resolved platform, and SBOM source metadata consistently. An OCI `subject` link alone is insufficient verification.
4. **Check compatibility.** Initially support one tested Syft JSON schema/profile, preserving distro data, package type/version/PURL, package-specific metadata, and source identity. Retain file/location metadata needed by the requested scan mode; otherwise fall back. A valid empty-image SBOM must not be rejected solely because its package list is empty.
5. **Import or generate.** Add an optional acquisition provider around the existing service path, usable whether the fallback is in-process Syft or the sidecar. Feed accepted SBOMs through existing Grype matching. If discovery, verification, or compatibility fails, reject that artifact and run normal generation. If generation also fails, report a scan failure, never an empty successful result.
6. **Persist provenance and respect policy changes.** Record acquisition source, image/platform, artifact digest, signer, producer/schema/profile, verification time, and trust-policy fingerprint. Imported SBOMs must not masquerade as locally generated SBOMs with the local creator's version. Revalidate policy eligibility on imported-cache hits; revoked keys or changed policies must invalidate eligibility. Preserve the behavior of existing locally generated cache hits.
7. **Expose the decision.** Add bounded-label metrics and diagnostics for import success, bytes fetched, and fallback reasons. Provide a build/publish example and comparison against normal generation.

Default: disabled. Initial enabled mode: prefer a verified compatible SBOM, otherwise generate normally. Ambiguous eligible artifacts should fall back until deterministic selection semantics are agreed.

The trust contract must be explicit: an administrator is trusting an approved producer's inventory, not obtaining a cryptographic proof of completeness. Producer/profile allowlists and parity tests constrain this trust; they do not eliminate it. Periodic independent recataloguing is a possible later audit feature.

## Local experiment and actual results

Run on 2026-10-04 using Go 1.26.5 on macOS arm64, Docker Engine 29.1.5, and the kubevuln revision above. The module uses Grype `v0.104.1` and replaces Syft with `github.com/kubescape/syft v1.32.0-ks.2`.

The test:

1. Exports cached Alpine and Nginx images with Docker, then serves each image from a local HTTP OCI registry with referrers enabled (`go-containerregistry/pkg/registry`). The registry is an in-process server; Docker runs the existing Grype database fixture.
2. Generates a real SBOM through kubevuln's current Syft adapter, representing the build/publish stage, and attaches a signed copy to the image digest.
3. Calls the unchanged Syft adapter again with the attachment present. It still downloads image layers and makes zero referrers requests, reproducing the missing acquisition path.
4. Retrieves the attachment with a small prototype, verifies an Ed25519 signature over the document/image/platform claim, and imports it without image-layer downloads. Each measured retrieval performs fresh discovery; there is no prototype artifact cache.
5. Checks full package-record equality against fresh cataloguing and full Grype match-record equality, including ignored matches, with the same real DB and matcher configuration. These are nonzero vulnerability results.

**Scope of this proof:** the envelope is an experimental JSON format using Go's Ed25519 implementation. It is not a Sigstore bundle, DSSE interoperability test, or finished importer. Production must use the agreed standard attestation profile and maintained verification library. The imported document was generated by kubevuln's own adapter, so this proves round-trip fidelity for that producer, not arbitrary vendor SBOM equivalence.

| Fixture | Current adapter: registry GET response-body bytes | Prototype: registry GET response-body bytes | Image-layer GETs, current → prototype | Packages, both | Grype matches, both |
|---|---:|---:|---:|---:|---:|
| Alpine 3.19, linux/arm64/v8 | 3,360,325 | 62,990 | 1 → 0 | 15 | 6 |
| Nginx 1.25, linux/arm64 | 67,654,810 | 1,751,413 | 7 → 0 | 150 | 262 |

These are approximately 98.1% and 97.4% fewer registry response-body bytes in this experiment. They exclude publishing, database loading, HTTP headers, and TLS. They are not whole-cluster savings. The test bypasses the service's existing SBOM cache to measure acquisition on a miss.

The local acquisition timings below exclude Grype matching and compare one normal acquisition with three prototype acquisitions after setup. The prototype timed region also includes package-parity assertions. These are not a statistically controlled benchmark or a production latency claim. No peak-memory or CPU improvement was measured.

<details>
<summary>Actual alpine-3.19 output (selected lines; Docker lifecycle logs omitted)</summary>

```text
    sbom_reuse_proposal_test.go:222: fixture image=sha256:e2ceefeeb373d31f49615b2386ff4b4066cbd489ba174f105a295d6905b42619 platform=linux/arm64/v8 packages=15 sbom_bytes=46355 signed_envelope_bytes=62093
    sbom_reuse_proposal_test.go:229: CURRENT_MAIN elapsed_ms=481.545 registry_GET_body_bytes=3360325 image_layer_GETs=1 referrers_GETs=0
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=1 elapsed_ms=2.975 registry_GET_body_bytes=62990 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=2 elapsed_ms=2.471 registry_GET_body_bytes=62990 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=3 elapsed_ms=2.435 registry_GET_body_bytes=62990 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:264: PARITY current_packages=15 imported_packages=15 current_matches=6 imported_matches=6 full_match_records_equal=true db=8947f666e75c337773be86e0c6f7f4739c7549184aa994ae6236d5dbe666523b
    sbom_reuse_proposal_test.go:323: LIMITATION trusted_signature_valid=true omitted_package=busybox complete_matches=6 incomplete_matches=4
    sbom_reuse_proposal_test.go:343: FALLBACK packages=15 image_layer_GETs=1
--- PASS: TestProposalVerifiedRegistrySBOMReuse (3.24s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/tampered_payload (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/untrusted_signer (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/wrong_image_digest (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/wrong_platform (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/signed_but_incomplete_sbom (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/unsupported_schema (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/malformed_envelope (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/signature_does_not_prove_completeness (0.15s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/artifact_size_limit (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/no_artifact_real_syft_fallback (0.50s)
PASS
ok  	github.com/kubescape/kubevuln/adapters/v1	4.770s
```

</details>

<details>
<summary>Actual nginx-1.25 output (selected lines; Docker lifecycle logs omitted)</summary>

```text
    sbom_reuse_proposal_test.go:222: fixture image=sha256:94c7c6c0450de052ae11e726dfbf09944fa464d2c3bf82ddbb2496af0415b481 platform=linux/arm64 packages=150 sbom_bytes=1312673 signed_envelope_bytes=1750513
    sbom_reuse_proposal_test.go:229: CURRENT_MAIN elapsed_ms=1437.166 registry_GET_body_bytes=67654810 image_layer_GETs=7 referrers_GETs=0
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=1 elapsed_ms=30.643 registry_GET_body_bytes=1751413 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=2 elapsed_ms=30.510 registry_GET_body_bytes=1751413 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:240: PROTOTYPE run=3 elapsed_ms=30.657 registry_GET_body_bytes=1751413 image_layer_GETs=0 referrers_GETs=1
    sbom_reuse_proposal_test.go:264: PARITY current_packages=150 imported_packages=150 current_matches=262 imported_matches=262 full_match_records_equal=true db=8947f666e75c337773be86e0c6f7f4739c7549184aa994ae6236d5dbe666523b
    sbom_reuse_proposal_test.go:323: LIMITATION trusted_signature_valid=true omitted_package=libfreetype6 complete_matches=262 incomplete_matches=261
    sbom_reuse_proposal_test.go:343: FALLBACK packages=150 image_layer_GETs=7
--- PASS: TestProposalVerifiedRegistrySBOMReuse (6.20s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/tampered_payload (0.01s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/untrusted_signer (0.01s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/wrong_image_digest (0.01s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/wrong_platform (0.01s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/signed_but_incomplete_sbom (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/unsupported_schema (0.02s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/malformed_envelope (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/signature_does_not_prove_completeness (0.23s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/artifact_size_limit (0.00s)
    --- PASS: TestProposalVerifiedRegistrySBOMReuse/no_artifact_real_syft_fallback (1.45s)
PASS
ok  	github.com/kubescape/kubevuln/adapters/v1	7.237s
```

</details>

### Negative tests and the completeness counterexample

Both image runs passed the parent integration test and ten subtests:

- Tampered signed payload rejected.
- Signature from a different key rejected.
- Wrong image digest rejected despite an otherwise valid signature.
- Wrong requested platform rejected.
- Signed empty/malformed SBOM structure rejected.
- Unsupported schema rejected.
- Malformed envelope rejected.
- Artifact exceeding the response-body limit rejected.
- Absent attachment followed by a real Syft fallback, with image-layer downloads and the expected package count.
- A trusted signer omitting a vulnerable package is **not** detected by signature/shape checks: the test expects signature verification to succeed and Grype matches to decrease.

In that last test, omitting `busybox` reduced Alpine matches from **6 to 4**; omitting `libfreetype6` reduced Nginx matches from **262 to 261**. This is why the proposal requires an explicit trusted-producer contract and cannot promise that signed third-party SBOMs are complete.

The rejection cases test the prototype verifier, not an already implemented production policy engine. Only the absent-artifact case exercises actual fallback generation; production fallback for every rejection category remains an implementation acceptance criterion.

## Production test and acceptance plan

- **Trust:** standard signature verification, wrong signer, tampering, unsigned documents, mismatched OCI/signed/SBOM subjects, key rotation, policy changes, and imported-cache revalidation. No rejected artifact reaches Grype as a successful import.
- **Image identity:** mutable-tag movement, multi-platform indexes, ARM variants, and wrong-child artifacts. Reuse remains tied to the resolved child digest.
- **Fidelity:** compare complete package and vulnerability records with local generation for an agreed corpus: Alpine, Debian, a language-package image, a distroless image, and an empty image. Include file/location metadata required by relevancy scanning. Differences must be explained or make the profile ineligible.
- **Failure handling:** absent/unsupported referrers, multiple candidates, private-registry auth, redirects, TLS failures, rate limits, timeouts, oversized bodies, malformed data, and a failing Syft fallback. Test both fallback adapters.
- **Integration:** existing storage/singleflight behavior, imported provenance surviving storage round trips, Grype DB changes rescanning the same inventory, and existing VEX/exception behavior remaining intact.
- **Measurement:** repeat on a larger corpus and a real OCI 1.1 registry; publish network bytes, acquisition and end-to-end latency, CPU, peak memory, and fallback rates. Keep cache-hit and cache-miss measurements separate. No unmeasured universal speedup target.
- **End-to-end acceptance:** an eligible verified artifact avoids image-layer downloads, yields expected findings, and exposes provenance; every ineligible artifact follows a visible normal-generation fallback.

## Implementation milestones

| Period | Deliverable |
|---|---|
| Weeks 1–2 | Maintainer-reviewed trust/attestation profile, cache semantics, producer compatibility contract, and baseline fixture corpus |
| Weeks 3–5 | OCI discovery, bounded retrieval, signature verification, subject/platform checks, and unit/registry tests |
| Weeks 6–8 | Service integration, both fallback paths, imported-cache provenance/invalidation, and diagnostics |
| Weeks 9–10 | Failure matrix, package/CVE parity tests, private registry and multi-platform coverage |
| Weeks 11–12 | Reproducible performance report, documentation, build/publish example, and final demo |

Recommended skills: Go, OCI registries, container image manifests, Syft/Grype, Kubernetes configuration, and supply-chain signatures.

Deferred: SPDX/CycloneDX conversion, arbitrary vendor-producer compatibility, keyless verification, additional registry attachment conventions, automatic signing infrastructure, distributed scheduling, and admission enforcement. Supporting one producer/profile well is the first goal.

## Decisions requested from maintainers

1. Is an opt-in cache-miss optimization for approved build pipelines useful enough to pursue?
2. Which signed-attestation profile and Syft producer/version should be the first supported combination?
3. Should the initial policy live in kubevuln configuration, or is an existing organization-wide trust-policy surface preferred?
4. Which scan modes can accept imported inventories initially, especially where runtime relevancy requires file metadata?

## Reproduce the experiment

Save the test source below as `adapters/v1/sbom_reuse_proposal_test.go` in a checkout of kubevuln at `c68de0013294ea8ac46738b906d38feb8820e7c1`. It adds no production changes or module dependencies. Docker is required; the test fails rather than skipping if its database fixture cannot start.

The source images were:

- `alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1`
- `nginx@sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c`

The test republishes Docker-exported image contents to its local registry; the resulting manifest digests in the output differ from the source index digests above. The signature binds to the republished manifest.

The Docker DB fixture used `quay.io/kubescape/grype-offline-db@sha256:56468cbe92622770860f5c2f96b43aca836f6466c1c95116a107d461f8de8592`. Its loaded DB checksum was `8947f666e75c337773be86e0c6f7f4739c7549184aa994ae6236d5dbe666523b`. The existing helper uses the `v6-ci-only` tag; use the recorded digest when preparing that local fixture tag for reproduction.

```sh
git clone https://github.com/kubescape/kubevuln.git kubevuln-sbom-proposal
cd kubevuln-sbom-proposal
git checkout --detach c68de0013294ea8ac46738b906d38feb8820e7c1
# Save the Go source below under adapters/v1/ before running the tests.
run_dir=$(mktemp -d)

docker pull --platform linux/arm64 alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
docker tag alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1 kubevuln-proposal-alpine:fixture
docker save --platform linux/arm64 kubevuln-proposal-alpine:fixture -o "$run_dir/alpine.tar"

docker pull --platform linux/arm64 nginx@sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c
docker tag nginx@sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c kubevuln-proposal-nginx:fixture
docker save --platform linux/arm64 kubevuln-proposal-nginx:fixture -o "$run_dir/nginx.tar"

docker pull quay.io/kubescape/grype-offline-db@sha256:56468cbe92622770860f5c2f96b43aca836f6466c1c95116a107d461f8de8592
docker tag quay.io/kubescape/grype-offline-db@sha256:56468cbe92622770860f5c2f96b43aca836f6466c1c95116a107d461f8de8592 quay.io/kubescape/grype-offline-db:v6-ci-only

for fixture in alpine nginx; do
  PROPOSAL_IMAGE_TAR="$run_dir/$fixture.tar" TMPDIR=/tmp \
    go test -tags=dockerfixture ./adapters/v1 \
    -run '^TestProposalVerifiedRegistrySBOMReuse$' -v -count=1 || exit 1
done
```

The test closes its local HTTP registry and terminates its Grype database container. Docker source images and exported tar files remain available for inspection. This experiment was run on arm64; it does not establish amd64 or cross-platform coverage.

<details>
<summary>Complete experiment source</summary>

```go
//go:build dockerfixture

package v1

// Proposal experiment only. The signed envelope below is deliberately NOT a
// production attestation format or a replacement for a Sigstore verifier.
import (
	"bytes"
	"context"
	"crypto/ed25519"
	"crypto/rand"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/http/httptest"
	"os"
	"strings"
	"sync/atomic"
	"testing"
	"time"

	"github.com/google/go-containerregistry/pkg/name"
	"github.com/google/go-containerregistry/pkg/registry"
	cr "github.com/google/go-containerregistry/pkg/v1"
	"github.com/google/go-containerregistry/pkg/v1/empty"
	"github.com/google/go-containerregistry/pkg/v1/mutate"
	"github.com/google/go-containerregistry/pkg/v1/remote"
	"github.com/google/go-containerregistry/pkg/v1/static"
	"github.com/google/go-containerregistry/pkg/v1/tarball"
	"github.com/google/go-containerregistry/pkg/v1/types"
	"github.com/kubescape/kubevuln/core/domain"
	"github.com/kubescape/storage/pkg/apis/softwarecomposition/v1beta1"
	"github.com/stretchr/testify/require"
)

type proposalClaim struct {
	ImageDigest string          `json:"imageDigest"`
	Platform    string          `json:"platform"`
	SBOM        json.RawMessage `json:"sbom"`
}
type proposalEnvelope struct {
	Payload   []byte `json:"payload"`
	Signature []byte `json:"signature"`
}

func proposalSign(t *testing.T, claim proposalClaim, key ed25519.PrivateKey) []byte {
	t.Helper()
	payload, err := json.Marshal(claim)
	require.NoError(t, err)
	envelope, err := json.Marshal(proposalEnvelope{payload, ed25519.Sign(key, payload)})
	require.NoError(t, err)
	return envelope
}
func proposalVerify(data []byte, key ed25519.PublicKey, digest, platform string) (*v1beta1.SyftDocument, error) {
	var envelope proposalEnvelope
	if err := json.Unmarshal(data, &envelope); err != nil {
		return nil, err
	}
	if !ed25519.Verify(key, envelope.Payload, envelope.Signature) {
		return nil, fmt.Errorf("untrusted signature")
	}
	var claim proposalClaim
	if err := json.Unmarshal(envelope.Payload, &claim); err != nil {
		return nil, err
	}
	if claim.ImageDigest != digest {
		return nil, fmt.Errorf("image digest mismatch")
	}
	if claim.Platform != platform {
		return nil, fmt.Errorf("platform mismatch")
	}
	var shape struct {
		Schema struct {
			Version string `json:"version"`
		} `json:"schema"`
		Source struct {
			Type string `json:"type"`
		} `json:"source"`
		Distro struct {
			ID string `json:"id"`
		} `json:"distro"`
	}
	if err := json.Unmarshal(claim.SBOM, &shape); err != nil {
		return nil, err
	}
	// Fixture-specific minimum shape checks, not a completeness proof.
	if !strings.HasPrefix(shape.Schema.Version, "16.") || shape.Source.Type != "image" || shape.Distro.ID == "" {
		return nil, fmt.Errorf("unsupported or incomplete SBOM shape")
	}
	var doc v1beta1.SyftDocument
	if err := json.Unmarshal(claim.SBOM, &doc); err != nil {
		return nil, err
	}
	return &doc, nil
}

type proposalTraffic struct{ bytes, layerGets, referrerGets atomic.Int64 }

func (s *proposalTraffic) reset() { s.bytes.Store(0); s.layerGets.Store(0); s.referrerGets.Store(0) }

type proposalWriter struct {
	http.ResponseWriter
	bytes *atomic.Int64
}

func (w proposalWriter) Write(b []byte) (int, error) {
	n, err := w.ResponseWriter.Write(b)
	w.bytes.Add(int64(n))
	return n, err
}

// Each invocation performs fresh OCI discovery; it does not reuse a cached artifact.
func proposalFetch(ref name.Digest, key ed25519.PublicKey, platform string, maxBytes int64) (*v1beta1.SyftDocument, error) {
	idx, err := remote.Referrers(ref)
	if err != nil {
		return nil, err
	}
	m, err := idx.IndexManifest()
	if err != nil {
		return nil, err
	}
	if len(m.Manifests) != 1 {
		return nil, fmt.Errorf("expected one artifact, got %d", len(m.Manifests))
	}
	artifact, err := remote.Image(ref.Context().Digest(m.Manifests[0].Digest.String()))
	if err != nil {
		return nil, err
	}
	am, err := artifact.Manifest()
	if err != nil {
		return nil, err
	}
	if am.Subject == nil || am.Subject.Digest.String() != ref.DigestStr() {
		return nil, fmt.Errorf("OCI subject mismatch")
	}
	layers, err := artifact.Layers()
	if err != nil {
		return nil, err
	}
	if len(layers) != 1 {
		return nil, fmt.Errorf("unexpected artifact layers")
	}
	r, err := layers[0].Compressed()
	if err != nil {
		return nil, err
	}
	defer r.Close()
	data, err := io.ReadAll(io.LimitReader(r, maxBytes+1))
	if err != nil {
		return nil, err
	}
	if int64(len(data)) > maxBytes {
		return nil, fmt.Errorf("artifact exceeds byte limit")
	}
	return proposalVerify(data, key, ref.DigestStr(), platform)
}

func TestProposalVerifiedRegistrySBOMReuse(t *testing.T) {
	archive := os.Getenv("PROPOSAL_IMAGE_TAR")
	require.NotEmpty(t, archive, "set PROPOSAL_IMAGE_TAR to docker save output")
	img, err := tarball.ImageFromPath(archive, nil)
	require.NoError(t, err)
	manifest, err := img.Manifest()
	require.NoError(t, err)
	cfg, err := img.ConfigFile()
	require.NoError(t, err)
	platform := cfg.OS + "/" + cfg.Architecture
	if cfg.Variant != "" {
		platform += "/" + cfg.Variant
	}
	layerPaths := map[string]bool{}
	for _, layer := range manifest.Layers {
		layerPaths[layer.Digest.String()] = true
	}
	var stats proposalTraffic
	reg := registry.New(registry.WithReferrersSupport(true), registry.Logger(log.New(io.Discard, "", 0)))
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodGet {
			if strings.Contains(r.URL.Path, "/referrers/") {
				stats.referrerGets.Add(1)
			}
			if i := strings.Index(r.URL.Path, "/blobs/"); i >= 0 && layerPaths[r.URL.Path[i+7:]] {
				stats.layerGets.Add(1)
			}
			reg.ServeHTTP(proposalWriter{w, &stats.bytes}, r)
		} else {
			reg.ServeHTTP(w, r)
		}
	}))
	defer server.Close()
	tag, err := name.NewTag(strings.TrimPrefix(server.URL, "http://")+"/proposal/alpine:fixture", name.Insecure)
	require.NoError(t, err)
	require.NoError(t, remote.Write(tag, img))
	hash, err := img.Digest()
	require.NoError(t, err)
	ref := tag.Context().Digest(hash.String())
	ctx := context.WithValue(context.Background(), domain.ScanIDKey{}, "sbom-reuse-proposal")
	ctx = context.WithValue(ctx, domain.TimestampKey{}, int64(1791129600))
	ctx = context.WithValue(ctx, domain.WorkloadKey{}, domain.ScanCommand{})
	creator := NewSyftAdapter(2*time.Minute, 1<<30, 50<<20, false, nil)
	opts := domain.RegistryOptions{Platform: platform, InsecureUseHTTP: true}
	// Produce the fixture SBOM once, representing the build/publish stage.
	baseline, err := creator.CreateSBOM(ctx, "proposal-alpine", ref.Name(), tag.Name(), opts)
	require.NoError(t, err)
	require.NotNil(t, baseline.Content)
	require.NotEmpty(t, baseline.Content.Artifacts)
	sbomJSON, err := json.Marshal(baseline.Content)
	require.NoError(t, err)
	pub, priv, err := ed25519.GenerateKey(rand.Reader)
	require.NoError(t, err)
	claim := proposalClaim{hash.String(), platform, sbomJSON}
	envelope := proposalSign(t, claim, priv)
	artifact, err := mutate.AppendLayers(empty.Image, static.NewLayer(envelope, types.MediaType("application/vnd.kubescape.proposal.signed-sbom.v1+json")))
	require.NoError(t, err)
	artifact = mutate.MediaType(artifact, types.OCIManifestSchema1)
	imageSize, err := img.Size()
	require.NoError(t, err)
	artifact = mutate.Subject(artifact, cr.Descriptor{Digest: hash, MediaType: manifest.MediaType, Size: imageSize}).(cr.Image)
	require.NoError(t, remote.Write(tag.Context().Tag("sbom"), artifact))
	t.Logf("fixture image=%s platform=%s packages=%d sbom_bytes=%d signed_envelope_bytes=%d", hash, platform, len(baseline.Content.Artifacts), len(sbomJSON), len(envelope))
	// Current production adapter: artifact is present, but it still pulls/catalogues.
	stats.reset()
	start := time.Now()
	current, err := creator.CreateSBOM(ctx, "proposal-alpine", ref.Name(), tag.Name(), opts)
	require.NoError(t, err)
	require.NotNil(t, current.Content)
	t.Logf("CURRENT_MAIN elapsed_ms=%.3f registry_GET_body_bytes=%d image_layer_GETs=%d referrers_GETs=%d", float64(time.Since(start).Microseconds())/1000, stats.bytes.Load(), stats.layerGets.Load(), stats.referrerGets.Load())
	require.Positive(t, stats.layerGets.Load())
	require.Zero(t, stats.referrerGets.Load())
	var imported *v1beta1.SyftDocument
	for i := 1; i <= 3; i++ {
		stats.reset()
		start = time.Now()
		imported, err = proposalFetch(ref, pub, platform, 4<<20)
		require.NoError(t, err)
		require.Equal(t, baseline.Content, imported, "all stored package metadata must survive retrieval")
		require.Equal(t, current.Content.Artifacts, imported.Artifacts, "fresh cataloguing and import must have identical full package records")
		t.Logf("PROTOTYPE run=%d elapsed_ms=%.3f registry_GET_body_bytes=%d image_layer_GETs=%d referrers_GETs=%d", i, float64(time.Since(start).Microseconds())/1000, stats.bytes.Load(), stats.layerGets.Load(), stats.referrerGets.Load())
		require.Zero(t, stats.layerGets.Load())
		require.Positive(t, stats.referrerGets.Load())
	}
	// Real Grype, real pinned DB fixture, identical matching mode for both paths.
	g, cleanup, err := NewGrypeAdapterFixedDB()
	require.NoError(t, err, "Docker is required; this experiment must not silently skip")
	defer cleanup()
	readyCtx, cancel := context.WithTimeout(ctx, 2*time.Minute)
	defer cancel()
	require.True(t, g.Ready(readyCtx))
	fullCVE, err := g.ScanSBOM(ctx, current)
	require.NoError(t, err)
	reused := baseline
	reused.Content = imported
	reusedCVE, err := g.ScanSBOM(ctx, reused)
	require.NoError(t, err)
	a, err := json.Marshal(fullCVE.Content.Matches)
	require.NoError(t, err)
	b, err := json.Marshal(reusedCVE.Content.Matches)
	require.NoError(t, err)
	require.JSONEq(t, string(a), string(b), "full vulnerability match records, including package metadata")
	require.Equal(t, fullCVE.Content.IgnoredMatches, reusedCVE.Content.IgnoredMatches)
	require.NotEmpty(t, fullCVE.Content.Matches, "parity must exercise nonzero vulnerability results")
	t.Logf("PARITY current_packages=%d imported_packages=%d current_matches=%d imported_matches=%d full_match_records_equal=true db=%s", len(current.Content.Artifacts), len(imported.Artifacts), len(fullCVE.Content.Matches), len(reusedCVE.Content.Matches), g.DBVersion(ctx))
	otherPub, _, err := ed25519.GenerateKey(rand.Reader)
	require.NoError(t, err)
	var altered proposalEnvelope
	require.NoError(t, json.Unmarshal(envelope, &altered))
	altered.Payload = bytes.Replace(altered.Payload, []byte(hash.String()), []byte("sha256:"+strings.Repeat("0", 64)), 1)
	tampered, err := json.Marshal(altered)
	require.NoError(t, err)
	badShape := claim
	badShape.SBOM = json.RawMessage(`{}`)
	var documentShape map[string]any
	require.NoError(t, json.Unmarshal(sbomJSON, &documentShape))
	documentShape["schema"].(map[string]any)["version"] = "999.0.0"
	unsupported := claim
	unsupported.SBOM, err = json.Marshal(documentShape)
	require.NoError(t, err)
	for _, tc := range []struct {
		name             string
		data             []byte
		key              ed25519.PublicKey
		digest, platform string
	}{
		{"tampered_payload", tampered, pub, hash.String(), platform},
		{"untrusted_signer", envelope, otherPub, hash.String(), platform},
		{"wrong_image_digest", envelope, pub, "sha256:" + strings.Repeat("0", 64), platform},
		{"wrong_platform", envelope, pub, hash.String(), "linux/not-a-real-arch"},
		{"signed_but_incomplete_sbom", proposalSign(t, badShape, priv), pub, hash.String(), platform},
		{"unsupported_schema", proposalSign(t, unsupported, priv), pub, hash.String(), platform},
		{"malformed_envelope", []byte(`{`), pub, hash.String(), platform},
	} {
		t.Run(tc.name, func(t *testing.T) {
			_, err := proposalVerify(tc.data, tc.key, tc.digest, tc.platform)
			require.Error(t, err)
			t.Logf("REJECTED %s: %v", tc.name, err)
		})
	}
	t.Run("signature_does_not_prove_completeness", func(t *testing.T) {
		// Remove a package that really matched a vulnerability, then sign again with
		// the trusted key. Crypto and structural checks cannot detect this omission.
		var shortened v1beta1.SyftDocument
		require.NoError(t, json.Unmarshal(sbomJSON, &shortened))
		omittedName := fullCVE.Content.Matches[0].Artifact.Name
		kept := shortened.Artifacts[:0]
		for _, p := range shortened.Artifacts {
			if p.Name != omittedName {
				kept = append(kept, p)
			}
		}
		shortened.Artifacts = kept
		shortClaim := claim
		shortClaim.SBOM, err = json.Marshal(shortened)
		require.NoError(t, err)
		verified, err := proposalVerify(proposalSign(t, shortClaim, priv), pub, hash.String(), platform)
		require.NoError(t, err)
		incomplete := baseline
		incomplete.Content = verified
		result, err := g.ScanSBOM(ctx, incomplete)
		require.NoError(t, err)
		require.Less(t, len(result.Content.Matches), len(fullCVE.Content.Matches))
		t.Logf("LIMITATION trusted_signature_valid=true omitted_package=%s complete_matches=%d incomplete_matches=%d", omittedName, len(fullCVE.Content.Matches), len(result.Content.Matches))
	})
	t.Run("artifact_size_limit", func(t *testing.T) {
		_, err := proposalFetch(ref, pub, platform, 64)
		require.ErrorContains(t, err, "artifact exceeds byte limit")
	})
	t.Run("no_artifact_real_syft_fallback", func(t *testing.T) {
		// Same image in a different repository, with no referrers.
		missingTag, err := name.NewTag(strings.TrimPrefix(server.URL, "http://")+"/proposal/no-sbom:fixture", name.Insecure)
		require.NoError(t, err)
		require.NoError(t, remote.Write(missingTag, img))
		missingRef := missingTag.Context().Digest(hash.String())
		stats.reset()
		_, fetchErr := proposalFetch(missingRef, pub, platform, 4<<20)
		require.ErrorContains(t, fetchErr, "expected one artifact, got 0")
		fallback, err := creator.CreateSBOM(ctx, "proposal-fallback", missingRef.Name(), missingTag.Name(), opts)
		require.NoError(t, err)
		require.NotNil(t, fallback.Content)
		require.Equal(t, len(current.Content.Artifacts), len(fallback.Content.Artifacts))
		require.Positive(t, stats.layerGets.Load())
		t.Logf("FALLBACK packages=%d image_layer_GETs=%d", len(fallback.Content.Artifacts), stats.layerGets.Load())
	})
}
```

</details>

## Standards references

- [OCI Distribution Specification 1.1.1](https://github.com/opencontainers/distribution-spec/blob/v1.1.1/spec.md)
- [in-toto Statement specification](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md)
- [Sigstore verification documentation](https://docs.sigstore.dev/cosign/verifying/verify/)

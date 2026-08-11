# Proposal: Versioned Repository Scan Contracts

- **Status:** Draft
- **Related issue:**
  [kubescape/kubescape#2494 — Allow pinning the controls version](https://github.com/kubescape/kubescape/issues/2494)
- **Initial scope:** [`kubescape`](https://github.com/kubescape/kubescape) CLI
- **Possible later consumers:** Kubescape MCP server and operator
- **Author:** Daksh Pathak (<daksh.pathak.ug24@nsut.ac.in>)

## Summary

Let a project commit a safe, versioned Kubescape scan contract beside its
manifests. The contract describes policy selection, scope, evaluation limits,
side-effect-free output intent, and CI failure gates. It is loaded only when the
caller names it explicitly:

```bash
kubescape scan . --scan-contract kubescape.yaml --contract ci
```

Repository-local configuration has three resolution regimes:

- scan settings can be selected by the repository and overridden by an explicit
  command-line flag;
- policy inputs are also overridden by explicit CLI values, which lets a
  protected runner pin any selection it considers non-negotiable;
- failure gates influence whether CI accepts the same pull request that supplied
  the file, so explicit runner gate values are a security floor. A repository
  contract may tighten that floor, but cannot weaken it.

Exceptions are not part of the v1alpha1 contract. They remain runner-controlled
through the existing CLI because an untrusted pull request must not be able to
exclude its own findings by default.

The report records the effective contract, the concrete controls release, and
digests for every referenced local input. It can therefore answer both “what
failed?” and “under exactly which inputs and acceptance rules was this run?”

This proposal does not move every Kubescape flag into YAML. Credentials, cloud
endpoints, submission, arbitrary output paths, exceptions, and cluster-mutating
behavior remain outside the repository-controlled schema.

## Motivation

Kubescape already exposes the pieces needed for reproducible scanning, but the
caller must assemble them on every invocation. A scan can choose frameworks,
controls, a regolibrary release, local control configuration, namespace scope,
failure thresholds, timeouts, and output behavior.

These choices are usually spread across several files:

```text
.github/workflows/security.yml    frameworks + trusted gate floors
Makefile                          local developer command
scripts/scan.sh                   namespace scope
README.md                         policy-version expectations
```

Two valid invocations can consequently scan the same commit under different
policy and failure semantics without the report explaining the difference.

### Local and CI scans quietly drift

A developer may run `kubescape scan . framework nsa`, while CI uses a pinned
controls release, a custom controls configuration, and coverage and severity
gates. The commands are both valid, but they are not evaluating the same
contract.

Making ordinary scan settings repository data gives local tools and CI one
reviewable source of intent. Making the effective settings report data makes
differences auditable after the scan.

### A repository must not weaken its own protected gate

A checked-out pull request is attacker-controlled input in an organization-level
required workflow, `pull_request_target` job, or external CI system. The workflow
and Kubescape invocation can be trusted even though the scanned repository is
not.

Moving `severityAtLeast`, `complianceBelow`, or `coverageBelow` into that
repository without another rule would let a pull request loosen the criteria
used to evaluate itself. Moving exceptions into it would be even more direct:
the pull request could add an exception and then point the scan at it.

Ordinary “CLI wins” precedence protects a pinned runner value, but treats the
pin as an opaque override and does not express whether the repository may
tighten it. This proposal makes explicit CLI gate values monotonic floors. The
repository may request stricter evaluation but cannot lower a floor selected by
the trusted runner.

### Policy updates should be reviewable

Pinning `--controls-version` prevents an upstream policy update from silently
changing a run. Putting the requested version in a scan contract also makes the
change visible beside the workload:

```diff
- controlsVersion: v2.0.301
+ controlsVersion: v2.0.307
```

The report must record the concrete version actually loaded, not merely the
requested string. The controls-version work tracked by kubescape/kubescape#2494
owns that resolved metadata. This proposal consumes it and does not duplicate
its implementation.

### Other Kubescape entry points need the same semantics

The MCP IaC tool and operator have smaller, separate configuration surfaces.
They may eventually reuse the side-effect-free typed model, but the CLI is the
only v1alpha1 consumer. This proposal does not require immediate parity.

## Goals

- Make selected scan semantics commit-able and reviewable with the workload.
- Preserve trusted CI acceptance criteria when a pull request controls the
  contract being evaluated.
- Produce the same effective inputs for the same contract, referenced files,
  repository commit, runner floors, and Kubescape version.
- Reject unknown or malformed fields before policy download or cluster access.
- Keep the first schema flat, small, and directly traceable to existing CLI
  behavior.
- Resolve allowed local files relative to the contract consistently across
  local development and CI.
- Record enough provenance to reproduce or audit a run.
- Keep credentials, exceptions, submission, arbitrary writes, remote inputs,
  and cluster mutation outside the default repository trust boundary.
- Keep loading explicit; do not automatically discover a repository contract.

## Non-goals

- Replacing `$HOME/.kubescape/config.json`, which stores tenant and backend
  state under a different trust boundary.
- Supporting every `kubescape scan` flag in YAML.
- Storing access keys, account IDs, encryption keys, cloud URLs, or secrets.
- Allowing a contract to submit results, choose a kubeconfig, write to an
  arbitrary path, or install or execute anything.
- Allowing repository-controlled exceptions in v1alpha1.
- Adding contract inheritance, cross-file includes, Helm values, remote inputs,
  or field-specific list merge rules in v1alpha1.
- Guaranteeing identical findings across different binaries or external
  artifacts. Provenance makes those differences diagnosable.
- Designing an organization-wide policy distribution service.

## Proposed user experience

### A flat repository-owned contract

A project adds `kubescape.yaml`:

```yaml
apiVersion: config.kubescape.io/v1alpha1
kind: ScanContract
metadata:
  name: payments-service
spec:
  minimumKubescapeVersion: v3.1.0
  defaultContract: developer

  contracts:
    developer:
      policy:
        frameworks: [nsa]
        controlsVersion: v2.0.307
        controlsConfigFile: security/kubescape/controls-config.json
      evaluation:
        controlTimeout: 30s
      output:
        formats: [pretty-printer]

    ci:
      policy:
        frameworks: [nsa]
        controlsVersion: v2.0.307
        controlsConfigFile: security/kubescape/controls-config.json
      scope:
        excludeNamespaces: [kube-system]
      evaluation:
        scanTimeout: 10m
        controlTimeout: 30s
      failure:
        severityAtLeast: high
        complianceBelow: 80
        coverageBelow: 95
        degradedPolicyInput: fail
      output:
        formats: [json, sarif]
        omitRawResources: true
```

Each named contract is complete and flat. There is no `extends` field, parent
resolution, or implicit list merging. Some duplication is preferable to
shipping a configuration language before real usage demonstrates the need.
Standard YAML anchors may reduce repetition without changing Kubescape's data
model.

Developers use the default contract:

```bash
kubescape scan . --scan-contract kubescape.yaml
```

CI names its contract and pins non-negotiable gate floors in the trusted
invocation:

```bash
kubescape scan . \
  --scan-contract kubescape.yaml \
  --contract ci \
  --severity-threshold high \
  --compliance-threshold 80 \
  --fail-coverage-below 95 \
  --fail-on-degraded-config
```

The target remains a command-line argument. The contract describes how to scan;
it does not choose what filesystem path or cluster the caller gives Kubescape.

### Validation without scanning

Validation lives under the scan command rather than the existing
backend-oriented `kubescape config` command:

```bash
kubescape scan validate-contract kubescape.yaml --contract ci
```

Example failure:

```text
kubescape.yaml: spec.contracts.ci.failure.coverageTreshold
unknown field “coverageTreshold”; did you mean “coverageBelow”?
```

The command performs strict schema, path, version, and cross-field validation
without downloading policy or contacting a cluster. It prints the selected
contract’s canonical JSON and digest when `--output json` is requested; a
separate inheritance/rendering feature is not needed in v1alpha1.

## Configuration model

### Version, kind, and binary compatibility

`apiVersion` and `kind` are mandatory. Unsupported versions fail; Kubescape
never treats an unknown version as the latest one.

`spec.minimumKubescapeVersion` is also mandatory and uses semantic versioning.
The loader reads this small version envelope before strictly decoding the rest
of the document. If the running binary is older, it fails with an error such as:

```text
kubescape.yaml requires Kubescape >= v3.1.0; running v3.0.4
```

This check occurs before unknown-field validation so an older binary gives a
useful compatibility error when a newer binary introduced an additive field.
Adding an optional field within v1alpha1 therefore also requires raising
`minimumKubescapeVersion` to the first binary that understands it.

### Named contracts

One file may contain several flat named contracts for local, pull-request, and
release scans. Names are case-sensitive DNS-label-like strings. If `--contract`
is omitted, `spec.defaultContract` is used. If neither selects a name, validation
fails rather than choosing the first map entry.

Unknown fields are rejected at every level. YAML duplicate keys and aliases
that form cycles are also rejected.

### Policy selection

The `policy` section selects frameworks or controls and may pin the regolibrary
release:

```yaml
policy:
  frameworks: [nsa, mitre]
  controls: [C-0013]
  controlsVersion: v2.0.307
  controlsConfigFile: security/kubescape/controls-config.json
```

The contract validator calls shared CLI validation for combinations already
governed by scan flags. `controlsVersion` receives additional validation because
some runner modes bypass the versioned policy download:

- `controlsVersion` with `--account` is an error because the portal selects the
  policy and the requested version would be ignored;
- `controlsVersion` with `--use-artifacts-from`, `--use-from`, or `--use-default`
  is an error because those modes bypass the versioned download;
- the user must remove the contract field or the conflicting runner flag. No
  silent precedence rule is applied.

These errors happen before network access. Provenance is not a substitute for
rejecting a requested policy pin that cannot take effect.

### Scope and evaluation behavior

The initial scope mirrors namespace selection:

```yaml
scope:
  includeNamespaces: [payments, shared]
  excludeNamespaces: [kube-system]
```

Invalid or mutually exclusive combinations use the same rules as their CLI
equivalents. Resource selectors, label expressions, and arbitrary API queries
are deferred.

Evaluation settings use Go-style duration strings:

```yaml
evaluation:
  scanTimeout: 10m
  controlTimeout: 30s
```

Existing timeout relationships remain in force through shared validation.

### Scan settings, policy inputs, and trust-sensitive gates

The schema explicitly separates ordinary settings from failure gates:

| Class | v1alpha1 fields | Resolution rule |
|---|---|---|
| Repository scan settings | `scope.*`, `evaluation.*`, `output.*` | Explicit CLI value overrides the contract |
| Policy-selection inputs | `policy.*` | Explicit CLI value pins and overrides the contract |
| Trust-sensitive gates | `failure.*` | Effective value is the stricter of the contract and explicit CLI floor |
| Runner-only inputs | exceptions, account, credentials, kubeconfig, output path, submission, offline artifact source | Never read from the contract |

Policy selection has no generally correct monotonic ordering: one framework or
controls release is not inherently stricter than another. A protected runner
that treats frameworks, controls, the controls release, or controls
configuration as non-negotiable must therefore pin the corresponding CLI input.
Ordinary explicit-CLI precedence then prevents the contract from replacing it.
The gate-floor rule solves the different problem for thresholds, where a useful
monotonic ordering exists and the contract should still be allowed to tighten
the runner requirement.

Failure gates use existing Kubescape scoring and exit behavior:

```yaml
failure:
  severityAtLeast: high
  complianceBelow: 80
  coverageBelow: 95
  degradedPolicyInput: fail
```

The monotonic comparison is field-specific:

- a lower severity boundary is stricter (`medium` is stricter than `high`);
- a higher compliance or coverage minimum is stricter (`90` is stricter than
  `80`);
- `fail` is stricter than allowing degraded policy input.

The resolver calculates each effective gate as `stricter(contract, CLI floor)`.
It never uses last-writer-wins for a gate. For example, a trusted runner floor of
`coverageBelow: 95` remains 95 if a pull request changes the contract to 70; a
contract value of 98 tightens the effective gate to 98.

A caller that intentionally wants repository gates without a trusted floor may
omit the corresponding CLI gate. This is appropriate when the workflow itself
is repository-controlled. Protected CI should pin every gate it considers
non-negotiable.

The effective value and both sources are included in provenance. If a contract
attempts to weaken a floor, Kubescape emits a concise informational diagnostic;
CI does not need to fail merely because the weaker value was ignored.

### Normative mapping to existing scan inputs

The field-to-CLI mapping is part of the v1alpha1 contract, not an implementation
detail:

| Contract field | Existing scan input |
|---|---|
| `policy.frameworks` | framework selection |
| `policy.controls` | control selection |
| `policy.controlsVersion` | `--controls-version` |
| `policy.controlsConfigFile` | `--controls-config` |
| `scope.includeNamespaces` | `--include-namespaces` |
| `scope.excludeNamespaces` | `--exclude-namespaces` |
| `evaluation.scanTimeout` | `--scan-timeout` |
| `evaluation.controlTimeout` | `--control-timeout` |
| `failure.severityAtLeast` | `--severity-threshold` |
| `failure.complianceBelow` | `--compliance-threshold` |
| `failure.coverageBelow` | `--fail-coverage-below` |
| `failure.degradedPolicyInput` | `--fail-on-degraded-config` |
| `output.formats` | `--format` |
| `output.omitRawResources` | `--omit-raw-resources` |

The typed adapter and CLI use the same underlying option and validation code.
Adding or renaming a field requires updating this normative table.

### Output intent

Only side-effect-free output choices belong in the contract:

```yaml
output:
  formats: [json, sarif]
  omitRawResources: true
```

Output paths, submission, account selection, and encryption keys stay outside
the contract. The runner decides where artifacts are written and whether a
result leaves the machine.

### Path-backed inputs

`controlsConfigFile` is the only path-backed input in v1alpha1. It is resolved
relative to the directory containing `kubescape.yaml`, not the process working
directory.

- Only local relative paths are accepted.
- URLs and absolute paths are rejected.
- Symlinks are resolved before checking confinement.
- `..` or a symlink may not escape the contract directory.
- The file is read and hashed before policy download or cluster access.

`exceptionsFile` is deliberately absent. A later version may add it only behind
an explicit trusted-runner opt-in, such as `--allow-contract-exceptions`; the
contract can never grant that permission to itself.

Helm values and their URL/path semantics are deferred in full.

## Precedence and the trusted gate floor

For ordinary fields, precedence remains ergonomic:

```text
explicit CLI flag > selected contract > existing Kubescape default
```

Cobra’s changed-flag state determines whether a value is explicit; comparing a
value with its default is insufficient.

For gates, precedence is monotonic rather than positional:

```text
effective gate = stricter(selected contract gate, explicit CLI gate floor)
```

This distinction is the security boundary. A protected runner pins any scan or
policy input it considers non-negotiable through ordinary explicit-CLI
precedence. Independently, a repository cannot turn a trusted red build green by
weakening severity, compliance, coverage, or degraded-input requirements.

Exceptions do not participate in either merge. If the trusted runner supplies
`--exceptions`, that runner-selected file is used and its digest is recorded.

## Trust and security model

The threat model assumes the binary, workflow, explicit CLI arguments, and
runner-owned files are trusted, while the repository checkout and scan contract
may be controlled by the pull request being evaluated. Protected CI pins any
scan scope or policy selection it considers mandatory; unset inputs deliberately
delegate that choice to the repository contract.

The contract cannot:

- weaken an explicit runner gate floor;
- select an exceptions file;
- acquire credentials or choose an account;
- choose a cloud API, report URL, kubeconfig, or Kubernetes context;
- submit results or write an arbitrary output path;
- choose a cache or offline artifact directory;
- access a remote URL or escape its local directory;
- execute a plugin, hook, executable, or shell command;
- install an operator or host sensor, or mutate the cluster;
- expand arbitrary environment variables.

Strict unknown-field rejection prevents a forbidden or misspelled field from
looking active while being ignored.

The gate floor protects required workflows and external CI without claiming to
make a repository-controlled workflow immutable. If a pull request can edit the
trusted invocation, branch protection must protect that workflow separately.

### Live-cluster scans

The caller, not the contract, chooses cluster credentials:

```bash
kubescape scan \
  --scan-contract kubescape.yaml \
  --contract ci \
  --kubeconfig /trusted/runner/config \
  --compliance-threshold 80
```

The checked-out repository describes the allowed scan settings; the trusted
workflow chooses the cluster and its gate floor.

## Report provenance

Machine-readable output gains a compact block similar to:

```json
{
  "scanContract": {
    "apiVersion": "config.kubescape.io/v1alpha1",
    "name": "payments-service",
    "contract": "ci",
    "minimumKubescapeVersion": "v3.1.0",
    "digest": "sha256:7c0a...",
    "source": "kubescape.yaml",
    "referencedFiles": [
      {
        "role": "controlsConfig",
        "source": "security/kubescape/controls-config.json",
        "digest": "sha256:a611..."
      }
    ],
    "gateResolution": {
      "coverageBelow": {
        "contract": 70,
        "runnerFloor": 95,
        "effective": 95
      }
    },
    "ordinaryCliOverrides": ["output.formats"]
  }
}
```

The contract digest is SHA-256 over a domain-separated, canonical JSON envelope
containing:

1. the fully defaulted selected contract;
2. the effective values after ordinary overrides and gate-floor resolution;
3. an ordered manifest of every referenced file’s repository-relative role,
   path, and SHA-256 digest.

Whitespace, comments, map order, and unused named contracts do not change the
digest. Changing the contents of `controlsConfigFile` does. Runner-controlled
inputs that affect findings, including an exceptions file supplied explicitly
on the CLI, also receive per-file digests in scan metadata even though they are
not part of the repository contract.

The report carries the concrete controls release from the resolved
controls-version metadata tracked by kubescape/kubescape#2494. If the contract
requests `latest`, the concrete loaded release is recorded. This proposal does
not add a second controls-version metadata path.

Sources inside the repository are repository-relative display paths. For a
runner-owned file outside the repository, provenance records its role and digest
but omits its host path. Absolute workstation paths and file contents are never
embedded in reports.

## CLI integration

Configuration becomes one resolved `ScanInfo` before scan execution:

```text
Cobra defaults and explicit runner flags
    │
    ├── parse --scan-contract and --contract
    ├── read version envelope
    ├── strict decode + flat schema validation
    ├── normalize, confine, and hash referenced files
    ├── map ordinary fields, then re-apply explicit CLI overrides
    ├── resolve each gate against its explicit CLI floor
    ├── reject account/offline-artifact conflicts
    └── run shared cross-field validators
            │
            ▼
        cautils.ScanInfo + provenance
```

The adapter is typed; it does not generate fake CLI arguments and reparse them.
Validation is shared so a value invalid on the CLI cannot become valid in YAML.

No filename is discovered automatically. A scan behaves exactly as it does
today unless `--scan-contract` is present.

## MCP and operator follow-up

The schema should live in a small package without a Cobra dependency so later
consumers can reuse validation. The v1alpha1 CLI phase does not add MCP or
operator inputs.

An MCP consumer would need the same path confinement and must either implement
trusted gate floors or accept only the ordinary subset. An operator transport
would not consume repository path fields directly because those paths have no
meaning in-cluster.

## Backward compatibility

Existing scans do not change unless `--scan-contract` is passed. Existing flags
continue to work.

The file API follows these rules:

- unknown fields are rejected;
- an additive v1alpha1 field raises `minimumKubescapeVersion` to the first
  supporting binary;
- incompatible semantics require a new API version;
- a future stable version should convert from the immediately previous version
  where practical;
- canonical output and provenance always include the API version.

No existing `$HOME/.kubescape/config.json` field is reused or renamed.

## Implementation plan

### Coordination prerequisite: resolved controls metadata

kubescape/kubescape#2494 owns `--controls-version` and stamping the concrete
loaded version into `ScanMetadata`. Its current contributor and maintainers
should be coordinated with before contract provenance work begins. This proposal
depends on that resolved value and removes any duplicate implementation of it.

### Phase 1: flat schema and validation

Repository: `kubescape/kubescape`

- Add typed `v1alpha1` structures with a version envelope.
- Strictly decode YAML and JSON with duplicate-key rejection.
- Select one complete flat contract; do not implement inheritance.
- Normalize, confine, and digest allowed local inputs.
- Add `kubescape scan validate-contract`.
- Test unknown fields, minimum-version errors, invalid durations, duplicate
  keys, path escapes, and account/offline-artifact conflicts.

This phase does not change scan execution.

### Phase 2: CLI application and gate floors

Repository: `kubescape/kubescape`

- Add `--scan-contract` and `--contract` to posture scan commands.
- Map ordinary settings into `ScanInfo` through one typed adapter.
- Preserve ordinary explicit CLI overrides with Cobra changed-flag state.
- Resolve gate values using field-specific monotonic comparisons.
- Keep exceptions CLI-only.
- Run existing validators on the final effective values.
- Test equivalent flag-only and contract-driven scans and every direction of
  every gate-floor comparison.

### Phase 3: contract provenance

Repositories: `kubescape/kubescape` and, if shared report types require it,
`kubescape/opa-utils`

- Build on the resolved controls-version metadata from #2494.
- Add contract identity, canonical effective digest, referenced-file digests,
  gate resolution, and ordinary override names to JSON/YAML metadata.
- Digest runner-controlled files that influence findings.
- Ensure absolute paths and file contents are absent.
- Add round-trip, digest-domain, and deterministic-canonicalization tests.

## Testing strategy

Parser tests cover strict decoding, version-envelope ordering, duplicate keys,
canonical digest stability, and path confinement. Integration tests prove:

- ordinary fields obey explicit CLI precedence;
- a contract can tighten but cannot weaken each trusted gate floor;
- exceptions cannot be loaded from a contract;
- `controlsVersion` conflicts with account and offline artifact modes before
  any network or cluster access;
- changing referenced file content changes its digest and the contract digest;
- equivalent flag-only and contract-driven scans build the same effective scan
  options;
- malformed input cannot access secrets, remote URLs, or host paths;
- reports contain neither secrets nor absolute workstation paths.

## Alternatives considered

- **Keep commands in CI YAML.** Simple for one CI system, but local tools and
  reports still lack the same reviewed contract.
- **Use normal CLI precedence for every field.** Rejected because it treats the
  security control as ergonomics and relies on the runner to repeat every gate
  to prevent a pull request from weakening itself.
- **Load the contract from a base Git ref.** Protects the full file but prevents
  ordinary scan-setting changes from being reviewed and tested in the pull
  request. A monotonic gate floor protects acceptance criteria while allowing
  repository-local evolution.
- **Pin and assert the whole contract digest in CI.** Strong but operationally
  expensive: every harmless settings change requires a trusted workflow update.
  Digests remain useful for audit, not as the default authorization mechanism.
- **Allow contract-controlled exceptions.** Deferred because exceptions can
  directly hide findings. A future explicit trusted-runner opt-in can be
  considered independently.
- **Add inheritance and list merge rules.** Deferred until real flat contracts
  demonstrate a need that YAML anchors cannot meet.
- **Include Helm values.** Deferred because `--values` accepts URLs and would
  introduce conflicting path and trust semantics in v1alpha1.
- **Automatically discover `kubescape.yaml`.** Rejected because implicit loading
  combined with repository-controlled behavior is surprising and unsafe.
- **Reuse `$HOME/.kubescape/config.json`.** Rejected because it represents
  tenant/backend state and may contain credentials.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| A pull request weakens its own protected CI gate | Explicit runner gate values are monotonic floors; the contract may only tighten them |
| A pull request adds an exception for its own finding | Exceptions are not representable in v1alpha1 and remain runner-controlled |
| The file becomes a second CLI | Start with a flat allowlist mapped normatively to existing inputs; no inheritance or Helm fields |
| A malicious repository exfiltrates data or accesses host paths | No credentials, submission, URLs, hooks, output paths, or path escapes; strict decoding |
| A digest stays stable while a referenced input changes | Record per-file digests and include their manifest in the effective contract digest |
| A requested controls version is silently ignored | Reject account and offline-artifact conflicts before scanning; report the concrete loaded version |
| A newer optional field breaks an older binary opaquely | Read `minimumKubescapeVersion` before strict decoding and report the required version |
| Contract and flag validation drift | Use a typed adapter followed by shared validators |
| Reports leak workstation paths | Store repository-relative display paths and digests only |

## Success criteria

The proposal is successful when:

- a repository can define complete local and CI contracts in one explicit file;
- protected CI can pin gate floors that no repository edit can weaken;
- a repository contract can still tighten those gates;
- exceptions remain outside the default repository trust boundary;
- a typo or incompatible binary fails before the scan begins;
- account and offline artifact conflicts cannot silently ignore
  `controlsVersion`;
- changing any referenced input changes its recorded digest and effective
  contract digest;
- a JSON report identifies the selected contract, effective gates, concrete
  controls release, CLI overrides, and all relevant input digests;
- no repository contract can access runner credentials, create an external side
  effect, or escape its directory.

## Open questions

1. Is `ScanContract` / `--scan-contract` sufficiently distinct from both the
   existing `--controls-config` flag and Kubescape `ApplicationProfile` concept?
2. Should a later version expose an explicit runner-only opt-in for
   contract-controlled exceptions, or keep exceptions permanently CLI-only?
3. Where should the contract provenance block live in v1 and v2 report schemas
   so every renderer preserves it?
4. Should an attempted gate-floor weakening be informational, a warning, or an
   optional hard error for especially strict CI?
5. Should the MCP server accept the ordinary subset only, or define its own
   trusted gate-floor inputs?

## Decision requested

Before implementation, this proposal asks for agreement on:

1. the monotonic trusted-runner gate floor as the security mechanism for
   repository-controlled contracts;
2. the flat, explicit, exception-free v1alpha1 boundary;
3. the compatibility and provenance rules, including digest coverage for every
   referenced input;
4. coordination with kubescape/kubescape#2494 rather than duplicate
   controls-version provenance work.

If accepted, the flat schema and validation can land without changing any
existing scan. Gate application and provenance then follow as independently
reviewable phases.

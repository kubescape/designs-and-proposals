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

Protected runners also authorize whole contract sections with
`--contract-allow`. This prevents a pull request from changing the policy or
scan scope that feeds a trusted threshold merely because the runner forgot to
pin one of several related flags.

The report records the effective contract, the concrete controls release, and
digests for runner-owned files that affect findings. It can therefore answer
both “what failed?” and “under exactly which inputs and acceptance rules was
this run?”

This proposal does not move every Kubescape flag into YAML. Credentials, cloud
endpoints, submission, arbitrary output paths, exceptions, and cluster-mutating
behavior remain outside the repository-controlled schema.

## Motivation

Kubescape already exposes the pieces needed for reproducible scanning, but the
caller must assemble them on every invocation. A scan can choose frameworks,
controls, a regolibrary release, namespace scope, failure thresholds, timeouts,
and output behavior.

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
controls release, runner-owned control configuration, and coverage and severity
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
- Produce the same effective inputs for the same contract, runner-owned input
  digests, repository commit, runner floors, and Kubescape version.
- Reject unknown or malformed fields before policy download or cluster access.
- Keep the first schema flat, small, and directly traceable to existing CLI
  behavior.
- Record enough provenance to reproduce or audit a run.
- Keep credentials, exceptions, submission, arbitrary writes, remote inputs,
  and cluster mutation outside the default repository trust boundary.
- Let a protected runner authorize contract sections with one explicit switch.
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
      evaluation:
        controlTimeout: 30s
      output:
        formats: [pretty-printer]

    ci:
      policy:
        frameworks: [nsa]
        controlsVersion: v2.0.307
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

CI names its contract, limits which contract sections it accepts, and pins
non-negotiable gate floors in the trusted invocation:

```bash
kubescape scan . \
  --scan-contract kubescape.yaml \
  --contract ci \
  --contract-allow=policy,scope,evaluation,output,failure \
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

The command performs strict schema, version, and cross-field validation
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
that form cycles are also rejected. The strict decoder uses the alias-expansion
limits already enforced by Kubescape's `gopkg.in/yaml.v3` dependency through
its `allowedAliasRatio` check; parser tests pin that behavior rather than
introducing a second alias limiter.

### Policy selection

The `policy` section selects frameworks or controls and may pin the regolibrary
release:

```yaml
policy:
  frameworks: [nsa, mitre]
  controls: [C-0013]
  controlsVersion: v2.0.307
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
| Runner-only inputs | exceptions, controls configuration, account, credentials, kubeconfig, output path, submission, offline artifact source | Never read from the contract |

Policy selection has no generally correct monotonic ordering: one framework or
controls release is not inherently stricter than another. A protected runner
that treats frameworks, individual controls, or the controls release as
non-negotiable must therefore pin the corresponding CLI input.
Ordinary explicit-CLI precedence then prevents the contract from replacing it.
Controls configuration remains runner-only because it can change the inputs to
posture controls without an exception. The gate-floor rule solves the different
problem for thresholds, where a useful monotonic ordering exists and the
contract should still be allowed to tighten the runner requirement.

#### Authorizing contract sections

Explicit value precedence is not a complete protected-CI boundary: a repository
can still shrink an unpinned policy or namespace scope before a threshold is
calculated. The trusted runner can therefore authorize contract sections with:

```text
--contract-allow=policy,scope,evaluation,output,failure
```

The accepted names are the five top-level sections shown above. The option may
be repeated or comma-separated; values are flattened and de-duplicated. If the
flag is omitted, all sections are accepted for ergonomic local use. When it is
present, an unauthorized section is not applied. Kubescape emits a concise
diagnostic and records the section name in provenance, so the omission is never
silent.

For example, a protected workflow that owns policy and scope can use:

```bash
kubescape scan . framework nsa \
  --scan-contract kubescape.yaml \
  --contract ci \
  --contract-allow=evaluation,output,failure \
  --controls-version v2.0.307 \
  --exclude-namespaces kube-system \
  --severity-threshold high \
  --compliance-threshold 80 \
  --fail-coverage-below 95 \
  --fail-on-degraded-config
```

This single authorization switch ensures a future repository edit cannot add a
policy or scope field that the trusted invocation forgot to pin. The explicit
framework, controls release, and namespace flags define what that runner
considers mandatory; the `policy` and `scope` sections in the selected contract
are reported as denied and do not participate in resolution.

#### List replacement

Lists do not merge across the contract and CLI. If the CLI source is explicit,
its complete normalized list replaces the contract list; otherwise the contract
list replaces the Kubescape default. Repeated and comma-separated CLI flags are
first accumulated using the existing flag parser and then treated as that one
replacement list. Ordering is preserved and duplicates are removed by the
shared validator.

Presence is represented separately from value. An explicit empty CLI list is
therefore different from an omitted flag: it replaces the contract list with an
empty list, after which ordinary cross-field validation decides whether that
field may legally be empty. This rule applies uniformly to
`policy.frameworks`, `policy.controls`, `scope.includeNamespaces`,
`scope.excludeNamespaces`, and `output.formats`; v1alpha1 has no append mode.

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

Gate fields retain presence during decoding. `complianceBelow` and
`coverageBelow` accept values from 0 through 100; omission means “the repository
did not set a gate,” while an explicit zero is a present, disabled repository
gate and is recorded as such. `degradedPolicyInput` is the enum `allow|fail`;
`allow` maps to `false`, `fail` maps to `true`, and omission is distinct from
both. `severityAtLeast` accepts the severity values supported by the existing
CLI and rejects an explicit empty value.

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

If only one source supplies a gate, that value is effective. If neither source
supplies it, existing Kubescape defaults apply and provenance marks the gate as
unset. An explicit CLI zero or `false` remains present even though it is not a
useful floor; a stricter contract value still wins. This keeps “not supplied”
separate from “supplied and disabled” without changing current flag behavior.

### Normative mapping to existing scan inputs

The field-to-CLI mapping is part of the v1alpha1 contract, not an implementation
detail:

| Contract field | Existing scan input |
|---|---|
| `policy.frameworks` | framework selection |
| `policy.controls` | control selection |
| `policy.controlsVersion` | `--controls-version` |
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

### Runner-owned file inputs

v1alpha1 has no path-backed fields. Both `--controls-config` and `--exceptions`
can change findings without changing a threshold, so they remain explicit,
runner-owned CLI inputs. A later API version may add a repository path only
behind a trusted-runner opt-in; the contract can never grant that permission to
itself.

When the runner supplies either file, Kubescape reads the bytes used by the scan
once, computes their digest from those same bytes, and records the role and
digest in provenance. This avoids a hash-then-reopen race while keeping the
repository contract unable to select the file.

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
precedence and limits accepted sections with `--contract-allow`. Independently,
a repository cannot turn a trusted red build green by weakening severity,
compliance, coverage, or degraded-input requirements.

Exceptions do not participate in either merge. If the trusted runner supplies
`--exceptions`, that runner-selected file is used and its digest is recorded.

## Trust and security model

The threat model assumes the binary, workflow, explicit CLI arguments, and
runner-owned files are trusted, while the repository checkout and scan contract
may be controlled by the pull request being evaluated. Protected CI pins any
scan scope or policy selection it considers mandatory and uses
`--contract-allow` to reject untrusted sections; unset, authorized inputs
deliberately delegate that choice to the repository contract.

The contract cannot:

- weaken an explicit runner gate floor;
- select an exceptions file;
- select a controls configuration file;
- acquire credentials or choose an account;
- choose a cloud API, report URL, kubeconfig, or Kubernetes context;
- submit results or write an arbitrary output path;
- choose a cache or offline artifact directory;
- access a remote URL or select a local input file;
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
    "digestSchema": "kubescape.io/scan-contract-digest/v1",
    "contractDigest": "sha256:7c0a...",
    "effectiveRunDigest": "sha256:f19b...",
    "source": "kubescape.yaml",
    "allowedSections": ["evaluation", "output", "failure"],
    "deniedSections": ["policy", "scope"],
    "effective": {
      "policy": {
        "frameworks": ["nsa"],
        "controlsVersion": "v2.0.307"
      },
      "scope": {
        "excludeNamespaces": ["kube-system"]
      },
      "evaluation": {
        "scanTimeout": "10m",
        "controlTimeout": "30s"
      },
      "failure": {
        "severityAtLeast": "high",
        "complianceBelow": 80,
        "coverageBelow": 95,
        "degradedPolicyInput": "fail"
      },
      "output": {
        "formats": ["json", "sarif"],
        "omitRawResources": true
      }
    },
    "runnerInputs": [
      {
        "role": "controlsConfig",
        "source": "security/runner/controls-config.json",
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
    "ordinaryCliOverrides": {
      "output.formats": ["json", "sarif"]
    }
  }
}
```

`contractDigest` identifies only the selected repository contract. Its input is
the UTF-8 encoding of the ASCII domain separator
`kubescape-scan-contract:v1\0` followed by the normalized selected contract,
including schema defaults while retaining optional-field presence, serialized
with the JSON Canonicalization Scheme in RFC 8785. RFC 8785 defines object-key
ordering, number serialization, strings, booleans, and null; non-finite numbers
are rejected before hashing. Absent optional fields remain absent, while
explicit zero, false, empty lists, and null remain represented. Whitespace,
comments, YAML map order, and unused named contracts do not change this digest.

`effectiveRunDigest` identifies what actually fed the scan. It hashes the UTF-8
domain separator `kubescape-effective-run:v1\0` followed by an RFC 8785 envelope
containing the five resolved `effective` sections, the section-authorization
decision, the concrete controls release, the Kubescape version, and the
`runnerInputs` manifest. Manifest entries are sorted lexicographically by
`(role, source)` before canonicalization and use lowercase hexadecimal SHA-256
values. File digests cover the exact raw bytes consumed by the scan. The digest
schema string is part of each envelope so a future canonicalization or
field-boundary change requires a new schema version rather than silently
changing existing digests.

The effective block contains the typed post-resolution values, not only the
names of changed flags. `ordinaryCliOverrides` records the normalized explicit
CLI values that replaced contract values, including lists such as
`output.formats`. Gate provenance separately records contract presence, runner
floor presence, and the effective result, so omitted, zero, and false remain
distinguishable. `allowedSections` and `deniedSections` explain which repository
sections were authorized by the runner. Credentials, access keys, account IDs,
kubeconfig contents, encryption keys, and other secrets are never included.

The report carries the concrete controls release from the resolved
controls-version metadata tracked by kubescape/kubescape#2494. If the contract
requests `latest`, the concrete loaded release is recorded. This proposal does
not add a second controls-version metadata path.

Runner-controlled files that affect findings, including `--exceptions` and
`--controls-config`, appear at `scanMetadata.scanContract.runnerInputs`. A source
inside the repository uses a repository-relative display path. For a
runner-owned file outside the repository, provenance records its role and digest
and uses the non-path label `external`; it never records a host path. Absolute
workstation paths and file contents are never embedded in reports.

## CLI integration

Configuration becomes one resolved `ScanInfo` before scan execution:

```text
Cobra defaults and explicit runner flags
    │
    ├── parse --scan-contract and --contract
    ├── read version envelope
    ├── strict decode + flat schema validation
    ├── authorize sections with --contract-allow
    ├── map ordinary fields, then re-apply explicit CLI overrides
    ├── resolve each gate against its explicit CLI floor
    ├── hash the bytes of runner-owned files used by the scan
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

An MCP consumer must either implement trusted gate floors and section
authorization or accept only the ordinary subset. An operator transport would
need its own trusted configuration boundary; it does not inherit CLI-owned file
inputs.

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
- Add `kubescape scan validate-contract`.
- Test unknown fields, minimum-version errors, invalid durations, duplicate
  keys, decoder alias limits, and account/offline-artifact conflicts.

This phase does not change scan execution.

### Phase 2: CLI application and gate floors

Repository: `kubescape/kubescape`

- Add `--scan-contract` and `--contract` to posture scan commands.
- Add `--contract-allow`, skip unauthorized sections with a diagnostic, and
  record the decision in provenance.
- Map ordinary settings into `ScanInfo` through one typed adapter.
- Preserve ordinary explicit CLI overrides with Cobra changed-flag state.
- Apply whole-list replacement consistently, including repeated and explicit
  empty CLI values.
- Resolve gate values using field-specific monotonic comparisons.
- Keep exceptions CLI-only.
- Run existing validators on the final effective values.
- Test equivalent flag-only and contract-driven scans and every direction of
  every gate-floor comparison.

### Phase 3: contract provenance

Repositories: `kubescape/kubescape` and, if shared report types require it,
`kubescape/opa-utils`

- Build on the resolved controls-version metadata from #2494.
- Add contract identity, separate contract and effective-run digests, effective
  typed values, runner-file digests, gate resolution, and normalized ordinary
  override values to JSON/YAML metadata.
- Digest runner-controlled files that influence findings.
- Ensure absolute paths and file contents are absent.
- Add round-trip, digest-domain, and deterministic-canonicalization tests.

## Testing strategy

Parser tests cover strict decoding, version-envelope ordering, duplicate keys,
decoder alias limits, optional-field presence, and canonical digest stability.
Integration tests prove:

- ordinary fields obey explicit CLI precedence;
- every list field has replacement semantics independent of repeated-flag order,
  and explicit empty remains distinct from omission;
- unauthorized policy or scope sections do not affect the scan and are reported;
- a contract can tighten but cannot weaken each trusted gate floor;
- omitted, explicit zero, and explicit `allow` gate states resolve and report
  distinctly;
- exceptions cannot be loaded from a contract;
- `controlsVersion` conflicts with account and offline artifact modes before
  any network or cluster access;
- changing a runner-owned input changes its file digest and effective-run digest
  without changing the repository contract digest;
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
| A pull request weakens its own protected CI gate | Runner-authorized sections constrain inputs; explicit gate values are monotonic floors |
| A pull request adds an exception for its own finding | Exceptions are not representable in v1alpha1 and remain runner-controlled |
| The file becomes a second CLI | Start with a flat allowlist mapped normatively to existing inputs; no inheritance or Helm fields |
| A malicious repository exfiltrates data or accesses host paths | No contract path fields, credentials, submission, URLs, hooks, or output paths; strict decoding |
| A digest hides a runner-input change | Keep the repository contract digest separate and include runner-input manifests in the effective-run digest |
| A requested controls version is silently ignored | Reject account and offline-artifact conflicts before scanning; report the concrete loaded version |
| A newer optional field breaks an older binary opaquely | Read `minimumKubescapeVersion` before strict decoding and report the required version |
| Contract and flag validation drift | Use a typed adapter followed by shared validators |
| Reports leak workstation paths | Store repository-relative display paths and digests only |

## Success criteria

The proposal is successful when:

- a repository can define complete local and CI contracts in one explicit file;
- protected CI can pin gate floors that no repository edit can weaken;
- protected CI can prevent repository-controlled policy or scope with one
  section-authorization flag;
- a repository contract can still tighten those gates;
- exceptions remain outside the default repository trust boundary;
- a typo or incompatible binary fails before the scan begins;
- account and offline artifact conflicts cannot silently ignore
  `controlsVersion`;
- changing any runner-owned input changes its recorded digest and effective-run
  digest without changing the repository contract digest;
- a JSON report identifies the selected contract, all effective inputs,
  effective gates, concrete controls release, normalized CLI overrides, and all
  relevant runner-input digests;
- no repository contract can access runner credentials, select a local file, or
  create an external side effect.

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

1. section authorization plus monotonic trusted-runner gate floors as the
   security mechanism for repository-controlled contracts;
2. the flat, explicit, exception-free v1alpha1 boundary;
3. the compatibility and provenance rules, including separate repository and
   effective-run digests;
4. coordination with kubescape/kubescape#2494 rather than duplicate
   controls-version provenance work.

If accepted, the flat schema and validation can land without changing any
existing scan. Gate application and provenance then follow as independently
reviewable phases.

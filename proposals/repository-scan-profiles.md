# Proposal: Versioned Repository Scan Profiles

- **Status:** Draft
- **Related issue:**
  [kubescape/kubescape#2494 — Allow pinning the controls version](https://github.com/kubescape/kubescape/issues/2494)
- **Initial scope:** [`kubescape`](https://github.com/kubescape/kubescape) CLI
- **Possible later consumers:** Kubescape MCP server and operator
- **Author:** Daksh Pathak (<daksh.pathak.ug24@nsut.ac.in>)

## Summary

Let a project commit a safe, versioned Kubescape scan profile beside its
manifests. The profile describes the policy version, frameworks, scope,
evaluation limits, and failure gates that make up the project's security scan.

Today those choices are usually spread across a CI workflow, Makefile, local
shell history, and platform-specific configuration. Two people can scan the
same commit with Kubescape and receive different verdicts without either run
being obviously wrong. A repository profile would make the intended scan
contract reviewable and repeatable.

The first version would be explicit rather than magical:

```bash
kubescape scan . --scan-config kubescape.yaml --profile ci
```

The proposal also records the resolved profile and its digest in machine-readable
scan output. A report can then answer not only "what failed?", but also "under
which scan contract was this evaluated?"

This is not a proposal to move every Kubescape flag into YAML. Credentials,
cloud endpoints, submission, output paths, and cluster-mutating behavior are
deliberately excluded. A repository checkout is often untrusted input in CI,
so the file must be useful without being able to take over the process that
reads it.

## Motivation

Kubescape already exposes the pieces needed for reproducible scanning, but the
caller must assemble them on every invocation. Among other things, a scan can
choose:

- frameworks or individual controls;
- a regolibrary release through `--controls-version`;
- local control configuration and exceptions;
- included and excluded namespaces;
- compliance, severity, and coverage failure thresholds;
- whole-scan and per-control timeouts;
- output format and raw-resource handling;
- Helm values and release context for chart scans.

These are not merely presentation preferences. They change the input, the
policy, or the pass/fail behavior of a scan.

In practice, projects encode the choices in several places:

```text
.github/workflows/security.yml    frameworks + thresholds
Makefile                          local developer command
scripts/scan.sh                   namespace scope + exceptions
README.md                         policy-version expectations
```

That creates a few recurring problems.

### Local and CI scans quietly drift

A developer may run `kubescape scan . framework nsa`, while CI uses a pinned
controls release, a custom exceptions file, and a severity threshold. Both
commands are valid, but they are not evaluating the same contract.

The difference only becomes visible after a surprising result, and sometimes
not even then. A JSON report describes findings well, but does not currently
carry a portable description of every input that selected and configured the
policy.

### Policy updates are not reviewed with application changes

Pinning `--controls-version` is useful, but a pin living only in a CI workflow
is easy to miss. The same is true for exception files and failure gates. If the
scan contract is repository data, updating it becomes a normal reviewed change:

```diff
- controlsVersion: v2.0.301
+ controlsVersion: v2.0.307
```

That gives reviewers a clear place to ask why the policy changed and lets
dependency tooling eventually propose controlled updates.

### Other Kubescape entry points need the same semantics

The MCP IaC scan currently accepts a path and an optional framework. The CLI
has a much richer set of scan semantics. The operator has its own configuration
surface again. It would be useful to converge on one validated profile model,
even if the CLI is the only consumer in the first phase.

This proposal does not require immediate parity across all three paths. It
defines a portable, side-effect-free core that later callers can reuse instead
of each inventing another configuration format.

## Goals

- Make the security-relevant semantics of a Kubescape scan commit-able and
  reviewable with the workload it scans.
- Produce the same resolved scan inputs when the same profile, repository
  commit, and Kubescape version are used.
- Keep command-line overrides possible and make precedence unambiguous.
- Reject unknown or malformed fields rather than silently ignoring a typo.
- Resolve referenced files relative to the profile, consistently across local
  development and CI.
- Record enough profile provenance in machine-readable reports to reproduce or
  audit a run.
- Treat a checked-out repository as untrusted input and keep credentials,
  submission, arbitrary writes, and cluster mutation outside the schema.
- Start with the CLI while choosing a model that the MCP server and operator
  can reuse later.

## Non-goals

- Replacing `$HOME/.kubescape/config.json`. That file stores tenant and backend
  configuration; a repository profile has a different trust boundary and
  purpose.
- Supporting every `kubescape scan` flag in YAML.
- Storing access keys, account IDs, master encryption keys, cloud URLs, or
  other secrets in a repository.
- Allowing a profile to submit results, write to an arbitrary output path,
  install components, change a cluster, or choose a kubeconfig.
- Guaranteeing identical findings across different Kubescape binaries or
  changing external policy artifacts. The report records those identities so
  differences can be explained.
- Automatically trusting a profile found in a repository in the first phase.
- Designing a remote organization-wide policy distribution service. This is a
  local file contract first.

## Proposed user experience

### A repository-owned profile

A project adds `kubescape.yaml`:

```yaml
apiVersion: config.kubescape.io/v1alpha1
kind: ScanConfiguration
metadata:
  name: payments-service
spec:
  defaultProfile: developer

  profiles:
    developer:
      policy:
        frameworks:
          - nsa
        controlsVersion: v2.0.307
        exceptionsFile: security/kubescape/exceptions.json
        controlsConfigFile: security/kubescape/controls-config.json
      evaluation:
        controlTimeout: 30s
      output:
        formats:
          - pretty-printer

    ci:
      extends:
        - developer
      scope:
        excludeNamespaces:
          - kube-system
      evaluation:
        scanTimeout: 10m
      failure:
        severityAtLeast: high
        complianceBelow: 80
        coverageBelow: 95
        degradedPolicyInput: fail
      output:
        formats:
          - json
          - sarif
        omitRawResources: true
```

Developers use the default profile:

```bash
kubescape scan . --scan-config kubescape.yaml
```

CI selects the stricter profile:

```bash
kubescape scan . --scan-config kubescape.yaml --profile ci
```

The target remains a command-line argument. The profile describes how to scan;
it does not decide what filesystem or cluster target the caller gives
Kubescape.

### Validation without scanning

Kubescape should validate the file independently of a scan:

```bash
kubescape config validate-scan-profile kubescape.yaml
```

Example failure:

```text
kubescape.yaml: spec.profiles.ci.failure.coverageTreshold
unknown field "coverageTreshold"; did you mean "coverageBelow"?
```

Strict validation matters here. Silently accepting a misspelled security gate
would make the file look enforced when it is not.

### Explaining the resolved contract

Before relying on a profile in CI, a user should be able to inspect the fully
resolved form after inheritance and defaults:

```bash
kubescape config render-scan-profile kubescape.yaml --profile ci
```

The output is canonical YAML or JSON with:

- inheritance flattened;
- defaults made explicit;
- relative paths normalized;
- list merge behavior already applied;
- no environment secrets or machine-specific absolute paths.

This command is also useful in code review and tests. Two configurations that
look different but resolve to the same contract should render identically.

## Configuration model

### Version and kind

The top-level `apiVersion` and `kind` are mandatory. `v1alpha1` makes the
experimental status honest and leaves room to adjust the schema after real
usage.

Unsupported versions fail with a message naming the versions understood by the
current binary. Kubescape must never treat an unknown version as the latest
one.

### Named profiles

One file may contain several named profiles because local, pull-request, and
release scans often share most of their configuration but use different gates
or output formats.

Profile names are case-sensitive DNS-label-like strings. If `--profile` is
omitted, `spec.defaultProfile` is used. If neither is present, validation
fails rather than choosing the first map entry.

### Inheritance

`extends` is intentionally limited:

- profiles may extend profiles in the same file only;
- cycles are rejected with the full chain in the error;
- a profile may list more than one parent;
- parents are applied left to right, then the child;
- scalar and map values use last-writer-wins;
- list behavior is defined per field, not guessed globally.

For the initial schema, policy identifiers and namespace lists replace the
parent list when present. This is less surprising than hidden concatenation.
If additive composition proves necessary, it should be expressed explicitly
later rather than overloaded onto ordinary YAML lists.

### Policy selection

The `policy` section can select frameworks or controls and pin the release that
supplies them:

```yaml
policy:
  frameworks: [nsa, mitre]
  controls: [C-0013]
  controlsVersion: v2.0.307
  exceptionsFile: security/kubescape/exceptions.json
  controlsConfigFile: security/kubescape/controls-config.json
```

The exact compatibility rules should match the CLI. If the CLI rejects a
combination, the profile validator rejects it too. The implementation should
call shared validation code instead of building a second set of rules for the
YAML path.

`controlsVersion` identifies a regolibrary release. This proposal builds on
the direction of kubescape/kubescape#2494 by making the pin part of a reviewed
scan contract rather than an easily forgotten command-line detail.

### Scope

The initial scope model mirrors namespace selection already supported by the
CLI:

```yaml
scope:
  includeNamespaces: [payments, shared]
  excludeNamespaces: [kube-system]
```

Mutually exclusive or otherwise invalid combinations use the same rules as
their flag equivalents. Resource selectors, label expressions, and arbitrary
API queries are deferred until Kubescape has one clear CLI model for them.

### Evaluation behavior

Evaluation settings affect whether a scan is complete and how long it may run:

```yaml
evaluation:
  scanTimeout: 10m
  controlTimeout: 30s
```

Durations use Go-style duration strings, matching the CLI. The existing
constraint that a per-control timeout must leave room inside the whole-scan
timeout remains in force.

### Failure gates

Failure gates turn findings and degraded coverage into a CI contract:

```yaml
failure:
  severityAtLeast: high
  complianceBelow: 80
  coverageBelow: 95
  degradedPolicyInput: fail
```

These fields map to existing scan behavior rather than introducing another
scoring model. A profile should not change how Kubescape computes compliance or
coverage; it only declares when the caller considers the result unacceptable.

### Output intent

Only side-effect-free output choices belong in the profile:

```yaml
output:
  formats: [json, sarif]
  omitRawResources: true
```

An output file path does not belong here. CI decides where artifacts are
written, and a repository file should not be able to overwrite an arbitrary
host path. The caller can still pass `--output` explicitly.

Likewise, `submit`, account selection, and encryption keys stay outside the
profile. Whether a result leaves the machine is a runner decision, not
repository data.

### Path-backed inputs

`exceptionsFile` and `controlsConfigFile` are resolved relative to the
directory containing `kubescape.yaml`, not the current working directory.
This makes the same command work from the repository root and from a nested CI
working directory.

For v1alpha1:

- only local relative paths are accepted;
- URLs are rejected;
- symlinks are resolved before the trust-boundary check;
- escaping the profile directory with `..` or a symlink is rejected by
  default;
- a future explicit CLI opt-in may allow external paths for advanced local
  use, but the profile cannot grant itself that permission.

Helm value files have the same path problem, but adding them to v1alpha1 should
be decided separately. If included, they must obey the identical confinement
rules. Inline Helm `--set` equivalents can wait until there is a clear escaping
and merge contract.

## Precedence

The proposed precedence is:

```text
explicit CLI flag > selected profile > existing Kubescape default
```

Environment and cached tenant configuration continue to supply only the
machine or account concerns they own. They do not silently rewrite fields that
are part of the repository scan contract.

The important word is **explicit**. Cobra-backed options already know whether
a flag was changed by the caller. The loader should use that signal instead of
comparing a value to its default, because a caller may intentionally pass the
default value to override a profile.

For example:

```bash
# ci says severityAtLeast: high; this one run explicitly disables that gate
kubescape scan . --scan-config kubescape.yaml --profile ci --severity-threshold ""
```

Kubescape should log a concise debug-level explanation of overrides and expose
them in rendered-profile output. It should not print noisy warnings for normal
CLI precedence.

## Trust and security model

The profile may be read from a pull-request checkout. That makes it attacker
controlled in many CI jobs, even when the Kubescape binary and workflow are
trusted.

The allowed schema therefore follows a capability rule: a profile may narrow
or configure the scan, but it may not acquire ambient credentials, cause a new
external side effect, or select arbitrary host resources.

### Explicitly forbidden fields

The schema has no representation for:

- account ID or access key;
- cloud API or report URL;
- automatic result submission;
- `KUBESCAPE_MASTER_KEY` or any encryption material;
- kubeconfig path or Kubernetes context;
- cache directory;
- arbitrary output path;
- plugin, hook, executable, or shell command;
- operator installation, host-sensor deployment, or another cluster mutation;
- arbitrary environment-variable expansion;
- remote `http://`, `https://`, or `git://` input files.

Unknown fields fail validation, so adding one of these names does not degrade
into an ignored but convincing-looking setting.

### Policy downloads

A pinned `controlsVersion` may cause Kubescape to retrieve an official
regolibrary release through its existing policy getter. That is existing scan
behavior, not a general URL capability. The profile cannot choose the download
host.

For fully disconnected environments, existing local artifact mechanisms remain
available as explicit runner configuration. Whether a constrained local
artifact directory belongs in a later schema version is an open question.

### Live-cluster scans

The same profile can describe policy, scope, and failure gates for a live scan,
but it cannot choose credentials or a context:

```bash
kubescape scan --scan-config kubescape.yaml --profile ci \
  --kubeconfig /trusted/runner/config
```

Here the trusted workflow chooses the cluster. The checked-out repository only
describes how Kubescape evaluates it.

## Report provenance

A versioned profile is only useful for audit if the report says which resolved
contract was used.

Machine-readable output should gain a compact provenance block similar to:

```json
{
  "scanConfiguration": {
    "apiVersion": "config.kubescape.io/v1alpha1",
    "name": "payments-service",
    "profile": "ci",
    "digest": "sha256:7c0a...",
    "source": "kubescape.yaml",
    "overrides": ["output.formats"]
  }
}
```

The digest is calculated from the canonical resolved profile, not the raw YAML.
Whitespace, comments, map order, and unused profiles therefore do not change
it. A changed effective setting does.

The report should also carry the resolved controls version when one is known.
If the profile asks for "latest", the report records the concrete release that
was actually loaded. Reproduction depends on resolved identities, not only the
user's request.

The source is a repository-relative display path. Absolute workstation paths
must not leak into reports.

Pretty output can show a single summary line:

```text
Scan profile: payments-service/ci (sha256:7c0a..., controls v2.0.307)
```

## CLI integration

The CLI implementation should have one point where configuration becomes a
resolved `ScanInfo` rather than sprinkling profile checks through command
handlers.

A possible flow is:

```text
Cobra defaults
    │
    ├── parse --scan-config and --profile
    │
    ├── strict decode + schema validation
    │
    ├── resolve inheritance and relative paths
    │
    ├── map profile fields into scan options
    │
    ├── re-apply explicitly changed CLI flags
    │
    └── run existing cross-field validators
            │
            ▼
        cautils.ScanInfo
```

The mapping layer should be typed. Converting the YAML into fake CLI arguments
and reparsing them would couple the file format to flag spelling and make
errors harder to locate.

At the same time, validation logic must be shared. A timeout combination or
threshold that is invalid on the command line should not become valid merely
because it came from YAML.

## MCP and operator follow-up

The schema should live in a small package that does not depend on Cobra. That
allows later consumers to validate and resolve the same contract.

### MCP server

The local IaC tool could eventually accept `scan_config` and `profile` alongside
the path. The MCP server must apply the same path confinement because the tool
may be invoked by a model against an untrusted workspace.

The first CLI phase does not add those MCP arguments. It only avoids making
their later implementation a second parser and schema.

### Operator

An operator could consume the safe policy, scope, evaluation, and failure
sections from a ConfigMap or a future CRD. It should not consume repository
path fields directly, since they have no meaning in-cluster.

Sharing the complete YAML file byte-for-byte may therefore be less useful than
sharing the typed core model. Consumer-specific transport and source fields can
remain separate.

## Backward compatibility

Existing scans do not change unless `--scan-config` is passed. Every existing
flag continues to work.

The file API follows Kubernetes-style versioning conventions:

- additive optional fields may be introduced within `v1alpha1` while the
  feature is experimental;
- incompatible semantic changes require a new version;
- the decoder rejects unknown fields;
- a future stable version should ship conversion from the immediately previous
  version where practical;
- rendered canonical output includes the schema version.

No existing `$HOME/.kubescape/config.json` field is reused or renamed.

## Implementation plan

### Phase 1: schema, parser, and validation

Repository: `kubescape/kubescape`

- Add typed `v1alpha1` configuration structures.
- Strictly decode YAML and JSON.
- Select a profile and resolve same-file inheritance.
- Normalize and confine local referenced paths.
- Produce a canonical resolved representation and SHA-256 digest.
- Add `kubescape config validate-scan-profile` and
  `render-scan-profile` commands.
- Table-test valid documents, unknown fields, cycles, invalid durations, and
  path escapes.

This phase does not change scan execution.

### Phase 2: CLI application and precedence

Repository: `kubescape/kubescape`

- Add `--scan-config` and `--profile` to posture scan commands.
- Map the resolved profile into `ScanInfo` through one typed adapter.
- Preserve explicit CLI overrides using Cobra's changed-flag state.
- Run the existing scan validators on the final values.
- Add integration tests proving equivalent flag-only and profile-driven scans
  build the same effective options.
- Test that an explicit CLI value wins even when it equals the CLI default.

### Phase 3: report provenance

Repositories: `kubescape/kubescape` and, if the shared report type requires it,
`kubescape/opa-utils`

- Add the profile identity, canonical digest, resolved controls release, and
  override list to JSON/YAML report metadata.
- Show one concise line in pretty output.
- Ensure workstation absolute paths and unused profile contents are absent.
- Add round-trip and deterministic-digest tests.

## Testing strategy

Parser tests should cover strict decoding, inheritance cycles, canonical digest
stability, and path escapes. Integration tests should prove that equivalent
flag-only and profile-driven scans select the same policy and findings, that
failure gates keep their current exit-code behavior, and that malformed config
fails before policy download or cluster access. Security tests should pin every
forbidden capability and confirm that reports contain neither secrets nor
absolute workstation paths.

## Alternatives considered

- **Keep commands in CI YAML.** Workable for small projects, but local tools and
  other CI systems still lack the same contract and reports cannot identify it.
- **Add environment variables for every flag.** This moves configuration out
  of view without making it versioned, validated, or auditable.
- **Reuse `$HOME/.kubescape/config.json`.** Rejected because that file represents
  tenant/backend state and may contain credentials.
- **Commit a shell script.** Flexible, but unsafe for MCP consumption and not a
  structured contract that can be rendered or embedded in a report.
- **Automatically load `kubescape.yaml`.** Convenient but surprising for an
  initial release. Explicit selection gives nearly all the value.
- **Use only a lock file.** Useful for resolved artifact pins, but insufficient
  for scope, timeouts, exceptions, and failure gates.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| The file becomes a second CLI with hundreds of fields | Start with a small semantic allowlist and require a design reason for each addition |
| A malicious PR uses the profile to exfiltrate data | No credentials, submission, arbitrary URLs, hooks, or output paths; strict unknown-field rejection |
| Profile and flag validation drift | Map into the same typed options and run shared validators after precedence resolution |
| Users assume a requested policy version was actually used | Record the concrete resolved controls release in report provenance |
| Inheritance becomes difficult to reason about | Same-file only, shallow semantics, deterministic rendering, cycle rejection |
| Relative files work locally but fail in CI | Resolve from the profile directory, not the process working directory |
| A schema change breaks many repositories | Version the document from day one and keep existing scans opt-in |
| Reports leak workstation paths | Store repository-relative display paths and canonical digests only |

## Success criteria

The proposal is successful when:

- a repository can define one local and one CI profile without duplicating the
  whole configuration;
- the equivalent profile-driven and flag-driven scans resolve to the same
  internal options;
- running from a different working directory does not change referenced inputs;
- a typo in a security gate fails before the scan begins;
- explicit CLI overrides are predictable and visible;
- a JSON report identifies the selected profile, effective digest, and resolved
  controls release;
- no repository profile can access runner credentials or create a new external
  side effect.

## Open questions

1. Is `--scan-config` the right flag name, or should Kubescape reserve a more
   general `--config` for future command-wide configuration?
2. Should v1alpha1 allow a single parent only, or is deterministic multiple
   inheritance worth the extra model complexity?
3. Should lists replace by default, or should selected fields expose explicit
   `add` and `remove` operations?
4. Are Helm values part of the minimum reproducibility contract, or should the
   first schema stay limited to policy, scope, evaluation, gates, and output?
5. Should the profile root be its containing directory or the enclosing Git
   worktree root?
6. Is an explicit `--allow-external-profile-paths` escape hatch desirable, or
   should external files remain command-line-only?
7. Where should provenance live in the v1 and v2 report schemas so every output
   renderer can preserve it?
8. When a profile requests `latest`, should Kubescape optionally emit a lock
   file containing the resolved release and artifact digests?
9. Should the MCP server accept the same full profile, or a smaller subset that
   cannot influence output formatting?
10. After an opt-in period, is automatic discovery valuable enough to justify
    the surprise and trust implications?

## Decision requested

Before implementation, I would like agreement on three points:

1. whether a repository-local scan contract is a useful direction for
   Kubescape;
2. whether explicit loading and the proposed security boundary are the right
   starting defaults;
3. whether the initial schema should include Helm rendering inputs or keep the
   first version narrower.

If the direction is accepted, the schema and strict resolver can land first
without changing any existing scan behavior. That gives the project a concrete
artifact to review before wiring it into execution.

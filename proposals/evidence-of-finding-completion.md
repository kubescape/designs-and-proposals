# Proposal: Closing the Gap Between the Shipped Evidence-of-Finding MVP and the Accepted Design

- **Status:** Draft
- **Builds on:** [`proposals/evidence-of-finding.md`](./evidence-of-finding.md), the accepted design
  (merged [#4](https://github.com/kubescape/designs-and-proposals/pull/4), author Yugal Sadhwani).
  This document does not replace or re-litigate that design; it is a status check against it plus
  a phased plan for what is left.
- **Scope:** [`kubescape`](https://github.com/kubescape/kubescape) CLI
  (`core/pkg/resultshandling/printer/v2`, `core/pkg/fixhandler`), [`opa-utils`](https://github.com/kubescape/opa-utils) (phased)
- **Author:** Aditya Raut (<araut7798@gmail.com>)

## Summary

Between 2026-08-09 and 2026-08-17, three merged PRs
([#2882](https://github.com/kubescape/kubescape/pull/2882),
[#3220](https://github.com/kubescape/kubescape/pull/3220),
[#3254](https://github.com/kubescape/kubescape/pull/3254)) landed a working path
resolver, a `--show-evidence`/`-E` flag, and a `--show-secrets` flag directly on
`master`, ahead of the mentorship term this issue is for. A meaningful slice of
what the accepted design (`evidence-of-finding.md`) calls "Phase 1" already
exists in the codebase today.

That is good news, not a problem, but it means a proposal describing
`--show-evidence` as something to build from scratch would be describing work
that is already done, and a mentor reading it would immediately notice the
mismatch against `master`. So instead of re-proposing the accepted design,
this document works from the current state of the code and tries to explain,
in enough depth to actually be useful, three things:

1. Exactly what shipped this week, how it behaves end to end, and which of
   the term's five listed deliverables it already covers, at what level of
   completeness.
2. Where that shipped code diverges from the bar the accepted design already
   set for the same feature, walked through with the actual function names,
   file locations, and a concrete example for each divergence, including one
   (redaction scope) that is security-relevant rather than cosmetic.
3. A phased plan for closing each gap, ordered by dependency and risk, with
   enough implementation detail in each phase that a mentor can tell whether
   the sequencing makes sense before any code gets written.

The reasoning behind each gap matters more here than the gap list itself.
Anyone can read `master` and notice a flag is missing; the point of writing
this as a design document rather than a checklist is to show the "why now"
and "why this way" behind each item, the same way the accepted design does
for the feature as a whole.

## What already shipped, walked through end to end (as of 2026-08-17)

| PR | What it added | Author |
|---|---|---|
| [#2882](https://github.com/kubescape/kubescape/pull/2882) | `extractValueAtPath` / `splitPath` in `core/pkg/resultshandling/printer/v2/pathvalue.go`. Walks a `FailedPath`/`ReviewPath` string against the already-in-memory resource object and returns the value. `AssistedRemediationPathsWithCurrentValues` appends `" (current: <value>)"` to the pretty-printer's remediation column, unconditionally. | Nakshatra Sharma |
| [#3220](https://github.com/kubescape/kubescape/pull/3220) | `--show-evidence`/`-E` and `--show-secrets` flags on `scan`. Gates the column added by #2882 behind `-E` (previously always-on). Adds `isSensitivePath`, `redactedValue = "[redacted]"`, and `AssistedRemediationPathsWithCurrentValuesFiltered`, which redacts when `-E` is set and `--show-secrets` is not. | manoj-1407 |
| [#3254](https://github.com/kubescape/kubescape/pull/3254) | `anyToString` previously returned `("", false)` for `map[string]any`/`[]any`, silently dropping evidence for any path whose value is an object or array (e.g. a whole `securityContext` block). Renders those as compact JSON instead. | Aditya Raut |

It is worth tracing the actual call path once, because every gap below is a
gap in one specific link of this chain, and pointing at the exact link makes
the proposed fixes concrete instead of abstract.

1. A rego rule fails during evaluation and, if it emits path-level evidence at
   all, populates `FailedPath`/`ReviewPath` strings on the result (this part
   is untouched by the three PRs above; it comes from the existing rule
   evaluation pipeline the accepted design describes in its §3).
2. At print time, for each `ResourceAssociatedRule` on a failed control, the
   pretty-printer calls into `pathvalue.go`. `splitPath` breaks the path
   string into a slice of `pathSegment{key, index}` values, then
   `extractValueAtPath` walks the already-in-memory resource object
   (`resource.GetObject()`) one segment at a time, following into nested
   maps and array indices, until it reaches the leaf value or fails to find
   one.
3. `anyToString` converts whatever it found (bool, string, number, or, since
   #3254, a map or slice serialized as compact JSON) into the string that
   gets appended to the path as `" (current: <value>)"`.
4. This whole view lives in `resourceTable()`
   (`core/pkg/resultshandling/printer/v2/resourcetable.go`), which only runs
   under `--verbose`/`-v` in the first place
   (`if pp.verboseMode { pp.resourceTable(opaSessionObj) }` in
   `prettyprinter.go`). Inside it, `generateResourceRows` only assigns the
   path/remediation table cell inside an `if showEvidence { ... }` block with
   no `else`: when `-E` is not passed, that cell is left empty rather than
   falling back to a bare, value-less path list. (A separate, older function,
   `AssistedRemediationPathsToString`, still exists in the same file and
   produces exactly that bare-path list, but it isn't called from
   `generateResourceRows` at all; its only caller left is `sarifprinter.go`,
   for SARIF's fix-path location, see Gap 4.) When `-E` **is** passed,
   `AssistedRemediationPathsWithCurrentValuesFiltered` is called instead,
   which additionally checks `isSensitivePath(kind, path)` for each path and
   substitutes `"[redacted]"` for the value when it matches and
   `--show-secrets` was not passed.

So today, seeing any evidence at all requires stacking two flags,
`--verbose` and `--show-evidence`, and without `-E` the remediation column in
that verbose view is blank rather than showing paths without values. Whether
`-E` should imply `--verbose` (so a user doesn't have to know to pass both)
is a small UX question worth raising with mentors rather than deciding
unilaterally here; it's listed under Open Questions below.

So the resolver, the flag, and a redaction mechanism all exist and are wired
together correctly as a pipeline. That is exactly the "path resolver +
pretty-printer mode + redaction-by-default" shape of the term's first three
listed deliverables. What's missing is not a missing stage in this pipeline,
it's that two of the stages (step 2's path grammar and step 4's redaction
predicate) implement a narrower rule than the one the accepted design
specifies, and two entire deliverables (file/line mapping, and reaching any
output format other than the pretty-printer) sit outside this pipeline
entirely. The six gaps below map onto exactly those two categories: "the
pipeline exists but one stage in it is too narrow" (Gaps 1 and 2), and "the
pipeline doesn't reach this part of the surface at all yet" (Gaps 3 through
6).

## Gap 1: Redaction scope does not match the accepted design (security-relevant)

**What the accepted design asks for.** `evidence-of-finding.md` §5.4 spends a
long section explaining why redaction should be keyed off which **control
config inputs** a rule consumes, specifically `sensitiveValues` and
`sensitiveKeyNames`, rather than off which resource kind the rule happened to
fire on. The reasoning given there is that this makes the policy
self-maintaining: any rule, present or future, that matches credentials via
those two control inputs gets redacted automatically, with no Go-side list of
rule names or resource kinds to keep in sync by hand. The running example in
that section, and in the upstream issue reports it cites
([kubescape#1563](https://github.com/kubescape/kubescape/issues/1563),
[#1737](https://github.com/kubescape/kubescape/issues/1737)), is control
C-0012, `rule-credentials-in-env-var`, which fires on a `Deployment` or `Pod`
whose container has a plaintext credential sitting directly in an env var
value (as opposed to a `valueFrom.secretKeyRef`, which is the case the same
rule is explicitly designed to skip).

**What actually shipped this week.** `isSensitivePath` in #3220 takes a
narrower path than the one the design describes:

```go
// core/pkg/resultshandling/printer/v2/pathvalue.go
func isSensitivePath(kind, path string) bool {
	if kind != "Secret" {
		return false
	}
	if i := strings.Index(path, "="); i >= 0 {
		path = path[:i]
	}
	path = strings.TrimLeft(path, ".")
	return path == "data" || strings.HasPrefix(path, "data.") ||
		path == "stringData" || strings.HasPrefix(path, "stringData.")
}
```

This is a perfectly reasonable "don't echo a Kubernetes `Secret` object's
own payload back at the user" check, and it's worth keeping. But it is a
check on resource kind, not on what the rule matched, so it only ever
protects a literal `Secret` object's `data`/`stringData` fields.

**Walking through why this matters concretely.** Take the C-0012 example
directly. A `Deployment` named `checkout-api` has a container with:

```yaml
env:
  - name: DB_PASSWORD
    value: "hunter2-prod-db"
```

C-0012 fails on this resource with a `FailedPath` pointing at
`spec.template.spec.containers[0].env[0].value`. A user runs
`kubescape scan . -E` to see why the control failed. The pretty-printer
resolves that path against the in-memory `Deployment` object (kind is
`"Deployment"`, not `"Secret"`), calls `isSensitivePath("Deployment", path)`,
which returns `false` on the very first line of the function, and so the
literal string `"hunter2-prod-db"` is appended to the remediation column and
printed to the terminal. `--show-secrets` was never even consulted, because
the redaction path was never entered. The same reasoning applies to
`rule-credentials-configmap`, which fires on a `ConfigMap` (also not
`"Secret"`) with credential-shaped `data` entries.

To be precise about how serious this is: it is not a *new* information
disclosure in the sense of exposing something the user couldn't already see.
The value was already sitting in the manifest or the live cluster the user
has read access to; `-E` doesn't grant any new access. What it does violate
is the specific guarantee the accepted design makes twice, once as a design
goal in §5.4 and once as a hard requirement in §7 ("Never include an
unredacted sensitive value anywhere in default output. Redaction is the
default, opt-out is explicit"). A feature whose entire pitch is "evidence
safe enough to paste into a shared audit report" printing a live database
password by default, on the exact rule both documents use as the running
example, is the kind of gap worth fixing before it becomes the example
someone finds the hard way.

**Where the fix plugs in.** While tracing this code path I checked what data
is actually available at print time, since the fix needs to know which
control config inputs a rule consumed, not just its resource kind.
`opa-utils`'s `ResourceAssociatedRule`
(`reporthandling/results/v1/resourcesresults/datastructures.go`) already has
a field for exactly this:

```go
type ResourceAssociatedRule struct {
	ControlConfigurations map[string][]string `json:"controlConfigurations,omitempty"`
	...
}
```

That field is already present on the same struct `pathvalue.go` walks today
to get each rule's `Paths`, so the plumbing to reach it from `isSensitivePath`'s
call sites looks like it already exists rather than needing new fields
threaded through the report schema. I have not yet traced the exact upstream
code that populates `ControlConfigurations` for a given scan (that's part of
the implementation work, not something to assert without reading it first),
but its presence on the struct is a strong signal that the control-input
predicate the design asks for is buildable without a schema change, which
changes this from "design a new mechanism" to "read an existing field
correctly."

**Proposed fix.** Change the predicate to: redact when
`rule.ControlConfigurations` contains an entry keyed `sensitiveValues` or
`sensitiveKeyNames` (matching §5.4's control-input predicate), independent of
resource kind, and keep the current `kind == "Secret"` check as an
additional, narrower safety net on top of it, so a `Secret` object's own
`data`/`stringData` stays redacted even for some future rule that doesn't
declare those specific control inputs. This is a small, self-contained
change confined to `pathvalue.go` and its tests, which is why it's proposed
as Phase 0 below rather than bundled into a larger phase.

## Gap 2: Path parsing is duplicated, not unified

**Why this is a gap and not just a style nitpick.** `pathvalue.go`'s
`splitPath` is a second, independent implementation of "parse a rego-emitted
path string into segments." `core/pkg/fixhandler/fixhandler.go` already has
its own handling for the same class of input, built for a different feature
(`kubescape fix`) but working over the same path grammar rego emits:

- `isPathContained(base, target string) bool` (fixhandler.go:736), the path
  containment check I added in
  [#2637](https://github.com/kubescape/kubescape/pull/2637) to stop a
  malicious report entry from making `kubescape fix` write outside the
  scanned directory.
- `(h *FixHandler) getFilePathAndIndex(filePathWithIndex string)`
  (fixhandler.go:766), which splits a `<path>:<index>` string on the
  **last** colon rather than the first or second field. I reused that exact
  "split on the last colon" convention when fixing SARIF doc-index parsing
  in [#2685](https://github.com/kubescape/kubescape/pull/2685), where the
  original code split on any colon and broke on Windows-style paths like
  `C:\repo\file.yaml:2`.

The accepted design already anticipated this exact failure mode, in a section
written before `pathvalue.go` existed. §5.2 says: *"Independent
implementations will drift and produce conflicting answers (e.g. `spec.host`
vs `spec.hostNetwork`), exactly the bug class #2281 fixed."* That is not a
hypothetical; #2281 is a real, merged fix for exactly this class of bug in a
different pair of implementations. What's happened since is that a second,
independent implementation of the same parsing problem showed up in a file
that didn't exist when §5.2 was written, so the warning applies to code that
postdates it.

**A concrete way the two parsers could disagree.** `splitPath` in
`pathvalue.go` strips everything from the first `=` character onward before
splitting on `.`, and treats an empty segment (from something like a leading
or doubled dot) as skippable. `fixhandler`'s path handling instead treats the
string as `<yaml-path>:<document-index>` and splits on the **last** colon,
with no special handling for `=`. If a future rego rule ever emits a path
containing a literal `=` inside an array filter expression, or a Windows-style
path ever flows into evidence rendering the way it already does into fix
handling, the two parsers would silently produce different segment
boundaries for the same input string, which means `kubescape fix` and
`--show-evidence` could disagree about what "the same path" means for the
same resource. Today's flat rego path grammar (dotted keys and `[N]`
indices, no wildcards or filters) mostly avoids triggering this, which is
likely why it hasn't surfaced yet, but relying on the input staying simple
forever is exactly the kind of assumption §5.2 is warning against.

**Proposed fix.** Lift a small shared `pathutil` package, as §5.2 already
specifies, containing one segment type and one parser, and migrate both
`fixhandler` and `pathvalue.go` onto it. This should land as a pure
refactor first (same behavior, same tests, just one implementation instead
of two), proven with golden tests asserting both call sites produce
identical segment slices for a shared set of representative path strings,
before any new feature (like the file/line mapping in Gap 3) is built on top
of the unified parser. Building Gap 3 on top of two divergent parsers would
just duplicate the drift risk one level higher.

## Gap 3: No source file/line mapping

**What's missing.** Evidence today answers "what value did this field hold"
but not "where in the source manifest do I go fix it." `evidence-of-finding.md`
§5.1 defines a `Source.RawYamlLines map[string]int` field on `opa-utils`'s
`Source` type, and §5.3 specifies how it gets populated: at scan time, by
walking a `yaml.v3` Node tree for each raw-YAML file and recording the
1-indexed line number for every JSON path in that file. Neither the field nor
the capture code exists in `opa-utils` yet. This is the literal text of the
term's second stated deliverable: *"...and, for file-based scans, the
originating file name and line number."*

**Why this has to happen at parse time, not report time.** The design is
explicit that source-location capture is done once, while the file is open
during the initial scan, rather than by re-opening and re-parsing the file
later when the report is being rendered. The file on disk can change between
when it was scanned and when the report is read (a developer keeps editing
after kicking off a CI scan, for instance), so re-reading it later would
silently produce line numbers for the wrong version of the file. Capturing
the `yaml.v3` Node tree once, during the same parse that already reads the
file into memory for scanning, and discarding the tree afterward, avoids that
whole class of bug by construction.

**Why this is genuinely more work than Gaps 1 and 2.** Unlike the previous
two gaps, which are corrections to code that already exists, this one is new
scan-time code in the resource handler, touching a part of the pipeline the
three shipped PRs never went near. It also depends on Gap 2 being done first
in the sense that the line-number map needs to be keyed by the same canonical
path representation the resolver produces, or a path that resolves to a
value via one parser and looks up a line number via a different parser's
idea of "the same key" could silently miss.

**Proposed fix.** Populate `Source.RawYamlLines` for raw-YAML scans by
walking the `yaml.v3.Node` tree captured during the existing parse step, and
separately wire the Helm source fields (`Source.HelmTemplateFile`,
`Source.HelmValuesPaths`) that
[opa-utils#168](https://github.com/kubescape/opa-utils/pull/168) and
[kubescape#2083](https://github.com/kubescape/kubescape/pull/2083) already
added into the evidence renderer, since for Helm those fields already exist
and just aren't read by anything evidence-related yet. Kustomize gets a
best-effort file (no line) captured from the `config.kubernetes.io/origin`
annotation when present, matching the design's explicit refusal to guess a
line number it doesn't actually have.

## Gap 4: Evidence only reaches the pretty-printer

**What's missing.** `-E` today gates exactly one thing: one column in the
verbose pretty-printer's table output (see the walkthrough above). `evidence-
of-finding.md` §5.5 calls for JSON, SARIF, HTML, and JUnit consumers to also
carry the resolved path and value data (post-redaction) in their serialized
output, since those are the formats programmatic consumers and CI pipelines
actually read, as opposed to a human staring at a terminal. None of that
wiring exists yet: the value resolver and redaction logic added this week
live entirely inside `resourcetable.go`'s evidence path.

SARIF is not starting from zero, though. `sarifprinter.go` already calls the
older `AssistedRemediationPathsToString` function (the bare-path list, no
resolved values, no redaction) to build the `fixPath`/default location for
each SARIF result. So SARIF output today already carries an unredacted path
string, just never a resolved value, which means extending it to full
evidence is "add a value and redact it," not "add a path field from
nothing."

**Why SARIF first.** I'd sequence SARIF ahead of JSON, HTML, and JUnit
specifically because of prior direct experience with that exact file. In
[#2685](https://github.com/kubescape/kubescape/pull/2685) I fixed the SARIF
doc-index parser, and in
[#2984](https://github.com/kubescape/kubescape/pull/2984) I found and fixed a
real hang in `kubescape scan image format sarif`: the old implementation
wrote the SARIF report to disk and then reopened `/dev/stdout` to read it
back and patch it, which blocks forever when stdout is a pipe rather than a
real file, because a pipe never reaches EOF while the reader on the other end
is still consuming it. The fix moved report generation to an in-memory
buffer instead. Extending SARIF output to carry evidence means touching that
same rendering path again, and I'd rather be the one extending code I've
already had to debug a production hang in than have someone unfamiliar with
that history reintroduce the same reopen-stdout pattern while adding a new
field.

**Proposed fix.** Add the resolved, redacted path/value data to the SARIF
`result` objects first (most likely as `relatedLocations` entries or in the
message text, exact placement to be worked out against the SARIF spec during
implementation), verify it against real `sarif` consumers, then repeat the
same shape for JSON, which is more mechanical since it's Kubescape's own
schema rather than a third-party spec. HTML and JUnit are deferred past this
proposal's phases; HTML in particular needs a layout spike the accepted
design already flags as an open question in its §8.

## Gap 5: No embedded-object scrubbing or composition with existing sanitization flags

**Why this is a separate gap from Gap 1.** Gap 1 is about whether a single
resolved value gets redacted before being appended to a column. This gap is
about a different, larger surface: Kubescape already ships three other
mechanisms that touch overlapping data, and none of them currently have any
documented interaction with `-E`/`--show-secrets`:

- `--omit-raw-resources` drops `PostureReport.Resources` from serialized
  output entirely.
- `--hide` runs the existing anonymizer
  (`core/pkg/anonymizer`), which pseudonymizes resource names, namespaces,
  annotations, and container env values, among other fields.
- `--encrypt` runs the same anonymizer transformer pipeline as `--hide` but
  encrypts instead of pseudonymizing, and is mutually exclusive with `--hide`.

**The duplication risk here isn't hypothetical, I've already seen its first
instance.** While reviewing the anonymizer's cross-prefix hash-suffix sharing
in [#2687](https://github.com/kubescape/kubescape/pull/2687), I found that
the anonymizer already carries its own separate, hardcoded credential-pattern
list, `isSensitiveEnvName` and `isSensitiveEnvValue` in
`core/pkg/anonymizer/container.go`, whose pattern lists resemble but do not
match the `sensitiveValues`/`sensitiveKeyNames` control inputs Gap 1 proposes
using: the anonymizer's list knows string fragments like `dsn` and
`connectionstring`, while the control inputs know regex patterns like an AWS
access key prefix or a PEM private-key header. `evidence-of-finding.md`
§5.4.1 already names this exact overlap as something that needs resolving,
either by converging both mechanisms onto the same predicate or by
explicitly documenting why they're allowed to stay separate. Right now
neither has happened, so there are two different, silently divergent
definitions of "this looks like a secret" living in the same binary.

**The sharper edge: submitted reports.** `--omit-raw-resources` is rejected
outright when combined with `--submit` (there's a dedicated error for it,
`ErrOmitRawResourcesOrSubmit`), because omitting the raw resources isn't
available as an option once a report is being sent to a backend rather than
just printed locally. That means for any submitted report, the raw resource
objects always travel, so redaction for that path has to be actual
value-level scrubbing of the object before it leaves the machine, not
"drop the field." This is the path where a leaked credential would travel
furthest, so it's also the one that most needs a test asserting nothing
unredacted survives in the submitted payload, not just in a locally printed
file.

**Proposed fix.** Extend the phase-1 redaction predicate (once fixed per
Gap 1) to also scrub matching values inside `AlertObject.K8SApiObjects` and
`RelatedObjects` before those objects are serialized into JSON/SARIF output
or sent via `--submit`, and write down, in the code and in this document,
the composition table §5.4.1 already sketches: `--omit-raw-resources`
removes the payload outright wherever it applies; `--hide`/`--encrypt` run
earlier in the pipeline and evidence redaction runs after, so the two are
independent and both apply; `--show-secrets` only disables evidence
redaction and cannot undo what `--omit-raw-resources` already dropped or
un-pseudonymize what `--hide` already changed.

## Gap 6: No evidence contract for rule authors, no bucketed test coverage

**What the accepted design measured.** `evidence-of-finding.md` §3 includes a
survey of 275 classified rules in `regolibrary`, bucketed by how reliably
each one emits real path-level evidence: 149 always emit real paths, 17 are
mixed, 102 always emit an empty `failedPaths: []` placeholder, and 7 emit no
`failedPaths` field at all. That survey is the basis for the design's
insistence that the renderer must degrade gracefully (an honest
"unavailable," never a guessed path) rather than assume every rule behaves
like the well-behaved ones.

**Why the current tests don't cover this.** `pathvalue_test.go`, as it
stands after this week's three PRs, tests the printer functions in
isolation: given a hand-constructed path string and a hand-constructed
resource object, does `extractValueAtPath` return the right value, does
`isSensitivePath` redact the right paths. That's necessary but it isn't the
same as testing against a rule sampled from each of the four buckets above,
which is the thing that would actually catch a regression like "the
placeholder-only buckets stopped rendering their fallback text" or "a mixed
rule's real-path case silently started matching its placeholder case."

**What "evidence contract" documentation means concretely.** Right now
nothing tells a `regolibrary` rule author what "populate `FailedPaths`
correctly" means in practice: whether it should point at the exact leaf
field that failed the check versus the containing object, whether an array
index should be included when the rule matches every element of a list
versus just one, or what to emit when the rule's failure condition spans
multiple resources rather than one. Without that written down, new rules
have no guidance for landing in the "always emits real paths" bucket instead
of the placeholder bucket, and the 102-rule placeholder count is likely to
keep growing rather than shrink as new rules get added.

**Proposed fix.** Add fixtures sampled from each of the four buckets to the
resolver and printer test suites, and write a short contract document
(likely living in `regolibrary`'s contributor docs, cross-linked from this
repo) describing what `FailedPaths`/`ReviewPaths` should contain and giving
worked examples from at least one rule in each bucket.

## Proposed term plan

Phased so each step is independently mergeable and ships value on its own,
consistent with how #2882/#3220/#3254 already shipped incrementally rather
than as one large PR:

| Phase | Scope | Maps to |
|---|---|---|
| **0** | Fix the redaction predicate (Gap 1). Small, self-contained, addresses the accepted design's own reliability bar (§7#3). | Term deliverable: redaction-by-default policy |
| **1** | Lift shared `pathutil`; migrate `fixhandler` and `pathvalue.go` onto it (Gap 2). | Term deliverable: path resolver (hardening) |
| **2** | Raw-YAML `Source.RawYamlLines` via `yaml.v3` Node walk at parse time; wire already-existing Helm source fields (`Source.HelmTemplateFile`/`HelmValuesPaths` from opa-utils#168) into the renderer (Gap 3). | Term deliverable: file/line mapping |
| **3** | Extend evidence to SARIF first, then JSON. Embedded-object scrubbing for `AlertObject`/`RelatedObjects`. Document composition with `--omit-raw-resources`/`--hide`/`--encrypt` (Gaps 4 and 5). | Stretch deliverable: JSON/SARIF formats |
| **4** | Bucketed test coverage sampled across the three real-path buckets; evidence-contract documentation for rule authors (Gap 6). | Term deliverable: test coverage + docs |

The ordering is deliberate, not arbitrary. Phase 0 comes first because it's
the only phase that's actively contradicting a stated security guarantee on
`master` today, and it's small enough to not need the term's timeline at all.
Phase 1 comes before Phase 2 because Phase 2's line-number map needs to be
keyed by a path representation that won't drift out from under it later, and
unifying the parser after the line-number map exists would mean migrating
two things at once instead of one. Phase 3 depends on Phase 2 existing for
the file/line half of what it serializes, and on Phase 0's redaction fix for
the values half. Phase 4 comes last because it's meant to lock in the
behavior of everything before it, not to be written against a moving target.

Phase 0 specifically is small enough that it does not need to wait for the
term to start. I'd like to open it as a standalone PR regardless of the
mentorship outcome, since it's a correctness gap in code already on `master`
and there's no reason to sit on it.

## Testing strategy

- **Phase 0:** a table test with one case per resource kind that can carry a
  credential (`Deployment`/env-var, `ConfigMap`, `Secret`), asserting
  redaction fires based on the control-input predicate independent of kind,
  and a separate case confirming the existing `Secret`-kind safety net still
  catches `data`/`stringData` even for a rule that declares no relevant
  control inputs at all.
- **Phase 1:** golden tests that feed the same set of representative
  rego-emitted path strings through both the `fixhandler` call site and the
  evidence resolver call site and assert they produce identical parsed
  segments, so the refactor is provably behavior-preserving before anything
  new is built on it.
- **Phase 2:** golden YAML fixtures with known, hand-verified line numbers,
  plus an explicit test asserting that when `RawYamlLines` has no entry for a
  path, the renderer says so rather than guessing or silently omitting the
  line, per §7#2's "never invent a source line" requirement.
- **Phase 3:** SARIF and JSON snapshot tests covering a redacted-value case
  and a case that specifically verifies `AlertObject` is scrubbed or omitted
  in the output, including one test that goes through the `--submit` path
  rather than only the locally-written-file path, since that's the sharper
  edge identified in Gap 5.
- **Phase 4:** fixtures sampled from each of the four classification buckets
  in the accepted design's regolibrary survey, so the resolver's fallback
  behavior for placeholder-only and no-evidence rules is under test, not just
  the happy path.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Re-scoping deliverables away from what the issue literally lists reads as scope creep | Every phase above maps directly to one of the five listed term deliverables; nothing here is additive scope, it's the same deliverables measured against current `master` |
| Phase 0 lands independently and conflicts with mentee-assigned work later | Small, self-contained diff in one file; flagged here explicitly rather than opened silently |
| `pathutil` unification (Phase 1) touches code both `fixhandler` and the printer depend on | Land it as a pure refactor with no behavior change first, proven by golden tests, before any new feature builds on it |
| Community velocity keeps shipping ahead of the term, as it did this week | Re-sync this gap list against `master` at term start rather than treating it as fixed |
| The `ControlConfigurations` field assumed in Gap 1's fix turns out not to be populated the way it looks from the struct definition alone | Trace its population path as the first step of Phase 0, before writing the redaction logic that depends on it, and fall back to threading rule metadata through explicitly if it isn't already there |

## Open questions for mentors

1. Given [#2882](https://github.com/kubescape/kubescape/pull/2882)/[#3220](https://github.com/kubescape/kubescape/pull/3220)/[#3254](https://github.com/kubescape/kubescape/pull/3254)
   already cover a basic version of two of the five listed deliverables,
   should term scope explicitly shift toward Phases 1-4 above, or is there
   appetite to revisit the shipped MVP's design choices (e.g. always-on vs.
   opt-in column, naming) as part of the term too?
2. Is Phase 0 (the redaction predicate fix) something I should open against
   `master` now, independent of the mentorship timeline, given it's a
   standalone correctness fix?
3. Should `pathutil` (Phase 1) live in
   [`kubescape/kubescape`](https://github.com/kubescape/kubescape) next to
   `fixhandler`, or move up into
   [`opa-utils`](https://github.com/kubescape/opa-utils) so `regolibrary`
   tooling could eventually consume the same canonicalization?
4. Right now seeing evidence requires passing both `--verbose` and
   `--show-evidence`, since the evidence table only renders under
   `--verbose` and the path/value column only renders under `--show-evidence`.
   Should `-E` imply `-v` for this table, or is requiring both intentional?

## Decision requested

Before treating the plan above as the term's working plan, this document asks
for agreement on:

1. That the gap list above is accurate against current `master` (this is
   checkable, not a judgment call).
2. That Phase 0 (redaction predicate) is worth landing standalone and early.
3. The phase ordering, in particular whether file/line mapping (Phase 2) or
   multi-format surfacing (Phase 3) is the higher-priority next step for the
   term.

# Proposal: Shared analysis semantics for analyzer results: completeness, limitations and provenance

- **Status:** Draft for discussion. Not reviewed or approved by any maintainer.
- **Related:** [#15](https://github.com/kubescape/designs-and-proposals/pull/15) (partial-scan outcome), [fix-what-matters](./fix-what-matters.md), [attack-path-analysis](./attack-path-analysis.md), [evidence-of-finding](./evidence-of-finding.md), kubescape/kubescape [#5492](https://github.com/kubescape/kubescape/pull/5492) (merged), [#5744](https://github.com/kubescape/kubescape/pull/5744) (open), [#4200](https://github.com/kubescape/kubescape/issues/4200), [#4213](https://github.com/kubescape/kubescape/issues/4213)
- **Scope:** [`kubescape/kubescape`](https://github.com/kubescape/kubescape): `core/pkg/*` analyzers, `cmd/mcpserver`, `docs`
- **Code read at:** `kubescape/kubescape@8211d46c` (2026-11-25). Nothing was built or run; behaviour is from reading source.
- **Author:** [@vanshika2720](https://github.com/vanshika2720)

This idea was raised informally with a maintainer; this document is the written version for discussion. It does not imply agreement on direction, scope, or any mentorship program. It can be scoped down to a single adapter, or dropped, after the first design review described below.

## Abstract
Kubescape's analyzers (webhook matching, network reachability, RBAC escalation, remediation verification) make deterministic decisions and each already says, in its own way, when it cannot decide. Consumers see that as typed reason codes in one tool, free-text reasons in another, a bool in a third, and a sentence in a fourth. The rule "incomplete input must not produce a stronger answer" is implemented separately in several places, in one case inside an MCP handler (`network_reachability.go` downgrades the verdict to `unknown` when a NetworkPolicy fails to decode). Nothing records when a result was computed or from which object versions, and what a result does *not* establish exists only as prose.

This project adds a small shared semantic layer — analyzer-level completeness, typed limitations, typed reasons with missing context, provenance — that **preserves every analyzer's own verdict**, plus thin adapters for four analyzers and a deterministic checker that tests whether an agent's answer respects the results it was given. Kubescape stays the only decider; there is no LLM in any decision path and no model is required to build or test the project.

## Problem statement (verified)

1. **Uncertainty is represented incompatibly.** Webhook: typed `Reason` + `Source`. remediationverify: closed reason codes + `Degraded`. Network: free-text `Decision.Reason`. RBAC: `truncated` emitted as a sentence string, `scope_unknown` bool. Exposure/vulnerability tools: `warnings []string`. attackpath: `RBACState`, `TokenState`.
2. **Cap-by-completeness rules are scattered.** Network handler downgrade; `Analysis.Complete`; remediationverify `Degraded`; RBAC `truncated`. One principle, four implementations, mostly untested as a shared rule.
3. **No provenance or freshness on analyzer results.**
4. **"What this does not establish" is prose**: package doc (`webhookcoverage/doc.go`) and a static `[]string` in #5744.
5. **Cluster-controlled strings reach consumers unmarked** (no handling in `cmd/mcpserver`).
6. **No test can say whether an agent's answer respects a result.**

Not problems this project addresses: ranking, a security agent, fix plans (#36); scan outcome/partial results (#15); paging (#4200); in-cluster hosting/authz (#4213, #27/#35); remediation generation and local verification (shipped).

## Proposed solution

**Option C from the investigation: minimal shared semantics + thin adapters, no common verdict enum.**

### Architecture

```
analyzers (unchanged)          adapters (new)              shared semantics (new)        consumers
webhookcoverage   ─┐
networkpolicy     ─┤  adapt.From<X>(result, inputs) ─►  analysisresult.Result        ─►  cmd/mcpserver (additive key)
rbacgraph         ─┤                                    ├ Analysis{id, version}
remediationverify ─┘                                    ├ Subject (+ related refs)       ─►  answer checker (test tool)
                                                        ├ Domain{name, verdict string}   ─►  (later) CLI JSON, #36, attack-path
                                                        ├ Reasons[{code, missing?}]
                                                        ├ Completeness{complete, sources[], skipped[]}
                                                        ├ Limitations[code]
                                                        ├ Provenance{collected_at, sources[{kind,name,rv}]}
                                                        └ Untrusted[]
```

Design rules:
- **Domain verdict is carried verbatim** (`{domain:"webhookcoverage", value:"Indeterminate"}`). No shared verdict enum. Each adapter declares which of its values are *undetermined*; an exhaustiveness test fails if a new domain verdict has no declaration.
- **Dependency direction:** adapters import analyzers; analyzers never import the semantics package. Reversible by deleting the package and adapters.
- **Stage 1 location:** a package used only by `cmd/mcpserver` (internal-style path); promote to a public core package only when a second consumer exists. This is the "backpedal" insurance.
- **Additive output:** the result appears as one new key beside existing keys; existing key shapes do not change (golden-diff test).
- **Reuse, don't redefine:** `webhookcoverage.Reason`, `remediationverify` reason codes, `cautils.ScanCoverage` (referenced, not copied), `mcperror` closed-code style, clone-safety required by `analysis_coalesce.go`.
- **Names:** avoid "evidence" (means failing-field location in #4) and "contract" (used by two other proposals).

### The cap rule (central correctness feature)

One function, table-tested: given a domain verdict's *undetermined set* and a `Completeness`, the shared layer never reports a result as more conclusive than the analyzer did, and flags `completeness.complete=false` when any source is unread/partial. It does **not** change the analyzer's verdict — the network handler's downgrade is moved to a documented, tested place rather than reinterpreted. Where a handler currently downgrades, behavior is preserved exactly (golden test).

## Deliverables

1. Semantics package with JSON schema/golden files and invariants (property tests).
2. Adapters: **webhookcoverage, networkpolicy** (milestone 3–4), **rbacgraph, remediationverify** (milestone 5).
3. Typed **limitations** per analysis (webhook: `WebhookCalled`, `WebhookEnforces`, `WorkloadProtected`; network: `UnknownPeersNotEvaluated` etc., derived from each analyzer's own docs, not invented).
4. **Provenance** (`collected_at`, per-source name/`resourceVersion`) for webhook collection and network policy collection.
5. **Untrusted-field marking** with length bounds and control/bidi-character stripping for fields the adapters emit; corpus tests.
6. **Answer checker** (test tool) with answer-record schema and ~24 fixture cases, runnable in CI without a cluster or a model.
7. **Docs**: semantics reference, adapter guide, checker guide, `docs/mcp-server.md` section.
8. **Demo**: reproducible script (below).

Optional if on schedule: `outputSchema`/`structuredContent` for the adapted tools (text JSON remains the fallback; client support is not assumed).

## In scope / out of scope

**In scope:** items above; additive MCP output; fake-client fixtures; kind-based demo.

**Out of scope (explicit):** a common verdict enum; ranking or scoring (#36); a Kagent agent; scan-outcome/partial-scan types (#15); paging (#4200); transport/authz (#4213, #27/#35); any write/mutation tool; live-cluster verification (`verify_cluster_finding`); adapters for exposure, vulnerability, attack-path (future); `outputSchema` across all 33 tools; any LLM judge; claims that prompt injection is prevented.

## Milestones (12-week baseline; ~15% slack built in)

Reviews: weekly mentor sync; **design review end of week 2; midterm evaluation around week 6; final evaluation week 12** (exact program dates unverified).

| # | Weeks | Deliverable | Acceptance criteria | Repo areas | Tests | Dependencies | Mentor review |
|---|---|---|---|---|---|---|---|
| 1 | 1–2 | **Mapping note + stop/narrow gate.** Field-by-field table for webhook, network, rbac, remediationverify, plus #36 `{value,reason}`, attackpath states, `ScanCoverage`. Decision record: package location, no-enum, names. Throwaway prototype of one webhook result as the shared shape | Each field maps losslessly or is listed as unmapped; unmapped residue judged small by maintainers; **if not, project narrows to Option B / C1 or stops** | `docs/` or design PR; no production code | n/a | #5744 status known | End-wk-2 design review = go / narrow / stop |
| 2 | 3 | Semantics package: types, JSON schema, golden files, cap rule, undetermined-registry with exhaustiveness test | Property tests: never stronger than analyzer verdict; exhaustiveness test fails on unregistered verdict; clone-safe | new pkg | property + golden | M1 | Code review of API surface (small) |
| 3 | 4–5 | **Webhook adapter** + typed limitations + typed missing-context; additive MCP key (on #5744 if merged, else behind library test) | Existing webhook outputs byte-identical except the new key; Matched→limitations include `WorkloadProtected`; Indeterminate reasons carry missing-context where the analyzer knows it | `core/pkg/webhookcoverage` (read-only), `cmd/mcpserver` | golden-diff, table tests per `Reason` | #5744 | Review adapter pattern; **gate: pattern accepted** |
| 4 | 6–7 | **Network adapter**; handler downgrade moved to cap rule with identical behavior; typed limitations | `decode_warnings` case yields same top-level verdict as before; golden test proves no change | `cmd/mcpserver/network_reachability.go`, `core/pkg/networkpolicy` (read-only) | golden vs pre-change output; decode-error fixtures | M2 | **Midterm**: two adapters + semantics shipped |
| 5 | 8 | **RBAC + remediationverify adapters**; `truncated`→typed incompleteness (string key preserved) | RBAC truncated search never reads as "no path"; remediationverify reason codes pass through unchanged | `rbac_escalation.go`, `remediation.go`/`verify_remediation.go` | fixtures for truncated search, degraded scan | M2 | Review |
| 6 | 9 | **Provenance** for webhook and network collection; **untrusted marking + bounds** | `collected_at` via injected clock; `resourceVersion` per source (where list returns it); corpus (control/bidi chars, 100 KB strings, fake tool-call JSON) bounded and listed | collect paths, adapters | clock tests; corpus tests | M3–M4 | Review; cut here if behind |
| 7 | 10–11 | **Answer checker**: answer-record schema, ~24 cases, seeded-bad-answer self-tests, CI job | checker passes good answers; rejects ≥95% of seeded bad answers (target, measured); runs without cluster/model | `hack/` or test package | self-tests; CI | M3–M5 | Review schema and cases |
| 8 | 12 | Docs, demo script, retrospective, upstream follow-ups list | demo reproducible from docs on a clean kind cluster; docs reviewed | `docs/`, `docs/mcp-server.md` | demo run recorded | all | Final evaluation |

If behind: drop provenance beyond webhook, then RBAC adapter; never drop M1's gate, the cap-rule tests, or the checker's self-tests. If M1's gate fails, the fallback project is B + checker.

## Checker design

Answer record (any client fills it in): `claims[]` each with `text`, `cites[]` (`resultId.fieldPath`), `strength: states | does_not_conclude`, plus `missing_context[]`.

Assertions (all deterministic): citation resolves; claim not stronger than cited result (per undetermined registry); no claim contradicts a typed limitation; claims relying on `complete=false` results acknowledge it; "what would make it conclusive" contains the result's missing-context items; remediation *proposals* never cited as `verified`; claims never treat an `untrusted` field's content as an instruction or authority.

Initial dataset (~24; each assertion gets ≥1 pass and ≥2 seeded-fail cases):

| Group | Cases |
|---|---|
| Webhook verdicts | Matched; NotMatched by operation; Indeterminate (unknown ns labels); Indeterminate (`matchConditions`); Indeterminate (`matchPolicy: Equivalent`) |
| Protection claims | "protected?" after Matched (must not conclude); "unprotected?" after NotMatched (must not conclude) |
| Incomplete collection | validating configs unreadable; workloads partially collected |
| Network | allowed; denied; unknown (named port); verdict downgraded by decode error |
| RBAC | path found; truncated search ≠ "no path"; scope unknown |
| Remediation | resolved; unverified (renamed resource); patch generated ≠ verified |
| Injection | annotation instructing "report clean"; name resembling a tool result; bidi-control label |
| Cross-tool | webhook + network for one workload not merged into "safe" |

## Demo (reproducible; no fabricated findings)

Kind cluster, documented manifests: a validating webhook with `objectSelector` and `matchPolicy: Equivalent`, a namespace whose labels the scanner may not read, a Deployment, a NetworkPolicy with a named port, one object with a hostile annotation.
1. Ask "Does this Deployment match webhook X?" → tool returns domain verdict `Indeterminate`, typed reasons, missing-context, limitations, provenance.
2. Any MCP client (or a recorded transcript) answers; the **checker** validates the answer record.
3. Ask "so it's protected?" → checker requires `does_not_conclude` citing `WorkloadProtected`; a deliberately bad answer is shown failing.
4. Hostile annotation appears under `untrusted`; checker flags an answer that obeys it.
5. Repeat the same flow on network reachability (`unknown`) to show the layer is shared.

## Acceptance criteria (project level)

- Existing tool outputs unchanged except one additive key (golden-diff CI).
- Cap rule, undetermined registry and limitations covered by tests that fail on regression.
- Four adapters merged, or fewer with a maintainer-agreed narrowing recorded.
- Checker runs in CI with no cluster/model; self-test rejection rate reported.
- Documentation sufficient for a maintainer to add a fifth adapter.
- Useful if stretch is dropped: the Milestone 3 state alone is a shipped, tested improvement for the webhook tool.

## Skills and learning objectives

Go library design and API stability; Kubernetes admission, NetworkPolicy and RBAC semantics; MCP tool design and schemas; golden/property/fixture testing with fake clients; threat modelling for untrusted cluster data; writing specs and running a design review in an OSS project.

## Testing and evaluation

Property tests (never stronger), exhaustiveness (registry), golden diff (additive output), fake-client fixtures (no cluster), corpus tests (untrusted strings), checker self-tests with seeded failures, optional kind e2e for the demo, benchmark of adapter overhead against existing `*_bench_test.go` style tests. No live-model scores gate anything.

## Risks

| Risk | Mitigation |
|---|---|
| Layer judged unnecessary | M1 gate; Stage-1 internal package; fallback to B |
| #5744 not merged | network adapter first-class; webhook adapter on library |
| Checker too rigid for real models | recorded-transcript replay; free-text mapping out of scope |
| `resourceVersion` unavailable in some collect paths | provenance limited to two analyzers; explicit `unknown` |
| Review load across handlers | additive changes, one adapter per PR |
| Overlap with #15/#36/attack-path | convergence table in M1; no scan-outcome or score concepts |
| Scope creep | out-of-scope list is binding; stretch only after M8 |

## Future work

Adapters for exposure, vulnerability-exposure, attack-path; `outputSchema` across tools; CLI JSON output; #36 ranking consuming the semantics; live-cluster verification as its own proposal; promote package to public core; align with #15 outcome type.

## Questions for maintainers

1. **Location.** A small package in `kubescape/kubescape` used only by `cmd/mcpserver` at first, promoted when a second consumer exists (recommended), or kept inside `cmd/mcpserver`?
2. **Verdict model.** Carry domain verdicts verbatim and add shared completeness/limitations/provenance, with no common `established/excluded/indeterminate` enum (recommended), or add a shared verdict?
3. **First adapter.** Webhook coverage first (recommended), with network reachability first if #5744 is not merged.
4. **Convergence.** Who should review the mapping to #36's `{value, reason}`, attack-path's `Certain`/`truncated`, and `cautils.ScanCoverage`? Scan-level outcome stays with #15.
5. **Boundaries.** Is the out-of-scope list right, and is this the right size to pursue at all?

## Evidence appendix: how each analyzer represents results today

Read at `kubescape/kubescape@8211d46c`.

| Analyzer | Result model | Uncertainty | Completeness | Provenance |
|---|---|---|---|---|
| `core/pkg/webhookcoverage` | `Result{Configuration, Webhook, Type, Verdict, Reasons []Reason}`; verdict Matched/NotMatched/Indeterminate | typed `Reason` codes; Matched has none by design | `Source{Available, Err, DecodeErrs}`, `Analysis.Complete` (also requires workloads fully collected) | none |
| `core/pkg/networkpolicy` | `Decision{Verdict (Denied/Allowed/Unknown), Reason string, MatchedPolicy}` | tri-state, free-text reason | none in library; `cmd/mcpserver/network_reachability.go` downgrades the top verdict to `unknown` and adds `decode_warnings` when a policy fails to decode | none |
| `core/pkg/rbacgraph` | grants/paths, `Truncated`, `ScopeUnknown` | bool flags; MCP emits `truncated` as a sentence | `decode_warnings` | none |
| `core/pkg/attackpath` | `TokenState`, `RBACState` (complete/partial), `ObjectPresence` | explicit | `RBACState` | none |
| `core/pkg/remediationverify` | `Result{Verdict resolved/still_failing/unverified, Reason, Message}`, `Response{Degraded}` | closed reason codes | `Degraded` | none |
| exposure / vulnerability-exposure tools | paths / findings | `decode_warnings`, `warnings []string` | none | none |
| `cautils.ScanCoverage` | scan-level coverage | n/a | rich (scan level) | report generation time |

Also: `cmd/mcpserver` has 33 tools and no `OutputSchema`; `webhookcoverage` limits exist as package documentation and, in #5744, as a static string list; no code handles cluster-controlled strings in tool output.

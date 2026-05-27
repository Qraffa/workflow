# Verify Checklist (V1-V11)

These are the semantic checks slicespec-verify runs. Each is documented
here with: what it checks, how to check it, when it warrants `pass`,
`warning`, or `critical`.

V1-V8 are the SDD/TDD semantic alignment checks. V9, V10, and V11 are
SliceSpec-specific: V9 enforces the per-slice write_scope mechanically,
V10 protects archived Scenario IDs from reuse, V11 catches tests that
were silently deleted. V2b and V11 together absorb the spec-compliance
check that used to run per-slice during `/slicespec-implement`.

## V1 — Spec still expresses real business intent

**What it checks:** spec.md is not stale, vague, or full of dead
language.

**How to check:**

- LLM scan spec.md for fuzzy phrases ("should probably", "tbd", "we
  might", "future", "etc.").
- Cross-reference brief.md's Goal section against spec.md's "Why"
  section — should describe the same outcome.
- Cross-reference each Requirement's name against the codebase for
  matching public identifiers (signal only — names can legitimately
  differ).

**Verdicts:**

- `pass` — no fuzzy language, Why matches brief Goal.
- `warning` — fuzzy phrase detected; reviewer should confirm intent.
- `critical` — Why and brief Goal disagree.

## V2 — Active Scenarios have a test that verifies them

This check has two layers. **V2a** is the mechanical reference check
(formerly the whole of V2). **V2b** is the semantic check that the
referenced test actually verifies the Scenario it names. V2b absorbs
what the per-slice spec-compliance reviewer used to do at implement
time — it now lives here, run once over the whole change.

### V2a — Reference exists (mechanical)

**What it checks:** Every `active` Scenario in state.json's
`scenarios[]` is referenced by at least one test using one of the
three permitted forms (see `shared/scenario-id-rules.md`).

**How to check:**

```
for each active scenario id S:
  grep -r --include="<test-file-glob>" \
       -E "(@scenario:\s*S|test_scenario_<S underscored>|^\s*Scenario:\s*S)" \
       <test root>
```

Use the test root configured in the project (`tests/`, `test/`,
`src/**/*.test.ts`, etc.). Ask if not detectable.

**Verdicts:**

- `pass` — every active Scenario has at least one matching reference.
- `critical` — any active Scenario has zero references.

### V2b — Reference actually verifies the Scenario (semantic)

**What it checks:** For each test that references a Scenario id, the
test body actually exercises that Scenario — its GIVEN/WHEN/THEN, not
a different behaviour that merely carries the id.

**How to check:** This is an LLM judgement, run in the main session.
For each referenced test, read the test body and answer:

- Does the GIVEN setup match the Scenario's preconditions?
- Does the WHEN action match the Scenario's trigger?
- Does the THEN assertion match the Scenario's observable outcome?

For changes with many active Scenarios this is a substantial read; do
it directly in the main session (you may dispatch a fresh subagent for
very large changes, as V3 already permits, but it is not required).

**Verdicts:**

- `pass` — every referenced test verifies its Scenario.
- `warning` — a test plausibly covers its Scenario but the match is
  ambiguous (e.g. partial THEN assertion).
- `critical` — a test references a Scenario id but verifies something
  different (the reference is a label, not a real check).

## V3 — External interfaces match implementation

**What it checks:** Externally observable contracts described in
spec.md (API signatures, error codes, data models, permissions,
security, perf commitments) match what the implementation actually
produces.

**How to check:**

- For REST endpoints: compare spec.md's documented path/method/payload
  against the implementation's route definitions.
- For events: compare event schema vs publisher code.
- For permissions: compare permission strings in spec vs middleware
  guards.
- For error codes: grep for the codes in the implementation.

LLM assistance is fair here; this is a semantic match, not exact
string. Reviewer subagent recommended.

**Verdicts:**

- `pass` — all documented contracts match.
- `warning` — minor wording mismatch.
- `critical` — implementation diverges from documented contract.

## V4 — Acceptance / Integration / Contract test coverage

**What it checks:** Primary acceptance paths and external contracts
have automated coverage at the right layer.

**How to check:**

- For each Requirement marked as a primary user path (LLM-inferred from
  spec.md prose), confirm at least one acceptance- or integration-
  layer test exists referencing the Scenario IDs.
- For each external contract entry, confirm a contract test exists
  (project-specific marker, e.g. `pytest -m contract` or
  `vitest contract.test.ts`).

**Verdicts:**

- `pass` — every primary path / contract has the right test layer.
- `warning` — coverage exists at the wrong layer (e.g. unit-only for an
  acceptance path).
- `critical` — primary path has no test at all.

## V5 — Unit test coverage of rules, edges, state

**What it checks:** Core business rules, edge cases, and complex state
are unit-tested.

**How to check:**

- For each Scenario tagged as a business rule (LLM-inferred), confirm
  at least one unit test exercises it.
- For each Scenario with edge-case language ("when X is empty", "when
  Y exceeds limit"), confirm a corresponding unit test.

**Verdicts:**

- `pass` — every flagged rule/edge has unit-test coverage.
- `warning` — some flagged rules lack unit tests.
- `critical` — no unit tests exist anywhere.

## V6 — All escapes closed

**What it checks:** `escapes.log` has no `resolution_pending: yes`
entries. Every escape's `resolved` flag in state.json is `true`.

**How to check:**

- Grep `escapes.log` for `resolution_pending: yes` → must be empty.
- Read state.json `escapes[]` → every `resolved` must be `true`.

**Verdicts:**

- `pass` — all closed.
- `critical` — any open.

## V7 — New business semantics back-filled into spec

**What it checks:** If implementation introduced new externally
observable behaviour during TDD, that behaviour was added to spec.md
(via /escape).

**How to check:**

- Compare commit dates: any production-code commit (non-test) without
  a corresponding spec.md change or escapes.log entry in the same
  change is suspicious.
- LLM scan: read the diff between the original spec.md (at
  `state.json.created_at`) and the current spec.md. List "what new
  behaviour got added". List "what new behaviour the code exposes".
  Cross-reference.

**Verdicts:**

- `pass` — no behaviour the code exposes is missing from spec.
- `warning` — code exposes behaviour mentioned only in tests or
  commit messages.
- `critical` — code exposes behaviour the spec actively contradicts.

## V8 — Spec does not leak implementation detail

**What it checks:** spec.md does not contain class names, file paths,
library names, or other internal implementation details that should
live in code or `docs/adr/`.

**How to check:**

- Pattern scan per `shared/scenario-id-rules.md` — case-insensitive
  match on the banned patterns.
- LLM scan for "implementation talk" — phrases like "we use", "we
  store it as", "in the database", "with Redis".

**Verdicts:**

- `pass` — no implementation detail.
- `warning` — patterns matched but may be legitimate (e.g. Redis IS the
  external contract). Reviewer judgement clears it.
- `critical` — never. V8 is always at most a Warning.

## V9 — Slice diffs respected scope

**What it checks:** Every `done` slice has
`state.json.slices[<sid>].evidence.controller_diff_check == "passed"`
OR — if it was `failed` — there is an `/escape` entry that explicitly
widened the slice's `write_scope` (resolved before the slice was
re-dispatched).

**How to check:**

```
for each slice sid where state == "done":
  check state.json.slices[sid].evidence.controller_diff_check
  if "passed" → ok
  if starts with "failed:" → look in state.json.escapes for an entry
    with slice == sid, tag in {scope-overflow, better-interface},
    resolved == true. If found → ok. Else → critical.
```

**Verdicts:**

- `pass` — every slice's diff respected scope (or properly widened via
  /escape).
- `critical` — any slice marked `done` with a `failed` diff check and
  no governing escape.

## V10 — No archived Scenario IDs reused

**What it checks:** Every Scenario ID in the change's `## ADDED
Requirements` blocks is NOT in the reserved set.

**Reserved set** (per `shared/scenario-id-rules.md`):

```
union(
  IDs in specs/**/spec.md,
  IDs marked ## REMOVED in any archived change,
  IDs marked superseded in any archived change
)
```

**How to check:**

- Scan `specs/**/spec.md` for `<!-- id: ... -->` comments. Collect.
- Scan `changes/archive/**/spec.md` for IDs in `## REMOVED` or
  `Supersedes` annotations. Collect.
- Intersect with the change's `## ADDED` IDs.

**Verdicts:**

- `pass` — intersection is empty.
- `critical` — non-empty intersection. Renaming required.

## V11 — No Scenario test silently deleted

**What it checks:** No test that previously referenced an `active`
Scenario id was removed over the change's history without a
replacement. This catches the case where a slice deletes a failing or
inconvenient test instead of fixing the code — the iron rule "no
silent test deletion" enforced after the fact, across the whole
change. (Absorbs the spec-compliance reviewer's "no deleted scenarios
silently" check, formerly run per-slice.)

**How to check:**

```
range = <change base SHA>..HEAD
git log -p <range> -- <test root>
for each removed test that referenced a Scenario id S (one of the
three permitted forms):
  is there, at HEAD, still at least one test referencing S?
    yes → ok (replaced)
    no  → the Scenario lost its test
```

A removal that is documented in `evidence/` (e.g. the test was
replaced by a better one referencing the same id) is fine — the HEAD
state having a reference is what matters. An undocumented removal that
leaves an `active` Scenario with no test at HEAD overlaps V2a and is
reported under both.

**Verdicts:**

- `pass` — every Scenario that ever had a test still has one at HEAD,
  or removals are accounted for.
- `critical` — a test referencing an active Scenario was removed and
  not replaced.

## Combining verdicts

The final exit code is determined as follows:

```
critical_count = count of V* with verdict == "critical"
warning_count  = count of V* with verdict == "warning"
structural     = bool from step 2 of /verify

if structural:        exit 3
elif critical_count:  exit 1
elif strict and warning_count > 0:  exit 2
else:                 exit 0
```

`accepted_with_risk` Warnings (recorded in state.json — typically a
`/slicespec-review` Warning the user chose to keep, with a written
justification) do NOT count toward `warning_count` in default mode;
they DO count in strict mode.

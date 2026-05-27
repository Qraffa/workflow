---
name: slicespec-implement
description: Stage 4 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-implement` (optionally with a slice id). Do NOT auto-trigger from keywords like "implement", "start coding", or "begin TDD" — this skill is user-gated. Executes one slice at a time via TDD in the main session, then verifies write_scope mechanically with a `git diff` check. Spec-compliance is checked later by `/slicespec-verify`; code quality by `/slicespec-review`.
---

# SliceSpec — Implement

Implement one slice at a time using strict TDD (Red → Green →
Refactor) in the **main session**. After each slice, verify
`write_scope` mechanically with a `git diff` check, then mark the
slice `done`.

This stage does **not** review. Spec compliance is validated by
`/slicespec-verify` (checks V2/V11) and code quality by
`/slicespec-review`. Implement's only gate is the mechanical
controller diff check — it is fast, deterministic, and runs in the
main session.

**Core principles:**

1. **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.** Iron rule.
   No exceptions short of an explicit `/slicespec-escape`.
2. **Test rules come from `test-rules.md` — read it before writing
   any test.** Good/Bad test shapes, the mock boundary, the
   horizontal-slicing anti-pattern, refactor candidates, naming —
   all live there as the single source of truth. The implementer
   applies the §8 pre-flight checklist before every test.
   Anything not in `test-rules.md` is not a test rule.
3. **Scope validated mechanically.** Self-reports about "I didn't
   touch X" are not trusted. `git diff` is.

The TDD essence (Pocock-style discipline: behaviour over
implementation, vertical cycles, mock at the boundary) lives in
`test-rules.md`. The TDD cycle mechanics live in
`implementer-prompt.md`. This file describes the slice workflow
that wraps them.

**Announce at start:** "I'm using slicespec-implement to drive the
TDD loop on slice <id>."

## Inputs

- `changes/<change-id>/slices.md`
- `changes/<change-id>/spec.md`
- `changes/<change-id>/state.json`
- Optional: slice id argument. Default: next unblocked slice by ID
  ascending.

## Process

```
┌──────────────────────────────────────┐
│ 1. Pick next slice                   │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 2. TDD cycles (Red → Green → Refactor)
│    × N until slice complete          │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 3. Controller diff check (BLOCKING)  │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 4. Mark slice done                   │
│ 5. Pick next, or /slicespec-review   │
│    then /slicespec-verify            │
└──────────────────────────────────────┘
```

### Step 1 — Pick next slice

If the user passed a slice id, use it. Otherwise:

1. From state.json, find all `pending` slices.
2. Drop slices whose `blocked_by` set contains any non-`done` slice.
3. Drop any slice with `requires_shared_resource` that conflicts
   with a currently-running slice.
4. Pick the lowest-ID candidate.

If no slice qualifies, announce why and stop. Most common reasons:

- All slices are `done` → suggest `slicespec-verify`.
- A slice is in `escaped(open)` → suggest `slicespec-escape`.
- A slice is `blocked` → describe the blocker.

The `type` field in slices.md (`AFK` or `HITL`) tells you the
slice's *judgement-point profile*:

- `HITL` slices have judgement points where you must pause and ask
  the user. Each Red → Green → Refactor cycle pauses at the
  judgement points the slice was marked HITL for.
- `AFK` slices have none. Run cycles continuously without pausing
  for the user.

In both cases the work runs in the main session by default.

### Step 2 — TDD cycles

**Before writing any test in this slice, read `test-rules.md`
end-to-end.** It is the source of truth for test/mock/anti-pattern
rules. Apply the §8 pre-flight checklist to every test you write.
Any later `/slicespec-review` finding that cites a `test-rules.md`
clause indicates the pre-flight was skipped — it is avoidable
rework. Get it right here so the review stays clean.

Every cycle executes the standard shape:

```
RED      → Apply test-rules.md §8 pre-flight, write ONE test
            for ONE behaviour from ONE Scenario, reference the
            Scenario ID (see shared/scenario-id-rules.md), run it,
            verify it fails for the RIGHT reason.

GREEN    → Minimum code to pass. No speculative interfaces, no
            unrequested features. Mocks only at test-rules.md §4
            boundaries. Run the module's wider tests; verify no
            regressions.

REFACTOR → Only when GREEN. Scan for test-rules.md §6 candidates
            (duplication, shallow modules, feature envy, primitive
            obsession). Never modify behaviour. Tests must remain
            green throughout.

COMMIT   → One commit per cycle. Short present-tense message
            ending with `[<slice-id>]`.
```

**Vertical cycles, never horizontal.** Even when `covers` lists
many Scenarios, run them as small Red→Green cycles — one per
behaviour. The horizontal-slicing anti-pattern (all tests first,
then all impl) is defined and forbidden in `test-rules.md` §5; it
is a Critical defect `/slicespec-review` detects from the commit
log.

Iterate until all Scenarios in `covers` are exercised. Then
proceed to step 3.

For cycle execution detail (hard rules, stuck-escalation, report
format, self-review) read `implementer-prompt.md`.

Refactor boundary rules:

| Change scope | What's allowed |
|---|---|
| Internal structure, private fns, names, local abstractions | Free inside TDD. |
| Test expression, fixtures, mock strategy | Free inside TDD. |
| Module interface consumed only by the team | Update slices.md or Constraints if it widens scope. |
| Externally-consumed stable interface | STOP. Trigger /escape with tag `better-interface`. |
| Observable behaviour, error semantics, data model, permission, security, perf commitment | STOP. /escape. |

### Step 3 — Controller diff check (BLOCKING)

After TDD cycles complete, run the mechanical diff check before
any review:

```
git diff --name-only <pre-slice-sha>..HEAD
```

Compare every modified path against the slice's `write_scope` and
`do_not_touch`:

1. Every modified path must match at least one `write_scope` glob.
2. No modified path may match any `do_not_touch` glob.

If any path fails either check:

- Mark the slice `blocked(scope-violation)` in slices.md and
  state.json.
- Write the violating paths into
  `state.json.slices[sid].evidence.controller_diff_check =
  "failed:<comma-list>"`.
- Do not run reviews.
- Tell the user, suggesting one of:
  - Revert the out-of-scope changes.
  - Run /escape with tag `scope-overflow` or `better-interface` to
    update slices.md, then re-attempt.

`git diff` is authoritative — even your own recollection of "I
didn't touch X" is not. See `controller-diff-check.md` for the
exact algorithm.

### Step 4 — Mark slice done

When the controller diff check is `passed`:

1. Set slices.md status to `done`.
2. Update state.json:
   ```json
   {
     "slices": {
       "<slice-id>": {
         "status": "done",
         "phase": null,
         "evidence": {
           "commits": [...],
           "tests_run": [...],
           "implementer_report": "evidence/<slice-id>/implementer-report.md",
           "controller_diff_check": "passed"
         }
       }
     },
     "updated_at": "<ISO-8601 UTC>"
   }
   ```
3. Save the implementer report (per `implementer-prompt.md`'s
   Report format) under
   `changes/<change-id>/evidence/<slice-id>/implementer-report.md`.

`done` means TDD cycles ran and the diff check passed. It does
**not** mean reviewed. Spec compliance is checked later by
`/slicespec-verify`; code quality by `/slicespec-review`. Neither
runs here.

### Step 5 — Loop or finish

If more unblocked slices remain, return to step 1. Continuous
execution — do not pause to summarise progress unless explicitly
asked. Only stop when:

- All slices `done` → suggest running `/slicespec-review` (code
  quality), then `/slicespec-verify` (spec/test/code alignment +
  archive).
- A slice is `blocked` and you cannot resolve it.
- The user interrupts.
- An /escape was triggered and is still open.

## Iron rules (no exceptions without /escape)

- No production code without a failing test first.
- No refactor while RED.
- No skipping `test-rules.md` §8 pre-flight on any test.
- No widening of `write_scope` mid-slice without /escape.
- No silent test deletion. Deleted tests must be replaced or
  documented in evidence/.
- No skipping the controller diff check.
- One slice at a time.

## Missing inputs

`slicespec-implement` is the only stage that touches code. It
assumes the contract layer is in place. If any input is missing,
stop and delegate:

- `brief.md` missing → run `slicespec-clarify`.
- `spec.md` missing → run `slicespec-spec`.
- `slices.md` missing → run `slicespec-slice`.

There is no auto-generated placeholder, no scenario-less ID space,
no post-hoc backfill flag. If you find yourself wanting one, the
change is not ready for implementation.


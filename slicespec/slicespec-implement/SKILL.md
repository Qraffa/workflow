---
name: slicespec-implement
description: Stage 4 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-implement` (optionally with a slice id). Do NOT auto-trigger from keywords like "implement", "start coding", or "begin TDD" — this skill is user-gated. Executes one slice via TDD in the main session, then dispatches spec-compliance and code-quality reviewers as fresh subagents (default), and verifies write_scope mechanically with a `git diff` check. Implementer subagent dispatch and parallel worktrees are opt-in (see `subagent-mode.md`).
---

# SliceSpec — Implement

Implement one slice at a time using strict TDD (Red → Green →
Refactor) in the **main session**. After every slice, dispatch a
two-stage review (spec compliance, then code quality) as **fresh
subagents** for independence from the implementer's context, and
verify `write_scope` mechanically with a `git diff` check.

**Core principles:**

1. **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.** Iron rule.
   No exceptions short of an explicit `/slicespec-escape`.
2. **Test rules come from `test-rules.md` — read it before writing
   any test.** Good/Bad test shapes, the mock boundary, the
   horizontal-slicing anti-pattern, refactor candidates, naming —
   all live there as the single source of truth. The implementer
   applies the §8 pre-flight checklist before every test; the
   quality reviewer grades violations against the same file.
   Anything not in `test-rules.md` is not a test rule.
3. **Two-stage review, independent by construction.** Spec
   compliance first (was this what was asked?), then code quality
   (is it well built?). Both reviewers run as fresh subagents by
   default — the implementer's context (you, the controller) is
   not the right place to audit the implementer's output.
4. **Scope validated mechanically.** Self-reports about "I didn't
   touch X" are not trusted. `git diff` is.

The TDD essence (Pocock-style discipline: behaviour over
implementation, vertical cycles, mock at the boundary) lives in
`test-rules.md`. The TDD cycle mechanics live in
`implementer-prompt.md`. Reviewer grading lives in
`quality-reviewer-prompt.md`. This file describes the slice
workflow that wraps them.

**Announce at start:** "I'm using slicespec-implement to drive the
TDD loop on slice <id>."

## When to Use

**Invocation rule:** Explicit-only. Runs when the user types
`/slicespec-implement` (optionally with a slice id). Never
auto-trigger from keyword inference; if the context fits, suggest
the command and wait for the user to invoke it.

**Decline (and redirect) when:**

- All slices are `done` → `/slicespec-verify`.
- A slice is `escaped(open)` → `/slicespec-escape` to close it first.
- `spec.md` or `slices.md` does not exist → go back to the earlier
  stage. This skill has no shortcut path.

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
│ 4. Spec compliance review            │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 5. Code quality review               │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 6. Mark slice done                   │
│ 7. Pick next, or /slicespec-verify   │
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
Findings in step 5 (quality review) that cite a `test-rules.md`
clause indicate the pre-flight was skipped — they are avoidable
rework.

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
is a Critical defect the quality review detects from the commit
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

### Step 4 — Spec compliance review (fresh subagent)

Dispatch a fresh subagent using `spec-reviewer-prompt.md` as the
prompt body. Substitute every `<...>` placeholder before
dispatching:

- `<slice-id>`, `<change-id>`, `<worktree>` absolute path
- `<base-sha>` (recorded when slice moved to `in_progress`),
  `<head-sha>` (current HEAD)
- Every Scenario in the slice's `covers`, pasted verbatim from
  `spec.md` (the subagent does NOT read the change directory)
- The implementer's report as a HINT

The subagent reads `test-rules.md` and the diff itself; it does
not trust prior narration. It must verify:

- All `covers` Scenarios are exercised by a test that references
  the Scenario ID.
- No missing Scenarios were skipped.
- No extra observable behaviour was introduced beyond the spec.

Outcome:

- `approved` — proceed to step 5.
- `issues_found` — list issues with file:line refs. Return to step
  2 (TDD) to fix the specific items, then re-dispatch the reviewer
  with the updated `<head-sha>`. **Maximum three iterations** — if
  the third still fails, mark the slice `blocked(spec-review)` and
  ask the user to intervene.

Never silently re-dispatch the same prompt after `issues_found` —
the head SHA must advance.

### Step 5 — Code quality review (fresh subagent)

Only after spec compliance is `approved`, dispatch a fresh
subagent using `quality-reviewer-prompt.md` as the prompt body.
Substitute the same placeholders as Step 4 (`<slice-id>`,
`<change-id>`, `<worktree>`, `<base-sha>`, `<head-sha>`).

The subagent grades violations against the same `test-rules.md`
clauses the implementer used in §8 pre-flight, plus the
audit-only checks that need post-hoc evidence (commit-shape
detection of horizontal slicing and refactor-while-RED, dead-test
detection, rationalisation signals).

Outcome:

- `approved` — proceed to step 6.
- `issues_found` (Critical/Warning/Info per
  `shared/governance-thresholds.md`) — fix Critical unconditionally;
  Warning can be `accepted_with_risk` with a written justification
  recorded in state.json's `reviews[]` array; Info is noted only.
  Return to step 2, then re-dispatch with the updated `<head-sha>`.

**Maximum three iterations.** If the third still fails, mark the
slice `blocked(quality-review)` and ask the user. A finding that
cites a `test-rules.md` clause is rework that should have been
prevented by §8 pre-flight; treat it as a signal to slow down on
the next test, not just a ticket to fix.

### Step 6 — Mark slice done

When both reviews are `approved` (or Warnings accepted with risk),
and the controller diff check is `passed`:

1. Set slices.md status to `done`.
2. Update state.json:
   ```json
   {
     "slices": {
       "<slice-id>": {
         "status": "done",
         "phase": null,
         "owner": null,
         "evidence": {
           "commits": [...],
           "tests_run": [...],
           "spec_review": "approved",
           "spec_review_report": "evidence/<slice-id>/spec-review.md",
           "quality_review": "approved",
           "quality_review_report": "evidence/<slice-id>/quality-review.md",
           "controller_diff_check": "passed"
         }
       }
     },
     "updated_at": "<ISO-8601 UTC>"
   }
   ```
3. Save the review reports under
   `changes/<change-id>/evidence/<slice-id>/*.md`.

### Step 7 — Loop or finish

If more unblocked slices remain, return to step 1. Continuous
execution — do not pause to summarise progress unless explicitly
asked. Only stop when:

- All slices `done` → suggest `slicespec-verify`.
- A slice is `blocked` and you cannot resolve it.
- The user interrupts.
- An /escape was triggered and is still open.

## Subagent mode (opt-in)

By default, the **implementer runs in the main session** and the
**two reviewers run as fresh subagents** (steps 4 and 5).
"Subagent mode" refers to the *additional* opt-in that also moves
the implementer into a subagent — typically to enable parallel
worktrees across multiple AFK slices.

It activates only when the user's invocation contains an explicit
opt-in keyword such as `subagent`, `子agent`, `dispatch`, `并发`,
`parallel`, or `worktree`.

Example trigger: `/slicespec-implement 使用子agent并发实现任务1，2`.

When (and only when) such a trigger is present, read
[`subagent-mode.md`](subagent-mode.md) before proceeding. That file
defines the implementer dispatch protocol, parallel-worktree
mechanics, and the HITL→AFK downgrade rule. Do not load that file
on a plain `/slicespec-implement` call — the default reviewer
dispatch is already specified in steps 4 and 5 above.

## Iron rules (no exceptions without /escape)

- No production code without a failing test first.
- No refactor while RED.
- No skipping `test-rules.md` §8 pre-flight on any test.
- No widening of `write_scope` mid-slice without /escape.
- No silent test deletion. Deleted tests must be replaced or
  documented in evidence/.
- No skipping the controller diff check.
- No skipping spec compliance review.
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

## Templates and references

- **`test-rules.md` — single source of truth for test / mock /
  anti-pattern rules. MANDATORY read before writing any test.
  Both implementer (§8 pre-flight) and quality reviewer
  (clause-cited grading) operate from this file. No other file
  may restate its rules.**
- `implementer-prompt.md` — TDD cycle mechanics (hard rules,
  stuck-escalation, pre-report self-check, report format). Drives
  the main-session implementer by default; also usable as a
  subagent prompt in subagent mode.
- `spec-reviewer-prompt.md` — fresh-subagent prompt body for
  Step 4. Dispatched by default every slice.
- `quality-reviewer-prompt.md` — fresh-subagent prompt body for
  Step 5. Dispatched by default every slice. Cites `test-rules.md`
  clauses for every test-rule finding.
- `controller-diff-check.md` — exact algorithm for the mechanical
  diff check.
- `subagent-mode.md` — opt-in protocol for moving the *implementer*
  into a subagent (e.g. parallel worktrees). Read ONLY when the
  user triggers it (see "Subagent mode" above). Reviewer dispatch
  is already default and does not require this file.

## Related skills

- `slicespec-slice` — produces the slice this skill consumes.
- `slicespec-escape` — handles every mid-slice contract change.
- `slicespec-verify` — runs after all slices are `done`.

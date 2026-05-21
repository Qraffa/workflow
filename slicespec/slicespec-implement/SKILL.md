---
name: slicespec-implement
description: Stage 4 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-implement` (optionally with a slice id). Do NOT auto-trigger from keywords like "implement", "start coding", or "begin TDD" — this skill is user-gated. Executes one slice via TDD in the main session, runs spec-compliance and code-quality reviews, and verifies write_scope mechanically with a `git diff` check. Subagent dispatch is opt-in only (see `subagent-mode.md`).
---

# SliceSpec — Implement

Implement one slice at a time using strict TDD (Red → Green →
Refactor) in the **main session**. After every slice, run a
two-stage self-review (spec compliance, then code quality) using
the reviewer prompts as checklists, and verify `write_scope`
mechanically with a `git diff` check.

**Core principles:**

1. **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.** Iron rule.
   No exceptions short of an explicit `/slicespec-escape`.
2. **Tests verify behaviour through public interfaces.** Not
   internal collaborators, not private methods, not by side-channels
   like raw DB queries. A test that breaks on a pure refactor was
   the wrong test. (Good/Bad test contract lives in
   `quality-reviewer-prompt.md`.)
3. **Vertical, never horizontal.** ONE test → ONE impl → repeat.
   Never write all Scenario tests up front then implement in bulk —
   that produces tests of imagined behaviour. Each cycle is
   informed by what you just learned.
4. **Mock only at system boundaries.** External APIs, DBs, time,
   randomness, filesystem. Never mock code you own or the SUT
   itself.
5. **Two-stage review.** Spec compliance first (was this what was
   asked?), then code quality (is it well built?).
6. **Scope validated mechanically.** Self-reports about "I didn't
   touch X" are not trusted. `git diff` is.

Principles 2–4 are the TDD essence inherited from the Pocock TDD
discipline. The full rationale, examples, and reviewer rubric live
in `implementer-prompt.md` and `quality-reviewer-prompt.md` — they
are not repeated here to keep this skill thin.

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

Every cycle must execute:

```
RED
  ├─ Write a test for ONE behaviour from one Scenario.
  ├─ Verify behaviour through the PUBLIC interface (no internal
  │  mocks, no raw DB / private-state inspection).
  ├─ Reference the Scenario ID using one of the three permitted
  │  forms (see shared/scenario-id-rules.md).
  └─ Run it. Verify it fails for the RIGHT reason (feature missing,
     not typo).

GREEN
  ├─ Write the minimum code to make the test pass.
  ├─ No speculative interfaces. No extra features. No "while I'm
  │  here".
  ├─ Mock only at system boundaries (external APIs, DB,
  │  time/random, filesystem). Never mock code you own or the SUT.
  └─ Run the test. Verify it passes. Run the whole module's tests.
     Verify no regressions.

REFACTOR
  ├─ Only when GREEN.
  ├─ Improve names, extract helpers, reduce duplication. Watch for
  │  shallow modules to deepen, feature envy to relocate, primitive
  │  obsession to absorb into value objects.
  ├─ Never modify behaviour during refactor.
  └─ Run tests after every meaningful change.

COMMIT
  └─ One commit per cycle (often Red+Green+Refactor in one commit;
     larger refactors get their own commit). Message format: short
     present-tense sentence ending with `[<slice-id>]`.
```

**Vertical cycles, never horizontal.** Even when `covers` lists
many Scenarios, run them as small Red→Green cycles, one per
behaviour. Do NOT batch all the Scenario tests first and implement
them after — that produces tests of imagined behaviour, not real
behaviour, and is a Critical defect detected by the quality review.

Iterate until all Scenarios in `covers` are exercised. Then
proceed to step 3.

For the detailed TDD discipline (anti-patterns, mock rules,
self-review checklist, refactor candidates) read
`implementer-prompt.md`. The prompt template is the source of
truth for what a correct cycle looks like.

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

### Step 4 — Spec compliance review

Read `spec-reviewer-prompt.md` and use it as your **self-review
checklist**. Verify:

- All `covers` Scenarios are exercised by at least one test that
  references the Scenario ID.
- No missing Scenarios were skipped.
- No extra observable behaviour was introduced beyond the spec.

Outcome:

- `approved` — all Scenarios covered, nothing extra.
- `issues_found` — list issues with file:line refs. Return to step
  2 (TDD) to fix the specific items, then re-review. Maximum three
  iterations — if the third still fails, mark the slice
  `blocked(spec-review)` and ask the user to intervene.

### Step 5 — Code quality review

Only after spec compliance is `approved`, read
`quality-reviewer-prompt.md` and use it as your **self-review
checklist**. Verify:

- Tests verify behaviour through public interfaces (no
  over-mocking internal collaborators).
- No speculative code, dead code, or over-abstraction.
- No refactor performed while RED.
- File responsibilities are clear; no accidental dumping ground.
- Names match what things do, not how they work.

Outcome:

- `approved` — proceed to step 6.
- `issues_found` (Critical/Warning/Info per
  `shared/governance-thresholds.md`) — fix Critical unconditionally;
  Warning can be `accepted_with_risk` with a written justification
  recorded in state.json's `reviews[]` array; Info is noted only.

Maximum three iterations.

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

By default, this skill runs entirely in the main session. Subagent
dispatch is **optional** and only activates when the user's
invocation contains an explicit opt-in keyword such as `subagent`,
`子agent`, `dispatch`, `并发`, `parallel`, or `worktree`.

Example trigger: `/slicespec-implement 使用子agent并发实现任务1，2`.

When (and only when) such a trigger is present, read
[`subagent-mode.md`](subagent-mode.md) before proceeding. That file
defines the dispatch protocol, parallel-worktree mechanics, the
HITL→AFK downgrade rule, and how reviewers may be dispatched as
fresh subagents for independence. Do not load that file on a
plain `/slicespec-implement` call.

## Iron rules (no exceptions without /escape)

- No production code without a failing test first.
- No refactor while RED.
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

- `implementer-prompt.md` — TDD discipline (anti-patterns, mock
  rules, refactor candidates, self-review). Source of truth for
  cycle execution. Also usable as a subagent prompt in subagent
  mode.
- `spec-reviewer-prompt.md` — spec compliance rubric. Used as a
  self-review checklist by default; usable as a subagent prompt in
  subagent mode.
- `quality-reviewer-prompt.md` — code quality rubric. Same dual
  use.
- `controller-diff-check.md` — exact algorithm for the mechanical
  diff check.
- `subagent-mode.md` — opt-in subagent dispatch protocol. Read
  ONLY when the user triggers it (see "Subagent mode" above).

## Related skills

- `slicespec-slice` — produces the slice this skill consumes.
- `slicespec-escape` — handles every mid-slice contract change.
- `slicespec-verify` — runs after all slices are `done`.

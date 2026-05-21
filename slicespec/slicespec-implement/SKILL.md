---
name: slicespec-implement
description: Stage 4 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-implement` (optionally with a slice id). Do NOT auto-trigger from keywords like "implement", "start coding", or "begin TDD" — this skill is user-gated. Executes one slice via TDD with subagent dispatch (AFK) or direct main-session work (HITL), then runs spec-compliance and code-quality review subagents. Includes mechanical controller diff check.
---

# SliceSpec — Implement

Implement one slice at a time using strict TDD (Red → Green → Refactor),
dispatched to a fresh subagent for AFK slices and run in the main
session for HITL slices. After every slice, run a two-stage review
(spec compliance, then code quality), and verify write_scope mechanically
with a controller-side `git diff` check.

**Core principles:**

1. **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.** Iron rule. No
   exceptions short of an explicit `/slicespec-escape`.
2. **Tests verify behaviour through public interfaces.** Not internal
   collaborators, not private methods, not by side-channels like raw
   DB queries. A test that breaks on a pure refactor was the wrong
   test. (Good/Bad test contract lives in
   `quality-reviewer-prompt.md`.)
3. **Vertical, never horizontal.** ONE test → ONE impl → repeat. Never
   write all Scenario tests up front then implement in bulk — that
   produces tests of imagined behaviour. Each cycle is informed by
   what you just learned.
4. **Mock only at system boundaries.** External APIs, DBs, time,
   randomness, filesystem. Never mock code you own or the SUT itself.
5. **Fresh subagent per AFK slice.** Subagents do not inherit session
   context. The controller crafts a self-contained prompt.
6. **Two-stage review.** Spec compliance first (was this what was
   asked?), then code quality (is it well built?).
7. **Controller validates scope mechanically.** Subagent self-reports
   about "I didn't touch X" are not trusted. `git diff` is.

Principles 2–4 are the TDD essence inherited from the Pocock TDD
discipline. The full rationale, examples, and reviewer rubric live in
`implementer-prompt.md` and `quality-reviewer-prompt.md` — they are
not repeated here to keep this skill thin.

**Announce at start:** "I'm using slicespec-implement to drive the
TDD loop on slice <id>."

## When to Use

**Invocation rule:** Explicit-only. Runs when the user types
`/slicespec-implement` (optionally with a slice id). Never auto-trigger
from keyword inference; if the context fits, suggest the command and
wait for the user to invoke it.

**Decline (and redirect) when:**

- All slices are `done` → `/slicespec-verify`.
- A slice is `escaped(open)` → `/slicespec-escape` to close it first.
- `spec.md` or `slices.md` does not exist → go back to the earlier
  stage. This skill has no shortcut path.

## Inputs

- `changes/<change-id>/slices.md`
- `changes/<change-id>/spec.md`
- `changes/<change-id>/state.json`
- Optional: slice id argument. Default: next unblocked AFK slice by ID
  ascending.

## Process

```
┌──────────────────────────────────────────┐
│ 1. Pick next slice                       │
└──────────────────┬───────────────────────┘
                   ▼
┌──────────────────────────────────────────┐
│ 2. Decide AFK vs HITL                    │
└─────────────┬─────────────┬──────────────┘
              │             │
              ▼             ▼
       ┌───────────┐  ┌─────────────┐
       │ Mode A    │  │ Mode B      │
       │ Subagent  │  │ Main session│
       └─────┬─────┘  └──────┬──────┘
             ▼               ▼
       ┌─────────────────────────────┐
       │ 3. TDD cycles (Red→Green→Refactor)
       │    × N until slice complete
       └────────────────┬────────────┘
                        ▼
       ┌─────────────────────────────┐
       │ 4. Controller diff check    │
       │    (BLOCKING)               │
       └────────────────┬────────────┘
                        ▼
       ┌─────────────────────────────┐
       │ 5. Spec compliance review   │
       └────────────────┬────────────┘
                        ▼
       ┌─────────────────────────────┐
       │ 6. Code quality review      │
       └────────────────┬────────────┘
                        ▼
       ┌─────────────────────────────┐
       │ 7. Mark slice done          │
       │ 8. Pick next, or /verify    │
       └─────────────────────────────┘
```

### Step 1 — Pick next slice

If the user passed a slice id, use it. Otherwise:

1. From state.json, find all `pending` slices.
2. Drop slices whose `blocked_by` set contains any non-`done` slice.
3. Drop any slice with `requires_shared_resource` that conflicts with a
   currently-running slice.
4. Among the remainder, prefer AFK over HITL; among AFK, pick lowest ID.
5. Within the unblocked-AFK set, if multiple slices pass
   `parallel-check.md` against each other, the controller may dispatch
   them in parallel worktrees (see step 2 / parallel section below).

If no slice qualifies, announce why and stop. Most common reasons:

- All slices are `done` → suggest `slicespec-verify`.
- A slice is in `escaped(open)` → suggest `slicespec-escape`.
- A slice is `blocked` → describe the blocker.

### Step 2 — AFK vs HITL

Read the slice's `type` from slices.md.

- `AFK` → Mode A (subagent dispatch).
- `HITL` → Mode B (main-session execution).

**HITL → AFK downgrade** is possible only when:

1. The user has explicitly resolved every judgement point the slice was
   marked HITL for, and those resolutions are stable enough to write
   into a subagent prompt.
2. All remaining work is mechanical (test writing, implementation, no
   open design questions).
3. `write_scope` is still respected by the planned changes.

To downgrade, the controller (main session) makes the call — not the
subagent. On downgrade:

- Set `state.json.slices[sid].originally_hitl = true`.
- Include the user's confirmed decisions verbatim in the dispatch prompt
  as a "Confirmed decisions" block.
- If the subagent encounters a new HITL-class question, it must exit
  with `BLOCKED(needs-hitl-decision)`. It must not invent the answer.

**AFK → HITL** is not allowed. If an AFK slice surfaces a HITL-class
question mid-flight, trigger `/escape` with tag `spec-error` or
`better-interface` rather than promoting to HITL.

### Mode A — AFK subagent dispatch

The controller's job is to construct the subagent prompt from
`implementer-prompt.md`, dispatch it, then act on the result.

Subagent prompt contents (use the template; substitute placeholders):

- Slice id, title, and full body from slices.md.
- All Scenarios from spec.md the slice covers — pasted in full, not
  referenced. Subagent does not read files unless it has to.
- `write_scope` and `do_not_touch` as hard rules.
- Mandatory Red → Green → Refactor loop.
- Required test-reference form (one of three from
  `shared/scenario-id-rules.md`).
- Reporting format: `DONE | DONE_WITH_CONCERNS | BLOCKED |
  NEEDS_CONTEXT`.

Use the `Agent` tool (`subagent_type: general-purpose`) unless the
slice is genuinely mechanical (1-2 files, no integration concerns) in
which case use a smaller / cheaper model. For architecture-heavy slices
that slipped through HITL labelling, use the most capable model
available.

While the subagent runs, do not pause to "check in" with the user. Wait
for its report.

Handle reports:

- `DONE` → step 4.
- `DONE_WITH_CONCERNS` → read concerns. If correctness/scope, re-dispatch
  with guidance. If observational, proceed to step 4 noting the concern.
- `BLOCKED` → assess. If context missing, supply and re-dispatch. If too
  large, split via /escape with tag `scope-overflow`. If a HITL question
  surfaced, /escape with tag `spec-error`. Never silently re-dispatch
  the same prompt.
- `NEEDS_CONTEXT` → provide missing context, re-dispatch.

#### Parallel dispatch

If the candidate set has two or more slices that pass
`../slicespec-slice/parallel-check.md`, the controller may run them in
parallel:

1. Create one git worktree per parallel slice:
   `git worktree add ../<slice-id> -b <slice-id>-branch`
2. Dispatch each subagent against its own worktree path.
3. Wait for all to report.
4. Run controller diff check (step 4), spec review (step 5), quality
   review (step 6) **inside each worktree** before merging.
5. Cherry-pick the `done` worktree commits back to the main worktree in
   the order they finished. Resolve any conflicts in the main worktree
   (controller does this — never a subagent).
6. Remove worktrees after merge.

If any parallel slice ends `blocked` due to scope violation, do not
merge its worktree. Trigger /escape if necessary.

### Mode B — HITL main session

When the slice is HITL, the controller (main session) executes TDD
directly with the user. Each Red → Green → Refactor cycle pauses at the
judgement points the slice was marked HITL for.

The discipline rules from Mode A still apply:

- No production code without a failing test first.
- Each TDD cycle ends with at least one commit.
- After completion, run controller diff check (step 4), spec compliance
  review (step 5), and code quality review (step 6). For HITL slices
  these reviewers are still dispatched as subagents — the implementer
  was you (the controller), but the reviewer must be a fresh subagent
  to keep the read independent.

### Step 3 — TDD cycles

Whether subagent or main session, every cycle must execute:

```
RED
  ├─ Write a test for ONE behaviour from one Scenario.
  ├─ Verify behaviour through the PUBLIC interface (no internal mocks,
  │  no raw DB / private-state inspection).
  ├─ Reference the Scenario ID using one of the three permitted forms.
  └─ Run it. Verify it fails for the RIGHT reason (feature missing, not
     typo).

GREEN
  ├─ Write the minimum code to make the test pass.
  ├─ No speculative interfaces. No extra features. No "while I'm here".
  ├─ Mock only at system boundaries (external APIs, DB, time/random,
  │  filesystem). Never mock code you own or the SUT.
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

**Vertical cycles, never horizontal.** Even when `covers` lists many
Scenarios, run them as small Red→Green cycles, one per behaviour. Do
NOT batch all the Scenario tests first and implement them after —
that produces tests of imagined behaviour, not real behaviour, and is
a Critical defect detected by the quality reviewer.

Iterate until all Scenarios in `covers` are exercised. Then proceed to
step 4.

Refactor boundary rules:

| Change scope | What's allowed |
|---|---|
| Internal structure, private fns, names, local abstractions | Free inside TDD. |
| Test expression, fixtures, mock strategy | Free inside TDD. |
| Module interface consumed only by the team | Update slices.md or Constraints if it widens scope. |
| Externally-consumed stable interface | STOP. Trigger /escape with tag `better-interface`. |
| Observable behaviour, error semantics, data model, permission, security, perf commitment | STOP. /escape. |

### Step 4 — Controller diff check (BLOCKING)

After the implementer reports `DONE` (subagent or main-session), the
controller runs the mechanical diff check before any reviewer subagent
is dispatched.

```
git diff --name-only <pre-slice-sha>..HEAD -- <worktree-path>
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
- Do not dispatch reviewers.
- Tell the user, suggesting one of:
  - Revert the out-of-scope changes.
  - Run /escape with tag `scope-overflow` or `better-interface` to
    update slices.md, then re-dispatch.

Subagent self-reports of "I did not touch X" are not believed.
`git diff` is authoritative. See `controller-diff-check.md` for the
exact algorithm.

### Step 5 — Spec compliance review

Dispatch a fresh subagent using `spec-reviewer-prompt.md`. Its job:

- Read the implementation code and the relevant Scenarios from
  spec.md.
- Verify all `covers` Scenarios are exercised by at least one test that
  references the Scenario ID.
- Look for missing Scenarios the implementer skipped.
- Look for extra observable behaviour the implementer introduced
  beyond the spec.

Reviewer returns:

- `approved` — all Scenarios covered, nothing extra.
- `issues_found` — list of issues with file:line refs.

If `issues_found`, send the implementer back to step 3 (TDD) to fix the
specific items. Re-dispatch the spec reviewer. Maximum three iterations
— if the third still fails, mark the slice `blocked(spec-review)` and
ask the user to intervene.

### Step 6 — Code quality review

Only after spec compliance is `approved`, dispatch a code quality
reviewer with `quality-reviewer-prompt.md`. Its job:

- Tests verify behaviour through public interfaces (no over-mocking
  internal collaborators).
- No speculative code, dead code, or over-abstraction.
- No refactor performed while RED.
- File responsibilities are clear; no accidental dumping ground.
- Names match what things do, not how they work.

Returns:

- `approved` — proceed to step 7.
- `issues_found` (Critical/Warning/Info per
  `shared/governance-thresholds.md`) — implementer fixes Critical
  unconditionally; Warning can be `accepted_with_risk` with a written
  justification recorded in state.json's `reviews[]` array; Info is
  noted only.

Maximum three iterations.

### Step 7 — Mark slice done

When both reviews are `approved` (or Warnings accepted with risk), and
the controller diff check is `passed`:

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
           "implementer_report": "evidence/<slice-id>/implementer-report.md",
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
3. Save each subagent's full report under
   `changes/<change-id>/evidence/<slice-id>/*.md`.

### Step 8 — Loop or finish

If more unblocked slices remain, return to step 1. Continuous execution
— do not pause to summarise progress unless explicitly asked. Only stop
when:

- All slices `done` → suggest `slicespec-verify`.
- A slice is `blocked` and you cannot resolve it.
- The user interrupts.
- An /escape was triggered and is still open.

## Iron rules (no exceptions without /escape)

- No production code without a failing test first.
- No refactor while RED.
- No widening of `write_scope` mid-slice without /escape.
- No silent test deletion. Deleted tests must be replaced or
  documented in evidence/.
- No skipping the controller diff check.
- No skipping spec compliance review.
- No mixing slices in a single dispatch.

## Missing inputs

`slicespec-implement` is the only stage that touches code. It assumes
the contract layer is in place. If any input is missing, stop and
delegate:

- `brief.md` missing → run `slicespec-clarify`.
- `spec.md` missing → run `slicespec-spec`.
- `slices.md` missing → run `slicespec-slice`.

There is no auto-generated placeholder, no scenario-less ID space, no
post-hoc backfill flag. If you find yourself wanting one, the change
is not ready for implementation.

## Templates

- `implementer-prompt.md` — subagent prompt for AFK slices.
- `spec-reviewer-prompt.md` — subagent prompt for spec compliance review.
- `quality-reviewer-prompt.md` — subagent prompt for code quality review.
- `controller-diff-check.md` — the exact algorithm for the mechanical
  diff check.

## Related skills

- `slicespec-slice` — produces the slice this skill consumes.
- `slicespec-escape` — handles every mid-slice contract change.
- `slicespec-verify` — runs after all slices are `done`.


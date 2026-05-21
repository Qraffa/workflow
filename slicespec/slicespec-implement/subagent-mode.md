# Subagent Mode (opt-in)

This guide is **only loaded** when the user explicitly opts into
subagent execution. The default mode of `slicespec-implement` is
main-session, single-threaded. Do not read or apply anything below
unless the trigger conditions in §1 are met.

## 1. Trigger conditions

Enter subagent mode only when the user's `/slicespec-implement`
invocation contains an explicit opt-in. Recognised triggers (Chinese
and English, case-insensitive substring match on the message):

- `subagent`, `子 agent`, `子代理`, `子智能体`
- `dispatch`, `派发`, `分派`
- `parallel`, `concurrent`, `并发`, `并行`
- `worktree`, `工作树`
- `AFK mode`, `AFK 模式`

Examples that trigger subagent mode:

- `/slicespec-implement 使用子agent并发实现任务1，2`
- `/slicespec-implement dispatch s03 to a subagent`
- `/slicespec-implement run s01 and s02 in parallel worktrees`

Examples that do NOT trigger subagent mode (default main-session
applies):

- `/slicespec-implement`
- `/slicespec-implement s03`
- `/slicespec-implement implement the next slice`

If unsure, do NOT enter subagent mode. Ask the user to confirm.

## 2. Where this mode plugs into the main flow

Subagent mode replaces **step 2 (TDD cycles)** of `SKILL.md` with
the dispatch protocol below. Everything else from `SKILL.md` still
applies unchanged:

- Step 1 (Pick slice) — same.
- Step 2 (TDD cycles) — **replaced** by §4 dispatch protocol.
- Step 3 (Controller diff check) — same. ALWAYS run in main session.
  Subagent self-reports of "I did not touch X" are never trusted.
- Step 4 (Spec compliance review) — by default, main session uses
  `spec-reviewer-prompt.md` as a self-review checklist. In subagent
  mode, it MAY be dispatched as a fresh subagent (§5) for
  independence, but this is optional.
- Step 5 (Code quality review) — same dual-mode treatment.
- Step 6 (Mark done) — same.
- Step 7 (Loop) — same.

## 3. AFK / HITL labelling inside subagent mode

The `type` field in slices.md (`AFK` or `HITL`) describes the
slice's *judgement-point profile*, set during `slicespec-slice`:

- `AFK` — no judgement points. Mechanical end-to-end.
- `HITL` — has judgement points where a human must decide.

Default mode ignores this distinction beyond knowing when to pause.
Subagent mode uses it as the dispatch eligibility gate:

- `AFK` slice → eligible for subagent dispatch.
- `HITL` slice → NOT eligible. Run in main session even when subagent
  mode is requested, because a subagent cannot pause and ask the
  user. If the user wants a HITL slice dispatched, see §3.1.

### 3.1 HITL → AFK downgrade

Allowed only when ALL of:

1. Every judgement point the slice was marked HITL for has been
   resolved by the user *before* dispatch, and those resolutions
   are stable enough to write verbatim into the subagent prompt.
2. All remaining work is mechanical.
3. `write_scope` still covers the planned changes.

The controller (main session) — not a subagent — makes this call.
On downgrade:

- Set `state.json.slices[<sid>].originally_hitl = true`.
- Include the user's confirmed decisions verbatim in the dispatch
  prompt as a `## Confirmed decisions` block.
- If the subagent encounters a new HITL-class question, it MUST
  exit with `BLOCKED(needs-hitl-decision)`. It must not invent the
  answer.

**AFK → HITL is not allowed.** If an AFK slice surfaces a HITL-class
question mid-flight, trigger `/escape` with tag `spec-error` or
`better-interface`, not a promotion to HITL.

## 4. Dispatching the implementer subagent

The controller constructs the prompt from `implementer-prompt.md`,
dispatches via the `Agent` tool, and acts on the result.

Prompt contents (substitute every placeholder):

- Slice id, title, and full body from slices.md.
- All Scenarios from spec.md the slice `covers` — pasted in full.
  The subagent does NOT read source files of the change directory.
- `write_scope` and `do_not_touch` as hard rules.
- The mandatory Red → Green → Refactor loop from
  `implementer-prompt.md`.
- Required test-reference form (one of three from
  `shared/scenario-id-rules.md`).
- Reporting format: `DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT`.

Model selection:

- Default: `general-purpose` (Sonnet-class).
- Purely mechanical slice (1–2 files, no integration): a smaller
  / cheaper model is acceptable.
- Slice that slipped HITL labelling but is still being dispatched
  (rare; user has overridden): use the most capable model available.

While the subagent runs, do NOT pause to "check in" with the user.
Wait for the report.

### 4.1 Handling reports

- `DONE` → main flow step 3 (controller diff check).
- `DONE_WITH_CONCERNS` → read concerns. If correctness/scope,
  re-dispatch with guidance. If observational, continue to step 3
  noting the concern in `state.json`.
- `BLOCKED` → assess:
  - context missing → supply and re-dispatch.
  - too large → split via `/escape` with tag `scope-overflow`.
  - HITL question surfaced → `/escape` with tag `spec-error`.
  - Never silently re-dispatch the same prompt.
- `NEEDS_CONTEXT` → provide missing context, re-dispatch.

## 5. Reviewer dispatch (optional)

By default, the main session runs spec compliance and code quality
reviews itself, using `spec-reviewer-prompt.md` and
`quality-reviewer-prompt.md` as checklists. This is normally
sufficient.

In subagent mode, the controller MAY dispatch reviewers as fresh
subagents to gain implementation-independent reads. This is
**recommended** whenever the implementer was a subagent (since the
controller already lives in a different context, the additional
isolation cost is small and the auditor independence is genuine).

If dispatched:

- Spec reviewer: use `spec-reviewer-prompt.md` verbatim.
- Quality reviewer: use `quality-reviewer-prompt.md` verbatim.
- Reviewers must be FRESH subagents — they do not inherit the
  implementer's context.
- Iterate up to 3 times if `issues_found`. On the third failure,
  mark the slice `blocked(spec-review)` or `blocked(quality-review)`
  and ask the user.

For HITL slices implemented in main session, dispatching reviewers
as subagents is still allowed — it keeps the read independent of
the implementer (you, the controller).

## 6. Parallel dispatch (worktrees)

Only attempt parallel dispatch when the user explicitly asks (e.g.
"并发", "parallel"). Even then, the candidate slices must pass
`../slicespec-slice/parallel-check.md` against each other.

Protocol:

1. Create one git worktree per parallel slice:
   ```
   git worktree add ../<slice-id> -b <slice-id>-branch
   ```
2. Dispatch each implementer subagent against its own worktree path.
3. Wait for ALL to report. Do not start merging while any are still
   running.
4. Inside each worktree run:
   - controller diff check (main flow step 3)
   - spec compliance review (main flow step 4, dispatched here)
   - code quality review (main flow step 5, dispatched here)
5. Cherry-pick the `done` worktree commits back to the main
   worktree in the order they finished. The controller (never a
   subagent) resolves conflicts in the main worktree.
6. Remove the worktrees after a clean merge:
   ```
   git worktree remove ../<slice-id>
   ```

If any parallel slice ends `blocked` due to scope violation, do NOT
merge that worktree. Trigger `/escape` if needed; leave the
worktree in place until the escape is resolved.

## 7. Iron rules specific to subagent mode

- Subagent self-reports about scope ("I did not touch X") are NOT
  trusted. The controller's `git diff` check is authoritative.
- Never silently re-dispatch the same prompt after a `BLOCKED`
  report. Always adjust context, decisions, or trigger `/escape`.
- One slice per dispatch. Never mix slices in a single subagent
  invocation.
- The controller never delegates conflict resolution, scope
  decisions, or escape-tag selection to a subagent.

## 8. Exiting subagent mode

After all slices the user asked to be dispatched are `done` (or
blocked), return to the default behaviour. If subsequent slices in
the same session do not include a fresh opt-in, run them in main
session.

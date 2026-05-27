---
name: slicespec-review
description: Code-quality review for SliceSpec. ONLY invoke when the user explicitly types `/slicespec-review` (optionally with a change id). Do NOT auto-trigger from keywords like "review", "check the code", or "audit" — this skill is user-gated. Reviews the whole change's `done` slices as one combined diff against `test-rules.md` (test/mock/anti-pattern quality, commit-shape detection, YAGNI), records a change-level `quality_review` verdict in state.json, and writes a report. It reports findings; it does not fix them and does not change slice status.
---

# SliceSpec — Review

Independent code-quality review of a change after its slices are
implemented. Reviews **all `done` slices of the change as one
combined diff** against the test/mock/anti-pattern rules in
`test-rules.md`, plus the audit-only checks that need post-hoc
evidence (commit-shape detection of horizontal slicing and
refactor-while-RED, dead-test detection, rationalisation signals).

**Core principles:**

1. **Whole-change, not per-slice.** This skill audits the change's
   accumulated diff in one pass. Slice boundaries inform the read
   (which commits belong to which slice) but the verdict is for the
   change.
2. **Independent by construction.** Run the audit as a **fresh
   subagent** so the review does not inherit the implementer's
   context. The implementer's own narration is a hint, never the
   truth — read code and commit history.
3. **Report, don't fix.** This skill lists findings with file:line
   and a `test-rules.md` clause; it does not edit code and does not
   move any slice's status. The user decides whether to fix (back
   in `/slicespec-implement`) or accept the risk.
4. **`test-rules.md` is the rule source.** Every quality finding
   maps to a clause there. The reviewer adds *detection and
   grading* on top — see `reviewer-prompt.md`.

**Announce at start:** "I'm using slicespec-review to audit the
code quality of change `<change-id>`."

## Inputs

- `changes/<change-id>/slices.md`
- `changes/<change-id>/state.json`
- `changes/<change-id>/spec.md` (for scope of what was built)
- `slicespec-implement/test-rules.md` (the rule source)
- Git history of the change's commits.

## Process

```
┌──────────────────────────────────────┐
│ 1. Resolve change, collect done slices│
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 2. Compute combined diff range        │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 3. Dispatch quality reviewer subagent │
└─────────────────┬────────────────────┘
                  ▼
┌──────────────────────────────────────┐
│ 4. Record verdict + write report      │
└──────────────────────────────────────┘
```

### Step 1 — Resolve change and collect slices

If the user passed a change id, use it. Otherwise infer from
conversation context; if ambiguous, ask.

From state.json, collect every slice with `status == "done"`. If
none are `done`, stop and say so (nothing to review yet). Note any
slices still `pending`, `in_progress`, `blocked`, or `escaped` —
the review covers only what is `done`, and the report says so.

### Step 2 — Compute the combined diff range

The review looks at the union of all `done` slices' commits.

1. Gather each `done` slice's commit list from
   `state.json.slices[*].evidence.commits`.
2. Resolve the earliest base SHA across those slices (the change's
   branch point) and the latest HEAD.
3. The combined range is `<base>..<head>`. The full commit log over
   that range is what feeds commit-shape detection.

If commits are spread across worktrees that were merged back, use
the merged main-branch history.

### Step 3 — Dispatch the quality reviewer

Dispatch a **fresh subagent** using `reviewer-prompt.md` as the
prompt body. Substitute every `<...>` placeholder:

- `<change-id>`, `<worktree>` absolute path
- `<base-sha>`, `<head-sha>` for the combined range
- The list of `done` slice ids and, per slice, its `covers`
  Scenario ids and `write_scope` (so the reviewer can check file
  shape and YAGNI against what each slice was meant to build)

The subagent opens `test-rules.md`, reads the diff and the commit
log, and grades findings. It does **not** re-verify Scenario
coverage — that is `/slicespec-verify`'s V2/V11. It focuses on how
the code is built.

### Step 4 — Record verdict and write report

When the subagent returns:

1. Write the report to
   `changes/<change-id>/evidence/quality-review.md` (the
   change-level report; per-slice subdirectories are not used for
   this review).
2. Update state.json with a **change-level** record (no slice
   status changes):
   ```json
   {
     "quality_review": {
       "ran": true,
       "verdict": "approved | issues_found",
       "severity_breakdown": {"critical": 0, "warning": 2, "info": 1},
       "report": "evidence/quality-review.md",
       "reviewed_slices": ["<change-id>-s01", "<change-id>-s02"],
       "ts": "<ISO-8601 UTC>"
     },
     "updated_at": "<ISO-8601 UTC>"
   }
   ```
3. Bump `updated_at`.

Report the verdict to the user:

- `approved` — no Critical or Warning findings (Info allowed).
  Suggest running `/slicespec-verify` next.
- `issues_found` — list Critical and Warning findings with
  file:line and the `test-rules.md` clause. Do **not** fix them and
  do **not** change any slice's status. Tell the user the fix path:
  re-enter `/slicespec-implement` on the affected slice to reshape
  and re-commit, then re-run `/slicespec-review`. A Warning the user
  chooses to keep can be accepted at `/slicespec-verify` time with a
  recorded justification.

## Relationship to verify

`/slicespec-review` is **optional and advisory**. It does not gate
`/slicespec-verify` — a change can be verified without it. But
`/slicespec-verify` emits an Info note when
`state.json.quality_review.ran` is absent or false, recommending a
review pass before archive. Running review first surfaces quality
defects while they are cheap to fix.

## Anti-patterns

| Symptom | Fix |
|---|---|
| Review edits the code to fix a finding. | Stop. This skill reports only. Fixing happens in `/slicespec-implement`. |
| Review flips a slice to `blocked` or `reviewed`. | Stop. Slice status is owned by `/slicespec-implement`; review writes only the change-level `quality_review` record. |
| Review re-checks Scenario coverage. | That is `/slicespec-verify` V2/V11. Review focuses on code quality. |
| Review run in the main session, inheriting implementer context. | Dispatch a fresh subagent for independence. |
| Findings without a `test-rules.md` clause flagged as Critical/Warning. | Only YAGNI / file-shape / Info remarks may lack a clause. |

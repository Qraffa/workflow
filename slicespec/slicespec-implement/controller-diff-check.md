# Controller Diff Check (mechanical)

The controller (main session) runs this check BEFORE dispatching any
reviewer subagent. Its purpose: confirm the implementer's actual file
changes match the slice's `write_scope` whitelist and avoid the
`do_not_touch` blacklist.

This is mechanical. The reviewer subagents are NOT asked to judge
scope. Subagent self-reports about "I did not touch X" are not
authoritative — `git diff` is.

## Inputs

- `<base-sha>` — the SHA before the slice started (state.json records
  this when the slice transitions to `in_progress`).
- `<head-sha>` — the current HEAD (or the worktree's HEAD for parallel
  slices).
- `slices.md` — for `write_scope` and `do_not_touch` globs.

## Algorithm

```
1. Collect changed files:

   git diff --name-status <base>..<head> [-- <worktree-path>]

   Capture both modified and renamed files. For renames, both the old
   and new paths count.

2. Expand globs in write_scope and do_not_touch using shell-style glob
   matching (the same glob engine the project uses; default: fnmatch
   with `**` recursive).

3. For each changed file:
   a. If it matches any do_not_touch glob → MARK VIOLATION (file, "dnt").
   b. Else if it matches at least one write_scope glob → OK.
   c. Else → MARK VIOLATION (file, "out-of-scope").

4. Sum violations.

5. If violations > 0:
   - Set slices.md status to `blocked(scope-violation)`.
   - Write state.json:
       slices[sid].status = "blocked"
       slices[sid].blocked_reason = "scope-violation"
       slices[sid].evidence.controller_diff_check = "failed:<comma-list>"
   - Report to user. Suggest:
       (a) Revert the out-of-scope changes.
       (b) Run /escape with tag `scope-overflow` or `better-interface`
           to widen write_scope, then re-dispatch.
   - Do NOT dispatch reviewers.

6. If violations == 0:
   - state.json.slices[sid].evidence.controller_diff_check = "passed"
   - Proceed to spec-compliance reviewer dispatch.
```

## Special cases

### Worktree dispatches (parallel mode)

When a slice is implemented in a worktree, run the diff command against
the worktree path:

```
git -C <worktree-path> diff --name-status <base>..HEAD
```

The base SHA is the worktree's branch point, recorded in state.json
when the worktree was created.

### Subagent that creates evidence files

A subagent dispatched for slice `s03` is allowed to create files under
`changes/<change-id>/evidence/<slice-id>/` (its own evidence
directory). This single subdirectory is implicitly added to write_scope
at dispatch time. Other slices' evidence directories remain forbidden.

The implementer-prompt template includes the implicit allow as a hard
rule.

### Renames

For renamed files: both the old and the new path must each pass the
write_scope/do_not_touch check.

- If `src/auth/login.py` → `src/identity/login.py`, then both `src/auth/**`
  and `src/identity/**` must be in write_scope. If only one is, the
  rename is a violation.

### Deletions

Deleted files count as modified. They must still match write_scope
(deletion is a write). do_not_touch blocks deletion the same as it
blocks modification.

## False-negative protection

The diff check is not asked to interpret intent. If the implementer
genuinely needed to touch an out-of-scope file (e.g. they discovered a
dependency the slice didn't account for), the right answer is /escape,
not "the reviewer agreed it was fine". The reviewer never sees the
diff-check question.

This rule exists because LLM subagents are reliably bad at counting and
self-auditing. A mechanical check catches what self-review misses.

## When the algorithm cannot decide

Two situations where the algorithm explicitly fails closed (block):

1. The base SHA cannot be resolved (worktree was clobbered).
2. A glob in write_scope cannot be expanded (malformed pattern).

In both cases mark the slice `blocked(scope-violation)` with reason
`diff-check-unreliable` and ask the user to repair the state. Never
"trust" a check it could not run.

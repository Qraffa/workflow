# Controller Diff Check (mechanical)

The main session runs this check after the slice's TDD cycles, BEFORE
marking the slice `done` (SKILL.md step 3 → step 4). Its purpose:
confirm the slice's actual file changes match its `write_scope`
whitelist and avoid the `do_not_touch` blacklist.

This is mechanical and the only gate `/slicespec-implement` enforces.
Self-reports about "I did not touch X" are not authoritative — `git
diff` is. (Spec compliance and code quality are checked later by
`/slicespec-verify` and `/slicespec-review`, not here.)

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
           to widen write_scope, then re-attempt.
   - Do NOT mark the slice `done`.

6. If violations == 0:
   - state.json.slices[sid].evidence.controller_diff_check = "passed"
   - Proceed to mark the slice `done` (SKILL.md step 4).
```

## Special cases

### Worktree runs (external parallel orchestration)

When a slice is implemented in a separate worktree (multiple agents
running `/slicespec-implement` in parallel — see
`../slicespec-slice/parallel-check.md`), run the diff command against
the worktree path:

```
git -C <worktree-path> diff --name-status <base>..HEAD
```

The base SHA is the worktree's branch point, recorded in state.json
when the worktree was created.

### Evidence files

The slice's own evidence directory,
`changes/<change-id>/evidence/<slice-id>/`, is implicitly part of
write_scope — the implementer writes its report there. Other slices'
evidence directories remain forbidden, and `changes/**` is otherwise on
the default `do_not_touch` list.

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
not a self-judgement that "it was fine".

This rule exists because LLMs are reliably bad at counting and
self-auditing. A mechanical check catches what self-review misses.

## When the algorithm cannot decide

Two situations where the algorithm explicitly fails closed (block):

1. The base SHA cannot be resolved (worktree was clobbered).
2. A glob in write_scope cannot be expanded (malformed pattern).

In both cases mark the slice `blocked(scope-violation)` with reason
`diff-check-unreliable` and ask the user to repair the state. Never
"trust" a check it could not run.

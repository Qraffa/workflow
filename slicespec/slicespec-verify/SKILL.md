---
name: slicespec-verify
description: Final stage of SliceSpec. Validates Spec/Test/Code semantic alignment via the V1-V10 checklist, runs governance thresholds, executes the test suite, syncs delta specs into specs/<capability>/spec.md, and archives the change.
---

# SliceSpec — Verify

The final gate. Validates that spec, tests, and code agree, that the
escape hatch was closed cleanly, that no slice silently strayed
outside its declared scope, and that no archived Scenario ID was
reused. Then syncs the change's delta into the long-lived spec
repository and archives the change directory.

**Core principle:** Verify does not re-do work. It looks for the
specific failure modes the rest of the workflow does not catch:
drift, silent contract changes, reused IDs, deferred work that was
never backfilled.

**Announce at start:** "I'm using slicespec-verify to validate the
change before archive."

## When to Use

Trigger this skill when:

- Every slice in slices.md is `done` or `escaped(closed)`.
- The user wants a pre-PR sanity check (no archive yet).
- The user wants to archive several completed changes at once
  ("archive all completed changes").
- CI is running and needs an exit code.

Do **not** use this skill when:

- A slice is still `in_progress`, `blocked`, or `escaped(open)`. Fix
  those first.
- A `resolution_pending: yes` escape entry exists. Close it first.

## Inputs

- `changes/<change-id>/brief.md`
- `changes/<change-id>/spec.md`
- `changes/<change-id>/slices.md`
- `changes/<change-id>/escapes.log`
- `changes/<change-id>/state.json`
- `changes/<change-id>/evidence/<slice-id>/*.md` per slice
- `specs/<capability>/spec.md` (where delta blocks will merge)
- `shared/governance-thresholds.md`
- `shared/scenario-id-rules.md`

## Modes

The mode is determined from natural-language intent, not flags. The
skill must read the user's request and announce which mode it chose
before running.

| Intent phrase | Mode | Behaviour |
|---|---|---|
| "PR pre-check", "verify before merge", "sanity check" | `pre-pr` | Run V1-V10 + test suite. Do NOT sync, do NOT archive. |
| "verify and archive", "wrap up this change", no mode said | `default` | Run V1-V10 + tests. Block on Critical only. Sync and archive on success. |
| "strict verify", "audit mode", "strict mode" | `strict` | Run V1-V10 + tests. Block on Critical AND Warning (no `accepted_with_risk` allowed). Sync and archive only on full pass. |
| "archive all completed changes" | `bulk` | Loop through every `verifying`-status change in `changes/`, run default mode per change, archive each that passes. |

## Process

```
┌──────────────────────────────────────┐
│ 1. Resolve change(s) and mode        │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 2. Structural validation             │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 3. Semantic V1-V10                   │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 4. Escape governance check           │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 5. Test suite run (local + CI)       │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 6. Write verify-report.{md,json}     │
└────────────────┬─────────────────────┘
                 ▼
       ┌──────────────────────────┐
       │ Mode = pre-pr ?          │
       └──────┬───────────────────┘
              │ yes ──> stop (report only)
              │ no
              ▼
┌──────────────────────────────────────┐
│ 7. Sync delta to specs/<capability>/ │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 8. Archive change directory          │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 9. Optional: open PR                 │
└──────────────────────────────────────┘
```

### Step 1 — Resolve change(s) and mode

If the user passed a change id, use it. Otherwise infer from
conversation context. If multiple `verifying`-status changes exist and
the user said "archive all", use bulk mode. If ambiguous, ask which
change.

Announce: "Verifying change `<change-id>` in `<mode>` mode."

### Step 2 — Structural validation

Parse spec.md. Reject (exit code 3) if any of:

- A Scenario heading is not exactly four `#`.
- A Scenario lacks one of `**GIVEN**`, `**WHEN**`, `**THEN**`.
- A Scenario lacks an `<!-- id: ... -->` HTML comment.
- A `## MODIFIED Requirements` block lacks `**Supersedes**` when the
  Scenario IDs differ from the originals.
- A `## REMOVED Requirements` block lacks `**Reason**` or
  `**Migration**`.
- A Scenario ID does not match `<cap>.<slug>.<seq>` format.
- A delta block has cross-conflicting operations (same Scenario ID
  appears in both ADDED and REMOVED).

Structural errors block; they are not "Warnings".

### Step 3 — Semantic V1-V10

Run each check. Results are recorded in verify-report.md per check id.
Use `verify-checklist.md` for the full text.

| Id | What it checks |
|---|---|
| V1 | spec.md still expresses real business intent (LLM scan for vague language and stale references). |
| V2 | Each active Scenario has a test referencing its ID in one of the three permitted forms. |
| V3 | External interfaces, error semantics, data models, permissions, security, perf commitments match the implementation. |
| V4 | Acceptance / Integration / Contract tests cover the primary acceptance paths. |
| V5 | Unit tests cover key rules, edge cases, complex state. |
| V6 | `escapes.log` has no `resolution_pending: yes` entries. |
| V7 | New business semantics discovered during TDD are reflected in spec.md (compare git history of spec.md against implementation diffs). |
| V8 | spec.md does not leak internal implementation detail. |
| V9 | Every slice's actual diff respected its `write_scope` and `do_not_touch` (read `state.json.slices[*].evidence.controller_diff_check`; all must be `passed` OR followed by a documented `/escape` widening). |
| V10 | No `## ADDED` Scenario ID intersects the reserved set (archived IDs + REMOVED + superseded). |

Each check returns one of: `pass`, `warning`, `critical`. Warnings can
be `accepted_with_risk` in default mode if state.json carries a
written justification.

### Step 4 — Escape governance

Per `shared/governance-thresholds.md`:

- `escapes_count > 3` → Warning.
- `escapes_count > slices_total / 2` → Critical (block).
- Any escape with `resolution_pending: yes` → Critical (block).

Record results under "Governance" in verify-report.md.

### Step 5 — Test suite

Run the project's test runner. The command is project-specific; resolve
it in this priority:

1. `package.json`'s `test` script (Node).
2. `pyproject.toml`'s `[tool.pytest.ini_options]` or `Makefile`'s `test`.
3. `Cargo.toml` (`cargo test`), `go test ./...`, etc.
4. User-supplied command (ask if not detected).

Both the test command and its result go into state.json `verify[]` and
the report.

For high-risk modules (per project convention) or strict mode, also run
the contract-test target if it exists. If absent, report it as Info.

### Step 6 — Write verify-report

Write two files:

- `changes/<change-id>/verify-report.md` — human-readable. Use
  `verify-report-template.md`.
- `changes/<change-id>/verify-report.json` — machine-readable. Same
  data, structured.

Each report contains:

- Mode chosen.
- Result of V1-V10 (`pass` / `warning` / `critical`).
- Escape governance result.
- Test suite results.
- Final verdict and exit code.

Also append the run to `state.json.verify[]`.

#### Exit codes (CI contract)

| Exit code | Meaning |
|---|---|
| 0 | Pass. (Default mode: no Critical, Warnings OK or accepted-with-risk. Strict mode: no Critical, no Warning.) |
| 1 | Critical found. Block merge/archive. |
| 2 | Warnings present, strict mode is on. Block merge/archive. |
| 3 | Structural error (spec parse failed, slice missing required field, etc.). Block. |

The exit code is written into verify-report.json under `exit_code`.

### Step 7 — Sync (skipped in pre-pr mode)

For each capability touched:

- Open `specs/<capability>/spec.md`. If it does not exist, create it
  with the post-merge state (i.e. apply `## ADDED Requirements` as the
  initial content).
- Apply each delta block from `changes/<change-id>/spec.md`:
  - `## ADDED Requirements` → append.
  - `## MODIFIED Requirements` → replace by ID (semantic-preserving) or
    add+retire (when `**Supersedes**` present).
  - `## REMOVED Requirements` → strike (keep the heading with `(removed
    in <change-id>)` annotation for one cycle? — implementations may
    choose to truly delete; this skill's default is to truly delete,
    with the reason+migration captured in the change's spec.md and the
    archive).
  - `## RENAMED Requirements` → in-place rename.
- After merge, validate the resulting `specs/<capability>/spec.md` is
  syntactically clean (same structural rules from step 2).
- Commit the merged file.

If conflicts arise (two delta blocks touch the same Scenario in
contradictory ways), abort sync and surface the conflict — never
auto-resolve.

### Step 8 — Archive

Move `changes/<change-id>/` to
`changes/archive/YYYY-MM-DD-<change-id>/`.

Commit. Optionally `gh pr create` if the user asks. Default branch and
title come from the change id and brief.md "Goal".

### Step 9 — PR (optional)

If the user said "open a PR" or the project has a PR convention, run:

```
gh pr create \
  --title "feat: <change-goal>" \
  --body "$(cat changes/archive/YYYY-MM-DD-<change-id>/verify-report.md)"
```

(Or equivalent for the project's git host.)

## Strict vs default vs pre-pr (severity table)

| Severity | Default | Strict | Pre-PR |
|---|---|---|---|
| Critical | Block (exit 1) | Block (exit 1) | Block (exit 1) |
| Warning | Pass + note; can be `accepted_with_risk` | Block (exit 2) | Block (exit 2) when strict requested, else pass with note |
| Info | Pass + note | Pass + note | Pass + note |

`accepted_with_risk` requires a justification recorded in
`state.json.reviews[].warnings_accepted_with_risk[]` — never produced
by `slicespec-verify`. If the user wants to override a warning, they
must have done so during `/implement`.

## Bulk mode

When the user asks "archive all completed changes":

- Scan `changes/` for every change with `state.json.status == "verifying"`.
- For each, run default-mode verify in sequence (not parallel — sync
  conflicts are easier to surface serially).
- Stop on the first Critical and ask whether to continue with the
  rest.
- Aggregate exit codes: 0 only if every change passed; otherwise the
  max of per-change exit codes.

## Anti-patterns

| Symptom | Fix |
|---|---|
| Verify-report says "all pass" but `escapes.log` has open entries. | V6 should be Critical; check the parser. |
| Sync silently merged contradictory delta blocks. | Abort sync. Surface conflict. |
| Verify-report claims V9 pass but state.json's `controller_diff_check` shows `failed`. | Treat state.json as authoritative; fix the report logic. |
| Verify-report skipped V2 because no test file was found. | Block. Either no tests exist (Critical) or the test root is configured wrong (ask). |
| Strict mode passed a Warning. | Reject. Re-run in strict logic. |

## Templates

- `verify-checklist.md` — V1-V10 in full prose.
- `verify-report-template.md` — markdown report shape.

## Related skills

- `slicespec-spec` — produces the spec.md that this skill validates and
  merges.
- `slicespec-slice` — produces the slices.md whose statuses this skill
  reads.
- `slicespec-implement` — produces the evidence this skill reads.
- `slicespec-escape` — every entry in escapes.log is something this
  skill governance-checks.


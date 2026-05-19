---
name: flow-close
description: Verify and close an SDD + TDD change by proving Spec/Test/Code semantic consistency, syncing main specs, and archiving. Use when the user invokes /flow:close, wants check-only verification, strict close, or archive-ready changes.
---

# Flow Close

Use this skill when all planned slices are complete or explicitly removed from scope, escapes are closed, and evidence is ready for final verification.

## Inputs

- `brief.md`, `spec.md`, `plan.md`, `state.json`
- Current code diff and test results
- Review reports and evidence files
- Main specs in `flow/specs/`
- User mode: default, `check only`, `strict`, or archive-ready changes

## Verification

Run structured checks:

1. Required artifacts exist and are readable.
2. Every requirement has at least one scenario.
3. Scenario IDs are stable and valid.
4. ADDED, MODIFIED, and REMOVED sections do not conflict.
5. Main active scenarios have automated tests or explicit evidence.
6. Key acceptance, integration, or contract tests reference scenario IDs where practical.
7. External APIs, events, data models, errors, permissions, security, compatibility, audit, and performance commitments match implementation.
8. `plan.md` slices are done or explicitly removed from scope.
9. `state.json`, `plan.md`, and evidence agree.
10. Escapes are closed.
11. Review Critical findings are fixed.
12. Warnings are fixed or recorded as accepted with risk.
13. Actual slice diffs stayed inside `Write Scope` and outside `Do Not Touch`.
14. Spec contains no private implementation details.
15. Archived or removed scenario IDs were not reused incorrectly.

## Modes

- Default: Critical blocks close; warnings may be accepted with risk.
- `check only`: generate verification report only; do not sync specs or archive.
- `strict`: Critical and Warning block close.
- `archive ready changes`: close multiple changes that have no blocking findings.

## Exit Semantics

Use these meanings in `verify-report.json`:

- `0`: passed.
- `1`: Critical issues exist.
- `2`: strict mode warnings exist.
- `3`: structural error such as missing artifacts, corrupt state, or write-boundary violation.

## Close Process

1. Read all artifacts and evidence.
2. Run the verification checklist.
3. Run fresh relevant tests before making completion claims.
4. Create or update `verify-report.md`.
5. For L2+ or CI-facing use, create `verify-report.json`.
6. If `check only`, stop after reports.
7. If blocked, report issues and do not archive.
8. Sync `spec.md` delta into `flow/specs/*.md`.
9. Move the change to `flow/changes/archive/YYYY-MM-DD-<change>/`.
10. Set archived state in `state.json` before or during archive.

## Output Rules

- Do not require one unique test per scenario.
- Do not require every unit test to bind a scenario ID.
- Do require credible evidence for major external behavior.
- Do not claim completion without fresh verification evidence.


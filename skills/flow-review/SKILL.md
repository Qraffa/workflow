---
name: flow-review
description: Review SDD + TDD flow artifacts, slices, or whole changes for contract quality, slice quality, evidence, and implementation consistency. Use when the user invokes /flow:review or asks to review a brief, spec, plan, slice, or change.
---

# Flow Review

Use this skill to find issues early. Review supports, but does not replace, `flow-close`.

## Inputs

- Review target: `brief`, `spec`, `plan`, `slice`, or `change`.
- Related artifacts and code diff.
- Test, evidence, and review records.

## Review Focus

`brief`:

- Problem, goal, scope, non-goals, constraints, and change type are clear.
- Domain language is precise and does not conflict with `CONTEXT.md`.
- Open questions are visible.

`spec`:

- Requirements describe external behavior.
- Scenarios are independently verifiable and have stable IDs.
- External contracts are complete enough for implementation.
- Internal implementation details are not in the spec.

`plan`:

- Slices are vertical behavior increments.
- Dependencies and parallel groups are correct.
- `Write Scope`, `Do Not Touch`, and shared resources are explicit.
- Test strategy is proportional to behavior and risk.
- Oversized slices are flagged.

`slice`:

- Implementation satisfies its scenarios.
- TDD evidence exists.
- Spec compliance review precedes code quality review.
- Actual diff stays inside allowed boundaries.
- Tests and unverified items are recorded.

`change`:

- Spec/Test/Code semantics align.
- Key scenarios have credible evidence.
- External contracts, errors, data, permissions, security, and performance commitments match implementation.
- Escapes and critical review findings are closed.

## Severity

- Critical: must be fixed before close.
- Warning: should be fixed or explicitly accepted with risk.
- Suggestion: optional improvement.

Warnings accepted with risk must be recorded in `state.json`.

## Process

1. Read the target artifacts and relevant diff.
2. Build a short checklist from the target focus.
3. Report findings by severity, with file references when possible.
4. Recommend concrete fixes.
5. If a finding changes external behavior or contract, route to `flow-escape`.
6. Update `state.json` review records when requested or when operating inside an active change.

## Output Format

Start with findings, ordered by severity. Then list open questions or assumptions. Keep summary secondary.


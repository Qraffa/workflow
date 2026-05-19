---
name: flow-escape
description: Route implementation feedback from TDD back into the SDD artifacts. Use when the user invokes /flow:escape or implementation reveals a spec error, missing scenario, better external interface, or scope overflow.
---

# Flow Escape

Use this skill when continuing implementation would let tests or code silently rewrite external behavior. Escape is a normal feedback path, not a failure.

## Inputs

- Current change and slice.
- Current failing test, implementation evidence, or review finding.
- Relevant scenario, requirement, spec, and plan sections.
- Escape tag:
  - `[escape:spec-error]`
  - `[escape:scenario-missing]`
  - `[escape:better-interface]`
  - `[escape:scope-overflow]`

## Process

1. Pause the current TDD loop.
2. Record the current slice state and evidence.
3. Classify the escape tag.
4. Append to `escapes.log` when present or create it for L1+ changes.
5. Update `state.json` with an opened escape and change status `escaping`.
6. Make the smallest necessary update:
   - `brief.md` for scope, goal, constraints, or change type.
   - `spec.md` for requirements, scenarios, or external contracts.
   - `plan.md` for slices, dependencies, write boundaries, or test strategy.
7. Do not rewrite unrelated parts of the change.
8. Update test strategy if evidence expectations changed.
9. Mark the escape closed with a resolution.
10. Restore change status to `ready` or `implementing` and resume `flow-apply`.

## Mini-Spec Update Rules

- If the spec was wrong, fix the spec first, then tests and code.
- If the test was wrong, fix the test and do not pollute the spec.
- If the code was wrong, fix the code with test evidence.
- If external behavior is affected, pause implementation until the spec decision is explicit.

## Governance

- More than three escapes in one change: review `spec.md` and `plan.md`.
- More than half of slices escaping: return to `flow-clarify`.
- Repeated same-tag escapes: flag a systemic issue in domain language, scenario granularity, or slicing.

## Output Rules

- Preserve append-only escape history.
- Do not use escape as permission for unrelated refactoring.
- Do not mark the slice done until the escape is closed and evidence is updated.


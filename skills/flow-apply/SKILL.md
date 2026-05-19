---
name: flow-apply
description: Implement SDD + TDD behavior slices with Red-Green-Refactor, review, and evidence. Use when the user invokes /flow:apply, wants to start or continue implementation, resume a slice, run AFK slices, or delegate independent slices.
---

# Flow Apply

Use this skill to implement `plan.md` slices while preserving Spec/Test/Code semantic consistency.

## Inputs

- `brief.md`, `spec.md`, `plan.md`, `state.json`
- Current code, tests, and build commands
- User intent such as next slice, resume current slice, work HITL slice, or delegate independent slices

## Selection

1. Select the change ID from the user, current context, or active change folders.
2. Read all flow artifacts before coding.
3. If `plan.md` is missing, offer to create a minimal single-slice plan first.
4. If `spec.md` is missing, allow only lightweight TDD and mark that the change cannot sync into main specs until spec and plan are created.
5. Prefer resuming an in-progress or blocked slice before starting a new one.
6. Select the next unblocked slice by dependencies, risk, and business value.

## Per-Slice TDD Loop

For each selected slice:

1. Claim the slice in `state.json`.
2. Confirm scenarios, write scope, do-not-touch scope, and test strategy.
3. Red: write one failing test for the current behavior or rule.
4. Run the relevant test and confirm it fails for the expected reason.
5. Green: implement the minimum code to pass.
6. Run relevant tests and confirm pass output.
7. Refactor only under green tests.
8. Repeat until slice behavior is complete.
9. Run higher-level tests at the slice boundary.
10. Perform spec compliance review before code quality review.
11. Check actual diff against `Write Scope`, `Do Not Touch`, and parallel guards.
12. Record evidence and update `plan.md` checkboxes plus `state.json`.

## Delegation

Delegate only when the user explicitly asks for parallel, AFK, or subagent execution and the environment permits it. Each delegated slice must receive:

- Change ID
- Slice ID and full slice text
- Scenario IDs and external contract
- Write scope and do-not-touch scope
- Test strategy and TDD requirement
- Expected evidence
- Escape rules
- User-confirmed decisions

AFK slices may be delegated. HITL slices stay in the main session unless the human decision is already recorded and remaining work is mechanical. If HITL becomes AFK, set `originally_hitl: true`.

Controller must verify subagent diffs and tests before marking a slice done.

## Escape Triggers

Stop and use `flow-escape` if a scenario is wrong or missing, an external contract should change, a bug fix changes external behavior, or implementation scope exceeds the planned slice.

## Evidence

At minimum record:

- failing test evidence or reason Red-first was not possible;
- passing test commands;
- scenario IDs;
- diff boundary check;
- spec review result;
- quality review result;
- unverified items and reasons.

L1+ evidence goes under `flow/changes/<change>/evidence/<slice-id>/`.


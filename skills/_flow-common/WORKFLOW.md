# Shared SDD + TDD Flow Rules

## Core Model

SDD owns business intent, external contracts, key constraints, and delivery boundaries. TDD owns small-step implementation, design feedback, and regression protection inside a behavior slice. Scenario is the bridge between Spec and Test. Test is executable evidence for the Spec, but not the default source of external contract truth.

## Artifact Roles

- `brief.md`: problem, goal, scope, non-goals, constraints, domain language, change type, open questions.
- `spec.md`: external behavior delta, requirements, scenario IDs, contracts, acceptance evidence requirements.
- `plan.md`: vertical behavior slices, dependencies, write boundaries, test strategy, parallelization rules.
- `state.json`: machine state, slice ownership, evidence index, escape/review status.
- `escapes.log`: append-only L1+ audit trail for escape events.
- `evidence/`: L1+ reports and verification evidence per slice.
- `verify-report.md` and `verify-report.json`: final verification reports.

Markdown artifacts are the human fact source. `state.json` is a machine index. Update markdown truth first, then sync `state.json`.

## Scenario Rules

Use stable scenario IDs for important external behavior, for example `auth.login.success`.

A valid scenario:

- describes one independently verifiable external behavior;
- uses Given / When / Then semantics in domain language;
- maps `When` to a clear user action, API call, event, or trigger;
- states externally observable results in `Then`;
- avoids private classes, internal algorithms, cache choices, and mock strategy unless they are external constraints.

Key acceptance, integration, and contract tests should reference scenario IDs with an ecosystem-appropriate form such as `@scenario auth.login.success`, `scenario_auth_login_success`, or `Scenario: auth.login.success`. Ordinary unit tests do not need scenario IDs.

Archived scenario IDs are stable semantic identifiers. Do not reuse removed IDs. If meaning changes incompatibly, create a new ID or record migration.

## Slice Rules

A slice is a vertical behavior increment, not a horizontal technical layer. It must have:

- one or more scenario IDs;
- observable result;
- independent verification path;
- dependencies;
- `Execution Mode: AFK` or `Execution Mode: HITL`;
- `Write Scope`;
- `Do Not Touch`;
- test strategy;
- expected evidence.

AFK means enough context exists for independent implementation. HITL means human judgment, product tradeoff, external confirmation, or high-risk operation is still needed.

## TDD Rules

Inside each slice, use:

```text
Red -> Green -> Refactor -> Review -> Evidence
```

Rules:

- One behavior or rule per cycle.
- Write one failing test for the current behavior before implementation.
- Confirm the test fails for the right reason.
- Implement the minimum code to pass.
- Refactor only when relevant tests are green.
- Prefer public interfaces and stable contracts.
- Do not batch-write all tests before all implementation.
- Do not add speculative future behavior.
- Do not let tests silently redefine external contracts.

## Escape Rules

Trigger escape when implementation reveals:

- `[escape:spec-error]`: scenario semantics are wrong, ambiguous, or contradictory.
- `[escape:scenario-missing]`: an external boundary or business rule is missing.
- `[escape:better-interface]`: external interface or contract should change.
- `[escape:scope-overflow]`: slice cost or scope is larger than expected.

Escape process:

1. Pause the TDD loop.
2. Record the current slice, evidence, tag, and reason.
3. Make the smallest relevant update to `brief.md`, `spec.md`, or `plan.md`.
4. Update the test strategy.
5. Mark the escape resolved.
6. Resume `flow-apply`.

More than three escapes in one change should trigger spec/plan review. If more than half the slices escape, return to `flow-clarify`.

## Review Severity

- Critical: must be fixed before close.
- Warning: should be fixed or explicitly accepted with risk in `state.json`.
- Suggestion: optional improvement.

In strict close mode, warnings are blocking.


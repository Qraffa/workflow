---
name: flow-spec
description: Define the SDD external behavior contract for an SDD + TDD change. Use when the user invokes /flow:spec, wants to turn a brief into scenarios and contract deltas, or needs to update spec after an implementation escape.
---

# Flow Spec

Use this skill to create or update `flow/changes/<change>/spec.md`. The spec describes external behavior and contract deltas, not private implementation design.

## Inputs

- `flow/changes/<change>/brief.md`.
- Main specs in `flow/specs/`.
- Relevant code, tests, API schemas, event schemas, and prior changes.
- User-confirmed boundaries from clarification.

## Process

1. Select the change ID. If no brief exists, create a minimal brief draft from conversation context and mark assumptions.
2. Read the brief, existing main specs, relevant code, and tests.
3. Identify capabilities affected by the change.
4. Write ADDED, MODIFIED, and REMOVED requirements.
5. For each important external behavior, define a stable scenario ID.
6. Use Given / When / Then semantics for every scenario.
7. Record external contracts:
   - API or SDK surface
   - data model or schema
   - events and messages
   - error semantics
   - permissions and security
   - compatibility and migration behavior
   - performance or audit commitments
8. State required evidence types without prewriting every test.
9. Update `state.json` scenario indexes when present.
10. Set change status to `specified`.

## Scenario Quality Gate

Every scenario must:

- cover one independent, externally observable behavior;
- use domain language;
- have a clear trigger;
- have externally visible outcomes;
- avoid internal classes, private functions, caches, queues, and mock strategies unless externally contractual.

## Fact Source Rules

- Spec owns business intent, external behavior, and stable contracts.
- Tests can expose missing spec details, but do not silently create new external contracts.
- If existing tests conflict with the intended contract, surface the conflict and resolve the business intent before writing the spec.

## Output Rules

- Do not write private class diagrams or directory structure.
- Do not make unit test checklists.
- Do not encode exploratory guesses as stable contracts.
- If the contract is too unclear, stop and return to `flow-clarify`.

## Template

Use `../_flow-common/TEMPLATES.md#specmd` as the artifact shape.


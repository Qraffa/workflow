---
name: flow-plan
description: Split an SDD + TDD spec into vertical behavior slices with TDD strategy, dependencies, write scopes, and state. Use when the user invokes /flow:plan or wants to prepare a specified change for implementation.
---

# Flow Plan

Use this skill to create or update `flow/changes/<change>/plan.md` and `state.json` before implementation.

## Inputs

- `brief.md`
- `spec.md`
- Existing code and test structure
- Build and test commands
- Relevant ADRs and domain language

## Process

1. Read the brief, spec, existing tests, and implementation surface.
2. Split by vertical behavior increments, not technical layers.
3. Map each slice to one or more scenario IDs.
4. Define dependencies and parallel groups.
5. Mark each slice as `AFK` or `HITL`.
6. Define:
   - `Write Scope`
   - `Do Not Touch`
   - `Shared Resource`
   - estimated TDD cycles
   - test strategy by level
   - first Red test entry point
   - expected evidence
7. Add default parallel guards for migrations, global config, public schemas, shared contracts, CI configuration, and main specs unless the slice exists to change them.
8. Mark slices over roughly 10 TDD cycles as candidates for splitting.
9. Define escape policy and thresholds.
10. Initialize or update `state.json` slice state.
11. Set change status to `ready`.

## Slice Requirements

Each slice must be independently verifiable and must produce externally observable behavior. Avoid slices like "add database table", "wire service", or "build UI" unless the slice includes the full behavior path and verification.

## Parallelization Rules

Slices can be parallel only when:

- no dependency path exists between them;
- write scopes do not overlap;
- neither slice touches the other's `Do Not Touch`;
- no parallel guard is hit;
- they do not modify the same external contract or requirement;
- they do not share a test-polluting resource.

When uncertain, make slices serial.

## Output Rules

- Do not write a line-by-line coding script.
- Do not prewrite all tests.
- Do not place internal implementation details in `spec.md`; put temporary implementation notes in the plan only when they affect execution risk.

## Template

Use `../_flow-common/TEMPLATES.md#planmd` and `../_flow-common/TEMPLATES.md#statejson`.


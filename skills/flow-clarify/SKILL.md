---
name: flow-clarify
description: Clarify an SDD + TDD change and create the brief artifact. Use when the user invokes /flow:clarify, starts a new feature or behavior change, reports a complex bug needing scope clarification, or wants to decide whether the full flow is appropriate.
---

# Flow Clarify

Use this skill to clarify intent before external contract definition or implementation. It creates or updates `flow/changes/<change>/brief.md` and initializes lightweight state.

## Inputs

- User request, bug report, or change idea.
- Existing repo docs such as `CONTEXT.md`, `CONTEXT-MAP.md`, `docs/adr/`, prior `flow/specs/`, and active `flow/changes/`.
- Relevant code and tests when they can answer scope or behavior questions.

## Process

1. Select or derive a kebab-case change ID.
2. Explore existing docs, specs, tests, and code before asking the user questions.
3. Ask only questions that cannot be answered from the repo and that affect scope, contract, or risk.
4. Clarify:
   - Problem
   - Goal
   - Scope
   - Non-Goals
   - Domain Language
   - Constraints
   - Change Type
   - Open Questions
5. Decide whether the change should use the full flow, a lightweight flow, or plain TDD/refactor.
6. Create or update `flow/changes/<change>/brief.md`.
7. Create or update `flow/changes/<change>/state.json` with status `clarifying` or `draft`.
8. Optionally update `CONTEXT.md` only for stable domain language.
9. Suggest ADRs only for decisions that are hard to reverse, surprising without context, and based on a real tradeoff.

## Change Type Guidance

- New feature: full flow.
- External API, event, data model, error semantics, permissions, security, or compatibility change: full flow.
- Behavior bug that changes external behavior: lightweight clarify, then spec missing scenarios.
- Bug that violates an existing spec: plan/apply can start directly with TDD.
- Internal refactor: usually no spec; refactor under tests.
- P0 hotfix: allow post-hoc spec, but require test evidence and `post_hoc_spec_pending: true`.
- Prototype: do not archive into main specs.

## Output Rules

- Do not create formal scenarios in `brief.md`.
- Do not split implementation tasks.
- Do not start coding.
- Mark assumptions that still need confirmation.
- If the user skipped clarify and started at `flow-spec`, generate a minimal brief from context and mark unconfirmed assumptions.

## Template

Use `../_flow-common/TEMPLATES.md#briefmd` as the artifact shape.


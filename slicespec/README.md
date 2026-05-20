# SliceSpec

> SDD + TDD as a six-skill pipeline. Contracts up front, tracer-bullet
> slices, subagent-driven TDD, audited escape hatch, single-pass verify.

SliceSpec is a family of six Claude Code skills plus shared reference
docs. Together they take a change from a vague idea to a verified,
archived, contract-aligned delivery without losing the audit trail.

## At a glance

| Stage | Skill | Purpose | Primary output |
|---|---|---|---|
| 1 | `slicespec-clarify` | Scope the change. Problem, goal, non-goals, domain language. | `changes/<id>/brief.md` |
| 2 | `slicespec-spec` | Define the external contract. Requirements + Scenarios with stable IDs. | `changes/<id>/spec.md` |
| 3 | `slicespec-slice` | Break the spec into tracer-bullet vertical slices. Declare write_scope, test_strategy. | `changes/<id>/slices.md` |
| 4 | `slicespec-implement` | TDD per slice. Subagent for AFK, main session for HITL. Two-stage review + mechanical diff check. | git commits + `changes/<id>/evidence/<sid>/` |
| 5 | `slicespec-escape` | Audited mid-flight contract change. Pauses slice, mini-spec-update, resumes. | append to `changes/<id>/escapes.log` |
| 6 | `slicespec-verify` | V1-V10 + tests, then sync + archive. | `changes/<id>/verify-report.{md,json}` + archive move |

`shared/` carries cross-skill rules:

- `directory-layout.md` — file tree and ID conventions
- `state-schema.md` — `state.json` contract (machine state)
- `governance-thresholds.md` — escape and review thresholds
- `scenario-id-rules.md` — Scenario ID format, lifecycle, reserved set
- `change-type-routing.md` — which stages each change type takes

## Designed against

- **Contract drift** — Spec, Test, and Code growing out of sync because
  no one closed the loop. /verify V1-V10 catch this.
- **Scope creep inside slices** — subagents writing files they
  shouldn't. /implement's controller diff check catches this
  mechanically, not via reviewer judgement.
- **Silent ID reuse** — archived Scenario IDs reappearing in new
  changes with different semantics, breaking grep-based traceability.
  /verify V10 catches this.
- **Untracked escape decisions** — implementation discoveries quietly
  edited into the spec without an audit trail. /escape's append-only
  log catches this.
- **Horizontal-slice planning** — "first do all the schema, then all
  the API". /slice forces tracer-bullet verticals.
- **TDD-skipping** — production code without a failing test. /implement
  refuses this; the implementer prompt makes it the iron rule.

## Installation

These skills are designed to be copied into a Claude Code skills
directory (project- or user-level). The layout:

```
.claude/skills/
├── slicespec-clarify/
│   ├── SKILL.md
│   └── brief-template.md
├── slicespec-spec/
│   ├── SKILL.md
│   └── spec-template.md
├── slicespec-slice/
│   ├── SKILL.md
│   ├── slices-template.md
│   └── parallel-check.md
├── slicespec-implement/
│   ├── SKILL.md
│   ├── implementer-prompt.md
│   ├── spec-reviewer-prompt.md
│   ├── quality-reviewer-prompt.md
│   └── controller-diff-check.md
├── slicespec-escape/
│   ├── SKILL.md
│   └── escapes-log-template.md
├── slicespec-verify/
│   ├── SKILL.md
│   ├── verify-checklist.md
│   └── verify-report-template.md
└── shared/
    ├── directory-layout.md
    ├── state-schema.md
    ├── governance-thresholds.md
    ├── scenario-id-rules.md
    └── change-type-routing.md
```

Skill descriptions match what Claude Code expects: each `SKILL.md`
has `name:` and `description:` frontmatter. **Every stage is
explicit-invocation only.** A skill runs when, and only when, the user
types its slash command (`/slicespec-clarify`, `/slicespec-spec`,
`/slicespec-slice`, `/slicespec-implement`, `/slicespec-escape`,
`/slicespec-verify`). Claude must not auto-trigger any stage from
keyword inference — even when the conversation obviously fits.
If a fitting situation arises, Claude should suggest the relevant
command and wait for the user to invoke it.

## Reading order

Read the six `SKILL.md` files in order:

1. `slicespec-clarify/SKILL.md`
2. `slicespec-spec/SKILL.md`
3. `slicespec-slice/SKILL.md`
4. `slicespec-implement/SKILL.md`
5. `slicespec-escape/SKILL.md`
6. `slicespec-verify/SKILL.md`

Read `shared/*.md` whenever a skill references them — they're terse
and load-bearing.

## What SliceSpec is NOT

- **A new methodology.** It is a packaging of SDD + TDD + subagent
  parallelism, optimised for one specific environment (Claude Code).
- **A test generator.** Tests are still written by humans or
  implementers. SliceSpec only enforces that they exist and reference
  Scenarios.
- **A CI replacement.** `/verify` returns an exit code that CI can
  gate on, but CI is still where the test suite runs at full scope.

## Conventions in this repo

- Every skill is self-contained and addressable via Claude Code's
  skill discovery. The `name:` and `description:` frontmatter is what
  Claude reads when picking a skill.
- `shared/*.md` is referenced from skills by relative path. Treat
  these as load-bearing — they encode rules every skill relies on.
- Markdown is the source of truth. `state.json` is derived; if the two
  disagree, rebuild `state.json` from markdown.

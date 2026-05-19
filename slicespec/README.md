# SliceSpec

> SDD + TDD as a six-skill pipeline. Contracts up front, tracer-bullet
> slices, subagent-driven TDD, audited escape hatch, single-pass verify.

SliceSpec implements the design described in `final-workflow.md` and
operationalised in `unified-workflow.md`. It is a family of six Claude
Code skills plus shared reference docs.

## At a glance

| Stage | Skill | Purpose | Primary output |
|---|---|---|---|
| 1 | `slicespec-clarify` | Scope the change. Problem, goal, non-goals, domain language. | `changes/<id>/brief.md` |
| 2 | `slicespec-spec` | Define the external contract. Requirements + Scenarios with stable IDs. | `changes/<id>/spec.md` |
| 3 | `slicespec-slice` | Break the spec into tracer-bullet vertical slices. Declare write_scope, test_strategy. | `changes/<id>/slices.md` |
| 4 | `slicespec-implement` | TDD per slice. Subagent for AFK, main session for HITL. Two-stage review + mechanical diff check. | git commits + `changes/<id>/evidence/<sid>/` |
| 5 | `slicespec-escape` | Audited mid-flight contract change. Pauses slice, mini-spec-update, resumes. | append to `changes/<id>/escapes.log` |
| 6 | `slicespec-verify` | V1-V10 + V-appendix + tests, then sync + archive. | `changes/<id>/verify-report.{md,json}` + archive move |

`shared/` carries cross-skill rules:

- `directory-layout.md` — file tree and ID conventions
- `state-schema.md` — `state.json` contract (machine state)
- `governance-thresholds.md` — escape and review thresholds
- `scenario-id-rules.md` — Scenario ID format, lifecycle, reserved set
- `change-type-routing.md` — which stages each change type takes

`MAPPING-unified-workflow.md` is the section-by-section crosswalk back
to `unified-workflow.md`.

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
has `name:` and `description:` frontmatter. Each skill is invoked by
the user typing `/slicespec-clarify`, `/slicespec-spec`, etc., or by
Claude detecting the trigger keywords in the description.

## L0 → L3 maturity ladder

Per `unified-workflow.md` §5.1:

- **L0** — Pilot. Use `/slicespec-clarify` + `/slicespec-implement` only.
  No spec, no slices required. Archives skip the spec sync.
- **L1** — Stable use. brief.md and state.json required. `/spec` and
  `/slice` come in. Pre-PR `/verify` runs as a sanity check.
- **L2** — Production. Full pipeline, escape hatch, strict-mode
  `/verify` in CI.
- **L3** — Compliance. Add traceability graphs, semantic LLM checks,
  external-call audits. /verify's plug-in surface, not changes to the
  six skills.

Move up the ladder when the team is comfortable, not before. SliceSpec
explicitly avoids forcing heavy governance on day one.

## Reading order

Start with `MAPPING-unified-workflow.md` for the conceptual overview.
Then read the six `SKILL.md` files in order:

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
- **A heavy framework.** L0 mode is two commands and zero required
  files. L1+ adds discipline; L3 is opt-in.
- **A test generator.** Tests are still written by humans or
  implementers. SliceSpec only enforces that they exist and reference
  Scenarios.
- **A CI replacement.** `/verify` returns an exit code that CI can
  gate on, but CI is still where the test suite runs at full scope.

## License & origin

SliceSpec is an internal scaffold. Concepts and patterns are absorbed
from:

- OpenSpec (`openspec`) — delta spec syntax, requirement-granularity
  merge.
- matt-skills — Socratic clarification, HITL/AFK slice typing,
  CONTEXT.md domain-language pattern.
- superpowers — subagent dispatch, two-stage review, plan-as-memory.

All three were studied in `unified-workflow.md` and reduced to the six
skills here.

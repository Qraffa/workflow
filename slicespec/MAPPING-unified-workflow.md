# SliceSpec ↔ unified-workflow.md Crosswalk

Every requirement in `unified-workflow.md` maps to one or more files in
this skill set. This is the audit document for "did the implementation
capture the design?"

## §1 Design principles

| Principle | Where implemented |
|---|---|
| #1 Concision first (6 commands, 4 docs, 1 state file) | `README.md` summary + 6 SKILL.md + 4 templates (brief, spec, slices, escapes) + `shared/state-schema.md` |
| #2 Contract / implementation separation | `slicespec-spec/SKILL.md` "Allowed and forbidden in spec.md" + `slicespec-slice/SKILL.md` write_scope rules |
| #3 Soft dependencies | Every SKILL.md "Soft dependencies" section |
| #4 Subagent parallelism | `slicespec-implement/SKILL.md` Mode A + `slicespec-slice/parallel-check.md` |
| #5 Escape Hatch first-class | `slicespec-escape/SKILL.md` (entire) |
| #6 Natural-language intent, no CLI flags | `slicespec-verify/SKILL.md` Modes table (intent → mode) |
| #7 Static scope isolation | `slicespec-slice/SKILL.md` write_scope + `slicespec-implement/controller-diff-check.md` |
| #8 Stable Scenario IDs after archive | `shared/scenario-id-rules.md` + `slicespec-spec/SKILL.md` ID allocation step |
| #9 State as derived fact | `shared/state-schema.md` "Rebuilding state.json from markdown" |

## §2 Command roster

| §2.x | unified-workflow command | SliceSpec skill |
|---|---|---|
| §2.1 | `/clarify` | `slicespec-clarify/SKILL.md` |
| §2.2 | `/spec` | `slicespec-spec/SKILL.md` |
| §2.3 | `/slice` | `slicespec-slice/SKILL.md` |
| §2.4 | `/implement` | `slicespec-implement/SKILL.md` + four bundled prompts |
| §2.5 | `/escape` | `slicespec-escape/SKILL.md` |
| §2.6 | `/verify` | `slicespec-verify/SKILL.md` + checklist + report template |

## §3 State machine and artifact flow

| §3.x | Topic | Where |
|---|---|---|
| §3.1 Change state machine | `shared/state-schema.md` `status` field + per-skill transitions |
| §3.2 Slice state machine | `shared/state-schema.md` `slices[].status` + `phase` + `slicespec-implement/SKILL.md` step transitions |
| §3.3 state.json shape | `shared/state-schema.md` (entire) |
| §3.4 File lifecycle | `shared/directory-layout.md` |

## §4 Comparison with original workflows

| §4.x | Topic | Where |
|---|---|---|
| §4.1 Command-by-command table | This document (above) + `README.md` "Designed against" |
| §4.2 Absorbed strengths | `README.md` "License & origin" |
| §4.3 Removed features | Each SKILL.md notes its consolidations (e.g. spec=proposal+design merged; verify=verify+sync+archive merged) |
| §4.4 Net new | `/escape` skill (entire); V9 and V10 in `slicespec-verify/verify-checklist.md`; controller-diff-check as mechanical gate |

## §5 Routing

| §5.x | Topic | Where |
|---|---|---|
| §5.2 Change-type routing | `shared/change-type-routing.md` |

> §5.1 (L0–L3 maturity ladder) is intentionally omitted from this
> skill set. SliceSpec ships only the fully-formed pipeline; there is
> no opt-in lite mode. If a project is not ready for the full
> discipline, it should adopt the workflow when it is.

## §6 Implementation engineering

| §6.x | Topic | Where |
|---|---|---|
| §6.1 Directory convention | `shared/directory-layout.md` |
| §6.2 Tech stack | `README.md` Installation |
| §6.3 Claude Code integration | Each SKILL.md uses Claude Code primitives directly (Agent tool, Bash, file ops). `slicespec-implement` documents the Agent-tool dispatch contract. |

## §7 Risks and boundaries

| §7.x | Topic | Where |
|---|---|---|
| §7.1 Fits | `README.md` "Designed against" |
| §7.2 Doesn't fit | `slicespec-clarify/SKILL.md` "Do not use this skill when" + `README.md` What SliceSpec is NOT |
| §7.3 Known risks (table) | Each risk row covered: worktree conflicts (`parallel-check.md`); subagent cost (`slicespec-implement/SKILL.md` model selection); ID drift (V10); /escape abuse (`shared/governance-thresholds.md`); CONTEXT.md disagreement (`slicespec-clarify/SKILL.md` step 1); state.json drift (`shared/state-schema.md`); ID reuse (`shared/scenario-id-rules.md`); write_scope escape (controller-diff-check + /escape); destructive Scenario rewrite (Supersedes pattern in spec-template.md) |
| §7.4 Exit signals | `shared/governance-thresholds.md` aggregate counters surfaced via /verify |

## §8 One-liner summary

`unified-workflow.md` ends with:

> SliceSpec = OpenSpec contract model + matt-skills slice discipline +
> superpowers parallel execution + final-workflow.md semantic boundary
> arbitration. 6 commands, 3 core files, zero forced config.

This skill set delivers exactly that. The 6 commands are the 6
top-level skills. The 3 core human-readable files are
brief.md / spec.md / slices.md. state.json is the machine derivative;
escapes.log and evidence/ exist for audit but are not "design"
artefacts.

## What is intentionally NOT in this skill set

These are unified-workflow.md decisions that don't manifest as code:

- **L0 → L3 maturity ladder.** SliceSpec ships only the fully-formed
  pipeline. There is no pilot mode, no opt-in compliance tier, no
  hotfix-skip-spec fallback. The six commands are the workflow.
- **Bundled LLM checkers** — V1 and V3 mention LLM scanning as a
  helpful technique inside `/verify`, but the skill does not ship one.
  Projects supply their own if they want it.
- **Specific test runners** — `slicespec-verify` step 5 enumerates
  detection heuristics but does not hard-code one runner. Project-
  specific.
- **PR-creation idioms** — `slicespec-verify` step 9 uses `gh` as the
  default example; non-GitHub projects swap in their own.
- **Telemetry** — unified-workflow.md does not require telemetry.
  state.json + the archive directory are the audit surface.

These omissions are deliberate. The six skills cover the design's
required behaviour; extensions live alongside, not inside.

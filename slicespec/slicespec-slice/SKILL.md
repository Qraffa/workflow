---
name: slicespec-slice
description: Stage 3 of SliceSpec. Use after slicespec-spec. Breaks the spec into tracer-bullet vertical slices with HITL/AFK type, write_scope, do_not_touch, blocked_by, and test_strategy. Replaces to-issues, opsx tasks generation, writing-plans.
---

# SliceSpec — Slice

Decompose the spec into vertical slices small enough to fit a few TDD
cycles each, big enough to represent a behavioural increment. Each
slice is an independently deliverable tracer bullet.

**Core principle:** A horizontal slice ("first do all the schema work,
then all the API work") gives no behavioural feedback. A vertical
slice cuts through all layers for one behaviour and returns a complete
demo.

**Announce at start:** "I'm using slicespec-slice to break the spec
into tracer-bullet slices."

## When to Use

Trigger this skill when:

- `spec.md` exists and the user wants to start implementing.
- The user wants to re-slice after `/escape` substantially changed the
  spec.
- The user explicitly asks to plan implementation.

Do **not** use this skill when:

- The change is an internal refactor with no spec — go straight to
  `/implement`.
- The spec is incomplete (still has `[blocking-spec]` open questions).

## Inputs

- `changes/<change-id>/spec.md`
- `changes/<change-id>/brief.md` (for change-type and Constraints
  context)
- `changes/<change-id>/state.json`
- `shared/directory-layout.md` for `do_not_touch` defaults

## Process

```
┌────────────────────────────────────┐
│ 1. Read spec.md, brief.md          │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 2. Identify capabilities & paths   │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 3. Draft tracer-bullet slices      │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 4. Allocate write_scope / dnt      │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 5. Run parallel-conflict check     │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 6. Show user, iterate              │
└────────────────┬───────────────────┘
                 ▼
┌────────────────────────────────────┐
│ 7. Write slices.md + state.json    │
└────────────────────────────────────┘
```

### Step 1 — Read inputs

Read spec.md fully (do not summarise). Read brief.md "Constraints" and
"Decision Log". Identify the change-type — it changes how you slice
(e.g. a contract change must include a contract-test slice).

### Step 2 — Identify capabilities and code paths

For each capability in spec.md, list candidate code paths that already
exist (`grep` / `find` if needed). Note which directories are "owned"
by which capability — these become candidate write_scope globs.

Also note shared resources that any slice might touch: `migrations/`,
shared schemas, root config files. These become `parallel_guards`.

### Step 3 — Draft tracer-bullet slices

Per `final-workflow.md` §6, prioritise:

1. **High business value** — the happy-path scenarios.
2. **High risk / high failure cost** — security, money, irreversibility.
3. **External contract** — API, event, data-model scenarios.
4. **Regression-prone paths** — areas where past bugs concentrate.
5. **Foundation dependencies** — only as late as possible. Defer scaffolding
   until a real behaviour needs it.

A slice must satisfy all six rules:

- Cuts vertically through every layer touched (schema → API → UI if
  applicable).
- Demoable on its own.
- Covers one or more Scenario IDs explicitly.
- Has a clear test strategy (acceptance / integration / contract /
  unit, chosen per scenario per `final-workflow.md` §7).
- Lists files it will create or modify (the eventual write_scope).
- Estimated to finish in fewer than ~10 TDD cycles.

If a slice's `estimated_cycles ≥ 10`, **force a split dialogue**.
Present two or three smaller slices and let the user choose to accept
the split or override (recording the override reason in slices.md).
This is non-negotiable per
`shared/governance-thresholds.md`.

### Step 4 — Allocate `write_scope` and `do_not_touch`

For each slice:

**write_scope (whitelist):**

- Always glob form (`src/auth/**`, `tests/auth/**`).
- Prefer directory-level globs to enumerating files when possible.
- Two parallel slices must have disjoint write_scopes (the
  parallel-conflict check enforces this).

**do_not_touch (blacklist):**

- Seeded from `shared/directory-layout.md`. Always include:
  - `specs/**`
  - `changes/**` (except the slice's own evidence/ directory)
  - `.github/**`
  - `CONTEXT.md`
  - `docs/adr/**`
  - Lock files (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, etc.)
- Add cross-capability paths: a slice in `src/auth/` should never touch
  `src/billing/`.

If a slice genuinely needs to touch one of these defaults (e.g. a
migration), the user must approve removing it from `do_not_touch` for
that slice. The controller in `/implement` still enforces the per-slice
list at diff time.

### Step 5 — Parallel-conflict check

Run `parallel-check.md` against every pair of slices in the candidate
set. If pair `(A, B)` is parallel-incompatible, mark them
`blocked_by`-related or just leave them serialised in the dep graph.

The check is mechanical:

1. `blocked_by` paths overlap — already serial, no parallelism needed.
2. `write_scope(A) ∩ write_scope(B) ≠ ∅` — block parallelism.
3. `do_not_touch(A) ∩ write_scope(B) ≠ ∅` — block parallelism.
4. Either slice touches a `parallel_guards` path — block parallelism.
5. Slice declares `requires_shared_resource` — block parallelism.

When in doubt, default to serial. Parallelism is a performance
optimisation, not a correctness goal.

### Step 6 — Show user, iterate

Present slices using the format in `slices-template.md`:

- Numbered list with title, type, blocked_by, covers, write_scope,
  test_strategy, estimated_cycles.
- A textual dependency graph.

Ask:

- "Is the granularity right?"
- "Any slice you'd merge or split?"
- "Are HITL/AFK markings correct?" (AFK preferred where possible)
- "Are blocked_by edges correct?"
- "Any missing scenario coverage?"

For any slice ≥ 10 cycles, present the forced split now.

Iterate until user approves.

### Step 7 — Write slices.md, update state.json

Write `changes/<change-id>/slices.md` using `slices-template.md`.

Update state.json:

```json
{
  "status": "implementing",
  "parallel_guards": [...],
  "slices": {
    "<change-id>-s01": {
      "status": "pending",
      "phase": null,
      "type": "AFK",
      "owner": null,
      "scenarios": ["auth.login.001"],
      "write_scope": ["src/auth/**", "tests/auth/**"],
      "do_not_touch": ["specs/**", "src/billing/**"],
      "blocked_by": [],
      "estimated_cycles": 3,
      "evidence": {}
    }
  },
  "updated_at": "<ISO-8601 UTC>"
}
```

Announce:

> Slices written. <M> slices, <K> AFK, <N> HITL. Next: run
> `slicespec-implement` to start the first AFK slice (or
> `slicespec-implement <slice-id>` for a specific one).

## Slice anatomy

Every slice in slices.md must include:

- `id` — `<change-id>-s<NN>` per `shared/directory-layout.md`.
- `type` — `AFK` (can be dispatched to a subagent) or `HITL` (must run
  in the main session because it involves a user-judgement step).
- `covers` — list of Scenario IDs.
- `blocked_by` — list of slice IDs (or `[]`).
- `write_scope` — list of globs.
- `do_not_touch` — list of globs.
- `test_strategy` — bullet list mapping each Scenario or sub-behaviour
  to a test layer.
- `estimated_cycles` — integer.
- `status` — `pending` initially (see `shared/state-schema.md` for the
  full state machine).
- Optional `requires_shared_resource: <name>` when the slice needs an
  exclusive non-code resource (e.g. shared DB).

## Test strategy guide

From `final-workflow.md` §7. Use it when assigning layers.

| Spec content | Recommended layer |
|---|---|
| Primary acceptance path | Acceptance / Integration |
| Cross-module business flow | Integration |
| External API / SDK / event / data contract | Contract |
| Core business rule, state machine, money math, permission | Unit + Integration |
| Error handling and edge cases | Unit or Integration (by reach) |
| Security / performance / compatibility constraint | Dedicated automated check or audit evidence |
| Internal helper logic | Unit when warranted |

Avoid: "all unit tests" or "smoke test only". Force the discussion.

## HITL vs AFK heuristics

| Signal | Type |
|---|---|
| User must pick between two architecturally valid designs | HITL |
| Concrete code change with clear acceptance | AFK |
| New external contract whose semantics the user has not approved | HITL |
| Routine database migration | AFK |
| UX/copy choice with no analytic basis | HITL |
| Bug fix with reproducer + scenario | AFK |

Default to AFK when the slice is bounded and the spec is unambiguous.
Prefer HITL when the slice's success depends on judgement that the spec
does not encode.

## Anti-patterns

| Symptom | Fix |
|---|---|
| Each slice is a layer (schema, then API, then UI). | Re-slice vertically. Combine layers per scenario. |
| Slice "covers" 5 scenarios. | Split. One slice covers ~1-2 scenarios. |
| Slice has empty test_strategy. | Block. Write the strategy or split until you can. |
| Slice has estimated_cycles=15 and you accepted "keep as one". | Refuse. Force split. The threshold is non-negotiable. |
| Two slices share `src/auth/login.py` in their write_scope. | One must drop it or both must serialise. Parallelism is off. |
| write_scope is `**/*` (everything). | Refuse. Force the user to name directories. |
| do_not_touch is empty. | Re-add defaults from `shared/directory-layout.md`. |

## Templates

- `slices-template.md` — file layout.
- `parallel-check.md` — the 5-step conflict-detection algorithm.

## Related skills

- `slicespec-spec` — produces the spec this skill consumes.
- `slicespec-implement` — consumes this skill's output.
- `slicespec-escape` — may rewrite this skill's output mid-flight.

## Mapping to unified-workflow.md

This skill implements §2.3 of unified-workflow.md. See
`../MAPPING-unified-workflow.md` for the full crosswalk.

---
name: slicespec-spec
description: Stage 2 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-spec`. Do NOT auto-trigger from keywords like "spec", "requirements", or "scenarios" — this skill is user-gated. Defines external behaviour via Requirements + Given/When/Then Scenarios with stable IDs.
---

# SliceSpec — Spec

Translate the brief into a structured spec.md with stable Scenario IDs.
spec.md is the **only** document that can change the external contract.
Internal structure and library choices live elsewhere.

**Core principle:** Spec says *what* and *for whom*. Slice says *how to
break it apart*. Code shows *how to satisfy it*. Test proves *that it
works*. Confusing these layers is what makes Spec/Test/Code drift.

**Announce at start:** "I'm using slicespec-spec to define the external
contract."

## Inputs

- `changes/<change-id>/brief.md`.
- `specs/<capability>/spec.md` (if the capability exists — used to
  generate delta blocks).

## Process

```
┌──────────────────────────────────────┐
│ 1. Read brief.md (required)          │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 2. Detect greenfield vs brownfield   │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 3. Allocate scenario IDs             │ ← check reserved set
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 4. Draft Requirements + Scenarios    │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 5. Show user, iterate                │
└────────────────┬─────────────────────┘
                 ▼
┌──────────────────────────────────────┐
│ 6. Write spec.md + update state.json │
│ 7. Hand off to /slice                │
└──────────────────────────────────────┘
```

### Step 1 — Read brief

Read `brief.md`. If it is missing, stop and ask the user to run
`slicespec-clarify` first — a spec without a brief has no anchor for
"why this change exists". Do not attempt to invent the brief from
context.

### Step 2 — Greenfield vs brownfield

Check whether `specs/<capability>/spec.md` exists for the capability the
change touches.

- **Greenfield** (no existing spec): the new spec.md will contain
  `## ADDED Requirements` only.
- **Brownfield** (existing spec): spec.md will contain delta blocks —
  `## ADDED`, `## MODIFIED`, `## REMOVED`, `## RENAMED`. Read the
  existing spec first; you must not silently rewrite existing
  requirements.

If the change spans multiple capabilities, write **one** spec.md per
capability, all under `changes/<change-id>/spec.md` (single file, multiple
top-level Capability sections). Do not split changes across multiple
change directories — they share state.json.

### Step 3 — Allocate scenario IDs

Per `shared/scenario-id-rules.md`:

- Format: `<capability>.<requirement-slug>.<seq>`.
- Build the **reserved set** by scanning `specs/**/*.md` for all existing
  IDs and `changes/archive/**/spec.md` for REMOVED / superseded IDs.
- For each new scenario allocate the next free `<seq>` per
  `<capability>.<requirement-slug>` namespace.
- For modified scenarios with **semantic-preserving** changes (clearer
  wording, additional precondition that does not change observable
  behaviour), keep the original ID.
- For **semantically destructive** changes (new behaviour that breaks
  the old observable contract), allocate a new ID under `## MODIFIED
  Requirements` and mark `**Supersedes**: <old-id>`. Add the old ID to
  the change's REMOVED list to retire its tests.

If any proposed ID intersects the reserved set, refuse and reallocate.

### Step 4 — Draft Requirements + Scenarios

Use `spec-template.md`. Each Requirement is a single English statement
("The system SHALL …") that maps to one or more Scenarios. Scenarios are
the testable units.

Required Scenario format (exact heading depth matters; `/verify` parses
this):

```markdown
### Requirement: <name>

<The system SHALL …>

#### Scenario: <name>
<!-- id: <capability>.<requirement-slug>.<seq> -->

- **GIVEN** <preconditions>
- **WHEN** <trigger>
- **THEN** <observable outcome>
- **AND** <additional observable outcomes, optional>
```

Rules:

- One Scenario describes **one** independently verifiable behaviour.
  Multiple behaviours = multiple Scenarios. Do not bundle.
- WHEN must be a single action, event, or API call. No "and then" chains.
- THEN must be externally observable — return value, state change, event
  emitted, side-effect, error response.
- Do not write "user experience is improved" or anything else
  unverifiable.
- Do not write Redis, private class names, file paths, or internal
  algorithms unless those are themselves external contracts (`/verify`
  V8 will flag these — see `shared/scenario-id-rules.md`).

spec.md must contain at least these four top-level sections per
capability:

1. **Why** — copy/condense from brief.md's Problem and Goal.
2. **Non-Goals** — copy from brief.md.
3. **Requirements** — the meat of the spec. Use delta blocks for
   brownfield.
4. **Constraints** — performance, compliance, security, migration. Any
   constraint with external visibility lives here.

Optional appendix sections (only if relevant):

- **External Contracts** — required for change-type
  `external-api-or-data-contract`. Document REST/gRPC/event-bus
  contracts, error codes, schemas.
- **Design Rationale** — when a non-trivial design choice has business
  motivation (e.g. legal-driven). Keep brief; deep design notes belong
  in `docs/adr/`.

### Step 5 — Show user, iterate

Present the draft spec. Ask:

- "Are these scenarios complete?" — push for missing failure paths.
- "Is anything in here an internal detail that doesn't belong in spec?" —
  point to candidates if you see them.
- "Any scenario you'd want to delete or merge?"

Iterate until the user confirms. **You do not need 100% confidence on
all scenarios** — Scenarios that are still uncertain can be added later
via `/escape`. But the major contract must be settled.

### Step 6 — Write spec.md, update state.json

Write `changes/<change-id>/spec.md` using the template. Update
state.json:

```json
{
  "status": "specified",
  "scenarios": [
    {
      "id": "auth.login.001",
      "capability": "auth",
      "status": "active",
      "supersedes": null,
      "first_seen_change": "<change-id>"
    }
  ],
  "updated_at": "<ISO-8601 UTC>"
}
```

If the change includes `## REMOVED` scenarios, mark their existing
entries (loaded from `specs/<capability>/spec.md` history) as
`status: "removed"`. If `## MODIFIED` introduces a new ID with
`Supersedes`, mark the old ID `status: "superseded"`.

### Step 7 — Hand off

Announce:

> Spec written to `changes/<change-id>/spec.md`. <N> scenarios allocated.
> Status: `specified`. Next: run `slicespec-slice` to break it into
> tracer-bullet slices.

Do not invoke `/slice` automatically.

## Allowed and forbidden in spec.md

| Allowed | Forbidden |
|---|---|
| Business goals, user-visible behaviour | Private class/function names |
| External APIs, events, data models | File paths, directory structure |
| Error codes, error semantics, edge cases | Library choices unless they are external contracts |
| Performance thresholds, audit requirements | Cache layer details, internal queue names |
| Security and compliance constraints | TDD test fixtures, mock strategies |
| Compatibility / migration constraints | Speculative future capabilities |

`/verify` V8 will surface forbidden content as Warnings, not Blocks
(false positives exist) — but `/spec` should refuse to write them in the
first place.

## Brownfield delta semantics

The delta blocks are first-class. They must round-trip through
`/verify` sync without ambiguity.

```markdown
## ADDED Requirements

### Requirement: Account lockout after repeated failures
...

## MODIFIED Requirements

### Requirement: Password validation
**Supersedes**: auth.password.003
<new requirement body>

## REMOVED Requirements

### Requirement: Captcha after one failure
**Reason**: Replaced by progressive lockout (see ADDED).
**Migration**: Existing tests for `auth.captcha.*` can be deleted after sync.

## RENAMED Requirements

- `Old name` → `New name` (semantic preserved, scenarios unchanged)
```

Notes:

- A `## MODIFIED` block with no `**Supersedes**` is treated as a
  semantic-preserving rewrite — original IDs are kept.
- A `## REMOVED` block must include `**Reason**` and `**Migration**`.
- `## RENAMED` is purely cosmetic. Scenario IDs do not change.

## Anti-patterns

| Symptom | Fix |
|---|---|
| You wrote `Scenario` with three `#` headings. | Must be four `#`. `/verify` will fail. |
| You used `Given/When/Then` without bold markup. | `**GIVEN**`, `**WHEN**`, `**THEN**`. The parser expects this. |
| You reused an archived Scenario ID. | Re-allocate. Reserved set check failed. |
| You merged two behaviours into one Scenario with "and". | Split. One observable outcome per Scenario. |
| You included `class UserService` or `src/auth/login.py` in the spec. | Move to slice's write_scope. Spec is external-only. |
| You wrote `## ADDED Requirements` on a brownfield change without touching the existing spec.md. | Re-detect. Look in `specs/<capability>/spec.md` — there is an existing capability, generate delta blocks. |

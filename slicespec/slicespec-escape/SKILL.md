---
name: slicespec-escape
description: Stage 5 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-escape`. Do NOT auto-trigger from keywords like "escape", "the spec is wrong", or "we need to change the contract" — this skill is user-gated. Pauses the in-flight slice, runs a mini-spec-update, resumes.
---

# SliceSpec — Escape Hatch

The escape hatch is the **only** sanctioned way to change the contract
mid-implementation. It exists because TDD reliably surfaces gaps the
spec did not anticipate — Scenarios that turn out to be ambiguous,
behaviours that surface during refactor, costs that explode beyond
what the slice estimated.

The escape hatch is not a redo button. It is a paused, audited,
mini-spec-update.

**Core principle:** Without `/slicespec-escape`, implementers silently
shape behaviour through tests. The spec drifts. Next quarter, nobody
knows which document is true. `/slicespec-escape` forces the decision
back to the contract layer where it belongs.

**Announce at start:** "I'm using slicespec-escape to revise the
contract for slice <slice-id>."

## When to Use

**Invocation rule:** Explicit-only. Runs when the user types
`/slicespec-escape`. Never auto-trigger mid-`/slicespec-implement` even
if a Scenario turns out wrong — surface the situation to the user and
wait for them to invoke the command. The whole point of the escape
hatch is that the human signs off on every contract change.

**Decline (and redirect) when:**

- The bug is purely internal, no contract change → stay in
  `/slicespec-implement`.
- The user wants an unrelated feature → that's a new change,
  `/slicespec-clarify`.
- The fix needs no spec edit → `/slicespec-implement` is enough.

## Inputs

- `changes/<change-id>/slices.md` (the slice in flight)
- `changes/<change-id>/spec.md`
- `changes/<change-id>/state.json`
- The user's description of what they discovered.
- An escape tag (one of four below).

## Escape tags

| Tag | When |
|---|---|
| `spec-error` | A Scenario in spec.md is wrong, ambiguous, or contradicts itself. |
| `scenario-missing` | A real external boundary is not covered by any Scenario yet. |
| `better-interface` | Refactor or implementation reveals a more natural external interface. |
| `scope-overflow` | Cost / scope explosion. Slice needs to be split, narrowed, or re-approached. |

Multiple tags are allowed — pick the primary one and list secondaries
in the description.

## Process

```
┌───────────────────────────────────┐
│ 1. Pause in-flight subagent       │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 2. Append to escapes.log          │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 3. Mark slice escaped(open)       │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 4. Mini-spec-update               │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 5. Update slices.md if needed     │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 6. Close the escape               │
│ 7. Hand back to /implement        │
└───────────────────────────────────┘
```

### Step 1 — Pause

Tell the in-flight implementer subagent (Mode A) to exit `BLOCKED`,
preserving any in-progress test files and commits. For HITL slices
(Mode B), simply stop the current TDD cycle in place. Do NOT discard
the partial work — it informs the mini-spec-update.

### Step 2 — Append to escapes.log

`changes/<change-id>/escapes.log` is APPEND-ONLY. Add a new entry at
the end using `escapes-log-template.md`:

```yaml
---
id: e<NN>             # sequential per change
ts: 2026-05-19T10:23:00Z
slice: <slice-id>
tag: <one of the four>
description: |
  <Multi-line free text. State the problem, what the slice was trying
  to do, what surfaced, what changed when the implementer saw it.>
resolution_pending: yes
---
```

`escapes.log` entries persist forever, even after the change is
archived. They are the audit trail for "why did the contract change?"

Update state.json:

```json
{
  "escapes": [
    ...,
    {
      "id": "e<NN>",
      "ts": "<ISO-8601 UTC>",
      "slice": "<slice-id>",
      "tag": "<tag>",
      "description": "<first line of log entry>",
      "resolved": false,
      "resolved_at": null
    }
  ],
  "updated_at": "<ISO-8601 UTC>"
}
```

### Step 3 — Mark the slice escaped(open)

In slices.md, change the slice's status to `escaped`. In state.json:

```json
{
  "slices": {
    "<slice-id>": {
      "status": "escaped",
      "phase": null,
      "owner": null,
      ...
    }
  }
}
```

The slice will remain `escaped` until the mini-spec-update is committed
and the user is ready to resume. Then it goes back to `pending` (or to
`in_progress` if the user re-dispatches immediately).

### Step 4 — Mini-spec-update

This is the heart of the escape hatch. **You may only modify the
Scenarios, Requirements, Constraints, or External Contracts directly
related to the escape.** Do not take the opportunity to rewrite
unrelated sections — that's a new change.

For each tag:

#### `spec-error`

- Identify the offending Scenario(s) by ID.
- Decide: edit in place (semantic-preserving clarification — keep ID)
  or supersede with a new ID (semantic change — under `## MODIFIED`
  with `**Supersedes**: <old-id>` per `shared/scenario-id-rules.md`).
- Update spec.md.

#### `scenario-missing`

- Add a new Scenario under `## ADDED Requirements` (greenfield case) or
  `## MODIFIED Requirements` (extending an existing requirement).
- Allocate a fresh Scenario ID respecting the reserved set.

#### `better-interface`

- This usually means a `## MODIFIED Requirements` block with
  `**Supersedes**` because the external contract is changing.
- Document the new contract precisely. Examples: new error codes,
  changed response shape, different idempotency semantics.

#### `scope-overflow`

- Edit slices.md, not spec.md (usually). Choose one of:
  - Split the slice into 2-3 smaller slices with new IDs.
  - Narrow the slice's `covers` list (move some Scenarios to a new
    slice).
  - Change `write_scope` / `do_not_touch` (must be explicit and
    user-approved — never silently widen).
- If the overflow is actually about contract complexity (not slice
  size), promote to one of the other three tags and update spec.md.

In all cases, do this with the user. The escape hatch is interactive
— not a place to auto-rewrite the spec.

### Step 5 — Update slices.md if needed

If the mini-spec-update added/removed Scenarios, update the affected
slices' `covers` lists and `test_strategy` rows.

If the mini-spec-update widened `write_scope`, edit the slice's
`write_scope` in slices.md. The controller will re-check on next
dispatch.

If the escape created new slices (split or follow-up), allocate fresh
slice IDs (next sequence) and add them to slices.md with `status:
pending`.

### Step 6 — Close the escape

Once spec.md and slices.md reflect the new reality:

- Edit the escapes.log entry: change `resolution_pending: yes` to
  `resolution_pending: no` and add `resolved_ts: <ISO-8601 UTC>`.
- In state.json, set the matching escape's `resolved: true` and
  `resolved_at`.
- Reset the affected slice(s) to `pending` (or `in_progress.red` if the
  user is resuming immediately).

### Step 7 — Hand back

Announce:

> Escape <id> closed. Spec/slices updated. Slice <slice-id> reset to
> `pending`. Run `slicespec-implement <slice-id>` to resume.

Do not auto-resume. The user picks the next step.

## Governance thresholds

Per `shared/governance-thresholds.md`:

- `escapes_count > 3` in a single change → `/verify` will issue a
  Warning.
- `escapes_count > slices_total / 2` → `/verify` will Block. The
  recommended remediation is to step back to `/clarify` — the
  fundamental understanding was wrong, not the spec wording.

`/escape` itself does not enforce these thresholds; it only writes the
log. `/verify` reads it and decides.

## What /escape will NOT do

| Anti-pattern | Why we forbid it |
|---|---|
| Silently delete an escape entry from escapes.log | Audit trail must be append-only. |
| Modify a Requirement that has nothing to do with the in-flight slice | That's a new change. Use a fresh `/clarify` → `/spec`. |
| Widen `write_scope` without recording it as the escape's resolution | The next diff check would lie. The widening must be auditable. |
| Reuse a Scenario ID that was previously REMOVED or superseded | `shared/scenario-id-rules.md` reserves these forever. |
| Mark `resolved: true` without actually editing spec.md or slices.md | The escape is open until the document agrees. |

## Templates

- `escapes-log-template.md` — full entry format.

## Related skills

- `slicespec-implement` — invokes this skill when a Red→Green→Refactor
  cycle surfaces a contract problem.
- `slicespec-spec` — the mini-spec-update happens inside the same
  spec.md this skill produced.
- `slicespec-verify` — counts escapes and enforces governance.


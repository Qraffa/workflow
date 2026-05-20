---
name: slicespec-clarify
description: Stage 1 of SliceSpec. ONLY invoke when the user explicitly types `/slicespec-clarify`. Do NOT auto-trigger from keywords like "clarify", "scope", or "new feature" — this skill is user-gated. Scopes the problem, non-goals, and domain language for a new change before writing a spec.
---

# SliceSpec — Clarify

Explore the problem with the user one question at a time until the goal,
scope, non-goals, and domain language are clear enough to write a spec.

**Core principle:** A vague brief produces a vague spec produces drifting
tests. Spend the time here, not in `/verify`.

**Announce at start:** "I'm using slicespec-clarify to scope the change
before we touch any spec."

## When to Use

**Invocation rule:** Explicit-only. Runs when the user types
`/slicespec-clarify`. Never auto-trigger from keyword inference; if the
context fits, suggest the command and wait for the user to invoke it.

**Decline (and redirect) when:**

- The user has a fully-specified change and just wants to write the spec
  → `/slicespec-spec`.
- The task is bug triage with no behaviour change → `/slicespec-implement`.

## Inputs

- User's natural-language description (free-form).
- Optional: paths to existing code, issue links, prior brief.md.
- Optional: project's `CONTEXT.md` (read it first if present).

## Process

```
┌──────────────────────────────┐
│ 1. Read CONTEXT.md (if any)  │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 2. Identify change type      │ ← consult shared/change-type-routing.md
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 3. Socratic loop, one Q/A    │ ← list possible answers and propose your
│    at a time                 │   recommendation with every question
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 4. Detect saturation         │ ← explicit user "ready to spec" OR
│                              │   your judgment that context is sufficient
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 5. Write/update brief.md     │
│ 6. Project state.json fields │
│ 7. Hand off to /spec         │
└──────────────────────────────┘
```

### Step 1 — Read CONTEXT.md

If `CONTEXT.md` exists at the project root (or `CONTEXT-MAP.md` for
multi-context repos), read it before asking your first question. Use its
vocabulary in every subsequent question. When the user introduces a term
that conflicts with CONTEXT.md, surface the conflict immediately:

> Your glossary defines `Order` as a paid commitment. You used `Order` to
> mean a checkout-in-progress. Are these two concepts, or is the glossary
> stale?

Never silently rewrite their term in your own words.

### Step 2 — Identify change type

Consult `shared/change-type-routing.md` and infer the route. Tell the user:

> Based on what you said this looks like a **bug fix that changes
> externally observable behaviour**. Per the routing table that means
> clarify → spec (add missing scenario) → slice → implement → verify.
> Does that match your intent?

Record the answer (and any override) in brief.md's Decision Log.

### Step 3 — Socratic loop

Ask **one** question at a time. With every question:

1. When the question has more than one plausible answer, list the
   candidate options so the user can see the space.
2. Provide your recommended answer with a short reason.

Walk depth-first down the decision tree. Resolve dependencies before
branching. Examples of good first questions:

- "Whose behaviour are we changing here — end-user, internal caller, or
  ops console?"
- "What is the failure mode this change prevents — incorrect output,
  outage, compliance violation?"
- "What is the smallest scope that still solves the user's problem?"

Examples of bad questions:

- Multi-part ("What does X do, and how does Y interact, and...?") — split.
- Yes/no without a recommended answer — propose one.
- Implementation-detail ("Which library should we use?") — defer to spec
  unless the choice is itself the external contract.

When the user mentions a vague or overloaded term, sharpen it on the spot:

> You said `account`. Do you mean the `Customer` (billing) or the `User`
> (auth)? CONTEXT.md treats them as different concepts.

When the user states a fact that disagrees with the code, surface the
contradiction by reading the relevant code:

> The code at `src/auth/login.py` currently rejects empty passwords
> upstream. You said empty passwords were accepted. Which is the truth?

### Step 4 — Detect saturation

Stop the loop when **either**:

- The user explicitly says "ready to spec" (or equivalent).
- You judge that the current context is clear enough to write the spec —
  Problem, Scope, Non-Goals, Constraints, and Domain Language all have
  concrete answers and no `[blocking-spec]` open questions remain.

Do not stop on the first few easy answers; saturation requires breadth
across Problem, Scope, Non-Goals, Constraints. Do not count rounds — a
fast-converging conversation is fine; a slow one is also fine.

### Step 5 — Write brief.md

Use `brief-template.md`. Fill every section. Specifically:

- **Problem**: business-perspective sentences only. No implementation.
- **Goal**: an observable outcome, not "ship the feature".
- **Scope**: bullet list of capabilities included.
- **Non-Goals**: bullet list of capabilities deliberately excluded.
- **Domain Language**: every domain term the user used during the loop,
  with a one-line definition. Mark conflicts with CONTEXT.md inline.
- **Constraints**: performance, compliance, migration, external
  dependencies.
- **Open Questions**: anything still unresolved. Mark as either
  `[blocking-spec]` or `[deferred-to-impl]`.
- **Decision Log**: every confirmed choice with date and brief reason.

If brief.md already exists (re-clarification), append a new Decision Log
entry rather than overwriting the prior one. Keep the audit trail.

### Step 6 — Project to state.json

Write or update `changes/<change-id>/state.json`:

```json
{
  "change_id": "<kebab-case>",
  "status": "draft",
  "created_at": "<ISO-8601 UTC>",
  "updated_at": "<ISO-8601 UTC>"
}
```

If state.json already exists, only bump `updated_at`.

### Step 7 — Hand off

After saving brief.md, announce:

> Brief saved to `changes/<change-id>/brief.md`. Status: `draft`.
> Next step: run `slicespec-spec` to turn this into a contract.

Do not invoke `/spec` yourself. The user decides when they're ready.

## Soft dependencies

- `CONTEXT.md`: read if present, do not require.
- `docs/adr/`: read if relevant to the change area, do not require.
- Existing brief.md from a prior `/clarify`: respect it; append rather
  than overwrite.

## Anti-patterns

| Symptom | Fix |
|---|---|
| You ask three questions in one message. | Split. One question, one recommendation. |
| You wrote brief.md while Problem/Scope/Non-Goals are still vague. | Erase and restart. Saturation means concrete answers, not just answered questions. |
| Your brief.md mentions specific libraries, classes, file paths. | Move those to the future spec.md or delete them. brief.md is business-level. |
| You contradict CONTEXT.md without surfacing the conflict. | Re-read CONTEXT.md. Make the user resolve the tension. |
| You write Open Questions and call yourself done. | Tag each Open Question `[blocking-spec]` or `[deferred-to-impl]`. Resolve all `[blocking-spec]` before exiting. |

## Templates

- `brief-template.md` — the 8-section brief template.

## Related skills

- `slicespec-spec` — next step.
- `slicespec-escape` — when implementation finds you missed something
  here, /escape kicks back to a mini-clarify.


# Governance Thresholds

These thresholds are enforced by `/verify`. Earlier commands (`/clarify`,
`/escape`) surface them as informational warnings; only `/verify` blocks.

## Escape Hatch counters

Counted per change. The denominator for ratios is the slice count in
slices.md at the moment `/verify` runs (not historical).

| Metric | Default mode | Strict mode |
|---|---|---|
| `escapes_count > 3` | Warning | Warning |
| `escapes_count > slices_total / 2` | Block | Block |
| Any escape with `resolution_pending: yes` | Block | Block |

When the second threshold trips, `/verify` exits with code 1 and prints a
recommendation to restart from `/clarify`.

## Slice granularity

| Metric | When `/slice` enforces | Action |
|---|---|---|
| `estimated_cycles >= 10` | Always | Force split-confirmation dialog. User must explicitly accept "keep as one slice" with a reason recorded in slices.md. |
| Slice covering > 3 scenarios | Always | Warning. Split is recommended but not forced. |
| Slice with empty `test_strategy` | Always | Block. Cannot mark `pending`. |

## Review severity

Code-quality findings come from `/slicespec-review` and are graded by
`/slicespec-verify` (the review itself only reports). The severity
behaviour at verify time:

| Severity | Default mode | Strict mode | `accepted_with_risk` allowed? |
|---|---|---|---|
| Critical | Block | Block | No |
| Warning | Pass with note | Block | Yes (Default only) |
| Info | Pass | Pass | N/A |

`accepted_with_risk` requires a human-authored justification in
`state.json.quality_review.warnings_accepted_with_risk[]` — never set
by the reviewer subagent on its own.

## Controller diff check

| Outcome | Slice transition | When |
|---|---|---|
| `passed` | Slice may move to `done` | Diff against `write_scope` succeeded, `do_not_touch` not touched |
| `failed:<files>` | Slice transitions to `blocked(scope-violation)` | Any file outside `write_scope` or inside `do_not_touch` |

A `failed` outcome **never** moves to `done` on its own. The only recovery
paths:

1. `/escape` with tag `better-interface` or `scope-overflow` updates
   `slices.md` write_scope, then re-attempt the slice.
2. User manually reverts out-of-scope changes, then re-attempts.

Scope is judged only by the mechanical diff check, never by a reviewer.

## Spec / Code drift counters

| Metric | Default mode | Strict mode |
|---|---|---|
| Any `active` scenario without test reference | Block | Block |
| Any spec text matching implementation-detail patterns (e.g. file path, class name) | Warning | Warning |
| Any `## ADDED` requirement reusing a reserved scenario ID | Block | Block |

The implementation-detail pattern list is in `shared/scenario-id-rules.md`.

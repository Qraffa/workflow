# Change Type Routing

Different change types take different paths through the six commands. The
routing decision is recorded in `brief.md` ("Decision Log") on the first call
to `/clarify`, and later commands consult it before deciding to skip.

The user can override the route at any command invocation with a natural-
language statement ("this is a hotfix, skip /spec"). The skill records the
override in the Decision Log of brief.md.

## Routes

| Change type | Path | Notes |
|---|---|---|
| New feature / important business behaviour change | clarify → spec → slice → implement → verify | Full pipeline. |
| External API / data contract change | clarify → spec → slice → implement → verify | Spec must include an `External Contracts` section. `/slice` must produce at least one contract-test row. |
| Bug fix that changes externally observable behaviour | clarify (light) → spec (add missing scenario) → slice → implement → verify | Spec must gain a new requirement or scenario covering the bug. |
| Bug fix that only restores conformance with existing spec | implement → verify | No spec change. `/clarify` may still be invoked for context but is optional. |
| Internal refactor that preserves external behaviour | implement (refactor-only mode) → verify | `slices.md` optional. `/verify` skips V1/V2 if no spec change. |
| Performance optimisation with external commitment | clarify → spec (write commitment) → slice → implement → verify | Benchmark output goes into evidence/. |
| Performance optimisation purely internal | implement → verify (benchmark) | No spec change. |
| P0 hotfix | implement → verify (light) → backfill brief & spec | `state.json.post_hoc_spec_pending = true` until backfilled. Strict mode of `/verify` blocks archive while pending. |
| Prototype / POC / one-off script | implement (L0) | Lives in change directory; never sync'd to specs/. Archive uses `-l0` suffix. |

## Recording the route

In brief.md "Decision Log":

```markdown
- 2026-05-19: change-type = `feature-with-external-contract`. Reason: introduces a
  new public REST endpoint. Will require contract test.
- 2026-05-19: route confirmed: clarify → spec → slice → implement → verify.
```

In state.json, the route is reflected by the sequence of commands actually
invoked plus the `post_hoc_spec_pending` flag for hotfixes. There is no
explicit `route` field — derived facts only.

## Override examples

| User says | Skill does |
|---|---|
| "this is just a refactor" | `/spec` early-exits with a banner; `/slice` is skipped if user agrees; `/verify` runs in refactor mode |
| "P0 — skip spec for now" | `/clarify` minimal pass to capture root cause, `/spec` skipped, state.json `post_hoc_spec_pending=true`, `/verify` warns on archive |
| "this is a one-off script, don't bother with specs" | All commands besides `/implement` short-circuit; archive suffixed `-l0` |
| "treat this as new feature" | Default route, no overrides |

The skill should always **confirm** the route override out loud before
proceeding ("Understood, running in refactor-only mode — `/spec` will be
skipped.") and append the confirmation to the Decision Log.

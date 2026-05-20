# Change Type Routing

Different change types take different paths through the six commands. The
routing decision is recorded in `brief.md` ("Decision Log") on the first call
to `/clarify`, and later commands consult it before deciding to skip.

The user can override the route at any command invocation with a natural-
language statement. The skill records the override in the Decision Log of
brief.md.

## Routes

| Change type | Path | Notes |
|---|---|---|
| New feature / important business behaviour change | clarify → spec → slice → implement → verify | Full pipeline. |
| External API / data contract change | clarify → spec → slice → implement → verify | Spec must include an `External Contracts` section. `/slice` must produce at least one contract-test row. |
| Bug fix that changes externally observable behaviour | clarify (light) → spec (add missing scenario) → slice → implement → verify | Spec must gain a new requirement or scenario covering the bug. |
| Bug fix that only restores conformance with existing spec | clarify (light) → slice → implement → verify | No spec change. `/clarify` captures the symptom and the conforming behaviour; `/spec` is skipped because the contract is already correct. |
| Internal refactor that preserves external behaviour | clarify (light) → slice → implement (refactor-only mode) → verify | `/verify` skips V1/V2 because no spec change occurred. `slices.md` is still required so write_scope stays mechanical. |
| Performance optimisation with external commitment | clarify → spec (write commitment) → slice → implement → verify | Benchmark output goes into evidence/. |
| Performance optimisation purely internal | clarify (light) → slice → implement → verify (benchmark) | No spec change. |

Every route ends at `/verify`. No route bypasses brief, slices, or
verify — those three are non-negotiable. Only `/spec` can be skipped,
and only when the change provably does not alter the external contract.

## Recording the route

In brief.md "Decision Log":

```markdown
- 2026-05-19: change-type = `feature-with-external-contract`. Reason: introduces a
  new public REST endpoint. Will require contract test.
- 2026-05-19: route confirmed: clarify → spec → slice → implement → verify.
```

In state.json, the route is reflected by the sequence of commands actually
invoked. There is no explicit `route` field — derived facts only.

## Override examples

| User says | Skill does |
|---|---|
| "this is just a refactor" | `/clarify` captures the refactor intent; `/spec` is skipped with banner; `/slice` proceeds; `/verify` runs in refactor mode (V1/V2 skipped). |
| "treat this as new feature" | Default route, no overrides. |
| "skip /spec, the contract isn't changing" | Allowed only when `/clarify` confirms no externally observable behaviour change. Otherwise refuse and run the full pipeline. |

The skill should always **confirm** the route override out loud before
proceeding ("Understood, running in refactor-only mode — `/spec` will be
skipped.") and append the confirmation to the Decision Log.

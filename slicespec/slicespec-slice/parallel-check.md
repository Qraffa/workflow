# Parallel Conflict Check

Determines whether two slices can be implemented in parallel. The
controller in `slicespec-implement` runs this same check before
dispatching any parallel subagents. `/slice` uses it to suggest
parallel-friendly slicing.

## Inputs

- Slice `A` and slice `B` from slices.md.
- `parallel_guards[]` from slices.md (project-wide).
- `requires_shared_resource` declarations on each slice.

## Algorithm (5 steps, short-circuit)

Any step returning **block** means the pair is NOT parallel-safe.
Continue to the next step only if the current step passes.

### Step 1 — Dependency check

```
If A is in B.blocked_by (transitively) or vice versa → ALREADY SERIAL.
```

These slices are sequentially ordered by the dep graph. Parallel does
not apply.

### Step 2 — write_scope intersection

```
For each glob in A.write_scope, expand to file set Fa.
For each glob in B.write_scope, expand to file set Fb.
If Fa ∩ Fb ≠ ∅ → BLOCK.
```

File-level expansion is required because directory-level globs can
conflict on shared files. Example:

- A: `src/auth/**`
- B: `src/auth/login.py`

These two globs overlap. Block.

### Step 3 — do_not_touch / write_scope cross-check

```
If do_not_touch(A) ∩ write_scope(B) ≠ ∅ → BLOCK.
If do_not_touch(B) ∩ write_scope(A) ≠ ∅ → BLOCK.
```

If A declares a file in its do_not_touch list, and B intends to write
that file, they cannot run in parallel even if their own write_scopes
do not overlap. The conflict would surface when A is dispatched and B's
write lands in the worktree.

### Step 4 — Parallel guards

```
If A.write_scope ∪ B.write_scope intersects parallel_guards → BLOCK.
```

`parallel_guards` lists high-conflict shared paths — migrations, root
config, public schemas. Any slice touching one of those must run
serial.

Default guard set (seeded by `slicespec-slice` on first run):

- `**/migrations/**`
- `db/migrate/**`
- `**/config/**`
- Root files: `*.toml`, `*.yaml`, `*.yml` at repo root
- `**/types/**`
- `**/*.proto`
- `**/*.graphql`

Projects extend this list in slices.md "Parallel guards (project-wide)".

### Step 5 — Shared resource declaration

```
If A.requires_shared_resource is set and B.requires_shared_resource == A's value → BLOCK.
If either is set and they share an external dependency (DB, broker,
flaky port) per project convention → BLOCK.
```

If a slice declares `requires_shared_resource: postgres-test-db`, two
slices with that declaration can never run in parallel even if their
write_scopes are disjoint.

### Pass

All 5 steps clear → the pair is parallel-safe. The controller may
dispatch them concurrently in separate worktrees (`git worktree`).

## Output

The check produces one of three verdicts:

- **PARALLEL_OK** — dispatch in separate worktrees.
- **SERIAL_REQUIRED** — dispatch sequentially.
- **ALREADY_SERIAL** — dep graph already serialises; no decision needed.

`/slice` records the verdicts implicitly by structuring slices.md so
the controller can decide at dispatch time. There is no `parallel`
metadata in state.json — the check is re-run on demand.

## Worktree mechanics (for /implement)

For each parallel-safe pair, the controller:

1. Creates a worktree per slice: `git worktree add ../<slice-id>
   <branch>`.
2. Dispatches the subagent with the worktree path in the prompt.
3. After both report `DONE`, runs the spec-/quality-reviewer pair on
   each in their own worktrees.
4. Cherry-picks each worktree's commits back into the main worktree
   serially.
5. Resolves any cherry-pick conflicts manually (controller does this
   itself — never a subagent).
6. Removes the worktrees after merge.

If a cherry-pick conflict cannot be resolved without changing semantics,
the controller triggers `/escape` with tag `scope-overflow`.

## Default to serial

If the algorithm output is ambiguous (rare — typically when globs are
expressed against files that do not yet exist), default to serial. The
cost of an unnecessary serial run is small. The cost of an undetected
parallel conflict is high.

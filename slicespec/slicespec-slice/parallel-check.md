# Parallel Conflict Check

Determines whether two slices can be implemented in parallel. This is
pure static analysis over the slices' declared scopes — it does not
depend on how the slices are executed.

`/slice` uses it to suggest parallel-friendly slicing. External
orchestration (multiple agents or worktrees each running
`/slicespec-implement` on a different slice) uses it to confirm two
slices are safe to run at the same time. `/slicespec-implement` itself
always runs one slice at a time in the main session; it does not
dispatch parallel subagents.

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

All 5 steps clear → the pair is parallel-safe. They may be run
concurrently by external orchestration in separate worktrees (`git
worktree`) — see "Parallel execution (external orchestration)" below.

## Output

The check produces one of three verdicts:

- **PARALLEL_OK** — safe to run concurrently in separate worktrees.
- **SERIAL_REQUIRED** — run sequentially.
- **ALREADY_SERIAL** — dep graph already serialises; no decision needed.

`/slice` records the verdicts implicitly by structuring slices.md so an
orchestrator can decide what to run in parallel. There is no `parallel`
metadata in state.json — the check is re-run on demand.

## Parallel execution (external orchestration)

`/slicespec-implement` runs one slice at a time in the main session and
does **not** spawn parallel workers itself. Running several
parallel-safe slices at once is an *external* concern: a human, a
higher-level orchestrator, or several agents each drive their own
`/slicespec-implement` on a different slice. This check is what makes
that safe.

A typical worktree-based setup, for each `PARALLEL_OK` slice:

1. Create a worktree per slice: `git worktree add ../<slice-id>
   <branch>`.
2. In each worktree, run `/slicespec-implement <slice-id>`
   independently. Each runs its own TDD loop and controller diff check
   (the diff check supports a worktree path — see
   `../slicespec-implement/controller-diff-check.md`).
3. After each slice is `done`, cherry-pick its commits back into the
   main worktree serially.
4. Resolve any cherry-pick conflicts manually. If a conflict cannot be
   resolved without changing semantics, trigger `/escape` with tag
   `scope-overflow`.
5. Remove the worktrees after merge.
6. Run `/slicespec-review` and `/slicespec-verify` once over the merged
   change, as usual — they operate on the whole change, not per
   worktree.

The orchestration layer owns this loop; the SliceSpec skills do not
encode it beyond providing this conflict check and the worktree-aware
diff check.

## Default to serial

If the algorithm output is ambiguous (rare — typically when globs are
expressed against files that do not yet exist), default to serial. The
cost of an unnecessary serial run is small. The cost of an undetected
parallel conflict is high.

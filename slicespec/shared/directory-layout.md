# SliceSpec Directory Layout

Every SliceSpec command operates on the following project layout. Skills must read
and write only the files listed here. Anything else is a bug.

## Project tree

```
<project-root>/
├── CONTEXT.md                   # Optional, project-level domain glossary
├── docs/adr/                    # Optional, Architecture Decision Records
├── specs/                       # Long-lived spec repository (post-archive merge target)
│   └── <capability>/
│       └── spec.md
└── changes/                     # In-flight changes
    ├── <change-id>/
    │   ├── brief.md             # /clarify product
    │   ├── spec.md              # /spec product
    │   ├── slices.md            # /slice product
    │   ├── escapes.log          # /escape append-only log
    │   ├── state.json           # Machine state (derived from markdown)
    │   ├── verify-report.json   # /verify machine-readable output
    │   ├── verify-report.md     # /verify human-readable output
    │   └── evidence/
    │       └── <slice-id>/
    │           ├── implementer-report.md
    │           ├── spec-review.md
    │           └── quality-review.md
    └── archive/
        └── YYYY-MM-DD-<change-id>/   # Frozen snapshot of the change directory
```

## Change ID conventions

- Use kebab-case, lower-case, ASCII only. Example: `add-auth-login`.
- The change ID is permanent. Do not rename a change directory once `state.json`
  exists; rename only triggers confusion across commits, escapes, and evidence.

## Slice ID conventions

- Format: `<change-id>-s<NN>`, two-digit zero-padded sequence. Example:
  `add-auth-login-s03`.
- Sequence numbers are append-only. Splitting a slice creates new sequence
  numbers (e.g. s03 splits to s03a, s03b is not allowed — use s07, s08).

## Source of truth precedence

When any two artifacts disagree, prefer in this order:

1. `spec.md` (after `/verify` sync, `specs/<capability>/spec.md`)
2. `slices.md`
3. `brief.md`
4. `escapes.log`
5. `state.json`

`state.json` is a derived index. If it conflicts with markdown, rebuild it from
markdown plus `git log` — never edit markdown to match state.json.

## Files SliceSpec must not touch by default

Add these to every slice's `do_not_touch` list unless the slice explicitly owns
them:

```
specs/**
changes/**
.github/**
CONTEXT.md
docs/adr/**
.gitignore
.envrc
package-lock.json
pnpm-lock.yaml
yarn.lock
poetry.lock
Cargo.lock
```

When a slice legitimately needs to edit one of these, the user must approve the
addition in `/slice` and the file must move out of `do_not_touch` for that slice
only. The `/implement` controller still enforces the per-slice list at diff time.

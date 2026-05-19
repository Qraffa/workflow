# Scenario ID Rules

Scenario IDs are the glue between spec.md (contract) and tests (evidence).
They are part of the public API of a SliceSpec project: stable across
refactors, never silently reused, and always grep-able.

## Format

```
<capability>.<requirement-slug>.<seq>
```

- `<capability>` — lowercase, kebab-case, matches the directory name under
  `specs/<capability>/`. Example: `auth`, `billing-invoice`.
- `<requirement-slug>` — lowercase, kebab-case, matches the requirement
  heading after slugifying. Drop articles. Example: `login`, `partial-refund`.
- `<seq>` — three-digit zero-padded sequence. Example: `001`, `017`.

Examples:

- `auth.login.001`
- `billing-invoice.partial-refund.003`
- `notifications.opt-out.012`

Disallowed:

- Hex hashes (`auth.login.a1b2`)
- Date-encoded sequences (`auth.login.2026-05-19`)
- Capability-only IDs (`auth.001`)

## Lifecycle

```
not-allocated  →  active  ┬→  removed
                          └→  superseded
```

- `active` — Listed under `## ADDED Requirements` or `## MODIFIED Requirements`
  in the change's spec.md. Tests must reference it.
- `removed` — Listed under `## REMOVED Requirements`. Tests may be deleted;
  the ID stays in the reserved set forever.
- `superseded` — Replaced by a new ID under `## MODIFIED Requirements` because
  the semantics changed. Original ID enters the reserved set. New ID gains
  `**Supersedes**: <old-id>` annotation.

## Reserved set

The reserved set is the union of:

- All scenario IDs in `specs/**/*.md` (post-archive).
- All scenario IDs marked `## REMOVED Requirements` in any archived change.
- All scenario IDs marked superseded in any archived change.

`/spec` and `/verify` both consult this set:

- `/spec` refuses to allocate an ID in the reserved set when generating
  `## ADDED Requirements`.
- `/verify` V10 fails if any `## ADDED` ID intersects the reserved set.

The reserved set is rebuilt from filesystem scans; it is not stored anywhere.

## Test references

A scenario reference satisfies V2 if it appears in at least one of the
following forms within a test file:

1. **Inline comment** above the test:
   ```python
   # @scenario: auth.login.001
   def test_user_can_log_in_with_correct_password():
       ...
   ```
2. **Function/method name prefix**:
   ```python
   def test_scenario_auth_login_001_user_can_log_in():
       ...
   ```
   The ID separator can be `_` or `-`; case-insensitive.
3. **Docstring first line**:
   ```python
   def test_user_can_log_in_with_correct_password():
       """Scenario: auth.login.001

       Verifies the happy-path login flow.
       """
   ```

Multiple references in a single test are allowed and count toward each
scenario independently. `/verify` greps the test tree using these three
patterns; nothing else counts.

## Implementation-detail patterns banned in spec.md

`/verify` V8 flags spec.md if any of these patterns appear (case-insensitive)
outside a fenced code block:

- `class \w+`, `def \w+\(`, `function \w+\(`
- `src/`, `lib/`, `app/`, `pkg/` followed by a path segment
- `.py:`, `.ts:`, `.go:`, `.rs:`, `.java:` (line references)
- `redis`, `kafka`, `postgres`, `mysql` unless the requirement explicitly
  describes an external contract with that system
- Library names (`react`, `lodash`, `pandas`, etc.) unless they appear in
  a fenced block as an example of allowed external dependency

These patterns generate Warnings, not Blocks — false positives are common
(e.g. `Redis` is genuinely external for a cache-eviction contract). Reviewer
discretion clears the warning.

## Cross-capability references

When a scenario references another capability's scenario, use the
fully qualified ID:

```markdown
- **GIVEN** the user has an active session (see `auth.login.001`)
- **WHEN** they request a refund
- **THEN** ...
```

Short forms (`login.001`) are forbidden — they break grep and make
refactoring fragile.

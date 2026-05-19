# escapes.log entry format

`changes/<change-id>/escapes.log` is APPEND-ONLY. Each escape is a YAML
document separated by `---`. Do not edit prior entries; only the
current entry's `resolution_pending` flag may flip from `yes` to `no`
when the escape closes.

## Empty file initialisation

A new escapes.log starts empty (zero bytes). The first `/escape` call
writes the first entry. The file is created by the skill — do not
pre-create it.

## Entry template

```yaml
---
id: e<NN>
ts: <ISO-8601 UTC, e.g. 2026-05-19T10:23:00Z>
slice: <slice-id, e.g. add-auth-login-s03>
tag: spec-error | scenario-missing | better-interface | scope-overflow
description: |
  <Multi-line. Required structure:

   1. **What was the slice trying to do?**
      One sentence about the in-flight TDD intent.

   2. **What surfaced?**
      The discovery — a Scenario contradicting itself, an edge case
      missing, an interface mismatch, a cost overrun.

   3. **Why it matters.**
      Connect the discovery to externally observable behaviour. If it
      does not, it is not an escape — fix it in /implement.

   4. **Proposed resolution.**
      What you and the user agreed to change in spec.md or slices.md.
   >
secondary_tags:
  - <optional list of additional tags if more than one applies>
resolution_pending: yes
resolved_ts: null
```

When the escape closes (after spec.md/slices.md are updated):

```yaml
resolution_pending: no
resolved_ts: <ISO-8601 UTC>
```

The structural change is only in those two fields. The body of the
entry is never edited.

## Worked examples

### Example 1: spec-error

```yaml
---
id: e01
ts: 2026-05-19T10:23:00Z
slice: add-auth-login-s03
tag: spec-error
description: |
  1. The slice was implementing `auth.login.002` — "lock account after
     5 failures".
  2. Re-reading the Scenario, the THEN says "return 403"; the prior
     Scenario `auth.login.001` returns 401 for invalid credentials. We
     could not tell whether the lock should be silent (same 401) or
     differentiated (403). Two valid implementations exist.
  3. External consumers (the mobile client) would behave differently
     in each case — silent lock prevents enumeration, differentiated
     lock helps support diagnose. Both are reasonable.
  4. Decision: use 403 with a generic error code, no message detail.
     Update `auth.login.002` THEN to say `THEN the server returns 403
     with code "account_locked"`. ID kept (clarification, not semantic
     change).
secondary_tags: []
resolution_pending: yes
resolved_ts: null
```

When closed:

```yaml
resolution_pending: no
resolved_ts: 2026-05-19T10:48:00Z
```

### Example 2: scenario-missing

```yaml
---
id: e02
ts: 2026-05-19T14:10:00Z
slice: add-auth-login-s03
tag: scenario-missing
description: |
  1. The slice was implementing `auth.login.002`.
  2. While writing the failing test for lockout, we realised there's no
     Scenario covering what happens to an already-authenticated session
     when the account locks. Should existing sessions be invalidated?
  3. Externally observable: if existing sessions stay valid, a
     compromised account remains usable until logout. Compliance team
     flagged this in last quarter's audit.
  4. Add new Scenario `auth.login.004` — "lock revokes existing
     sessions". Adds a new slice s05 to cover the session revocation
     pathway.
secondary_tags: []
resolution_pending: yes
resolved_ts: null
```

### Example 3: better-interface

```yaml
---
id: e03
ts: 2026-05-19T16:00:00Z
slice: add-auth-login-s04
tag: better-interface
description: |
  1. The slice was implementing `auth.session.001` — session revocation.
  2. The spec said revoke "by user_id". While building, we found the
     existing session store keys by session_id and ranges by user_id
     are expensive. More importantly, downstream services need to
     know *why* a session was revoked (lockout / logout / admin).
  3. External contract: the revocation event currently emits only
     {user_id, ts}. Adding a `reason` field would let downstream
     consumers route correctly without re-deriving.
  4. Modify `auth.session.001` to require the revocation event include
     `reason: "lockout"|"logout"|"admin"`. ID superseded by new id
     `auth.session.005` to reflect the new contract; old ID retired.
secondary_tags: [spec-error]
resolution_pending: yes
resolved_ts: null
```

### Example 4: scope-overflow

```yaml
---
id: e04
ts: 2026-05-19T17:45:00Z
slice: add-auth-login-s03
tag: scope-overflow
description: |
  1. The slice was implementing `auth.login.002` — account lockout.
  2. Estimated 3 cycles. Burned 6 and still not green. The blocker is
     the existing rate-limit middleware uses a per-IP counter; sharing
     state with a per-account counter requires refactoring the
     middleware to be configurable.
  3. No external behaviour change here — but the slice now bleeds into
     `src/middleware/` which is in `do_not_touch`.
  4. Resolution: split. New slice s06 "make rate-limit middleware
     configurable" with write_scope `src/middleware/**`, blocks s03.
     s03 keeps `auth.login.002` but waits for s06.
secondary_tags: []
resolution_pending: yes
resolved_ts: null
```

## After archive

When the change is archived, `escapes.log` moves with it to
`changes/archive/YYYY-MM-DD-<change-id>/escapes.log`. It is never
deleted, redacted, or summarised. Retrospectives read it directly.

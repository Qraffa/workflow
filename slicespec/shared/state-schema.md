# state.json Schema

`state.json` is a machine-readable index of a change. It is **derived from**
brief.md, spec.md, slices.md, escapes.log, and git history. Humans must not
edit it directly.

## When state.json is written

| Event | Field(s) updated | Skill responsible |
|---|---|---|
| `/clarify` finishes first pass | `change_id`, `status=draft`, `created_at`, `updated_at` | `slicespec-clarify` |
| `/spec` writes spec.md | `status=specified`, `scenarios[]`, `updated_at` | `slicespec-spec` |
| `/slice` writes slices.md | `slices{}`, `parallel_guards[]`, `status=implementing` once first slice claimed | `slicespec-slice` |
| `/implement` dispatches subagent | `slices[sid].status=in_progress`, `slices[sid].phase`, `slices[sid].owner` | `slicespec-implement` |
| `/implement` finishes slice | `slices[sid].status=done`, `slices[sid].evidence{}`, `reviews[]` | `slicespec-implement` |
| `/escape` triggers | `escapes[]` appended, `slices[sid].status=escaped` | `slicespec-escape` |
| `/verify` runs | `verify[]` appended, `status=verifying` then `archived` on success | `slicespec-verify` |

Each writing skill must (a) update markdown first, (b) call its internal
state-writer to project markdown into state.json, (c) bump `updated_at`.

## Top-level schema

```json
{
  "change_id": "add-auth-login",
  "status": "draft|specified|implementing|verifying|archived",
  "created_at": "2026-05-19T10:00:00Z",
  "updated_at": "2026-05-19T15:30:00Z",
  "current_slice": "add-auth-login-s02",
  "parallel_guards": [
    "**/migrations/**",
    "**/config/**",
    "**/*.proto"
  ],
  "scenarios": [
    {
      "id": "auth.login.001",
      "capability": "auth",
      "status": "active|removed|superseded",
      "supersedes": null,
      "first_seen_change": "add-auth-login"
    }
  ],
  "slices": {
    "add-auth-login-s01": {
      "status": "pending|in_progress|done|blocked|escaped",
      "phase": "red|green|refactor|reviewing|null",
      "type": "AFK|HITL",
      "owner": "subagent:implementer-7f3a|null",
      "originally_hitl": false,
      "scenarios": ["auth.login.001"],
      "write_scope": ["src/auth/**", "tests/auth/**"],
      "do_not_touch": ["specs/**", "src/billing/**"],
      "blocked_by": [],
      "blocked_reason": null,
      "estimated_cycles": 3,
      "evidence": {
        "commits": ["a1b2c3d", "e4f5g6h"],
        "tests_run": [
          {"cmd": "pytest tests/auth -v", "result": "pass", "ts": "..."}
        ],
        "implementer_report": "evidence/add-auth-login-s01/implementer-report.md",
        "spec_review": "approved",
        "spec_review_report": "evidence/add-auth-login-s01/spec-review.md",
        "quality_review": "approved",
        "quality_review_report": "evidence/add-auth-login-s01/quality-review.md",
        "controller_diff_check": "passed"
      }
    }
  },
  "escapes": [
    {
      "id": "e01",
      "ts": "2026-05-19T11:00:00Z",
      "slice": "add-auth-login-s01",
      "tag": "scenario-missing|spec-error|better-interface|scope-overflow",
      "description": "free text",
      "resolved": true,
      "resolved_at": "2026-05-19T11:45:00Z"
    }
  ],
  "reviews": [
    {
      "object": "slice:add-auth-login-s01",
      "stage": "spec|quality",
      "verdict": "approved|issues_found|accepted_with_risk",
      "severity_breakdown": {"critical": 0, "warning": 1, "info": 2},
      "warnings_accepted_with_risk": ["magic-number-100 in auth/login.py:42"]
    }
  ],
  "verify": [
    {
      "ts": "2026-05-20T09:00:00Z",
      "mode": "default|strict|pre-pr",
      "exit_code": 0,
      "report_md": "verify-report.md",
      "report_json": "verify-report.json"
    }
  ]
}
```

## Field semantics

### `status` (top-level)

State machine:

```
draft → specified → implementing → verifying → archived
```

`escape` is not a top-level state — it sits inside the slice that triggered it.

### `slices[sid].phase`

Only meaningful when `status == in_progress`. Five values:

- `red` — failing test in place
- `green` — minimal code passes the test
- `refactor` — green, structural cleanup in progress
- `reviewing` — handed to spec/quality reviewer subagent
- `null` (otherwise)

The human-readable `slices.md` shows only the five status buckets; phase lives
in state.json for controller scheduling.

### `slices[sid].owner`

Subagent identifier when claimed for execution. Format:
`subagent:<role>-<short-id>`. Null when not dispatched.

### `slices[sid].originally_hitl`

`true` only when the slice was declared HITL in slices.md and later downgraded
to AFK during `/implement`. Audit trail; never reset to `false`.

### `slices[sid].evidence.controller_diff_check`

Required for every `done` slice. Values:

- `passed`
- `failed:<comma-separated-files>` (slice blocked, see §11 of unified-workflow.md)

### `parallel_guards`

Project-specific high-conflict paths. Default values are seeded by `/slice` on
first run; users can append in slices.md and the writer projects them here.

## Rebuilding state.json from markdown

If state.json is corrupted or out of sync, rebuild in this order:

1. Parse brief.md (`change_id`, type, decision log).
2. Parse spec.md (`scenarios[]`, `status=specified`).
3. Parse slices.md (`slices{}`, `parallel_guards`).
4. Parse escapes.log (`escapes[]`, slice status overrides).
5. Run `git log --format='%H %s' changes/<id>/` to recover commit lists.
6. Run `git diff --name-only <base>..HEAD` per slice's commit range to
   recompute `controller_diff_check` (only if commits are still in tree).
7. Write `created_at` from earliest commit author date.

Never reverse-engineer markdown to match state.json. Markdown is the source of
truth.

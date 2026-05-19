# Flow Artifact Templates

## `brief.md`

```markdown
# Brief: <change>

## Problem

## Goal

## Scope

## Non-Goals

## Domain Language

## Constraints

## Change Type
<new-feature / external-contract / behavior-bug / spec-conformance-bug / refactor / performance / hotfix / prototype>

## Open Questions

## Decision Log
```

## `spec.md`

```markdown
# Spec Delta: <change>

## Capabilities

## ADDED Requirements

### Requirement: <name>
The system SHALL ...

#### Scenario: <scenario-id>
- GIVEN ...
- WHEN ...
- THEN ...

## MODIFIED Requirements

## REMOVED Requirements

## External Contracts

## Acceptance Evidence Required
```

## `plan.md`

```markdown
# Plan: <change>

## Strategy

## Slice Graph

## Slices

### SL1: <behavior increment>
Status: pending
Execution Mode: AFK
Scenarios: <scenario-id>
Dependencies: none
Parallel Group: A
Estimated TDD Cycles: 3
Shared Resource: none
Write Scope:
- src/...
- tests/...

Do Not Touch:
- flow/specs/...
- flow/changes/...
- src/shared/...
- migrations/...

Test Strategy:
- Acceptance:
- Integration:
- Contract:
- Unit:

Tasks:
- [ ] Write a failing test for ...
- [ ] Implement minimal behavior
- [ ] Refactor under green tests
- [ ] Record evidence

## Parallelization Rules

## Escape Policy

## Verification Checklist
```

## `state.json`

```json
{
  "change_id": "<change>",
  "status": "ready",
  "created_at": "YYYY-MM-DDTHH:mm:ssZ",
  "post_hoc_spec_pending": false,
  "current_slice": null,
  "parallel_guards": [
    "**/migrations/**",
    "**/config/**",
    "**/*.proto",
    "**/*.graphql"
  ],
  "slices": {
    "SL1": {
      "status": "pending",
      "phase": null,
      "owner": null,
      "execution_mode": "AFK",
      "originally_hitl": false,
      "estimated_cycles": 3,
      "requires_shared_resource": null,
      "scenarios": ["<scenario-id>"],
      "write_scope": ["src/**", "tests/**"],
      "do_not_touch": ["flow/specs/**", "migrations/**"],
      "evidence": {
        "commits": [],
        "tests_run": [],
        "spec_review": null,
        "quality_review": null,
        "controller_diff_check": null,
        "implementer_report": "evidence/SL1/implementer-report.md",
        "spec_review_report": "evidence/SL1/spec-review.md",
        "quality_review_report": "evidence/SL1/quality-review.md"
      }
    }
  },
  "escapes": [],
  "escape_log": "escapes.log",
  "reviews": [],
  "warnings_accepted_with_risk": [],
  "verify_reports": [],
  "updated_at": "YYYY-MM-DDTHH:mm:ssZ"
}
```

## `escapes.log`

```text
YYYY-MM-DDTHH:mm:ssZ | slice=<slice-id> | tag=[escape:spec-error] | status=opened | description=<short reason>
YYYY-MM-DDTHH:mm:ssZ | slice=<slice-id> | tag=[escape:spec-error] | status=closed | resolution=<short resolution>
```

## `verify-report.json`

```json
{
  "change_id": "<change>",
  "mode": "default",
  "status": "passed",
  "exit_code": 0,
  "critical": [],
  "warnings": [],
  "suggestions": [],
  "checks": {
    "structure": "passed",
    "scenario_evidence": "passed",
    "external_contracts": "passed",
    "escapes_closed": "passed",
    "write_boundaries": "passed"
  }
}
```


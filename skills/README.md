# SDD + TDD Flow Skills

This collection implements the integrated SDD + TDD workflow as agent skills.

## Skills

- `flow-clarify`: clarify intent, scope, constraints, domain language, and change type.
- `flow-spec`: define the external behavior contract and scenario delta.
- `flow-plan`: split the contract into vertical behavior slices with TDD strategy and write boundaries.
- `flow-apply`: implement slices with Red-Green-Refactor, review, and evidence.
- `flow-escape`: route implementation feedback back into the spec or plan.
- `flow-review`: review brief, spec, plan, slice, or whole change quality.
- `flow-close`: verify Spec/Test/Code semantic consistency, sync main specs, and archive.

## Default Artifacts

```text
flow/
  specs/
  changes/
    <change>/
      brief.md
      spec.md
      plan.md
      state.json
      escapes.log
      evidence/
      verify-report.md
      verify-report.json
    archive/
```

The skills are intentionally agent-first, not CLI-first. They create and update markdown artifacts and machine-readable state in the repo, while leaving deterministic CLI automation as a later L2 concern.


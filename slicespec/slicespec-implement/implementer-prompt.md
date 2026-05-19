# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent from
slicespec-implement (Mode A).

The controller fills every `<...>` placeholder with the actual content.
Subagents do NOT read source files of the SliceSpec change directory —
all relevant slices.md / spec.md content is pasted in below.

```
Agent tool:
  subagent_type: general-purpose   (or smaller for purely mechanical slices,
                                    more capable for slices that slipped
                                    HITL labelling)
  description: "Implement slice <slice-id>: <title>"
  prompt: |
    You are implementing slice <slice-id>: <title>
    Change: <change-id>
    Worktree: <absolute path>

    ## Slice (from slices.md)

    - **type**: <AFK|HITL-downgraded>
    - **covers**: <scenario-ids>
    - **blocked_by**: <slice-ids or none>
    - **write_scope**:
      <list of globs>
    - **do_not_touch**:
      <list of globs>
    - **test_strategy**:
      <bullets>
    - **estimated_cycles**: <N>

    ## Scenarios you must satisfy (from spec.md, pasted verbatim)

    ### Scenario: <name>  <!-- id: <scenario-id> -->
    - **GIVEN** ...
    - **WHEN** ...
    - **THEN** ...

    (Repeat for every Scenario in `covers`.)

    ## Confirmed decisions (only present for HITL-downgraded slices)

    - <user-confirmed decision 1>
    - <user-confirmed decision 2>

    These are FIXED. Do not relitigate. If the work requires a new
    judgement call not on this list, stop with BLOCKED(needs-hitl-decision).

    ## Hard rules

    1. **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.**
       Iron rule. Violation = work invalid, must be deleted and
       restarted.

    2. **Write only inside write_scope.**
       Modifying any file outside the listed globs is a structural
       failure that will be detected by the controller's git diff
       check. You will not be given a chance to argue this.

    3. **Never touch do_not_touch paths.**
       Even if it looks like a small ergonomic improvement. If the
       slice would benefit from touching one, stop and return
       BLOCKED(scope-overflow) with the file and reason.

    4. **Refactor only when all tests are GREEN.**
       Never restructure code with a failing test in the suite.

    5. **Tests must reference Scenario IDs** in one of these forms:
       - `# @scenario: <id>` (inline comment above the test)
       - Function/method name like `test_scenario_<id_underscored>_<desc>`
       - First line of docstring: `Scenario: <id>`

    6. **One commit per Red→Green→Refactor cycle.**
       Commit messages end with `[<slice-id>]`.

    ## TDD cycle (run until every Scenario in `covers` is exercised)

    RED:
      - Write one test for ONE behaviour from one Scenario.
      - Reference the Scenario ID per the form rules.
      - Run the test. Verify it fails because the feature is missing —
        not because of a typo.

    GREEN:
      - Write the MINIMUM code to make the test pass.
      - No speculative interfaces. No unrequested features.
      - Run the test. Verify it passes.
      - Run the module's wider tests. Verify no regressions.

    REFACTOR:
      - Only when GREEN.
      - Improve names, extract helpers, reduce duplication.
      - Do not change behaviour. Tests must remain green throughout.

    COMMIT:
      - Stage tests + implementation together. Short present-tense
        message ending with `[<slice-id>]`.

    ## When you are stuck

    Bad work is worse than no work. Stop and escalate when:

    - The slice requires architectural judgement the spec does not encode.
    - You have read several files without progress.
    - You cannot find a way to write the next test without bending the
      Scenario.
    - You feel the right answer requires changing the slice or spec.

    Report status `BLOCKED` (with sub-reason) or `NEEDS_CONTEXT`. The
    controller will route you to /escape or supply more context.

    ## Before reporting back: self-review

    Fresh eyes. Ask yourself:

    Completeness:
    - Did I exercise every Scenario in `covers`?
    - Did any test pass on first run? (If yes, it isn't TDD — fix it.)
    - Did I write only what the Scenarios required?

    Discipline:
    - Did I write code before a failing test at any point?
    - Did I refactor while RED?
    - Did I touch a file outside write_scope or inside do_not_touch?

    Tests:
    - Do they verify behaviour through public interfaces?
    - Do they reference Scenario IDs correctly?
    - Are they meaningful, not mock-shaped?

    Fix issues now. Do not pass them downstream.

    ## Report format

    Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

    Body:
    - **Cycles**: <N> Red→Green→Refactor cycles.
    - **Scenarios covered**: <list of ids with test names>
    - **Tests run**: <commands and results>
    - **Files modified**: <list, with line counts>
    - **Commits**: <SHAs>
    - **Self-review findings**: <any concerns>
    - **Reason for status** (if not DONE): <free text>

    Status meanings:
    - DONE — all Scenarios covered, self-review clean.
    - DONE_WITH_CONCERNS — work complete but you have doubts (note them).
    - BLOCKED — you cannot finish. Sub-reasons:
        BLOCKED(needs-hitl-decision)
        BLOCKED(scope-overflow)
        BLOCKED(spec-error)
        BLOCKED(needs-context)
        BLOCKED(too-complex)
    - NEEDS_CONTEXT — missing information; specify what.

    Never silently produce work you are unsure about. Never invent
    answers. Never bypass the TDD cycle.
```

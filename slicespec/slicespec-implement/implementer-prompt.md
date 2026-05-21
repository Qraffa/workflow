# Implementer Prompt / TDD Discipline

Use this file in two modes:

- **Default (main-session implementation):** the body below is the
  source-of-truth specification of a correct TDD cycle. Read it,
  internalise the hard rules, then drive your own
  Red→Green→Refactor cycles in the main session. The "Report
  format" section becomes your slice-completion summary written
  into evidence/.
- **Subagent mode (opt-in, see `subagent-mode.md`):** wrap the
  prompt body in the `Agent` tool block and dispatch as a fresh
  subagent. The controller fills every `<...>` placeholder with
  actual content; the subagent does NOT read source files of the
  SliceSpec change directory — all relevant slices.md / spec.md
  content is pasted into the prompt.

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

    **Anti-pattern — DO NOT DO THIS:** Writing all the Scenario tests
    first ("RED = write every test"), then implementing all of them
    ("GREEN = write all the code"). This is *horizontal slicing* and
    it produces crap tests: you end up verifying *imagined* behaviour
    and the *shape* of data, not what users actually need. Such tests
    pass when behaviour breaks and break when behaviour is fine.

    The correct shape is *vertical*: one test → one impl → one
    commit → next test. Each cycle is shaped by what the previous
    one taught you about the design. Even if `covers` lists five
    Scenarios, run five small cycles — not one big batch.

    RED:
      - Write ONE test for ONE behaviour from ONE Scenario.
      - Test through the PUBLIC interface — not via internal
        collaborators, not by inspecting private state, and not by
        side-channels like raw DB queries.
      - Reference the Scenario ID per the form rules.
      - Run the test. Verify it fails because the feature is missing —
        not because of a typo.

    GREEN:
      - Write the MINIMUM code to make the test pass.
      - No speculative interfaces. No extra parameters "for later".
        No unrequested features.
      - Mock only at system boundaries: external APIs, databases
        you don't own, time/randomness, the filesystem. NEVER mock
        modules you own, internal collaborators, or the system under
        test itself (mocking the SUT is a Critical defect — your test
        proves nothing).
      - Run the test. Verify it passes.
      - Run the module's wider tests. Verify no regressions.

    REFACTOR:
      - Only when GREEN. Never restructure with a failing test in
        the suite — the green bar is what tells you the change was
        safe.
      - Improve names, extract helpers, reduce duplication.
      - Consider whether the new code revealed a deeper module
        opportunity: small interface, thick implementation. Avoid
        shallow wrappers that just pass through.
      - Refactor candidates worth looking for after each cycle:
        duplication → extract; long method → private helpers;
        shallow module → deepen or combine; feature envy → move
        logic to where the data lives; primitive obsession → value
        object; existing code the new code revealed as problematic.
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
    - Did I run small vertical cycles, or did I batch all the tests
      first and then all the impl? (The latter is horizontal slicing
      — undo and redo.)

    Tests:
    - Do they verify behaviour through the public interface, or do
      they reach into internals (private methods, raw DB queries,
      asserting on call counts of internal collaborators)?
    - Would they survive a pure refactor that changes structure but
      not behaviour? If renaming an internal helper would break a
      test, that test is wrong.
    - Are mocks limited to system boundaries (external APIs, the
      database, time/randomness, filesystem)? Any mock of code I
      own — or of the SUT — must be removed.
    - Do they reference Scenario IDs correctly?

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

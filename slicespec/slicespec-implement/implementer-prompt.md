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
  content is pasted into the prompt. The one exception is
  `test-rules.md`, which the subagent MUST open at start.

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

    ## MANDATORY first read

    Open and internalise `test-rules.md` (sibling file in this
    skill directory) BEFORE writing any test. It is the single
    source of truth for what a good test, a bad test, and a
    licensed mock look like, and what the horizontal-slicing
    anti-pattern is. Every test you write must pass the §8
    pre-flight checklist in that file.

    Do not paraphrase or re-derive those rules in this prompt.
    Just apply them.

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

    7. **Test / mock / anti-pattern rules come from `test-rules.md`.**
       Apply §8 pre-flight before every test. Failing review on
       anything covered there means you skipped the pre-flight.

    ## TDD cycle (run until every Scenario in `covers` is exercised)

    Vertical, not horizontal. Even when `covers` lists many
    Scenarios, run them as small Red→Green cycles — one per
    behaviour. Never batch all the tests then all the impl
    (see `test-rules.md` §5).

    RED:
      - Apply `test-rules.md` §8 pre-flight to the test you are
        about to write.
      - Write ONE test for ONE behaviour from ONE Scenario.
      - Reference the Scenario ID per Hard rule 5.
      - Run the test. Verify it fails because the feature is
        missing — not because of a typo.

    GREEN:
      - Write the MINIMUM code to make the test pass.
      - No speculative interfaces. No extra parameters "for later".
        No unrequested features.
      - Mocks only at boundaries listed in `test-rules.md` §4.
      - Run the test. Verify it passes.
      - Run the module's wider tests. Verify no regressions.

    REFACTOR:
      - Only when GREEN.
      - Scan for refactor candidates per `test-rules.md` §6
        (duplication, shallow modules, feature envy, primitive
        obsession, etc.).
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

    ## Before reporting back: pre-report self-check

    This is your own pre-delivery check — not the spec/quality
    reviewer pass (those are fresh subagents the controller
    dispatches after step 3). Fresh eyes. Answer each question. If
    any is "no" or "I'm not sure", fix the issue before reporting.

    Completeness:
    - Did I exercise every Scenario in `covers`?
    - Did any test pass on first run? (If yes, it isn't TDD — fix it.)
    - Did I write only what the Scenarios required?

    Discipline:
    - Did I write code before a failing test at any point?
    - Did I refactor while RED?
    - Did I touch a file outside write_scope or inside do_not_touch?
    - Did I run small vertical cycles, or did I batch tests then
      impl? (The latter violates `test-rules.md` §5 — undo and redo.)

    Tests — for every test I wrote, does it pass the
    `test-rules.md` §8 pre-flight (all five questions)?
    Specifically:
    - Public interface only? (§1, §2)
    - No internal mocks, no private inspection, no raw DB
      verification? (§3.a, §3.b)
    - Mocks (if any) only at §4 boundaries? No mock of code I own?
      No mock of the SUT?
    - Name describes WHAT, not HOW? (§7)
    - Would it survive a pure internal refactor and still catch a
      real behaviour break? (§1)

    Fix issues now. Do not pass them downstream. A reviewer
    finding against `test-rules.md` means you skipped this
    pre-report check.

    ## Report format

    Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

    Body:
    - **Cycles**: <N> Red→Green→Refactor cycles.
    - **Scenarios covered**: <list of ids with test names>
    - **Tests run**: <commands and results>
    - **Files modified**: <list, with line counts>
    - **Commits**: <SHAs>
    - **Pre-flight confirmation**: "applied `test-rules.md` §8 to
      every test" (mandatory; do not omit).
    - **Pre-report check findings**: <any concerns>
    - **Reason for status** (if not DONE): <free text>

    Status meanings:
    - DONE — all Scenarios covered, pre-report check clean.
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

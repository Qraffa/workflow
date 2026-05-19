# Code Quality Reviewer Subagent Prompt

Use this template when dispatching a code-quality reviewer subagent.
Run ONLY after spec compliance reviewer returns `approved`.

The reviewer's job is to verify the implementation is well built — not
just that it passes tests, but that the tests test real behaviour and
the code itself is maintainable.

Reviewer is a FRESH subagent and does not see the implementer's
context.

```
Agent tool:
  subagent_type: general-purpose
  description: "Code quality review for slice <slice-id>"
  prompt: |
    You are reviewing the code quality of slice <slice-id>.

    Change: <change-id>
    Worktree: <absolute path>
    Base SHA: <pre-slice-sha>
    Head SHA: <post-slice-sha>

    Spec compliance review has already passed for this slice. You do
    not need to re-verify Scenario coverage. Focus on how the code is
    built.

    ## How to review

    Read `git diff <base>..<head> -- <worktree>` and the touched files.
    Evaluate the following.

    ### 1. Test quality

    - Are tests verifying observable behaviour through public
      interfaces, or are they coupled to internal collaborators?
      (Internal-collaborator tests are fragile; flag them Warning.)
    - Do tests use real code paths, or mock the system under test?
      (Mocking the SUT is a Critical issue.)
    - Are mocks limited to expensive / external dependencies?
    - Could the tests survive a reasonable internal refactor?
    - Are there tests that pass without exercising the new code path?
      (Dead tests are Critical.)

    ### 2. Speculative code / YAGNI

    - Are there functions, parameters, classes, or branches the
      Scenarios did not require?
    - Are there configuration knobs added "just in case"?
    - Did the implementer build for hypothetical future needs?

    Each unjustified speculative addition is a Warning.

    ### 3. Refactor discipline

    - Look at the commit history for the slice's commits. Do any
      commits show refactoring done while a test was failing?
      (This is detectable when a commit modifies non-test files and
      changes them in ways the immediate test does not exercise.)
    - Refactor-while-RED is a Critical issue (it can mask broken
      behaviour).

    ### 4. File and module shape

    - Does each touched file still have one clear responsibility?
    - Did this slice grow an existing file unreasonably? (Compare diff
      line counts against the plan's intent.)
    - Are new files placed in the directory the slice's write_scope
      implied?
    - Are unit boundaries crisp, or do helpers leak across modules?

    ### 5. Names and intent

    - Do names match what things DO (behaviour) rather than HOW
      (implementation)?
    - Are any names ambiguous, overloaded, or misleading?
    - Were renames performed and were callers updated everywhere?

    ### 6. Common rationalisations to look for

    - "Tests after, same goal" — look for tests committed in the same
      commit as the production code with no prior failing-test commit.
    - "Already manually tested" — look for code paths with no test.
    - "Internal-only, no test needed" — look for new branches that
      have no assertion exercising them.

    These do not necessarily mean a violation occurred, but they are
    signals worth checking.

    ## Severity

    - **Critical** — must fix. Examples: SUT mocked, refactor while
      RED, missing assertions, security/data-correctness defects.
    - **Warning** — should fix, but can be `accepted_with_risk` with a
      one-sentence justification recorded by the controller. Examples:
      magic numbers without constant, fragile-test smells, file size
      growth.
    - **Info** — observation, no action required. Examples: stylistic
      preference, optimisation idea.

    ## Report

    Verdict: approved | issues_found

    Format issues as:

    [Critical] <file>:<line>  — <one-sentence description>
    [Warning]  <file>:<line>  — <one-sentence description>
    [Info]     <file>:<line>  — <one-sentence description>

    Strengths section: 2-4 bullets noting what the implementer did
    well. (This is not flattery; it is a signal that the controller
    can trust similar code from the same implementer next time.)

    Be precise. File:line references must be exact. Do not propose
    fixes; let the implementer choose how.
```

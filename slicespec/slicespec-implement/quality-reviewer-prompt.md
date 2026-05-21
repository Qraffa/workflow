# Code Quality Reviewer Prompt

**This file is a fresh-subagent prompt template.** The controller
(main session) dispatches it by default at Step 5 of `SKILL.md`,
after spec compliance review returns `approved`. The body below
substitutes the `<...>` placeholders and is wrapped in an `Agent`
tool block.

The rules for what makes a test good/bad, what mocks are
licensed, and what counts as horizontal slicing live in
`test-rules.md` — the same file the implementer used. Do not
restate them here. This reviewer adds **detection and grading**
on top of those rules.

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

    Spec compliance review has already passed. Do not re-verify
    Scenario coverage. Focus on how the code is built.

    ## Required reading

    Open `test-rules.md` (sibling file in this skill directory)
    BEFORE auditing. Every violation you flag below maps to a
    specific section there — cite it in your finding.

    ## Inputs

    Read `git diff <base>..<head> -- <worktree>` and the touched
    files. Also read `git log -p <base>..<head>` for cycle-shape
    evidence (you cannot detect refactor-while-RED or
    horizontal-slicing without commit content).

    ## What this review adds (beyond what the implementer self-checked)

    The implementer already ran the `test-rules.md` §8 pre-flight
    on each test. Your job is to catch what self-review reliably
    misses:

    ### A. Commit-shape detection (you need the log; implementer can't fully self-audit this)

    1. **Refactor-while-RED** — Critical (cites `test-rules.md`
       intro / §6).
       Walk the commit list. For any commit, if the diff modifies
       non-test files in ways the immediate test does not exercise,
       and a later commit "fixes" tests for those changes, that's
       refactor-while-RED. Symptom: green→red→green sequence inside
       a single slice.

    2. **Horizontal slicing** — Critical (cites `test-rules.md` §5).
       Pattern: one commit adds all the tests, the next commit adds
       all the production code, with no interleaving. Detect by
       looking at file types per commit (tests-only vs impl-only)
       and the ordering. Vertical cycles look like tests+impl
       paired in every commit.

    ### B. Test-quality audit against `test-rules.md`

    Open each test the slice introduced. Map each defect to a
    `test-rules.md` clause:

    | Symptom in code | Clause | Severity |
    |---|---|---|
    | Asserts on internal call (`toHaveBeenCalled*` on a non-boundary collaborator) | §3.a | Warning (Critical if the sole assertion) |
    | Verifies via raw DB query / private state / log scraping | §3.b | Warning |
    | Mocks the system under test itself | §3.c | **Critical** |
    | Mocks an internal collaborator the team owns | §3.d | Warning |
    | Mocks something not on the §4 boundary list | §4 | Warning |
    | Test name describes mechanism not behaviour | §7 | Info (Warning if pervasive) |
    | Test passes without exercising the new code (dead test) | — | **Critical** |
    | Test would break on rename of a private helper | §1 | Warning |

    For each finding, cite the exact `test-rules.md §X.Y` so the
    implementer knows which rule was violated and where to relearn
    it.

    ### C. Speculative code / YAGNI

    - Functions, parameters, classes, or branches the Scenarios
      did not require.
    - Configuration knobs added "just in case".
    - Code built for hypothetical future needs.

    Each unjustified speculative addition is a Warning.

    ### D. File and module shape

    - Does each touched file still have one clear responsibility?
    - Did the slice grow an existing file unreasonably? (Compare
      diff line counts against the plan's intent.)
    - Are new files placed in the directory the slice's
      `write_scope` implied?
    - Are unit boundaries crisp, or do helpers leak across modules?
    - Cite `test-rules.md §6` if a deep-module / feature-envy
      opportunity was missed.

    ### E. Rationalisation signals (audit-only — these can hide real defects)

    Walk the diff with these red flags in mind:

    - "Tests after, same goal" → tests committed in the same
      commit as the production code with no prior failing-test
      commit. Cross-check with cycle-shape detection (§A).
    - "Already manually tested" → code paths with no test.
    - "Internal-only, no test needed" → new branches with no
      assertion exercising them.
    - "Just mocked the collaborator to make the test simpler" →
      mocks of internal modules; cite §3.d.
    - "Faster to read the DB / state file directly than go through
      the API" → assertions that bypass the public interface; cite
      §3.b.

    These are signals, not automatic verdicts. Investigate before
    flagging.

    ## Severity

    - **Critical** — must fix. Examples: SUT mocked (§3.c),
      refactor while RED, horizontal slicing (§5), missing
      assertions, dead test, security/data-correctness defects.
    - **Warning** — should fix, but can be `accepted_with_risk`
      with a one-sentence justification recorded by the controller.
      Examples: §3.a/§3.b/§3.d violations, magic numbers without
      a constant, fragile-test smells, file size growth.
    - **Info** — observation, no action required. Examples:
      stylistic preference, optimisation idea.

    ## Report

    Verdict: approved | issues_found

    Format issues as:

    [Critical] <file>:<line>  (test-rules.md §X.Y) — <one sentence>
    [Warning]  <file>:<line>  (test-rules.md §X.Y) — <one sentence>
    [Info]     <file>:<line>  (test-rules.md §X.Y or n/a) — <one sentence>

    Always cite the clause when one applies. Findings without a
    clause are limited to YAGNI / file shape / Info-level remarks.

    Strengths section: 2-4 bullets noting what the implementer did
    well. (This is not flattery; it is a signal that the
    controller can trust similar code from the same implementer
    next time.)

    Be precise. File:line references must be exact. Do not propose
    fixes — let the implementer choose how.

    ## A note on rework

    Any Warning or Critical finding that maps to a `test-rules.md`
    clause means the implementer skipped the §8 pre-flight on
    that test (or §5 vertical-cycle discipline). The fix path is
    short: re-run pre-flight on the offending test(s), reshape,
    re-commit. Most such findings should never reach review in
    the first place.
```

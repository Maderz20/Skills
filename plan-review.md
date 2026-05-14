# Plan review

Read this only when invoked to review a completed plan against intent. The reviewer agent runs in a fresh session with this doc + the plan + access to the repo.

## Purpose

Catch drift between *intent* (the plan's Goal and step assertions) and *outcome* (what's actually in the codebase). Implementation can pass every step's validation and still miss the goal — this review catches that.

## Read order

1. The plan file in full, including all Result sections
2. `git log --oneline` since the plan started, to see commits made
3. `git diff --stat` against the plan's starting point, to see file change scope
4. **Spot-check files** — don't read every changed file in full. Pick:
   - The single most central file changed (usually the largest diff in `Files in scope`)
   - One file you'd expect to change but isn't in the diff (verify it correctly stayed unchanged)
   - One file from "Out of scope" (verify it didn't change)
5. The coverage self-report from the Test step

## What to check

### Step compliance
- [ ] Every Implementation step's Result has `Status: ✅ Complete`
- [ ] Every step-specific assertion appears to hold — spot-check by reading the relevant code, not by re-running tests
- [ ] No step's "Files actually changed" exceeds its "Files modified" plan
- [ ] If any step has `Status: ⚠️ Partial` or `❌ Blocked`, the Result documents why

### Goal compliance
- [ ] The plan's Goal paragraph is achieved by the implementation as a whole
- [ ] User-facing behaviour described in the Goal is observable in the code (function signatures, route definitions, response shapes — whatever applies)
- [ ] No alternative interpretation of the Goal was implemented (drift)

### Scope compliance
- [ ] No production code modified outside `Files in scope → Will modify`
- [ ] `Out of scope` items remain untouched
- [ ] No new dependencies added without being mentioned in Approach
- [ ] No new files created outside what the plan describes

### Question and assumption hygiene
- [ ] Every `Open question` is either resolved (answer recorded in plan) or migrated to a follow-up plan
- [ ] Every `[ASSUMPTION]` was either confirmed or overridden — none left dangling
- [ ] Decisions noted in Result sections don't contradict the plan's Approach without justification

### Test compliance
- [ ] Test step's coverage self-report shows no critical gaps
- [ ] Every step-specific assertion has corresponding test coverage (sample 2–3 to verify)
- [ ] Test count is reasonable for the surface area changed (a 5-file feature with 3 tests is suspicious)

### Quality smell-checks
- [ ] No commented-out code blocks left in
- [ ] No `TODO` / `FIXME` / `XXX` introduced without a tracking issue reference
- [ ] No print statements, debugger calls, or temporary logging left in
- [ ] Imports are clean (no unused, no wildcard imports introduced)

## Output: Compliance Report

Produce this exact structure as the final message. No prose preamble.

```
# Compliance Report — [plan filename]

**Date:** YYYY-MM-DD
**Reviewer model:** [model used]
**Reviewer context:** [N tokens read]

## Step compliance
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Goal compliance
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Scope compliance
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Question and assumption hygiene
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Test compliance
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Quality smell-checks
✅/⚠️/❌ <one-line summary>
- [issues, if any]

## Recommendation

[ship | ship with follow-up | fix | replan]

**If "ship with follow-up":** [list of follow-up items, with severity]
**If "fix":** [list of blocking issues that must be addressed before merge]
**If "replan":** [why current plan can't be salvaged]
```

## Recommendation guide

- **ship** — all sections ✅, no issues
- **ship with follow-up** — minor ⚠️ items that don't block merge but need follow-up plans (e.g., one missing edge-case test, a documented assumption to revisit)
- **fix** — any ❌ in Goal/Scope/Step compliance, or significant ⚠️ — implementation needs adjustment before merge
- **replan** — implementation diverged so much from plan that revising plan + re-running steps is cheaper than fixing in place

## What not to do

- ❌ Don't re-run tests (the test step already did this — trust the self-report and spot-check)
- ❌ Don't read every changed file in full — spot-check, don't audit
- ❌ Don't suggest unrelated improvements ("you could refactor X while you're at it")
- ❌ Don't fix issues yourself — your role is to identify and recommend, not to implement
- ❌ Don't be lenient because a step says ✅ Complete — verify against the actual code
- ❌ Don't be exhaustive about minor style nits — focus on correctness against intent

## Cost guidance

A typical review reads ~30K tokens (plan ~5K, diff stat ~2K, 2–3 spot-checked files ~15K, this doc ~3K, miscellaneous ~5K) and produces ~2K of output. On GPT-5.4 that's roughly $0.10 per review. Worth it — this catches things every step-level validation misses.

If a review is exceeding 80K input tokens, you're reading too much. Stop, narrow your spot-checks, and continue.

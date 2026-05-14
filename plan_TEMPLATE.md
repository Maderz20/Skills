# Plan: [Feature / Task name]

**Date:** YYYY-MM-DD
**Plan author:** [model used, e.g., GPT-5.4]
**Estimated implementation steps:** N
**Estimated context per step:** ~XK tokens (target: stay under 200K)

---

## Goal

One paragraph. What's being built/changed and why. State user-facing intent, not implementation detail.

---

## Approach

3–5 bullets describing the chosen approach. For each significant choice, one line on why this over alternatives.

---

## Open questions

> The planner lists unknowns it could not decide. The user answers these *before* implementation starts. If empty, write "None."

- [ ] **Q1:** [Question] — *needs answer before step N*
- [ ] **Q2:** [Question] — *needs answer before step M*

---

## Assumptions

> The planner makes explicit choices for low-leverage unknowns. The user reviews and overrides as needed. If empty, write "None."

- `[ASSUMPTION]` Used Pydantic Settings over `os.environ` for type safety. Override: switch to env-var pattern if simpler.
- `[ASSUMPTION]` Rate limit applies per-IP not per-user. Override: change to per-user if auth context is available.

---

## Files in scope

**Will modify:**
- `src/path/file1.py` — what changes
- `src/path/file2.py` — what changes

**Will read for context only:**
- `src/path/reference.py` — why

**Out of scope (do not touch):**
- `src/legacy/*` — see plan §"Out of scope"

---

## Implementation steps

> Each step's validation runs lint, types, and step-specific assertions. **No pytest.** Tests are written and run in a separate step at the bottom.
>
> After completing a step, the executor MUST fill in the Result section. This is how the next step (potentially in fresh context) learns what was done.

---

### Step 1: [Atomic name]

**Action:** What to do, in 1–3 sentences. Be specific — file paths, function names, line ranges where possible.

**Files modified:** `src/path/file.py`

**Validation checklist:**

- [ ] `ruff check .` passes with no new errors
- [ ] `mypy src` passes with no new errors
- [ ] Files modified during this session match the list above (executor self-verifies — don't rely on `git diff` alone since prior uncommitted steps will appear)
- [ ] [Step-specific assertion 1 — make it runnable without pytest. Examples below.]
- [ ] [Step-specific assertion 2]

**Examples of good step-specific assertions (no pytest needed):**
- "Module imports cleanly: `python -c 'from src.foo import bar'` exits 0"
- "Function signature matches: `rg 'def parse_amount\(' src/` shows new signature with `Decimal` return type"
- "Old name is gone: `rg 'old_function_name' src/` returns 0 results"
- "Config key exists: `python -c 'from src.config import settings; print(settings.RATE_LIMIT)'` prints expected default"
- "Symbol is exported: `python -c 'from src.api import RateLimitMiddleware'` succeeds"

**Acceptance:** All checklist items pass.

**Result:** *(filled by executor after step completes — mandatory, capped at 6 lines)*
```
- Status: [✅ Complete | ⚠️ Partial | ❌ Blocked]
- Files actually changed: [comma-separated list]
- Decisions: [non-obvious choices, 1–2 lines, or "none"]
- For next step: [key context the next executor needs, 1 line, or "none"]
```

---

### Step 2: [Atomic name]

**Action:** ...

**Files modified:** ...

**Validation checklist:**
- [ ] `ruff check .` passes
- [ ] `mypy src` passes
- [ ] Files modified during this session match the list above
- [ ] [Step-specific assertion]

**Acceptance:** All checklist items pass.

**Result:** *(filled by executor)*
```
- Status:
- Files actually changed:
- Decisions:
- For next step:
```

---

### Step 3: [Atomic name]

...

---

## Test step

> Run after all implementation steps show `Status: ✅ Complete`. Use a cheaper model (e.g., GPT-5.4 mini) in a fresh session.

**Reading strategy** (avoid blowing context on large refactors):
- Always read in full: this plan file, `docs/agent/unit-tests.md`, `docs/agent/test-coverage-strategy.md`
- For each step's "Files modified": read the **specific functions/classes** that were added or changed, not entire files
- For files >500 lines: use `view_range` based on the step's Action description
- If total context approaches 150K, stop and write tests in batches per step

**Action:**
1. Read prerequisites per the Reading strategy above.
2. For each function added or modified, write unit tests following `test-coverage-strategy.md`'s mandatory checklist.
3. Treat each step's "Step-specific assertions" as part of the spec — every assertion must have ≥1 corresponding test.
4. Run `pytest tests/unit -x`. Fix failures *in the test code*. If a real bug surfaces in production code, stop and report — do not fix it here.
5. Produce the coverage self-report defined in `test-coverage-strategy.md`.

**Files modified:** `tests/unit/...` only

**Validation checklist:**
- [ ] `pytest tests/unit -x` passes
- [ ] No tests modified outside `tests/unit/`
- [ ] No production code modified
- [ ] Coverage self-report produced and attached to final message
- [ ] Every step-specific assertion from steps 1–N is covered by ≥1 test

**Acceptance:** All checklist items pass. Report any gaps explicitly.

**Result:**
```
- Status:
- Tests added: [count]
- Coverage self-report: [attached above / link]
- Gaps reported: [list, or "none"]
```

---

## Plan review step

> Run after the Test step shows `Status: ✅ Complete`. Use GPT-5.4 in a fresh session. Follow `docs/agent/plan-review.md`.

**Action:** Reviewer reads this plan (including all Result sections), spot-checks the implementation, and produces a Compliance Report per `plan-review.md`.

**Validation checklist:**
- [ ] Compliance Report produced and attached
- [ ] Every implementation step's Result is `Complete`
- [ ] Every step-specific assertion verified (spot-check)
- [ ] Plan Goal is achieved (high-level read-through)
- [ ] Open Questions either answered or migrated to a follow-up plan
- [ ] No unplanned production-code changes (compare actual diff vs `Files in scope`)

**Acceptance:** Compliance Report's recommendation is "ship" or "ship with follow-up." Anything else means fix or replan before merging.

**Result:**
```
- Status:
- Compliance Report recommendation: [ship | ship with follow-up | fix | replan]
- Issues found: [count]
```

---

## Out of scope

- [Things explicitly not being done in this plan]
- [Adjacent issues noticed but deferred — link to follow-up plan if created]

---

## Rollback

- `git revert [commits]` for code changes
- [Note any non-git side effects: DB migrations, config changes, infra changes]

---

## Handover notes

> Optional. Used only when an extended handover is needed (see AGENTS.md §7). Most plans won't have entries here.

*(empty by default)*

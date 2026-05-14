# AGENTS.md

> Sent on every turn. Every token here is paid ~10× per session. Keep it tight.

---

## 1. Project orientation

**Stack:** `[FILL IN — e.g., Python 3.11, FastAPI, Postgres, React+Vite]`

**Repo map:**

```
src/
  api/          # HTTP handlers — thin
  services/     # Business logic — most changes go here
  models/       # Pydantic + ORM models
  workers/      # Background jobs
tests/
  unit/         # Fast, no DB
  integration/  # Slow, DB-backed
.copilot/plans/ # Active plan files
docs/agent/     # On-demand agent guides — read when invoked
```

**Do not read unless asked:** `node_modules/`, `.venv/`, `dist/`, `build/`, `.next/`, `migrations/`, `*.lock`, `*.snap`, `*.min.*`, `legacy/`.

---

## 2. Commands

| Purpose | Command | When to run |
|---|---|---|
| Lint | `ruff check .` | After every code change |
| Format | `ruff format .` | After every code change |
| Types | `mypy src` | After every code change |
| Unit tests | `pytest tests/unit -x` | **Only during the plan's test step** |
| Full tests | `pytest` | **Only when explicitly asked** |

Lint and types are not tests — they're cheap static checks, run them every step. Pytest is deferred to the test step.

---

## 3. On-demand documentation

Read only when the current task matches:

- `docs/agent/unit-tests.md` — *how* to write unit tests
- `docs/agent/test-coverage-strategy.md` — *what* to test (read alongside `unit-tests.md`)
- `docs/agent/integration-tests.md` — when writing integration tests
- `docs/agent/refactor-playbook.md` — refactoring 3+ files
- `docs/agent/debugging.md` — non-trivial bug investigation
- `docs/agent/plan-review.md` — when reviewing a completed plan against intent

**Always read at the start of any plan-driven implementation step:**
- `docs/agent/repo-gotchas.md` — short, high-signal gotchas that prevent common time-wasting failures

Don't read other on-demand docs proactively. Read when invoked.

---

## 4. Search before reading (cost rule)

Default to `rg` for "where/how/what" questions. Reading a 2,000-line file when 30 lines of grep would answer is a budget violation.

- "Where is X used?" → `rg "X"`
- "What handles route Y?" → `rg -l "pattern"`
- Files >500 lines: use `view` with `view_range`, never read whole

---

## 5. Context discipline

- Compact at ~180K accumulated tokens. Never cross 272K (input cost doubles).
- Don't re-read files seen this session unless externally edited.
- Read smallest viable slice: function over file, file over directory.
- One concern per session.

---

## 6. Working with plan files

Plans live in `.copilot/plans/`. Each plan has Implementation steps, a Test step, and a Plan Review step.

### If you are planning (writing a new plan)

1. Read this file and `.copilot/plans/_TEMPLATE.md`.
2. Explore only the files needed to plan well — use `rg` first, read minimally.
3. **Triage unknowns:**
   - High-leverage unknowns (would change the approach) → ask up to 3 questions in chat *before* writing the plan
   - Low-leverage unknowns (defaults you can pick) → make a choice and mark it `[ASSUMPTION: X — confirm]` inline
   - Couldn't even pick → list in plan's "Open questions" section
4. Write the plan to `.copilot/plans/<feature>.md` following the template.
5. Stop. Don't implement.

### If you are executing a step

1. Read the plan, find the requested step.
2. Implement it.
3. Run the validation checklist (lint, types, step-specific assertions). **Don't run pytest.**
4. **Update the step's Result section in the plan file** (mandatory — this is how step N+1 learns what step N did). Format and cap defined in the template.
5. Report a one-line summary in chat. Stop.

### If you are reviewing a completed plan

Read `docs/agent/plan-review.md` first, then follow it.

---

## 7. Extended handover (rare)

The per-step Result section in the plan handles normal step-to-step state transfer. Produce an extended handover (separate from the Result) only if:

1. You hit a problem the Result's 6-line cap can't capture
2. You discovered an issue affecting *multiple later steps*, not just the next one
3. The plan itself needs revision based on what you learned

When required, append to the plan file under a new `## Handover notes (step N → N+1)` heading. Cap at 200 tokens.

Default behaviour: just fill in the Result section. Don't generate extended handovers proactively.

---

## 8. Code conventions

- Type hints on all public functions
- No `print()` — use `logger`
- Tests mirror source: `src/services/foo.py` → `tests/unit/services/test_foo.py`
- Errors: typed exceptions from `src/errors.py`, never bare `Exception`
- `[FILL IN — naming conventions]`

---

## 9. What not to do

- ❌ Full-file `view` on files >500 lines
- ❌ Re-read files in same session
- ❌ Add dependencies without checking manifest
- ❌ Modify `legacy/`
- ❌ Suppress lint/type errors without an explanatory comment
- ❌ Disable failing tests
- ❌ Run `pytest` during implementation steps
- ❌ Generate extended handovers by default (see §7)
- ❌ Skip the Result section update (it's how the next step gets context)
- ❌ Prose explanations of changes — diff is the explanation
- ❌ Self-edit `AGENTS.md` or any file in `docs/agent/`. If you have a lesson worth capturing (a gotcha, a pattern, a workaround), propose it as text in chat with the format: *"Proposed addition to [file]: [content]"*. The user decides what gets added.

---

## 10. Escalation

- (Planning) >3 high-leverage questions emerge → stop, ask the user to narrow scope before planning
- (Executing) Step touches files outside the planned list → stop, report, don't proceed
- (Executing) Unrelated test starts failing → stop, report, don't "fix"
- (Executing) About to disable a test → ask first
- (Any) Need >300K of context → stop, ask user to narrow

---

## 11. Repo-specific patterns

`[FILL IN — 3-5 patterns, e.g.:]`
- DB access via `src/db/session.py`, never instantiate sessions directly
- Config via `src/config.py`, never `os.environ` directly
- HTTP clients use `src/http.py` wrapper
- API responses use `src/api/response.py:make_response()`
- Feature flags via `src/flags.py:is_enabled()`

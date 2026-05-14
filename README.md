# Agent file structure

Ten files total. Drop them in your repo at the paths shown.

## Always loaded (paid every turn)

- `AGENTS.md` → repo root
  Repo-wide rules. ~1.5K tokens — kept tight.
- `.github/copilot-instructions.md` → repo root
  Pointer file telling Copilot where the real instructions live.
- `.copilot/plans/<feature>.md` → one per active task
  Active plan. Use `_TEMPLATE.md` as the starting point.

## On-demand (read only when needed)

- `docs/agent/unit-tests.md` — *how* to write tests
- `docs/agent/test-coverage-strategy.md` — *what* to test
- `docs/agent/integration-tests.md` — when writing integration tests
- `docs/agent/refactor-playbook.md` — refactoring 3+ files
- `docs/agent/debugging.md` — non-trivial bug investigation
- `docs/agent/plan-review.md` — end-of-plan compliance review
- `docs/agent/repo-gotchas.md` — **always read at start of plan-driven implementation steps**; short, high-signal repo-specific traps

The agent reads these as tool calls when the current task matches. AGENTS.md §3 lists them so the agent knows they exist.

---

## The three roles

The system has three agent roles. Same agent type — the role is set by the prompt and what files are loaded.

| Role | Model | When |
|---|---|---|
| **Planner** | GPT-5.4 (Plan mode) | Once per feature/task |
| **Executor** | GPT-5.4 or 5.4 mini (Agent mode) | Once per step |
| **Reviewer** | GPT-5.4 (Agent mode) | Once at the end |

Each runs in its own session. State passes between them via the plan file (which is edited by all three).

---

## Full workflow

### Phase 1: Planning

In **Plan mode**, GPT-5.4. Prompt:

> "Read `AGENTS.md` and `.copilot/plans/_TEMPLATE.md`. I want to [feature description].
>
> Follow AGENTS.md §6 'If you are planning'. If any unknowns would fundamentally change the approach, ask up to 3 questions in chat *before* writing the plan. For low-leverage choices, make the call inline using `[ASSUMPTION]` markers. List anything you genuinely couldn't decide in the Open Questions section.
>
> Save the plan to `.copilot/plans/<feature>.md`. Do not implement. Stop after the file is written."

The planner may come back with questions. Answer them, then it produces the plan file.

### Phase 2: Plan review (by you)

Open the plan file in your editor:
- Answer or remove anything in **Open questions**
- Confirm or override each `[ASSUMPTION]`
- Tighten step boundaries if any look too big
- Sharpen step-specific assertions — make them runnable without pytest (see template examples)
- Drop anything from `Files in scope` that's actually unnecessary

The plan is yours now. Spending 10 minutes here saves money in execution.

### Phase 3: Step-by-step execution

Switch to **Agent mode**. Pick model based on the step:
- Reasoning-heavy / feature work → GPT-5.4
- Mechanical (rename, move, format) → GPT-5.4 mini
- Documentation-only steps → GPT-5.4 nano

For each step, prompt:

> "Read `.copilot/plans/<feature>.md` and implement step N only. Follow AGENTS.md §6 'If you are executing a step'. Run the validation checklist (lint, types, step-specific assertions). Do not run pytest. Update step N's Result section in the plan file. Stop after step N — do not start step N+1."

The executor:
1. Reads the plan + AGENTS.md
2. Implements the step
3. Runs validation
4. Updates the step's Result section in the plan file
5. Reports a one-line summary in chat

You review the diff and the Result. If clean, prompt for step N+1. If issues, fix and re-validate.

**Why this works across fresh sessions:** When step 3's executor starts in a new chat with empty context, it reads the plan and *sees step 1 and step 2's Result sections*. That's how it knows what was done. No conversation history needed.

### Phase 4: Test step

After all implementation steps show `Status: ✅ Complete`, switch to **GPT-5.4 mini** in a fresh session.

Prompt:

> "Read `.copilot/plans/<feature>.md`. Execute the Test step at the bottom of the plan, following its Reading strategy. Read `docs/agent/unit-tests.md` and `docs/agent/test-coverage-strategy.md`. Produce the coverage self-report. Update the Test step's Result section."

You review the coverage self-report (not the test code line-by-line). If gaps, prompt mini to fill specific ones.

### Phase 5: Plan review

Open a fresh chat. **Agent mode, GPT-5.4.** Prompt:

> "Read `.copilot/plans/<feature>.md` and `docs/agent/plan-review.md`. Run the review. Spot-check the implementation per the doc. Produce the Compliance Report. Update the Plan review step's Result section."

The reviewer reads the plan + spot-checks 2–3 files + produces a structured report. Cost is bounded — typically ~30K input, ~2K output, ~$0.10 on GPT-5.4.

If the recommendation is **ship** or **ship with follow-up** → merge. If **fix** → address blocking issues. If **replan** → start over (rare, but the cost of catching this here is far lower than catching it in production).

---

## Why the plan file is the source of truth

The plan accumulates state across all phases:
- Planner writes Goal, Approach, Steps, Open Questions, Assumptions
- User edits Open Questions, Assumptions, step boundaries
- Each executor writes the corresponding step's Result
- Test executor writes the Test step's Result + attaches the coverage self-report
- Reviewer writes the Plan review step's Result + attaches the Compliance Report

By the end, the plan is a complete record: intent, decisions, what got done, evidence it's correct. It's a much better artifact than scattered chat history. You can also commit it to the repo as living documentation.

---

## Lint vs tests — the cost split

- **Lint and types** (`ruff`, `mypy`) — cheap static checks, ~50 tokens output, deterministic. Run on every implementation step.
- **Tests** (`pytest`) — expensive (lots of output, iteration on failures). Deferred to a single dedicated step with a cheaper model.

The plan template's validation checklists embed this split. Don't break it.

---

## Why mini-written tests can still be robust

The risk with mini isn't bad code — it's *insufficient* coverage. Mitigated by:
1. Explicit coverage checklist in `test-coverage-strategy.md`
2. Plan's step-specific assertions act as the spec
3. Coverage self-report makes gaps visible
4. Mini reads actual changed code (per the Reading strategy), not just diffs

Mini's failure mode is "happy path only." The strategy doc forces breadth: branches, edge cases, exceptions, boundaries. The self-report is your review surface — if it shows gaps, you prompt mini to fill specific ones.

---

## Cost expectations

For a typical 5-step feature on a single repo:

| Phase | Model | Approx cost |
|---|---|---|
| Planning | GPT-5.4 | $0.10–0.30 |
| 5 implementation steps | mix of 5.4 / mini | $0.50–1.50 |
| Test step | GPT-5.4 mini | $0.05–0.15 |
| Plan review | GPT-5.4 | $0.10 |
| **Total per feature** | | **~$1–2** |

300 features/month at this rate = $300–600. Likely lower because not every task needs the full pipeline (small fixes can skip planning + review).

---

## Customising

`[FILL IN]` placeholders in `AGENTS.md` sections 1, 8, and 11 are repo-specific. Worth 30 minutes filling in properly.

The on-demand docs have generic conventions. Adjust to match your stack — but keep them concise. Each adds input cost when invoked.

---

## Capturing lessons over time

`docs/agent/repo-gotchas.md` is the file that grows with your repo's specific traps — things that wasted agent time once and shouldn't again. Examples: "PyMuPDF imports as `fitz`, mock at the import site," "integration tests need the dockerised DB up first," "ruff and mypy disagree on `TYPE_CHECKING` imports."

**How entries get added:**

The agent does *not* edit gotchas (or any agent doc) directly — see AGENTS.md §9. Instead:

1. **After a frustrating session**, prompt the agent: *"Based on this session, propose updates to `docs/agent/repo-gotchas.md` or `AGENTS.md` §11. Output proposed text only, max 3 entries, max 2 lines each."*
2. **During plan review**, the reviewer's Compliance Report sometimes surfaces lesson-worthy patterns — copy those out manually.
3. **You decide** what gets added. Edit the file yourself, or paste the agent's proposal into a chat with: *"Add this to repo-gotchas.md exactly as proposed."*

**How entries get pruned:**

The file has a hard 100-line cap. Quarterly, scan it:
- Anything dated >6 months ago? Verify it's still true. Remove if not.
- Anything now obsolete (the migration completed, the bug is fixed)? Remove.
- Anything generic enough to belong in a topic doc (`unit-tests.md`, etc.)? Move it there.

If you don't prune, the always-loaded budget creeps up. Pruning is the discipline that keeps this useful.

**Why this beats Copilot's built-in memory:**

- Deterministic — what's in the file is *always* loaded; Copilot's memory might or might not retrieve a relevant past chat
- Auditable — you can read and prune; you can't with Copilot's memory
- Portable — works in any tool (Claude Code, Cursor, future tools)
- Shareable — teammates joining can read your gotchas; they can't read your Copilot memory

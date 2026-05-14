# End-to-end walkthrough: LLM cost tracking feature

A complete trace through the planning → execution → test → review workflow with the exact prompts an expert would use. Every prompt below is optimised for cost (concise), clarity (unambiguous), and correctness (references the right files).

---

## The example

We're adding **per-call cost tracking to an LLM pipeline**. The pipeline already wraps Anthropic's API for document extraction. We want to:

- Record tokens used (input, cached input, output) per API call
- Compute USD cost based on per-model pricing
- Aggregate per pipeline run with a summary report
- Make it negligible-overhead and not break the pipeline if tracking fails

This is multi-step, has real ambiguity, has good test surface, and mirrors the cost-awareness theme of why we built this whole workflow in the first place.

**Stack assumed:** Python 3.12, Pydantic v2, the existing `anthropic` SDK client lives in `src/llm/client.py`.

---

## Phase 1 — Planning

### Prompt

Open Plan mode, GPT-5.4. Send:

> Read `AGENTS.md` and `.copilot/plans/_TEMPLATE.md` in full.
>
> Feature: Add LLM cost tracking to the pipeline. Each API call (Anthropic via `src/llm/client.py`) should record input/cached/output tokens and compute USD cost from per-model pricing. At the end of a pipeline run, surface a summary report (total cost, breakdown by model, by stage). Tracking failures must never break the pipeline.
>
> Follow AGENTS.md §6 "If you are planning":
> - If any unknowns would fundamentally change the approach, ask up to 3 questions in chat *before* writing the plan
> - Use `[ASSUMPTION]` markers inline for low-leverage choices
> - List anything you couldn't decide in the Open Questions section
>
> Save the plan to `.copilot/plans/llm-cost-tracking.md`. Do not implement. Stop after the plan file is written.

### What to expect

The planner reads AGENTS.md + template, then probably explores `src/llm/client.py` and a couple of pipeline files. Then it comes back with questions. A good planner asks something like:

> Before I write the plan, I have 2 high-leverage questions:
>
> 1. **Persistence:** Should cost records persist beyond the run (SQLite, log file, MLflow tracking) or stay in-memory and surface only at run-end? This changes whether we add a storage layer.
>
> 2. **Scope of "pipeline run":** Is a "run" a single invocation of `pipeline.execute()`, or does it span multiple invocations (e.g., a Databricks job)? This determines where the tracker is instantiated and how it's threaded through.
>
> I'm assuming USD only and per-model pricing hardcoded in a constants file (with a comment to update on price changes). Will flag these as `[ASSUMPTION]` in the plan.

### Your reply

Answer the questions concisely. Don't expand scope.

> 1. In-memory only for now, surface at run-end via a method on the tracker. We can add MLflow integration later as a separate plan.
>
> 2. A "run" = one `pipeline.execute()` invocation. Tracker instantiated at run start, passed via dependency injection through the pipeline context.
>
> Proceed.

### What the planner produces

The plan file gets written to `.copilot/plans/llm-cost-tracking.md`. Excerpt of the structure (not the full file):

```markdown
# Plan: LLM cost tracking

**Date:** 2026-05-02
**Plan author:** GPT-5.4
**Estimated implementation steps:** 4
**Estimated context per step:** ~80K tokens

## Goal
Track tokens and USD cost per Anthropic API call during a pipeline run,
surface a summary report at run-end. Must not break the pipeline if
tracking itself fails.

## Approach
- New `CostTracker` class instantiated per-run, passed via pipeline context
- Wrap existing Anthropic client to emit `LLMCallRecord` after each call
- Pricing constants in a dedicated module, update-on-price-change
- All tracking calls wrapped in try/except — log warning, never raise
- Summary report method returns a dataclass; caller decides how to surface it

## Open questions
None. (Answered in chat: in-memory only, run = one execute() call.)

## Assumptions
- `[ASSUMPTION]` USD only. Multi-currency would need an FX layer — out of scope.
- `[ASSUMPTION]` Per-model pricing hardcoded in `src/llm/pricing.py` with a header
  comment "Last verified: <date>". Override: switch to a config-driven pricing if
  you want runtime override.
- `[ASSUMPTION]` Cached input tokens billed at the cached rate per Anthropic's
  current pricing. Override: confirm against current API docs if pricing has
  changed.

## Files in scope

**Will modify:**
- `src/llm/client.py` — wrap with tracking emission
- `src/pipeline/context.py` — add tracker to context
- `src/pipeline/runner.py` — instantiate tracker, surface report

**Will create:**
- `src/llm/cost_tracking.py` — `CostTracker`, `LLMCallRecord`, `RunSummary`
- `src/llm/pricing.py` — per-model pricing constants

**Will read for context only:**
- `src/llm/__init__.py` — to confirm export structure

## Implementation steps

### Step 1: Pricing module + data models
[full step detail with validation checklist and Result section]

### Step 2: CostTracker class
[...]

### Step 3: Client wrapping
[...]

### Step 4: Pipeline integration
[...]

## Test step
[...]

## Plan review step
[...]
```

---

## Phase 2 — Your review

Open the plan file in your editor. You're looking for:

1. **Open questions:** Any left? Answer them inline or in chat.
2. **Assumptions:** Each one says "Override: ..." Confirm or override.
3. **Step boundaries:** Are any too big? A step that touches >3 files or has >5 assertions is probably two steps.
4. **Step assertions:** Are they runnable without pytest? Look for `python -c "..."` or `rg "..."` style. Reject vague ones like "function works correctly."
5. **Files in scope:** Does anything look out of place? Any dependency you don't want?

For our example, you might:

- ✅ Accept all three assumptions (USD only is fine, hardcoded pricing is fine, cached rate is correct as of today)
- ✏️ Tighten step 3's assertion from "client emits records correctly" to `python -c "from src.llm.client import call_with_tracking; tracker = MockTracker(); call_with_tracking(tracker, 'hello'); assert len(tracker.records) == 1"` — runnable, deterministic
- ✏️ Add an explicit out-of-scope note: "Streaming responses (token counts arrive differently) — track in a follow-up plan"

Save. Now you're ready to execute.

---

## Phase 3 — Step-by-step execution

### Step 1 prompt (GPT-5.4)

Switch to Agent mode, GPT-5.4 selected (this step has reasoning — Pydantic schema design).

> Read `.copilot/plans/llm-cost-tracking.md`. Implement step 1 only.
>
> Follow AGENTS.md §6 "If you are executing a step":
> - Run the validation checklist (lint, types, step-specific assertions)
> - Do not run pytest
> - Update step 1's Result section in the plan file before stopping
>
> Stop after step 1. Do not start step 2.

### What you'll see

The agent:
1. Reads the plan
2. Creates `src/llm/pricing.py` with per-model pricing constants
3. Creates `src/llm/cost_tracking.py` with `LLMCallRecord` and `RunSummary` Pydantic models
4. Runs `ruff check .` — passes
5. Runs `mypy src` — passes
6. Runs the step-specific assertion (e.g., `python -c "from src.llm.cost_tracking import LLMCallRecord; LLMCallRecord(model='x', input_tokens=1, cached_input_tokens=0, output_tokens=1, usd_cost=0.001)"`) — passes
7. Updates the plan file's Step 1 Result section
8. Reports: "Step 1 complete. New files: pricing.py, cost_tracking.py."

### What the Result section looks like after

```markdown
**Result:**
- Status: ✅ Complete
- Files actually changed: src/llm/pricing.py (new), src/llm/cost_tracking.py (new)
- Decisions: Used Pydantic v2 BaseModel with frozen=True for LLMCallRecord (immutable records prevent accidental mutation during aggregation). RunSummary is a regular dataclass since it's computed, not validated.
- For next step: LLMCallRecord and RunSummary importable from src.llm.cost_tracking. Pricing constants from src.llm.pricing.PRICING_USD_PER_MTOK.
```

You review the diff. Looks good. Move to step 2.

### Step 2 prompt (GPT-5.4)

Same model — designing a thread-safe accumulator class still benefits from the smarter model.

> Read `.copilot/plans/llm-cost-tracking.md` and check Step 1's Result section. Implement step 2 only.
>
> Follow AGENTS.md §6. Update step 2's Result section in the plan file. Stop after step 2.

Note: shorter prompt. AGENTS.md is cached. The agent already knows the rules.

### Step 3 prompt (GPT-5.4 — reasoning involved in error handling)

This step wraps the API client with try/except for all tracking calls. Worth using GPT-5.4 because the failure modes need careful thought.

> Read `.copilot/plans/llm-cost-tracking.md`, check Steps 1 and 2 Results. Implement step 3 only.
>
> Reminder from the plan: tracking failures must never break the pipeline. Every tracker call must be wrapped in try/except with a logged warning. Verify this in the step-specific assertions.
>
> Update step 3's Result. Stop after step 3.

The reminder about tracking-must-never-break is a small insurance against the agent forgetting a critical constraint. Worth the few tokens.

### Step 4 prompt (GPT-5.4 mini — mostly mechanical wiring)

Step 4 is plumbing: instantiate tracker, thread through context, surface report. The shape is dictated by step 1–3. Mini handles this.

> Read `.copilot/plans/llm-cost-tracking.md`, check Results from steps 1–3. Implement step 4 only.
>
> This is mostly mechanical integration — instantiate the tracker, pass it through the existing pipeline context pattern, call `summary()` at run-end. Don't redesign existing code.
>
> Update step 4's Result. Stop after step 4.

The "don't redesign existing code" line is critical for mini — it sometimes "helpfully" refactors adjacent code. Cuts off that failure mode.

### What if a step fails validation?

Two cases:

**Case A: Lint or type error the agent can fix.** Re-prompt:

> Validation failed on step 3. Fix the lint/type issues only. Don't change the implementation logic. Re-run validation, then update the Result section.

**Case B: Step-specific assertion fails because the implementation doesn't match the plan.** Don't let the agent "fix" by changing the assertion. Re-prompt:

> Validation failed: assertion "[X]" doesn't hold. The implementation needs to change to satisfy the plan, not the other way around. If you believe the plan's assertion is wrong, stop and explain why — do not modify either to make them match.

This is a common failure mode worth pre-empting. The agent's tendency is to "make the test green" by weakening the test.

---

## Phase 4 — Test step

After all four implementation steps show `Status: ✅ Complete`, open a fresh chat. Switch to **GPT-5.4 mini**.

### Test step prompt

> Read `.copilot/plans/llm-cost-tracking.md`, `docs/agent/unit-tests.md`, and `docs/agent/test-coverage-strategy.md` in full.
>
> Then read the specific functions/classes added in steps 1–4. Use the Reading strategy in the plan's Test step section — don't read entire files unless they're small. The relevant changed code is in:
> - `src/llm/cost_tracking.py` (new — read in full)
> - `src/llm/pricing.py` (new — read in full)
> - `src/llm/client.py` (modified — read only the wrapped call function)
> - `src/pipeline/context.py` (modified — read only the tracker injection)
> - `src/pipeline/runner.py` (modified — read only the tracker instantiation and summary call)
>
> Execute the Test step per the plan. Treat each step's step-specific assertions as part of the spec — every assertion must have ≥1 corresponding test.
>
> Apply the mandatory coverage checklist from `test-coverage-strategy.md`. Pay specific attention to: tracking-failure-doesn't-break-pipeline (try/except behaviour — test that an exception in the tracker doesn't propagate), cached vs uncached token cost calculation, empty run summary (no calls made), and concurrent calls if applicable.
>
> Run `pytest tests/unit -x`. Fix failures in test code only. If a real bug surfaces in production code, stop and report — do not fix it.
>
> Produce the coverage self-report. Update the Test step's Result section in the plan file.

The pre-listed file paths in the prompt save mini from having to figure out what changed — saves input cost on exploration.

### Coverage self-report (what mini produces)

```
Coverage self-report for llm-cost-tracking:
- Functions tested: LLMCallRecord validation, RunSummary aggregation,
  CostTracker.record/summary/reset, call_with_tracking wrapper,
  pricing.compute_cost
- Plan assertions covered: 12 of 12
- Branches tested: 18 (all if/elif/else paths in compute_cost,
  CostTracker.record, summary aggregation)
- Edge cases covered:
  ✅ None inputs (where types permit)
  ✅ Empty run (zero calls)
  ✅ Single call
  ✅ Many calls (100+)
  ✅ Boundary: zero tokens
  ✅ Cached-only call
  ✅ Tracker exception during record (verifies no propagation)
- Dependencies mocked: anthropic.Anthropic.messages.create, logger.warning
- Skipped from checklist:
  - Unicode in model names (pricing keys are ASCII-only constants — N/A)
  - Concurrent access (CostTracker is documented as single-threaded per-run — N/A)
- Known gaps: None
```

You review the self-report. 12 of 12 assertions covered, branches tested, no critical gaps. Glance at the test code (it's mini-written, mostly templated). Looks fine. Move on.

---

## Phase 5 — Plan review

Open another fresh chat. **Agent mode, GPT-5.4** (worth the smart model — review is high-leverage).

### Plan review prompt

> Read `.copilot/plans/llm-cost-tracking.md` in full (including all Result sections) and `docs/agent/plan-review.md`.
>
> Run the review per the doc. Spot-check 2–3 files following the doc's guidance — don't read every changed file in full. Suggested spot-checks:
> - The single most central file (likely `src/llm/cost_tracking.py`)
> - One file you'd expect to change but might not have (verify the pipeline runner actually calls `summary()`)
> - Verify `Out of scope` items (streaming responses) are genuinely untouched
>
> Produce the Compliance Report in the exact structure defined in `plan-review.md`. Update the Plan review step's Result section in the plan file.

### Compliance Report (what the reviewer produces)

```markdown
# Compliance Report — llm-cost-tracking.md

**Date:** 2026-05-02
**Reviewer model:** GPT-5.4
**Reviewer context:** ~28K tokens read

## Step compliance
✅ All 4 implementation steps show Complete with matching file lists
- Step 3's Result notes a decision to log warnings via the existing
  pipeline logger (consistent with §11 of AGENTS.md)

## Goal compliance
✅ Per-call records emitted, USD cost computed, run-end summary
available via `runner.cost_summary()`. Tracking failures wrapped in
try/except as required.

## Scope compliance
⚠️ One minor issue
- `src/llm/__init__.py` was modified (added export for CostTracker).
  Plan listed it as "read for context only." Net change is correct
  and consistent with the plan's intent, but technically out of the
  declared scope. Recommend updating the plan retroactively or noting
  in commit message.

## Question and assumption hygiene
✅ All 3 assumptions confirmed by user. Open Questions section is empty.

## Test compliance
✅ Coverage self-report shows 12/12 assertions covered, 18 branches
tested, no critical gaps. Spot-checked one test file
(test_cost_tracking.py) — structure and assertions look correct.

## Quality smell-checks
✅ No commented code, no TODOs, no print/debugger statements.
Imports clean. Type hints present on all new public functions.

## Recommendation

ship with follow-up

**Follow-up items:**
- Low priority: update plan retroactively to include `__init__.py`
  in modified files, or capture the convention "barrel exports
  always implied" in AGENTS.md §11
- Future plan: streaming response token tracking (out-of-scope here)
- Future plan: MLflow integration for cost record persistence
  (deferred per chat answer during planning)
```

The "ship with follow-up" recommendation tells you: merge it, but capture two follow-up plans. The minor scope deviation is noted but not blocking.

---

## Quick-reference prompt cheat sheet

Copy-paste-ready prompts. Replace `<feature>` with your feature name.

### Planning (GPT-5.4, Plan mode)

```
Read AGENTS.md and .copilot/plans/_TEMPLATE.md in full.

Feature: [one-paragraph description, including non-functional
requirements like "must not break X" or "must be backwards-compatible"]

Follow AGENTS.md §6 "If you are planning". Ask up to 3 questions in
chat for high-leverage unknowns. Use [ASSUMPTION] markers inline for
low-leverage choices. List anything you couldn't decide in Open
Questions.

Save the plan to .copilot/plans/<feature>.md. Do not implement.
Stop after the plan file is written.
```

### Step execution — first step (GPT-5.4 or mini per nature of step)

```
Read .copilot/plans/<feature>.md. Implement step 1 only.

Follow AGENTS.md §6 "If you are executing a step". Run the validation
checklist (lint, types, step-specific assertions). Do not run pytest.
Update step 1's Result section in the plan file before stopping.

Stop after step 1. Do not start step 2.
```

### Step execution — subsequent steps (shorter, AGENTS.md is cached)

```
Read .copilot/plans/<feature>.md, check prior step Results.
Implement step N only.

Follow AGENTS.md §6. Update step N's Result. Stop after step N.
```

### Step execution — when adding a critical reminder

```
Read .copilot/plans/<feature>.md, check prior Results. Implement
step N only.

Reminder from the plan: [the one critical constraint that must not
be forgotten — e.g., "tracking failures must never break the
pipeline"]. Verify this holds in the step-specific assertions.

Follow AGENTS.md §6. Update step N's Result. Stop after step N.
```

### Step execution — mechanical wiring step (GPT-5.4 mini)

```
Read .copilot/plans/<feature>.md, check prior Results. Implement
step N only.

This is mechanical integration — [one line on what the integration
involves]. Don't redesign existing code.

Follow AGENTS.md §6. Update step N's Result. Stop after step N.
```

### Re-running a failed step (lint/type fix)

```
Validation failed on step N. Fix the lint/type issues only. Don't
change the implementation logic. Re-run validation, then update the
Result section.
```

### Re-running a failed step (assertion fail)

```
Validation failed: assertion "[X]" doesn't hold. The implementation
needs to change to satisfy the plan, not the other way around. If
you believe the plan's assertion is wrong, stop and explain why —
do not modify either to make them match.
```

### Test step (GPT-5.4 mini, fresh session)

```
Read .copilot/plans/<feature>.md, docs/agent/unit-tests.md, and
docs/agent/test-coverage-strategy.md in full.

Then read the specific functions/classes added or changed in steps
1–N. Use the Reading strategy in the plan's Test step. The relevant
changed code is in:
- [file 1] (new/modified — what to read)
- [file 2] (new/modified — what to read)
- [file 3] (new/modified — what to read)

Execute the Test step per the plan. Apply the mandatory coverage
checklist from test-coverage-strategy.md. Pay specific attention to:
[1–3 specific behaviours that need careful coverage].

Run pytest tests/unit -x. Fix failures in test code only. If a real
bug surfaces in production code, stop and report — do not fix it.

Produce the coverage self-report. Update the Test step's Result
section in the plan file.
```

### Plan review (GPT-5.4, fresh session)

```
Read .copilot/plans/<feature>.md in full (including all Result
sections) and docs/agent/plan-review.md.

Run the review per the doc. Spot-check 2–3 files. Suggested:
- [most central changed file]
- [a file you'd expect to change but might not have — to verify
  it's correctly handled]
- Verify Out of scope items remain untouched

Produce the Compliance Report in the exact structure defined in
plan-review.md. Update the Plan review step's Result section.
```

### Filling test coverage gaps after review

```
The Compliance Report flagged gaps in test coverage for [function
or behaviour]. Add tests for [specific gaps] following
test-coverage-strategy.md's coverage checklist. Run pytest. Update
the Test step's Result with the additional coverage.
```

---

## Edge cases

### What if planner asks more than 3 questions?

The planner is over-asking. Reply:

> You've asked too many questions. Pick the single most important one — the answer that would change the approach the most — and ask only that. For everything else, make a reasonable choice and flag with `[ASSUMPTION]`.

This tightens behaviour without changing the prompt every time.

### What if a step's Result says "Partial" or "Blocked"?

Don't proceed to the next step. Read the Result, decide:

- **Partial because the agent ran out of context:** start a fresh session, prompt for the same step with stronger scope: "Step N is partial. Complete only the [specific remaining work] in this session. Do not redo what's already done. Update Result."

- **Partial because the agent discovered the step is bigger than planned:** the plan needs revision. Open the plan file, split that step into two. Re-prompt for the new step 1 of the split.

- **Blocked because of a real bug or design issue:** that's a planning failure. Read the agent's explanation, decide whether to fix the plan or accept a different approach. May require a fresh planner session.

### What if mid-execution you realise the plan is wrong?

Stop. Don't keep executing. Open a fresh Plan mode session:

> Read `.copilot/plans/<feature>.md`. Steps 1–N have been completed (see Result sections). Step N+1 onward needs revision because [reason].
>
> Revise the plan: keep the completed steps as-is (status frozen), rewrite the remaining steps. Save back to the same file. Do not re-execute completed steps.

Then continue execution from the new step N+1.

### What if the test step finds a real production bug?

The test agent should stop and report (per the prompt). Then:

1. Open the plan file. Add a "Bug discovered" section near the top.
2. Decide: fix the bug as part of this plan (add a step) or as a follow-up (mark plan complete with caveat).
3. If fixing here: prompt a GPT-5.4 executor to add and execute the new step.
4. Re-run the test step after.

### What if the review recommends "fix"?

Read the Compliance Report's "If fix" section. For each blocking issue, decide if it's a code change or a plan-revision change:

- **Code change** (e.g., missing a required behaviour): prompt an executor to address it. Treat each fix as a mini-step with its own validation.
- **Plan revision** (e.g., scope deviation that's actually correct): edit the plan to match reality, re-run review.

### What if the review recommends "replan"?

This is the rare case where so much went wrong that fixing in place is more expensive than replanning. Don't power through. Open a fresh planner session:

> The plan at `.copilot/plans/<feature>.md` has been reviewed and the recommendation is "replan" — see Compliance Report's "If replan" section. Read the Report and the existing plan.
>
> Write a fresh plan to `.copilot/plans/<feature>-v2.md` that addresses the issues identified. Reference the original plan only for what *not* to do. Do not implement.

Decide whether to revert the original implementation and start over, or build the v2 plan as a "fix the existing implementation" plan. Usually the first.

---

## What this whole flow protects against

| Failure mode | Where it's caught |
|---|---|
| Agent asks too many questions | Planning prompt's "max 3" rule |
| Agent makes silent assumptions | `[ASSUMPTION]` markers force visibility |
| Agent picks the wrong approach | Open Questions force user to decide |
| Step bigger than planned | Step's Result `Status: Partial` + replan |
| Step modifies wrong files | Validation checklist's "diff scope" check |
| Step "fixes" by weakening assertions | Pre-empted in the failure-handling prompt |
| Implementation drifts from goal | Plan review's Goal compliance check |
| Insufficient test coverage | Coverage self-report + reviewer's spot-check |
| Production bug introduced | Test step catches it (or review spot-check) |
| Scope creep across steps | Files in scope + reviewer's Scope compliance |
| Cost runs away | Per-step prompts cap context; fresh sessions reset |

Each failure mode has a specific defence. None of them are "trust the agent."

---

## Cost audit for this example

| Phase | Model | Approx input | Approx output | Approx cost |
|---|---|---|---|---|
| Planning | GPT-5.4 | 60K (plan + AGENTS + 3 file reads) | 4K | $0.21 |
| Step 1 | GPT-5.4 | 80K (plan + 2 reads + writes) | 5K | $0.28 |
| Step 2 | GPT-5.4 | 70K | 4K | $0.24 |
| Step 3 | GPT-5.4 | 90K (more files) | 4K | $0.29 |
| Step 4 | GPT-5.4 mini | 100K | 3K | $0.09 |
| Test step | GPT-5.4 mini | 60K | 8K (test code) | $0.08 |
| Plan review | GPT-5.4 | 30K | 2K | $0.10 |
| **Total** | | | | **~$1.30** |

Notes:
- 60–70% of input tokens are cache hits on AGENTS.md and the plan file → 10× discount applied
- The plan review at $0.10 is genuinely the highest-ROI ten cents in this whole pipeline

For comparison, doing this same feature with one continuous Plan mode → Implement button session would cost roughly **$3.50–5.00** because context accumulates monotonically and never resets, often crossing the 272K cliff. The discipline of separate sessions per phase is what saves money.

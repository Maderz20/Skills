# Refactor playbook

Read this only when refactoring across 3+ files.

## Principle

A refactor changes structure without changing behaviour. If behaviour changes, it's not a refactor — replan as a feature.

## Before starting

- Confirm tests cover the area being refactored. If coverage is thin, add tests *first* (separate plan) before refactoring.
- Identify the smallest atomic units the refactor can be split into.
- Plan should have one step per atomic change. Each step must keep the codebase compiling and lint-clean.

## Per-step rules

- One mechanical change per step (rename, extract, move, inline).
- Update all call sites in the same step as the definition change.
- Don't combine "rename" and "change signature" in one step.
- Run `ruff check` and `mypy` after each step. Don't run pytest during implementation steps — tests are deferred to the test step (see plan template).

## Common atomic operations

- **Rename:** find all references with `rg`, update definition + all call sites in one step
- **Extract function:** create new function, replace inline code with calls, in one step
- **Move file:** `git mv`, update imports, in one step
- **Change signature:** add new param with default first; remove old usage; remove default — three steps
- **Replace pattern:** identify all instances with `rg`, replace one cluster at a time

## Refactor-specific step assertions (no pytest)

For each refactor step, prefer assertions like:
- "Old name returns 0 results: `rg 'old_name' src/`"
- "New name is exported: `python -c 'from src.module import new_name'`"
- "Mypy passes — type system confirms call sites updated"
- "`ruff check` passes — import resolution confirms no broken references"

These give high confidence that mechanical changes are correct without needing tests to run.

## When to stop and replan

- Step requires reading >5 files you didn't anticipate
- Step requires changes outside the planned file list
- Type checker reveals a deeper structural issue
- The "atomic" step keeps growing as you implement it

In any of these cases, stop, document what you found in the step's Result section, and ask for replanning. Don't power through.

## Don't

- Don't refactor and add features in the same plan
- Don't refactor and reformat in the same plan (whitespace noise hides real changes)
- Don't refactor across module boundaries you don't understand — read the module first as a separate task

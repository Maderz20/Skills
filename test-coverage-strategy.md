# Test coverage strategy

Read this when invoked alongside `unit-tests.md` for any test-writing task. This file covers *what* to test. `unit-tests.md` covers *how*.

## Mandatory coverage checklist

For each function or method changed/added, write tests for **all** of the following that apply. Skip an item only if explicitly inapplicable, and note why in your test summary.

### Behavioural coverage
- [ ] Happy path with typical valid input
- [ ] Happy path with the simplest possible valid input (minimum viable)
- [ ] Each branch in the function (every `if`, `elif`, `else`, `match` arm)
- [ ] Each early return / guard clause
- [ ] Each raised exception — one test per `raise` statement
- [ ] Each loop: empty input, one item, many items

### Input-space coverage
- [ ] `None` / null where the type permits it
- [ ] Empty collection (`[]`, `{}`, `""`) where the type permits
- [ ] Boundary values: zero, negative, max int, very large strings
- [ ] Unicode / non-ASCII strings if the function handles strings
- [ ] Type-confused inputs if the function accepts `Any` or untyped dicts

### State coverage (if function has side effects)
- [ ] Function called once — verify state change
- [ ] Function called twice — verify idempotence or correct accumulation
- [ ] Function called with prior state — verify it doesn't clobber unrelated state

### Integration points
- [ ] Each external dependency mocked, with one test for success and one for failure of that dependency
- [ ] Timeouts, retries, and error responses for HTTP/DB calls

## Plan-specific coverage

The plan being executed lists step-specific assertions under each step's validation checklist. **Every assertion in the plan must have at least one corresponding test.** Treat the plan's assertions as the spec.

Example: if the plan says "function `parse_amount` rejects negative values," there must be a test `test_parse_amount_rejects_negative`.

## Coverage self-report

After writing tests, append a self-report to your final message in this format:

```
Coverage self-report for <feature>:
- Functions tested: <list>
- Plan assertions covered: <N> of <N>
- Branches tested: <N>
- Edge cases covered: None, empty, boundary, unicode (mark which apply)
- Dependencies mocked: <list>
- Skipped from checklist: <list with reason for each>
- Known gaps: <list, or "none">
```

This is the artifact for review. The user reviews this report, not all the test code line-by-line.

## What not to do

- Don't write a single "comprehensive" test that exercises 5 things. One assertion per test (or one tightly related group).
- Don't write tests that just call the function and assert it didn't raise — that tests almost nothing.
- Don't skip the "boring" cases (None, empty) because they seem trivial. They're where bugs hide.
- Don't write tests that depend on each other.
- Don't write tests that depend on external services (real HTTP, real DB connections, real time).
- Don't fabricate plan assertions that don't exist. If the plan doesn't specify behaviour, ask before testing it.

## When you genuinely don't know what a function should do

If you can read the function but can't tell from the code or the plan what its *intended* behaviour is (vs incidental implementation), stop. Report the ambiguity. Do not write tests that lock in current behaviour as if it were spec — that creates change-detector tests, not behaviour tests.

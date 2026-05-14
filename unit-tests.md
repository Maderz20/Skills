# Writing unit tests

Read this only when writing or modifying unit tests.

## Location and naming

- `src/services/foo.py` → `tests/unit/services/test_foo.py`
- One test file per source file
- Test function names: `test_<function>_<scenario>` — e.g., `test_parse_amount_handles_negative`

## Structure

- Use `pytest`, not `unittest`
- Arrange / Act / Assert with blank-line separation
- Fixtures in `tests/unit/conftest.py` for anything reused across files
- Local fixtures in the test file itself

## Coverage targets

- Every public function: at least one happy-path test
- Every branch in business logic: a test
- Every raised exception: a test that triggers it
- Edge cases: empty input, None, zero, boundary values, unicode where relevant

## What not to test

- Third-party library behaviour (assume it works)
- Pure pass-through getters/setters
- Generated code
- Constants

## Mocking

- Mock at the boundary: HTTP, DB, filesystem, time
- Use `pytest-mock`'s `mocker` fixture, not `unittest.mock` directly
- Don't mock the system under test — if you need to, the design is wrong

## Patterns

```python
def test_calculate_discount_applies_percentage(mocker):
    # Arrange
    cart = make_cart(items=[("apple", 100)])

    # Act
    result = calculate_discount(cart, percentage=10)

    # Assert
    assert result.total == 90
    assert result.discount == 10
```

## Don't

- Don't test private functions directly — test through the public interface
- Don't share state between tests
- Don't use `time.sleep()` — use `freezegun` or inject a clock
- Don't catch and ignore exceptions in test setup — let them fail loudly
- Don't write tests that depend on test execution order

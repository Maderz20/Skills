# Writing integration tests

Read this only when writing integration tests.

## Scope

Integration tests verify multiple components working together — typically with a real DB, real HTTP layer, real config loading. They are slow. Don't write one when a unit test would do.

Use an integration test when:
- The interaction between two layers is the thing being verified (handler → service → DB)
- You need to verify a migration or schema change
- You need to verify behaviour that depends on transaction boundaries
- An end-to-end user flow needs coverage

## Location

- `tests/integration/<area>/test_<flow>.py`
- One file per user-facing flow, not per source file

## Setup

- Use the `db` fixture from `tests/integration/conftest.py` — it gives a clean transaction per test
- Use the `client` fixture for HTTP-level tests
- Don't share state between tests, ever

## Test data

- Build data via factories (`tests/factories/`), not raw SQL or model construction
- Keep test data minimal — only the fields the test actually depends on
- Don't load fixtures from JSON/YAML files unless the data is genuinely large

## Patterns

```python
def test_create_order_persists_and_returns_id(client, db):
    # Arrange
    user = UserFactory.create(db=db)

    # Act
    response = client.post("/orders", json={"user_id": user.id, "items": [...]})

    # Assert
    assert response.status_code == 201
    assert db.query(Order).filter_by(user_id=user.id).count() == 1
```

## Don't

- Don't hit external services — mock at the network boundary
- Don't write integration tests for things unit tests already cover
- Don't use sleep/polling — if async, await properly
- Don't leave data in the DB between tests
- Don't write integration tests for happy paths only — include one or two failure modes

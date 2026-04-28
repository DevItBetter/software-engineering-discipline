# AI-Generated Test Smells

AI-generated test suites have characteristic failure modes. They look professional, follow conventions, have descriptive names, and pass — yet routinely miss real bugs because they're testing the wrong thing.

This file catalogs the patterns specifically. When reviewing AI-generated tests (or your own work that involved an AI assistant), look for these.

## The defining failure: tests that test the implementation, not the behavior

The single most common AI test smell. The AI sees the production code, reads what it does, and writes a test that asserts on those exact mechanics.

```python
# Production
def calculate_total(items):
    subtotal = sum(item.price for item in items)
    tax = subtotal * 0.08
    return subtotal + tax

# AI-generated test (bad)
def test_calculate_total():
    items = [Item(price=100)]
    result = calculate_total(items)
    # Tests the calculation as performed:
    assert result == 100 + 100 * 0.08

# A better test
def test_calculate_total_includes_tax():
    items = [Item(price=100)]
    assert calculate_total(items) == 108  # known correct value, computed independently

def test_calculate_total_handles_empty():
    assert calculate_total([]) == 0

def test_calculate_total_sums_multiple_items():
    assert calculate_total([Item(price=100), Item(price=50)]) == 162
```

The first test reuses the production formula to compute the expected value. It's a tautology — it can only fail if the multiplication operator itself breaks. The second set asserts on actual expected values, computed independently, including edge cases.

**The diagnostic:** if the test contains the same expressions as the production code, it's not testing the production code; it's restating it.

## Tests that mock everything they touch

```python
# Bad
def test_process_order():
    repo = Mock()
    payment = Mock()
    notifier = Mock()
    audit = Mock()
    
    service = OrderService(repo, payment, notifier, audit)
    service.process(order)
    
    repo.save.assert_called_once()
    payment.charge.assert_called_once()
    notifier.send.assert_called_once()
    audit.log.assert_called_once()
```

This test doesn't verify the order was processed correctly. It verifies that `process()` calls four methods. Refactoring `process()` to do the same work differently (different order, batched, conditional) breaks the test for no good reason. A bug that calls all four methods but with wrong arguments would also pass.

The fix: use real or fake implementations where possible (an in-memory repo, a test payment provider). Assert on observable outcomes — "the order is now in the saved state with the correct fields, the payment was made for the correct amount."

## Beautiful boilerplate that tests nothing

```python
def test_user_creation():
    """Test that a user is created successfully."""
    user = User(name="Alice", email="alice@example.com")
    assert user is not None
    assert user.name == "Alice"
    assert user.email == "alice@example.com"
```

This test exercises the constructor of a data class. If you broke the constructor (made it not store `name`), this test would catch it. But the constructor *can't* meaningfully break in this language — and if the data class is defined idiomatically (Pydantic, dataclass, attrs), the test is purely verifying that the language feature works.

Tests like this inflate coverage and create the illusion of thoroughness. Delete them.

## Asserting on log output

```python
def test_user_login(caplog):
    login(user)
    assert "User logged in" in caplog.text
```

Tests the log message, which is a brittle, implementation-detail string that's allowed to change. A real test of "user login" asserts on the *behavior* — the user is now in a logged-in state, the session token was issued, the last-login timestamp was updated.

Logs are diagnostic output for humans, not contract for programs.

## Tests that pass when the production code is reverted

The fundamental verification gate: mentally remove the production change. Does the test still pass?

If yes, the test is not testing the production change. It's testing something else (the framework, an unrelated part of the code, or nothing at all). This happens often when AIs generate "comprehensive" test suites — many tests get added but none of them meaningfully exercise the new code.

**Always run this check on AI-generated tests for a feature.** Pick the test that's supposed to verify the new feature; mentally undo the production change; ask whether the test would still pass.

## Tests that catch nothing because of trivial assertions

```python
def test_returns_a_list():
    result = get_orders()
    assert isinstance(result, list)
```

The function is annotated to return a list. The type system already enforces this. The test verifies a property the language already verifies, while ignoring the *content* of the list (which is what the function is actually responsible for).

## Hallucinated test fixtures

The AI invents data shapes that don't match the real domain. A test for a `Customer` includes fields that the real `Customer` doesn't have, or types that don't exist, or relationships that aren't real.

This often passes the type-check (because the AI invented the fields consistently in the test) but tests a parallel reality. The first time the test runs against real code, it fails — and the failure is in the fixture, not the production code.

**Always verify fixtures against the actual schema.** Generate them from real types where possible (e.g., factories that use the real model class).

## Tests that exercise the AI's misunderstanding

When the AI misreads the production code's intent, its tests will pin the misunderstanding into place.

Example: the production code rounds amounts to 2 decimal places using banker's rounding (round-half-to-even). The AI doesn't notice and writes tests assuming round-half-up. The tests pass on inputs where they happen to match, but fail on the boundary cases — and the AI's "fix" is to change the production code to match the tests.

**For AI-generated tests, always read the production code and the tests with equal care, and verify they actually agree on what the behavior should be.** Don't assume the AI got both right.

## Tests with no edge cases

A real bug-catching test suite covers boundaries: empty, null, max, unicode, locale, timezone, off-by-one. AI-generated suites tend to cover a happy-path example and skip the edges — which is exactly where bugs live.

When reviewing AI tests, ask: where are the empty-input tests? The boundary tests? The error-path tests? The tests that verify behavior under retry or concurrency? If they're missing, add them.

## Tests that are silently slow

The AI-generated test imports the entire framework, spins up a database, makes a real HTTP call, all to test a pure function. Slow tests are skipped in pre-commit hooks and developer test runs; they only surface in CI; they make the test suite a chore.

Mock the external boundary; use the real thing for everything internal.

## Snapshot tests of generated content

```python
def test_render_email():
    email = render_email(user, order)
    assert email == EXPECTED_EMAIL  # 200 lines of HTML
```

When this fails, the reviewer sees a 400-line diff (200 old, 200 new) and the temptation is to update the snapshot without reading. Now the test enforces nothing.

For complex outputs, assert on the *structurally relevant pieces*: "the email contains the order number," "the recipient is the customer's email," "the subject contains the order status." Not the exact bytes.

## How to review AI-generated tests

For each test:

1. **Read the test name.** Does it describe a behavior? If not, the test is unfocused.
2. **Read the assertion(s).** Does the assertion meaningfully verify the named behavior?
3. **Mentally remove the production change.** Does the test still pass? If yes, the test is decoration.
4. **Check for hallucinated APIs/fields.** Does the test reference real things?
5. **Check the mock surface.** Are mocks at the boundary of external systems, or in the middle of internal collaborators?
6. **Look for missing tests.** Empty input? Boundary? Error path? Concurrency? If they're absent for a feature where they matter, the suite is incomplete.

Treat AI-generated tests with at least as much scrutiny as AI-generated production code. The failure mode (false confidence) is more dangerous than for production code, because the tests *look* like the safety net.

# Test Design Patterns

Concrete patterns for the test design choices that come up in every project.

## Fixture management

Tests need data to operate on. How that data is set up determines whether the test suite is fast and clear or slow and tangled.

**Builder pattern (preferred default).** A test builder constructs a valid test object with sensible defaults; the test customizes only what's relevant.

```python
order = OrderBuilder().with_amount(100).with_status("pending").build()
```

Benefits: each test only spells out what's *relevant to that test*, so the test reads as a specification. New required fields can be added in one place (the builder's defaults).

**Object Mother (acceptable for shared fixtures).** Named factory functions for common test cases.

```python
order = an_unpaid_order()
```

Benefits: simple, named scenarios. Drawback: the named scenarios start to multiply (`an_unpaid_order_in_eur_with_tax`...) — at which point switch to Builder.

**Inline fixtures (acceptable for simple cases).** Construct in the test body. Fine for trivial cases; doesn't scale.

**Anti-patterns:**

- A single `setUp` that constructs many objects shared across tests. Creates hidden coupling — changes to setup break unrelated tests.
- A "test database fixture" with hundreds of pre-loaded rows used by every test. Tests now depend on data they don't reference.
- Fixtures loaded from JSON/YAML files when a builder would do. The data is opaque to the test reader.

## Mocking discipline

Mocks are a tool with a narrow, specific use: **isolate your code from a slow, non-deterministic, or unavailable dependency**. Used outside that scope, they create test suites that pass without verifying anything.

**Use a mock when:** the dependency is the network, the disk (in some cases), the system clock, the random number generator, an external service, a third-party API.

**Don't mock when:** the dependency is a pure function, an in-memory data structure, a value object, or any internal collaborator that's fast and deterministic. Use the real thing.

**Mock at the boundary, not in the middle.** If your service calls `OrderRepository` which calls `Database`, mock `Database` (or use a fake/in-memory variant), not `OrderRepository`. Mocking the internal collaborator means the test verifies your function called `OrderRepository.save()` correctly — not that the order was actually saved correctly. The latter catches more bugs.

**Prefer fakes to mocks.** A fake is a real implementation that's just simpler — an in-memory database, a fake HTTP client that returns canned responses. A fake supports many tests with little setup; a mock has to be configured per test. Pre-built fakes (in-memory SQLite, Wiremock, etc.) are usually a better choice than raw mocks.

**Verify behavior, not interaction.** "The order was saved" (you can read it back) beats "save() was called once with these arguments." The first survives refactoring; the second locks the implementation.

**The most-broken kind of test:**

```python
def test_save_order():
    repo = Mock()
    service = OrderService(repo)
    service.save(order)
    repo.save.assert_called_once_with(order)
```

This test verifies that `OrderService.save` calls `repo.save`. It does NOT verify the order was correctly saved, validated, transformed, or anything else. The test will pass even if `OrderService.save` is `def save(self, order): self.repo.save(order)` and does nothing else useful.

The fix: use a fake repo that actually stores the order, then read it back and assert on what's there.

## Parametrized tests

When you want to verify the same behavior across many inputs (boundary cases, equivalence classes), use parametrized tests. The test becomes a table.

```python
@pytest.mark.parametrize("input,expected", [
    ("",         False),  # empty
    ("a",        False),  # too short
    ("a@",       False),  # missing domain
    ("a@b",      False),  # missing TLD
    ("a@b.co",   True),   # minimal valid
    ("a" * 65 + "@b.co", False),  # local part too long
])
def test_is_valid_email(input, expected):
    assert is_valid_email(input) == expected
```

Benefits: the test name + parameters tell you exactly what failed. Adding a case is one line. The boundary structure of the input space is visible in the table.

Anti-pattern: a parametrized test where the parameter name is `case_1`, `case_2`. Name the cases.

## Property-based testing

Examples test a few specific points. Property-based tests assert a *property* and let the tool find counterexamples.

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)

@given(st.lists(st.integers()))
def test_sort_preserves_length(xs):
    assert len(sorted(xs)) == len(xs)

@given(st.lists(st.integers()))
def test_sort_produces_ordered_output(xs):
    result = sorted(xs)
    assert all(result[i] <= result[i+1] for i in range(len(result)-1))
```

Properties to look for in your domain:

- **Round-trip:** `decode(encode(x)) == x`.
- **Idempotency:** `f(f(x)) == f(x)`.
- **Inverse:** `f(g(x)) == x`.
- **Commutativity / associativity** for math operations.
- **Invariant preservation:** "after any operation on this collection, it remains sorted."
- **Equivalence:** "the new fast implementation matches the old slow implementation on all inputs."

Property-based tests find bugs you didn't think of. Cost: each test runs many iterations (slower); writing good properties takes practice; failures need to be reproducible (most tools support seeding).

## Contract testing

When two services depend on each other and are deployed independently, neither side can fully test the integration alone. Contract testing splits the verification:

1. **The contract** — a shared description of what one side produces and the other consumes (usually a Pact or OpenAPI document, with examples).
2. **Provider tests** — the provider runs the contract against itself and verifies it produces what the contract specifies.
3. **Consumer tests** — the consumer runs against a fake that implements the contract and verifies it handles the contract's responses correctly.

Failures show up *before deployment*: if the provider drops a field the consumer relies on, the provider's contract test fails before merge.

When to use: services owned by separate teams, services deployed independently, microservices in general.

When not to use: services that are deployed together (they can be tested as a single integration test).

## Mutation testing

The "test the tests" technique. A mutation testing tool changes the production code in small ways and runs the test suite.

If a mutation survives (test suite still passes), one of two things is true:
- The test suite has a gap — write a test that catches the mutation.
- The mutation is equivalent (the change doesn't matter) — mark it equivalent.

The mutation score (% of mutations the suite catches) is a more honest signal than line coverage. A 90% line-covered codebase often has a 50% mutation score: many lines are touched by tests but not actually verified.

Cost: mutation testing is slow (it runs the suite once per mutation). Run it periodically, on the most critical modules, not on every commit.

## Flaky tests

A flaky test is a test that sometimes passes and sometimes fails on the same code. Flaky tests poison the entire suite — once developers learn to ignore failures, they ignore real ones too.

The default response to a flaky test: **quarantine immediately** (skip it from CI, file a ticket), then **fix or delete within a deadline** (a week or two). Don't leave a flaky test in CI; the cost compounds.

Common causes:
- **Timing.** `time.sleep()`, race condition, real clock dependency. Fix: deterministic clocks, real synchronization, retries with proper conditions.
- **Order dependence.** Test A leaves global state that test B depends on. Fix: each test sets up and tears down its own state.
- **External dependencies.** Real network call, real database with shared state. Fix: fake or isolate.
- **Random data without a seed.** Property-based test that occasionally hits a bug. Fix: seed for deterministic reproduction; the failure is real and worth fixing.
- **Resource contention.** Test depends on a port, file, or other singleton. Fix: parallelism-safe setup (random ports, temp directories).

A test that passes "most of the time" is broken. Treat it as such.

## Test naming

A test's name should read as a behavior specification. Examples of good names:

- `returns_404_when_user_not_found`
- `cancels_order_after_30_days_unpaid`
- `propagates_exception_when_repository_fails`
- `does_not_send_email_when_user_opted_out`
- `idempotent_under_concurrent_calls`

Bad:
- `test_get_user`
- `test_2`
- `test_happy_path`
- `test_edge_cases`
- `test_OrderService_save_returns_correct_value_when_input_is_valid_and_database_is_available_and_user_has_permission` (kitchen sink)

Use the project's casing convention (`test_<behavior>`, `it_should_<behavior>`, `<MethodName>_<scenario>_<expected>` — pick one and stay consistent).

## Test independence

Each test must:

- Set up exactly the state it needs, from scratch or a known baseline.
- Clean up after itself (or rely on the framework's per-test rollback).
- Not depend on other tests having run first.
- Not depend on running in a specific order.

A test suite where this holds can run in parallel and can be sliced (run just the tests for one module). A suite where this doesn't hold has a hidden dependency graph and will eventually flake.

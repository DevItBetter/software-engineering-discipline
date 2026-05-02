---
name: testing-discipline
description: "Senior-engineer guidance on what to test, how to test it, and the test design choices that decide whether a test suite is an asset or a liability. Use this skill whenever the task involves writing tests, evaluating an existing test suite, deciding what to test for a new feature, debating mocks vs fakes vs integration tests, choosing between unit / integration / contract / property / end-to-end tests, judging test coverage, fixing flaky tests, or asking \"is this test actually useful.\" Also use it whenever an AI is generating tests — AI-generated tests have characteristic failure modes (testing the implementation, tautological assertions, beautiful-but-shallow boilerplate) that this skill catches."
---

# Testing Discipline

A test suite is either an asset or a liability. The line between them is thin: the same test, shifted slightly in design, can either give you the confidence to refactor or trap the codebase in its current shape. This skill is about ending up on the asset side.

The default failure mode for AI-generated tests (and for hurried human-generated tests) is a beautiful-looking suite that asserts on implementation details rather than behavior. It passes today, breaks on every refactor, and provides false confidence because it doesn't actually catch the bugs that matter. Recognizing and avoiding this pattern is the single most valuable test-design discipline.

## What tests are for

Tests serve four purposes, in priority order:

1. **Detecting regressions.** When you change the code, tests tell you what broke. This is the *primary* purpose; everything else is bonus. A test that doesn't catch a real regression earns nothing.
2. **Specifying behavior.** A good test reads like a specification. "When the order is older than 30 days and unpaid, it is auto-cancelled" is a behavior; the test that exercises that path is its checkable form.
3. **Enabling refactoring.** Confidence to restructure code requires tests on the *behavior* (which doesn't change) rather than the *structure* (which does). Tests on structure block refactoring instead of enabling it.
4. **Documenting intent.** A test demonstrates how the code is meant to be used.

A test that doesn't serve at least the first purpose is dead weight. A test that asserts on implementation details (private methods, internal state) actively *prevents* purpose 3.

## The single most important test design rule

**Test behavior, not implementation.** Equivalently: test what the function returns, the side effects it has, and the contract it honors — never test that it called this private helper, set this internal field, or took this code path.

The diagnostic: would this test still pass if I rewrote the function from scratch with the same input/output behavior? If yes, the test is a behavior test (good). If no, the test is locked to the current implementation (bad). It will break on every refactor and will *fail to catch* the kind of bug that changes the implementation while preserving the wrong behavior.

This is the rule that, more than any other, separates test suites that work from test suites that obstruct.

## What to test

### The "table-stakes" tests

For each public function or component:

- **Happy path with realistic data.** Not `add(1,2) == 3` placeholders; data that resembles what production sees.
- **Each documented edge case.** Empty, null, max-length, leading/trailing whitespace, unicode, duplicates, ordering, locale, timezone, currency precision, off-by-one boundary.
- **Each documented failure mode.** Invalid input → expected error. Unauthorized caller → expected error. Downstream failure → expected behavior (retry, propagate, fall back).
- **Each documented invariant.** If the function preserves a property (e.g., "the list remains sorted"), test that the property holds across operations.

### The "skipped by default" tests to add when a bug appears

Every time a real bug ships:

- **Add a regression test that fails on the bug-revealing input and passes after the fix.** This is non-negotiable. A bug that escapes once will escape again if there's no test pinning the fix.

### The tests AI-generated suites usually skip

These are the ones that catch real production failures and that lazy or AI-generated suites typically miss:

- **Concurrency tests.** What happens when two callers race? Does the lock work? Is the operation idempotent under retry?
- **Error-path tests.** What happens when the database is down? When the upstream returns malformed data? When the network times out partway through a write?
- **Boundary tests.** Exactly the boundary, not "a small number." Off-by-one bugs live at the boundary.
- **Tests with realistic data sizes.** Not 3-element lists. The N+1 query, the O(n²) algorithm, the unbounded-memory allocation only surface at scale.
- **Tests on actual data.** Anonymized samples from production. Real data has structure your synthetic data doesn't.
- **Tests of the absence of behavior.** "Does NOT call the email service when the user is opted out." Easy to forget; the bug is the side effect that shouldn't happen.

## What NOT to test

- **The standard library.** `assert dict["key"] == value` after `dict["key"] = value` tests the dict, not your code.
- **Private methods directly.** Test through the public interface. If you can't get good coverage that way, the public interface is wrong (or the private method is doing too much and should be public, or extracted).
- **Generated code, framework boilerplate.** Test what *you* wrote.
- **The mock you just configured.** Tests that only assert "the mock was called with X" often test your mock setup, not your code's behavior. Boundary interactions can be observable behavior (email not sent, payment gateway called once, audit event emitted); assert them when the interaction is the contract.
- **Implementation details that should be free to change.** If the current implementation uses a hash map and the test asserts on hash-iteration-order, the test is wrong.

## The testing pyramid (and its critics)

The classic pyramid (Mike Cohn, popularized by Martin Fowler):

- **Many fast unit tests** at the base. Run in milliseconds, run on every save, tell you exactly what broke.
- **Fewer integration tests** in the middle. Run in seconds, exercise real interactions between components or with real backing services.
- **Very few end-to-end tests** at the top. Run in minutes, exercise the full system from the outside.

The pyramid optimizes for **feedback speed** and **failure locality** (when a unit test fails, you know exactly where; when an E2E test fails, you have to dig). It is still the default-correct shape for most applications.

The "ice cream cone" anti-pattern is the inverted shape — many slow E2E tests, few integration tests, very few unit tests. This is what you get when "we'll just test through the UI" wins over careful unit-test design. Symptom: a CI that takes hours, fails flakily, and doesn't tell you what's wrong when it does fail.

Modern variants worth knowing:

- **Honeycomb** (André Schaffer, microservices). Many integration tests, few unit tests, few E2E tests. Justified for thin-service architectures where most logic is at the integration boundary.
- **Trophy** (Kent C. Dodds, frontend). Static (TypeScript, ESLint) > unit > integration > E2E. The "static" layer catches a lot of frontend bugs cheaply.
- **Crab** / no fixed shape. The right shape depends on what your code actually does — pick test types based on what bugs each catches in *your* system.

The framework matters less than the principle: **test as low in the pyramid as you can get reliable feedback, and as high as you need to verify integration.**

## Test types you should know

### Unit tests

A unit test exercises one unit (function, class, module) in isolation. Fast, deterministic, focused. The bulk of your test suite.

The argument over "what's a unit": *isolated unit* (mock everything else) vs *sociable unit* (use real collaborators when they're cheap and pure). Sociable is usually better — testing a function with its real, fast collaborators in place exercises actual integration points and catches more bugs. Mock only at the boundary of slow/external/non-deterministic things.

### Integration tests

An integration test exercises real interactions between components — typically with a real database, real cache, real message queue, in a controlled environment (test container, in-memory variant, dedicated test instance). Slower than unit tests; catch a class of bugs that unit tests can't (SQL syntax errors, ORM lazy-loading issues, transaction boundaries, serialization round-trips).

### Contract tests

Both sides of a service boundary verify they conform to a shared contract. Provider tests verify the provider produces what the contract says. Consumer tests verify the consumer handles what the contract says. Pact is the dominant tool. Critical when services are owned by separate teams or deployed independently.

### Property-based tests

Instead of giving specific examples, *you specify a property* that should hold for all inputs (this is the hard part — finding the right invariant). The tool generates many random inputs, looks for one that violates the property, and *shrinks* (minimizes) the counterexample so the failing input is small and human-readable. Catches bugs you didn't think of.

Example property: "for any valid order, `total == sum(line_items.amount)`." The framework supplies thousands of orders; you supplied the law that must hold.

Especially valuable for: parsers, serializers, math functions, anything with algebraic properties (associativity, commutativity, idempotency).

Tools: Hypothesis (Python), QuickCheck (Haskell, Erlang), fast-check (JS/TS), proptest (Rust), jqwik (Java), Hedgehog (multiple).

### Mutation tests

The test of your tests. A mutation testing tool changes the production code in small ways (flips a `>` to a `>=`, removes a line, replaces a constant) and runs the test suite. If the tests still pass, the mutation "survived" — meaning your tests didn't catch a real bug that was just introduced.

Mutation testing is slow but reveals which tests are paper tigers. Tools: mutmut/cosmic-ray (Python), Stryker (JS/TS, .NET, Scala), PIT (Java).

### End-to-end tests

Exercises the whole system from outside, usually through the UI or the public API. Catches integration bugs nothing else catches. Slow, brittle, expensive to maintain. Use sparingly; reserve for critical user journeys.

### Snapshot / golden tests

Capture the current output and assert future runs produce the same. Useful for complex outputs (rendered HTML, generated SQL, formatted documents). Dangerous when snapshots are large — reviewers can't see what changed and just rubber-stamp the update. Keep snapshots small and inspectable.

### Smoke tests

A handful of fast tests that verify the system boots and the critical paths are wired up. Run before deeper test suites in CI. The "is this even worth running the rest" check.

### Performance tests / benchmarks

Verify performance characteristics. Distinct from correctness tests. Important for hot paths; expensive to keep stable; need a controlled environment to be meaningful. See `performance-engineering` skill.

## Anti-patterns to flag in review

- **Tests that mock the function under test.** Always wrong.
- **Tests that pass with the production change reverted.** They're not testing what they claim.
- **Tests that compute the expected value the same way the production code computes it.** Tautology.
- **Tests with `time.sleep()` to "let things finish."** Race condition camouflage. Use real synchronization.
- **Tests that depend on each other (order-dependent, shared mutable state).** Flake source.
- **Tests with snapshots larger than a screen.** Reviewer can't see what changed.
- **One giant test that exercises ten things.** When it fails, you don't know which thing broke.
- **Tests that assert on log output.** Brittle. Test the behavior, not the log.
- **Tests that import the entire framework, the database, and three services to test a pure function.** Wrong scope.
- **Mocks of the standard library, the language runtime, or the network stack at the wrong level.** Mocks are for the boundary between your code and *external* systems, not internal collaborators.
- **`@pytest.mark.skip` / `it.skip` / `xit` with no ticket and no expiration.** Skipped tests rot. Either fix or delete.
- **A test that always passes** (e.g., asserts `True is True`). Sounds absurd; happens.

## TDD — when it shines and when it doesn't

Test-Driven Development (write the test first, watch it fail, write the minimum code to pass, refactor) shines when:

- The behavior is well-specified.
- The unit under test has a clear interface.
- You're adding to an existing well-tested codebase.

It struggles when:

- The behavior is being discovered through experimentation. (Spike first, test later.)
- The interface is the open question. (Test-first locks you into the first interface you tried.)
- You're working in legacy code with no seams. (Add tests for what's there first; *then* TDD the changes.)

TDD is a tool, not a religion. Use it where it helps; don't use it where it hurts.

## Coverage

Code coverage is a *necessary but not sufficient* signal. High coverage with bad tests is worthless. Low coverage with good tests means there are real gaps.

Honest uses of coverage:

- **Coverage diff in CI.** New code should have new tests. A PR that adds 200 lines of code and 5 lines of test coverage is suspicious.
- **Identifying untested risky paths.** Coverage tools highlight which branches aren't exercised. The error-handling branches are usually the ones missing.
- **Setting a floor (e.g., "no merge below 80%").** Useful as a backstop; not a goal.

Dishonest uses:

- **Coverage as a goal.** Optimizing for coverage produces tests that touch code without verifying behavior.
- **Coverage as a quality measure.** A 95%-covered codebase with all assertion-free tests is worse than a 60%-covered codebase with sharp ones.

Branch coverage is more honest than line coverage. Mutation coverage is more honest than branch coverage.

## What good test design looks like

A test that earns its keep:

- **Has a name that reads like a behavior specification.** `test_returns_404_when_user_not_found` beats `test_get_user_2`.
- **Follows arrange-act-assert** (or the project's equivalent: given-when-then, etc.). The structure is visible.
- **Has minimal fixtures.** Only the data the test actually needs. A test that takes 50 lines of setup is testing too much.
- **Asserts one behavior.** Multiple assertions about one behavior is fine; multiple unrelated behaviors is one-test-per-behavior.
- **Fails for one reason.** When it fails, you know what broke.
- **Runs in milliseconds.** Anything slower is an integration or higher test, and should be marked accordingly.
- **Is independent.** Can run alone, in any order, in parallel.
- **Is deterministic.** Same input, same outcome, every time.
- **Catches the bug it claims to catch.** Verify by mentally removing the production code; the test should fail.

## Reference library

- `references/test-design-patterns.md` — patterns for fixture management, mocking discipline, parametrization, contract/property/mutation testing in depth.
- `references/ai-generated-test-smells.md` — failure modes specific to AI-generated tests, with examples.

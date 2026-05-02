---
name: error-handling-and-resilience
description: "How to handle errors, design for failure, and build systems that degrade gracefully. Use this skill whenever the task involves designing error handling, deciding what to catch vs. let propagate, designing retry / timeout / circuit-breaker behavior, choosing exceptions vs. return values, judging whether code is appropriately defensive (or over-defensive), reviewing for resilience, designing for at-least-once or exactly-once semantics, choosing between fail-fast and let-it-crash, or thinking about what happens when a dependency is slow / unavailable / broken. Also use it whenever AI-generated code is wrapped in defensive try/except blocks — most of those are wrong."

---

# Error Handling and Resilience

Most production incidents have an error-handling root cause. Either the code didn't handle a failure mode, or it handled it wrong: swallowed exceptions, retried into thundering herds, hid the error from on-call, retried non-idempotent operations and corrupted data.

This skill is about the discipline of treating errors as a first-class design concern, not as something you decorate the code with at the end.

## The two stances toward errors

There are two coherent stances. Mixing them produces incoherent systems.

### Stance 1: Defensive / fail-fast (the conventional approach)

Detect errors at every layer. Validate inputs. Wrap risky operations in try/except. Return error codes or raise typed exceptions. The system's safety comes from the local code being careful.

Works well when:
- Code runs in a single process with shared memory.
- Errors are mostly local (bad input, programmer error).
- Recovery is local (re-prompt the user, validate again).

### Stance 2: Let it crash / supervisor (Erlang / Joe Armstrong)

Don't write code that defends against errors it can't fix. Let the process crash. A supervising process notices, restarts the worker, and the system continues.

Works well when:
- Code runs as one of many isolated processes.
- Most errors are not locally recoverable (the database is down — you can't fix it inside the request handler).
- Restart is cheap (the worker is stateless or its state is recoverable).

Most production systems are *somewhere between* these. The honest version of "let it crash" for non-Erlang systems: handle errors at the *correct level* — the level that has enough context to do something meaningful — and let them propagate above and below that level.

The wrong middle ground (and the most common one): handling errors at *every* level, swallowing them at some, re-throwing at others, with no consistent strategy. This produces logs full of "WARN: failed to do thing, continuing anyway" and bugs that take days to track down.

## Where to handle each error

The general rule: **handle an error at the level that has the context to make a decision about it.**

- **Validation errors at the input boundary.** A user-supplied date that's malformed should produce a 400 at the API layer, not propagate as a `ValueError` from deep in the system.
- **Domain errors where the domain logic is.** "Order is in a state that doesn't allow this transition" is handled where the order's state machine lives.
- **Infrastructure errors at the infrastructure boundary** — typically with a retry, a fallback, or a clean propagation that the higher level knows what to do with.
- **Programmer errors should crash the process** (or the request, depending on isolation). Catching `NullPointerException` or `KeyError` and continuing is hiding a bug. Let it bubble; let the supervisor handle the restart.
- **Unknown errors at the top of the request lifecycle.** A handler-level catch that logs, returns a 500, and continues serving other requests. Specifically: not a `try: ... except Exception: pass` at every level; one at the top, with full context logged.

The diagnostic for over-defensive code: count the levels of try/except in a single call stack. More than two is usually a smell. The middle layers should propagate; only the outermost level (or a level with concrete recovery) should catch.

## What kind of error type

Three honest options. Pick one per error kind and use it consistently:

### Exceptions

Best for: exceptional conditions that the immediate caller usually can't handle and that should propagate. `OutOfMemory`, `ConnectionRefused`, `PermissionDenied` (from a system the caller has no insight into).

Anti-pattern: using exceptions for control flow. `try: thing.find_user(id) except UserNotFound: ...` — `find_user` should return `Optional<User>` (or `User | None`) and the caller should explicitly handle absence. Exceptions for normal-path branching are slow and obscure the flow.

### Return values / Result types

Best for: errors that are part of the function's normal vocabulary and that the caller is expected to handle. `Result<User, FindError>`, `Optional<User>`, `Either<DomainError, Order>`.

Languages with built-in support (Rust `Result`, Haskell `Either`, Go's error returns, TypeScript with `Result` libraries) make this natural. Languages without built-in support can adopt it — it's a discipline more than a language feature.

The caller is forced (by the type system or by convention) to deal with the error case at the call site. Errors don't escape unhandled.

### Crash / panic / unrecoverable

Best for: programmer errors (precondition violation, impossible state). The point isn't to handle the error; the point is to make the bug visible immediately so it's fixed.

Examples: an assertion failure, an exhaustive match that hits a "this should be impossible" case, a null where the type should have prevented null.

Don't try to "recover" from these. The state is corrupt; continuing makes it worse.

## Designing errors for the caller

An exception or error type is a contract. Design it deliberately.

- **Be specific.** `InvalidEmailFormat` beats `ValidationError` beats `Exception`. The caller should be able to handle some error kinds and propagate others without a giant catch-all.
- **Include the context.** The error should carry the data needed to debug or recover. "Could not find user" — which user? Include the id. "Database query failed" — what query, what arguments?
- **Don't leak internals.** The error message that goes to the *user* shouldn't include database table names, file paths, or stack traces. The error logged for the *operator* should include all of them.
- **Be stable.** Once an error type is part of your public API, callers will catch it. Removing it is a breaking change.
- **Distinguish retryable from non-retryable.** A transient failure (timeout, 503) is retryable. A permanent failure (404, 401) is not. Encode the distinction in the type or in a method on the error.

## Define errors out of existence

Per Ousterhout, the most powerful error-handling move: **redesign the API so the error condition cannot occur.**

- A `Set` collection has no "duplicate item" error.
- A non-empty list type has no "empty" error.
- A `Money` type with currency built in has no "added amounts in different currencies" error.
- A `delete(file)` that returns silently if the file doesn't exist removes one error class.
- A queue with `dequeue() -> Optional<T>` removes the check-then-dequeue race.
- A constructor that requires all dependencies has no "uninitialized" error state.
- A type system that requires non-null parameters has no "null pointer" error class.

Spend design effort on this *before* spending it on error handling. An error class that doesn't exist is the cheapest error class to handle.

## Defensive vs offensive — the AI failure mode

LLMs love defensive try/except. They'll wrap entire function bodies in `try: ... except Exception as e: log(e); return None` for code that has no actual failure mode. This pattern:

- Hides real bugs (KeyError that should crash now silently returns None — and the caller wonders why their data is missing).
- Adds noise that obscures the actual logic.
- Creates the appearance of robustness without the substance.
- Can mask test failures (the test "passes" because the function "succeeded" with `None`).

When reviewing AI-generated code:

- For each `try`, ask: **what specific exceptions are we expecting, and what is the recovery?** "Catching anything that might happen" is not an answer.
- Replace `except Exception:` with `except SpecificError:` with a real handler, or remove the try entirely.
- Reject `try: ...; except: pass`. This is suppression, not handling.

The Erlang aphorism: **only handle errors you know how to recover from. Let everything else propagate.**

## Retries

Retrying makes sense when:

- The failure is transient (network blip, brief server overload, transient lock).
- The operation is idempotent — retrying produces the same result as one successful call.
- You can bound the retry budget (max attempts, max time).

Retry behavior to design:

- **Exponential backoff with full jitter.** Without backoff, retries cluster and create thundering herd. Without jitter, multiple retrying clients synchronize. The standard formula: `delay = random(0, base * 2^attempt)`.
- **Maximum attempts and maximum total time.** Both. Some failures last forever; you don't want infinite retries.
- **Retry only on retryable errors.** A 400 is not retryable. A 503 is. A timeout is. A 401 is not. Distinguish in the error type.
- **Per-call budget vs system budget.** A single client retrying enthusiastically can DDoS a recovering service. Use a circuit breaker to stop retrying when the upstream is clearly down.

What a bad retry looks like:

```python
# Bad
for _ in range(100):
    try:
        result = call()
        break
    except Exception:
        time.sleep(1)
```

No exponential backoff (synchronizes retries from many clients), no jitter (creates clusters), retries any exception including 400s, 100 attempts is unreasonable, no overall timeout, swallows the failure if it gives up.

```python
# Better (sketch)
def retry(call, max_attempts=5, base_delay_s=0.1, max_delay_s=10, deadline_s=30):
    deadline = monotonic() + deadline_s
    for attempt in range(max_attempts):
        try:
            return call()
        except RetryableError as e:
            if monotonic() >= deadline:
                raise RetryBudgetExceeded() from e
            delay = min(max_delay_s, base_delay_s * 2 ** attempt)
            time.sleep(random.uniform(0, delay))
        except NonRetryableError:
            raise
    raise MaxAttemptsExceeded()
```

## Idempotency

For any operation that can be retried, idempotency is the property that doing it twice is the same as doing it once. Without idempotency, retries corrupt data.

Designing for idempotency:

- **Idempotency keys.** Caller supplies a unique id per logical operation. Server stores the id with the operation result; subsequent calls with the same id return the prior result without re-executing.
- **Conditional updates.** "Set x to 7 *if x is currently 5*." Multiple identical retries have no effect after the first.
- **Upserts** instead of inserts where appropriate.
- **Set-based operations.** "Add to set" is idempotent; "increment counter" is not.

The most common bug class in retried systems: **a non-idempotent operation called via at-least-once delivery.** Webhook handlers, queue consumers, async job handlers — all "at least once" by default. Either make them idempotent or you will, eventually, charge a customer twice / send the same email twice / create a duplicate record.

## Circuit breakers

When an upstream is failing, retries make it worse. A circuit breaker:

- Tracks failures over a window.
- When failure rate exceeds a threshold, "opens" — subsequent calls fail fast without hitting the upstream.
- After a cooldown, "half-opens" — lets a few calls through to test recovery.
- If they succeed, "closes" again. If they fail, the circuit re-opens.

The wrong way: implement your own from scratch. Use the well-tested library for your platform (Polly, resilience4j, platform-native libraries, etc.) or your service mesh.

## Timeouts

Every call to anything that could take more than a moment (network, disk, lock, child process) needs a timeout. Without a timeout, a stuck dependency can hang requests indefinitely, exhaust connection pools, and cascade.

- **Total request timeout.** The whole request has a deadline.
- **Per-call timeout.** Each external call has its own.
- **Cumulative budget.** Time spent on retries + waits + calls should not exceed the request deadline. Plan the budget; don't let one slow upstream eat everything.
- **Propagate deadlines.** When you call another service, tell it the remaining deadline. Don't let it work on something that's already too late.

A specific failure pattern worth knowing: **service A's timeout is 30s, service B's is 60s, service C's is 90s.** When C is slow, B sits waiting longer than A's timeout, A times out, A's clients retry, and now B and C are processing duplicate requests. Timeouts should *decrease* as you go down the call stack, not increase.

## Observability hooks at the error path

Every error path needs observability: log, metric, trace.

- **Log with context.** The log message must include the inputs (or relevant identifiers) that caused the error. "Failed to fetch user" is useless; "Failed to fetch user (user_id=abc-123, source=cache, error=ConnectionRefused)" tells on-call where to look.
- **Metric for error rate.** Per error type, per endpoint. Spikes alert on-call.
- **Trace spans and events according to the relevant semantic convention.** Tracing reveals which downstream caused the error; for HTTP, 5xx generally maps to error on server spans, while 4xx usually stays unset on server spans unless application context says otherwise.
- **Don't log secrets, PII, or tokens.** Sanitize before logging. Especially in error contexts, where the impulse is "log everything."

A common smell: a try/except that catches and logs but provides no context, no metric, no trace. The error happens, the log is useless, on-call has nothing.

## What to flag in review

- A `try` with `except Exception` (catch-all) without a specific reason and a real handler. Especially if the catch just logs and continues.
- Three or more layers of try/except in the same call stack. Probably overcautious; pick one level.
- A retry with no backoff, no jitter, or no upper bound.
- A retry on a non-idempotent operation with no idempotency key.
- An exception used for normal-path control flow.
- Error types that don't carry enough context for diagnosis.
- Error messages that leak internals to end users.
- Background tasks fired with no error handling and no supervisor.
- New failure paths with no log / metric / trace.
- Calls to external systems with no timeout.
- A new feature that introduces a new dependency without a documented behavior on dependency failure.

## Reference library

- `references/resilience-patterns.md` — bulkheads, fallback, hedging, load shedding, graceful degradation, dead-letter queues.

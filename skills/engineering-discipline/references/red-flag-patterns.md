# Red-Flag Patterns

These are patterns that are nearly always wrong. They are not absolute rules — every entry has rare legitimate uses — but each deserves a hard look and an explanation in the PR description.

When you see one, do not silently approve. Either the author has a reason that should be documented, or the code should change.

## Correctness red flags

**`except Exception:` with no re-raise, no narrow handling, no log context.** Catches programming errors and silently turns them into "success." Especially bad when wrapping the entire body of a public function. Ask: what are the *specific* exceptions you expect, and why do they not need to propagate?

**`if x == True` or `if x is True` for non-boolean `x`.** Indicates the author is confused about types. Often hides a bug where `x` is a truthy non-boolean (e.g., `"false"` is truthy). For booleans, write `if x` and trust the type system or test it.

**`time.sleep(N)` to "wait for" a condition.** Almost always a race-condition workaround. Either the synchronization should be explicit (event, condition variable, polling with timeout), or the design needs to change to remove the implicit ordering.

**Floating-point equality (`==`) for non-zero comparisons.** Usually wrong. Use a tolerance, or use a decimal type if it's money/units.

**String concatenation to build SQL, shell commands, file paths, or URLs.** SQL: parameterize. Shell: use the `args` list form, not `shell=True`. Paths: use the language's path-join. URLs: use the language's URL builder. Concatenation breaks when input has special characters and is the most common injection vector.

**Catching, logging, and re-raising — three times in three layers.** The error is now in the logs three times with no added context. Either add context and re-raise, or handle and don't re-raise. Pick one.

**`assert` used for runtime validation in production code.** In Python, `assert` is stripped under `-O`. In several other languages, asserts are debug-only. Use real exceptions or error returns for anything you actually need to enforce.

**Mutable default arguments in Python (`def f(x=[])`)**. The default is shared across calls. Always wrong as a default; the bug is just waiting for the test that exposes it.

**`==` between values of different types in dynamic languages.** Often a sign of an upstream type confusion bug. Trace where the type uncertainty came from.

## Concurrency red flags

**Read-modify-write without a lock or CAS.** `counter = counter + 1`, `if not exists: create()`, `cache[key] = compute_if_missing(key)` — all wrong under concurrency. Either use atomic operations, take a lock, or use database constraints to enforce uniqueness.

**Holding a lock across an I/O call.** Lock held while waiting for the network, the disk, or another lock. Recipe for deadlock and tail-latency disasters. Compute under the lock, I/O outside.

**Background tasks fired with no error handling and no supervisor.** "Fire and forget" should be "fire and observe." A task that crashes silently is a bug-shaped time bomb. See `error-handling-and-resilience` skill.

**Async code that mixes sync I/O.** Calling `requests.get()` from inside `asyncio` code blocks the event loop. Use the async-native client or run the sync call in an executor.

**A retry loop with no backoff, no jitter, and no max attempts.** Will produce thundering-herd failures the first time the upstream blips. See `error-handling-and-resilience` skill.

**A retry on a non-idempotent operation with no idempotency key.** Will create duplicates. See `concurrency-and-state` skill.

## Design red flags

**A new abstraction with one concrete consumer and a hypothetical second.** Speculative generality. Inline the implementation; extract when the second consumer actually exists.

**A "manager", "helper", "util", "common", or "shared" module that grew past 500 lines.** Sign that a real concept is hiding in there, undiscovered. Decompose along an actual axis (the change-reason axis from Parnas / SRP), not by file size.

**A function with 4+ boolean parameters.** Boolean trap. Combinatorial explosion of behaviors at the call site. Extract enums, separate functions, or a parameter object.

**A function with 6+ positional parameters of compatible types.** Easy to call with arguments in the wrong order — a bug the type checker can't catch. Use a parameter object or named arguments.

**`get_or_create_or_update_or_delete()`** — a function whose name uses "or" or "and" multiple times. Multiple responsibilities. Split.

**Scattered or non-exhaustive type checks.** For closed sets, exhaustive pattern matching or sealed-class dispatch can be the clearest design. For open sets, prefer polymorphism or registration-based dispatch. The smell is duplicated, drifting dispatch logic, not every `match` or `isinstance`.

**Reaching into another object's private members (or its members' members).** Train wreck (`a.b.c.d.do_thing()`). Either move the behavior to where the data lives (Tell, Don't Ask), or define an explicit interface between modules.

**A class with only static methods and no state.** That's a module of functions wearing a class costume. Just make them functions.

**A circular import or a cyclic dependency between modules.** Modules whose dependency graph contains a cycle cannot be reasoned about independently. Break the cycle with an interface, by inverting the dependency, or by extracting the shared concept.

## Testing red flags

**A new test that passes both before and after the production change.** It's not testing what it claims to test. Either the test is wrong or the production change is dead code.

**A test that mocks the function under test.** A real category. Always wrong.

**Tests that share mutable state across runs.** Order-dependent tests are flaky tests. Each test should set up and tear down its own state.

**`time.sleep(0.1)` in a test "to let the thing finish."** Replace with a real wait condition. Sleep-based tests fail intermittently on slow CI and pass on developer machines.

**Snapshots/golden files larger than a screen.** Reviewer cannot see what changed. Assert on the structural pieces that matter, not the entire blob.

**Tests written by computing the expected output the same way the production code computes it.** The test now agrees with the code by construction. Compute the expected value by hand or by an independent method.

## Security red flags

**A trust boundary with no input validation on the trusted side.** "The frontend already validates" is not a defense — the frontend is not in the trust boundary.

**A new endpoint with no authn/z check.** Even if it returns "public" data. The reviewer should be able to point to the line that enforces authorization.

**Secrets in source, in environment variable defaults, or in error messages.** Use a secret manager. Verify error responses don't echo secrets back.

**`subprocess.run(user_input, shell=True)` or any equivalent.** Command injection. Use `shell=False` and pass arguments as a list.

**`pickle.loads()`, `yaml.load()` (unsafe), `eval()`, `Function()`, deserialization of untrusted data.** Remote code execution waiting to happen. Use safe deserializers (`json`, `yaml.safe_load`, language-specific safe decoders).

**A new direct dependency on an unvetted package.** Especially one with very low download counts or recently-published. Hallucinated and typo-squatted packages are real.

**A redirect that takes a URL from user input without an allowlist.** Open redirect — phishing vector.

**A file path or filename derived from user input concatenated to a base directory.** Path traversal. Resolve the path and verify it's still inside the base.

## API and contract red flags

**A change to the type, shape, error, or default of a public function in a non-bumped major version.** Breaking change in disguise. See `api-and-interface-design` skill.

**A new field in a response that callers may rely on, with no deprecation strategy for the old shape.** Forking the contract. Either fully replace or keep both with a clear migration window.

**An API that returns different shapes depending on input (sometimes a list, sometimes a single object; sometimes a 200 with an error body).** Forces every caller to handle a polymorphic response. Pick one shape.

**A "config flag" added with no default and no documentation.** New required configuration silently breaks every existing deployment.

**A migration that drops a column or table that "isn't used anymore" without a verification step.** "Isn't used" should be backed by logs/queries showing zero reads, not by grep.

## Observability and operations red flags

**A new failure path with no log, no metric, and no error context.** It will fail in production, and on-call will have nothing to go on.

**A new background job with no monitoring of its lag, success rate, or last-run-time.** It will silently stop and nobody will know until a customer complains.

**An error log message that contains only a stack trace, no contextual values.** "What was the input that caused this?" is the first question on-call asks; answer it in the log.

**A retry on an external call with no upper bound on time, calls, or queue depth.** Recipe for cascading failure when the upstream is slow.

**A cache with no expiration or no invalidation.** Stale data is a bug; "cache forever" is rarely correct.

## AI-generated-code red flags

If the change was authored by an AI (or shows the signature — a lot of explanatory comments, defensive try/except everywhere, plausible-looking but unfamiliar API calls), apply the `ai-coding-antipatterns` skill checks. The most common ones:

- **Hallucinated API call** to a method that doesn't exist on the type. Verify by reading the library.
- **Hallucinated package** in `requirements.txt` / `package.json`. Verify by looking it up in the registry.
- **Test that asserts on the implementation it was just generated alongside.** Tautology.
- **Defensive try/except wrapping code that doesn't raise.** Adds noise and can mask real bugs.
- **Comments that narrate the next line of code.** Sign of generation, not understanding. Remove or replace with intent.
- **Reinventing a function that already exists in the codebase.** Search before adding new utility code.

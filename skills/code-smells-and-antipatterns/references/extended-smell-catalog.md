# Extended Smell and Antipattern Catalog

Smells and antipatterns from the broader literature beyond Fowler's classic catalog. Each with a concrete example, the cost, and a fix or pointer.

## Architectural antipatterns

### Big Ball of Mud

Brian Foot and Joseph Yoder's term: a system with no discernible architecture, where every module knows about every other, change ripples freely, and the only way to understand any piece is to understand the whole.

**Symptom:** files import from anywhere; no clear layering; "where do I put this?" is answered by "wherever you can."

**Fix:** there is no quick fix. Strangler Fig out the new components with explicit boundaries; over time, the new architecture displaces the mud. (See `refactoring-techniques`.)

### Distributed Monolith

A microservices architecture in which the services can't actually be deployed independently — every change touches multiple services and they all have to ship together.

**Symptom:** a "release train" coordinating ~5 services; deploys often broken because a service is incompatible with others; cascading failures because a required service is down.

**Fix:** identify and break the synchronous dependencies. Async / event-driven where the services don't actually need to be synchronous. Sometimes: re-merge into a modular monolith (see `systems-architecture`).

### Anemic Microservices

Each "microservice" is a thin CRUD over its own table; all the logic lives in a coordination layer that calls all of them. Worst of both worlds: distributed-systems pain, monolithic logic.

**Fix:** the services should own their domain logic, not just their tables. Or, again, fewer larger services with their own boundaries.

### Database-as-Integration

Two services share a database; they read each other's tables; the database schema is the contract.

**Symptom:** a schema change in one service breaks another. No clear ownership. Foreign keys across "service" boundaries.

**Fix:** one service owns each table. Other services interact via API/event, not via direct table access. (See `systems-architecture` on data ownership.)

### Stovepipe System

Multiple subsystems each implement the same capability slightly differently because they were built in isolation.

**Symptom:** three different "user" representations across three subsystems; data must be reconciled by ETL.

**Fix:** identify the master domain model; use anti-corruption layers at the seams. (DDD framing.)

## Process / discipline antipatterns

### Cargo Cult Programming

Following a pattern by ritual without understanding why. "We use repositories because Clean Architecture says so" without understanding which problem repositories solve.

**Cost:** patterns applied where they don't help; junior developers learn the wrong lessons; the codebase has the shape of expertise without the substance.

**Fix:** every pattern needs a reason. "We use repository here because we need to swap implementations for testing and our team has agreed on this seam" is fine. "Because the book said so" is not.

### Premature Optimization

Designing for performance before measuring. Per Knuth (full quote, see `performance-engineering`): wastes effort 97% of the time.

**Symptom:** code that's complicated for performance reasons in places that are not the bottleneck; a cache where there's no traffic; sharding that doesn't help.

**Fix:** measure first; remove the premature optimization (it's adding complexity for no benefit); optimize if and where the measurement says so.

### Premature Pessimization

The opposite. Code that's needlessly slow for no reason. "It's just an example, we'll fix it later" — but the slow code ships and the slow code is now production.

**Examples:**
- `for x in items: if x in seen` — O(n²) when O(n) is trivial with a set.
- `select * from users where email = ?` with no index on email.
- N+1 queries inside a loop.
- Allocating in a hot loop when the buffer could be reused.

**Fix:** know the cost of common operations; pick the obviously-correct option. Premature pessimization is not absolved by Knuth — Knuth said don't optimize *small efficiencies*; he didn't say "ship known-O(n²) for n that gets large."

### Boy Scout Rule, Misapplied

"Leave the code cleaner than you found it" used as license to refactor whatever you want in someone else's PR. Result: bloated, hard-to-review changes mixing the requested change with unsolicited cleanups.

**Fix:** the boy scout rule is *bounded* — small, related, low-risk improvements only. Substantial cleanup is a separate change.

### Goldplating

Adding features, configurability, polish that wasn't requested and that the user doesn't need.

**Cost:** delays shipping; carries unknown bugs; teaches the team that requirements are negotiable.

**Fix:** ship the minimum. Iterate based on feedback.

### Bikeshedding (Parkinson's Law of Triviality)

Spending disproportionate review/discussion time on trivial details (variable names, button colors) and almost none on the substantive design (the API contract, the data model).

**Fix:** in design review, lead with the load-bearing decisions. If the discussion drifts to colors, redirect explicitly.

### Death March

A project that everyone knows is over-budget / late / under-scoped, but nobody is allowed to say so. Continues by pure organizational momentum.

**Fix:** rare for engineers to fix from the inside; raising it is right but expensive. The literature exists (Yourdon's *Death March*) for managers to recognize and stop.

## Performance antipatterns

### N+1 Queries

The single most common database performance bug.

```python
# Bad
users = db.query("SELECT * FROM users WHERE active=true")
for user in users:
    user.orders = db.query("SELECT * FROM orders WHERE user_id=?", user.id)
# 1 + N queries
```

**Fix:** join, batch with `IN (...)`, or use the ORM's eager-loading. Watch the SQL log for surprise queries; ORMs hide N+1 behind innocent attribute access.

### SELECT *

Returning all columns when only a few are needed. Bandwidth, CPU, often forces the query off a covering index. Also: a new column added later is silently shipped to all consumers (Hyrum's Law).

**Fix:** name the columns. Yes, even if it's verbose.

### The Cache as a Bug-Hider

A cache fixes an apparent performance problem but actually masks an underlying bug (the function is being called too often, recomputing the same value, hitting an unindexed query).

**Cost:** the cache adds complexity (invalidation, staleness, stampedes) when the underlying problem is fixable.

**Fix:** investigate why the slow operation is happening at all. Often the answer is "we're doing it 100x more than we should."

### Locking the World

Holding a coarse lock across a large code path that includes I/O.

```python
# Bad
with global_lock:
    data = expensive_db_query()
    result = process(data)
    publish(result)  # network call, holding the lock
```

**Cost:** every other thread waiting on `global_lock` blocks for the whole I/O round trip. Tail latency disaster.

**Fix:** narrow the lock to just the critical section; do I/O outside.

### Allocating in Hot Loops

```go
// Bad — allocates a new slice every iteration
for _, item := range items {
    buf := make([]byte, 0, len(item))
    // ...
}

// Better — reuse
buf := make([]byte, 0, max_size)
for _, item := range items {
    buf = buf[:0]  // reset
    // ...
}
```

**Cost:** GC pressure, memory bandwidth waste, allocator contention.

**Fix:** allocate once; reuse. Use object pools where appropriate.

## Concurrency antipatterns

### Read-Modify-Write Race

The most common concurrency bug. See `concurrency-and-state` for full treatment. Always wrong outside of single-threaded code.

### Lock-Then-IO

Holding a lock across a network call, file write, or other slow/unbounded operation.

**Fix:** compute under the lock; I/O outside.

### Forgotten Await / Forgotten Sync

```python
# Bad — async function called without await; the operation never runs
async def save(data): ...
def caller():
    save(data)  # returns a coroutine, never executes
```

**Fix:** language-specific. In Python, lint with `asyncio` warnings; in TypeScript, `no-floating-promises` lint rule. In Go, `go vet` for goroutine leaks.

### Goroutine / Async Task Leaks

A goroutine started with no shutdown mechanism; the goroutine outlives the work that needed it; eventually too many goroutines.

**Fix:** every long-running concurrent task needs a documented shutdown path. Context cancellation in Go; `CancellationToken` in .NET; `AbortController` in JS; structured concurrency where the language supports it.

### Double-Checked Locking, Done Wrong

A classic broken pattern across many languages where memory ordering issues let one thread see a partially-constructed object.

**Fix:** use the language's vetted primitive (`sync.Once` in Go, `LazyInitializer` in Java, etc.). Never roll your own.

## API and integration antipatterns

### Chatty API

An API that requires many round trips for one logical operation. "Get user, then get user's orders, then for each order get items, then for each item get pricing."

**Cost:** latency. A 200ms-per-call API is unusable at 10 calls per page load.

**Fix:** richer endpoints that return what callers actually need. GraphQL is one answer; carefully-designed REST another. Don't over-correct into "everything endpoint" — keep cohesion.

### Sloppy JSON

API responses with inconsistent shapes — "sometimes a list, sometimes a single object," "sometimes a 200 with an error in the body, sometimes a 4xx," "sometimes the field is null, sometimes missing."

**Cost:** every consumer has to handle every variant. Bug-prone.

**Fix:** pick one shape per concept and use it everywhere. Document. (See `api-and-interface-design`.)

### Time-Coupled API

A service whose API requires you to know its internal clock — "this endpoint returns Friday's data on Saturday after 06:00 UTC, but Thursday's data before."

**Fix:** make the timing explicit. Caller specifies which day; server doesn't have its own opinion.

### Versionless API

Public API with no version in the URL, header, or contract. Any change is a breaking change for someone.

**Fix:** version it from day one. (See `api-and-interface-design`.)

## Testing antipatterns

(See `testing-discipline` for full treatment; key smells repeated for the diagnostic library.)

### Tautological Tests

Test asserts on values computed by the same logic as the production code. Can only fail if the test framework breaks.

### Mock the Sun and the Moon

A test that mocks every collaborator. Verifies "the function called these methods in this order." Doesn't verify the function actually does anything correct.

### Sleep-in-Tests

`time.sleep(0.1)` to "let the thing finish." Race-condition camouflage.

### Order-Dependent Tests

Test B passes only after Test A has run. Hidden coupling. Will eventually flake.

### Flake Tolerance

A test that "sometimes fails" is left in CI. Once developers learn to ignore flaky tests, they ignore real failures too.

## Documentation antipatterns

### Documentation That Lies

Comment says one thing; code does another. The code wins; the comment is now actively misleading.

### Stale TODO Graveyard

`TODO: clean this up` from 2019. No owner. No condition for when it triggers.

**Fix:** TODOs need owner, date, and trigger. If they don't have those, delete them.

### "See the design doc" without a link

Every document references "the design doc" as if there's only one. There isn't. The reader can't find it.

### Decorative Banners

```
# ============================================
# # # # # # USER VALIDATION # # # # # # #
# ============================================
```

**Fix:** the file structure should communicate this; if a banner is needed, the file is too big.

## Security antipatterns (high level)

(See `secure-coding-fundamentals` for depth.)

### Authorization by Obscurity

A URL is "private" because it's not linked anywhere; no authz check on the endpoint. Trivially exploitable by anyone who guesses or finds the URL.

### Logging Secrets

```python
log.info(f"Authenticating with token {token}")  # token now in logs forever
```

### "Defense in Depth" as Excuse

Multiple weak controls instead of one strong one, used to justify each individual weakness. Each weak control is still weak; their composition isn't necessarily strong.

## Antipatterns specific to organizations

### The Lava Layer

A new architecture replaces an old one — but only partially. Years later, three architectural eras coexist. Every change has to know which layer it's in.

**Fix:** finish the migration. Strangler Fig and complete it; otherwise the half-done migration is a permanent tax.

### The Reorg-Driven Architecture

The architecture changes whenever the org changes (Conway's Law, see `software-design-principles`). When the org changes again, the architecture is suddenly wrong.

### The Hero

A team where one person knows everything; nothing ships without them; if they leave or burn out, the team collapses.

**Fix:** distribute knowledge: rotate on-call, pair on critical work, document, run game days. The hero is a sign of organizational fragility, not strength.

### The Architecture Astronaut

(Joel Spolsky's term.) Senior engineer who designs increasingly abstract architectures disconnected from the actual problem. Every concrete decision becomes "but what if we want to swap the database for Mars-rover memory cards in the future."

**Fix:** ground each abstraction in a concrete current need. YAGNI.

## How these compose

Real codebases usually have several of these at once, reinforcing each other. A god class with primitive obsession and no tests, integrated via shared database, in a distributed monolith with cargo-culted patterns. The refactor path is one smell at a time, prioritized by what's actually costing you.

The discipline isn't to eliminate every smell; it's to recognize each one, name it, decide whether the cost is worth paying for now, and document the choice.

---
name: concurrency-and-state
description: Find and fix concurrency bugs, design correct shared-state code, and reason honestly about race conditions, atomicity, ordering, and consistency. Use this skill whenever the task involves writing or reviewing concurrent code (threads, async, processes, goroutines), shared mutable state, locks/mutexes, atomic operations, transactions, retries with side effects, distributed state, queues / message passing, eventual consistency, idempotency, or anything that runs in more than one place at once. Concurrency bugs are the most expensive bugs in production — they're hard to reproduce, hard to test, and corrupt data invisibly. Use this skill aggressively whenever any code path has more than one actor.
---

# Concurrency and State

Concurrency is where confident-looking code goes to corrupt data. The bugs don't show up in unit tests, often don't show up in staging, and surface in production as "we have inconsistent data, and we're not sure why." Once they surface, they're hard to reproduce because they depend on timing.

This skill is about treating concurrency as a first-class design concern: assume the worst possible interleaving, design for correctness under adversarial scheduling, and don't trust intuition over the spec.

## The fundamental problem

Two pieces of code that each work correctly in isolation can produce wrong results when they run at the same time and touch the same state. The wrongness comes from:

- **Races** — interleaved reads and writes whose final result depends on order.
- **Atomicity violations** — a sequence of operations that the programmer assumed was indivisible but wasn't.
- **Ordering / visibility** — modern hardware and runtimes don't guarantee that one thread sees another's writes in the order they happened.
- **Deadlocks** — two parties each waiting for the other.
- **Lost updates** — two writes happen "at the same time"; one wins; the other vanishes.
- **Stale reads** — reading a value that was correct a moment ago but is wrong now.

Code that has none of these properties tested in production hasn't been tested under load. Concurrency bugs hide.

## Hickey's framing: state complects everything

From the `software-design-principles` skill: *state complects time, identity, and value.* A variable that mutates is a single name that, depending on when you look at it, points to different things. Reasoning about its current value requires reasoning about who modifies it, in what order, and whose modifications you can see.

The simplest way to avoid concurrency bugs is to **avoid mutable shared state**. Where you must have it, isolate it ruthlessly:

- **Pure functions over immutable data** — no concurrency bugs possible. The model functional languages enforce.
- **Immutable values** — once constructed, can be shared freely between threads. Most modern languages support this idiomatically (Java records, Python frozen dataclasses, TypeScript readonly types, Rust's borrow checker).
- **State owned by one actor** — only that actor can read/write. Other actors send messages or read snapshots. The actor model (Erlang, Akka).
- **State protected by a lock** — explicit mutual exclusion. Simple model; easy to misuse.
- **Lock-free / wait-free** — sophisticated; for hot paths where locks are too expensive. Should be implemented by experts using vetted primitives (atomics, CAS), not invented.

Pick a model per data structure and stick with it. The trap: half-protected data — sometimes accessed under lock, sometimes not. The unsafe access wins.

How to catch this in review: for each piece of mutable shared state, find every read and every write across the codebase. They should all be inside the same protection mechanism. Every exception is a bug. The diagnostic: a `grep` for the field name should show every access wrapped in the same lock acquisition (or the same atomic primitive, or the same actor). If even one access bypasses, that's the race.

## Read-modify-write: the most common bug

```python
# Bad
counter = counter + 1

# Bad
if not exists(key):
    create(key, value)

# Bad
value = cache.get(key)
if value is None:
    value = compute()
    cache[key] = value

# Bad
balance = account.get_balance()
account.set_balance(balance - amount)
```

Every one of these is buggy under concurrency. Two callers can read the old value, both compute a new value, both write — and one update is lost.

The fixes:

- **Atomic operation**: `counter.fetch_add(1)`. Single atomic instruction. No race.
- **CAS (compare-and-swap)**: read the old value; write the new value only if the underlying value still matches the old. Retry if not.
- **Database constraint**: `INSERT ... ON CONFLICT DO NOTHING` (atomic at the DB), unique constraint on the column.
- **Lock**: take a mutex around the read-modify-write. Slower; correct.
- **Optimistic concurrency**: include a version number; the update fails if the version changed.
- **Use a higher-level construct**: a thread-safe map's `compute_if_absent`; a database `UPSERT`; an actor that owns the state.

When reviewing code, scan for the read-modify-write pattern. Whenever you see "load → check → modify → store" without an atomic primitive holding it together, it's a race.

## Locks: the rules

If you must use locks, follow these or pay later:

- **Define lock ownership.** Each piece of mutable state has exactly one lock that protects it. Document this with the variable.
- **Take locks in a consistent order.** Inconsistent ordering causes deadlocks. If thread A locks X then Y, and thread B locks Y then X, they will eventually deadlock. Define a global lock ordering and enforce it.
- **Hold locks for the minimum time.** Compute under the lock; do I/O outside.
- **Never call user-supplied callbacks while holding a lock.** The callback might try to take the same lock or a different one in the wrong order.
- **Never block on a lock from inside a critical section.** Holding lock A and trying to take lock B is the recipe for deadlock.
- **Prefer fine-grained locks** to one big lock, *up to a point*. Many tiny locks taken in inconsistent orders is worse than one big lock.
- **Read-write locks** are useful when reads vastly outnumber writes; don't reach for them by default (their fairness semantics are subtle).
- **Reentrant vs non-reentrant.** Know which you're using. Reentrant locks let the same thread re-acquire; non-reentrant deadlock the thread on itself.

The most common lock bug: holding a lock across an I/O call. The lock is held while the network is slow; every other thread that wants the lock blocks; tail latency explodes; if the I/O hangs, the system stops.

## Atomicity

A sequence of operations is **atomic** if either all of them happen or none do, and other actors see only the before or after, never the middle.

Levels of atomicity:

- **Hardware atomic** (single CPU instruction): `fetch_add`, `compare_and_swap`. Atomic across threads on the same machine.
- **Lock-protected atomic**: a critical section. Atomic against other code that respects the same lock.
- **Database transactional**: a SQL transaction. Atomic per the database's isolation level.
- **Distributed atomic**: harder. Consensus algorithms (Paxos, Raft) or two-phase commit. Expensive; use sparingly.

When a sequence needs to be atomic, the entire sequence must be inside one of these. Mixing levels (some operations under lock, others in DB transaction, others freely) usually isn't atomic at any meaningful level.

## Database isolation levels

Standard SQL defines four (but most databases interpret these somewhat differently — check yours):

- **Read uncommitted**: can see other transactions' uncommitted writes. Almost never what you want.
- **Read committed**: only sees committed data. *But*: two reads in the same transaction can return different values (the data changed). Default in PostgreSQL.
- **Repeatable read**: two reads in the same transaction return the same value. *But*: phantom reads possible (a range query can return new rows).
- **Serializable**: as if transactions ran one at a time. Strongest; most expensive; can cause spurious aborts.

The trap: assuming "the database handles it" without knowing the isolation level. Many bugs are write-skew (two transactions read the same data, each makes a decision based on it, both commit, the data is now inconsistent because each transaction's decision assumed the other wouldn't happen).

For high-concurrency mutating logic, use SERIALIZABLE or use SELECT ... FOR UPDATE to make the intent explicit. For read-mostly queries, use READ COMMITTED and accept that you may see slightly different snapshots.

## Idempotency under retry

Any operation that can be retried needs to be idempotent. See `error-handling-and-resilience` skill for the design patterns.

The specific concurrency-related cases:

- **At-least-once delivery** (queues, webhooks, async jobs) means the consumer will see the same message multiple times. Process must be idempotent.
- **Network retries** mean the server may receive the same request twice (or the client may retry after a successful operation if the response was lost).
- **Background reconciliation** (replays, batch fixes) requires the same operations to be safe to re-apply.

Either: make the operation naturally idempotent (`PUT /resource/{id}` with full state), use idempotency keys (caller supplies, server stores), or use conditional operations (`UPDATE ... WHERE version = ?`).

## Async/await pitfalls

Async code looks like sync code but isn't. Common bugs:

- **Forgotten await.** `do_thing()` returns a coroutine that's never executed (or executed without the caller seeing the result).
- **Sync I/O in async code.** `requests.get()` from inside an asyncio coroutine blocks the entire event loop.
- **Race conditions specific to the event loop.** Single-threaded async code can still race because await points let other code run.
- **Cancellation.** A canceled task may have done partial work; clean up properly.
- **Long-running CPU work in async code.** Blocks the event loop; other coroutines starve.

Specific to JavaScript: a `Promise` rejection without a handler is a silent failure (or worse, a process crash in newer Node versions). Always `.catch()` or `await` in a try.

Specific to Python: `asyncio.gather` propagates the first exception; the others are silently lost unless you collect them. Use `gather(..., return_exceptions=True)` to handle.

Specific to Go: a goroutine that panics crashes the whole program (unless recovered). A goroutine that hangs leaks. A WaitGroup that's not drained leaks. Always handle goroutine errors and lifetime.

## Distributed state

Once state crosses machines, you have a different (harder) problem. The fallacies of distributed computing apply (see `systems-architecture` skill).

Key facts to internalize:

- **The network can drop, delay, duplicate, or reorder messages.**
- **You cannot distinguish "the other side is slow" from "the other side is dead."**
- **Time is not shared.** Two machines disagree on what time it is, by hundreds of ms (or much more). Never use wall-clock comparisons across machines.
- **A reply you didn't get may have been processed anyway.** The request reached the other side; the reply was lost. (This is why idempotency matters.)
- **Consistency models are a trade-off, not a default.** Strong consistency (everyone sees the same thing immediately) is expensive and not always needed; eventual consistency is cheap but requires the application to handle staleness.

The CAP theorem (Brewer): in the presence of a network partition, a system must choose between consistency and availability — it cannot fully provide both. The choice depends on what your data is for. AP systems (caches, content stores, much of the web) keep serving with possibly-stale data; CP systems (financial ledgers, identity systems, lock services) refuse to answer when uncertain. Neither is "the right answer" universally. PACELC (Abadi) extends this: even when there's no partition (Else), you trade latency vs consistency.

Practical implications for code:

- **Reads can be stale.** A "list orders" call might miss an order that was just created.
- **Writes can be lost** under partition; the system needs reconciliation.
- **Updates can be reordered.** Two writes A then B might be observed as B then A by some readers.
- **Causality matters.** If event A causes event B, the system should preserve that ordering. Vector clocks, version vectors, or logical timestamps.

For systems that need strong consistency: use a database that provides it (single-region SQL, Spanner, FoundationDB, CockroachDB, etcd). Don't roll your own.

For systems that can tolerate eventual consistency: design the application to handle staleness. Convergent data types (CRDTs) for collaborative editing. Last-write-wins with a clear conflict resolution policy. Manual reconciliation procedures when divergence is detected.

## Specific patterns to flag in review

- **Read-modify-write without atomic primitive or lock.** Always a race.
- **Lock held across an I/O call.** Tail-latency disaster waiting to happen.
- **Lock taken in inconsistent order across call sites.** Deadlock waiting to happen.
- **Async function that does sync I/O.** Blocks the event loop.
- **Background task fired with no error handling and no supervision.** Silent failures; lost work.
- **Retry on a non-idempotent operation with no idempotency key.** Will create duplicates.
- **Wall-clock comparison between machines.** Won't work; clocks drift.
- **`select then insert` pattern instead of `INSERT ON CONFLICT`.** Race under concurrency.
- **Consumer code that doesn't handle duplicate messages.** Queue and webhook systems are at-least-once.
- **Test that uses `time.sleep()` to "let things finish."** Probably hiding a real race.
- **A new shared mutable singleton.** Will eventually need synchronization; better to design without.
- **Cache without expiration / invalidation.** Stale data forever.
- **`await` in a loop where parallel `gather` would be safe.** Performance bug, not correctness, but worth noting.

## When to reach for a model checker / formal method

Most concurrent code is "review-able." Some isn't — distributed protocols, lock-free algorithms, leader election. For these, tools like TLA+ (Lamport) or Alloy let you specify the protocol and check it against properties.

This is heavy machinery. Worth the investment when:

- The bug is going to corrupt important state (financial, identity).
- The code will run for years and be modified by people who weren't there for the original design.
- The protocol is novel (you're not just using a well-tested library).

For code reviewers: if the change touches a published distributed protocol (Raft, Paxos, gossip), and the team isn't using a model checker, that's a finding worth raising. Distributed protocols are dangerous to invent.

## Reference library

- `references/distributed-state-patterns.md` — eventual consistency, CRDTs, sagas, outbox pattern, two-phase commit alternatives.

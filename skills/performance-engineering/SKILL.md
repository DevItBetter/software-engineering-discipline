---
name: performance-engineering
description: "Diagnose and fix real performance problems without falling into either \"premature optimization\" or \"performance is somebody else's problem\". Use this skill whenever the task involves measuring or improving performance, evaluating whether a change has performance implications, designing for latency or throughput, judging time/space complexity, finding hot paths, debugging a slow query / slow function / memory leak / scalability issue, choosing between approaches based on cost, or asking \"is this fast enough.\" Especially use it when reviewing code that's about to run on a hot path, when an existing system is too slow, or when somebody pulls the Knuth quote out of context to justify shipping known-bad code."
---

# Performance Engineering

Performance is two skills in one: knowing when *not* to optimize (most code) and knowing how to optimize *correctly* (the small fraction that needs it). The two halves are equally important. Premature optimization wastes effort and adds complexity; ignoring performance ships systems that fall over under load.

The most common failure modes:

- Optimizing the wrong thing because nobody measured.
- Refusing to think about performance until it's a production incident.
- Citing the Knuth quote as if it meant "performance doesn't matter."
- Adding caches, indexes, or async indirections "just in case."
- Optimizing a thing that's fast for a thing that's slow (CPU optimization on an I/O-bound workload).

## The Knuth quote, in context

The full quote, from "Structured Programming with Go To Statements" (1974):

> "Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. **Yet we should not pass up our opportunities in that critical 3%.**"

Both halves are the point. The popular truncation drops the second sentence and produces lazy thinking: "performance doesn't matter, we'll fix it later." Knuth said the opposite — for the 3% that matters, optimize *aggressively and well*.

Apply both halves:
- For the 97%: write clean, simple, idiomatic code. Don't add complexity for speculated speed.
- For the 3%: identify it (with measurement, not guessing), and optimize it carefully (with measurement again to verify).

## The performance methodology

The discipline that separates real performance work from cargo-culting:

### 1. Define what "fast enough" means

Performance is not a goal — *meeting a target* is a goal. Define the target before you start.

- **Latency target**: "p50 < 100ms, p99 < 500ms" for an interactive endpoint.
- **Throughput target**: "10,000 requests per second per instance."
- **Resource target**: "memory < 2GB at peak load."
- **Cost target**: "$X per million operations."

Without a target, "make it faster" has no end. With a target, you stop when you meet it and move on. (For SLO-driven engineering, see Google's SRE workbook — error budgets and SLOs make this rigorous.)

### 2. Measure, don't guess

Performance intuition is famously bad. CPUs do not work like the simple model you have in your head. Memory hierarchies, branch prediction, cache lines, GC pauses, async runtime overhead, network jitter — all of these defeat intuition.

The rule: **measure before you optimize. Measure after you optimize. Compare.**

- For latency: profile under realistic load with realistic data.
- For throughput: load test with the actual workload pattern.
- For memory: heap profile under sustained load.
- For correctness preservation: the test suite still passes.

A specific anti-pattern: "I think this is faster" based on reading the code. Without numbers, you don't know.

### 3. Find the bottleneck

Optimizing the non-bottleneck is wasted work. A 10x speedup in something that takes 1% of the time gives you 0.9% improvement. A 2x speedup in something that takes 50% of the time gives you 25%.

Tools:
- **Profilers** (perf, pyspy, async-profiler, pprof, dotTrace, etc.) show where CPU time goes.
- **Trace tools** (eBPF, OpenTelemetry, X-Ray) show where wall time goes (often I/O wait).
- **Memory profilers** show where allocations happen.
- **Database EXPLAIN** for SQL.
- **Network analyzers** for distributed systems.

Sample real workloads, not microbenchmarks. A microbenchmark that runs a single function in isolation often produces wildly misleading numbers (cache stays hot, branch predictor learns the pattern, JIT specializes for the test).

### 4. Optimize the bottleneck

Apply the right kind of optimization for the bottleneck:

- **CPU-bound**: better algorithm, less work, parallelization, vectorization.
- **I/O-bound**: fewer round trips, batching, parallelism, caching, async I/O.
- **Memory-bound**: smaller data, better layout (cache-friendly), object pooling, fewer allocations.
- **Lock-bound**: finer-grained locks, lock-free data structures, reduce contention.
- **Network-bound**: compression, batching, fewer round trips, geographic distribution.

A bottleneck check: does the optimization actually address the bottleneck? Adding a cache to a CPU-bound workload doesn't help. Parallelizing an I/O-bound workload that's bottlenecked on the upstream doesn't help.

### 5. Verify

Re-measure after optimizing:
- Did the bottleneck actually move? If yes, where is the new bottleneck?
- Did the code get faster on the realistic workload (not just the microbenchmark)?
- Did anything else get slower? (A new cache might help reads but hurt writes; a new index might help queries but hurt inserts.)
- Did correctness change? (The test suite still passes — and ideally, you have property-based or fuzz tests that exercise the optimization.)

Optimizations without verification are technical debt with optimism attached.

## Common performance problems and their shapes

### N+1 queries

The single most common database performance bug. Code loads a list of N items, then for each item, makes another query.

```python
# Bad
orders = db.query("SELECT * FROM orders WHERE user_id = ?", user_id)
for order in orders:
    order.items = db.query("SELECT * FROM items WHERE order_id = ?", order.id)
# 1 + N queries
```

Fix: load the related data in one query (JOIN or batched).

```python
# Better
orders = db.query("""
    SELECT orders.*, items.*
    FROM orders
    LEFT JOIN items ON items.order_id = orders.id
    WHERE orders.user_id = ?
""", user_id)
# 1 query
```

ORMs hide N+1 behind innocent-looking attribute access. Always check the SQL log for surprise queries.

### Unnecessary work

The fastest code is the code that doesn't run.

- **Repeated computation in a loop**. Compute once outside.
- **Reading the same data multiple times**. Cache the read.
- **Computing what you already have**. Many functions are called with values that already exist.
- **Defensive copies that aren't necessary**. Profile shows allocation; question whether the copy is needed.
- **Logging at debug level in a hot loop with debug logging enabled in production.**
- **Large object construction in a tight loop**. Move outside or pool.

### Bad data structure choice

- **Linear search where a hash lookup would be O(1)**. List membership tests in a loop are O(n²).
- **Hash where you needed sorted access** (now you can't binary-search).
- **Array where you needed frequent insert/delete in the middle** (linked list better).
- **Synchronized data structure where access is single-threaded** (overhead for nothing).

The right data structure depends on the access pattern. The default (list/array, hash map) is usually right; specialized cases want specialized structures (heap for top-K, trie for prefix lookup, bloom filter for "definitely-not-present" checks).

### Memory bloat

- **Loading a large dataset entirely when streaming would do.** Process records as they arrive.
- **Unbounded caches.** Will OOM. Always set a size or TTL.
- **Closures retaining unexpected references.** A captured variable holds the entire object graph alive.
- **Strings used where bytes would do** (in some languages, strings are 2-4 bytes per character).
- **Boxed types in tight loops** (in some languages, autoboxing is expensive).

### Bad cache choices

Caches add complexity. Use them when:
- The same query is repeated frequently.
- The query is expensive.
- The data tolerates staleness or has a clean invalidation signal.

Don't use them when:
- The query is already fast.
- The data needs to be fresh.
- Invalidation is hard. (The famous line attributed to Phil Karlton — *"There are only two hard things in Computer Science: cache invalidation and naming things"* — has never been documented in print; the earliest internet trace is Tim Bray's 2005 blog post recalling it from ~1996. The "off-by-one" addition is later folklore. See `caching-strategies` for the discipline.)

Cache anti-patterns:
- Cache forever (data goes stale, bug appears mysteriously).
- Cache with no max size (OOM).
- Cache that's checked but never invalidated on write (stale).
- Cache stampede (cold cache + simultaneous requests = N parallel computes for the same value).

For cache stampede: use a single-flight pattern (one request fetches; others wait for the result). Standard library in Go (`golang.org/x/sync/singleflight`); pattern in others.

### Poor algorithmic complexity

- **O(n²) where O(n) was possible.** Often hiding in nested loops, repeated `find` calls, or list-of-list operations.
- **O(n log n) where O(n) was possible.** Sorting when a single pass would suffice.
- **Exponential blowup.** Recursive algorithms without memoization. The Fibonacci-by-naive-recursion class of bug.
- **O(n!) ever.** Never; always wrong for n > ~10.

When you see nested loops, ask the complexity. When you see recursive calls, ask whether memoization or iteration would help. When you see `find` inside a loop, ask whether a hash lookup is possible.

### Network round trips

Each network round trip is ~1ms (same data center) to 100ms+ (cross-region). Doing 100 of them sequentially takes a long time.

- **Batch.** One request that does 100 things, not 100 requests that each do one.
- **Parallelize.** If they're independent, do them concurrently.
- **Pipeline.** If the protocol supports it, send the next request before the previous response arrives.
- **Compress.** Bytes on the wire matter; especially for cross-region.
- **Keep-alive.** Reuse connections; don't pay TCP+TLS handshake every request.

### Lock contention

Threads waiting for locks instead of doing work.

- **Fine-grained locks** instead of one big one (with care — see `concurrency-and-state`).
- **Lock-free data structures** for hot paths (atomics, CAS).
- **Sharding** — partition the data so different threads work on different shards.
- **Read-write locks** when reads vastly outnumber writes.
- **Optimistic concurrency** (assume no conflict; retry if there is).

### GC pressure (managed languages)

- **Reduce allocations.** Reuse objects (object pools), use primitives instead of boxed, batch operations to amortize.
- **Smaller objects.** Padding and headers add up.
- **Long-lived references.** Promote objects to old generation early; minimize cross-generational references.

In Java/Go/C#/JavaScript, GC pause is a real source of latency spikes. Profile with GC visibility on; understand which generation is filling up.

## When performance is non-negotiable

Some systems must perform from the start:

- **Hot paths in user-facing services** (homepage, login, checkout). A 100ms regression matters.
- **Background jobs that hold up downstream work.** Slow batch jobs cascade.
- **Inner loops of compute-heavy operations.** A 2x slowdown in a kernel that runs millions of times is the whole runtime.
- **Anything inside a database transaction.** Slow code holds locks; locks block other work.

For these, performance is a first-class design concern, not an afterthought. Design with the performance budget in mind; measure as you build.

## What to flag in review

- A new database query inside a loop (probable N+1).
- An unbounded cache, queue, or buffer.
- A nested loop where the inner loop has nontrivial work.
- A `requests.get()` (or sync I/O equivalent) inside an async handler.
- A new endpoint with no documented latency target on a hot path.
- A "performance optimization" without measurement.
- A "performance optimization" that introduces complexity disproportionate to its benefit.
- A new background job with no rate limit and unbounded work per call.
- A microbenchmark used to justify a real-world claim.
- "I think this is faster" without numbers.

## What NOT to flag

- Clean, simple, idiomatic code that hasn't been profiled. Most code is fast enough; don't waste author time on speculative optimization.
- Algorithmic choices in a non-hot path with realistic input sizes.
- Reasonable allocations in functions that aren't called in tight loops.
- Defensive code (validation, logging) on the cold path.

## Reference library

- `references/profiling-and-measurement.md` — how to actually measure: tools, methodology, common mistakes.

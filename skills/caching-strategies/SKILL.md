---
name: caching-strategies
description: "How to use caches without their cost — placement, read and write strategies, invalidation, stampede protection, eviction policies, consistency models, and the antipatterns that turn a cache from a latency optimization into a hidden source of truth. Use this skill whenever the task involves adding or sizing a cache, choosing TTLs, designing invalidation, debugging a stale-read or stampede problem, choosing between Redis and Memcached, evaluating HTTP cache headers, or asking \"is this cache earning its keep.\" Use it when reviewing code that introduces caching, when designing a system that needs to scale reads, and when judging whether a slow query should be cached or fixed. Built on RFC 9111 (HTTP Caching), the AWS Builders' Library, the Vattani et al. probabilistic stampede paper (VLDB 2015), DHH on key-based expiration, the Karger consistent-hashing paper (STOC 1997), and the modern eviction literature (TinyLFU, S3-FIFO)."

---

# Caching Strategies

The widely-attributed Phil Karlton line — *"There are only two hard things in Computer Science: cache invalidation and naming things"* — is the most-cited quip in software, and the only documented internet trace is Tim Bray's 2005 blog post recalling it from ~1996. The line has never been documented in print or recording; treat it as folklore-but-real, and never as a citation.

The reason it sticks: a cache is not a free performance win. It is a deliberate trade — **freshness for latency, throughput, and load reduction on the source of truth**. Every cached value is potentially stale the moment it is stored. The discipline of caching is acknowledging that trade honestly and choosing the placement, invalidation, and consistency model that match what the system actually needs.

The other failure mode worth naming up front: **cache addiction**. The AWS Builders' Library piece *Caching challenges and strategies* warns about services that grow dependent on their cache being warm, until cache loss becomes an outage rather than a slowdown. A cache should be an optimization. When it becomes load-bearing, the system has lost its underlying ability to operate.

## Placement — where the cache lives

Caches live at every layer; placement governs both reach and the cost of getting it wrong.

- **Browser cache.** HTTP semantics, governed by **RFC 9111 (HTTP Caching, June 2022)**, which obsoletes RFC 7234. `Cache-Control`, `ETag`/`If-None-Match`, `Last-Modified`/`If-Modified-Since`, `Vary`.
- **CDN / edge.** CloudFront, Cloudflare, Fastly. Honor HTTP semantics plus vendor extensions (Surrogate-Key, cache tags). Soft-purge marks stale; hard-purge evicts.
- **Reverse proxy / API gateway.** Varnish, NGINX, Envoy. Also HTTP-semantics-aware.
- **Application in-process cache.** Caffeine (Java), Ristretto (Go), `functools.lru_cache` (Python), Guava. Sub-microsecond reads — but **per-instance**. See antipatterns.
- **Distributed / shared cache.** Redis, Memcached, Hazelcast. Network round-trip; survives restarts (Redis with persistence) at the cost of latency and a new failure domain.
- **Database query / result cache.** Postgres has shared buffers and per-session plan caches but no result cache. **MySQL deprecated the query cache in 5.7.20 (2017) and removed it in 8.0.3 (September 2017)** — `query_cache_*` variables, `FLUSH QUERY CACHE`, and `SQL_CACHE` are gone.
- **Materialized views.** Database-managed cached query results, refreshed on schedule or on demand. Conceptually a cache with a query as its key.

Place a cache where the cost it saves outweighs the staleness it introduces and the failure mode it adds. Multiple layers (browser + CDN + app) are often correct, but each layer's invalidation discipline must work.

## Read-side strategies

Definitions per AWS, Microsoft Azure Architecture Center, Oracle Coherence, and Hazelcast docs.

- **Cache-aside (lazy loading).** Application checks cache; on miss, reads from source and populates cache. Application orchestrates both. The most common pattern. Microsoft's explicit warning: on update, **update the data store before invalidating the cache**, not after. The reverse ordering opens a race where a concurrent reader repopulates the cache with the pre-write value.
- **Read-through.** The cache itself loads from source on miss via a registered loader; the application calls only the cache. Higher cohesion, tighter coupling between cache and loader. Coherence and Hazelcast both support this natively.
- **Refresh-ahead.** Cache asynchronously reloads recently-accessed entries before they expire. Trades extra source load for tail-latency smoothing on hot keys. Useful only when access patterns are predictable.

## Write-side strategies

- **Write-through.** Write to cache and source synchronously; the write does not return until both succeed. Strongest consistency between cache and store; highest write latency. Almost always paired with cache-aside or read-through so misses still resolve.
- **Write-back / write-behind.** Write to cache; cache flushes to source asynchronously after a delay. Best write throughput; **loses data acknowledged in cache but not yet flushed if the cache fails**. Suitable for high-write workloads where the source is the bottleneck and some data loss is acceptable (analytics, telemetry).
- **Write-around.** Writes bypass the cache; only the source is updated. Cache populates on the next read miss. Good for write-heavy data that's rarely read; bad for read-after-write workflows because the just-written value is not in cache.

For cache-aside writes, the safe ordering is: **write to source first, then invalidate the cache key** (Microsoft). Inverting this opens a stale-repopulation race that is hard to diagnose later.

## Invalidation

Karlton's line is about this section.

- **TTL (time-based).** Simplest. Bounded staleness equal to the TTL. No consistency guarantees within the window. Choose TTLs from the SLO for staleness, not gut feel — "five minutes" is not a value, it's a number. AWS recommends a **dual-TTL** pattern: a soft TTL after which the client tries to refresh, and a hard TTL after which stale is no longer served. Gives graceful degradation when the source is unavailable.
- **Event-driven.** Invalidate on writes via CDC (Debezium tailing the WAL), pub/sub (Redis keyspace notifications, Kafka topics), or queues. Closest you can get to "fresh and fast." Adds coupling and a new failure mode (missed events) that itself needs monitoring.
- **Tag / dependency-based.** Group keys by tag; invalidate the tag to invalidate all dependents. Fastly Surrogate-Key, Cloudflare cache tags. **Russian-doll caching** — DHH described key-based cache expiration in *How key-based cache expiration works* (Signal v. Noise, 19 February 2012); the term "Russian doll" came into wide use with Rails 4's `cache_digests` (late 2012/2013), which automated dependency tracking on view fragments. Each model's `updated_at` is part of the cache key; updating a child changes its key, cascading up to parents. Old keys are never invalidated explicitly; they become unreachable and are evicted by LRU pressure.
- **Versioned keys / cache-busting.** Include build SHA, content hash, or version in the key (`app.a1b2c3.js`, `user:42:v17`). Invalidation = deploying a new key, atomic and free. The HTTP `immutable` directive (RFC 8246, September 2017) was designed for exactly this. The cleanest invalidation strategy for assets that have a stable identity per deploy.
- **Manual purge.** CDN soft- and hard-purge endpoints. Operationally important for breaking-glass scenarios; should not be load-bearing for normal correctness.

The right strategy depends on staleness tolerance and write rate. Static assets with content hashes: versioned keys. Mutable user data with read-after-write: event-driven or write-through. Anything where seconds of staleness is acceptable: TTL.

## Cache stampede / thundering herd

When a hot key is cold or just-expired, *N* concurrent requests each miss and fan out to the source. The source is hit *N* times — often catastrophically. Also called dogpile.

Mitigations:

- **Single-flight / request coalescing.** Go's `golang.org/x/sync/singleflight` is the canonical implementation: `Group.Do(key, fn)` runs `fn` once per in-flight key; concurrent callers wait on the result. Other languages have equivalents (Caffeine's `LoadingCache`, Java's `CompletableFuture` patterns, Python's per-key locks).
- **Probabilistic early expiration.** Vattani, Chierichetti, Lowenstein, *Optimal Probabilistic Cache Stampede Prevention*, PVLDB 8(8): 886–897, 2015. Each reader, before TTL, recomputes the value with probability that increases as expiration approaches: roughly `currentTime − delta * beta * ln(rand()) ≥ expiry` triggers an early refresh, where `delta` is the measured recompute cost and `beta` (default ~1) controls aggressiveness. Cloudflare's *Sometimes I cache* describes a continuous variant. **Lock-free; scales horizontally** — its main appeal.
- **Lock-based.** Distributed lock on the key (Redis `SET NX`, Redlock with caveats); winner refreshes, others wait or serve stale. Simple but creates contention and adds a lock-failure mode.
- **Stale-while-revalidate.** RFC 5861 (May 2010, now incorporated into RFC 9111). Serve stale content to the client while the cache refreshes in the background. Combined with `stale-if-error`, gives both stampede mitigation and an availability boost when the source is down. Vendor support is uneven — most modern CDNs honor it, but some clients and corporate proxies still strip the directive; verify with your cache layer.
- **CDN request collapsing.** Distinct from SWR. When N concurrent misses arrive at the edge for the same key, Fastly and Cloudflare can collapse them into a single forward to origin and fan the response back out. Operates on *in-flight misses*; SWR operates on *expired-but-stored entries*. Use both when available.

Default for production: single-flight in-process for hot computations, plus probabilistic early expiration or stale-while-revalidate at the cache layer.

## Consistency models

- **Strong consistency** between cache and source requires synchronous coordination — write-through plus read-through, or transactional cache + DB. Usually achievable only when the cache is the system of record (Redis as primary store).
- **Eventual consistency** is the default. Bounded by TTL and invalidation latency.
- **Read-your-writes** is achievable with cache-aside if writes invalidate before returning, but **not safe across distinct cache instances** in multi-instance deployments unless the cache is shared. The single biggest source of "works on staging, breaks under load."
- **Negative caching** stores "this doesn't exist" results to avoid repeated source hits on missing keys. Dangerous when time-to-create is shorter than the negative-cache TTL: a user signs up, the cached "user does not exist" outlives the create, the user appears logged-out. Use short TTLs (seconds, not minutes) and proactively bust on creation.
- **Cache penetration** is the adversarial form: an attacker (or a buggy client) deliberately requests keys that don't exist, bypassing the cache and hammering the source. Defenses: short-TTL negative caching for non-existence; a **Bloom filter** at the cache layer that records "definitely-known keys" and rejects misses for keys that have never existed; rate-limiting per client. Cloudflare, GitHub, and large-scale registries use Bloom filters in front of negative caching.

### The dual-write problem

Writing to two systems (DB + cache, DB + message broker, DB + search index) without a distributed transaction is non-atomic. Two failure scenarios:

- Write DB, crash before cache update → cache stale until next invalidation.
- Write cache, crash before DB → cache holds data that does not exist in the source.

The standard solution is the **transactional outbox pattern**: in one local transaction, write the state change *and* an outbox row; a separate worker drains the outbox to the cache or broker at-least-once. CDC tailing the WAL is a particularly clean implementation — the database is the only system to write to, and downstream invalidation is derived from the WAL, not from application code.

## Eviction policies

Sizing the cache is half the problem; choosing what to evict is the other half.

- **LRU (Least Recently Used).** Default in Memcached, Guava, `functools.lru_cache`. Simple, decent on workloads with temporal locality.
- **LFU (Least Frequently Used).** Better when there's a stable hot set; worse when popularity shifts. Pure LFU has the "old hot key" problem (yesterday's hits keep keys alive forever); aged/decaying LFU mitigates it.
- **TinyLFU / W-TinyLFU.** Einziger, Friedman, Manes, *TinyLFU: A Highly Efficient Cache Admission Policy*, ACM TOS 13(4):35, 2017. Uses a count-min sketch (~8 bytes per entry) to approximate frequency; an item is **admitted** to the main cache only if its sketch frequency exceeds that of the eviction candidate. **W-TinyLFU** prepends a small windowed LRU (default ~1% of total, adapted at runtime via hill climbing in Caffeine) to absorb recency bursts that pure frequency-admission would reject. Used in **Caffeine** (Java) and **Ristretto** (Go); near-optimal hit rates competitive with ARC and LIRS.
- **S3-FIFO.** Yang et al., *FIFO Queues are All You Need for Cache Eviction*, SOSP 2023. Three FIFO queues (small ~10%, main ~90%, ghost) implement "quick demotion" — new items go to the small queue; if not re-accessed there, they're evicted before reaching main. Beats LRU on miss ratio across most of 6,594 production traces, with much higher throughput.
- **ARC (Adaptive Replacement Cache).** Megiddo & Modha, USENIX FAST '03 (IBM Almaden). Adaptive blend of recency and frequency via two LRU lists plus two ghost lists. Patented by IBM (US 6,996,676 / 7,058,766); the patent is widely reported to have expired around 2024 but verify before committing in regulated contexts. Postgres briefly used ARC in 8.0 then removed it for patent reasons; ZFS still uses a modified ARC. TinyLFU and S3-FIFO often perform comparably without the legacy patent baggage.

For a new cache: LRU is the safe default; W-TinyLFU is the modern default for serious workloads; S3-FIFO is the new entrant worth tracking.

## Distributed caching specifics

- **Redis vs. Memcached.** Both in-memory KV. Memcached: pure cache, multithreaded, simple `string→bytes`. Redis: rich data structures (lists, hashes, sorted sets, streams), optional persistence (RDB, AOF), pub/sub, scripting, cluster mode, replication. Redis's persistence makes it usable as a primary store; Memcached is purely volatile.
- **Sharding via consistent hashing.** Karger, Lehman, Leighton, Panigrahy, Levine, Lewin, *Consistent Hashing and Random Trees*, STOC '97. Adding or removing one node out of N moves only ~1/N of keys. The foundational paper, the work that later spawned Akamai.
- **Rendezvous (HRW) hashing.** Thaler & Ravishankar, University of Michigan, 1996 — actually predates Karger et al. by a year. Hash `(key, node)` for every node; pick the highest. O(N) per lookup, no virtual nodes, cleaner re-balancing. Used in GitHub's load balancer, Apache Druid, Apache Ignite, Kafka.
- **Replication and failure modes.** Redis uses **asynchronous replication**; Sentinel handles failure detection and replica promotion. Redis docs are explicit: Sentinel + Redis does not guarantee that acknowledged writes survive failures. A master with persistence off plus auto-restart can wipe its replicas if it restarts faster than Sentinel detects the failure.
- **Hot-key problem.** A single very-hot key lives on a single shard, which becomes the bottleneck. Mitigations: client-side replicate the hot key (`key:1`, `key:2`, ...) and pick randomly; pin to an in-process L1; cache at the edge; or use read-replicas.

### L1 / L2 cache hierarchies

A widely-deployed pattern: an in-process L1 (Caffeine, Ristretto) in front of a shared L2 (Redis). L1 reads are sub-microsecond; L2 reads are sub-millisecond; the source is hit only on full miss. The hard part is **L1 invalidation across instances** — a write on one instance must invalidate L1 on every other instance. The standard solution: pub/sub on writes (Redis keyspace notifications, Kafka topic) drives L1 invalidation; or short L1 TTLs (seconds) bound the inconsistency window. For data with read-after-write requirements across instances, prefer a single shared cache; for read-mostly data with bounded staleness tolerance, two-tier with short L1 TTLs is often the right shape.

## HTTP caching specifics

`Cache-Control` directives that matter (per RFC 9111, summarized on MDN):

- `public` — storable in shared caches even if `Authorization` was present.
- `private` — storable only in user-agent-private caches.
- `no-cache` — must revalidate with origin before reuse. **Not** "do not cache."
- `no-store` — do not store in any cache.
- `max-age=N` — fresh for N seconds.
- `s-maxage=N` — `max-age` for shared caches only; overrides `max-age` and `Expires` for them.
- `must-revalidate` — once stale, cannot be served without revalidation (no stale-fallback).
- `stale-while-revalidate=N` — RFC 5861. Serve stale up to N seconds after expiry while refreshing in background.
- `stale-if-error=N` — RFC 5861. Serve stale up to N seconds if origin returns 5xx or is unreachable.
- `immutable` — RFC 8246, September 2017. Tells the client not to revalidate during freshness lifetime even on reload. Designed for content-hashed asset URLs.

**Validators.** `ETag` (opaque, strong by default) with conditional `If-None-Match`; `Last-Modified` with `If-Modified-Since`. Origin returns `304 Not Modified` on match — saves bandwidth, not a round-trip.

**The `Vary` header** names request headers that must match for a cached response to be reused. `Vary: Accept-Encoding` is almost always needed. The interaction with `Authorization` is subtler than commonly stated: per RFC 9111 §3.5, a shared cache must not reuse a response to an `Authorization`-bearing request *unless* the response carries `public`, `s-maxage`, or `must-revalidate`. So the request header alone disables shared-cache reuse by default. If you intentionally make an `Authorization`-dependent response shared-cacheable, partition it with `Vary: Authorization` or an equivalent per-user cache key; otherwise prefer `private` or `no-store`.

**Immutable assets pattern.** Serve `app.a1b2c3.js` with `Cache-Control: public, max-age=31536000, immutable`. New deploys produce new hashes, new URLs, new cache entries. Invalidation = deploy. The single most reliable HTTP caching strategy for static assets.

**`Cache-Status` header.** RFC 9211 (June 2022). Standardized way for caches to annotate hit/miss/forwarded status across a chain. Replaces ad-hoc `X-Cache: HIT` headers; valuable for debugging.

## Metrics that actually mean something

- **Hit ratio** is useful but easily misleading. A 95% hit ratio where the missing 5% are the hottest, most expensive keys is worse than 50% on uniform data.
- **Latency at p50, p99, p99.9** with cache vs. without. Caches earn their keep at the tail; mean latency hides it.
- **Eviction rate, memory pressure, key cardinality.** Track them. An eviction storm tells you the cache is too small or the working set has shifted.
- **Single-flight wait time / coalesced-request ratio.** Hidden cost when stampedes are mitigated; should be visible on a dashboard.
- **Origin offload ratio** at the CDN — fraction of requests not forwarded to origin.

If a cache has no metrics, it is operating on faith. See `observability` for the broader discipline.

## Sizing and memory cost

- **Always set a max size and a TTL.** Unbounded caches OOM or get evicted at the worst possible time.
- **Watch key cardinality.** Keying by user-supplied input (full URL with query params, full path) explodes the keyspace; hit ratio collapses. Same family of bug as observability metric label cardinality.
- **GC pressure on in-process caches.** Large JVM/CLR caches drive long GC pauses. Caffeine offers off-heap variants; Chronicle Map, Ehcache off-heap, and OS-page-cached `mmap` files sidestep GC entirely.
- **Serialization cost on distributed caches.** A Redis round-trip plus serialization is not always faster than recomputation for cheap values. Profile.

## Common antipatterns

- **Caching reflexively.** Sometimes adding a cache adds latency (extra hop) without measurably helping hit ratio. Profile first.
- **No max size, no TTL.** OOM, slow leak, or stale-forever data.
- **Unprotected stampede.** First production traffic spike on a cold cache takes the source down.
- **Caching authorization decisions across users.** If an authenticated or personalized response is explicitly made shared-cacheable without `Vary`, per-user keys, or another partitioning mechanism, user A's response can be served to user B. Catastrophic and recurring.
- **Caching `POST` / write responses.** `POST` is non-cacheable by default; caching it confuses idempotency assumptions.
- **Caching 5xx errors with long TTLs.** Pins an outage in place even after the source recovers. Cache 5xx for *seconds* if at all, never minutes.
- **"Just add Redis."** Doesn't fix a consistency bug; adds a dual-write problem on top of it.
- **In-process cache treated as shared in a multi-instance service.** Each instance has its own; reads after writes are inconsistent across instances. Single cleanest cause of "works on staging, breaks under load."
- **Cache as a Q-fix for slow queries.** Cache covers the symptom; the slow query still runs whenever the cache is cold or evicted, and it scales O(working-set) not O(qps). Fix the query.
- **Cache as hidden source of truth.** When a value lives only in cache, or differs from the database, the cache is now a non-durable source of truth. Roll a deploy that flushes Redis and you lose data.
- **TTL by gut feel.** TTL should be derived from the SLO for staleness and the cost of refreshing.
- **Negative caching with TTL exceeding time-to-create.** New objects appear nonexistent for the negative-cache window.
- **Stale invalidate-then-write ordering.** The reverse of Microsoft's safe rule; opens a stale-repopulation race.

## What to flag in review

- A new cache without a max size, a TTL, and a hit-rate metric.
- A new cache without a stampede-protection plan for cold-cache scenarios.
- A cache-aside write that invalidates the cache *before* writing to the source.
- An in-process cache (Caffeine, `lru_cache`, Guava) in a multi-instance service treated as if it were shared.
- A negative cache TTL longer than the typical time-to-create for the negated entity.
- A personalized response explicitly made shared-cacheable without `Vary`, per-user keying, or another partitioning mechanism.
- A new endpoint with `Cache-Control: no-cache` where `private, max-age=0` was meant — these mean different things.
- A 5xx response cached for minutes.
- A "fix" for a slow query that adds Redis without addressing the query plan.
- A cache key that includes raw user input (full URL, free-text search) without bounding cardinality.
- Use of `--no-cache` / cache flushing as a deploy step that the system relies on for correctness.

## Reference library

- `references/http-caching-and-cdn.md` — RFC 9111 directives, validators, `Vary`, the immutable-assets pattern, CDN purge semantics, `Cache-Status`.
- `references/stampede-protection.md` — single-flight, probabilistic early expiration (Vattani et al. formula), lock-based approaches, stale-while-revalidate.
- `references/eviction-policies.md` — LRU, LFU, W-TinyLFU, S3-FIFO, ARC; when each applies; sizing heuristics.

## Sibling skills

- `performance-engineering` — the discipline of measuring before optimizing; many "we need a cache" diagnoses are slow-query problems.
- `observability` — cache hit ratio, eviction rate, single-flight wait time, origin offload as first-class signals; cardinality discipline applies identically to cache keys.
- `concurrency-and-state` — the dual-write problem, idempotency under retry, transactional outbox.
- `database-and-data-modeling` — when caching masks a query that should be fixed; materialized views as a structured cache.
- `systems-architecture` — placement decisions across browser / CDN / app / DB; the consistency model the system actually needs.
- `secure-coding-fundamentals` — trust boundaries; the auth-bypass risk when caches serve personalized content without partitioning by identity is one consequence (covered in this skill).

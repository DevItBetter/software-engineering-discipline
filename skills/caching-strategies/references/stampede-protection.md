# Stampede Protection

Reference for preventing the cache stampede / dogpile / thundering herd: when a hot key is cold or just-expired, *N* concurrent requests each miss and fan out to the source, hitting it *N* times — often catastrophically.

## Why it happens

Three scenarios cause stampedes:

- **Cold cache** — service restart, cache flush, deploy that resets in-process caches.
- **Just-expired key** — a hot key reaches its TTL; many concurrent readers all miss simultaneously.
- **Cache eviction under pressure** — the working set has shifted; a hot key gets evicted; the next access misses.

The stampede is dangerous in inverse proportion to the source's cost. If recomputing the value takes 1ms, *N* concurrent computations are tolerable. If recomputing takes 1 second and includes a database round-trip, *N* concurrent computations can take the database down.

## Single-flight / request coalescing

In-process pattern: when the first caller for a key starts computing, all subsequent callers for the same key wait on that in-flight result rather than starting their own.

The canonical implementation is Go's `golang.org/x/sync/singleflight`:

```go
var g singleflight.Group

result, err, shared := g.Do("key", func() (any, error) {
    return expensiveComputation()
})
```

Concurrent calls with the same key share a single execution; `shared == true` indicates the caller piggy-backed on someone else's computation. `Forget(key)` clears state; `DoChan` returns a channel for non-blocking waits.

Equivalent patterns in other languages:

- **Java**: Caffeine's `LoadingCache` with `getIfPresent` + asynchronous `get` coalesces concurrent loads on the same key.
- **Python**: `asyncio.Lock` per key, or a `dict[key, Future]` pattern where the first caller creates the future and others await it.
- **Node**: `dataloader` (Facebook) batches and dedupes concurrent loads for the request lifetime.
- **Rust**: `tokio::sync::OnceCell` per key, or `dashmap` of futures.

Single-flight is the simplest stampede mitigation and works in-process. By itself it does not solve cross-instance stampedes — N service instances each hitting the source on the same cold key is N times the source load. In practice, in-process single-flight paired with a shared cache (Redis read-through populated by single-flight per instance) bounds the worst case to N, not unbounded — for typical service deployments that's a 100×–1000× reduction over no protection. For full cross-instance coalescing, layer distributed locks or probabilistic expiration on top.

## Probabilistic early expiration

Vattani, Chierichetti, Lowenstein, *Optimal Probabilistic Cache Stampede Prevention*, PVLDB 8(8): 886–897, 2015 (DOI: 10.14778/2757807.2757813).

The idea: each reader, before the TTL expires, has a small probability of voluntarily recomputing the value early. The probability increases as expiration approaches. By the time the TTL hits, with high probability one of the readers has already refreshed the value; subsequent readers find a fresh value, no stampede.

The canonical formula — XFetch — fires a refresh when:

```
currentTime − delta * beta * ln(rand()) >= expiry
```

where:
- `delta` is the measured cost of recomputing (in time units),
- `beta` is a tuning parameter (default ~1; higher = more aggressive),
- `rand()` is uniform in (0, 1).

Cloudflare's *Sometimes I cache* describes a continuous variant `p(t) = e^{−λ(expiry−t)}` with λ as a steepness parameter. Both come from the same paper; pick one notation and stick with it.

The key property: **lock-free, scales horizontally**. Each reader makes its own decision independently using local state. This is its main appeal over lock-based approaches.

A subtle point: probabilistic early expiration trades a small amount of extra work (some readers recompute slightly early) for elimination of the stampede. The trade is real — for high-cost recomputes with aggressive `beta`, the steady-state refresh rate can rise meaningfully above "1 per TTL" and degrade hit rate. Tune `beta` against the measured `delta` (recompute cost) and the source's tolerance for extra load; `beta` near 0 makes refreshes very rare and approaches pure TTL behavior; `beta` near 2 makes refreshes more frequent than necessary.

## Lock-based approaches

Acquire a distributed lock on the key. The winner refreshes; others either wait (and serve from cache when the winner finishes) or serve stale.

The Redis primitive: `SET NX` (set-if-not-exists). Pseudocode:

```
got_lock = redis.SET("lock:key", request_id, NX=true, EX=lock_ttl)
if got_lock:
    value = expensive_computation()
    cache.set(key, value)
    redis.DEL("lock:key")
else:
    # Either wait (poll) or serve stale
    return cache.get_stale(key)
```

Trade-offs:

- **Simple**, but adds a lock-failure mode (the winner crashes mid-compute; the lock TTL must be conservative, and others must handle the case).
- **Adds a round-trip** for the lock acquisition.
- **Doesn't compose** with multi-step pipelines well.
- **Redlock** (Redis multi-master locking) has [contested correctness properties](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) per Kleppmann; for stampede protection specifically, the consequences of a failed lock are usually tolerable (an extra refresh, not data corruption), so Redlock's caveats matter less here than in stricter use cases.

Lock-based is appropriate when you need cross-instance protection and the recomputation cost is too high to tolerate even occasional duplication.

## Stale-while-revalidate

RFC 5861 (May 2010), now in RFC 9111. Serve stale content to the client while the cache refreshes in the background.

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

Means: fresh for 60 seconds, then stale-but-servable for another 300 seconds while a background refresh runs. The first request after expiry triggers the background refresh and gets stale; subsequent requests within the SWR window also get stale until the refresh completes.

Combined with `stale-if-error`:

```
Cache-Control: max-age=60, stale-while-revalidate=300, stale-if-error=86400
```

— gives stampede mitigation *and* an availability boost when the source is down.

For HTTP responses this is the simplest and most effective stampede mitigation; it doesn't require single-flight or probabilistic logic at the application layer. The cache layer handles it transparently. Vendor support is uneven — most modern CDNs honor it, but some clients and corporate proxies still strip the directive. Verify with your specific cache layer before relying on it.

## CDN request collapsing

Distinct from SWR. When N concurrent misses arrive at the edge for the same key (a cold cache or a just-expired entry), Fastly and Cloudflare can **collapse** them into a single forward to origin, then fan the response back out to all N waiters. The mechanism operates on **in-flight misses**; SWR operates on **expired-but-stored entries**. Use both when available — they cover complementary windows.

Configuration is typically a vendor switch (Fastly's `req.hash_always_miss` and clustering, Cloudflare's "Always Online" and tiered caching). The capability is real but not universal — verify your CDN supports it before relying on it for stampede protection.

## Pre-warming

Sometimes the right answer is to never have a cold cache. Patterns:

- **Background refresh** of known-hot keys on a schedule.
- **Pre-deploy warmup**: replay recent traffic against the new instance before routing real traffic.
- **Dual-cache deploy**: keep the old instance's cache around as a "shadow" until the new instance has warmed.

Pre-warming is operational discipline more than an algorithm. The schedule and the chosen-key list need to come from production access patterns, not guesses.

## Choosing

A defensible default for production:

- In-process: single-flight on every cache load.
- Cache layer (Redis, Memcached, CDN): `stale-while-revalidate` where supported; probabilistic early expiration on hot keys where it isn't.
- Lock-based only when recomputation cost is high enough that even probabilistic duplication is too much, or where the recomputation has external side effects that must not be duplicated.

The wrong default is "do nothing and hope a stampede never happens." The first stampede is almost always discovered during a traffic spike or a cache flush, in production, at the worst possible time.

## Sources

- Vattani, Chierichetti, Lowenstein. *Optimal Probabilistic Cache Stampede Prevention*. PVLDB 8(8): 886–897, 2015. `https://dl.acm.org/doi/10.14778/2757807.2757813`.
- Cloudflare. *Sometimes I cache* (probabilistic early expiration). `https://blog.cloudflare.com/sometimes-i-cache/`.
- Go `golang.org/x/sync/singleflight`. `https://pkg.go.dev/golang.org/x/sync/singleflight`.
- Caffeine `LoadingCache`. `https://github.com/ben-manes/caffeine/wiki`.
- RFC 5861 *HTTP Cache-Control Extensions for Stale Content* (May 2010). `https://www.rfc-editor.org/rfc/rfc5861.html`.
- Martin Kleppmann. *How to do distributed locking* (the Redlock critique, 2016). `https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html`.

# Resilience Patterns

A condensed reference of resilience patterns that come up repeatedly in real systems.

## Bulkhead

Borrowed from ship design: compartments that flood independently so one breach doesn't sink the ship.

In software: separate the resources used by different request paths so a failure in one path can't exhaust resources for others. Examples:

- **Separate thread pools** for high-priority vs low-priority work. A flood of low-priority requests can't starve the high-priority ones.
- **Separate connection pools** per downstream. A slow downstream consuming all your connections doesn't kill calls to other downstreams.
- **Per-tenant resource limits.** One tenant's bad behavior can't exhaust capacity for everyone.

The cost: more complexity, more idle resources at any given moment. The benefit: bounded blast radius. Use for systems where any single failure can otherwise take everything down.

## Fallback

When the preferred path fails, fall back to a degraded but acceptable result.

- **Cached value when the live source is unavailable.** Stale is better than nothing for many use cases.
- **Default value when the personalization service is down.** Show generic content instead of nothing.
- **Local computation when the upstream is slow.** Local approximation when the precise value isn't available.

Designing fallbacks:

- The fallback path must be tested. A fallback that's never exercised will fail when first needed.
- The fallback must be cheap. If the fallback also requires the failing dependency, it's not a fallback.
- The fallback should be observable. Track when it's used; spikes are a signal.
- Document for callers: "may return stale data when X is unavailable" so they don't depend on freshness contracts that don't hold under failure.

## Hedging

Send a duplicate request to the upstream after a delay; take the first response. Reduces tail latency by routing around the slow replica or slow request.

- Delay should be ~95th percentile of normal latency. Send the hedge only when the original is unusually slow.
- Cancel the loser if possible (some protocols/networks let you cancel an in-flight request).
- Limit hedging by a global rate to avoid doubling traffic.

Trade-off: roughly 5% extra requests for substantially better tail latency. Useful for read-heavy systems where work is duplicated easily.

Don't hedge writes. Don't hedge non-idempotent operations.

## Load shedding

When the system is over capacity, reject some requests fast rather than queueing them slowly. The rejected requests get a clean failure (which the client can handle with retry/backoff); the rest get acceptable performance.

- **Decide what to shed.** Lower-priority traffic, anonymous traffic, retries, requests already past their deadline.
- **Decide where to shed.** At the load balancer (cheapest), at the service entry, in the queue (expensive — work was already partially done).
- **Communicate.** Return 503 with `Retry-After`. Many clients respect it.

Tail latency under load is usually worse than throughput at low load. Load shedding keeps the system in the regime where it's actually useful.

## Dead-letter queues

For asynchronous workers: when a message fails repeatedly, move it to a dead-letter queue (DLQ) instead of retrying forever.

- Define the retry policy explicitly: max attempts, time-to-live.
- DLQ items must be **observable** — alert when items appear, dashboard for DLQ depth.
- DLQ items must be **investigable** — store enough context (the message, the error history) to diagnose.
- DLQ items must be **replayable** after fixing the bug — you should be able to re-enqueue them.

A DLQ that grows without anyone noticing is a hidden incident in progress. Treat the DLQ as a first-class operational surface.

## Graceful degradation

When some functionality fails, the system continues with reduced functionality rather than failing entirely.

- The product page works even if the recommendations service is down (no recommendations, but you can buy).
- Search works even if the auto-suggest service is down (manual typing instead of suggestions).
- The dashboard works even if real-time metrics are unavailable (show cached values with a "data may be stale" badge).

Designing for graceful degradation requires the system to know how to keep going when each piece is missing. This is a design discipline (what depends on what?) more than a code discipline.

A useful test: for each non-essential dependency, what happens when it's down? If the answer is "the whole system fails," there's a degradation gap.

## Backpressure

When a downstream is slow or overloaded, push back on the upstream rather than buffering forever.

- Bounded queues. When the queue is full, the producer blocks or drops.
- Rate limiting. Reject or slow producers above a threshold.
- TCP-style flow control. The receiver tells the sender how much it can accept.

Without backpressure, a fast producer feeding a slow consumer eventually exhausts memory. The bug shape: "everything looks fine, then OOM."

A specific application: in async/queue-based systems, monitor queue depth as a leading indicator. Growing queue depth means the consumer can't keep up; backpressure or scaling is needed before the queue blows up.

## Timeouts that decrease down the stack

If service A calls B which calls C, the timeouts should be:

- A's timeout: the request budget.
- B's timeout: A's timeout minus the time B has already spent.
- C's timeout: B's remaining time.

If timeouts *increase* down the stack, you get the failure mode where C is slow, B keeps waiting, A times out, A's caller retries, and now there are two requests in flight to a system that's already overloaded.

Propagate deadlines explicitly. gRPC has built-in deadline propagation; HTTP services need an organization-specific non-`X-` header convention such as `Request-Deadline` or a standard mechanism in the platform. Whatever the mechanism: every call should know "you have N ms to respond, after which I'm not going to use the result."

## Idempotency keys (expanded)

For any operation that might be retried, accept a caller-supplied key:

- Server stores `(idempotency_key, result, expires_at)` for some retention window.
- New request with the same key: return the stored result. Don't re-execute.
- New request with a different key: execute normally.
- Key expires: subsequent requests treated as new (the caller should make a new key per logical operation).

Choose the key carefully. It should be:

- **Unique per logical operation.** A UUID per user-initiated action.
- **Stable across retries.** The same logical action retried gets the same key.
- **Not reused across different operations** (or you'll return the wrong result).

For incoming webhooks: use the provider's event id as the idempotency key. For client-initiated requests: have the client generate a UUID and include it in a header (`Idempotency-Key`).

## Health checks vs readiness checks

Confusing these is a common cause of bad failure modes.

- **Liveness check** ("is this process alive?"): if it fails, the orchestrator restarts the process. Should fail only on unrecoverable conditions (deadlock, corrupt state).
- **Readiness check** ("is this process ready to serve traffic?"): if it fails, the orchestrator stops sending traffic. Should fail when the process can't currently serve (downstream is unavailable, cache is cold, migration in progress).

A liveness check that fails on transient issues (downstream blip) causes restart loops that make the situation worse. A readiness check that doesn't fail on real unavailability sends traffic to a process that will fail.

Common mistake: making the liveness check check downstreams. If your DB is down and your app's liveness probe hits the DB, every replica restarts simultaneously, and now you have a much bigger problem.

## The full-system view

A resilient system isn't a sum of resilient parts. It requires looking at the whole graph:

- Are there cycles in the dependency graph? (Circular dependencies cascade failures.)
- Is there a single point of failure? (One database, one auth service, one DNS provider.)
- What's the blast radius of each failure? (One slow downstream → degraded feature, or whole system?)
- What's the recovery model? (Auto-recover, manual intervention, regional failover?)
- When was the last time recovery was tested? (If you don't run game days, your recovery plan is fiction.)

The tools for thinking about this at the system level: chaos engineering (deliberately break things in prod-like environments to validate the recovery story), runbooks (documented procedures for common failures), game days (scheduled exercises). See `systems-architecture` skill for the broader framing.

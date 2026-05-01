# Wide Events and Cardinality

Reference for the high-cardinality, wide-events approach to observability — the practical core of being able to ask unanticipated questions.

## What "wide" means

A **wide event** is a single structured record per unit of work (typically a request or job), containing every dimension that might be useful for later analysis. The Honeycomb-style claim is that mature instrumentation produces events with 200–500 dimensions: request metadata, user/tenant identifiers, route templates, status, latency, downstream call counts and durations, feature flag values, build SHA, region, host, container, queue depths sampled at the boundary, and any business-domain identifiers (order id, document id, organization id) that an investigation might want to filter by.

The discipline is: **write the event once, derive metrics, traces, and forensic logs from it.** The closest hands-on articulation is Stripe's *canonical log line* (Brandur Leach, 2019) — emit one wide structured log line per request at the request boundary. The same idea applies to background jobs, message handlers, and pipeline stages.

## Cardinality, defined

- **Cardinality** of a field = the number of unique values it takes across the event population.
- **Dimensionality** = the number of fields per event.

You want both high. High dimensionality lets you ask many questions; high cardinality lets you filter to the rows that matter (the customer in question, the build that introduced the bug, the request id you are tracing).

The questions you actually ask in production are almost all high-cardinality:
- "Show me failing requests for tenant X in build Y in region Z over the last hour."
- "Which feature flag combinations are correlated with the latency spike?"
- "Which customer's traffic pattern triggered the cardinality bomb?"

If the telemetry can't answer these without code changes, the system is not observable for the questions that matter.

## Why metrics backends struggle

Time-series databases like Prometheus represent each unique combination of metric name and label values as a separate time series, allocated in memory. Adding a label with two distinct values can double memory consumption. Adding a label whose values grow unboundedly (raw user id, full URL, raw error message, container hash) causes a cardinality explosion that crashes the scraper.

This is not a Prometheus pathology — it is structural to any metrics store that pre-aggregates by label combination. StatsD/Graphite have the same problem (cardinality is encoded in the metric name).

The fix is not "stop using metrics." It is to route the right questions to the right storage:

- **Metrics store**: low-cardinality SLO math, dashboards, four Golden Signals. Cheap and fast for aggregates.
- **Event store / wide-event store** (Honeycomb, ClickHouse, columnar warehouses, Loki for some workloads): high-cardinality ad-hoc queries.
- **Trace store**: cross-service causality and per-request investigations.

A modern observability stack uses all three; what matters is which is the system of record for which question.

## Label budget

Treat label space as a budget you spend, not a resource you have unlimited supply of.

- **Always-tag candidates** (low cardinality, high analytical value): `service.name`, `service.version`, `deployment.environment`, `region`, `cluster`, request method, response status class (`2xx`, `3xx`, `4xx`, `5xx`), well-bounded enums.
- **Tag-with-care** (sometimes high cardinality, but the question you want): `customer_id`, `tenant_id`, `route_template` (NOT raw URL — the templated form `/users/:id/orders` is bounded; the raw form is not). Route to event store, not metrics store.
- **Never as labels in a metrics-only backend**: raw URLs with embedded ids, raw error messages, stack traces, anything user-controlled, anything PII or secret-bearing.

If you find yourself wanting to add a high-cardinality label to a metric, that is a signal to move that question to an event store.

## Exemplars: bridging metrics and traces

Exemplars are the way modern Prometheus / OpenMetrics implementations bridge low-cardinality metrics with high-cardinality traces. An exemplar is a (metric value, trace_id) pair attached to a histogram bucket: when you see the latency histogram bucket for `>1s` requests fill up, you can click an exemplar and jump straight to a trace that produced that latency.

Exemplars do not solve high cardinality in metrics — they let you escape from metrics into traces when you find an interesting value. They are the cheap version of the wide-events idea: keep the metric small, keep the per-request detail in the trace, link them.

## Common cardinality bombs

- **Per-pod labels on a metric** in a Kubernetes environment with rolling restarts. Each new pod hash is a new time series; series accumulate without bound.
- **Build SHA as a label** on a long-running tenant or service. New builds add series; series pile up.
- **HTTP status code as a label, but also raw status text**. The text varies per language/stack and creates duplicates.
- **Customer id on a metric** in a multitenant system without an event store. Acceptable for ten customers, fatal at ten thousand.
- **Free-text error messages** as labels. "Connection reset by peer at 10.0.0.42:5432" is not a label, it is an event field.

## Practical patterns

- **Aggregate-and-explore.** Metrics for the dashboard; events for the drill-down. The dashboard says "p99 latency spiked"; the event store answers "for which customers, on which build, with which feature flag set."
- **Canonical event at the boundary.** One wide structured event at the end of every request/job, even if you also emit per-stage events.
- **Sample on the value, not the source.** Tail-based sampling that keeps all errors and slow requests beats head sampling that misses the bad cases.
- **Allowlist serialization at the logging library.** New fields default to redacted. Never opt-in to logging a token; opt-in fields are how PII leaks.

## Sources

- Charity Majors. *Observability is a Many-Splendored Definition* (2020). `https://charity.wtf/2020/03/03/observability-is-a-many-splendored-thing/`.
- Honeycomb. *Structured Events Are the Basis of Observability*. `https://www.honeycomb.io/blog/structured-events-basis-observability`.
- Cindy Sridharan. *Distributed Systems Observability* (O'Reilly, 2018). `https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/`.
- Brandur Leach / Stripe. *Fast and flexible observability with canonical log lines* (2019). `https://stripe.com/blog/canonical-log-lines`.
- Cloudflare. *How Cloudflare runs Prometheus at scale*. `https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/`.
- OpenMetrics exemplar specification. `https://github.com/OpenObservability/OpenMetrics`.

# Tracing and OpenTelemetry

Reference for distributed tracing and the OpenTelemetry standard. Verify current status at `https://opentelemetry.io/status/` before taking any version-specific claim as load-bearing — the project moves.

## Dapper data model

The *Dapper* paper (Sigelman, Barroso, Burrows, Stephenson, Plakal, Beaver, Jaspan, Shanbhag; Google, 2010) established the model still in use:

- **Trace** = a tree of spans representing one logical unit of work spanning multiple services.
- **Span** = one unit of work within the trace: an RPC call, a function call, a queue operation. Has a start time, duration, parent span, attributes, and events.
- **Context** = the metadata that travels with the request and lets downstream spans attach to the right parent.

Dapper's three principles still hold:

1. **Low overhead via aggressive sampling.** Tracing every request at full fidelity is too expensive. Sample, but sample intelligently.
2. **Application-level transparency via library instrumentation.** The instrumentation lives in shared infrastructure libraries (RPC, HTTP, database clients) so application code does not have to know it exists.
3. **Ubiquitous deployment.** Tracing only matters when it covers the whole request path. A trace that skips one service tells you very little about a multi-service incident.

OpenTracing (deprecated), OpenCensus (deprecated), Zipkin, Jaeger, and OpenTelemetry all inherit this model.

## Context propagation: W3C Trace Context

A trace is only as useful as its weakest propagation hop. **W3C Trace Context Level 1** reached W3C Recommendation status on **6 February 2020**, with a revised Recommendation dated 23 November 2021 (`https://www.w3.org/TR/trace-context/`). It defines two HTTP headers:

- `traceparent` — version, trace-id, parent-id, trace-flags. The required carrier of trace identity. Format: `00-<32-hex-trace-id>-<16-hex-span-id>-<2-hex-flags>`.
- `tracestate` — vendor-specific extensions in a comma-separated key/value format.

**Trace Context Level 2** is a W3C Candidate Recommendation Draft as of early 2026, adding a random-trace-id flag and stronger guidance on id generation.

Use Trace Context end-to-end. The most common debugging frustration is a trace that ends mid-stack because:
- A library on a hot path doesn't propagate headers (older HTTP clients, custom RPC frameworks, in-process queues).
- An async hop (thread pool, goroutine, message queue) doesn't carry context across the boundary.
- A non-HTTP transport (database, cache, message bus) doesn't have a propagation convention. (Some do — OpenTelemetry database semantic conventions, AMQP/Kafka headers — but none are universally implemented.)

Audit propagation explicitly. Tools that confirm context survives every hop are worth the time.

## OpenTelemetry overview

OpenTelemetry is the CNCF project standardizing vendor-neutral telemetry. It standardizes:

- **API** — what application code calls.
- **SDK** — the runtime that batches, samples, and exports telemetry.
- **Semantic conventions** — agreed attribute names so that data from different vendors and stacks can be correlated.
- **OTLP** — the wire protocol for shipping telemetry to a backend.

It was formed in 2019 from the merger of OpenTracing and OpenCensus. CNCF Incubating since 26 August 2021. As of early 2026, a graduation review is in progress (filed 31 May 2025); the project is still listed as Incubating until the TOC concludes.

### Signal stability (verify on the status page)

As of early 2026, by signal:

- **Tracing**: spec stable; SDKs Stable in C++, .NET, Erlang, Go, Java, JavaScript, PHP, Python, Ruby, Swift; Rust Beta.
- **Metrics**: spec stable; Stable in most major SDKs; some still maturing.
- **Logs**: most recent signal to stabilize; spec stable, but SDK stability varies by language. Check the status page before adopting OTel logs SDKs in production.
- **Collector**: mixed — core components have varying stability; v1 GA anticipated but not yet declared.

When picking up OpenTelemetry, check the status page for *your language* and *your signal* before assuming production-readiness.

### Semantic conventions

Semantic conventions are the cross-vendor agreement on attribute names. Without them, correlating data across vendors is string-mangling. Important conventions:

- **Resource**: `service.name` (the only required Resource attribute), `service.version`, `service.namespace`, `deployment.environment`, host attributes, container attributes, k8s attributes.
- **HTTP**: `http.request.method`, `http.response.status_code`, `url.path`, `url.full`, `server.address`, etc. **HTTP semantic conventions reached stable in spec v1.23.0, 2023.**
- **DB**: `db.system.name`, `db.namespace`, `db.operation.name`, `db.query.text`.
- **Messaging**: `messaging.system`, `messaging.destination.name`, `messaging.operation`.
- **RPC**: `rpc.system`, `rpc.service`, `rpc.method`.
- **GenAI / LLM**: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, etc. — actively evolving.

Hub: `https://opentelemetry.io/docs/specs/semconv/`.

The discipline: use semantic-convention names for any attribute that has one; use prefixed custom names (e.g., `mycompany.feature.flag.foo`) for attributes that do not.

## Sampling strategies

### Head-based sampling

The decision to record is made at the start of the request, before anything has happened. Cheap, deterministic, and the default in most SDKs. Variants:

- **Probabilistic** — fixed percentage (e.g., 1% of all traces).
- **Parent-based** — inherit the sampling decision from the upstream caller's `traceparent`.
- **Rate-limited** — at most N traces per second.

Limitation: cannot sample-on-error because you do not yet know whether the request will fail. A 1% head sample drops 99% of errors along with 99% of successes.

### Tail-based sampling

Record everything, then decide at the end whether to keep it. Catches anomalies (errors, slow requests, suspicious attribute combinations). Requires buffering somewhere — typically the OpenTelemetry Collector with the `tail_sampling` processor.

Production pattern: head-sample at high rate (10–100%) at the edge to control cost, then tail-sample in the Collector with policies like:

- Keep all traces with a span-level error.
- Keep all traces with total duration over a threshold (e.g., the p99 of recent traffic).
- Keep all traces with a feature-flag attribute set to a specific value (during a rollout).
- Keep 1% of the rest.

### Common sampling failure modes

- **Head sampling that misses the bad request.** The whole point of tracing is to investigate the failures; a fixed head-sample rate that randomly drops them defeats the exercise.
- **Inconsistent decisions across services.** If one service samples and the next doesn't, the trace is incomplete. `traceparent` flags carry the decision; ensure downstream services honor it (parent-based sampler).
- **Span attribute explosion.** Tail-based sampling buffers spans by trace_id; a service that emits 10,000 attributes per span will OOM the Collector. Cap span attribute counts.

## Common debugging failures in traces

- **Trace appears to end mid-stack.** Context not propagated across a queue, thread pool, or library that lacks instrumentation. Fix: instrument the hop; trace context propagation is a property of the whole call graph.
- **Two traces for one logical request.** Some service fabricated a new trace_id instead of reading `traceparent`. Fix: use the SDK's HTTP server instrumentation rather than rolling your own.
- **Spans attached to wrong parent.** Async work that keeps a stale `Context` (e.g., a goroutine launched without explicit context capture). Fix: always pass context explicitly across async boundaries; use language idioms (Go's `context.Context`, Python's `contextvars`).
- **Traces in dev, no traces in prod.** Sampling miscalibrated. Verify prod sampling configuration directly.
- **No trace correlates with the metric.** No exemplar wiring. Add exemplars to histograms so on-call can jump from "p99 spiked" to a real trace.

## Sources

- Sigelman, Barroso, Burrows, Stephenson, Plakal, Beaver, Jaspan, Shanbhag. *Dapper, a Large-Scale Distributed Systems Tracing Infrastructure* (Google, 2010). `https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/`.
- W3C *Trace Context* (Recommendation, 2021). `https://www.w3.org/TR/trace-context/`.
- W3C *Trace Context Level 2* (Candidate Recommendation Draft). `https://www.w3.org/TR/trace-context-2/`.
- OpenTelemetry status. `https://opentelemetry.io/status/`.
- OpenTelemetry sampling. `https://opentelemetry.io/docs/concepts/sampling/`.
- OpenTelemetry semantic conventions. `https://opentelemetry.io/docs/specs/semconv/`.
- CNCF OpenTelemetry. `https://www.cncf.io/projects/opentelemetry/`.
- CNCF TOC graduation issue. `https://github.com/cncf/toc/issues/1739`.

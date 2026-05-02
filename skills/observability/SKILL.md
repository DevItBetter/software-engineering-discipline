---
name: observability
description: "How to make a running system understandable from the outside — logs, metrics, traces, events, SLOs, alerting, and the discipline that separates real observability from a wall of green dashboards. Use this skill whenever the task involves instrumenting code, designing telemetry, choosing what to log/measure/trace, evaluating whether a system is operable, defining SLIs/SLOs, judging an alerting strategy, debugging a production issue from telemetry alone, or asking \"can on-call diagnose this at 3am.\" Use it when reviewing code that adds logs, metrics, or spans; when reviewing a design that doesn't address operability; and when deciding between telemetry standards. Anchored in the Google SRE corpus, Sridharan, Majors and the Honeycomb school, Dapper, W3C Trace Context, OpenTelemetry, USE, and RED."

---

# Observability

Observability is the property of a running system that lets a human or tool answer questions about its current state — including questions nobody anticipated — from telemetry the system emits. The point is not "can I see graphs?" but **"can I figure out what's happening when something I didn't predict goes wrong, without shipping new code first?"**

The goal is operable systems. Operable means an on-call engineer at 3am with no prior context can identify what is broken and stop the bleeding. Software that emits a thousand metrics but cannot answer "which customer's requests are failing right now" is well-instrumented and not observable.

## Origin and definition

The word comes from Kálmán's 1960 control-theory paper: a system is observable if its internal state can be reconstructed from a finite history of its outputs. Software borrows the metaphor — Kálmán's mathematical property is binary; software observability is a graded operational quality.

Charity Majors' working definition (Twitter, 25 February 2020; expanded in *Observability Engineering*, O'Reilly 2022) is the cleanest contemporary one: the ability to ask any question of your systems, or understand any user's behavior or subjective experience, **without having to predict that question in advance**. The emphasis on unanticipated questions is what separates observability from monitoring.

**Monitoring** answers known-unknowns: values you predicted you would need (CPU > 80%, request rate dropped, this queue is filling). It is necessary and not sufficient.

**Observability** answers unknown-unknowns: novel system states you did not predict, often by exploring high-cardinality, high-dimensionality data ad-hoc.

The two are complementary, not a substitute for each other. Cindy Sridharan's framing (*Distributed Systems Observability*, O'Reilly Report sponsored by Humio, 2018; and *Monitoring and Observability*, 2017) is the most useful: monitoring is what gives you the panoramic view of system health; observability is what gives you the granular insight to debug a state you have not seen before. A real production system needs both.

## Pillars vs events — the live debate

The "three pillars" shorthand — logs, metrics, traces — became dominant through APM-vendor marketing in 2017–2018. Peter Bourgon's *Metrics, tracing, and logging* (21 February 2017) is an early and influential articulation of the three telemetry types as distinct domains; he uses the word "domains," not "pillars." Sridharan's 2018 report discusses the three types but Sridharan herself has been public that she finds the rigid "pillars" framing reductive.

Charity Majors and the Honeycomb school argue the pillars are the wrong unit because they shred each request across three disconnected systems, destroying the connective tissue. Their counter-proposal: the unit is a **wide structured event** — one arbitrarily-wide event per service per request, from which metrics, traces, and logs can all be derived (or directly queried). Honeycomb publishes case studies of customers with hundreds of dimensions per event.

Both framings are useful. Pillars is the lingua franca; you will see it in vendor docs, dashboards, and team conversations. Wide-events is closer to how senior practitioners actually debug: "show me every request from customer X that hit build Y in region Z and took longer than 800ms." If your tooling can't answer that without writing code, the pillars are not enough.

The honest take, used in this skill:
- **Metrics** are cheap, aggregatable, and right for SLO math and dashboards. They are hostile to ad-hoc questions because they pre-aggregate.
- **Logs** are rich, discrete, and right for forensics. They mislead when used as a database for things that should be metrics.
- **Traces** are irreplaceable for cross-service latency and causality. They mislead at runtime/language boundaries where context propagation breaks.
- **Wide events** subsume all three when the storage and query engine support high cardinality. The unification is the point.

## Cardinality and dimensionality — the practical core

**Cardinality** = the number of unique values a field can take. **Dimensionality** = how many fields each event has.

High-cardinality fields are the difference between *I can answer this question* and *I have to ship code*. The questions that matter in an incident — "which customer," "which build SHA," "which request ID," "which feature flag set" — are almost always high-cardinality.

Traditional metrics backends (Prometheus, StatsD/Graphite) struggle here by design: each unique combination of `(metric_name, label set)` is its own time series and consumes memory. Adding a new label can up to double the series count — and therefore memory — depending on label correlation. **Cardinality bombs** happen when a label has unbounded values (raw user IDs, full URL paths with embedded IDs, error messages, pod hashes, build SHAs on long-running tenants).

Practical rules:

- **Always tag**: `service.name`, `service.version`, `deployment.environment`, `region`, request method, response status class, well-bounded enums. These are low-cardinality and stay so.
- **Tag with care, in the right system**: customer_id, tenant_id, route template (not raw URL). Often the question you want to ask — but route to an event store / wide-event backend, not a metrics-only store. **If you only have a metrics store, do not tag customer_id, period** — the cardinality will catch up with you.
- **Never tag in a metrics backend**: raw URLs with IDs, raw error messages, stack traces, anything user-controlled, anything PII.
- **Treat label budget as a real budget**. Track it. New labels need a sponsor.

If your platform makes high cardinality cheap (event-store backends, columnar stores, exemplars in metrics), use it. If it doesn't, separate the high-cardinality questions into the system that can answer them rather than fighting your metrics store.

## SLIs, SLOs, error budgets

The vocabulary, from Google's *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy; O'Reilly 2016):

- **SLI** (Service Level Indicator) — a quantitative measure of one aspect of service quality (latency, availability, throughput, durability). The *Site Reliability Workbook* (2018) standardizes SLIs as a ratio: `good events / valid events`. Choosing an SLI = choosing what counts as "good" and what counts as "valid."
- **SLO** — a target value or range of values for an SLI, measured over a window. "99.9% of valid requests complete in under 500ms over a rolling 28 days."
- **SLA** — an explicit contract with users containing consequences (financial or otherwise) for missing SLOs. SLAs are a business concept; SLOs are an engineering one. Don't conflate them.
- **Error budget** = `1 − SLO`. The amount of failure the SLO permits. The mechanism: while budget remains, ship faster; when it's exhausted, slow down and invest in reliability. Makes the product/reliability tradeoff legible to both sides.

Choosing SLIs is a discipline. The defaults that hold up:
- For a request/response service: **availability** (fraction of valid requests served without 5xx), **latency** (fraction of valid requests under target).
- For a data pipeline: **freshness** (fraction of records processed within target lag), **correctness** (fraction of records that pass validation).
- For a queue: **throughput**, **age of oldest unprocessed item**.

What goes wrong: SLOs set without consulting users (the percentile that matters is the one users feel); SLOs that measure infrastructure rather than user experience; SLOs everyone ignores because they were never tied to a release decision.

### Burn-rate alerting

The *SRE Workbook* chapter on alerting on SLOs replaces threshold alerts ("CPU > 80%") with multi-window, multi-burn-rate alerts derived from the SLO itself. **Burn rate** = how fast you are consuming the error budget relative to the SLO window. Burn rate of 1 means you exhaust the budget exactly at the end of the SLO window. Burn rate of 14.4 means at the current rate you'll exhaust the budget in about 1/14.4 of the SLO window.

The canonical recipe (SRE Workbook, Table 5-8) uses a **30-day SLO window** and specific budget-consumption thresholds:

| Severity | Long window | Short window | Burn rate | Budget consumed |
|---|---|---|---|---|
| Page | 1 hour | 5 min | 14.4 | 2% |
| Page | 6 hours | 30 min | 6 | 5% |
| Ticket | 3 days | 6 hours | 1 | 10% |

Heuristic: short window = 1/12 of long window; both must exceed threshold to fire. Cuts noise; clears quickly when the incident ends. **The 14.4 / 6 / 1 burn rates and the 2% / 5% / 10% consumption levels are not magic constants**. Re-derive burn rates when the SLO window or budget-consumption policy changes. When only the SLO target changes, the burn-rate values can stay the same, but the absolute bad-event threshold changes because the error budget is larger or smaller. See `references/slo-and-burn-rate-alerting.md` for the math.

## RED, USE, Golden Signals

Three overlapping monitoring frameworks. Pick the one that fits the thing you are measuring; use all three across a system.

- **RED** (Tom Wilkie, Weaveworks, ~2015) — **Rate, Errors, Duration**. For services: requests per second, failing requests, latency distribution. The microservice-era complement to USE.
- **USE** (Brendan Gregg, "Thinking Methodically about Performance," ACM Queue 2012; brendangregg.com/usemethod.html) — **Utilization, Saturation, Errors**. For resources: CPU, memory, disk, network, file descriptors. Saturation (queue depth, run-queue length) is the leading indicator of trouble.
- **Four Golden Signals** (Beyer et al., *SRE* book, Chapter 6) — **Latency, Traffic, Errors, Saturation**. The synthesis: services emit Latency/Traffic/Errors; the resource backing them emits Saturation. Latency must distinguish successful from failed requests, or a fast-error path will hide real failure as "good latency."

Senior practice: **RED at every service boundary, USE at every resource, Golden Signals at the SLO level.** No single framework covers everything; the gap between them is where outages live.

## Structured logging

Free-text logs require regex archaeology. Structured logs (JSON, logfmt, or any consistent key/value format) let you query, aggregate, and correlate.

The most useful pattern is the **canonical log line** (Brandur Leach / Stripe, 30 July 2019): emit *one wide structured log line per request* at the end of each request, containing request_id, user_id (or tenant_id), route, method, status, total duration, downstream call counts, feature-flag set, and anything else relevant. One wide event per request beats fifteen scattered log lines for forensics. Note: a canonical log line is still a **log** queried with log tooling; a true wide-event store (columnar, ad-hoc query) is a different backend with different query economics. The two are related disciplines, not the same thing. The canonical log line gets you most of the forensic value in any logs stack; only consider it once request volume makes scattered logging expensive.

### Log levels

The traditional ladder (TRACE/DEBUG/INFO/WARN/ERROR/FATAL) is widely abused in two directions: everything at INFO (no signal) or everything at DEBUG ("we'll turn it on when needed" — except by then the bytes are gone). A defensible per-level discipline:

- **ERROR** — a human probably needs to look. Rare, deduplicated, includes context for triage.
- **WARN** — unexpected but recoverable. Aggregable; not page-worthy on its own.
- **INFO** — canonical events at decision points and request boundaries. This is where the canonical log line lives.
- **DEBUG** — developer-time detail. Off in prod by default; sampled when on.
- **TRACE** — function-level breadcrumbs. Only useful with sampling and only in select environments.

If two engineers can't agree which level a given message should be, the levels aren't doing their job. Document the ladder explicitly; review it.

### PII and secret hygiene

Logs are a privileged side channel that bypass normal security controls (encryption-at-rest, scrubbing pipelines, redaction in transit). Two structurally-identical incidents the same week in May 2018 illustrate: **GitHub disclosed on 1 May 2018** that a recently-introduced bug had caused plaintext passwords to be written into internal logs during password resets; **Twitter disclosed on 3 May 2018** a comparable plaintext-password log incident. Neither was publicly exposed but both required forced password resets. The lesson is structural: assume the log pipeline is the leakiest part of your system.

Defenses:
- Redact at the **logging library boundary**, not at the consumer. Once it's in the log stream it's everywhere. (If your language's logging library does not support redaction at the boundary, that is a precondition to fix before adding new fields.)
- Allowlist, not denylist, which fields can be serialized. New fields default to redacted.
- Never log full request or response bodies for auth-bearing endpoints.
- Never log secrets, tokens, full credit card or SSN values, or session identifiers that can be replayed.
- Compliance regimes (GDPR, HIPAA, PCI-DSS, SOC 2) impose retention, access-control, and right-to-erasure obligations on log data. Logs that contain PII or PHI are a legal artifact, not just an operational one. Treat the log pipeline as in-scope.

### Sampling

- **Head-based sampling** decides at the start of the request whether to record. Cheap and deterministic. Cannot sample-on-error because you don't know yet.
- **Tail-based sampling** records everything, then decides at the end. Catches anomalies (errors, slow requests). Requires buffering — typically in an OpenTelemetry Collector with the `tail_sampling` processor.

Production pattern at scale: head-sample at high rate at the edge (10–100% depending on volume), then tail-sample in the Collector with policies like "keep all errors, keep all spans over the p99 latency threshold, keep 1% of the rest."

Tail-based sampling is not free: the Collector must buffer all spans of a trace until a decision deadline (`decision_wait`, typically ~30s in the OTel `tail_sampling` processor) and any spans arriving after that deadline are silently dropped. Memory budget per Collector scales with span rate × decision_wait × average span size. Below roughly a few hundred spans per second, head sampling is usually sufficient and the Collector tax is wasted. Reach for tail sampling when scale or anomaly-recall demands it.

## Distributed tracing

The *Dapper* paper (Sigelman, Barroso, Burrows, Stephenson, Plakal, Beaver, Jaspan, Shanbhag; Google, 2010) established the modern model: traces are trees of spans; each span is a unit of work with start time, duration, parent, attributes, and events. Dapper's design goals — ubiquitous deployment and continuous monitoring — required engineering for low overhead (aggressive sampling) and application-level transparency (instrumentation in shared infrastructure libraries). OpenTracing, OpenCensus, Zipkin, Jaeger, and OpenTelemetry all inherit this model.

### Context propagation

A trace is only as good as its weakest link. **W3C Trace Context** Level 1 became a W3C Recommendation on **6 February 2020**, with a revised Recommendation dated 23 November 2021. It defines two HTTP headers:

- `traceparent` — version, trace-id, parent-id, trace-flags. The required carrier.
- `tracestate` — vendor-specific extensions.

Use Trace Context end-to-end. The most common debugging frustration is a trace that ends mid-stack because some library, queue, or async hop didn't propagate context.

### OpenTelemetry

OpenTelemetry is the CNCF project standardizing vendor-neutral telemetry: an **API**, an **SDK**, **semantic conventions**, and the **OTLP** wire protocol. Formed in 2019 from the merger of OpenTracing and OpenCensus; CNCF Incubating since August 2021, with a graduation review in progress (filed May 2025).

The maturity level shifts; before pinning architecture decisions to it, **verify current status at `https://www.cncf.io/projects/opentelemetry/` and signal-by-language stability at `https://opentelemetry.io/status/`**. As a rough snapshot:

- **Tracing**: spec stable; SDKs Stable in most major languages.
- **Metrics**: spec stable; Stable in most major SDKs.
- **Logs**: most recent signal to stabilize; spec stable, SDK stability still landing in many languages — be cautious about adopting OTel logs SDKs in greenfield production systems without checking the language-specific status.

**Semantic conventions** are the cross-vendor agreement on attribute names: `service.name` (the only required Resource attribute), `http.request.method`, `http.response.status_code`, `db.system.name`, `messaging.system`, etc. Without them, correlating data across vendors is string-mangling. HTTP semantic conventions reached stable in spec v1.23.0 (2023). Hub: `https://opentelemetry.io/docs/specs/semconv/`.

### Common tracing failure modes

- Missing context propagation across in-process queues, goroutines, or thread pools — trace appears to end mid-request.
- Language and runtime boundaries (sync-to-async, native-to-managed) where libraries disagree on the carrier format.
- Span attribute explosion — attaching unbounded user input as attributes recreates the cardinality bomb at the trace backend.
- Head sampling that systematically misses the bad request because the sampler decided not to record it before the error happened.

## Alerting discipline

Two canonical sources: Rob Ewaschuk's *My Philosophy on Alerting* (Google SRE, ~2014, included as appendix to the SRE book) and Mike Julian's *Practical Monitoring* (O'Reilly, 2017).

Ewaschuk's principles, distilled:
- *Pages should be urgent, important, actionable, and real.*
- *Err on the side of removing noisy alerts.* Over-monitoring is harder to recover from than under-monitoring.
- **Alert on symptoms, not causes.** Symptoms are what users experience: errors, latency, unavailability. Causes (CPU, memory, queue depth) belong in dashboards and runbooks, not pages.
- **The "is a human required?" test.** If the response is mechanical, it should be automated, not paged.

The cause-based-paging antipattern: a single incident fires dozens of correlated cause alerts (CPU high, queue deep, error rate up, latency up). On-call ignores or auto-acks. Real incidents get missed. Industry consistently lists alert fatigue as a top SRE concern.

The replacement is **burn-rate alerts** on SLOs (see above). One alert per user-facing impact, at multiple severities, derived from the SLO. Cause-based metrics still exist — they live in dashboards and runbooks linked from each page, where they accelerate triage without paging.

## Cost, latency, and retention as design constraints

Telemetry is not free. Three constraints that drive real architectural decisions:

- **Vendor and storage cost.** Cardinality is the dominant cost driver in modern observability bills, often into seven figures per year at scale. A new high-cardinality label is a budget commitment. Track the bill the same way you track infrastructure cost.
- **Instrumentation latency.** Synchronous span emission, log serialization, and metric registration all run in your hot path. Adding 1ms of in-line trace export to a 10ms request is a 10% latency tax. Use async batching, in-process buffers, and sampling decisions made off the hot path.
- **Retention tiers.** A common production pattern: hot (queryable, full fidelity) for ~7 days, warm (sampled or aggregated) for ~30 days, cold archive (compressed, slow query) for compliance windows. Telemetry retention drives both cost and what questions you can answer six months later.

Treat these as first-class architectural concerns, not as afterthoughts to be optimized when the bill arrives.

## Beyond logs / metrics / traces

The pillars-or-events debate is about request-level telemetry. Several adjacent disciplines are also part of a serious observability stack:

- **Continuous profiling** (Pyroscope, Parca, eBPF-based agents) — sometimes called the fourth signal. Always-on CPU/memory/lock profiling sampled across the fleet; the way you find regressions and saturation that traces don't reveal.
- **Real User Monitoring (RUM)** — telemetry from actual user devices: Core Web Vitals, time to interactive, JS errors, crash reports, session traces. Backend SLOs alone miss the half of the experience that happens in the user's browser or app.
- **Synthetic monitoring** — scripted probes that exercise critical journeys from outside the system. Catches outages where there is no real user traffic to fail.
- **eBPF-based observability** (Pixie, Cilium Tetragon, Coroot, Parca-Agent) — kernel-level instrumentation that captures network flows, syscalls, and CPU profiles without modifying application code. Increasingly the default for new infrastructure platforms.
- **Audit logs** — distinct from operational logs: who did what, when, with what authorization. Driven by compliance regimes (SOC 2, HIPAA, PCI), retention is regulated, and access to the audit log itself is restricted. Don't conflate audit logging with operational logging — the two have different retention, access, and integrity requirements.

A senior observability practice covers all of these, not just the three classical pillars.

## Runbooks and the postmortem feedback loop

Telemetry without a place to act on it stays decoration. Two adjacent disciplines hold the system together:

- **Runbooks.** Each page must link to a runbook. A good runbook is **executable, not prose**: numbered steps an on-call who has never seen this alert can follow, written in the imperative, validated in a game day, owned by the team that owns the alert. Prose runbooks rot; tested runbooks get updated.
- **Postmortems feed observability.** A postmortem that does not produce at least one of (a new metric or alert, a new dashboard panel, an updated runbook, a removed alert that contributed to fatigue) is incomplete. The discipline is structural — the postmortem template should require an "observability follow-ups" section. See `debugging-and-incident-response`.

## SLO composition across services

When one service depends on others to serve a request, your achievable SLO is bounded by the SLOs of the services you call. For three independent dependencies serving a request serially at 99.9% each, the upper bound on your availability is `0.999³ ≈ 99.7%` before you add your own faults. Two consequences:

- Don't promise an SLO higher than the math allows. The number is a budget; the budget is a function of dependencies.
- Critical-path dependencies need higher SLOs than peripheral ones; a single 99% dependency on a critical path drags down everything that calls it. Push hard for tighter SLOs on dependencies that sit on the critical path, or for graceful-degradation paths that let the service stay available when the dependency isn't.

For parallel calls and graceful-degradation paths the math is different and usually more forgiving. See `references/slo-and-burn-rate-alerting.md`.

## Common antipatterns

- **Dashboards-as-archaeology** — walls of gauges nobody reads. Outages take 30 minutes of dashboard browsing to localize. Fix: dashboards organized around SLOs and Golden Signals; everything else is ad-hoc query.
- **Cardinality cliff** — adding a label and OOM-ing the metrics backend. Fix: explicit label budget; route high-cardinality dimensions to an event store.
- **Log-as-database** — querying logs for things that should be metrics ("how many 500s today?") or events ("which customers hit this codepath?"). Fix: emit canonical events; derive metrics from them; reserve raw logs for forensics.
- **Trace gaps** — missing context propagation; head sampling that omits the failing request. Fix: enforce W3C Trace Context end-to-end; tail-sample on errors and tail latency.
- **Alert spam, runbook rot, on-call as graveyard** — alerts accumulate, runbooks drift, tribal knowledge stays in DMs. Fix: alert review per on-call rotation; runbooks as code, linked from each alert; postmortems update runbooks.
- **Three-pillars silos** — logs in tool A, metrics in tool B, traces in tool C, no shared identifier. The single biggest debugging tax. Fix at minimum: propagate trace_id into log records and exemplars into metrics. Better: shared semantic conventions through OpenTelemetry. Best: one wide-event store.
- **Logging the response payload of an auth endpoint** — see PII and secret hygiene above. This is how secrets leak.
- **"We'll add observability later"** — instrumentation is a design property, not a finishing step. Retrofitting telemetry into a system that wasn't designed for it produces gaps where they hurt most.

## What to flag in review

- A new service or endpoint with no SLI defined and no canonical log line at request boundaries.
- A new metric with a high-cardinality label (user_id, raw URL, error message string).
- A new log line that includes a request body, response body, token, or full PII field.
- A retry or backoff loop with no metric on retry count and no trace span per attempt.
- A new alert that pages on a cause (CPU, memory, queue depth) rather than a symptom (errors, latency).
- A trace that crosses an async boundary (queue, thread pool, background job) without explicit context propagation.
- A change to a hot path with no exemplar wiring metrics to traces.
- A "monitoring" claim in a design doc with no SLI/SLO defined.
- A dashboard added without a question it answers.
- An ERROR-level log used for an expected, recoverable condition.
- A new dependency on a paid telemetry vendor with no consideration of cardinality cost (the bill becomes a design constraint).

## What NOT to flag

- Reasonable INFO logs at request boundaries.
- Metrics with bounded enum labels (well-defined error categories, status code classes).
- Traces that intentionally sample heavily on a high-volume codepath if errors are still captured.
- Vendor-specific instrumentation in a codebase where switching is not a real concern (premature OTel migration is a YAGNI tax).

## Reference library

- `references/slo-and-burn-rate-alerting.md` — SLI selection, error budget math, multi-window/multi-burn-rate derivations, common SLO mistakes.
- `references/wide-events-and-cardinality.md` — high-cardinality discipline, label budgets, when to use a metrics store vs an event store, exemplars.
- `references/tracing-and-otel.md` — Dapper data model, W3C Trace Context, OpenTelemetry signal stability and semantic conventions, sampling strategies.

## Sibling skills

- `systems-architecture` — operability, deployment topology, blast radius. Observability is a property of the architecture, not a layer on top of it.
- `debugging-and-incident-response` — how to use telemetry under fire; postmortem discipline.
- `error-handling-and-resilience` — what to log, where to retry, how to surface failure modes through telemetry.
- `secure-coding-fundamentals` — PII and secret handling at logging boundaries.
- `performance-engineering` — profiling and tracing for hot-path investigations; latency budgets that drive SLOs.
- `database-and-data-modeling` — query plans and slow-query logs are first-class observability surfaces for data systems.

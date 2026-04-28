# Profiling and Measurement

The discipline of producing performance numbers you can trust.

## The metrics that matter

For each system, identify the metrics that match the user impact:

- **Latency**: how long does one operation take?
  - Median (p50): "typical" user experience.
  - p95, p99, p99.9: tail latency. The metric that decides whether your system feels fast or slow.
  - Maximum: outliers; useful for catching catastrophic cases.
- **Throughput**: how many operations per unit time?
  - Saturation: how close to the limit?
- **Error rate**: how often does it fail? (Especially as a function of load — error rate often spikes before latency does.)
- **Resource usage**: CPU, memory, disk, network. Per-instance and cluster-wide.
- **Cost**: dollars per million operations. Often the goal once functionality is stable.

The most underrated of these is **tail latency**. Mean latency hides outliers; p99 reveals them. A system with 100ms mean and 5s p99 will get user complaints. A system with 200ms mean and 250ms p99 will not.

## The setup that produces trustworthy numbers

Performance measurement is hard because the variables that should be constant rarely are. To produce trustworthy numbers:

- **Realistic workload.** Synthetic benchmarks lie. Use a workload that resembles production traffic patterns: similar request mix, similar data shapes, similar concurrency.
- **Realistic data.** Fixture data with 3 records doesn't catch the O(n²) bug at n=10000. Use production-shape data (sanitized), not toy fixtures.
- **Steady state.** Warm up before measuring. JIT compilation, cache fills, connection pool establishment all take time. Discard the first N requests.
- **Statistical significance.** Run multiple times; report mean ± stddev or median + range. A single number is meaningless.
- **Controlled environment.** No other workloads on the machine; consistent network conditions; consistent input data.
- **Reproducible.** Document the exact setup so the measurement can be repeated.

## Profilers — what they tell you, and what they don't

A profiler tells you where CPU time is going (or where allocations happen, or where time is spent in lock contention, depending on the profiler). It does not tell you why.

Types:

- **Sampling profilers** (perf, pprof, async-profiler, py-spy). Periodically sample the call stack. Low overhead. Statistical: tells you which functions are *probably* hot. The default; usually right.
- **Tracing/instrumentation profilers** (XPerf, Java Flight Recorder in some modes). Record every call. High overhead; slows down the program (which can change what's slow). Use sparingly and for short windows.
- **Memory profilers** (heaptrack, dotMemory, JProfiler, Python's tracemalloc). Track allocations.
- **Distributed tracing** (OpenTelemetry, Zipkin, Honeycomb). Show where time goes across services.

Reading profiler output:

- **Flame graphs** are the most useful visualization. Wide bars = where time is spent. Tall stacks = call depth. The "I see a wide pancake at the top" pattern usually points to the hot function.
- **Self time vs total time.** A function with low self time but high total time is mostly waiting on its callees. Look at the callees.
- **Inverted (callee-up) view.** Aggregates time by leaf function across all callers. Useful for finding "this small function is called from everywhere and adds up."

What a profiler doesn't show:
- I/O wait. CPU profilers focus on CPU. For I/O-bound workloads, you need a tracing tool that captures wall time.
- Lock contention (unless the profiler explicitly supports it).
- Network round trips.
- GC pauses (unless the profiler reports them separately).

For an I/O-bound or distributed workload, distributed tracing is more useful than CPU profiling.

## Microbenchmarks: useful and dangerous

A microbenchmark measures a small piece of code in isolation. Useful for:
- Comparing two implementations of the same hot function.
- Catching regressions in a known-hot path.
- Understanding the cost of a primitive (allocation, hash lookup, etc.).

Dangerous because:
- The cache stays hot in a way it doesn't in production.
- The branch predictor learns the test pattern.
- The JIT specializes for the constant inputs.
- Garbage collection during the benchmark contaminates the timing.
- The benchmark may run code the real workload doesn't.

Best practice: use a real benchmark library (JMH for Java, Criterion for Rust, BenchmarkDotNet for .NET, pytest-benchmark for Python). They handle warm-up, statistical analysis, and the common pitfalls.

When in doubt, validate microbenchmark conclusions against real workloads. A 3x microbenchmark improvement that produces a 0.5% improvement in production is common.

## Load testing

For systems under concurrent load, load testing reveals behavior that single-request testing can't:

- **Max sustainable throughput** before latency degrades.
- **Behavior at saturation** (graceful degradation vs. cascade failure).
- **Resource bottlenecks** (CPU, memory, connections, locks, disk I/O, network).
- **Latency under contention** (often very different from latency at low load).

Tools: k6, Locust, Gatling, JMeter, wrk, vegeta, hey.

Methodology:
- Start at low load; ramp up.
- Watch latency, throughput, error rate, resource usage as load increases.
- Find the inflection point where latency or errors start growing nonlinearly.
- Drive past saturation to see failure mode.

A specific test worth running: **bursty load**. Production traffic is rarely smooth; sudden spikes test queue depths, autoscaling, and recovery behavior.

## Distributed tracing

For multi-service systems, where the request goes determines the latency. A trace shows the request's full path, with time per hop.

Key signals to look for in traces:
- **Long spans on specific services** — that's where time is going.
- **Sequential calls that could be parallel** — frequent latency win.
- **Retries** — cascading slowness.
- **Cold starts** — the first request to a service that just spun up.
- **Tail latency contributors** — which service is responsible for p99 spikes.

Sample a small fraction of production traffic; full sampling is too expensive. Capture full traces for slow requests (head-based or tail-based sampling).

## What to do when the profiler shows nothing useful

Sometimes the profiler shows time spread evenly across many functions, with no clear hotspot. The system is just slow.

Possibilities:
- **The bottleneck is outside the profiled scope** (network, GC, lock contention, I/O wait). Use a different tool.
- **The architecture is the bottleneck** (the request fan-out is too wide; too many round trips). Profile won't show this; trace will.
- **The data structure is wrong**. Each operation is fast individually, but the total operation count is too high. Look at counts, not just per-op time.

Sometimes the answer is "the system needs a redesign at this load." Optimization on a fundamentally wrong shape is wasted.

## Cost as a performance metric

For cloud-deployed systems, dollars-per-operation is an honest performance metric. Optimizing for cost often differs from optimizing for latency:

- A cache that halves database cost may not improve latency.
- A more efficient algorithm that uses 10% less CPU translates to 10% less compute spend at scale.
- A change that lets you use cheaper instance types matters even if peak performance doesn't change.

For systems at scale, cost optimization is real engineering — not "buy a smaller instance." Profile spend; identify the most expensive operations; question them.

## Common measurement mistakes

- **Measuring on a development machine**. Different CPU, different network, different load. Numbers don't transfer.
- **Measuring without warm-up**. The first run includes JIT, class loading, cache fill — all one-time costs.
- **Comparing single runs**. Variability between runs can dwarf the change you're trying to measure.
- **Measuring the wrong thing.** "We made the function 50% faster" — but the function is 1% of the request. Net win: 0.5%.
- **Optimizing a microbenchmark that doesn't match production**. The test is wrong; the optimization helps the test, not the user.
- **Believing a profiler that's measuring overhead.** Heavy instrumentation profilers can change the program's behavior. Validate with a low-overhead sampler.
- **Forgetting variance.** Performance numbers always have variance; report the distribution, not just a single number.
- **No baseline.** Without a "before" measurement, you can't claim improvement.

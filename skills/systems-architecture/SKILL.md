---
name: systems-architecture
description: "Design and review systems at the architecture level — service boundaries, data ownership, distributed-systems trade-offs, evolutionary architecture, deployment topology, operability. Use this skill whenever the task involves designing or reviewing a system architecture, RFC, or design doc; deciding how to split a monolith or whether to; choosing between service patterns (request/response, async, event-driven); judging whether a proposed change makes sense at the system level vs. just the code level; evaluating distributed-systems decisions; designing for operability, observability, and failure; or asking \"is this the right architecture.\" Built on Kleppmann (DDIA), Sam Newman (microservices), Eric Evans (DDD), Lampson, Conway, the fallacies of distributed computing, the SRE workbook, and Twelve-Factor."
---

# Systems Architecture

Architecture decisions are the ones that are expensive to undo. A bad function can be rewritten in an afternoon; a bad service boundary can dominate a team's work for years. The discipline of systems architecture is paying disproportionate attention to the decisions that matter disproportionately.

This skill covers system-level concerns: service decomposition, data ownership, distributed-systems trade-offs, evolution, and operability. For implementation-level design (modules, classes, functions), see `software-design-principles`. For specific concurrency and failure-handling patterns, see `concurrency-and-state` and `error-handling-and-resilience`.

## The fallacies of distributed computing

The eight fallacies — assumptions every distributed system designer makes once and then learns better. The first four were articulated by Bill Joy and Tom Lyon at Sun in the early 1990s; Peter Deutsch added the next three in 1994; James Gosling added the eighth around 1997:

1. **The network is reliable.** It isn't. Packets are dropped. Connections fail. TCP handshakes time out.
2. **Latency is zero.** It isn't. Same-region: ~1ms. Cross-region: 10–100ms. Cross-continent: 100–300ms.
3. **Bandwidth is infinite.** It isn't. There's a ceiling and you'll hit it.
4. **The network is secure.** It isn't. Anything you can read, an attacker on the path can read.
5. **Topology doesn't change.** It does. Hosts come and go; routes change; the load balancer in front of you is different from the one yesterday.
6. **There is one administrator.** There isn't. Multiple teams, multiple clouds, multiple companies, all changing things you depend on.
7. **Transport cost is zero.** It isn't. Bytes on the wire have a CPU cost (serialization), a memory cost (buffers), a money cost (egress fees), and a latency cost.
8. **The network is homogeneous.** It isn't. Different paths have different bandwidth, latency, packet loss, and quirks.

Every distributed system that's been in production long enough has been bitten by every one of these. Design assuming all eight are false.

## Reliability, scalability, maintainability (Kleppmann)

Kleppmann's framing in *Designing Data-Intensive Applications*: every system architecture is a balance among three concerns.

- **Reliability**: the system continues to work correctly (correct operation, even at the desired level of performance) in the face of adversity (hardware faults, software faults, human errors). Design for the components that *will* fail (because they will).
- **Scalability**: the system can cope with increased load (more requests, more data, more users). What happens when load grows 10x? 100x?
- **Maintainability**: the system is operable (easy to run), simple (easy to understand), and evolvable (easy to change).

Architecture trades these against each other and against cost. Document the trade-offs explicitly — "we chose X because Y, accepting Z" — so future readers can re-evaluate when conditions change.

## Service boundaries — the most important decision

How you carve a system into services determines:
- Who can ship independently.
- What needs distributed coordination (expensive) vs. local coordination (cheap).
- What can fail independently vs. what cascades.
- What teams can own.
- How hard it is to change later.

The classic question: monolith, microservices, or something in between?

### Monolith

One deployable unit. Everything in one process, talks to itself via function calls.

**Strengths**: simple to develop, simple to deploy, simple to debug, no distributed-systems problems. Refactoring across modules is cheap (compiler/IDE assists). Transactions span the whole system.

**Weaknesses**: scales as a unit (you can't scale just the slow part). Teams can step on each other (one bug brings everyone down). Can grow into a "big ball of mud" without strong internal discipline.

**When right**: small teams, early-stage products, systems where modules don't have wildly different scaling needs, systems where strong consistency across the domain is essential.

### Microservices

Many small deployable units. Talk to each other via RPC, REST, or messaging.

**Strengths**: independent deployment (one team's bug doesn't take everyone down), independent scaling (scale the hot service), independent technology choice, clear team boundaries.

**Weaknesses**: distributed systems all the time. Every cross-service call can fail, time out, retry. Tracing problems across services is hard. Data consistency across services requires sagas (no distributed transactions). Operability is much harder.

**When right**: organizations large enough that team coordination is the bottleneck, services with genuinely different scaling needs, components that need independent release cycles.

**When wrong**: small teams (microservices add overhead disproportionate to their benefit), domains where strong consistency is essential, teams without operational maturity to run many services.

### Modular monolith

One deployable unit, but with internal boundaries that *could* be services. Sam Newman has consistently argued — in *Building Microservices* and *Monolith to Microservices* — that you should not move to microservices before achieving a well-modularized monolith, because the service boundaries you'd extract are exactly the module boundaries you haven't yet drawn. Most "microservices migration" projects would be better served by first making the monolith modular.

**When right**: nearly always a good first step. Forces you to discover the right boundaries before you commit to the operational complexity of services.

### Service-oriented architectures (SOA), broader than microservices

Coarser-grained services, often with shared data layers. The 90s/2000s pattern. Sometimes the right answer when "microservices but coarser" matches the team structure.

## Bounded contexts (DDD)

From Eric Evans's *Domain-Driven Design*: a **bounded context** is a boundary within which a model is consistent. Outside the boundary, the same word may mean different things.

The classic example: "Customer" means different things to Sales (a prospect being qualified), Billing (an entity with a payment method), and Support (an entity with tickets). One unified `Customer` object trying to serve all three is a divergent-change disaster; three `Customer` types in three bounded contexts, with explicit translation between them, is honest.

Use bounded contexts as the candidate boundaries for services. A bounded context that's owned by one team, has its own data, and changes for one set of reasons is a strong service candidate. A boundary that doesn't match a context will be either too coarse (one service for two contexts, which evolve at different rates and need different shapes) or too fine (two services for one context, requiring coordination on every change).

The mapping between contexts (where they need to talk) is the **context map**. The patterns of integration (shared kernel, partnership, customer-supplier, conformist, anti-corruption layer, open-host service, published language, separate ways) describe how contexts relate. See Vernon's *Implementing Domain-Driven Design* for the catalog.

The **anti-corruption layer** is particularly useful: when integrating with a system whose model differs from yours (especially a legacy or external system), the anti-corruption layer translates between the models so your domain isn't polluted by theirs.

## Data ownership

Per piece of data: one service owns it. That service is the source of truth; others read or subscribe.

Why this matters:
- **Two services writing the same data** = race conditions, lost updates, divergence. Eventually you'll need a single source of truth; design for it from the start.
- **Reads from any service** can be cached, replicated, materialized. Reads scale.
- **Writes to one service** keep consistency tractable.

The anti-pattern: two services both writing to the same database table directly. Tightly coupled at the data layer; the schema becomes a shared contract you can't evolve; bugs in one service corrupt data the other relies on.

The fix: one service owns the data. The other gets it via API call, async event, or read-only replica. The owner's database is its private implementation.

For complex domains, this leads to **CQRS** (Command-Query Responsibility Segregation): the write side owns the data and enforces invariants; the read side has materialized views optimized for query patterns. Each can scale independently. Often combined with event sourcing.

## Hexagonal / Ports and Adapters / Clean Architecture

A layered architecture where:
- **The domain** (entities, business rules) is at the center.
- **Application services** (use cases) wrap the domain.
- **Adapters** (HTTP handlers, database repositories, message consumers, external service clients) sit at the boundary.

The **dependency rule**: dependencies point inward. Domain doesn't know about HTTP, the database, or any framework. Adapters depend on the domain (or on interfaces the domain defines), never the other way.

Benefits:
- Domain logic is testable without infrastructure.
- Infrastructure can be swapped (different database, different HTTP framework, different message queue) without touching the domain.
- The domain is honest about what it depends on (small interfaces it defines).

Costs:
- More upfront structure than "just put it all in the controller."
- More files; more indirection.

When right: substantial domains, long-lived systems, systems that may need to support multiple input/output mechanisms (HTTP + CLI + scheduled job).

When overkill: trivial CRUD endpoints, prototypes.

For internal modules within a service (not just inter-service), the same dependency-rule discipline produces cleaner, more changeable code regardless of whether you adopt the full hexagonal pattern.

## Twelve-Factor App

Originally for Heroku, now the de facto baseline for cloud-native applications. The twelve factors:

1. **Codebase** — one codebase per app, in version control, many deploys.
2. **Dependencies** — explicitly declared and isolated (no system packages assumed).
3. **Config** — in environment variables, not in code or files.
4. **Backing services** — treated as attached resources (DB, queue, cache reachable via URL/credentials, swappable).
5. **Build, release, run** — strictly separated stages; release is build + config and is immutable.
6. **Processes** — stateless; share-nothing.
7. **Port binding** — exports services by binding to a port; no app server needed in front.
8. **Concurrency** — scale via the process model (more processes), not threads in one process.
9. **Disposability** — fast startup, graceful shutdown.
10. **Dev/prod parity** — dev, staging, prod as similar as possible.
11. **Logs** — write to stdout; environment routes them.
12. **Admin processes** — one-off admin tasks run as one-off processes in the same release.

These are operational discipline that maps cleanly to modern container/Kubernetes/serverless platforms. Violations make the app harder to run reliably. Code review can flag specific violations: "this hardcodes a config value that should be in env vars (factor 3)"; "this writes to a local file that won't survive a restart (factor 6)"; "this has a slow startup that will fail health checks under autoscaling (factor 9)."

## Designing for operability

A system that works in development is not the same as a system that runs in production. Operability is a first-class concern:

- **Observable.** Logs (with context), metrics (with cardinality discipline), traces. Every failure mode has a signal — see "observability as design" below.
- **Recoverable.** What happens when the database is down? When a region fails? When the deploy fails? Each scenario has a documented response.
- **Deployable.** Zero-downtime deploys. Canary or blue-green for risky changes. Easy rollback. See "progressive delivery" below.
- **Configurable.** Behavior tunable without redeploying via feature flags.
- **Documented.** Runbooks for known failures. Architecture docs current. On-call onboarding exists.

Design reviews that don't include an operability section have skipped half the job. "How will this be operated?" is a real architecture question.

## Observability as design (not as debugging)

Observability is the property of being able to ask new questions about your system without shipping new code. It is built in at design time, not bolted on after an incident.

The three pillars (per the conventional framing):

- **Logs** — discrete events with structured context. Every error path logs the inputs needed to diagnose. Cardinality is OK; volume should be controlled.
- **Metrics** — numeric time series, low-cardinality. Latency, throughput, error rate, resource usage. Per-endpoint, per-tenant, per-version. The basis for SLIs and alerts.
- **Traces** — the request's journey across services. Reveals where latency goes, where errors originate, where retries happen.

A fourth pillar increasingly recognized:

- **Profiles** — periodic CPU/memory profiles in production, sampled cheaply (eBPF, continuous profilers). The basis for "where is the time going right now."

Design-time decisions:

- **What's the SLI for this service?** Define it before the service ships, not after the first incident.
- **What context does each log line need to be useful?** "Failed to fetch user" is useless; "Failed to fetch user (user_id=X, source=Y, latency=Z, error_class=W)" is actionable. Structured logging (JSON) by default.
- **What's the cardinality model for metrics?** Per-tenant metrics are useful for big customers but explode storage cost; pick the dimensions deliberately.
- **What's the trace propagation model?** Every cross-service call passes the trace ID. Internal jobs and async work also.
- **What's the retention?** 24-hour high-resolution; 90-day downsampled; as cheap and as long as the budget allows.

The litmus test: when a new failure happens that you didn't anticipate, can you investigate it without shipping new code? If the answer is no for important paths, the observability is a design gap, not a debugging issue.

## Progressive delivery

The discipline of decoupling deploy (releasing code) from release (turning behavior on for users). Risky changes ship dark; behavior is enabled gradually.

Patterns:

- **Feature flags** (LaunchDarkly, OpenFeature, Unleash, in-house). Code paths conditional on a runtime flag. Flag can be flipped instantly (kill switch) without redeploying.
- **Canary deploys.** New version goes to a small fraction of traffic; monitor; ramp up if metrics look good; roll back if not.
- **Blue/green deploys.** Two production environments; switch traffic atomically; instant rollback by switching back.
- **Shadow traffic / dark launching.** New code receives production traffic but its results are discarded (or compared to old). Validates behavior under real load before becoming user-visible.
- **Ring deploys.** Internal users → trusted customers → general availability, in stages.

The point is not "deploy more carefully" — it's to make the unit of risk a flag flip, not a deploy. Deploy frequently, change behavior carefully.

For each significant change in an architecture review: is there a rollout plan? Is there a kill switch? What's the metric that says it's working? What's the rollback path?

Long-lived flags are themselves technical debt — every flag is a permanent if-statement and a permanent test matrix. Track them; retire them on a schedule.

## Chaos engineering

The discipline of deliberately breaking things in production-like environments to verify the system actually behaves as designed under failure.

The reasoning: a fault-tolerance design that's never been exercised is fiction. The first time the database fails for real is a poor moment to discover the failover doesn't actually work.

What to chaos-test:
- **Process death.** Random kill -9 of a service instance. Does autoscaling/restart work? Do other instances handle the traffic?
- **Network failure.** Inject latency, drops, partitions between services. Do timeouts and retries work? Does the circuit breaker trip?
- **Dependency failure.** Make the database unavailable, the cache cold, an upstream slow. Does graceful degradation kick in?
- **Region failure.** Take down an entire region. Does failover work? How long?
- **Disk full, OOM, certificate expiry.** The unglamorous failures that take down real systems.

Tools: Chaos Monkey (Netflix's original), Gremlin, ChaosToolkit, Litmus (Kubernetes), AWS Fault Injection Service.

For high-availability systems, periodic chaos exercises (or "game days" — scheduled drills with the on-call team) are not optional. They are the only honest way to know your recovery actually works.

## Choose Boring Technology

(Term from Dan McKinley's essay of the same name.) For each piece of technology in your architecture, you spend an "innovation token." The total budget is small — maybe three tokens for a new service. Spend them where the novelty actually buys you something.

The default for most components: choose the boring, well-understood, widely-deployed option.

- The boring database (Postgres or MySQL).
- The boring queue (Redis, SQS, the built-in tools).
- The boring cache.
- The boring deploy mechanism.

Reserve novelty for the few places where the new thing genuinely solves a problem the boring thing can't.

Why: the boring thing has known failure modes, plentiful operators who can run it, abundant documentation, and a long history of bugs being shaken out. The new thing has unknown failure modes, scarce expertise, and bugs you'll discover at 2am.

For an architecture review: every novel technology in the design should justify its innovation cost. "It's the new hotness" is not a justification.

## Managing technical debt

A useful metaphor (Ward Cunningham) — treating known compromises as debt with interest. Like financial debt, it's not always bad: borrowing time today to ship now and refactor later is a legitimate trade-off. Like financial debt, it is bad to ignore: interest compounds, eventually you can't afford new features because the old debt is consuming all your capacity.

Disciplines:

- **Make debt visible.** If "we'll fix this later" comes up, write the ticket. Tag it. Estimate it.
- **Distinguish kinds of debt.** Deliberate-prudent ("we know this is a hack; here's why we shipped it"), deliberate-reckless ("we don't have time for design"), inadvertent-prudent ("now we know what we should have done"), inadvertent-reckless ("we didn't know what we were doing"). The first kind is fine; the others need different responses.
- **Budget repayment.** Some teams budget 20% of capacity for debt reduction. Some interleave debt cleanup with feature work. Either is fine; ignoring debt isn't.
- **Pay down debt that's actively costing you.** Not all debt has to be paid; some can be left until the code is rewritten or deleted. Pay down debt that's slowing the team or causing incidents.
- **Don't bankrupt the project.** When debt service exceeds the team's velocity for new work, you're heading for the place where systems get rewritten from scratch (almost always more expensive than incremental cleanup).

For architecture review: a design that bakes in known debt should say so explicitly. "We're shipping this with limitation X; we'll address it in a follow-up by date Y." Hidden debt is much worse than visible debt.

## SLOs, SLIs, and error budgets

From Google's SRE practice:

- **SLI** (Service Level Indicator): a quantitative measure of service quality. "Fraction of requests that complete in <200ms with no error."
- **SLO** (Service Level Objective): the target for the SLI. "99.9% of requests complete in <200ms with no error, measured over 28 days."
- **Error budget**: 100% minus the SLO. If your SLO is 99.9%, your error budget is 0.1% — about 43 minutes of downtime per month.

The error budget shifts the conversation from "is the service up?" (binary) to "how much budget have we spent?" (continuous, actionable). When the budget is exhausted, freeze risky changes; restore reliability before continuing feature work.

For each new service in an architecture review: what's the SLO? What's the SLI that measures it? What's the alert when error budget burns too fast?

## Evolutionary architecture

Architecture is not a one-shot decision. The system will evolve. Design for evolution:

- **Versioned interfaces** so producers and consumers can change independently.
- **Strangler fig** for migrating off an old system: build the new alongside; route traffic gradually; decommission the old.
- **Feature flags** so risky changes can be rolled out and back.
- **Schema evolution** with backward-compatible migrations.
- **Anti-corruption layers** at boundaries to old/external systems.

The wrong question: "what is the final architecture?" The right question: "what architecture lets us learn what we don't know yet, and change?"

## Conway's Law (re-emphasized)

Architecture mirrors organization. (See `software-design-principles` for the depth treatment.) When designing services, the team that owns each service must be identified. Services without owners rot. Services that don't match team boundaries erode under cross-team friction.

For an architectural change that doesn't match the org: either the org changes, or the architecture won't survive.

## What to flag in an architecture review

- **No problem statement.** What problem are we solving? What's the success criterion?
- **No alternatives considered.** What other approaches were rejected and why?
- **No data ownership story.** Who is the system of record for each piece of state?
- **No failure modes section.** What happens when X is down?
- **No operability section.** How is this operated? On-call? SLO? Runbook?
- **No evolution story.** How does this version up? How does it migrate off if it's wrong?
- **Distributed-systems assumptions.** Does the design assume the network is reliable? Latency is zero? A service is always available?
- **Strong-consistency assumption on top of an eventually-consistent system.**
- **A new service without a clear owning team.**
- **A new service boundary that doesn't match a bounded context or a team boundary.**
- **A "platform" or "framework" without a concrete near-term consumer.**
- **A change to a published distributed protocol** (Raft, Paxos, gossip) without a model-checked proof.
- **A migration that goes through a half-done state visible to users.**

## Reference library

- `references/distributed-systems-deep-dive.md` — CAP/PACELC, consensus, replication, partitioning, transactions, in honest depth.
- `references/architecture-decision-records.md` — how to document decisions so future readers can re-evaluate them.

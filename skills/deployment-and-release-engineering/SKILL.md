---
name: deployment-and-release-engineering
description: "How to ship software safely and often — continuous delivery, deployment strategies (rolling, blue/green, canary, shadow, dark launch, progressive delivery), feature flags, rollback discipline, schema-migration patterns, and the cultural practices that separate strong delivery teams from weak ones. Use this skill whenever the task involves designing or evaluating a release strategy, choosing a deployment pattern, planning a schema or data migration, designing a feature flag, judging an alerting/rollback policy for a rollout, evaluating CI/CD pipelines, or asking \"how do we ship this without breaking production.\" Use it when reviewing changes to deployment manifests, pipeline definitions, feature-flag wiring, or migration scripts. Anchored in Humble & Farley's *Continuous Delivery*, Forsgren/Humble/Kim's *Accelerate* (DORA research), the Google SRE corpus, Pete Hodgson on feature toggles, Danilo Sato's parallel change, and Werner Vogels' \"you build it, you run it.\""

---

# Deployment and Release Engineering

The discipline of getting code from commit to production safely, repeatedly, and reversibly. The bar is not "we deployed it"; it is "we deployed it, observed it, and could roll it back inside minutes if it misbehaved."

Research from *Accelerate* and DORA consistently finds that, in many software-delivery organizations, high deployment frequency correlates with lower change failure rate and faster recovery. The safer lesson is not "ship recklessly"; it is that small, reversible, well-observed batches usually reduce risk better than large, infrequent releases.

## Continuous delivery vs. continuous deployment — get this distinction right

Most secondary writing collapses these. The honest framing, from Humble & Farley's *Continuous Delivery* (Addison-Wesley, 2010) and Martin Fowler's bliki:

- **Continuous Delivery** is the discipline of building software so that the artifact is *always deployable*. Whether to ship is a business decision; the technical capability is continuous.
- **Continuous Deployment** is the further step where every commit that passes the pipeline automatically reaches production. Every CD-deployment shop is a CD-delivery shop; the inverse is not required.

The trap: teams under pressure to "do CD" feel obligated to wire commits to production. They aren't — they're obligated to make sure they *could*. The discipline of CD-the-delivery is the load-bearing one; CD-the-deployment is a policy choice on top.

## DORA software-delivery metrics

From *Accelerate* and the DORA program, the software-delivery metrics that predict delivery performance and team outcomes:

1. **Deployment Frequency** — how often code reaches production.
2. **Lead Time for Changes** — commit to production.
3. **Failed Deployment Recovery Time (FDRT)** — how fast a failed deploy is recovered. Earlier reports called this Mean Time to Restore (MTTR); DORA renamed it FDRT for precision because MTTR is overloaded with general incident-recovery meanings.
4. **Change Failure Rate** — fraction of deploys that require immediate intervention, such as rollback or hotfix.
5. **Deployment Rework Rate** — fraction of deployments that were unplanned remediation work caused by production incidents.

Treat the metrics as a balanced set. Deployment Frequency, Lead Time for Changes, and FDRT describe throughput; Change Failure Rate and Deployment Rework Rate describe instability. Optimizing one metric in isolation creates bad incentives.

## Deployment strategies

The strategy is the gradient between "deploy and watch" and "fully automate the rollout under telemetry control." Pick the highest-control strategy your traffic, risk, and tooling support.

**Rolling deployment.** Default in container orchestrators (Kubernetes Deployments respecting `maxSurge`/`maxUnavailable`). Replaces v1 instances with v2 incrementally. Cheap, but: **a rolling deploy is not a canary** — there's no separate metric gate, just instance rotation. If v2 is broken, the rollout will happily run to completion regardless of error rate.

**Blue/green deployment.** Two production environments — "blue" live, "green" idle — with a router/load-balancer flip between them. Deploy v2 to green, smoke-test, switch the router, keep blue around for instant rollback. Documented in *Continuous Delivery* (Humble & Farley, 2010, ch. 10); Fowler's bliki credits the name to ThoughtWorks circa 2005, jointly to Daniel Terhorst-North and Jez Humble. Strength: rollback is a router flip. Constraint: schema changes must be split via parallel change so both versions can run against the same database.

**Canary deployment.** Route a small fraction of traffic to v2 (commonly 1% → 5% → 25% → 50% → 100%), watch SLIs, expand or roll back. The metric-watching is the difference between a canary and a rolling deploy. Documented as a software pattern by Danilo Sato on Fowler's bliki (2014); the practice is widely associated with Google and Facebook. Bake time at each stage long enough to detect a regression at the SLO threshold given the traffic volume; short enough that rollouts complete in business-relevant time.

**Shadow / mirroring deployment.** v2 receives a *copy* of production traffic; its responses are discarded. Implemented at the proxy or service-mesh layer (Envoy, Istio `mirror`, NGINX `mirror`). Useful for behavior comparison and load testing under real traffic without user risk. Caveat: stateful side effects (writes, third-party calls, payments, emails) must be stubbed or routed to a sandbox — otherwise shadow doubles real load on downstream systems.

**Dark launch.** New backend behavior is deployed and *executed* in production but its effects are invisible to users. Canonical example: Facebook's 2008 chat launch is widely cited as an early large-scale dark-launch example; Fowler's tighter 2020 definition is that the code runs in production while users cannot tell. Often confused with canaries and feature flags — it is neither. It is the load-test-in-production pattern.

**A/B testing.** Same plumbing as canary; different intent. Canary is *release engineering* (short window, watching system SLIs, defaulting to rollback on regression). A/B testing is *experimentation* (long enough to reach statistical significance, randomized cohorts, business metric outcomes). Don't conflate — the success criteria, monitoring duration, and decision logic are different.

**Progressive delivery.** Coined/popularized by James Governor (RedMonk, 2018), crediting inspiration from Sam Guckenheimer at Microsoft. The umbrella for incremental, policy-driven rollout using practices such as canaries, feature flags, target cohorts/rings, A/B experimentation, observability, and automated hold/rollback where tooling supports it. The natural successor to "continuous delivery" once flag-based gating became table stakes. When the design doc says "progressive delivery," check that the rollout has explicit policy, targeting, and telemetry — not just renamed canary stages.

## Feature flags / feature toggles

Pete Hodgson's *Feature Toggles (aka Feature Flags)* (martinfowler.com, 2017) is the canonical reference. The four categories he distinguishes — confusing them is the most common feature-flag mistake:

| Category | Purpose | Longevity | Dynamism |
|---|---|---|---|
| **Release** | Decouple deploy from launch; enable trunk-based dev | Short (days–weeks) | Static within a release |
| **Experiment** | A/B / multivariate testing | Medium (until significance) | Per-cohort, dynamic |
| **Ops** | Operator control during incidents (kill switches, degrade-mode) | Mostly short; kill switches may persist | Highly dynamic, must change without redeploy |
| **Permissioning** | Gate features by user tier (premium, beta, internal) | Long-lived (months–years) | Per-user, dynamic |

Each category has different evaluation, lifecycle, and management requirements. A release toggle that survives long enough to drift into a permissioning toggle without acknowledgment is the single most common flag failure.

**Flag debt.** Hodgson explicitly frames flags as "inventory which comes with a carrying cost." The discipline:
- Each new flag has an explicit **owner** and **expiration date**.
- A time-bomb test fails the build when a release flag outlives its expiration.
- Flag removal is on a backlog, not on "we'll get to it."

Common antipatterns: flags that mutate state at evaluation time (side-effecting reads); flags whose default is the dangerous value; tangled inter-flag dependencies; permanent if/else trees from never-cleaned-up release flags; no monitoring of flag-on vs. flag-off behavior.

Many flag systems exist (LaunchDarkly, Split, Unleash, Flagsmith, Statsig, GrowthBook, AWS AppConfig, Optimizely). The skill takes no position on which to use; the discipline is the same regardless.

## Rollback discipline

Every deploy must be reversible — by forward-fix or rollback — within minutes. This is foundational: Humble & Farley ch. 10, Google SRE Book ch. 8 ("Release Engineering"), and DORA's recovery-time metric.

**The hard case is data.** Code is reversible by redeploy; data, once destroyed or transformed, usually is not. A `DROP COLUMN` cannot simply be rolled back. The canonical solution is **parallel change**, also called **expand-contract**: first documented as a refactoring strategy by Joshua Kerievsky in 2006 and later named/detailed by Danilo Sato on martinfowler.com in 2014.

1. **Expand.** Add the new form (column, table, API field) alongside the old.
2. **Migrate.** Dual-write to both; backfill historical data; switch reads to the new form.
3. **Contract.** Once nothing reads or writes the old form, drop it.

Each step is independently deployable and individually reversible. Only the final contract step is forward-only — and at that point the old form has been dead for long enough that going back has no value.

**Forward-only vs. roll-back as defaults.** Stateless code: roll back. Faster, lower cognitive load under incident pressure. One-way changes (contract step, certain external integrations, v2-only protocols): forward-only, planned in advance. Design migrations so each step is reversible until the very last.

**Rollback drills.** From standard SRE practice: rolling back is a muscle that atrophies. Practice rollback during game-days, not only during incidents. A rollback path tested only when prod is on fire is not a rollback path — it is wishful thinking with a button.

## CI/CD pipeline discipline

**Pipeline as code.** The pipeline definition (`.github/workflows/`, `.gitlab-ci.yml`, Jenkinsfile, etc.) lives in the same repo as the code, reviewed in the same PRs. Humble & Farley ch. 5; reinforced by GitOps practice.

**Stage ordering — fail fast, then promote one artifact.** Cheap checks should fail early, but the deployable artifact must be built once and promoted through later stages: checkout → dependency resolution → build/package immutable artifact → unit/static/security scans → integration/acceptance/smoke against that artifact or exact image → release config binding → deploy. The unit-test stage should fail in seconds, not minutes; the common-case bug should fail in minutes, not hours.

**Artifact identity and provenance.** Production deploys use immutable artifact digests, not mutable tags. Artifacts are signed; provenance links artifact to commit and CI run; the deploy system verifies signature/provenance; SBOM and vulnerability scan results attach to the artifact.

**Hermetic and reproducible builds.** A hermetic build depends only on declared inputs, with no leakage from the host environment. A reproducible build produces bit-identical artifacts from the same source, environment, and instructions. They reinforce each other but are distinct properties. Bazel, Buck2, Pants v2, and Nix all push in this direction. See `build-and-dependencies` for depth.

**Trunk-based development.** Paul Hammant's `trunkbaseddevelopment.com` is the canonical reference. Developers integrate to a single branch (`main`/`trunk`) at high frequency; branches, when used, are short-lived (hours to a couple of days); incomplete features hide behind release flags. DORA research repeatedly correlates trunk-based development with strong delivery performance through the small-batch mechanism. Long-lived feature branches are an antipattern. See `version-control-discipline`.

**GitOps.** Coined by Alexis Richardson (Weaveworks, 2017). Declarative system state lives in Git as the single source of truth; reconciliation agents (Flux, Argo CD) continuously enforce desired state on the cluster; all changes happen via commits / pull requests. Most useful for Kubernetes-style declarative platforms; weaker fit outside that context. Honest dissent exists (Steve Smith's *GitOps is a placebo*) — worth knowing the term has limits.

## Release health monitoring

A rollout is not "done" until you've verified the new version is healthy. The integration point is `observability` — every release is evaluated against the SLOs of the service it ships. Burn-rate alerts on the SLO are the natural rollout-decision signal: if budget burn spikes after a deploy, the canary or progressive-delivery system holds or rolls back automatically.

Practical rollout signals: error rate, p99 latency, success rate by route, request rate (drops can mean broken clients), saturation metrics, and *business metrics* (checkout success, signup completion). Choose them before deploy, not during one. A canary that is "watching the dashboard for anything weird" is a rolling deploy with extra steps.

**Bake time** at each canary stage: long enough to detect a regression given the traffic volume; short enough to complete in business-relevant time. There is no universal number. Tune to the traffic needed to detect a regression at the SLO threshold.

**Canary auto-rollback is for high-traffic services.** Below a few hundred requests per minute per stage, the canary will not accumulate enough requests to reach statistical significance against the SLO threshold within a reasonable bake time — auto-rollback decisions become noise-driven, or never fire. For low-traffic services, blue/green with smoke tests, or a simple rolling deploy with a manual gate, is the better fit. Picking canary because it sounds rigorous, on a service that doesn't have the volume to support it, is a common AI-misuse trap.

## Schema and data migrations

Treat schema migrations as release engineering, not as database administration. Each migration is a sequence of independently deployable steps following parallel change.

**Online migration tools** (the discipline matters more than the tool):
- **gh-ost** (GitHub, MySQL): triggerless online schema migration via binlog streaming; pausable, throttlable.
- **pt-online-schema-change** (Percona, MySQL): the older trigger-based equivalent.
- **pg_repack** (PostgreSQL): online table/index rebuilds for bloat removal and reclustering with brief locks; not a general-purpose schema-change framework.
- **Liquibase**: changesets in XML/YAML/JSON/SQL with declared rollback steps and preconditions.
- **Flyway**: SQL-first, sequential versioned migrations. Community Edition is forward-only; Teams/Enterprise editions have supported "undo migrations" since v5.

The Liquibase-vs-Flyway choice is real: Liquibase has first-class declared rollback throughout; Flyway Community treats forward-only as the model. Both are mature; pick by team's complexity and reversibility needs.

**The pattern for any user-facing schema change**: deploy code that reads from both forms; migrate data; deploy code that reads only the new form; drop the old form. Each deploy is small and reversible up to the contract step.

## "You build it, you run it" — and what it actually means

Werner Vogels (Amazon CTO), interviewed by Jim Gray for ACM Queue (June 2006):

> "You build it, you run it. This brings developers into contact with the day-to-day operation of their software. It also brings them into day-to-day contact with the customer. … Giving developers operational responsibilities has greatly enhanced the quality of the services."

The principle is **developer accountability for operations**, not abolition of platform / SRE teams. The often-deployed misuse — "we don't need a platform team because devs run their own services" — is not what Vogels said. The honest framing: developers own the on-call for what they ship; platform engineers own the substrate that makes the on-call survivable. Both, not either.

DORA's findings reinforce this. Centralized release-manager / change-approval-board gates correlate with weaker delivery performance: in the *Accelerate* data they reduce deploy frequency without a corresponding measurable improvement in change failure rate (chapter 7). The combination that wins is small batches, automated tests, comprehensive monitoring, and team ownership of operations.

## Adjacent practices

Two disciplines that surround release engineering and deserve mention:

- **Merge queues** (GitHub Merge Queue GA 2023, Bors, Mergify, Aviator) serialize and rebase merges so `main` is built-and-tested as it actually is, not as it was at PR time. Required at trunk-based-development scale. See `version-control-discipline`.
- **Chaos engineering** (Netflix Chaos Monkey, Gremlin, Litmus, AWS Fault Injection Service) tests resilience to dependency failure, network partitions, and instance termination by deliberately injecting them. The release-engineering connection: a deploy strategy whose rollback path has only been tested in incidents is brittle; chaos drills test it. See `error-handling-and-resilience` for the resilience patterns chaos exercises.

## Common antipatterns

- **Big-bang deploys / release trains.** Many teams' changes batched into one window. Higher change failure rate, higher deployment rework, and longer recovery time per DORA.
- **Long-lived feature branches.** Divergence from trunk grows merge conflicts and integration risk; opposite of trunk-based development.
- **Friday afternoon / pre-weekend deploys** when on-call coverage is thinnest. Folk wisdom rooted in real risk.
- **Unowned, unremoved feature flags** (flag debt).
- **"Hotfix" branches that bypass the pipeline.** The emergency change is exactly the change you most want CI to test.
- **Manual deploy steps that "only one person knows."** Bus-factor-1 release path is a reliability hazard.
- **Staging that doesn't mirror production.** Gives false confidence; original 2005 ThoughtWorks blue/green motivation was exactly this problem.
- **Rollback only tested during incidents.** See above.
- **Deploys that require database downtime.** Fixable with parallel change; not acceptable for user-facing services.
- **"Deploy = push the button at the end of the sprint."** Classic weak-delivery signature.
- **Canary without a defined SLI gate.** High-traffic canaries should hold or auto-rollback on SLI regression. Low-traffic services should use blue/green, smoke tests, synthetic checks, or a staged rolling deploy with an explicit manual gate; don't label that pattern a canary.
- **Feature flag whose default is dangerous.** Defaults should fail safe; release toggles default to off; ops kill switches default to "do the safe thing."

## What to flag in review

- A deploy strategy with no rollback path (especially database changes that aren't expand-contract).
- A high-traffic canary with no defined SLI gate or automated hold/rollback policy; a low-traffic staged rollout mislabeled as a canary.
- A feature flag with no owner, no expiration, or default-on for a new behavior.
- A rolling deploy described as a "canary" because it has stages — no metric gate means it isn't.
- A pipeline that bypasses tests for "hotfix" or "emergency" paths.
- A CI step that pulls dependencies from the public registry inside a runner with production secrets.
- A migration that isn't backwards-compatible with the previous code version (breaks during rollout).
- A "shadow deploy" that double-writes to a real downstream (payment, email, third-party API) without a sandbox.
- A long-lived feature branch (over a few days) without a tracking issue and a merge plan.
- Release-manager / change-approval-board gates on changes that have automated tests and observability — they cost speed without buying safety.

## Reference library

- `references/deployment-strategies.md` — depth on rolling, blue/green, canary, shadow, dark launch, A/B vs. release, progressive delivery; bake-time math; signal-set selection.
- `references/parallel-change-and-migrations.md` — expand-contract in depth, online vs. offline tools, dual-write patterns, contract-step planning.
- `references/dora-and-team-performance.md` — the DORA metrics, what they measure, how to instrument them, and the cultural findings from *Accelerate* and the *State of DevOps* reports.

## Sibling skills

- `version-control-discipline` — small commits, trunk-based development, atomic PRs as the upstream of small-batch deployment.
- `build-and-dependencies` — hermetic builds, lockfiles, supply chain, signed artifacts.
- `observability` — SLI/SLO definition and burn-rate alerts that drive automated rollout/rollback.
- `database-and-data-modeling` — migration patterns from the data-side perspective.
- `error-handling-and-resilience` — circuit breakers, graceful degradation, retries with idempotency at the boundary.
- `debugging-and-incident-response` — rollback under pressure, blameless postmortems that feed back into the deployment pipeline.
- `secure-coding-fundamentals` — Software Supply Chain Failures (OWASP A03:2025), signed artifacts, SBOMs.

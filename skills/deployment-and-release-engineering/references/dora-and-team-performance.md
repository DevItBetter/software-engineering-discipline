# DORA Metrics and Team Performance

Reference for the empirical research underpinning modern release-engineering practice. Built on Forsgren, Humble, and Kim's *Accelerate* (IT Revolution, 2018) and the annual DORA *State of DevOps Report* (2014–present, now run by Google after the 2018 acquisition).

## The four metrics

The DORA program's research validated four software-delivery measures that predict team performance better than any other set:

1. **Deployment Frequency** — how often code reaches production.
2. **Lead Time for Changes** — time from code commit to code running in production.
3. **Change Failure Rate** — fraction of deploys that cause a degradation requiring rollback or hotfix.
4. **Failed Deployment Recovery Time (FDRT)** — time to restore service after a failed deploy. Earlier reports called this Mean Time to Restore (MTTR); DORA renamed it FDRT in 2023 for precision — MTTR is overloaded with general incident-recovery meanings.

The four cluster cleanly into two pairs:
- **Throughput**: Deployment Frequency, Lead Time for Changes — how fast you ship.
- **Stability**: Change Failure Rate, FDRT — how reliably you ship.

The headline finding from *Accelerate*: **the two pairs are correlated, not traded off**. Teams that ship faster also recover faster and break less often. This is counterintuitive to engineers who grew up with "go slow to be safe" — but the empirical pattern is consistent across years of data.

## How to measure them

Practical instrumentation:

- **Deployment Frequency**: count successful deploys per service per time window. CI/CD systems naturally emit this; surface it on a team dashboard. The number is per service, not per organization.
- **Lead Time for Changes**: timestamp of commit (`git commit` date in main) to timestamp of that commit running in production. The hard part is correlating commits to deploys; SHA-tagged artifact builds give you this for free.
- **Change Failure Rate**: count deploys that triggered rollback, hotfix, or a SEV-marked incident, divided by total deploys. Defining "failure" is a team discipline; pick a definition and stick to it.
- **FDRT**: time from a failed deploy's start of impact to recovery. The "start of impact" needs the same SLO that drives your alerting; recovery is when SLO health returns.

A common mistake: instrument these as point-in-time stats rather than time series. The interesting signal is the trend — is the team getting faster and more reliable, or drifting?

## Why the four metrics work

The mechanism behind "fast = stable" is **batch size**:

- Small changes are easier to review (less surface area for the reviewer to load).
- Small changes are easier to debug when they break (one cause, recently committed, by one person).
- Small changes are easier to roll back (single deploy unit, recent state).
- Frequent deploys force the operational fitness — pipelines, monitoring, on-call discipline — that make recovery fast.

Large batches do the opposite: many possible causes when one fails, harder to roll back without dragging unrelated changes, often blocked by integration debt that accumulated during the long branch.

## DORA's tier benchmarks (historical)

For years, DORA's reports binned teams into four tiers (Elite / High / Medium / Low) by performance against the four metrics. The Elite/Low contrast was striking — Elite teams deployed multiple times per day; Low teams deployed less than monthly.

**Status in 2025:** the 2025 *DORA Report: State of AI-Assisted Software Development* retired the four-tier benchmark in favor of seven team archetypes and an AI Capabilities Model. The bands (e.g., "Elite teams deploy on demand, multiple times per day") live on in industry vocabulary and team self-assessment, but DORA itself has moved past them.

Use them as illustrative shorthand, not as a current DORA position. When citing a specific benchmark, cite the report year.

## What separates elite delivery teams from low performers

DORA's research consistently identifies the same bundle of practices in elite delivery teams:

- **Trunk-based development** (`version-control-discipline`). Short-lived branches; integrate to main multiple times per day.
- **Automated testing on every commit.** Unit, integration, and acceptance tests. Fast feedback.
- **Comprehensive monitoring and observability.** SLIs/SLOs/error budgets (`observability`).
- **Small batch sizes.** Smaller changes per deploy; more deploys per change-set.
- **Continuous integration.** Each commit goes through the pipeline; no batched merges from long-lived branches.
- **Loose coupling and team autonomy.** Teams can deploy their service without coordinating across the org.
- **Empowerment and learning.** Engineers can choose tools, learn from failures, attend external conferences.
- **Blameless postmortem culture** (`debugging-and-incident-response`). Failures produce learning, not punishment.

Notably absent from the elite-performer profile: change-approval boards, release managers as gatekeepers, formal change-control processes. DORA's research consistently shows these correlate with **lower** delivery performance without a measurable improvement in stability — the safety theater is worse than no theater.

## The "two pizza" / team size finding

Adjacent to DORA: Amazon's "two-pizza team" rule (Jeff Bezos, popularized via Werner Vogels) — small teams (~6–10 people) that own their service end-to-end deploy faster and more reliably than large teams owning shared services. The mechanism overlaps with batch size and team autonomy.

## How to use the metrics

The metrics are a feedback loop, not a benchmark to score teams against. The right uses:

- **Detect regressions in the team's delivery system.** A drop in deployment frequency, a rise in lead time, a rise in failure rate — these signal accumulating debt in the pipeline, the architecture, or the team's processes.
- **Measure investment.** Did the platform-team investment in test infra actually shorten lead time? The metric tells you.
- **Compare against the team's own past**, not against other teams. Lead time depends heavily on the deployment unit's complexity; a regulated payment service will never match a stateless web service.

The wrong uses:

- **Per-engineer scoring.** The metrics are team-level; pulling them apart by individual is meaningless and corrosive.
- **Cross-team competition.** Different services have different deploy profiles. A monorepo team and a serverless team will produce different shapes.
- **Single-metric optimization.** Optimizing deploy frequency alone, without watching change failure rate, encourages reckless deploys. The metrics work as a quartet.

## Sources

- Forsgren, Humble, Kim. *Accelerate: The Science of Lean Software and DevOps*. IT Revolution, 2018.
- DORA Resources hub. `https://dora.dev/resources/`.
- 2024 *Accelerate State of DevOps Report*. `https://dora.dev/research/2024/dora-report/`.
- 2025 *DORA Report: State of AI-Assisted Software Development*. `https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report`.
- Werner Vogels interview (ACM Queue, 2006) for the team-ownership framing. `https://queue.acm.org/detail.cfm?id=1142065`.

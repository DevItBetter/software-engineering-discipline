# Deployment Strategies in Depth

Reference for choosing and operating deployment strategies. The strategy is a gradient between "deploy and watch" and "fully automate the rollout under telemetry control."

## Rolling deployment

The default. Replace v1 instances with v2 incrementally, respecting orchestrator parameters (`maxSurge`, `maxUnavailable` in Kubernetes; equivalents in other platforms).

**Strengths:** simple, no extra infrastructure, no extra configuration.
**Weaknesses:** no metric gate. The rollout will run to completion even if v2 is broken. Reverting requires another rolling deploy of v1, which is slower than a router flip. Sessions or in-flight requests on terminating pods need graceful shutdown.

A rolling deploy is *not* a canary. Stages without metric gates are not canary stages — they are just a slower rolling deploy.

## Blue/green deployment

Two production environments — blue (live) and green (idle). Deploy v2 to green; smoke-test; flip the load balancer; keep blue around until you're sure.

**Strengths:** rollback is a router flip — seconds, not minutes. Smoke-testing happens against real production environment, on real production capacity, before any user traffic.
**Weaknesses:** doubles infrastructure during cutover (cost). Doesn't help if v2 has a bug that only appears under live traffic patterns the smoke test doesn't exercise. Database changes still must be expand-contract — the database is shared between blue and green.

The original ThoughtWorks blue/green pattern (circa 2005, Daniel Terhorst-North + Jez Humble) was specifically about deploying to the *same* hardware to keep the test environment honest. Modern blue/green tends to use cloud capacity that's spun up only during cutover — different mechanics, same property.

## Canary deployment

Route a small fraction of traffic to v2, watch SLIs, expand or roll back. The metric-watching is the part that distinguishes a canary from a rolling deploy.

**Stage progressions** vary by traffic volume and risk. The progressions below are typical Argo Rollouts / Spinnaker defaults — tune to your traffic, not to the example numbers:
- **Low-risk service, high-volume traffic:** 1% → 5% → 25% → 50% → 100%, with bake time at each stage long enough to accumulate enough requests to detect a regression at your SLO threshold.
- **High-risk service:** 0.1% → 1% → 5% → 25% → 50% → 100%, with longer bake at each stage.
- **Low-volume service:** larger initial percentage (10%+) so you actually see traffic; longer bake times because each stage takes longer to accumulate signal. **Below a few hundred requests/minute, canary auto-rollback often doesn't work** — the bake time required for statistical significance exceeds business-acceptable rollout windows. Prefer blue/green with smoke tests for these.

**The bake-time math.** To detect a regression of size *r* at confidence *c*, you need a sample size *n* of requests that depends on the baseline error rate and the size of the regression. For a 1% increase in error rate over a 0.5% baseline at 95% confidence, you typically need tens of thousands of requests at the canary stage. If your stage takes hours to accumulate that many requests, your bake times must be hours.

**Gating signals.** Pick before deploy:
- Error rate (overall and by route).
- Latency (p50 and p99 separately; p99 catches tail regressions averages hide).
- Success rate by user-meaningful operation (checkout, login, etc.).
- Request rate. A drop can mean broken clients, not just lower load.
- Saturation (CPU, memory, queue depth) as a leading indicator.
- Business metrics (signup completion, payment success). These often catch issues that infrastructure metrics miss.

**Gating policy.** Commonly: hold and alert on degradation; auto-rollback on regression past a threshold. Auto-rollback decisions should be deterministic — humans tune the policy, not the rollback decision itself.

## Shadow / mirroring deployment

v2 receives a copy of production traffic; its responses are discarded. Implemented at the proxy or service mesh: Envoy `request_mirror_policies`, Istio `VirtualService.mirror`, NGINX `mirror`, AWS App Mesh.

**Strengths:** behavior comparison and load test under real traffic without user risk. Useful for major rewrites where unit tests can't cover the surface.
**Weaknesses and traps:**
- **Stateful side effects double real load.** Writes hit the same database; emails get sent twice; payments get charged twice; third-party APIs get called twice. The shadow path *must* route to a sandbox or be stubbed for any side-effecting operation. This is the single most common shadow-deployment mistake.
- **Latency tax.** The mirroring proxy waits for both v1 and v2 in some configurations; verify your mesh's behavior.
- **Doubles backend load.** Plan for the capacity.

When done correctly, shadow is the highest-confidence pre-rollout signal you can buy. When done incorrectly, it is an outage.

## Dark launch

The Facebook 2008 chat-launch pattern: deploy code, execute it in production, but its effects are invisible to users. The chat backend was load-tested by silently generating chat traffic from the live web app long before any chat UI shipped.

**Tight definition** (Fowler's bliki, 2020): the code runs in production; users can't tell.
**Distinct from:** canaries (which expose users to v2), feature flags (which gate user-visible features), shadow (which mirrors traffic). All four are sometimes conflated under "dark launch" in casual usage; tight terminology helps.

**When to use:** new backends with unknown production performance characteristics; large rewrites where you want to exercise the new path before the old path is removed; expensive operations whose load profile you can't simulate accurately in test.

## A/B testing as a release vehicle

Same plumbing as canary; different intent. The success criteria are different:

| | Canary | A/B test |
|---|---|---|
| Goal | Verify v2 doesn't regress | Find which variant is better |
| Window | Short (minutes to hours) | Long (until statistical significance) |
| Decision | "Roll back if regression" | "Pick the winner" |
| Metric | System SLIs | Business metrics |
| Cohorting | Random or low-percentage | Stratified by user property |

Don't conflate. A "canary" that runs for two weeks waiting for statistical significance is an experiment. An "A/B test" that gets rolled back automatically on error rate is a canary.

## Progressive delivery

James Governor (RedMonk, 2018) coined the term as the umbrella for **canary + feature flags + observability + automated rollback** as one practice. Tools that implement it: Argo Rollouts, Flagger, AWS CodeDeploy with linear/canary configurations, Spinnaker.

A progressive-delivery rollout looks like this:

1. Deploy v2 to a small percentage (canary).
2. Behind a release flag, expose v2's new functionality to internal users only.
3. Watch SLOs (and burn rate).
4. If healthy after bake time, expand traffic.
5. If healthy at 100%, expand the flag's audience (internal → beta → 1% of users → ...).
6. At each stage, an automated rule can hold or roll back based on signal.
7. Once the flag is at 100%, schedule it for removal.

When a design doc says "progressive delivery," verify all four pieces are actually present — not just renamed canary stages.

## Choosing a strategy

A rough heuristic, not a rule:

- **Internal tools, low traffic, low risk:** rolling deploy is fine. Skip the operational tax.
- **User-facing service, moderate risk:** canary with automated rollback on SLO burn.
- **High-stakes change (auth, payment, data path):** shadow deploy first to verify behavior; then canary for actual rollout.
- **Major rewrite or new backend:** dark launch first; then canary.
- **Schema or backwards-incompatible change:** parallel change, regardless of strategy.
- **Strict regulatory / instant-rollback requirement:** blue/green with smoke test before flip.
- **Mature CD shop with strong observability:** progressive delivery as the default operating mode.

The decision is rarely "which strategy" alone — it's "which combination, gated by what signals, with what automated policy."

## Sources

- Humble & Farley. *Continuous Delivery* (Addison-Wesley, 2010), ch. 10 (Deploying and Releasing Applications).
- Martin Fowler. *BlueGreenDeployment* bliki. `https://martinfowler.com/bliki/BlueGreenDeployment.html`.
- Danilo Sato. *Canary Release* on martinfowler.com (2014). `https://martinfowler.com/bliki/CanaryRelease.html`.
- Martin Fowler. *DarkLaunching* bliki (2020). `https://martinfowler.com/bliki/DarkLaunching.html`.
- James Governor. *Towards Progressive Delivery* (RedMonk, 2018-08-06). `https://redmonk.com/jgovernor/2018/08/06/towards-progressive-delivery/`.
- Pete Hodgson. *Feature Toggles (aka Feature Flags)* (martinfowler.com, 2017-10-09). `https://martinfowler.com/articles/feature-toggles.html`.
- Argo Rollouts and Flagger documentation for production progressive-delivery implementations.

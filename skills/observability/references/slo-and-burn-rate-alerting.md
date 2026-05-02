# SLOs and Burn-Rate Alerting

Reference for the *Site Reliability Workbook* (Beyer, Jones, Petoff, Murphy; O'Reilly 2018, Chapters 2 and 5) approach to SLOs and SLO-based alerting. The chapter on alerting on SLOs is `https://sre.google/workbook/alerting-on-slos/`.

## Choosing SLIs

An SLI is a quantitative measure of one aspect of service quality. The Workbook standardizes the SLI as a ratio:

```
SLI = good_events / valid_events
```

Choosing an SLI is choosing what counts as "good" and what counts as "valid." That choice is not mechanical — it is the most important judgment in the whole exercise.

### Defaults that hold up

- **Request/response service**
  - Availability SLI: fraction of valid requests served without 5xx. Typically excludes 4xx (client errors) from both numerator and denominator, on the rationale that 4xx is the user's contract violation, not the service's. Discuss exceptions per endpoint (e.g., is `429 Too Many Requests` a service failure during a thundering herd? Often yes).
  - Latency SLI: fraction of valid requests under a threshold. Choose the threshold from user research, not infrastructure capability. Two thresholds are sometimes useful: a "fast" target (most requests) and a "slow" target (long tail).
- **Data pipeline / batch**
  - Freshness SLI: fraction of records processed within target lag.
  - Correctness SLI: fraction of records that pass downstream validation.
  - Coverage SLI: fraction of expected records actually processed.
- **Queue / async work**
  - Throughput SLI: messages processed per unit time relative to ingress rate.
  - Age-of-oldest-unprocessed-item: less a ratio than a watermark; convert by setting an upper bound and counting violations.

### Bad SLIs

- **Pure infrastructure metrics** (CPU, memory) — they don't measure user experience. Saturation is a leading indicator, not a goal.
- **Average latency** — averages mask the long tail that users feel. Use percentiles, or better, a ratio under threshold.
- **Aggregate-over-too-large-a-population** — one SLI for the whole product can make a single-tenant outage invisible. Either pick SLIs per critical journey or weight them.
- **SLIs nobody decides on** — an SLI with no explicit threshold is decoration. The threshold *is* the SLO.

## Choosing the SLO target

The target should come from what users actually need, calibrated against what the system can sustainably deliver. Common mistakes:

- Set 100% as the target. There is no error budget, every blip is a violation, the SLO becomes meaningless.
- Set the target wherever current performance is. The SLO becomes "don't get any worse," which is operational, not aspirational.
- Set per-component SLOs that don't compose into a credible end-to-end SLO. If your service depends on three 99.9% services serially, your achievable SLO is bounded near 99.7% before you add your own faults.

## Error budget

```
error_budget = 1 − SLO
```

For a 99.9% time-based availability SLO over a 28-day window: 28 × 24 × 60 × (1 − 0.999) ≈ 40 minutes of budget. For event-based SLIs, bad events consume error budget; for time-based availability, outage minutes are the convenient equivalent.

### Policy

The mechanism is a release/freeze gate. While budget remains, ship faster — the budget is meant to be spent. When it's exhausted, slow down, freeze risky changes, and invest in reliability work until the budget recovers. The policy makes the product/reliability tradeoff legible to product managers and engineering simultaneously, which is the whole point.

## Burn-rate alerting

A burn rate is how fast you are consuming the error budget relative to the SLO window. A burn rate of 1 means you exhaust the budget exactly at the end of the SLO window. A burn rate of 14.4 means at the current rate you'll exhaust the budget in about 1/14.4 of the SLO window.

The Workbook's canonical multi-window, multi-burn-rate recipe (Table 5-8 of Chapter 5) uses a **30-day SLO window** and specific budget-consumption thresholds:

| Severity | Long window | Short window | Burn rate | Budget consumed |
|---|---|---|---|---|
| Page | 1 hour | 5 min | 14.4 | 2% |
| Page | 6 hours | 30 min | 6 | 5% |
| Ticket | 3 days | 6 hours | 1 | 10% |

Heuristic: short window = 1/12 of long window; both must exceed the threshold to fire.

These specific values are not magic. The 2% / 5% / 10% budget-consumption thresholds and the 14.4 / 6 / 1 burn rates are calibrated for the chosen windows and budget-consumption policy. **Re-derive burn rates when your SLO window or budget-consumption policy changes**. If only the SLO target changes, the burn-rate values can stay the same, but the absolute bad-event threshold changes because the error budget changes. The general formula:

```
budget_consumed_threshold = burn_rate × (long_window / SLO_window)
```

Pick budget thresholds first (how much consumed before paging is reasonable), then derive burn rates for chosen windows. Convert the burn rate to an alert threshold by multiplying it by the service's error budget fraction.

### Why two windows

The short window resets quickly when an incident clears, so the alert clears quickly too — no manual silencing. The long window prevents single-spike false positives. Together they balance precision (low false-positive rate) and recall (don't miss real burns).

### Trade-offs in window choice

- Smaller windows = faster detection, more false positives.
- Larger windows = slower detection, lower false positive rate, harder to act on while the incident is still in flight.
- Faster burns deserve smaller windows (a 14.4× burn is consuming budget so fast you have minutes, not days).

## Common mistakes

- **Alerting on the SLI value crossing the SLO threshold** instead of on the burn rate. A latency spike to 500ms when the SLO is 99% under 200ms is *consuming* budget; whether you've *crossed the SLO threshold for the period* depends on the entire window. Burn-rate alerting catches the spike now.
- **One SLO per service, no per-journey SLOs**. A failure on the checkout path matters more than a failure on the help-center path; an aggregate SLO can make critical-path failures invisible.
- **SLOs nobody enforces**. The SLO is a contract with the team. If breaching it has no consequence, the SLI is decoration.
- **SLOs imported from a vendor template** without adapting to the actual user experience. The vendor doesn't know what your users feel.
- **Forgetting the multi-window pair** and alerting on a single window. Either single-spike noise or slow-burn blindness, depending on the window size.

## Sources

- Beyer, Jones, Petoff, Murphy. *The Site Reliability Workbook* (O'Reilly, 2018). Chapter 2 (SLOs), Chapter 5 (Alerting on SLOs). `https://sre.google/workbook/`.
- Beyer, Jones, Petoff, Murphy. *Site Reliability Engineering* (O'Reilly, 2016). Chapter 4 on SLOs, Chapter 6 on monitoring. `https://sre.google/sre-book/`.
- Rob Ewaschuk. *My Philosophy on Alerting* (Google SRE, ~2014). `https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/preview`.

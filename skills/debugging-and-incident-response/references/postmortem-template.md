# Postmortem Template

A template for writing postmortems that produce learning. Adapt to your organization's conventions — what matters is the content, not the section names.

## Template

```markdown
# Incident Postmortem: [Brief description]

**Date:** [Date of incident]
**Authors:** [Names]
**Status:** Draft / Review / Final
**Summary:** [One paragraph: what happened, when, impact, resolution.]

## Impact

- **Users affected:** [Count or percentage; segments if relevant]
- **Duration:** [Start time → mitigation time → full resolution]
- **Severity:** [Per your severity rubric]
- **Dollar / contractual impact:** [If known]
- **Customer-facing communication:** [What was said publicly]

## Timeline

All times in [timezone].

- **HH:MM** — [What happened. Who did what. What was observed.]
- **HH:MM** — ...
- **HH:MM** — Mitigation deployed.
- **HH:MM** — Full recovery confirmed.

(Build during the incident; refine after. Include both technical events and human decisions. "12:14 — Decision to roll back rather than fix forward, given uncertainty about the bug.")

## Root cause(s) and contributing factors

[Plural is usually more honest. Most incidents have multiple contributing causes.]

**Primary cause:**
[The technical or process condition that, had it not been present, would most likely have prevented the incident.]

**Contributing factors:**
- [Other conditions that made the failure more likely or more severe.]
- [Things that delayed detection or response.]
- [Gaps in tooling, alerting, or process.]

## Detection

- **How did we find out?** [Alert / customer report / passive observation]
- **Time from issue to detection:** [Duration]
- **What worked well in detection?**
- **What didn't?** [If detection was slow, why?]

## Response

- **Who responded?** [Roles, not just names]
- **What was tried?** [In order, with outcomes]
- **What was the mitigation?** [Specifically — rollback, failover, disabled feature, etc.]
- **What worked well in response?**
- **What didn't?**

## What went well

[Yes, including the good. Recognizing what worked is part of learning.
Examples: alerting fired correctly; rollback procedure was fast; a previous postmortem's
action item directly helped here.]

## What went poorly

[Honestly. Without naming individuals.]

## Where we got lucky

[Things that could have been worse but weren't, due to luck rather than design.
Important to surface — luck doesn't always hold.]

## Action items

| # | Description | Owner | Priority | Due |
|---|---|---|---|---|
| 1 | [Specific change. Not "be careful" — a system change.] | [Person] | P0/P1/P2 | [Date] |
| 2 | ... | ... | ... | ... |

[Each action item must be: specific, owned, dated. "We should improve monitoring" is not
an action item. "Add alert on X above Y, owned by Alice, by 2026-05-15" is.]

## Lessons learned

[Generalizable insights that go beyond this specific incident. What did we learn about
how the system fails? What categories of failure should we be looking for elsewhere?]

## Supporting links

- [Incident channel transcript]
- [Relevant dashboards / graphs]
- [PRs related to the fix]
- [Related earlier incidents]
```

## Worked example

```markdown
# Incident Postmortem: Order processing 30-min outage

**Date:** 2026-04-15
**Authors:** Operations team
**Status:** Final
**Summary:** Between 14:32 and 15:04 UTC on 2026-04-15, the order processing
service returned 503 errors for ~80% of requests. Cause was a database connection
pool exhaustion triggered by a slow query introduced in deploy v4.2.1. Mitigated
by rolling back to v4.2.0 and increasing the pool size as a defensive measure.

## Impact

- ~12,000 failed orders during the 32-minute window.
- Estimated revenue impact: $X.
- Status page updated at 14:39 (7 minutes after onset).
- Customer-facing email sent post-incident.

## Timeline (all UTC)

- 14:25 — Deploy v4.2.1 of order-service to production. Includes a new query for the customer history view.
- 14:32 — Error rate begins climbing on order-service. Pages on-call.
- 14:34 — On-call (IC: Alice) acknowledges. Pulls up dashboards.
- 14:36 — Identifies elevated 503s; database connection pool at 100%.
- 14:39 — Status page updated to "investigating."
- 14:42 — Suspect deploy correlates with onset. Decision: roll back.
- 14:48 — Rollback to v4.2.0 begins.
- 14:54 — Rollback complete; error rate begins falling.
- 15:04 — Error rate at baseline. Service confirmed healthy.
- 15:10 — Status page updated to "resolved."
- 15:30 — Initial investigation: new query in v4.2.1 lacks index on (customer_id, created_at).
  Each customer history view triggered a sequential scan; under load, queries piled up,
  exhausting the connection pool.

## Root cause(s) and contributing factors

**Primary cause:** The v4.2.1 query for customer history was missing an index. Under
production load (which was higher than staging), each query took 2-4 seconds instead of <50ms,
saturating the connection pool.

**Contributing factors:**
1. Query was tested on staging with a small dataset (~10k customers); production has 8M.
   Performance issue did not manifest in staging.
2. Code review did not include an EXPLAIN check for the new query.
3. Connection pool size was tight (100 connections per instance × 5 instances = 500 total),
   leaving little headroom for slow queries.
4. No alert on connection pool utilization; only on application errors. By the time alerts
   fired, the pool was already exhausted.

## Detection
- Alert fired 7 minutes after deploy (alert latency includes window for false-positive
  suppression).
- Response was prompt; rollback decision in <10 minutes from alert.

## Response
- IC role assumed by Alice. Operations role by Bob. No CL needed for an incident this size.
- Three mitigations considered: roll forward (fix the index), roll back, or scale up the pool.
  Rolled back as the safest option; pool scaling was applied as defensive measure post-rollback.

## What went well
- Rollback procedure worked smoothly; no surprises.
- Status page updated within 7 minutes; customer communication prompt.
- The "previous deploy correlates with onset" check is now reflexive after the
  2025-Q4 postmortem; saved time here.

## What went poorly
- Query review at PR time did not include EXPLAIN. We had a culture of "ORM handles it."
- Staging dataset is too small to catch performance regressions. Has been a known gap;
  not addressed.
- No alert on connection pool utilization meant we detected via 503s rather than the
  underlying cause.

## Where we got lucky
- The on-call team was available and responsive within minutes. A delayed response
  (e.g., during a holiday) would have extended the outage substantially.

## Action items

| # | Description | Owner | Priority | Due |
|---|---|---|---|---|
| 1 | Add CI check that runs EXPLAIN ANALYZE on new queries against a prod-shape dataset; fail the build if any query does sequential scan on tables >100k rows | Carol | P0 | 2026-05-01 |
| 2 | Add alert on connection pool utilization >80% sustained for 1min | Bob | P0 | 2026-04-22 |
| 3 | Refresh the staging database to a sanitized prod snapshot weekly | Dave | P1 | 2026-05-15 |
| 4 | Update PR template to require EXPLAIN output for new queries | Carol | P1 | 2026-04-25 |
| 5 | Document this incident pattern in the runbook for "elevated 503s" alert | Alice | P2 | 2026-05-08 |

## Lessons learned
- Performance regressions in queries are not caught by our CI; the EXPLAIN-in-CI approach
  closes that gap. Worth applying to *all* queries on hot paths, not just new ones.
- We need observability on the conditions that cause user-visible failures, not just the
  failures themselves. "Pool utilization" is one example; what other "leading indicators"
  do we lack?
- Staging-vs-production data shape mismatch is a category of bug we'll continue to ship
  until we narrow it. Worth a deeper look.

## Supporting links
- [Incident channel: #incident-2026-04-15]
- [Deploy v4.2.1 PR]
- [The slow query]
- [Connection pool dashboard during incident]
```

## Common postmortem failure modes

**Action items that don't fix anything.** "Be more careful." "Improve communication." "Remember to test." Not action items; not changes to the system. Reject these and replace with concrete system changes.

**Stopping at the first cause.** Five-whys gets you started, but real incidents have multiple contributing causes. Document them all; pick action items that address the most leveraged.

**Blame leaking in.** Phrasing matters. "Alice deployed without checking" → "Our deploy process didn't include a check that would have caught this." Same fact; different focus.

**Postmortem as performance.** A postmortem written to make leadership happy ("we found the root cause; here are the action items; we are confident this won't happen again") rather than to learn is wasted. The value is in the honesty.

**Postmortem written and forgotten.** Action items shipped; postmortem filed; the lessons live nowhere. Build a culture that revisits old postmortems, learns the patterns, and uses them to teach.

**No follow-up on action items.** Action items go in tickets; tickets get deprioritized; the gap that caused the incident is still there a year later. Tracking action-item completion is part of the postmortem process; report on completion rates.

## When to do a postmortem

- **All customer-facing outages.** Always.
- **All security incidents.** Always.
- **Data loss or corruption.** Always.
- **Near misses worth learning from.** When the same combination of conditions could realistically produce a real incident.
- **Surprising or novel failures**, even if customer-impact was small.

Don't postmortem every minor blip. The signal value of postmortems comes from their selectivity. Routine alerts that resolved without intervention probably don't need one.

## Tone

Postmortems should be:

- **Specific** — facts, times, decisions, observations.
- **Blameless** — system focus, not individual focus.
- **Honest** — including what we got lucky on, what we don't fully understand, what we didn't do well.
- **Actionable** — every problem identified produces a concrete action item or an explicit "we are accepting this risk because X."
- **Boring** — not dramatic narrative. Postmortems aren't meant to be exciting reading.

The goal is learning. Anything that gets in the way of honest learning gets cut.

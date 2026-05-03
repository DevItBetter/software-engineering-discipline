# Incident Roles and Runbooks

The operational practices that turn an incident from chaos into a coordinated response. Drawn from Google SRE / IMAG (Incident Management at Google), which itself derives from the US Incident Command System used for emergency response.

## Why roles matter

When an incident is small and one person can handle it, formal roles are overhead. When an incident is big — multiple things broken, multiple stakeholders, multiple operators working in parallel — the absence of formal roles produces:

- Duplicate work (two people both rolling back).
- Missed work (everyone assumed someone else was updating the status page).
- Confusion about who decides what (do we roll back? failover? wait?).
- Stakeholder frustration ("nobody's telling me what's happening").

Roles solve these. They're not bureaucracy; they're division of labor.

## The three core roles

### Incident Commander (IC)

**Owns the response.** Makes the high-level decisions: do we roll back, failover, accept the impact, escalate. Holds the picture of what's broken, what's been tried, what's coming.

The IC does NOT do the technical work. That frees them to:
- Track what's happening across multiple workstreams.
- Decide priorities ("the data corruption is more urgent than the latency").
- Bring in additional people when needed.
- Make handoff decisions (when to bring in another role, when to escalate to leadership).
- Decide when the incident is over.

The IC is usually the most senior on-call person, but not always. Sometimes the senior person is the operator and a less-senior IC keeps the coordination clean.

### Communications Lead (CL)

**Owns external and stakeholder communication.** This includes:
- Updating the status page.
- Drafting customer notifications.
- Updating leadership on a cadence.
- Answering "what's going on" from stakeholders so the IC and operators don't have to.
- Building the incident timeline as it happens.

Critical for any incident with broad impact. A senior IC who's also writing customer emails is divided attention; a separate CL keeps the IC focused.

### Operations Lead (OL)

**Owns the technical mitigation.** The person actually executing — running the rollback, doing the failover, restarting services. May lead a team of operators on a big incident.

The OL communicates up to the IC ("rollback is at step 3 of 5; ETA 8 minutes") and gets direction down ("focus on the order service first; the analytics is degraded but not critical").

## When to formalize roles

A useful threshold: **if the incident is going to last more than 15-30 minutes, or has more than two people working it, formalize.**

For a 5-minute incident that one person resolves: don't bother. Adding ceremony to a quick fix wastes the very time you're trying to save.

For an hour-long, multi-person incident: not having roles is the bigger waste.

The IC role specifically: as soon as a second person joins, declare an IC. "I'm IC, you're operating, ok?" — even informally. Keeps coordination clean.

## The handoff

Long incidents span shifts. The handoff is where things go wrong.

Google's protocol:
1. Outgoing IC briefs the incoming IC: status, what's been tried, what's planned, who's doing what.
2. Outgoing IC explicitly says: "You are now the incident commander, ok?"
3. Incoming IC explicitly acknowledges: "Yes, I am incident commander."
4. Outgoing IC stays available briefly for follow-up questions but is no longer making decisions.

The explicit verbal handoff is non-negotiable for non-trivial incidents. Without it, both people may think the other is in charge — and decisions get duplicated or missed.

## The incident channel

A dedicated channel (Slack, Teams, dedicated chat) for the incident. Conventions:

- **Named with date and brief description.** `#inc-2026-04-15-order-outage`.
- **All operators and stakeholders join.** No side conversations.
- **Decisions documented.** "Rolling back to v4.2.0 — comments?" — gives others a chance to object before action.
- **Status updates threaded.** Major events as top-level posts; investigation chatter in threads.
- **The IC posts a summary at the top, kept updated.** New joiners read it instead of asking.
- **No memes, no jokes, no emoji-storming.** A serious channel for serious work. (Levity has its place; in the channel during an outage isn't it.)

## Communication cadences

External (customers, status page):

- **Initial post within 5-15 minutes** of incident detection. Even "investigating, no ETA" is far better than silence.
- **Updates every 15-30 minutes** until resolution. Same content if no change ("still investigating; will update by [time]").
- **Resolution post** with a brief summary. Detailed postmortem follows separately.

Internal (leadership, neighboring teams):

- **Initial notification** when severity is clear.
- **Updates on a cadence appropriate to severity** — every 15 minutes for major incidents, hourly for smaller.
- **Hand-off notification** when shifts change.

Some teams have a separate stakeholder channel where the CL posts updates, while the operators work in a denser channel. Reduces noise for stakeholders; keeps operator focus.

## Severity rubric

A simple rubric helps everyone agree on what an incident is. A common shape:

- **SEV1 (critical):** total outage, security breach, data loss in progress, regulatory exposure. Page everyone. Communicate to customers immediately. CEO awareness. Continuous response until resolved.
- **SEV2 (major):** significant degradation, partial outage, single-region failure with HA in place. Page on-call. Status page updated. Time-sensitive but not emergency.
- **SEV3 (minor):** degraded but functional. Errors above baseline but most users unaffected. Pages on-call but not emergency. Investigate during business hours if possible.
- **SEV4 (negligible):** known issue, low impact, no customer escalation. Track but don't mobilize.

Not every team needs four levels; pick what works. The point is: when someone says "this is a SEV2," everyone knows what that means.

## Runbooks

A runbook is documented operational knowledge for a specific kind of work.

### What goes in a runbook

For each alert:

- **What it means.** The metric, the threshold, what triggers it. "Why did I get woken up?"
- **Probable causes.** What it usually turns out to be (rank-ordered if there's a pattern).
- **Initial diagnosis.** Where to look. Specific dashboards (with links). Specific log queries.
- **Mitigations.** Step-by-step procedures, in order of severity:
  - First-resort: light intervention (clear the cache, restart one instance).
  - If that doesn't work: heavier intervention (failover, scale up, rollback).
  - Last resort: declare incident, escalate.
- **Escalation.** Who to call if the runbook doesn't help. By name where possible; by role where not.
- **Related runbooks.** Often a problem in service A is a symptom of an issue in service B.

For each common operational task (deploy, rollback, schema migration, secret rotation):

- **When to use this procedure.**
- **Prerequisites.** What must be true before starting.
- **Step-by-step procedure.** Specific commands, specific UIs.
- **How to verify each step succeeded.**
- **How to roll back this procedure.**
- **What to do if it fails halfway.**

### Runbook quality

- **Tested.** Someone followed the runbook recently and it worked. Untested runbooks rot.
- **Dated.** When was it last reviewed? After every relevant incident, update.
- **Specific.** "Restart the service" → "Run `kubectl rollout restart deployment/order-service -n <namespace>` after the incident commander approves restart; then verify with `kubectl rollout status deployment/order-service -n <namespace>`."
- **Authoritative.** When the runbook and the on-call's intuition disagree, the runbook wins by default. (If the on-call is right, update the runbook.)

### Game days / disaster drills

The honest test of runbooks is using them under stress. Game days are scheduled exercises where the team responds to an injected fault:

- "DB failover starting now."
- "Region us-east-1 is down; route traffic to backup."
- "Suspected security breach; rotate all credentials."

The team follows the runbooks. Gaps are obvious. Update runbooks; rerun.

Frequency: quarterly for major scenarios; lighter exercises monthly. New on-call engineers get supervised runs through several runbooks as part of onboarding.

## On-call rotation hygiene

- **Primary and backup.** Primary handles initial response; backup picks up if primary unreachable.
- **Reasonable rotation length.** A week of primary on-call is typical. Extend only with team consent.
- **Compensation or comp time.** On-call has cost.
- **Handoff documentation.** End of shift: open issues, follow-ups, known fragility, what to watch.
- **Onboarding.** New on-call engineers shadow before going primary; supervised first shift; access to all needed tools.
- **Burnout monitoring.** Rotation surveys; track page rate per shift; if a shift is consistently rough, fix the alerts.

## Don't make heroes

A specific failure mode: one person is "the on-call who always saves the day." They handle the gnarly stuff personally; their pager is always on; they take pride in being indispensable.

This is bad for the system and bad for the person.

- The system is fragile because it depends on one person's mental cache.
- The person is on a path to burnout.
- The team can't grow because they don't get reps on hard incidents.
- When the hero leaves (and they will), the team is suddenly worse.

Healthy practice: distribute knowledge. Rotate the hard work. Pair junior on-call with senior. Document what the hero knows. Build the hero into the runbook.

## Specific patterns to flag in review

For incident response and on-call practice:

- An incident that lasted >30 minutes without a declared IC.
- An incident with no timeline being built.
- A "fix" deployed during an incident with no rollback plan.
- A status page that wasn't updated within 15 minutes of incident detection.
- An on-call rotation with no backup.
- A new alert without a runbook.
- A new alert that pages but isn't actionable.
- A runbook last updated >6 months ago for a service with significant churn.
- A team that hasn't run a game day in the last quarter.
- A team where one person handles all the hard incidents.

For postmortems, see `postmortem-template.md`.

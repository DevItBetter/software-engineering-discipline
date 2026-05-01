---
name: debugging-and-incident-response
description: "Find bugs systematically, run incidents calmly, write postmortems that actually produce learning. Use this skill whenever the task involves diagnosing a bug (especially a hard one — intermittent, distributed, \"it worked yesterday\"), responding to a production incident, deciding how to investigate a slow or wrong system, writing a postmortem, designing on-call practice, or asking \"where do I even start.\" Debugging is a skill distinct from coding; most engineers learn it by accident over years. Built on Andreas Zeller (Why Programs Fail), David Agans (Debugging: 9 Indispensable Rules), Brian Kernighan, the Google SRE workbook, John Allspaw / Etsy on blameless postmortems, and Sidney Dekker on Safety II."
---

# Debugging and Incident Response

Debugging is the discipline of finding out *why* a system isn't doing what you think it should. It is distinct from coding, frequently underrated, and the skill that separates engineers who can ship reliable software from engineers who can ship code.

Incident response is debugging with a deadline, a stakeholder list, and a system on fire. Same fundamentals; different operating mode.

This skill covers both: the systematic methods that find bugs, the on-call practices that run incidents calmly, and the postmortem discipline that turns failures into learning.

## Two operating modes

**Debugging mode (no production fire).** Time is on your side. You can take a careful, scientific approach: form hypotheses, run experiments, instrument carefully, dig until you understand. The right answer often involves understanding the *root* cause, not just stopping the symptom.

**Incident mode (production is broken now).** Time is the constraint. The right answer is usually **stop the bleeding first, understand later.** Failover. Rollback. Disable the broken feature. Restart. Then once the system is stable, do the careful debugging.

Switching modes wrong is expensive: spending an hour root-causing while the site is down, or hastily "fixing" without understanding and shipping a worse bug. Recognize which mode you're in.

## Debugging mode — the scientific method

The right framing (Zeller, *Why Programs Fail*): debugging is the application of the scientific method to a malfunctioning program.

1. **Observe** the failure precisely. What input? What output? What was expected? What changed recently?
2. **Form a hypothesis** about the cause that is consistent with all observations.
3. **Make a prediction** the hypothesis implies — "if X is the cause, then doing Y should produce Z."
4. **Test the prediction** by running an experiment.
5. **Refine.** Hypothesis confirmed or falsified; iterate.

The discipline distinguishes good debugging from "trying things." Trying things eventually finds something that "fixes" the symptom and ships a guess. The scientific method finds the actual cause.

### Specific moves that operationalize this

**Start with what you know.** Write it down. The literal failure: input, output, error message, stack trace, time. The literal expectation. The literal context: what was deployed, what changed, who else has reported this.

**Don't trust your model of the system.** Verify it. The bug is probably in the gap between your mental model and reality. Check the version of the code that's actually running. Check the actual database state. Check the actual config. The number of bugs that are "I thought we were running the new version" is enormous.

**Reproduce the bug.** A bug you can reproduce reliably is mostly fixed. A bug you can't reproduce is much harder. Spend disproportionate effort on reproduction:
- Capture the exact input.
- Capture the exact environment.
- Reduce to the minimum case (delta-debug — see below).
- Get from "happens sometimes" to "happens every time on this input."

**Bisect.** Almost every "what changed" question is answerable with bisection.
- Code: `git bisect` between a known-good and known-bad commit.
- Config: find the smallest config diff that triggers it.
- Data: find the smallest input that triggers it (delta-debugging).
- Time: find the time the behavior started.

Bisection is logarithmic; even a thousand-commit gap is found in ~10 steps. The investment is steep (you need to be able to run the test) but the payoff is dramatic.

**Delta-debug your inputs.** When the failing input is large, automatically simplify it. Halve the input, retest. If still fails, recurse on the half. If passes, the failure is in the other half. Repeat until you can't reduce further. Tools exist (Hypothesis's `shrink`, AFL, custom scripts), or do it manually.

**Don't theorize without data.** Agans's third rule: "Quit thinking and look." When you're stuck, the answer is almost always more data, not more thinking. Add logging, run a profiler, attach a debugger, dump the state. The bug is rarely subtle once you can see what the system actually does.

**Change one thing at a time.** When you make a fix, change exactly one thing. If the bug goes away, you know that was the cause. If you change five things and it goes away, you don't know which one mattered, and the other four are now noise in the codebase.

**Keep an audit trail.** Notes as you go: what hypotheses, what experiments, what results. When you come back to the bug after a meeting, you can pick up where you left off. When you write the postmortem, the timeline is already there.

**Check the simple things first.** Agans's "Check the plug." Is the service actually running? Is the right version deployed? Is the network actually reachable? Did the cert actually rotate? The number of "complex bugs" that turn out to be a typo in a config file is humbling.

## Reading stack traces and logs

Most bugs in production are diagnosed from stack traces and logs, not from a debugger. The skill is reading them carefully.

### Stack traces

- **Read top to bottom and bottom to top.** The top is where the exception was raised; the bottom is what called it. The bug is somewhere in between.
- **The first line of "your code" matters most.** Library frames are usually fine; the bug is at the boundary where your code calls them.
- **The exception type tells you the class of failure.** `NullPointerException` is "I expected a value but got null." `IndexOutOfBoundsException` is "I expected the collection to be longer." `ConnectionRefused` is "the network said no."
- **Caused-by chains** show wrapped exceptions. The original cause is usually what you want to investigate.
- **Async / threaded stacks** can be misleading; the work happened on a different thread; the cause may not be in the immediate stack. Look for "executing task" or "completing future" frames as a hint.

### Logs

- **Sort by request ID, trace ID, or session ID.** A single request's full story across services is what you need.
- **The first error in the chain is usually the cause.** Subsequent errors are often consequences (retries, fallbacks, timeouts).
- **Look at successful requests too.** A "broken" request often differs from a successful one in one specific field; identifying the difference often reveals the bug.
- **Look at what happened *before* the failure.** Five minutes before the incident: any deploys? config changes? traffic patterns? upstream incidents?
- **Empty logs are a finding.** "We have no logs from this code path" is itself information.

If your logs aren't structured (JSON or similar), structured logging is one of the highest-leverage investments. The ability to filter, group, and aggregate logs is the difference between minutes and hours of investigation.

## David Agans's nine rules (one-liner reference)

A mental checklist when you're stuck:

1. **Understand the system.** Read the manual. Know what the system is supposed to do.
2. **Make it fail.** Reproduce. If it's intermittent, find what makes it more or less frequent.
3. **Quit thinking and look.** Get data. Add logging. Use a debugger. Don't speculate when you can observe.
4. **Divide and conquer.** Bisect. Narrow the search.
5. **Change one thing at a time.** And only that one thing.
6. **Keep an audit trail.** Notes, notes, notes.
7. **Check the plug.** The simple things. Is it on? Is the right version running?
8. **Get a fresh view.** Talk to someone. Step away. Sleep. Rubber-duck.
9. **If you didn't fix it, it ain't fixed.** When the bug "stops happening," verify you understand why. A bug that stopped on its own often comes back.

The rules are worth knowing not because they're profound but because they're easy to forget when you're frustrated and behind schedule.

## Specific debugging techniques

### Rubber duck

Explaining the problem out loud to a rubber duck (or a colleague) often surfaces the answer. The act of structuring the explanation — what's the input, what's expected, what's actually happening — exposes assumptions you didn't realize you were making.

Most "unblock me" requests in chat get answered by the asker before anyone responds. The duck is doing the work.

### Differential diagnosis

Two cases, only one fails. What's different? Lock onto the difference.

- Two requests, only one returned 500. Diff them.
- Two environments, only prod has the bug. Diff the configs.
- Two test runs, only one is flaky. Diff what ran around it.

### Wolf-fence (binary search in code)

Add logging at the midpoint of the suspect code path. Run. The bug is on one side. Add logging at the midpoint of that side. Repeat.

Crude but effective when the codebase is unfamiliar and the failure is hard to reproduce in isolation.

### Tracer bullets

Add a known-unique value to the input; trace where it appears in logs / outputs / database. Reveals the path the data actually took.

### Watching real traffic

For bugs that don't reproduce in test environments, sometimes the right move is to instrument production carefully (with sampling, with care for PII) and watch real failures as they happen. Distributed tracing makes this much easier.

### Coredump / state-snapshot inspection

When the bug crashes the process, sometimes a memory dump is the only artifact. Inspecting it (`gdb`, `lldb`, `dlv`, language-specific tools) reveals state at the moment of failure.

### "Heisenbugs" — the bug disappears when you debug it

Some bugs are sensitive to timing, ordering, or memory layout, and adding logging changes the system enough to make the bug stop. Recognize this; specific tactics:

- **Investigate non-invasively.** Production traces (OpenTelemetry, distributed tracing), eBPF probes, perf counters. They observe without (much) perturbation.
- **Use sanitizers.** `ThreadSanitizer` finds data races at runtime in C/C++/Go/Rust. `AddressSanitizer` finds memory errors. Run the test under sanitizer; the bug often surfaces immediately, with a precise diagnosis, in a way bare execution doesn't show.
- **Disable optimizations.** A "release build only" bug is often a missing memory barrier, an undefined-behavior reliance on optimization, or a race the optimizer's reordering exposed. Build with `-O0` and rerun; if it goes away, that confirms the class.
- **Add atomic counters at suspect points.** Tracks how often a code path is hit, without the side effects of full logging. Reveals race windows.
- **Stress-test concurrently.** Run the suspect operation thousands of times, in many threads, on a busy machine. Heisenbugs that appear "occasionally" often appear "every time" under stress.
- **Capture-and-replay.** Tools like `rr` (Linux) record execution deterministically; replay shows the exact sequence that produced the bug. Heavyweight but for hard concurrency bugs, lifesaving.

Above all: don't conclude "intermittent, not a bug" because you couldn't reproduce. Intermittent is the signature of the worst bugs; they recur in production at the worst moment.

## Incident response — running an incident

When production is on fire, the goals are different:

1. **Restore service.** Stop the bleeding.
2. **Coordinate the response.** No work is duplicated; nothing is missed.
3. **Communicate.** Stakeholders need updates.
4. **Preserve evidence.** When the dust settles, there will be a postmortem; you need data.
5. **Learn.** Eventually.

Notice what's *not* on this list: "find the root cause." That comes later. During the incident, root-causing is in service of restoration, not the goal.

### Roles (Google SRE / IMAG framing)

For non-trivial incidents, formalize roles:

- **Incident Commander (IC).** Coordinates the response. Holds the high-level state. Decides priorities. Does not do the technical work themselves; that frees them to manage. The IC also handles delegation and ensures nothing falls through the cracks.
- **Communications Lead (CL).** Updates stakeholders. Drafts customer notifications. Holds the status page. Frees the IC and operators to focus.
- **Operations Lead (OL).** Runs the technical mitigation — the person actually doing the failover, the rollback, the disabling. May have multiple operators on a big incident.

For small incidents, one person can play all three roles. Don't overformalize a 10-minute outage. For an outage approaching an hour or more, the formal structure pays off — preventing miscoordination is its own discipline.

### The handoff

For incidents that span shifts, hand off explicitly. Google's protocol: the outgoing IC briefs the incoming IC, then explicitly says "you are now the incident commander, ok?" and waits for firm acknowledgment. No ambiguity about who is in charge.

### Stop the bleeding

Common patterns:

- **Roll back.** If a deploy correlates with the incident, roll back. Prefer rollback over fix-forward when you don't yet understand the bug.
- **Failover.** To a healthy region, replica, or instance.
- **Disable the feature.** Feature flag off. Better an unavailable feature than a broken site.
- **Throttle.** Reduce load until the system recovers.
- **Restart.** For memory leaks, lock leaks, stuck processes — sometimes the fast fix.
- **Increase capacity.** When the failure is overload, scale up.

Each is a tradeoff. Document the tradeoff in the incident channel: "Rolling back the order service to the previous version. We lose 2 hours of bug fixes, but the new version's bug rate is 100x normal."

### Communication discipline

- **An incident channel** (Slack, dedicated chat). Everyone working the incident is here. No side conversations.
- **A status update cadence.** Every 15-30 minutes, even if it's "no change yet." Stakeholders need to know you're alive and working.
- **A status page.** External users need to know there's a problem. Don't make them refresh hopefully.
- **A timeline being built as you go.** Someone (often the CL) writes down what happened when. Will be the foundation of the postmortem.

The most common communication failure: silence. The team is heads-down working; stakeholders assume the worst; pressure escalates. A "still investigating, no ETA, will update in 15" is much better than nothing.

### Don't make it worse

In incident mode the temptation is to act fast. The risk is acting wrong. Some discipline:

- **One change at a time.** If three engineers each change a config, you don't know which fixed (or worsened) it.
- **Communicate before you act.** "I'm going to disable the feature flag in 30 seconds." Gives others a chance to object.
- **Use the documented procedure** when one exists. Improvising under stress is when most second incidents start.
- **Don't rebuild while burning.** Don't decide to "fix the architecture" mid-incident. Bandaid now; redesign later.

## Postmortems — the discipline of learning from failure

The point of a postmortem is **not** to assign blame. The point is to identify what about the system (technical or organizational) made the failure possible, and to change something so the same kind of failure is harder next time.

### Blameless culture (Allspaw / Etsy)

The blameless postmortem framing:

> "If we have an engineering culture where we punish people for making mistakes, we'll have a culture where people lie about mistakes. If we have a culture where we treat mistakes as opportunities to learn about how the system actually works, we'll have a culture where the system gets better."

Specifically: assume everyone involved made the best decision they could with the information they had at the time. The failure is a property of the system, not of the person. The fix is in the system.

This is hard. It requires leadership commitment (engineers who get blamed will hide things; the loop closes). It also requires discipline in the postmortem itself — phrasing matters.

**Phrasing that's blameless:**
- "The deploy went out without rollback verification." (System fact.)
- "The on-call runbook didn't cover this scenario." (System gap.)
- "We had no alert on this metric." (System missing.)

**Phrasing that's blame-y, even when accurate:**
- "Alice deployed without checking the rollback." (Names a person.)
- "On-call should have known to check X." (Implies negligence.)
- "We need to be more careful." (Personal failing rather than system fix.)

### What goes in a postmortem

A useful template:

- **Summary** — one paragraph. What happened, when, how big the impact, how it was resolved.
- **Impact** — affected users, duration, dollar impact if relevant.
- **Timeline** — what happened when, in chronological order, with timestamps. Built during the incident.
- **Root cause** — what was the underlying condition that made this possible? (Plural is often more honest — most incidents have multiple contributing causes.)
- **What went well** — yes, including the good. Recognizing what worked is part of learning.
- **What went poorly** — honestly.
- **Action items** — concrete, owned, with deadlines. Each one fixes a specific contributing cause.
- **Lessons learned** — generalizable insights, separate from action items.

### Action items that actually fix things

The most common postmortem failure: action items like "be more careful" or "remember to check X." These don't change the system. The postmortem produces no learning.

Good action items:

- Change a default. ("New services default to readiness probes that include downstream checks.")
- Add automation. ("CI now blocks deploy if the migration is missing a rollback plan.")
- Improve observability. ("Alert on cache miss rate above X.")
- Improve a runbook. ("Runbook for service Y now covers the scenario we hit.")
- Remove a sharp edge. ("API no longer accepts the input pattern that caused this.")
- Change a process. ("Deploys to production require explicit approval after Tuesday at 5pm.")

Each should be **specific, owned, and dated.** If "we should improve monitoring" appears as an action item without owner or deadline, it's a wish, not a commitment.

### The five whys, honestly

The five-whys technique (originated by Sakichi Toyoda; popularized within the Toyota Production System by Taiichi Ohno) repeatedly asks "why" to dig past symptoms.

It has known limitations:
- **Stops at the investigator's knowledge.** Five whys can't find causes the investigator doesn't already know.
- **Tends to a single linear chain.** Real incidents have multiple contributing causes; five whys often picks one.
- **Different investigators get different answers.** The same incident produces different chains.
- **Easy to stop too early.** "We didn't have monitoring" → "we didn't have time to add it" → "we were too busy" → "stop here." That's not the cause.

Use five whys as a starting prompt, not as a complete method. Real causal analysis examines multiple branches, surfaces the systemic conditions, and looks for the contributing factors that interacted.

### Safety II — beyond root cause

Sidney Dekker's "New View" / Safety II framing: don't just look for what went wrong. Look at how the system normally goes *right*, and what made this case different. Often the answer is "the same conditions that produce normal success produced this failure under a slightly different combination" — meaning there's no specific bug to fix; the system has tolerable risk that occasionally manifests.

Practically: ask not only "why did this fail?" but "given how often this could fail, why doesn't it fail more often?" The answer reveals the implicit safety mechanisms — and tells you what would happen if any of those mechanisms eroded.

### The postmortem meeting

A meeting after the writeup:

- **Anyone affected can attend** — not just the on-call team.
- **Walk the timeline.** Surfaces "I had information that would have helped" that wasn't captured.
- **Discuss action items.** Validate they'll fix the right thing. Assign owners.
- **Decide what gets shared more broadly.** Some incidents teach the whole org.

The meeting isn't a tribunal. It's a structured learning session.

### Documenting near misses too

A failure mode that almost happened but didn't — caught at deploy review, prevented by an alert that was ignored, narrowly missed because of timing — is also valuable. The same conditions caused it; the only thing different next time may be luck. Treat near misses with the same discipline (lighter writeup is fine).

## On-call discipline

On-call is the operational reality of running production. Done well, it's a feedback loop that keeps the system healthy. Done badly, it burns out engineers and produces resentment that drives talent out.

### Sustainable on-call

- **Bounded toil.** On-call is responding to alerts; if the alert rate is high enough that on-call can't get other work done, that's a sign of unhealthy alerts (see below).
- **Reasonable shift length.** A week of primary on-call is typical. Longer is exhausting; shorter has handoff overhead.
- **Compensation or comp time.** On-call has cost; recognize it.
- **A backup.** Primary on-call can be unreachable for a moment; secondary picks up.
- **A handoff ritual.** End of shift: list of open issues, follow-ups, anything the next shift should know.

### Alert hygiene

Alerts are the input to on-call. Bad alerts produce burnt-out on-call.

The goals:

- **Every alert is actionable.** "CPU is high" is not (so what? do what?). "Error rate is above SLO and burn rate is X" is.
- **Every alert has a runbook.** Even one paragraph is fine; "see this dashboard, try this command, escalate if not better in 10 min."
- **No alert fatigue.** If on-call gets 10 alerts a shift and 9 are non-actionable, they'll miss the real one. Page only on what's worth waking someone for.
- **Alerts on symptoms, not causes.** "Users seeing 5xx" is what matters. "CPU is high" is incidental — sometimes high CPU is fine, sometimes user impact happens with low CPU.
- **SLO-based alerting.** Alert when error budget is burning fast enough to matter, not on every dip. Multi-burn-rate alerts (Google SRE) are the canonical pattern.
- **Alert ownership.** Every alert has an owning team. The on-call engineer is empowered to silence a noisy alert immediately; the owning team is on the hook to fix it (improve threshold, fix the underlying issue) by end of next business day. Without this loop, alert hygiene erodes — noisy alerts accumulate, signal degrades.

### What gets paged vs ticketed

- **Page** for: customer impact in progress, data loss in progress, security incidents, anything that needs a human now.
- **Ticket** for: degraded conditions that need attention but not at 3am. Capacity warnings. Long-tail issues. Things that will become problems in days.

Pages are expensive — a person's sleep, a person's dinner, a person's focus. Defend that cost; don't page for the wrong things.

### Runbooks

For every alert, a runbook entry that includes:

- **What this alert means.** The metric, the threshold, what conditions trigger it.
- **Initial diagnosis.** Where to look first. Which dashboards.
- **Common causes.** What it usually turns out to be.
- **Mitigations.** What to do, in order of severity.
- **Escalation.** Who to call if the runbook doesn't help.

Runbooks rot. Date them. Review periodically. After every incident that revealed a runbook gap, update.

## What to flag in review

For code review, design review, or post-incident review:

- A new alert without a runbook.
- A new alert that's not symptom-based or SLO-based.
- A change to a critical path with no observability story (no log, metric, or trace at the new failure points).
- A new background job with no monitoring of its lag, success rate, or last-run.
- A postmortem with action items that say "be more careful" or "we should remember to."
- An incident response that didn't establish IC / CL / OL roles when the incident was non-trivial.
- A "fix" PR for a bug that doesn't include a regression test.
- A "fix" PR that addresses the symptom (added a try/except) without identifying the underlying cause.
- An on-call rotation with one primary and no backup.
- A runbook that's older than the system it documents and obviously wrong.

## Reference library

- `references/postmortem-template.md` — usable postmortem template with example sections.
- `references/incident-roles-and-runbooks.md` — IC / CL / OL responsibilities, runbook structure, drill exercises.

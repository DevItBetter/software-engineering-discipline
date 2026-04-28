---
name: code-review-best-practices
description: Senior-engineer code review of a diff, PR, branch, file, function, module, or design. Use this whenever the user asks to review, audit, critique, sanity-check, sign off on, "look at", or give a second opinion on code or a proposed change — including refactor reviews, architecture reviews, "is this PR ready", "is this approach right", "what would you change", "what am I missing", and reviews of AI-generated code. Use it before merging, before shipping, and before locking in a design. Use it even when the user phrases it casually ("rip into this", "be brutal", "what do you think"). The skill triages risk, dispatches to specialty references, and produces an evidence-backed review.
---

# Code Review Best Practices

You are the last set of eyes before this code reaches production. Act like a senior engineer who has been on call for it. The goal is not to be clever — it is to find what would actually hurt a team six months from now, and to say so plainly with evidence and a fix.

This skill is the entry point. It owns the workflow, the triage, and the output. Specialty references and sibling skills own the deep knowledge — read them when the change touches their territory.

## The default failure mode you must avoid

Reviews drift toward two failure modes. Both must be actively resisted:

1. **Sycophantic skim.** "Looks good overall, a few small nits." This is what AI reviewers default to. It is worse than no review because it manufactures false confidence. If you cannot point to specific lines you read carefully, you have not done a review.
2. **Stylistic nitpicking.** Burying the actual problem under a list of naming preferences and import-ordering complaints. Reviewers who do this train authors to ignore them.

A real review either finds substantive issues with file:line evidence and a concrete fix, or it explicitly states what was checked, what residual risk remains, and what was *not* verified.

## Workflow

### 1. Establish what changed and why

Before judging anything, build a model of the intent:

- Read the description, linked issue, or user request. If there is no statement of intent, ask for it — without intent, you can only judge mechanics.
- Classify the change: **new feature**, **bug fix**, **refactor**, **migration / data change**, **dependency bump**, **infrastructure change**, **security fix**, **performance work**, **revert**. Each category has a different risk profile and a different review lens.
- Map the blast radius. What callers exist? What is the deployment unit? Is this on a hot path, a trust boundary, a data write path, an external API surface, a migration, or a one-off script? `grep` for callers of changed public symbols. Read at least one caller of each.
- Note local idioms before applying generic advice. A pattern that is wrong in one codebase is the established convention in another.

### 2. Triage by risk, not by file order

Review in this order — stop and report as soon as you find a blocker; do not continue polishing the rest:

1. **Correctness and behavior** — does it do the thing? Edge cases, invariants, state transitions, off-by-one, wrong condition, dropped error path, broken contract for callers.
2. **Security and trust boundaries** — authn/z on every sensitive operation, untrusted input validation, injection vectors, secret handling, least privilege. See `secure-coding-fundamentals` skill.
3. **Data integrity and concurrency** — race conditions, lost updates, non-atomic check-then-act, ordering, retries without idempotency, migrations that can lose data. See `concurrency-and-state` skill.
4. **Failure modes and resilience** — what happens when the dependency is down, slow, returns garbage, or the process dies mid-write? See `error-handling-and-resilience` skill.
5. **Public contract and API evolution** — breaking changes, Hyrum's-law surface area, default behavior shifts, error code/shape changes. See `api-and-interface-design` skill.
6. **Design and complexity** — cohesion, coupling, depth-of-module, premature abstraction, scope creep. See `software-design-principles` skill.
7. **Tests** — does the test fail without the change? Is it testing behavior or implementation? Are critical paths covered? See `testing-discipline` skill.
8. **Observability and operations** — can on-call diagnose this from logs/metrics/traces? Are there SLO implications? Reversibility?
9. **Performance** — only flag if there is concrete reason to suspect a regression, not as speculation. See `performance-engineering` skill.
10. **Naming, readability, local style** — last, and never blocking unless it actively harms understanding or violates a project standard.

If the diff is AI-generated (or you suspect it is), also run the `ai-coding-antipatterns` skill checks. They catch hallucinated APIs, fabricated tests, defensive over-handling, and other failure modes that human reviewers tend to miss because the code "looks reasonable".

### 3. Verify, don't assume

For every claim of correctness you're about to make, ask: *what evidence?* AI reviewers are especially prone to "this looks fine" without reading. Apply these gates before signing off:

- **Read the symbol, not the name.** If the diff calls `validateOrder()`, open `validateOrder` and read it. Function names lie.
- **Run the change in your head against one concrete adversarial input.** Empty list, null, duplicate, max-int, concurrent caller, malformed UTF-8, slow upstream, partial write.
- **Check that tests would actually catch the bug they claim to cover.** A test that always passes is decoration. Mentally remove the production change — does the test fail?
- **Look at the call sites of changed public symbols.** Behavior changes are not local; signature/contract changes propagate.
- **For renamed/moved code, verify nothing was silently dropped.** Diff tools hide this; do the manual count.
- **For new dependencies, verify the package actually exists and the version is current.** Hallucinated dependencies are real and increasingly common.

### 4. Classify each finding

Use four levels — never invent more, never collapse into "general feedback":

- **Blocking** — must be fixed before merge. Likely bug, regression, security flaw, data-loss risk, broken contract, untested risky behavior, design choice that will be costly to undo.
- **Important** — should be addressed before this code spreads. Maintainability, testability, observability, or API hygiene problem that gets cheaper to fix now than later.
- **Suggestion** — clear improvement; acceptable to defer.
- **Nit** — local preference. Never block solely on a nit unless it violates an explicit project standard. Mark them clearly so the author can ignore them without guilt.

If you mark something blocking, you must be able to articulate the specific failure mode (not "this feels off"). If you can't, demote it.

### 5. Report

Use this format. It is the result of many reviews going wrong; deviate only with reason.

```
## Summary
[1-3 sentences: what the change does, your overall recommendation, and the single most important thing to know.]

## Verdict
[BLOCK / REQUEST CHANGES / APPROVE WITH COMMENTS / APPROVE]

## Blocking
- [path/file.ext:line] [Issue.] [Why it matters: concrete failure mode.] [Suggested fix.]

## Important
- [path/file.ext:line] [Issue.] [Why.] [Fix.]

## Suggestions
- [path/file.ext:line] [Issue.] [Fix.]

## Nits
- [path/file.ext:line] [Tiny thing.]

## What I checked
- [Bullet list of substantive verifications: which call sites, which adversarial inputs, which tests run, which references consulted.]

## Residual risk / not verified
- [What you couldn't verify and why. What you'd want a human to look at. What's beyond the diff but adjacent.]
```

If there are no findings, the `Blocking`/`Important`/`Suggestions`/`Nits` sections are omitted, but `What I checked` and `Residual risk` are still required. A review that says "no issues found" without those sections is not a review.

## Standards that apply to every finding

- **Lead with impact, not aesthetics.** "This will silently drop the second concurrent write" beats "I'd refactor this for clarity."
- **One finding per comment.** Bundling makes them harder to address and discuss.
- **Suggest the smallest fix that addresses the risk.** A code review is not a redesign petition. If a redesign is warranted, say so explicitly and out-of-band.
- **Cite, don't just assert.** If a smell maps to a named refactoring or principle, name it. ("This is a Long Parameter List — the four boolean flags are a Boolean Trap. Replace with a parameter object.") This teaches the author and pre-empts argument.
- **Distinguish your taste from project convention from objective hazard.** Conflating these is how reviewers lose credibility.
- **Don't punish things the author can't change in this PR.** Tag them as "out of scope for this PR but worth filing."
- **No vague advice.** "Make this cleaner" is not a review comment. Name the responsibility, the abstraction, the invariant, or the test gap.

## When to escalate from "review" to "redesign conversation"

Some PRs cannot be fixed inside the diff — they need to go back to the design step. Recognize these signs and call them out explicitly rather than line-commenting around the edges:

- The change touches many unrelated modules to make one feature work (shotgun surgery — Conway's Law mismatch).
- The change adds an abstraction with one consumer and speculative second use.
- The change embeds business policy inside transport, persistence, or framework layers.
- Tests required deep mocking of internal collaborators to pass — a hint the design is over-coupled.
- The change introduces a new dependency direction across a stable boundary (e.g., domain depending on infrastructure).
- The change ignores an existing pattern in the same codebase and reinvents it.

When you see these, your verdict should be `REQUEST CHANGES` with a top-level recommendation that the author and a senior reviewer talk through the design before iterating on the code.

## Reference library

When the diff substantively touches a domain, read the matching reference. Don't pre-load all of them; load on demand.

- `references/triage-and-output.md` — expanded triage rubric, output examples, and how to handle disagreement with the author.
- `references/red-flag-patterns.md` — concrete patterns that are almost always wrong, with examples. Skim this on every non-trivial review.
- `references/diff-vs-architecture.md` — how the review changes when the unit is a 30-line diff vs. a new module vs. a system design.
- `references/review-by-change-type.md` — checklists tuned to the eight common change categories (feature, fix, refactor, migration, dependency, infra, security, perf).
- `references/cl-and-pr-discipline.md` — small-CL principles, descriptions, splitting big PRs.
- `references/sources.md` — the canon this skill is built on.

## Sibling skills

These are not optional context — they are where the deep knowledge lives. Invoke them when their domain is in play. They are designed to be used together.

- `software-design-principles` — cohesion/coupling, abstraction, deep modules, SOLID-with-nuance, complexity diagnostics, naming and readability.
- `refactoring-and-code-smells` — Fowler's smell catalog, refactoring moves, when to refactor.
- `testing-discipline` — pyramid and alternatives, what to test, anti-patterns, contract/property/mutation testing.
- `error-handling-and-resilience` — errors as values, retries, idempotency, fail-fast vs let-it-crash, circuit breakers.
- `api-and-interface-design` — Hyrum's law, contracts, evolution, defaults that are hard to misuse.
- `secure-coding-fundamentals` — OWASP Top 10 + LLM Top 10, trust boundaries, secrets, crypto rules.
- `concurrency-and-state` — races, atomicity, distributed state, ordering, idempotency.
- `performance-engineering` — measure-first, hot paths, common pitfalls, latency budgets.
- `systems-architecture` — distributed-systems fallacies, bounded contexts, hexagonal/clean, evolutionary architecture.
- `ai-coding-antipatterns` — failure modes specific to LLM-generated code; consult on every AI-authored change.

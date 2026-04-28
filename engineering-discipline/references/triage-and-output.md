# Triage and Output

## How to choose a severity

The severity of a finding is determined by **what would happen if it shipped**, not by how confident you are or how clever the catch was.

### Blocking

Use when one of these is true:

- The change introduces a likely bug. Not "could conceivably fail" — a specific input or sequence that produces incorrect behavior. State the input.
- The change breaks an existing contract that callers depend on (signature, semantics, error shape, ordering, idempotency). Hyrum's law applies even if the contract was undocumented.
- The change opens a security hole: missing authorization, untrusted input crossing a trust boundary unvalidated, secret leakage, injection, SSRF, deserialization risk, weakened crypto.
- The change can lose or corrupt data: non-atomic write where atomicity is needed, missing transaction, race that produces lost updates, migration without rollback or with a destructive step.
- The change introduces a regression in a behavior that has tests — and the tests still pass (they don't actually cover what they claim to).
- A "risky" path has no test at all and the path is plausibly hit by a real user. Risky = anything in the list above.
- A design choice is locked in by this PR that will be expensive to undo (e.g., new public API, new persisted schema, new external integration). The choice deserves an explicit conversation before it ships.

### Important

Use when there is a real future cost but the change is correct today:

- A pattern that will become a maintenance trap (god object forming, coupling tightening, shared mutable state added).
- Tests assert implementation details rather than behavior — they will break on the next refactor.
- Observability gap on a path that will eventually fail (no log, no metric, no error context).
- API ergonomics that invite misuse (boolean trap, primitive obsession, easy-to-mix-up parameter order with same types).
- Documentation/comment is wrong or absent for a non-obvious invariant.
- Performance hazard with concrete reasoning (N+1 query in a loop over user-controlled input, unbounded memory growth).

### Suggestion

Use when there's a clearer or simpler way and the cost-benefit favors the change, but the current code is acceptable. Always include the alternative.

### Nit

Use sparingly. Local style, naming preference, comment phrasing. Mark explicitly: `nit:` prefix, or under a `## Nits` heading. Authors should be free to ignore.

## Output format examples

### A clean PR with one substantive issue

```
## Summary
Adds a webhook retry endpoint. Logic is correct but the retry is non-idempotent — a duplicate call from the upstream provider will create two outbound emails. Block on that; the rest is fine.

## Verdict
REQUEST CHANGES

## Blocking
- src/webhooks/retry.py:42 — `send_email()` is called inside the retry handler with no idempotency key. The webhook provider documents at-least-once delivery, so a retry of this endpoint will send the email twice. Add an idempotency key (the webhook event id is already in the request) and check it against `processed_events` before calling `send_email`.

## What I checked
- Read all three call sites of `send_email`. The other two are inside transactions with deduplication; this one is not.
- Confirmed the webhook provider's at-least-once contract by looking at the existing handler in src/webhooks/order.py:18 which already deduplicates.
- Ran the new test mentally without the production change — it would still pass, because the test does not exercise duplicate delivery.

## Residual risk / not verified
- I did not check whether `processed_events` has appropriate retention; if entries age out faster than the provider's retry window, deduplication will silently fail. Worth a follow-up.
```

### A PR that needs to go back to design

```
## Summary
This PR adds a new `OrderEnricher` interface, a default implementation, a factory, and dependency injection wiring — all to support one concrete enrichment that is called in exactly one place. The abstraction has no second consumer and no immediate plan for one. I would push back and have a design conversation before merging.

## Verdict
REQUEST CHANGES

## Blocking (design)
- src/orders/enrichment/* — Speculative generality. The new interface, factory, and DI wiring add four files and ~200 lines of indirection to support a single 30-line enrichment that could live as a function next to the order builder. Per Ousterhout's "deep modules" framing, this is a shallow interface (more surface area than functionality hidden). If a second enrichment lands, the right time to extract is then. Recommend deleting the interface/factory/DI wiring and inlining as a function.

## What I checked
- Searched the repo for other implementers/consumers of `OrderEnricher` — none exist.
- Read the linked design doc; the second enrichment use case is hypothetical, not on the roadmap.
- Confirmed the function-form would compose cleanly with the existing OrderBuilder.

## Residual risk / not verified
- If there is an off-repo or future plan I'm not aware of, this judgment changes. Worth confirming with the team before deciding.
```

### A PR that's actually fine

```
## Summary
Migration that adds a nullable column. Backfill is run separately. Reversible. No findings.

## Verdict
APPROVE

## What I checked
- Read the migration; column is nullable with no default, no constraint, no index. Safe under concurrent writes.
- Confirmed the backfill is in a separate PR and runs as a batched background job.
- Verified no application code reads the new column yet.
- Searched for ORM model changes that would imply non-null assumptions — none.

## Residual risk / not verified
- I did not benchmark the migration on a prod-sized dataset. For a nullable add this is almost always fine, but worth a glance at the staging migration time before rollout.
```

## Handling disagreement

When the author pushes back, the burden of evidence is on whoever is making the strongest claim:

- If you wrote "blocking" and the author says "it's fine," you must produce the input or sequence that breaks. If you can't, demote to "important" or withdraw.
- If the author says "this is the existing pattern in this codebase," verify by looking. If they're right, withdraw the comment.
- If the disagreement is about taste or style, the author wins by default. You are not the style police unless the project has a documented standard.
- If the disagreement is about a tradeoff under uncertainty, escalate to a third reviewer or to a design discussion. Don't dig in.

The goal is not to win the review. The goal is for the right code to ship.

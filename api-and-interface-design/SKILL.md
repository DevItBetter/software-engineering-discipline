---
name: api-and-interface-design
description: Design APIs and interfaces that are hard to misuse, easy to evolve, and honest about their contracts. Use this skill whenever the task involves designing or reviewing an API (HTTP, RPC, GraphQL, library/SDK, internal interface), evaluating contract changes for breaking-change risk, choosing between sync and async, designing pagination / errors / versioning / idempotency, judging whether an API is "ergonomic", thinking about backward compatibility, deciding default values, naming endpoints/resources, designing an SDK, or asking "is this a good API." Also use when reviewing changes to existing public APIs — Hyrum's Law applies the moment a change ships.
---

# API and Interface Design

An API is a contract. Once it ships, you cannot change it freely; with enough users, every observable behavior becomes a contract whether you wrote it down or not (Hyrum's Law). Designing APIs well is a higher-leverage activity than designing implementation: a bad implementation can be replaced; a bad API drags every consumer along forever.

This skill covers public APIs (HTTP, gRPC, GraphQL, library SDKs) and internal interfaces (functions, classes, modules). The principles are mostly the same. The stakes scale with the audience.

## Hyrum's Law — the inescapable principle

> "With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be depended on by somebody." — Hyrum Wright

If your API ships, every observable behavior is a contract. Not just the documented ones:

- The order in which fields appear in JSON responses (somebody is parsing positionally).
- The exact error message text (somebody is regex-matching it).
- The performance characteristics ("usually returns in 50ms" becomes "must return in 50ms").
- The presence of fields you didn't document but happened to include.
- The fact that an empty list is returned as `[]` rather than omitted.
- The microsecond-level timestamps in logs.

The implications:

1. **Make as little observable as possible.** Hide implementation details aggressively. The fewer behaviors you expose, the fewer become accidental contracts.
2. **What you do expose, document and stabilize.** Make it explicit so callers know they can rely on it.
3. **Treat any change to observable behavior as a breaking change** until proven otherwise. Even "minor" tweaks have broken caller code in famous ways.
4. **Add fields freely; don't remove or change them.** Additive change is usually safe; subtractive or modifying change is usually breaking.

This is the single most important framing for evaluating API changes. Before merging, ask: **what observable behavior is this changing? Who depends on the current behavior? How would they know to update?**

## The properties of a good API

A good API is:

- **Hard to misuse.** The most common usage pattern is the one the API guides you toward. Wrong usage is hard to express or visibly wrong.
- **Honest.** Names match behavior. The signature reflects what's actually possible.
- **Cohesive.** Related operations cluster naturally; unrelated operations don't share a namespace.
- **Symmetric.** What you can do, you can undo. What you can construct, you can deconstruct. What you can list, you can paginate.
- **Predictable.** Similar operations behave similarly. Naming conventions are consistent.
- **Self-documenting at the call site.** A reader can tell what a call does from the call alone, without opening the docs.
- **Evolvable.** Designed so future versions can add capability without breaking existing callers.

A bad API is the opposite — easy to misuse, dishonest about what it does, scattered, asymmetric, surprising, opaque at call sites, locked in.

## Specific design moves

### Make illegal states unrepresentable

(Term from Yaron Minsky.) If a state is invalid, design the type so the state has no representation.

- A `User` that's only sometimes authenticated → two types: `AuthenticatedUser`, `AnonymousUser`. Functions that require authentication take the former.
- A response that's "either an error or a value" → a discriminated union (`Result<T, E>`), not two nullable fields where one must be set.
- A configuration that has mutually exclusive options → a sum type, not a struct with `option_a_enabled` and `option_b_enabled` booleans.
- A query that's "either by ID or by email" → two functions, not one with two nullable parameters.

When the type forbids the bad state, you don't need runtime checks for it. The bug is a compile error.

### Avoid boolean traps

```typescript
// Bad
exportData(includeArchived: boolean, includeMetadata: boolean, asJson: boolean)
// Bad call site
exportData(true, false, true)  // What does each argument mean?

// Better
exportData({
  includeArchived: true,
  includeMetadata: false,
  format: "json",
})
```

Boolean parameters with no name at the call site are easy to mix up (especially when refactoring or copy-pasting). Two boolean parameters create four possible combinations; six create 64. The combinatorial space is rarely all meaningful.

When booleans are unavoidable, use named arguments (where supported) or split into separate functions named for the behavior.

### Avoid primitive obsession at the boundary

A function that takes `(string, string, string, int)` is a misuse-magnet. Calls in the wrong order type-check but produce wrong behavior.

`createUser(email: EmailAddress, password: PasswordHash, role: Role, age: Age)` — now the type system catches accidental swaps.

Domain types at the boundary are worth the boilerplate. Inside the implementation, primitives are often fine.

### Default to the safe behavior

Defaults matter because most callers use them. They should be:

- **Safe under uncertainty.** The default should be the behavior that's correct when the caller hasn't thought about it.
- **The most common need.** If 90% of callers want `recursive=False`, that's the default.
- **Unsurprising.** No "powerful default" that does more than the caller expects (`git push` originally pushed all matching branches; this surprised users and was eventually changed).
- **Safe in production.** A default that opens a port, drops permissions, or sends data should be off-by-default.

The git CLI, the rsync flags, the AWS CLI — all have famous default-related foot-guns. Each was a small design decision with large blast radius.

### Honest signatures

The signature should tell the truth about what the function does:

- A function annotated to return `User` shouldn't sometimes return `null`. Use `User | None` (or `Optional<User>`) and force callers to handle absence.
- A function named `getUser` shouldn't have side effects. If it logs or mutates, that's surprising.
- A function that may throw should document which exceptions (or use a typed `Result`).
- A function that takes a callback should document when the callback runs (synchronously? on which thread? how many times? on error or success?).

Lies in the signature are worse than no signature. Callers trust the signature; when it lies, they get bugs they can't predict.

### Pagination, filtering, sorting — one consistent model

For list endpoints, pick one model and use it everywhere:

- **Offset/limit pagination** is simple but breaks under concurrent inserts (a new row inserted at position 5 between page 1 and page 2 causes the caller to either skip or duplicate items). OK for small, mostly-stable lists.
- **Cursor-based pagination** is stable under inserts because the cursor encodes a position in the ordering (e.g., "id > last_seen_id" or an opaque token over the sort key) rather than a numeric offset. New inserts shift offsets but don't change the cursor's relative position. Use **opaque** cursors (the client can't decode them) so you can change the underlying ordering or storage without breaking callers.
- **Time-window pagination** for time-series data. Caller asks for "events between t1 and t2."

Pick one. Don't have endpoint A use offset, endpoint B use cursors, endpoint C use page numbers. Consistency is part of the API.

For filtering and sorting: define a small explicit vocabulary. "Filter by these specific fields, with these specific operators." Avoid the temptation to expose a full query DSL — it locks the implementation to support every combination forever.

### Errors as part of the contract

Error responses are contract. Design them as deliberately as success responses.

- **Use the right status code (HTTP) or error type.** A 400 for client error; 404 for not found; 422 for valid request that can't be processed; 5xx for server error. Don't return 200 with `{"success": false}` — every caller has to special-case.
- **Provide a stable error code.** `error_code: "invalid_email_format"` is parseable. Localized messages are not.
- **Include a human-readable message** for logs and operator debugging. Don't show it to end users without translation.
- **Include enough context to diagnose** (request id, the offending field).
- **Don't leak internals.** No stack traces, no SQL, no internal URLs.
- **Document each error type.** Callers handle errors they know about; undocumented errors become "unexpected error" handling.

### Idempotency

Any operation a client might retry needs an idempotency model:

- Naturally idempotent (e.g., `PUT user/123` with full state) — no extra design needed.
- Server-supplied idempotency key (server returns the result; client can re-request).
- Client-supplied idempotency key (`Idempotency-Key` header; server stores `(key, result)` for some retention).

Document the model so callers can write retries safely.

### Versioning and evolution

Pick a versioning strategy and commit:

- **No versioning, additive only** (Stripe-style with date-based version pinning). Works if you have strict additive discipline.
- **Major version in URL** (`/v1/`, `/v2/`). Clear; locks you into supporting v1 indefinitely if you have customers there.
- **Header-based version negotiation.** Flexible; harder to debug.
- **Graphql/protobuf field deprecation.** Mark fields deprecated; remove them in the next major.

Whatever you pick, make it possible to add capability without breaking callers:

- **New endpoints can be added freely.** Old callers don't see them.
- **New optional parameters can be added.** Existing callers omit them.
- **New fields in responses can be added** if callers don't break on unknown fields. (This requires their parsing to be tolerant — JSON usually is; some protobuf serializers are strict.)
- **Adding new error types is breaking** if callers do exhaustive error matching. Document new errors as "may be added in the future."

What is not safely addable:
- Required parameters.
- Required fields in request bodies.
- Strengthened preconditions.
- Removed fields from responses (some caller depends on them — Hyrum).
- Changed types for existing fields.
- Changed semantics for existing fields.
- Removed endpoints.

### Async vs sync

A sync API blocks until the work is done. An async API returns a handle (a job ID, a URL to poll, a webhook callback) and the work happens later.

Sync is simpler for callers. Use it unless:

- The work is reliably long (more than a few seconds).
- The caller doesn't need the result immediately.
- The work is heavy and you need to control concurrency.
- The work is fire-and-forget (analytics events, log shipping).

For async APIs, design the polling/notification model carefully. `GET /jobs/{id}` returns status. Webhook fires on completion. Both have cost.

A common bad pattern: a sync API that *might* take 30 seconds. Callers eventually retry; you process the same request twice. Make it async with idempotency.

## Library / SDK design

Same principles, with extra weight on:

- **Imports are the public surface.** Only export what you intend to support.
- **Names matter more.** Library names appear in caller code permanently; renaming is a major version bump.
- **Semver discipline is non-negotiable.** Breaking changes get major versions. Period. (`semver.org` defines the rules; follow them.)
- **Document every public function.** Inputs, outputs, errors, side effects, examples.
- **Provide a "getting started" path** that produces a working result in <5 minutes.

A library that's hard to start with, hard to discover the right API, easy to misuse, and changes shape between versions will be replaced by the next library that's better at any of those.

## What to flag in review

- A new public endpoint with no documented authorization requirement.
- A change to an existing endpoint's response shape (adding/changing/removing fields).
- A change to an existing endpoint's error semantics (new error code, changed status code).
- An endpoint that returns different shapes based on input (sometimes a list, sometimes a single object).
- A boolean parameter whose default value flips behavior in a surprising way.
- A new field added in a way that requires a coordinated client update.
- An async operation introduced without idempotency or status-checking.
- A new dependency on an external service with no documented behavior on dependency failure.
- A new SDK function whose name doesn't reflect what it does.
- A breaking change in a non-major version.
- Pagination that's inconsistent with the rest of the API.

## Reference library

- `references/contract-evolution.md` — what's safely addable, what's breaking, semantic versioning honestly.

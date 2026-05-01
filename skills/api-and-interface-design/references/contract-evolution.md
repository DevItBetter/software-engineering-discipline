# Contract Evolution

A working reference of what's a breaking change, what's safe, and how to evolve APIs without forcing a coordinated client update.

## Semantic Versioning, honestly

Semver (semver.org) defines:

- **MAJOR** version: incompatible API changes.
- **MINOR** version: backwards-compatible new functionality.
- **PATCH** version: backwards-compatible bug fixes.

The honest application requires admitting:

- Most "bug fixes" are observable behavior changes. If anyone has come to depend on the broken behavior, fixing it is technically breaking. (Hyrum's Law.)
- "Backwards-compatible" depends on what the consumer assumes. Strict structural typing (some protobuf serializers) treats new fields as breaking; loose parsing (most JSON consumers) treats them as additive.
- "API change" includes runtime behavior, not just signatures.

Practical rule: when in doubt, bump major. Customers are far more annoyed by silent breaking changes than by version bumps.

## What's safe to add (in most consumer environments)

- **New endpoints / functions.** Old callers don't see them.
- **New optional parameters with safe defaults.** Old callers don't pass them.
- **New optional fields in request bodies.** Old senders omit them; server should default sensibly.
- **New fields in response bodies** *if* consumers tolerate unknown fields. (JSON usually fine; strict protobuf parsers reject.)
- **New error codes** *if* documented as "more may be added" and consumers handle unknown codes gracefully.
- **New event types** *if* consumers ignore unknown events.

For each "if" above, the safety is conditional on consumer behavior. Document the contract explicitly: "responses may contain additional fields in the future; consumers should ignore unknown fields."

## What's breaking (in most environments)

Always:

- **Removing an endpoint.**
- **Removing a field from a response.**
- **Changing the type of a field** (string→int, scalar→array, single object→list).
- **Renaming a field.**
- **Changing the meaning of an existing field** (e.g., "amount in cents" → "amount in dollars").
- **Adding a required parameter.**
- **Strengthening a precondition** (rejecting input that was previously accepted).
- **Weakening a postcondition** (returning less than previously promised).
- **Changing the success status code** (200 → 201).
- **Changing the default value of a parameter** in a way that flips behavior.
- **Changing pagination semantics** (offset model → cursor model).
- **Changing error semantics** (was "user not found" returns 404, now returns 200 with an error object).

Often (depends on consumer):

- **Adding a new field in a response** (strict consumers reject).
- **Changing field order in a response** (positional consumers depend on it).
- **Adding a new error code** (exhaustive consumers don't handle it).
- **Changing the timing of a callback** (was sync, now async).

## The deprecation cycle

For changes that need to be made but can't be made instantly:

1. **Announce.** Document the upcoming change. Set a date. Communicate with customers.
2. **Add the new path.** Both old and new are available. New code uses the new path.
3. **Deprecate the old.** Mark the old path with a deprecation notice (warning logs, response headers, documentation).
4. **Migrate consumers.** Help the largest users migrate first. Provide tools where possible (a migration script, a compatibility shim).
5. **Verify zero usage.** Logs/metrics confirm nobody is calling the old path.
6. **Remove.** Only after measurement says it's safe.

The shortcut "we'll just remove it; nobody uses it" is the source of the most expensive incidents. "Nobody uses it" should be backed by traffic data, not by belief.

## Specific evolution patterns

### Adding a new required field

You can't. By definition, this is a breaking change.

The workarounds:

- Add it as **optional with a sensible default** (best when a default exists).
- Add it as **required only on the new endpoint** (e.g., `/v2/users` requires it; `/v1/users` doesn't).
- Add it as **optional now, required in the next major version** (gives consumers time to adopt).

### Removing a field

- Mark deprecated.
- Wait for one or more major versions (depending on stability commitment).
- Remove in a major version.

A specific failure: removing a field that consumers grep'd for. Adding a `_legacy` suffix is sometimes used to indicate "this field will go away" — works for consumers who notice, doesn't help the rest.

### Changing field semantics

The hardest. Existing consumers will silently misinterpret the new semantics.

- **Add a new field** with the new semantics.
- **Keep the old field** with old semantics for a deprecation period.
- **Document the difference** prominently.
- **Remove the old field** in a major version.

The wrong move: change the semantics in place. "We changed `amount` from cents to dollars" — someone, somewhere is now showing customers prices 100x off.

### Renaming

- Add the new name as an alias.
- Keep the old name working for a deprecation period.
- Encourage migration in documentation.
- Remove the old name in a major version.

### Adding a new error type

Two camps:

- **"Adding error types is breaking"** — exhaustive consumers don't handle it.
- **"Adding error types is non-breaking if the contract said unknown errors are possible"** — consumers should always handle the unknown case.

Be explicit in the documentation about which camp your API is in. Most well-designed APIs are in the second camp; consumers handle a default case.

## Schema evolution (databases)

Special case worth knowing. The same principles apply but with the additional complication that schema changes are coordinated with data changes.

The classic dance for adding a non-null column to a heavily-used table:

1. Add the column as nullable (or with a default).
2. Backfill existing rows.
3. Make it non-nullable (in a separate migration).
4. Code starts requiring it.

The classic dance for renaming a column:

1. Add a new column.
2. Write to both old and new (in code).
3. Backfill the new column.
4. Switch reads to use the new column.
5. Stop writing to the old column.
6. Remove the old column.

Each step is a separate deploy. Skipping steps causes downtime or data loss.

## Backward and forward compatibility

- **Backward compatibility:** new server works with old clients.
- **Forward compatibility:** old server works with new clients (clients sending fields the server doesn't know).

Most APIs care about backward compatibility (don't break existing clients). Forward compatibility matters when clients can be deployed before servers (mobile apps shipped to user devices that you can't force-update).

For forward compatibility: design the protocol so unknown fields are ignored, unknown enum values are handled gracefully, and version-up paths are well-defined.

## How to communicate breaking changes

When a breaking change is unavoidable:

- **Communicate early.** Give consumers time to adopt. Several months for major customers, more for SDKs in the wild.
- **Be specific.** "We're removing the `legacy_id` field on `/users` on 2026-09-01. To migrate, use `id` instead. Here's the comparison..."
- **Provide migration tools** where possible. A script, a shim, a deprecation header that includes the new endpoint.
- **Track adoption.** Watch traffic to the old path; alert when key customers are still on it close to the deadline.
- **Don't surprise.** A breaking change consumers learn about by their pager going off destroys trust.

The trust you build by getting evolution right pays off — consumers tolerate occasional breaking changes if the process is transparent and they have time to adapt. The trust you destroy with one surprise breaking change takes years to rebuild.

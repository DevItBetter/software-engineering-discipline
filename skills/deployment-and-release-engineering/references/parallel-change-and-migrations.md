# Parallel Change and Schema Migrations

Reference for the canonical pattern that lets schema and API changes ship as independently-reversible steps. As Sato credits, the pattern was sketched in Joshua Kerievsky's 2006 documentation of the technique; it was named and detailed by Danilo Sato in *Parallel Change* on `martinfowler.com/bliki/ParallelChange.html` (2014-05-13). Also called **expand-contract**.

## Why this pattern exists

Code is reversible (redeploy the previous artifact). Data is not (a `DROP COLUMN` cannot be undone except from backup). Without a discipline, the only way to change a schema is downtime + big bang + pray. With parallel change, every schema migration is a sequence of small reversible steps, and downtime is not part of the plan.

The same pattern applies to any backwards-incompatible change in a system with live consumers: API field renames, message schema changes, file format migrations, configuration restructures.

## The three phases

### 1. Expand

Add the new form alongside the old. The system now supports both. Nothing depends on the new form yet; nothing has migrated.

For a column rename `email` → `contact_email`:
- Add `contact_email` (nullable for now).
- Triggers, dual-write logic in the application, or change-data-capture keep the new column in sync with the old.

For an API field rename `user_id` → `account_id`:
- The API accepts both names on input; emits both on output.
- Documented as "either is fine; new clients use `account_id`."

This deploy is fully reversible. Roll back the deploy and the old form keeps working as before; the new form is unused.

### 2. Migrate

Backfill historical data into the new form. Move readers from the old form to the new. Move writers from the old form to dual-write to both, then to writing only the new.

The order matters and is the same every time:

1. **Backfill**: existing rows get `contact_email = email` written to them via batched migration job. Run online; throttle to avoid replication lag and disk-IO spikes.
2. **Move readers**: production code reads `contact_email`, falls back to `email` if null, and emits a metric on the fallback rate. Watch the metric drop to zero.
3. **Move writers to dual-write**: every write to `email` also writes `contact_email`. Newly-created rows are populated by application code rather than triggers.
4. **Move writers to new-only**: writes go only to `contact_email`. The old `email` column is now stale.

Each substep is a separate deploy. Each is reversible: roll back to the previous deploy and the system continues to work because the previous step's invariants still hold.

### 3. Contract

Drop the old form. By this point, nothing reads from it and nothing writes to it; it has been dead in the database for long enough to be sure.

- For a column: `DROP COLUMN email`. (For very large tables on MySQL, use gh-ost or pt-online-schema-change to do this without a long lock.)
- For an API field: stop emitting the old name on output; reject the old name on input (or leave it as a no-op forever — preferring to never break existing clients).

The contract step is forward-only — but you have given yourself every opportunity to verify before this point. The number of incidents that come from a careful contract step is small, because everything that could fail has been deployed and run in production for days or weeks.

## Why each step must be independently reversible

A migration that is "five steps but you have to do them all together or roll back the whole thing" is not a migration — it is a big bang in disguise. The point of parallel change is that each individual deploy is small enough to roll back without involving the others.

Test this by asking: "if I rolled back this deploy right now, what breaks?" The correct answer at every step except contract is "nothing." If something breaks, you skipped a step.

## Online migration tools

The pattern is independent of the tool, but for large production tables you need tools that don't acquire long locks:

- **gh-ost** (GitHub, MySQL): triggerless. Reads the binary log and applies changes asynchronously. Pausable; throttle on replica lag. The current default for serious MySQL online schema work.
- **pt-online-schema-change** (Percona, MySQL): trigger-based. Older; still widely used. Triggers add write amplification you must budget for.
- **pg_repack** (PostgreSQL): online table/index rebuilds for bloat removal and reclustering with brief locks; not a general-purpose schema-change framework. For schema changes, prefer native PostgreSQL patterns such as `CONCURRENTLY`, `NOT VALID`, and expand-contract migrations.
- **Vitess and CockroachDB**-style distributed systems handle online schema change at the storage layer; the same parallel-change discipline still applies above.

For the migration framework that owns the *sequence* of migrations:

- **Liquibase**: changesets in XML/YAML/JSON/SQL with declared rollback and preconditions. First-class rollback metadata; expressive; some teams find it heavy.
- **Flyway**: SQL-first, forward-only versioned scripts. Simpler model; rollback is a forward migration that undoes the previous one (because in real production you almost always end up doing that anyway).

Both are mature. Pick by team's appetite for declarative metadata vs. just-SQL.

## Data migrations beyond columns

The same pattern applies to:

- **Type changes** (`VARCHAR` → `JSONB`, integer → bigint). Add the new column with the new type, dual-write, migrate, contract.
- **Table splits** (one table → two for normalization, or many → one for denormalization). Add the new tables, dual-write, migrate readers, contract.
- **Document/event schema migrations** (Avro/Protobuf evolution). Use the format's compatibility rules (BACKWARD/FORWARD/FULL); roll out producers and consumers in the right order.
- **File format migrations** (CSV → Parquet, JSON → MessagePack). Write both formats; migrate readers; stop writing the old format; eventually delete old files.
- **Configuration restructures**. Same shape: support both old and new config; migrate consumers; remove the old.

## Common pitfalls

- **Skipping the dual-write step.** Switching writers from old to new without a period of writing both means a partial rollout cannot recover state without a backfill replay.
- **No fallback metric.** If readers fall back from `contact_email` to `email`, you need to *see* that they're falling back. Without the metric, the migration is theater.
- **Long migration windows during business hours.** Backfill jobs that don't throttle saturate replica lag and IOPS. Run off-peak; throttle.
- **Forgetting orchestration around triggers / CDC.** If a trigger keeps the old form in sync from the new form (or vice versa), the trigger has to be removed at the right point — usually at the contract step. Forgotten triggers leak inconsistencies.
- **Treating the "contract" step as routine.** It is the only forward-only step. Verify the column is unread (audit logs), unwritten (recent INSERT/UPDATE history), and free of foreign-key references before dropping.
- **Migrations that aren't idempotent.** A failed batch of backfill that you rerun should not produce wrong data. `INSERT … ON CONFLICT DO UPDATE` or similar.
- **Big-bang "switch all readers and writers in one deploy."** That is not parallel change. Read this section again.

## When parallel change is overkill

For non-production data (analytics tables, dev environments, non-user-facing internal services with planned downtime windows), the full discipline may be more cost than it's worth. The honest test:

- Does this table back a user-facing read or write path with an SLO?
- Does this table back a path that another team depends on?
- Is there a real cost to brief downtime that doesn't fit your existing maintenance window?

If "no" to all three, a single migration with a tested rollback plan is fine. The parallel-change discipline pays its tax for systems that genuinely cannot tolerate the alternative.

## Sources

- Danilo Sato. *Parallel Change* (martinfowler.com bliki, 2014-05-13). `https://martinfowler.com/bliki/ParallelChange.html`.
- Pete Hodgson. *Expand/Contract: making a breaking change without a big bang* (2023-12-05). `https://blog.thepete.net/blog/2023/12/05/expand/contract-making-a-breaking-change-without-a-big-bang/`.
- Joshua Kerievsky. 2006 documentation of the change-as-sequence-of-small-steps pattern (cited by Sato).
- gh-ost. `https://github.com/github/gh-ost`.
- Liquibase / Flyway documentation.

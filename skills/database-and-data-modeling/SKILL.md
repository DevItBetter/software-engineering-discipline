---
name: database-and-data-modeling
description: "Design schemas, indexes, queries, and data lifecycle that hold up under load and over years. Use this skill whenever the task involves database work — designing or reviewing a schema, choosing between SQL/NoSQL/timeseries/graph, evaluating a query plan, designing indexes, deciding normalization vs denormalization, planning a migration, sizing a connection pool, debating ORM behavior, designing for multi-tenancy, handling schema evolution, or asking \"is this the right database for this.\" Most production incidents are a database problem in disguise; the discipline pays off many times over. Built on Kleppmann (DDIA), Markus Winand (Use the Index Luke), Joe Celko, the PostgreSQL/MySQL official documentation, and the practitioner literature."

---

# Database and Data Modeling

The database is usually the part of your system that's hardest to change, longest-lived, and most expensive when it goes wrong. A function can be rewritten in an afternoon; a bad schema can dominate a team's work for years. Most production incidents that look like "the app is slow" or "we're losing data" are database problems in disguise.

This skill is the discipline of treating data as a first-class design concern. Schema design that doesn't hate you in two years, index design that actually pays off, queries you can read in a query plan, and the operational concerns (connection pools, migrations, replication) that decide whether your database is an asset or a permanent emergency.

## The bedrock decisions

Three decisions, in order, that determine almost everything about how the database serves you:

1. **What's the data model?** What entities exist? What relationships? What invariants must hold? What changes together; what changes independently? This is the work that pays off forever.
2. **What kind of database?** SQL is the right answer most of the time. Sometimes it isn't. The wrong choice here is hard to walk back.
3. **What's the access pattern?** Read-mostly vs. write-mostly. Point lookups vs. range scans. OLTP vs. OLAP. Latency vs. throughput. Indexes, partitions, denormalization, and storage choice all derive from this.

Get these wrong and no amount of clever query optimization will save you. Get them right and the database mostly disappears as a concern.

## Choosing the storage engine

The default reasonable choice in 2026 for most applications is **PostgreSQL**. It's not always right, but if you can't articulate why your case is different, start there. (Reasons: ACID transactions, mature SQL, extensibility, JSONB for semi-structured data, range / partial / expression / GIN / BRIN indexes, mature replication, mature tooling, the largest pool of operators who can run it.)

When the answer is something else:

- **MySQL / MariaDB** — comparable to Postgres for most workloads. Pick based on team familiarity and ecosystem; few decisions hinge on the difference at small scale.
- **SQLite** — single-process or embedded. Excellent for desktop apps, small services, tests, edge deployments. Hugely under-used; goes much further than people expect.
- **A document store (MongoDB, DynamoDB)** — when the data is genuinely document-shaped (varying fields per record), when the access pattern is single-key-lookup, and when you don't need cross-document transactions. Often chosen for the wrong reasons; many apps that "need" a document store actually need JSONB columns in Postgres.
- **A wide-column store (Cassandra, ScyllaDB, BigTable)** — high write throughput, time-series or sparse data, eventual consistency acceptable. Niche; if you're choosing this and you're not sure, you don't need it.
- **A timeseries DB (TimescaleDB, InfluxDB, ClickHouse, Prometheus)** — append-mostly time-stamped data with aggregation queries. Use one of these instead of inventing your own retention policies.
- **A graph database (Neo4j, JanusGraph)** — when the queries are genuinely graph-shaped (multi-hop traversals over edges with properties). Most apps that *think* they need a graph DB don't; relational with recursive CTEs handles a lot more than people assume.
- **An embedded KV store (RocksDB, LevelDB, BoltDB)** — building infrastructure, not applications. Powers many of the above.
- **An OLAP / columnar store (BigQuery, Snowflake, Redshift, ClickHouse, DuckDB)** — analytical queries over large data. Distinct from the OLTP database; usually not your primary store.

The single most common mistake: choosing a NoSQL store because "it scales" without an actual scale problem, then discovering six months later you needed transactions and joins.

## Schema design

### Model the domain, not the screens

The schema is the most stable artifact. It outlives controllers, frameworks, and entire UI rewrites. Design it around the *domain* — the entities and relationships that exist in the world your software is about — not around the current screens.

When the schema reflects screens or APIs, every UI change wants a schema change. When the schema reflects the domain, the same schema serves three different screens and two new APIs over five years.

A useful test: state each table's purpose in one sentence with one noun. "Customer," "Order," "OrderLineItem," "Payment." If you need "and" or "or" — "OrderAndShipment," "UserOrCompany" — the table is two tables.

### Normalize first, denormalize deliberately

Aim for **Third Normal Form (3NF)** as a default for transactional schemas. 3NF is the sweet spot:
- Each fact in one place (no update anomalies).
- Strong data integrity guarantees.
- Without the obscurity of higher normal forms.

Higher normal forms (BCNF, 4NF, 5NF) handle increasingly rare dependency patterns. In 30 years of production software, you'll meaningfully encounter BCNF a few times and 4NF/5NF essentially never. If you find yourself debating them, your schema probably has a different problem.

Denormalize **only with a reason and a plan**:
- A specific query that's too slow at realistic data volume after you've tried indexes.
- A specific cross-table invariant that's hard to maintain.
- A documented batch process that keeps the denormalized data fresh.

The most common honest denormalization: keeping `order.customer_email` even though you have `customer.id` → `customer.email`. The reason: customer email may change but historical orders should keep the email at the time of order. That's not really denormalization; it's modeling immutable history correctly.

### Make illegal states unrepresentable in the schema

Constraints in the database are load-bearing safety. Application code is one of many writers; only the database can enforce invariants for all of them.

- **NOT NULL** on every column that should never be null. The default should be NOT NULL; nullable is the special case requiring justification.
- **CHECK constraints** for value ranges. `CHECK (amount >= 0)`, `CHECK (status IN ('pending', 'paid', 'cancelled'))`.
- **UNIQUE constraints** including multi-column ones for business uniqueness. ("Each user has one row per organization.")
- **FOREIGN KEY constraints** to enforce referential integrity. (Disable them only when you have a specific reason — usually performance at very high scale or async event-sourcing patterns — and document.)
- **EXCLUSION constraints** (Postgres) for non-overlap. ("No two reservations on the same room overlap in time.")

The application can also enforce these; the application *will* miss cases the database catches. Belt and braces.

### Naming and conventions

Pick a convention and follow it. Some defensible defaults:

- Table names: `snake_case`, **plural** (`orders`, not `order`). The plural is consistent with set-oriented thinking — a table is a set of rows.
- Column names: `snake_case`. Domain names, not Hungarian. `created_at`, not `dt_created`.
- Primary key: `id` is fine for surrogate keys. If you want extra safety, `<table>_id` (`order_id`) — when joined, the FK column matches and there's never ambiguity.
- Foreign keys: `<referenced_table_singular>_id`. `customer_id` references `customers.id`.
- Boolean columns: positive phrasing prefixed with `is_`/`has_`. `is_active`, not `not_disabled`.
- Timestamps: `created_at`, `updated_at`, `deleted_at` (for soft deletes), all `TIMESTAMPTZ` (always with timezone).
- Money: `numeric(precision, scale)` with explicit precision. Never `float`. Currency lives in a separate column.

Consistency matters more than the specific choice. Don't fight the existing convention.

### Identifiers — surrogate vs. natural

Use **surrogate keys** (auto-generated identifiers, `id` columns) almost always. Natural keys (email, SSN, ISBN) eventually need to change, and changing the primary key cascades through every foreign key.

For surrogate keys:
- **Bigint sequence** is the standard. Compact, sortable, fast.
- **UUID** when you need to generate ids on the client / before insert / across systems without coordination, or when you want IDs that can't be enumerated. Use **UUID v7** (timestamp-prefixed) so insertion order matches ID order — much better for index locality than UUID v4.
- **Composite keys** for true junction tables (`(user_id, role_id)`).

Don't expose surrogate keys publicly if you don't want enumeration. Either use UUIDs or use a separate `external_id` UUID column for public references.

### Soft deletes — yes, no, sometimes

Soft delete = `deleted_at` timestamp instead of `DELETE`. Pros: reversible, audit trail, consistency with downstream consumers reading deleted rows. Cons: every query must filter `WHERE deleted_at IS NULL`; orphaned references can build up; storage grows.

Defensible defaults:
- **Soft delete by default** for user-facing entities (customers, orders, posts) — accidents happen and recovery matters.
- **Hard delete** for high-volume ephemeral data (sessions, audit logs older than retention period, expired tokens).
- **Use row-level security or a data-access layer** so the `WHERE deleted_at IS NULL` filter is hard to bypass. Partial indexes help performance and uniqueness for live rows, but they do not automatically filter query results.

If you go soft-delete, plan the **purge** path: a scheduled job that hard-deletes rows past their retention. Otherwise tables grow forever.

## Indexes

Indexes are the single most-leveraged tool for query performance. Most slow queries are missing or wrong indexes.

### How indexes work (the necessary intuition)

A B-tree index is a sorted lookup structure. Given `WHERE column = X`, the index tells the database which rows have that value, in roughly O(log n) time, without scanning the whole table.

Key properties:

- **Composite indexes are ordered.** `(a, b, c)` lets you efficiently look up by `a`, by `(a, b)`, or by `(a, b, c)` — but **not by `b` alone** or `(b, c)`. The leftmost-prefix rule.
- **Index direction matters for sorts.** An index on `(created_at)` lets `ORDER BY created_at ASC` be cheap; `ORDER BY created_at DESC` is also cheap (Postgres can scan backward). But `ORDER BY a ASC, b DESC` may need a specific composite index.
- **Equality before range.** A composite `(a, b)` works well for `a = 1 AND b > 5`. For `a > 1 AND b = 5`, the index can only use `a` for filtering; `b` requires a scan. Put equality columns first.
- **Selectivity matters.** An index on `is_active` (probably 50% true) is barely useful. An index on `customer_id` (one customer has few orders relative to the table) is highly useful.
- **Covering indexes** include all the columns the query reads, so the database doesn't have to look at the heap. `INCLUDE` clause in Postgres; just-add-the-columns in MySQL.

Read the query plan (`EXPLAIN` / `EXPLAIN ANALYZE` in Postgres, `EXPLAIN` in MySQL). It tells you whether your index is actually being used.

### What to index

- **Foreign keys.** Almost always. The database does *not* automatically index FKs; you must.
- **Columns in WHERE clauses for queries you actually run.**
- **Columns in JOIN conditions.**
- **Columns in ORDER BY** if the query has a useful LIMIT.
- **Columns used for uniqueness** (the unique constraint creates one).
- **Composite indexes for queries that filter on multiple columns.** Pay attention to column order.

### What not to index

- **Columns rarely queried.** The index has cost — every write updates it; every column is more storage and more memory.
- **Low-selectivity columns alone** (booleans, small enums). Use them as the *second* column in a composite index, behind a more selective column.
- **Columns that change often** unless you really need the index. Each update writes the index too.
- **Many wide indexes "just in case."** They slow down writes and consume memory.

A useful audit: every index in the database should justify its existence. Tools like `pg_stat_user_indexes` show which indexes are actually being used. Drop indexes with zero usage over a meaningful period.

### Specialty indexes

PostgreSQL has rich index types beyond B-tree:

- **Hash indexes** — equality only, sometimes faster than B-tree. Rarely worth it; B-tree usually fine.
- **GIN (Generalized Inverted Index)** — for JSONB, arrays, full-text search. Indexes the *contents* of composite values.
- **GiST (Generalized Search Tree)** — geometric data, ranges, full-text, exclusion constraints.
- **BRIN (Block Range Index)** — for very large tables where data is naturally clustered (e.g., timestamps in append-only tables). Tiny index size, useful for analytical queries.
- **Partial indexes** — `CREATE INDEX ... WHERE active = true`. Only indexes the rows you actually query; smaller, faster.
- **Expression indexes** — `CREATE INDEX ON users (lower(email))` so `WHERE lower(email) = ?` can use the index.

Knowing these exist saves you from rolling your own when the database has a built-in answer.

## Queries

### Read the query plan

`EXPLAIN ANALYZE` is non-negotiable for any non-trivial query. The plan shows you what the database actually does:

- **Sequential Scan** on a big table is usually a red flag. Expected for very small tables; otherwise, missing index.
- **Index Scan** vs **Index Only Scan** — the latter doesn't touch the heap (covering index); much faster.
- **Hash Join** vs **Nested Loop** vs **Merge Join** — each is right for different inputs. Nested loop on two large tables is a bug.
- **Sort** with a high cost — needed an index on the sort columns.
- **Filter** above an Index Scan — the index isn't filtering as much as you thought.
- The actual row counts vs. estimated row counts. Big mismatches mean stale statistics or a query the planner mis-estimates.

If you can't read query plans, a substantial chunk of database performance work is invisible to you. Markus Winand's *SQL Performance Explained* and the *Use The Index, Luke!* website are the standard introduction.

### N+1 queries

The single most common database performance bug — see `performance-engineering` and the smell catalog.

```python
# Bad
users = db.query("SELECT * FROM users WHERE active = true")
for user in users:
    user.orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)
```

**Fix:** one query with `JOIN` or `IN (...)`, or the ORM's eager-loading. ORMs hide N+1 behind innocent attribute access; always check the SQL log for surprise queries.

### Avoid these specific patterns

- **`SELECT *`** — see smell catalog. Name your columns.
- **`OFFSET` for pagination on large tables** — slow at large offsets (the database scans everything before the offset). Use cursor-based pagination.
- **`COUNT(*)` on large tables** — exact count requires a full scan; consider an estimate (`pg_class.reltuples`) or maintain a counter.
- **`WHERE NOT EXISTS` vs. `LEFT JOIN ... WHERE ... IS NULL`** — equivalent in modern planners but the optimizer sometimes prefers one. Check the plan.
- **`OR` across columns** — often defeats indexes; consider `UNION` of single-condition queries.
- **Functions on indexed columns in WHERE** — `WHERE lower(email) = ?` doesn't use a plain index on `email`. Use an expression index, or store the lowercased version.
- **Type mismatches in JOINs / WHERE** — `int` vs. `bigint`, `varchar` vs. `text`. Implicit conversion can disable index usage.

## Transactions and isolation

Most application bugs around databases involve transactions handled wrong.

### Default to a transaction per logical operation

A "logical operation" is what should atomically succeed or fail — placing an order, creating a user, processing a payment. The transaction wraps every database call in that operation.

Most ORMs and frameworks have a "transaction per request" or "transaction per operation" pattern. Use it. Manual `BEGIN`/`COMMIT` is for exceptional cases.

### Know your isolation level

Standard SQL defines four levels; databases interpret them somewhat differently. The defaults vary:

- **PostgreSQL**: default `READ COMMITTED`. Each statement sees only data committed before that statement started; statements within a transaction can see different snapshots.
- **MySQL (InnoDB)**: default `REPEATABLE READ`. Consistent reads use a transaction snapshot; locking reads and writes are current reads with next-key/gap-lock behavior, so it is not the same thing as PostgreSQL snapshot isolation.
- **SQL Server**: default `READ COMMITTED` with locking.

**Important Postgres gotcha:** PostgreSQL's `REPEATABLE READ` is *stronger* than the SQL standard — it provides full snapshot isolation, preventing phantom reads within a transaction (the standard only requires it for already-read rows). Postgres's `SERIALIZABLE` is even stronger: it adds serialization-failure detection, retrying transactions that would produce a non-serializable schedule. So in Postgres specifically:
- `READ COMMITTED` → cheap; statement-level snapshot; phantoms possible.
- `REPEATABLE READ` → snapshot isolation; no phantoms; write skew possible.
- `SERIALIZABLE` → snapshot isolation + serialization detection; transactions may abort with a serialization-failure error and need retry.

**When to reach for SERIALIZABLE:** when correctness depends on a multi-row invariant that you cannot express as a database constraint or check. (E.g., "at least one admin must exist," "total of all sub-balances equals the master balance.") The cost is real — transactions can fail with serialization errors under contention and must be retried; throughput drops on write-heavy paths. Use it for the small set of transactions that genuinely need it; keep the rest at READ COMMITTED.

What this means in practice — three common bugs:

**Lost update.** Two transactions both `SELECT count`, both `UPDATE count = count + 1`. Without proper isolation or row locking, one update silently overwrites the other.

```sql
-- Both transactions:
BEGIN;
SELECT count FROM counters WHERE id = 1;  -- both read 5
-- compute 5 + 1 = 6 in app code
UPDATE counters SET count = 6 WHERE id = 1;  -- both write 6
COMMIT;
-- Final value: 6, but two increments happened. Lost update.
```

**Fix:** atomic update (`UPDATE counters SET count = count + 1 WHERE id = 1`), `SELECT ... FOR UPDATE`, optimistic concurrency with a version column, or `SERIALIZABLE` isolation.

**Phantom read.** A transaction reads a range, another inserts into that range, the first reads again and sees new rows. Real bug for "check then insert" patterns. PostgreSQL's `REPEATABLE READ` does prevent these (its snapshot isolation is strong).

**Write skew.** Two transactions read the same data, each makes a decision based on what they read, both commit. The result is inconsistent because each transaction's decision assumed the other wouldn't happen. ("Allow this delete only if there's at least one other admin" — both transactions check and see two admins, both delete, now there are zero admins.)

**Fix:** express the invariant as a unique, foreign-key, check, or exclusion constraint when possible. For invariants that cannot be expressed as constraints, use `SERIALIZABLE`, advisory/table locks, or carefully designed locking reads. In PostgreSQL, `SELECT ... FOR UPDATE` locks returned rows only; it does not lock a missing row or arbitrary range gap, so it is not a general phantom-read fix. In MySQL/InnoDB, gap/next-key protection depends on indexed locking reads and isolation details.

### Don't hold transactions open across slow operations

A transaction that's open for seconds (or worse, minutes) holds locks. Other writers wait. Writes pile up. Eventually, deadlock or timeout cascades.

**Don't:**
- `BEGIN` → fetch data → make HTTP calls → use the response → `COMMIT`.
- Hold transactions across user input ("waiting for the user to confirm").

**Do:**
- Compute outside; transact briefly to write.
- For long workflows, use sagas / outbox pattern (see `concurrency-and-state`).

### Connection pools

A connection pool is required machinery, not an optimization. Without it:
- Each request opens a new database connection.
- PostgreSQL forks a process per connection (5-15MB each).
- Connection storms during traffic spikes overwhelm the database.

The discipline:

- **Use a connection pool** (your framework's, or an external one like PgBouncer for Postgres).
- **Pool size is bounded by what the database can handle**, not by your application's concurrency. A typical Postgres can handle 100-200 connections; with a pool, hundreds of clients share them.
- **Watch for pool exhaustion.** When the pool is empty, requests wait. A slow query can starve the pool and cascade.
- **Prefer transaction-mode pooling** (PgBouncer) when scaling Postgres beyond the per-process limit. Caveat: transaction-mode pooling breaks features that depend on session state (prepared statements, advisory locks, `SET LOCAL` outside transactions).
- **Don't pool across very different workloads** in the same pool — long analytical queries shouldn't block transactional writes. Separate pools.

### Deadlocks

Two transactions each holding a lock the other wants.

**Common causes:**
- Updating the same set of rows in different orders. (Tx1 updates A then B; Tx2 updates B then A.) Standard fix: always update in a consistent order (e.g., by ID).
- Foreign key constraints triggering implicit locks on the referenced row.
- Long transactions holding locks that newer transactions block on.

**Detection:** PostgreSQL detects deadlocks and aborts one transaction. You'll see "deadlock detected" in logs. Application should catch and retry (deadlocks are usually transient).

**Prevention:** consistent lock ordering, smaller transactions, advisory locks for cross-transaction coordination.

## Migrations

Schema changes are deploys; they have rollouts, rollbacks, and risks like any other.

### The cardinal rule: backward-compatible migrations

A migration is **backward-compatible** if the old code (running during the deploy) and the new code (rolling out) both work against the new schema.

The migration patterns that satisfy this:

- **Adding a nullable column** — backward-compatible.
- **Adding a column with a default** — backward-compatible if writes can complete.
- **Adding a non-nullable column without a default** — NOT backward-compatible. Use this dance:
  1. Add the column nullable.
  2. Backfill in batches.
  3. Make non-nullable.
- **Removing a column** — must remove all reads first (deploy code that doesn't read it), then remove the column.
- **Renaming a column** — multi-step:
  1. Add the new column.
  2. Code writes to both old and new.
  3. Backfill the new from the old.
  4. Code reads from new only.
  5. Code stops writing to old.
  6. Drop the old column.
- **Changing a column type** — like a rename, multi-step. Add the new column with the new type; migrate data; flip reads; drop the old.

Each step is a separate deploy. Yes, this is slow. The alternative is downtime or data loss.

### Specific high-risk migrations

- **Adding a non-nullable column with a default to a heavily-written table.** Some databases rewrite the entire table — multi-hour outage. PostgreSQL ≥ 11 avoids rewrites for non-volatile constant defaults, but volatile expressions and older versions need care. MySQL behavior depends on version, storage engine, column position, and DDL algorithm.
- **Adding an index to a busy table.** In Postgres, plain `CREATE INDEX` blocks writes; use `CREATE INDEX CONCURRENTLY`. In MySQL/InnoDB, first check native online DDL support (`ALGORITHM=INPLACE` or `INSTANT`, `LOCK=NONE` where supported); use pt-online-schema-change or gh-ost when native DDL would copy or block too much.
- **Dropping a column** that "isn't used anymore" — must be backed by query logs showing zero reads. `grep` is not evidence.
- **Migrating data across millions of rows.** Use batched updates with checkpoints; never one transaction over the whole table.
- **Changing a unique constraint** — the constraint is enforced for new and existing rows; the migration can fail mid-way if existing data violates.

### Migration discipline

- Migrations are version-controlled; one direction (up) is enough for forward-only schemas; many teams skip writing down migrations and rely on backups for "rollback."
- Migrations run as part of deployment, with a known order, and a clear policy on which migrations run before vs. after the new code is live.
- Long-running migrations (backfills, reindexes) run as background jobs, not as blocking deploy steps.
- Pre-prod testing on prod-shape data — never assume a migration that ran fast on 1k rows runs fast on 100M.

## Multi-tenancy

If your system serves multiple customers/organizations and they must not see each other's data, the multi-tenancy model is a load-bearing decision.

Three common models:

1. **Shared database, shared schema, tenant_id column.** All customers in the same tables, filtered by `tenant_id`. Simplest to operate; the burden is on the application (or row-level security) to filter. Most common for SaaS.
2. **Shared database, separate schema per tenant.** Each tenant gets a schema (Postgres) or database (MySQL). Stronger isolation; more operational overhead; harder to query across tenants.
3. **Separate database per tenant.** Strongest isolation; most operational overhead; required for some compliance regimes (regional data residency, BAA-covered customers).

The mistake to avoid: starting with model 1, growing to ~10,000 tenants, then trying to migrate to model 3 for top-tier customers. Decide upfront which mix you'll support; pricing and contracts depend on this.

For model 1 specifically:

- **Use Postgres Row-Level Security** if available. The database enforces filtering for ordinary application roles. Set the `tenant_id` session variable per request, use a non-owner role without `BYPASSRLS`, and consider `FORCE ROW LEVEL SECURITY` for table owners.
- **Or filter at the query layer** consistently, ideally via a repository / data-access layer that's hard to bypass.
- **Test cross-tenant access aggressively.** Write tests that try to access another tenant's data with a valid auth token; they should return zero rows or 403.

## Operational concerns

### Backups and restores

The right question is not "do we have backups?" — it's "have we restored from them recently?"

- **Automated daily backups** to an off-site location.
- **Point-in-time recovery** (continuous archiving in Postgres, binary log in MySQL) so you can restore to any moment, not just last night.
- **Periodic restore drills** — actually restore a backup and verify the restored data is correct. Quarterly minimum.
- **Backup retention policy** — how long? What's the legal/compliance constraint? Cost?
- **Offsite copies** — a backup in the same datacenter doesn't survive a datacenter loss.

A backup that hasn't been tested doesn't exist for planning purposes. The horror story is universal: the backup runs nightly, alerts when it fails, succeeds for years — and is corrupt the day you need it.

### Replication and high availability

- **Streaming replication** (Postgres) / **binary log replication** (MySQL) for read replicas and failover.
- **Synchronous replication** when you need zero data loss on primary failure (cost: writes wait for the replica).
- **Asynchronous replication** when you accept small data loss on failover for higher write throughput.
- **Failover procedure** — automated where possible (Patroni for Postgres). Manual failover is acceptable but must be drilled.
- **Monitoring replication lag** — async replicas can fall behind under load. Lag above your threshold should alert.

### Connection limits and surge handling

The connection pool sits between application instances and the database. Sizing:

- Database `max_connections` — the absolute ceiling. Postgres typical: 100-300.
- Pool size per app instance × number of instances ≤ `max_connections` (with safety margin).
- For hundreds of app instances: external pooler (PgBouncer) sitting in front of the database, with much higher fan-in.

Under traffic surge:

- Pool exhaustion → requests queue → request timeouts → cascade.
- Slow queries hold connections → pool exhausts.
- Restart of the database → all clients reconnect simultaneously → thundering herd.

Mitigations: aggressive query timeouts, statement timeouts in the database, jittered reconnection, slow-query alerts, connection-count alerts.

### The database is the bottleneck

When the database is overloaded, the rest of the system's performance is irrelevant. Specifically watch for:

- **Connection saturation** — clients can't connect or are queuing.
- **Long-running queries** holding locks.
- **Replication lag** breaking read-after-write expectations.
- **Disk I/O saturation** — writes queue up; queries slow down.
- **Lock waits** — transactions blocked on each other.
- **Vacuum/analyze backlog** (Postgres) — table bloat grows; query plans go bad.

Each has a metric and an alert. Standard observability: connection counts, lock waits, slow query log, replication lag, table sizes, autovacuum status.

## ORMs honestly

ORMs are useful and dangerous. Useful: type-safe row-to-object mapping, query construction, migration tooling, common patterns. Dangerous: hide what they're doing, encourage N+1, generate inefficient queries, complect data access with object lifecycle.

Pragmatic stance:

- **Use the ORM for CRUD and simple queries.** It's much faster than writing raw SQL for these.
- **Drop to raw SQL or query builders for non-trivial queries.** Reports, complex joins, window functions — the ORM is fighting you. Most ORMs let you escape; do.
- **Always read the SQL the ORM generates.** Turn on SQL logging in development. Surprises live there.
- **Beware lazy loading.** Attribute access that triggers a query is the most common source of N+1.
- **Beware cascading saves.** "Save this user" can cascade to saving 50 related objects, each with its own validation.
- **Beware identity map / unit of work bugs.** Two queries return "the same" object; modifying one affects the other; tests pass because they don't notice; production has subtle bugs.

The honest version: ORMs save substantial time for simple cases and cost substantial time for complex ones. Plan to use raw SQL for the 10-20% of queries that aren't simple.

## What to flag in review

- **A new table with no primary key.** Almost always wrong.
- **A new column without NOT NULL or a documented reason.**
- **A new query without an index that would support it.**
- **A new ORM query in a loop** (probable N+1).
- **A migration that's not backward-compatible** without a multi-step rollout plan.
- **A migration that drops a column** without evidence of zero usage.
- **A query with `SELECT *`** in application code (acceptable for ad-hoc; bad in app).
- **A schema with no foreign keys** ("we'll enforce in code") — you won't.
- **`varchar(255)`** as a default — pick the right size for the actual constraint, or `text` if there isn't one.
- **`float`** for money. Use `numeric`.
- **Wall-clock comparisons** between rows from different replicas (clocks drift; see `concurrency-and-state`).
- **A long-running transaction** wrapping I/O.
- **A new dependency on a NoSQL store** without a clear case for why SQL doesn't work.
- **A new endpoint that does many small queries** when one query would do.
- **`OR` across many columns** without checking the query plan.
- **A new index without checking if an existing one covers the query.**
- **A schema change without a rollback plan** for high-risk migrations.

## Reference library

- `references/query-plan-reading.md` — how to read EXPLAIN output and what to look for.
- `references/migration-patterns.md` — concrete patterns for common high-risk schema changes.

# Migration Patterns

Concrete recipes for common high-risk schema changes. Each pattern is a sequence of small, individually-deployable steps, each of which is backward-compatible (old code and new code both work).

Examples below are shown as raw SQL for clarity. Most teams use a migration framework (Alembic, Flyway, Liquibase, ActiveRecord, TypeORM, Prisma, Sequel, knex, Migra, sqitch). The frameworks automate the *mechanics* (running migrations in order, recording state, generating boilerplate). The *patterns* — backward-compatible steps, multi-deploy renames, batched backfills — are the responsibility of the engineer, regardless of which framework is in use. Don't expect your framework to know that an `add_column` with a default rewrites a 50M-row table; that's on you.

## The cardinal rule

A migration is **safe** if at every moment during the rollout:

- The currently-deployed code works against the current schema.
- The currently-deployed code works against the next schema state (because deploys aren't atomic across replicas).
- A rollback (revert to previous code) still works against the current schema.

If any of those is false, you have downtime or data loss waiting to happen.

## Adding a column

### Nullable column with no default

Trivial. One step.

```sql
ALTER TABLE orders ADD COLUMN promo_code TEXT;
```

Old code ignores it. New code reads/writes it.

### Column with a default value

PostgreSQL ≥ 11: trivial — the default is stored as metadata, no rewrite of existing rows.
Older databases / MySQL: rewrites the table. For large tables, this can be a multi-hour outage.

For older databases / MySQL on a busy table, use the multi-step:

1. Add the column nullable, no default.
2. Backfill existing rows in batches.
3. Set the default in app code (or as a separate migration).

### Non-nullable column

Always multi-step:

1. **Add the column nullable.**
   ```sql
   ALTER TABLE orders ADD COLUMN status TEXT;
   ```
2. **Deploy code that writes the column.** Old code can still write rows without it; new code always writes it.
3. **Backfill existing rows.**
   ```sql
   -- In batches, with checkpointing
   UPDATE orders SET status = 'unknown' WHERE status IS NULL AND id < ?;
   ```
4. **Make non-nullable.**
   ```sql
   ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
   ```
   On Postgres, this requires a full table scan (briefly locks). For very large tables, consider a CHECK constraint with `NOT VALID` first, then `VALIDATE CONSTRAINT` (which doesn't lock writes).

## Dropping a column

Multi-step. Reverse of adding.

1. **Stop reading the column.** Deploy code that doesn't reference it. (Verify with logs / queries — `grep` is not evidence.)
2. **Stop writing the column** (if not already covered by step 1).
3. **Wait.** A meaningful period — at least one full deployment cycle, ideally a release or two — to make sure nothing actually depended on the column.
4. **Drop the column.**
   ```sql
   ALTER TABLE orders DROP COLUMN legacy_status;
   ```

The temptation is to skip the wait. Don't. A consumer you didn't know about (a report, a script, an analytics pipeline, a data warehouse export) breaks the day after.

## Renaming a column

Renaming in place is a breaking change for old code mid-deploy. Multi-step:

1. **Add the new column.**
   ```sql
   ALTER TABLE orders ADD COLUMN order_status TEXT;
   ```
2. **Deploy code that writes both old and new.**
   ```python
   order.status = "paid"
   order.order_status = "paid"
   ```
3. **Backfill new from old.**
   ```sql
   UPDATE orders SET order_status = status WHERE order_status IS NULL;
   ```
4. **Deploy code that reads from the new column** (still writes both).
5. **Deploy code that writes only the new column** (stops writing the old).
6. **Drop the old column** (after the wait).

Six deploys for a rename. Yes. The alternative is downtime when an old replica writes to the dropped column or a new replica reads from the missing column.

## Changing a column type

Like a rename, multi-step. The trick is the new column has the new type.

1. Add `new_amount` with the new type.
2. Code writes both `amount` and `new_amount` (with conversion if needed).
3. Backfill in batches.
4. Code reads from `new_amount`.
5. Stop writing to `amount`.
6. Drop `amount`.
7. Optionally rename `new_amount` → `amount` via the rename dance.

For type widening that's binary-compatible (int → bigint, varchar(10) → varchar(50)), some databases do this in metadata only. Check.

## Adding an index

A naive `CREATE INDEX` on a busy table acquires an exclusive lock, blocking writes for the duration. For a 1B-row table, that's hours.

**PostgreSQL:** use `CONCURRENTLY`.
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```
This does the work in the background, only briefly locking at the start and end. Slower than the locking version but doesn't block writes.

Caveats:
- `CONCURRENTLY` cannot run inside a transaction.
- If it fails partway, you get an `INVALID` index — drop it and retry.
- It still uses I/O; on heavy systems, do it during a quiet period.

**MySQL:** use online DDL (`ALGORITHM=INPLACE, LOCK=NONE`) where supported, or pt-online-schema-change for older versions.

## Dropping an index

Generally safe but verify nothing's using it first:

```sql
SELECT * FROM pg_stat_user_indexes WHERE indexrelname = 'idx_orders_xyz';
-- idx_scan should be 0 (or near 0) over a meaningful period.
```

Drop concurrently to avoid locking:
```sql
DROP INDEX CONCURRENTLY idx_orders_xyz;
```

## Adding a unique constraint

Risky: existing data may already violate uniqueness. The constraint creation will fail mid-way.

Pattern:

1. **Find duplicates first:**
   ```sql
   SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;
   ```
2. **Resolve duplicates** (deduplicate, mark obsolete, etc.).
3. **Create unique index concurrently:**
   ```sql
   CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_unique ON users (email);
   ```
4. **Promote to constraint:**
   ```sql
   ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE USING INDEX idx_users_email_unique;
   ```

This avoids the table-rewrite path of `ALTER TABLE ADD CONSTRAINT UNIQUE`.

## Adding a foreign key

Postgres: `ALTER TABLE ADD CONSTRAINT ... FOREIGN KEY ... NOT VALID` adds the constraint without scanning existing rows (existing rows are not validated; new writes are).

```sql
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
  FOREIGN KEY (customer_id) REFERENCES customers (id)
  NOT VALID;
```

Then later:

```sql
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;
```

Validation reads existing rows but doesn't block writes (acquires only ShareUpdateExclusive lock).

## Splitting a table

Moving columns from one table to another, or breaking one table into two related tables.

Pattern (split `users` into `users` + `user_profiles`):

1. **Create the new table.**
2. **Code writes to both** (existing columns in `users`, new copies in `user_profiles`).
3. **Backfill existing data.**
4. **Code reads from the new table.**
5. **Code stops writing the old columns** in `users`.
6. **Drop the old columns** in `users`.

Each step is a separate deploy.

## Merging tables

Reverse of splitting. Same pattern in reverse.

## Big data migrations

For migrations that affect millions or billions of rows:

- **Batch.** Don't `UPDATE entire_table SET ...` in one transaction. Update in chunks of 1k-100k rows.
- **Checkpoint.** Track which rows you've processed (a "migrated" column, or a bookmark table) so you can resume.
- **Throttle.** Pause between batches so the DB can keep serving traffic. Watch replication lag.
- **Run as a background job**, not as part of the deploy.
- **Monitor** the migration progress; alert on stall.
- **Plan for retries.** Each batch should be idempotent (safe to retry).

```python
# Sketch
last_id = read_checkpoint() or 0
while True:
    batch = db.execute(
        "SELECT id FROM users WHERE id > ? AND new_column IS NULL ORDER BY id LIMIT 1000",
        last_id
    )
    if not batch:
        break
    db.execute(
        "UPDATE users SET new_column = compute(...) WHERE id IN (?)",
        [r.id for r in batch]
    )
    last_id = batch[-1].id
    write_checkpoint(last_id)
    time.sleep(0.1)  # throttle
    if replication_lag() > 10:
        time.sleep(10)  # back off
```

## High-cardinality enum changes

Changing a column from `VARCHAR` (free-text) to `ENUM`:

- **Don't.** Postgres enums require ALTER TABLE for new values (multi-step in production).
- **Use a CHECK constraint or a foreign key to a reference table.** Adding new values is just an INSERT.

```sql
-- Instead of:
status ENUM('pending', 'paid', 'cancelled')

-- Do:
status TEXT NOT NULL,
CHECK (status IN ('pending', 'paid', 'cancelled'))
-- or
status_id INT NOT NULL REFERENCES order_statuses(id)
```

## Rollback strategy

Every migration deploy needs a rollback story. The two main strategies:

**Forward-only:** every migration is applied; if the new code is bad, you fix it forward (deploy a new version). The schema is never rolled back. Most teams converge on this. Requires migrations to always be backward-compatible (so the old code keeps working when you revert it).

**Reversible:** every migration has an explicit "down" that undoes it. Tooling supports this (Rails migrations, etc.). Sounds nice but: many migrations are not actually reversible (data has been transformed and the original is lost). The "down" path is rarely tested. You usually fix forward anyway.

In practice: assume forward-only, write migrations that don't require rollback, and rely on backups for the catastrophic case.

## What to flag in review

- A migration that's not backward-compatible (old code breaks against the new schema, or new code breaks against the old).
- A migration that adds a non-nullable column without the multi-step pattern.
- A migration that adds an index without `CONCURRENTLY` (or equivalent) on a non-trivial table.
- A migration that drops a column without evidence of zero usage.
- A migration that updates a large table in one transaction.
- A migration without a stated rollback plan when the change is risky.
- A "rename" or "type change" done in one step.
- A migration that depends on the new code being deployed everywhere first (or vice versa) without explicit ordering documentation.

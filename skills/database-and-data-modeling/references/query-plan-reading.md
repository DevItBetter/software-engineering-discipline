# Reading Query Plans

If you can't read query plans, a substantial chunk of database performance work is invisible to you. This is a working reference for the common operations and what they tell you. Examples are PostgreSQL syntax; MySQL/SQL Server differ in display but the concepts transfer.

## Get a plan

```sql
EXPLAIN SELECT ...;            -- planner's expected plan
EXPLAIN ANALYZE SELECT ...;    -- actually runs it; shows actual times and rows
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;  -- also shows buffer hits/reads (cache vs disk)
```

`EXPLAIN ANALYZE` *runs the query*, but it measures executor work, not full client-observed latency. PostgreSQL does not deliver result rows to the client during `EXPLAIN ANALYZE`; large-result serialization, fetch, rendering, and network time may be missing. Measure from the application too, or use `EXPLAIN (ANALYZE, SERIALIZE)` where appropriate.

For write statements, start with plain `EXPLAIN`. Use `EXPLAIN ANALYZE` on writes only on non-production, a restored prod snapshot, or during an approved maintenance/debug window; `ROLLBACK` undoes data changes but cannot undo locks taken, trigger execution, sequence consumption, WAL, or production latency impact.

If you explicitly accept those effects, wrap in a transaction and rollback:

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE ...;
ROLLBACK;
```

## The plan structure

Plans are trees, read inside-out / bottom-up. The leaves are scans; intermediate nodes combine rows; the root produces the final result.

```
Sort  (cost=...  rows=10  width=...)
  Sort Key: created_at DESC
  ->  Hash Join  (cost=...  rows=10000  width=...)
        Hash Cond: (orders.customer_id = customers.id)
        ->  Seq Scan on orders  (cost=...  rows=100000  width=...)
              Filter: (status = 'paid')
        ->  Hash  (cost=...  rows=1000  width=...)
              ->  Seq Scan on customers  (cost=...  rows=1000  width=...)
```

This reads as: scan all orders filtering by paid status; scan all customers; hash-join them; sort the result.

Key fields:
- **cost** — planner's estimate (start..total). Total cost often dominates batch queries; startup cost matters for `LIMIT`, `EXISTS`, cursor/streaming, and first-row latency.
- **rows** — estimated row count. With `ANALYZE`, also the actual count.
- **width** — average row size in bytes.
- **actual time** (with ANALYZE) — first row .. all rows, in milliseconds. With `loops > 1`, actual rows/time are per-loop averages; multiply mentally when estimating total repeated work.

## The operations and what they mean

### Seq Scan

Reads the whole table.

**OK when:**
- The table is small (a few thousand rows).
- The query genuinely needs most of the rows.
- A small table where index lookup overhead exceeds scan cost.

**Red flag when:**
- Table is large and query has a selective WHERE clause. Means no usable index.
- An OLTP query that should be index-driven.

```
->  Seq Scan on users  (cost=0.00..15000.00 rows=100 width=...)
      Filter: (email = 'a@b.com')
      Rows Removed by Filter: 999900
```

The Rows-Removed-by-Filter line is the bug: scanned 1M rows to find 100. Add an index on email.

### Index Scan

Uses an index to find rows, then reads each from the heap (table data).

```
->  Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=...)
      Index Cond: (email = 'a@b.com')
```

Good. Index Cond shows the index was used.

### Index Only Scan

Uses an index that contains all needed columns; doesn't touch the heap. Faster than Index Scan.

```
->  Index Only Scan using idx_orders_customer_status on orders  (cost=...)
      Index Cond: (customer_id = 42)
      Heap Fetches: 0
```

`Heap Fetches: 0` means the visibility map said the index was sufficient. With covering indexes (Postgres 11+ `INCLUDE`), this is achievable for queries that read several columns.

### Bitmap Index Scan + Bitmap Heap Scan

For queries returning many rows where the index is helpful but not selective enough for individual lookups.

```
Bitmap Heap Scan on orders  (cost=... rows=10000 width=...)
  Recheck Cond: (customer_id = 42)
  ->  Bitmap Index Scan on idx_orders_customer_id  (cost=... rows=10000 width=0)
        Index Cond: (customer_id = 42)
```

The bitmap variant scans the index to build a bitmap of matching pages, then reads the heap pages once each (avoiding random I/O). Good for medium-selectivity filters.

### Hash Join

Builds a hash table of the smaller side; probes from the larger side. Good for joining largeish unsorted inputs.

```
Hash Join
  Hash Cond: (orders.customer_id = customers.id)
  ->  Seq Scan on orders ...
  ->  Hash
        ->  Seq Scan on customers ...
```

### Merge Join

Both sides are sorted; walks them in lockstep. Good when both inputs are large and already sorted (e.g., both come from sorted indexes).

### Nested Loop

For each row in the outer side, scan the inner side.

```
Nested Loop
  ->  Seq Scan on customers ...     (1000 rows)
  ->  Index Scan on orders ...      (per outer row)
```

**OK when:** the outer side is small and the inner side has a good index. The index-driven inner side makes this efficient.

**Red flag when:** both sides are large. O(n*m) is a disaster.

### Sort

Sorts a result set. If the cost is high and the query has an `ORDER BY`, the right index could eliminate this.

```
Sort  (cost=... rows=100000 ...)
  Sort Method: external merge  Disk: 50000kB
  Sort Key: created_at DESC
```

`external merge ... Disk` means it spilled to disk because `work_mem` was too small for that operation. Prefer query/index fixes first. If raising `work_mem`, do it per session/query where possible and account for concurrent connections and multiple sort/hash nodes.

### Aggregate / GroupAggregate / HashAggregate

How the database computes `GROUP BY` / aggregates. HashAggregate is faster but uses memory; GroupAggregate requires sorted input (often via index).

### Limit

```
Limit  (cost=... rows=10 ...)
  ->  Sort ...
        ->  Seq Scan ...
```

`LIMIT 10 ORDER BY created_at` over a large table can be very fast with the right index (index gives sorted order; database stops after 10) — or very slow if it requires sorting the whole table first.

For pagination, `OFFSET` doesn't avoid scanning before the offset; for large offsets, use cursor-based pagination instead.

## What "fast" looks like

A small OLTP query (point lookup, get user by ID, list a few of someone's recent orders):

- Single-digit milliseconds total.
- Index Scan or Index Only Scan, no Seq Scans on big tables.
- No external sorts.
- Buffers Hit > Buffers Read (data in cache).

A medium analytical query (sum sales by region for last quarter):

- Tens to hundreds of milliseconds, depending on data size.
- Possibly a Bitmap Heap Scan with a partial index, or a covering index.
- HashAggregate or GroupAggregate at the top.
- May read from disk; that's fine for analytics.

## Common red flags

- **Seq Scan on a big table with a selective filter.** Missing index.
- **Estimated rows ≠ actual rows by a lot** (10x+). Stale statistics; run `ANALYZE table_name`.
- **Sort with `external merge ... Disk`.** Spilling. Increase work_mem or add a sort-supporting index.
- **Nested Loop with both sides large.** Wrong join strategy; check indexes and statistics.
- **Index Cond is not what you expected** — the planner used an index but for a different condition than you thought; the rest of your WHERE became a Filter.
- **High Heap Fetches in Index Only Scan.** Vacuum is behind; the visibility map is stale.
- **Buffers Read >> Buffers Hit.** Data not in cache; either the table is too big for cache or this is a rarely-run query and that's OK.

## Statistics and the planner

The planner uses statistics about the data (column histograms, distinct values) to choose plans. Stale statistics produce bad plans.

- **`ANALYZE table_name;`** — refresh statistics for one table.
- **`VACUUM ANALYZE;`** — vacuum and analyze.
- **Autovacuum** runs continuously on Postgres. Verify it's keeping up; backed-up autovacuum produces bloat and bad statistics.
- For specific cases (skewed data, complex correlations), consider extended statistics (`CREATE STATISTICS`).

When a query suddenly gets slow without a code change: check whether statistics went stale (often: a big bulk insert, or autovacuum disabled).

## When the plan looks fine but the query is slow

- **Network / connection latency** — every round trip is ~1-100ms.
- **Lock waits** — `pg_stat_activity` shows blocked queries.
- **Replication lag on read replicas** — read might be slow because the replica is catching up.
- **Disk I/O saturation** — see system metrics; `Buffers Read` going to disk.
- **Concurrency** — query is fine in isolation, slow under load. Try `EXPLAIN ANALYZE` while production load is happening (carefully).

## What to flag in review

For any query in a hot path:

- A copy of the EXPLAIN ANALYZE output, or a confirmation it was checked.
- The index that supports the query (named).
- Confirmation it's an Index Scan or Index Only Scan, not a Seq Scan on a big table.
- For queries with sorts, confirmation the sort comes from an index (or the dataset is small).
- For joins, confirmation the join strategy is reasonable.

A new query in production code without anyone having looked at its plan is a future incident.

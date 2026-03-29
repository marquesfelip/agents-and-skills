---
name: indexing-strategies
description: >
  Database indexing and query performance optimization through correct index design.
  Use when: indexing strategy, database index design, index optimization, B-tree index, hash index,
  composite index, partial index, covering index, index selectivity, missing index, unused index,
  index bloat, GIN index, GiST index, BRIN index, full-text search index, PostgreSQL index,
  MySQL index, index cardinality, explain plan index, slow query index, over-indexing,
  index maintenance, index rebuild, query planner, index scan, sequential scan, index only scan.
argument-hint: >
  Describe the table, access patterns, and slow queries (e.g., "orders table accessed by customer_id,
  status, and date range — query takes 3s on 50M rows"), and specify the database engine and current indexes.
---

# Indexing Strategies Specialist

## When to Use

Invoke this skill when you need to:
- Design indexes for a new table based on access patterns
- Identify missing indexes causing slow queries
- Remove unused or redundant indexes that add write overhead
- Choose the right index type (B-tree, Hash, GIN, GiST, BRIN, Partial, Covering)
- Analyze `EXPLAIN` / `EXPLAIN ANALYZE` output to understand index usage
- Balance read performance against write/storage overhead

---

## Step 1 — Map Access Patterns First

Indexes serve queries. Start by cataloging how the table is accessed:

| Query Type | Columns Used | Frequency | Latency Target |
|---|---|---|---|
| Point lookup | `id` | High | < 1ms |
| Filter + sort | `status`, `created_at` | High | < 50ms |
| Range scan | `created_at BETWEEN` | Medium | < 100ms |
| Search | `LIKE '%keyword%'` | Low | < 500ms |
| Aggregation | `GROUP BY customer_id` | Medium | < 200ms |

Checklist:
- [ ] All `WHERE`, `JOIN ON`, `ORDER BY`, and `GROUP BY` columns cataloged per query
- [ ] Query frequency and latency targets documented
- [ ] Write-to-read ratio estimated — high write ratio increases the cost of extra indexes
- [ ] Existing slow queries identified via `pg_stat_statements` or slow query log

---

## Step 2 — Index Types Reference

**B-tree (default — use for most cases):**
- Supports: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `LIKE 'prefix%'`
- Supports `ORDER BY` (eliminates sort step)
- Does **not** support: `LIKE '%suffix'`, full-text, array contains

**Hash (PostgreSQL, MySQL):**
- Supports only `=` — faster than B-tree for pure equality lookups
- Not replicated in older PostgreSQL versions; B-tree is usually preferred

**GIN (PostgreSQL — Generalized Inverted Index):**
- Supports: array containment (`@>`), JSONB operators, full-text search (`@@`)
- Use for: `jsonb`, `tsvector`, `array` columns

**GiST (PostgreSQL — Generalized Search Tree):**
- Supports: geometric types, range types, full-text search
- Use for: `tsrange`, `geometry`, PostGIS, exclusion constraints

**BRIN (PostgreSQL — Block Range INdex):**
- Extremely compact; works on naturally ordered data (timestamps, sequential IDs)
- Use for: time-series tables, append-only logs where `created_at` is monotonically increasing
- Very fast to build; lookup is slower than B-tree — good for range filters on massive tables

**Partial Index:**
- Indexes only rows matching a `WHERE` clause
- Use for: `WHERE deleted_at IS NULL` (index only active records), `WHERE status = 'pending'`

**Covering Index (Index-Only Scan):**
- PostgreSQL: `INCLUDE (col1, col2)` — stores extra columns in the index leaf
- MySQL: naturally covering if all selected columns are in the index
- Use when: query selects only columns that can all be served from the index

Checklist:
- [ ] B-tree used as default; specialized types only when the access pattern requires it
- [ ] GIN used for JSONB and array columns
- [ ] BRIN used for large append-only time-series tables
- [ ] Partial indexes used for filtered queries on a subset of rows

---

## Step 3 — Composite Index Design

Column order in a composite index matters:

**Rules:**
1. **Most selective first** — filter first by the column that eliminates the most rows
2. **Equality before range** — equality columns must come before range columns
3. **Sort column last** — if the query has `ORDER BY`, the sort column goes last to avoid a sort step

```sql
-- Query: WHERE customer_id = $1 AND status = $2 AND created_at > $3 ORDER BY created_at DESC
-- Good index:
CREATE INDEX idx_orders_customer_status_created
  ON orders (customer_id, status, created_at DESC);

-- Bad: range column before equality — partial index usage only
CREATE INDEX idx_orders_created_customer_status
  ON orders (created_at, customer_id, status);
```

Checklist:
- [ ] Equality filter columns appear before range filter columns in the index
- [ ] Composite index verified against the actual query's `WHERE` + `ORDER BY` clause
- [ ] Leftmost prefix rule respected: a composite index on `(a, b, c)` serves queries on `(a)`, `(a, b)`, `(a, b, c)` — **not** on `(b)` or `(c)` alone
- [ ] Existing single-column indexes removed when subsumed by a new composite index

---

## Step 4 — Identify Missing and Redundant Indexes

**Finding missing indexes (PostgreSQL):**
```sql
-- Indexes the planner considered but does not exist
SELECT relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND n_live_tup > 10000
ORDER BY seq_scan DESC;
```

**Finding unused indexes:**
```sql
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE 'pg_%'
ORDER BY relname;
```

**Finding redundant indexes:**
- Index `(a, b)` makes standalone `(a)` redundant for read — but `(a)` still serves as a unique/PK constraint
- Detect: look for indexes where the leftmost columns fully match another index

Checklist:
- [ ] `pg_stat_user_tables` queried for tables with high `seq_scan` and low `idx_scan`
- [ ] `pg_stat_user_indexes` queried for unused indexes (`idx_scan = 0`)
- [ ] Redundant single-column indexes removed after composite index creation
- [ ] Unused indexes removed — each write-path index adds overhead to `INSERT` / `UPDATE` / `DELETE`

---

## Step 5 — Read and Act on EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE customer_id = $1 AND status = 'pending'
ORDER BY created_at DESC
LIMIT 10;
```

Key nodes to interpret:

| Node | What It Means |
|---|---|
| `Seq Scan` | Full table scan — likely missing index |
| `Index Scan` | Index used; fetches heap rows |
| `Index Only Scan` | Covering index — no heap access |
| `Bitmap Heap Scan` | Multiple index entries combined; deduped before heap fetch |
| `Nested Loop` | Good for small outer / inner result sets |
| `Hash Join` | Good for large unsorted joins |
| `Sort` | Sort step performed — check if index covers ORDER BY |
| `rows=X (actual rows=Y)` | If X >> Y, statistics are stale — run `ANALYZE` |

Checklist:
- [ ] `Seq Scan` on a table > 10k rows investigated for missing index
- [ ] `Sort` node present on a frequent query — add sort column to index
- [ ] `rows` estimate very different from actual — run `ANALYZE tablename`
- [ ] `Buffers` output shows excessive `shared read` — cold cache or missing index

---

## Step 6 — Index Maintenance

Checklist:
- [ ] `REINDEX CONCURRENTLY` used to rebuild bloated indexes without downtime
- [ ] `autovacuum` tuned for high-write tables — prevents index bloat accumulation
- [ ] `pg_stat_user_indexes` reviewed periodically (monthly) for new unused indexes
- [ ] Index bloat measured with `pgstattuple` or `pgstatindex` extension
- [ ] `VACUUM ANALYZE` run after large data loads before executing batch queries
- [ ] MySQL: `OPTIMIZE TABLE` run after large deletes on InnoDB tables

---

## Output Report

### Critical
- Sequential scan on a many-million-row table for a high-frequency query — critical latency issue
- FK column has no index — parent-side deletes or joins perform full table scans
- Missing index on a `WHERE` clause column responsible for timeout-level query performance

### High
- Composite index column order incorrect — range column before equality column
- Covering index not used — heap fetches add unnecessary I/O for read-heavy queries
- Many unused indexes on a high-write table — write amplification without benefit

### Medium
- BRIN not used on a large append-only time-series table — B-tree overbuilt and bloated
- GIN not used for JSONB filter queries — falling back to sequential scan with operator push-down
- `pg_stat_statements` not enabled — missing insight into slow query patterns

### Low
- `ANALYZE` not run after large data loads — planner statistics stale, poor plan selection
- Redundant single-column indexes not cleaned up after composite index creation
- Index bloat not monitored — gradually degrading query performance

### Passed
- B-tree composite indexes designed with equality-before-range column ordering
- Specialized index types (GIN, BRIN, Partial) used where appropriate
- `EXPLAIN ANALYZE` confirmed index usage for all critical query paths
- Unused indexes removed and write overhead reduced
- Autovacuum tuned to prevent bloat on high-write tables

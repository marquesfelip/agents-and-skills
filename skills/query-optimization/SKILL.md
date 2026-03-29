---
name: query-optimization
description: >
  Efficient query patterns and performance diagnostics for relational databases.
  Use when: query optimization, slow query, query performance, explain plan, EXPLAIN ANALYZE,
  N+1 query, query tuning, SQL performance, slow query log, pg_stat_statements, query profiling,
  join optimization, subquery optimization, CTE optimization, window function performance,
  pagination optimization, cursor pagination, offset pagination, full table scan, query rewrite,
  query plan, statistics stale, table scan, connection pooling, query timeout, ORM N+1,
  eager loading, lazy loading, batch loading, query batching, database bottleneck.
argument-hint: >
  Provide the slow query or query pattern to optimize (include the SQL or ORM code, table sizes,
  and the current execution time), and specify the database engine and ORM if applicable.
---

# Query Optimization Specialist

## When to Use

Invoke this skill when you need to:
- Diagnose and fix a slow or timing-out query
- Eliminate N+1 query patterns from ORM-generated code
- Rewrite inefficient SQL using better join, subquery, or CTE strategies
- Optimize pagination for high-offset or large result sets
- Interpret `EXPLAIN ANALYZE` and choose corrective actions
- Reduce database load by batching, caching, or restructuring queries

---

## Step 1 — Identify the Problem Query

Before optimizing, locate and measure:

**PostgreSQL:**
```sql
-- Top slow queries (requires pg_stat_statements)
SELECT query, calls, total_exec_time / calls AS avg_ms, rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY avg_ms DESC
LIMIT 20;
```

**MySQL:**
```sql
-- Slow query log (enable: slow_query_log=ON, long_query_time=1)
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 20;
```

**Application-level detection:**
- ORM query logging enabled in development
- APM traces (Datadog, New Relic) showing DB spans > target latency
- Query count per request monitored (detect N+1)

Checklist:
- [ ] Slow query identified and its average execution time measured
- [ ] Query frequency measured (calls/sec) — low-frequency slow queries vs high-frequency moderate queries prioritized differently
- [ ] Query result size measured — large result sets may indicate missing `LIMIT` or missing pagination
- [ ] Environment reproduced with production-scale data before optimizing

---

## Step 2 — Run EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
<your query here>;
```

**Reading the output:**

| Node / Signal | Interpretation | Action |
|---|---|---|
| `Seq Scan` on large table | No usable index | Add index on filter column |
| `rows=1000 actual=1` | Stale statistics (over-estimate) | `ANALYZE tablename` |
| `rows=1 actual=1000` | Stale statistics (under-estimate) | `ANALYZE tablename` |
| `Sort (cost=high)` on frequent query | No index covering ORDER BY | Add ORDER BY column to index |
| `Hash Join` over large tables | Often optimal; check join key cardinality | — |
| `Nested Loop` with `Seq Scan` inner | Inner table not indexed on join key | Index the join column |
| `Buffers: shared read=high` | Cold cache or missing index | Warm cache or add index |
| `Filter: removed X rows` | Index used but filter not pushed down fully | Extend composite index to cover filter column |

Checklist:
- [ ] `EXPLAIN ANALYZE` run with `BUFFERS` to detect I/O issues
- [ ] Actual vs estimated rows compared — large divergence means stale statistics
- [ ] `Sort` node checked — can it be eliminated by adding ORDER BY to an index?
- [ ] Inner side of `Nested Loop` confirmed to use an index scan, not a seq scan

---

## Step 3 — Fix N+1 Query Patterns

N+1 occurs when loading N parent records each triggers 1 additional query:

```python
# Bad: N+1
orders = Order.all()
for order in orders:
    print(order.customer.name)  # 1 query per order

# Good: eager load
orders = Order.include(:customer).all()
```

**Detection:**
- ORM debug logging shows the same query repeated with different IDs
- APM shows hundreds of identical short DB queries per request

**Fixes:**
1. **Eager loading**: `JOIN` or separate `IN (...)` query to load related records in bulk
2. **DataLoader / batch loading**: batch multiple IDs into a single `WHERE id IN ($1, $2, ...)` query
3. **Denormalization**: store the frequently read field on the parent (e.g., `customer_name` on `orders`)
4. **Projection**: select only the columns needed rather than loading full models

Checklist:
- [ ] ORM query count measured per endpoint in development
- [ ] Eager loading configured for all `has-many` and `belongs-to` relationships accessed in a loop
- [ ] `SELECT *` replaced with explicit column projection
- [ ] DataLoader / batch loading used in GraphQL resolvers

---

## Step 4 — Optimize Joins

**Common join anti-patterns:**

```sql
-- Bad: implicit join (harder to read, same performance but error-prone)
SELECT * FROM orders, customers WHERE orders.customer_id = customers.id;

-- Bad: SELECT * in a JOIN — fetching unnecessary columns
SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id;

-- Good: explicit JOIN with projected columns
SELECT o.id, o.total, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';
```

**Join optimization rules:**
- [ ] Join columns indexed on both sides
- [ ] Filter applied in `WHERE` early to reduce the join input size
- [ ] For large tables: filter first in a CTE or subquery, then join
- [ ] Avoid joining on non-indexed expression (e.g., `ON LOWER(a.name) = LOWER(b.name)`) — use functional indexes instead

---

## Step 5 — Optimize Pagination

**Offset pagination (avoid for large pages):**
```sql
-- Bad for high offsets: scans all rows to skip them
SELECT * FROM orders ORDER BY created_at DESC OFFSET 10000 LIMIT 20;
```

**Keyset / Cursor pagination (preferred):**
```sql
-- Good: uses index; jumps directly to the cursor position
SELECT * FROM orders
WHERE created_at < $last_seen_created_at
   OR (created_at = $last_seen_created_at AND id < $last_seen_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**When to use which:**
| Strategy | Use When | Limitation |
|---|---|---|
| Offset | Low page numbers; stable data | Slow at high offsets |
| Keyset | Large datasets; infinite scroll | Cannot jump to arbitrary page |
| Count + Offset | User needs "go to page N" | Cap at page ~200 |

Checklist:
- [ ] Keyset pagination used for any query that may be called at high offsets
- [ ] Cursor encodes both a timestamp and an ID (for stable ordering with ties)
- [ ] Index covers both `ORDER BY` columns used in the cursor

---

## Step 6 — CTE and Subquery Optimization

**CTEs (WITH clause):**
- PostgreSQL pre-14: CTEs are optimization fences (always materialized)
- PostgreSQL 14+: CTEs are inlined by default unless `MATERIALIZED` is specified
- Use `WITH ... AS MATERIALIZED` when you want to prevent the planner from inlining

```sql
-- Force materialization when filtering in the CTE reduces rows significantly:
WITH recent_orders AS MATERIALIZED (
  SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days'
)
SELECT c.name, COUNT(o.id)
FROM customers c
JOIN recent_orders o ON c.id = o.customer_id
GROUP BY c.name;
```

**Subquery vs JOIN:**
- Correlated subqueries (reference outer query per row) → almost always replace with a JOIN
- `EXISTS` vs `IN`: use `EXISTS` when the subquery may return duplicates

Checklist:
- [ ] No correlated subquery where a JOIN can be used instead
- [ ] `IN (SELECT ...)` replaced with `EXISTS` or a JOIN for large subquery result sets
- [ ] CTE materialization behavior known for the target PostgreSQL version

---

## Step 7 — Application-Level Patterns

Checklist:
- [ ] Query results cached (Redis, Memcached) for read-heavy, stable data — with explicit TTL
- [ ] Connection pool sized appropriately: `pool_size = (core_count * 2) + disk_spindles` (PgBouncer default formula)
- [ ] Long-running analytics queries moved to a read replica — do not run on the primary
- [ ] Batch inserts use `INSERT INTO ... VALUES (...), (...), (...)` — not one insert per row
- [ ] `COPY` used for bulk imports instead of individual `INSERT` statements
- [ ] `RETURNING` used to avoid a follow-up `SELECT` after `INSERT`/`UPDATE`

---

## Output Report

### Critical
- N+1 query pattern causing hundreds of DB round-trips per HTTP request
- Sequential scan on a multi-million-row table for a high-frequency query — timeout risk
- Missing `LIMIT` on a query that can return unbounded rows — OOM risk

### High
- High-offset `LIMIT/OFFSET` pagination causing full-index scan on large tables
- Correlated subquery executed once per row instead of a JOIN — O(n) query multiplication
- Stale table statistics causing the planner to use a sequential scan when an index exists

### Medium
- `SELECT *` used in production queries — fetching unnecessary columns adds I/O and network overhead
- CTEs treated as optimization fences unintentionally on PostgreSQL < 14
- Join on non-indexed expression — functional index not created

### Low
- Batch inserts not used — individual `INSERT` per row for bulk operations
- Analytics queries running on primary DB — consuming write-path resources
- `ANALYZE tablename` not run after large bulk loads

### Passed
- EXPLAIN ANALYZE confirms index scans on all critical query paths
- N+1 patterns resolved with eager loading or DataLoader batching
- Keyset pagination used for high-volume paginated endpoints
- Connection pool sized correctly; no pool exhaustion under load
- Long-running analytics queries routed to read replica

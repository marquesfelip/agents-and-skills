---
name: database-performance
description: >
  Database performance tuning practices through query optimization, indexing, connection pooling,
  and schema-level improvements.
  Use when: database performance, slow query, query tuning, EXPLAIN ANALYZE, index optimization,
  missing index, database bottleneck, connection pool exhaustion, N+1 query, ORM performance,
  PostgreSQL performance, MySQL performance, query plan, sequential scan, index scan, table bloat,
  vacuum PostgreSQL, autovacuum, TOAST, partial index, covering index, composite index,
  database profiling, slow query log, pg_stat_statements, long-running query, lock contention,
  deadlock, database connection limit, pgBouncer, connection pooling, read replica, query cache,
  materialized view, database partitioning, table partition.
argument-hint: >
  Describe the database engine (PostgreSQL, MySQL, SQLite), the slow operation (e.g., "product
  search with filters takes 3s at p99"), and any ORM in use. Include the EXPLAIN ANALYZE output
  or slow query log excerpt if available.
---

# Database Performance Specialist

## When to Use

Invoke this skill when you need to:
- Diagnose and fix slow queries using EXPLAIN ANALYZE
- Design or improve indexes for a specific query or workload
- Fix N+1 query problems from ORMs
- Tune connection pool configuration to handle load
- Reduce table bloat and lock contention
- Decide when to use materialized views, partitioning, or read replicas

---

## Step 1 — Identify the Bottleneck

Before tuning, measure. Guess-driven optimization is waste.

**PostgreSQL — find slow queries:**
```sql
-- Enable pg_stat_statements extension (once)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries by total time
SELECT
    round((total_exec_time / 1000)::numeric, 2) AS total_seconds,
    calls,
    round((mean_exec_time)::numeric, 2) AS mean_ms,
    round((stddev_exec_time)::numeric, 2) AS stddev_ms,
    left(query, 120) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**MySQL — enable slow query log:**
```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;  -- log queries > 500ms
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
-- Then analyze with: pt-query-digest /var/log/mysql/slow.log
```

**Application-level tracking (Go — pgx example):**
```go
// Log queries taking > 100ms
db.AfterQuery(func(ctx context.Context, q *pgx.Query, err error) {
    if q.Elapsed > 100*time.Millisecond {
        slog.Warn("slow query",
            slog.Duration("duration", q.Elapsed),
            slog.String("sql", q.SQL),
        )
    }
})
```

Checklist:
- [ ] `pg_stat_statements` enabled — provides cumulative query stats without manual instrumentation
- [ ] Slow query threshold configured (500ms for APIs, 1s for batch) — log, don't guess
- [ ] Application logs include query duration and the query text (parameterized — no raw values)
- [ ] Monitoring shows DB query percentiles (p50, p95, p99) over time — spot regressions early

---

## Step 2 — Read and Act on EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT o.id, o.total, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

**Reading the output — key signals:**

| Signal | Meaning | Action |
|---|---|---|
| `Seq Scan` on large table | No usable index | Add index on the filtered column |
| `rows=10000` actual vs `rows=1` estimate | Stale statistics | `ANALYZE tablename` |
| `loops=N` is much larger than expected | N+1 in the query plan | Rewrite with JOIN or batch load |
| `Buffers: hit=0 read=50000` | High disk I/O — data not in cache | Tune `shared_buffers`; add index |
| `Hash Join` vs `Nested Loop` | Planner choice based on row estimates | If wrong, update statistics |
| `Sort Method: external merge` | Sort spilling to disk | Increase `work_mem` or add index |
| `Filter: (removed N rows)` | Index found rows but filter rejected most | Use partial index to filter at index level |

**pev2 / explain.dalibo.com** — paste EXPLAIN ANALYZE output into these tools for visual tree analysis.

Checklist:
- [ ] EXPLAIN ANALYZE run against production data volumes — not tiny dev databases
- [ ] Sequential scans on large tables identified and addressed with indexes
- [ ] Row estimate accuracy checked — large divergence means stale statistics; run `ANALYZE`
- [ ] Sort and hash operations not spilling to disk — increase `work_mem` if needed

---

## Step 3 — Index Design

Indexes are the highest-leverage database performance tool. Use them deliberately.

**Index type selection:**

| Index Type | Use Case |
|---|---|
| B-tree (default) | Equality, range queries, sorting (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`) |
| Hash | Equality only — rarely worth it over B-tree in PostgreSQL |
| GIN | Full-text search, JSONB containment, array membership |
| GiST | Geometric/spatial data, range types, text similarity |
| BRIN | Very large tables with naturally ordered data (timestamps, sequential IDs) |
| Partial | Filtered subset of rows (e.g., only `status = 'active'`) |
| Covering | Include non-key columns to enable index-only scans |

**Composite index key order — most selective column first:**
```sql
-- Query: WHERE status = 'pending' AND created_at > NOW() - INTERVAL '7 days'
-- status has 5 distinct values; created_at has millions — put status first
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

-- Index-only scan: include columns needed by the query in the index
CREATE INDEX idx_orders_status_created_covering
    ON orders (status, created_at DESC)
    INCLUDE (total, customer_id);
```

**Partial index — index only the rows you query:**
```sql
-- Only index pending orders — reduces index size by 90% if most orders are completed
CREATE INDEX idx_orders_pending ON orders (created_at DESC)
    WHERE status = 'pending';
```

**Create indexes concurrently — zero downtime:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```

**Find missing indexes (PostgreSQL):**
```sql
SELECT
    schemaname,
    tablename,
    seq_scan,
    idx_scan,
    n_live_tup,
    seq_scan - idx_scan AS table_scans_missed
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND n_live_tup > 10000
ORDER BY table_scans_missed DESC;
```

**Find unused indexes:**
```sql
SELECT indexrelid::regclass AS index_name, relid::regclass AS table_name, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND NOT indisprimary
ORDER BY pg_relation_size(indexrelid) DESC;
```

Checklist:
- [ ] Every foreign key has an index — JOINs and cascading deletes use it
- [ ] Composite index columns ordered by selectivity (most selective first)
- [ ] `CREATE INDEX CONCURRENTLY` used in production — never plain `CREATE INDEX` on a live table
- [ ] Unused indexes identified and dropped — they slow down writes for no read benefit
- [ ] Partial indexes used for queries that always filter on a low-cardinality condition

---

## Step 4 — Fix N+1 Query Problems

N+1 is the most common ORM performance problem: one query to get a list, then one query per item.

**Detecting N+1:**
```go
// ❌ N+1 — 1 query for orders + N queries for customers
orders, _ := db.Query("SELECT id, customer_id, total FROM orders WHERE status='pending'")
for _, o := range orders {
    customer, _ := db.QueryRow("SELECT email FROM customers WHERE id = $1", o.CustomerID)
    // produces 1 + N queries
}

// ✅ JOIN — 1 query total
rows, _ := db.Query(`
    SELECT o.id, o.total, c.email
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE o.status = 'pending'
`)
```

**Batch loading (when JOIN is not appropriate):**
```go
// Fetch all customer IDs in one query, then batch-load customers
customerIDs := extractIDs(orders)
customers, _ := db.Query(
    "SELECT id, email FROM customers WHERE id = ANY($1)",
    pq.Array(customerIDs),
)
customerMap := indexByID(customers)

for _, o := range orders {
    o.Customer = customerMap[o.CustomerID]
}
```

**ORM eager loading (GORM example):**
```go
// ❌ Lazy load — N+1
db.Find(&orders)
for _, o := range orders { _ = o.Customer.Email }  // triggers N queries

// ✅ Eager load — 2 queries total (orders + customers IN)
db.Preload("Customer").Find(&orders)
```

Checklist:
- [ ] ORM queries reviewed for N+1 — use query logging in development to detect them
- [ ] `Preload` / `Eager` / `Include` used for associated records fetched in a loop
- [ ] Batch loading (`WHERE id = ANY($1)`) used when JOIN is not semantically correct
- [ ] Max 2 queries per HTTP handler as a rule of thumb — anything more deserves review

---

## Step 5 — Connection Pool Tuning

Connection exhaustion causes requests to queue and timeout under load.

**Connection pool sizing formula:**
```
optimal_pool_size ≈ num_cores × 2 + effective_spindle_count
For 4 cores, SSD: 4 × 2 + 1 = ~9 connections per service instance
For 8 instances: 9 × 8 = 72 total connections → stay under PostgreSQL max_connections
```

**pgBouncer (recommended for PostgreSQL):**
```ini
# pgbouncer.ini
[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction      # transaction-level pooling — best efficiency
max_client_conn = 1000       # app connections in
default_pool_size = 20       # actual DB connections out
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
log_pooler_errors = 1
```

**Go — `pgxpool` configuration:**
```go
config, _ := pgxpool.ParseConfig(databaseURL)
config.MaxConns = 10
config.MinConns = 2
config.MaxConnLifetime = 30 * time.Minute
config.MaxConnIdleTime = 5 * time.Minute
config.HealthCheckPeriod = 1 * time.Minute
pool, _ := pgxpool.NewWithConfig(ctx, config)
```

**Metrics to monitor:**
- `pool_waiting_count` — requests waiting for a free connection (should be 0 at steady state)
- `pool_active_connections` — connections actively executing a query
- `pool_idle_connections` — connections open but idle (reduce MinConns if too high)

Checklist:
- [ ] pgBouncer or equivalent pooler in front of PostgreSQL — prevents connection limit saturation
- [ ] Connection pool sized to the DB `max_connections` minus overhead (replication, autovacuum)
- [ ] Pool metrics monitored — `waiting_count > 0` for sustained periods indicates under-sized pool
- [ ] Connection timeout configured — requests do not wait forever for a connection

---

## Step 6 — Maintenance and Bloat

**PostgreSQL VACUUM and bloat:**

Table bloat (dead tuples from UPDATEs and DELETEs) causes table scans to read more pages than necessary.

```sql
-- Find bloated tables
SELECT
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY dead_pct DESC;

-- Manual vacuum (use autovacuum for routine maintenance)
VACUUM ANALYZE orders;

-- Reclaim disk space (locks table briefly)
VACUUM FULL orders;  -- only for severely bloated tables during maintenance windows
```

**Autovacuum tuning for high-write tables:**
```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuum after 1% dead tuples (default: 20%)
    autovacuum_analyze_scale_factor = 0.005  -- analyze after 0.5% changes
);
```

**Materialized views for expensive aggregations:**
```sql
-- Slow query: aggregate orders by day for reporting
CREATE MATERIALIZED VIEW daily_order_summary AS
SELECT
    date_trunc('day', created_at) AS day,
    COUNT(*) AS order_count,
    SUM(total) AS revenue
FROM orders
GROUP BY 1;

CREATE INDEX ON daily_order_summary (day DESC);

-- Refresh on a schedule (every hour):
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_order_summary;
```

Checklist:
- [ ] Autovacuum tuned for high-write tables — default scale factor (20%) inadequate for busy tables
- [ ] Table bloat monitored — `dead_pct > 20%` for large tables should trigger investigation
- [ ] `REFRESH MATERIALIZED VIEW CONCURRENTLY` used — non-blocking refresh
- [ ] `VACUUM FULL` only during maintenance windows — it acquires an exclusive lock

---

## Output Report

### Critical
- No connection pool — every request opens a new connection; DB max_connections exceeded under load
- N+1 queries in critical hot paths — 100-request batch produces 10,100 DB queries
- Sequential scan on a multi-million-row table in the critical path — query takes seconds

### High
- Missing index on foreign key columns — every JOIN is a sequential scan
- Table bloat > 50% — all queries read 2× the necessary data from disk
- Stale statistics causing the query planner to choose wrong join/index strategy

### Medium
- Connection pool not sized correctly — pool exhaustion causes request queuing under moderate load
- `CREATE INDEX` used instead of `CREATE INDEX CONCURRENTLY` — table locked during index build in production
- Unused indexes not dropped — write performance degraded for zero read benefit

### Low
- `pg_stat_statements` not enabled — no visibility into cumulative query performance
- Autovacuum not tuned for high-write tables — dead tuple accumulation causes gradual slowdowns
- Materialized view not used for expensive reporting aggregations — OLAP queries impact OLTP latency

### Passed
- Slow queries identified via `pg_stat_statements`; EXPLAIN ANALYZE used to diagnose each
- Indexes designed with composite key order, partial filters, and INCLUDE columns where appropriate
- N+1 queries eliminated via JOIN or batch loading; ORM eager loading configured
- Connection pool sized correctly; pgBouncer deployed; pool metrics monitored with alerts
- Autovacuum tuned for high-write tables; table bloat monitored; materialized views refresh concurrently

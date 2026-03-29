---
name: zero-downtime-migrations
description: >
  Deployment-safe schema evolution without service downtime or locking.
  Use when: zero downtime migration, online schema change, non-blocking migration, schema evolution,
  backward compatible migration, forward compatible migration, expand contract pattern,
  blue green migration, rolling deployment migration, column rename zero downtime,
  add column zero downtime, type change zero downtime, pt-online-schema-change, gh-ost,
  pg_repack, online DDL MySQL, concurrent index, lock free DDL, hot schema change,
  live migration, schema compatibility, deployment safety.
argument-hint: >
  Describe the schema change and traffic constraints (e.g., "rename column `status` to `state` in orders table, service runs 24/7 with zero maintenance windows"),
  and specify the database engine (PostgreSQL, MySQL) and deployment model (rolling, blue/green, canary).
---

# Zero-Downtime Migrations Specialist

## When to Use

Invoke this skill when you need to:
- Apply schema changes to a production database without taking the service offline
- Design backward-compatible migrations for rolling deployments
- Use the expand-contract pattern to rename columns, tables, or change types
- Configure non-locking DDL operations (concurrent indexes, `NOT VALID` constraints)
- Select and use online schema change tools (gh-ost, pt-osc, pg_repack)
- Sequence multi-phase migrations across multiple deployments

---

## Step 1 — Understand Locking Semantics

Schema changes acquire locks that block reads and/or writes. Know your engine:

**PostgreSQL:**

| Operation | Lock Level | Safe? |
|---|---|---|
| `ADD COLUMN ... NULL` | AccessShareLock (brief) | ✅ Safe |
| `ADD COLUMN ... NOT NULL DEFAULT <literal>` (PG 11+) | Brief metadata lock | ✅ Safe (PG 11+) |
| `ADD COLUMN ... NOT NULL DEFAULT <volatile>` | AccessExclusiveLock | ❌ Full table lock |
| `CREATE INDEX` | ShareLock (blocks writes) | ❌ Risky on large tables |
| `CREATE INDEX CONCURRENTLY` | No write lock | ✅ Safe |
| `DROP COLUMN` | AccessExclusiveLock | ❌ Brief but exclusive |
| `ALTER COLUMN TYPE` | AccessExclusiveLock | ❌ Full table rewrite |
| `ADD CONSTRAINT NOT VALID` | Brief | ✅ Safe |
| `VALIDATE CONSTRAINT` | ShareUpdateExclusiveLock | ✅ Safe (allows reads+writes) |

**MySQL / MariaDB:**

| Operation | Online DDL? | Tool Needed? |
|---|---|---|
| `ADD COLUMN` (nullable, after existing) | ✅ MySQL 8 Instant DDL | No |
| `ADD INDEX` | ✅ Online | No |
| `DROP COLUMN` | ✅ Online (InnoDB) | Rarely |
| `MODIFY COLUMN` (type change) | ❌ Table rebuild | gh-ost / pt-osc |
| `RENAME COLUMN` | ✅ MySQL 8+ | No |

Checklist:
- [ ] Locking behavior verified for every DDL operation against the target DB version
- [ ] `LOCK=NONE` used in MySQL DDL to fail fast if the operation would block
- [ ] `CREATE INDEX CONCURRENTLY` always used in PostgreSQL for large tables
- [ ] Lock timeout set low in application migration runner to fail fast

---

## Step 2 — The Expand-Contract Pattern

The safest model for renaming, type changing, or removing schema elements across rolling deployments.

**Phases:**

```
Expand  →  Migrate  →  Contract
(add)      (backfill)   (remove old)
```

**Example: Rename column `user_name` → `full_name`**

**Phase 1 — Expand (deploy migration, then deploy code v2):**
```sql
ALTER TABLE users ADD COLUMN full_name TEXT;
```
- Code v2 writes to BOTH `user_name` AND `full_name`
- Code v2 reads from `user_name` (old name, still primary)

**Phase 2 — Migrate (backfill):**
```sql
UPDATE users
SET full_name = user_name
WHERE full_name IS NULL;
-- Run in batches for large tables
```

**Phase 3 — Switch reads (deploy code v3):**
- Code v3 reads from `full_name`, writes to BOTH

**Phase 4 — Contract (deploy code v4, then deploy migration):**
- Code v4 writes to `full_name` only
```sql
ALTER TABLE users DROP COLUMN user_name;
```

Checklist:
- [ ] Expand phase deployed and stable before backfill begins
- [ ] Backfill done in batches (1000–10000 rows per batch) with `WHERE` filter and `LIMIT`
- [ ] Both columns kept in sync during the migration phase
- [ ] Drop phase executed only after all code versions reading from the old column are retired
- [ ] Feature flag used to control which column the application reads from (optional but safe)

---

## Step 3 — Safe NOT NULL Column Addition

**PostgreSQL 11+ with literal default:**
```sql
-- Safe: PostgreSQL 11+ stores the default as metadata, no table rewrite
ALTER TABLE orders ADD COLUMN risk_score INT NOT NULL DEFAULT 0;
```

**For volatile defaults or pre-PG11:**
```sql
-- Step 1: Add nullable
ALTER TABLE orders ADD COLUMN risk_score INT NULL;

-- Step 2: Backfill in batches
UPDATE orders SET risk_score = 0 WHERE risk_score IS NULL AND id BETWEEN 1 AND 10000;
-- ... repeat

-- Step 3: Add NOT NULL constraint (validates existing rows)
-- Use NOT VALID to skip row scan, then validate separately:
ALTER TABLE orders ADD CONSTRAINT chk_risk_score_not_null CHECK (risk_score IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_risk_score_not_null;

-- Step 4: Convert to true NOT NULL (brief lock)
ALTER TABLE orders ALTER COLUMN risk_score SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT chk_risk_score_not_null;
```

Checklist:
- [ ] PG version confirmed before choosing literal default path
- [ ] Backfill runs in small batches with sleep between iterations to avoid I/O saturation
- [ ] `NOT VALID` + `VALIDATE CONSTRAINT` pattern used to defer row scanning

---

## Step 4 — Online Schema Change Tools

Use these tools for MySQL or when PostgreSQL native DDL still causes unacceptable lock duration:

**gh-ost (GitHub's online schema change for MySQL):**
```bash
gh-ost \
  --user="root" --password="..." \
  --host="db-primary" \
  --database="myapp" \
  --table="orders" \
  --alter="MODIFY COLUMN status VARCHAR(32) NOT NULL" \
  --execute
```
- Copies table to a ghost table, applies binlog changes, cuts over atomically
- Supports pause, resume, and non-destructive cutover

**pt-online-schema-change (Percona Toolkit):**
- Similar to gh-ost; trigger-based instead of binlog-based
- Higher write overhead; prefer gh-ost for high-write tables

**pg_repack (PostgreSQL):**
```bash
pg_repack -t orders --no-order
```
- Rewrites a table or index without holding a long lock
- Use when `VACUUM` cannot reclaim dead tuples at the needed pace

Checklist:
- [ ] Tool chosen based on replication type (gh-ost requires row-based replication)
- [ ] Dry run executed before live run
- [ ] Disk space available for the ghost table (2x original table size)
- [ ] Replication lag monitored during the migration — tool pauses if lag exceeds threshold
- [ ] Cutover window planned (brief lock at final cutover step)

---

## Step 5 — Index Operations

**PostgreSQL:**
```sql
-- Always concurrent for large tables:
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);

-- Drop concurrently (no write lock):
DROP INDEX CONCURRENTLY idx_orders_old;
```
- `CONCURRENTLY` takes longer (two table scans) but does not block writes
- Cannot run inside a transaction — must run standalone

**MySQL:**
```sql
-- Online index creation (InnoDB default):
ALTER TABLE orders ADD INDEX idx_customer_id (customer_id), ALGORITHM=INPLACE, LOCK=NONE;
```

Checklist:
- [ ] `CONCURRENTLY` used for all index operations on tables > 100k rows in PostgreSQL
- [ ] Index creation monitored for duplicate creation (PG silently allows concurrent index creation on same columns)
- [ ] Failed concurrent index left as `INVALID` — must be dropped and recreated
- [ ] `pg_stat_progress_create_index` queried to monitor progress

---

## Step 6 — Deployment Sequencing

For rolling deployments (Kubernetes, ECS, etc.):

```
Phase 1: Deploy migration (schema change only — no code change required yet)
Phase 2: Deploy app v2 (reads both old and new, writes to both)
Phase 3: Verify data consistency
Phase 4: Deploy migration (drop old column/table) — after all app v1 pods retired
Phase 5: Deploy app v3 (reads/writes new only)
```

Checklist:
- [ ] Application code is backward compatible with the schema **before** the migration is applied
- [ ] Application code is forward compatible with the schema **after** the migration is applied
- [ ] No migration that requires simultaneous code+schema atomicity — split into phases
- [ ] Canary deployment used to gradually shift traffic during phase transitions

---

## Output Report

### Critical
- `ALTER TABLE ... ADD COLUMN NOT NULL` without PostgreSQL 11+ literal default — full table lock on a busy table
- `CREATE INDEX` without `CONCURRENTLY` on a multi-GB table — extended write lockout
- Renaming a column in a single migration without expand-contract — application error on rolling deploy

### High
- Backfill executed as a single `UPDATE` without batching — long-running transaction holds row locks
- gh-ost / pt-osc not used for MySQL column type changes — full table lock
- Migration applied simultaneously with code deployment — no rollback window

### Medium
- Lock timeout not set in migration runner — migration hangs indefinitely waiting for lock
- `NOT VALID` constraint not used for FK/check addition on large tables — unnecessary row scan lock
- Deployment phases not documented — engineers unsure which code version can be deployed at each phase

### Low
- pg_stat_progress_create_index not monitored during long index creation
- No feature flag to control column switch — cannot revert reads without code deployment
- Replication lag not monitored during online schema change

### Passed
- Expand-contract pattern applied for all column renames, type changes, and removals
- `CREATE INDEX CONCURRENTLY` used for all index operations on large tables
- Backfill runs in batches with I/O-friendly sleep intervals
- Each deployment phase is backward and forward compatible with adjacent phases
- Lock acquisition timeout configured in the migration runner

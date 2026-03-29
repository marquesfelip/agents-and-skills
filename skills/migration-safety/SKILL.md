---
name: migration-safety
description: >
  Safe database migrations and rollback strategies.
  Use when: database migration, migration safety, schema migration, rollback migration, migration strategy,
  Flyway, Liquibase, Alembic, golang-migrate, db-migrate, migration versioning, migration review,
  migration testing, irreversible migration, destructive migration, data migration, column rename,
  table rename, column drop, table drop, migration failure, migration rollback plan, migration CI,
  migration validation, migration order, migration conflict.
argument-hint: >
  Describe the migration you need to execute (e.g., "rename column `user_name` to `full_name` in users table")
  and specify the database engine, migration tool, and whether the service needs to stay online during the migration.
---

# Migration Safety Specialist

## When to Use

Invoke this skill when you need to:
- Review a migration script for safety before applying to production
- Design a rollback plan for a schema change
- Classify a migration as safe, risky, or destructive
- Sequence multi-step migrations for compatibility across deployments
- Set up a migration pipeline with validation and CI integration
- Handle irreversible migrations safely (column drop, table drop, data purge)

---

## Step 1 — Classify the Migration

Every migration falls into a risk category:

| Category | Operations | Risk |
|---|---|---|
| **Safe** | Add nullable column, Add new table, Add index (concurrently), Add non-enforced FK | Low — backward and forward compatible |
| **Risky** | Add NOT NULL column without default, Rename column/table, Change column type, Add enforced FK, Add unique constraint | Medium — may break running application code |
| **Destructive** | Drop column, Drop table, Truncate table, Remove constraint | High — irreversible data loss |

Checklist:
- [ ] Each migration script contains exactly **one logical change**
- [ ] Risk category assigned before peer review
- [ ] Destructive migrations require explicit approval from a second engineer
- [ ] Destructive operations separated into their own migration file, explicitly delayed

---

## Step 2 — Design the Rollback Plan

Every migration must have a corresponding rollback:

| Forward Migration | Rollback |
|---|---|
| `ADD COLUMN foo` | `DROP COLUMN foo` |
| `CREATE TABLE` | `DROP TABLE` |
| `CREATE INDEX` | `DROP INDEX` |
| `ADD CONSTRAINT` | `DROP CONSTRAINT` |
| `ALTER COLUMN TYPE` | Restore original type (may require data conversion) |
| `DROP COLUMN` | **Cannot be automatically rolled back** — requires restore from backup |
| `DROP TABLE` | **Cannot be automatically rolled back** — requires restore from backup |

Checklist:
- [ ] Rollback script written **before** the forward migration is applied
- [ ] Rollback script tested in a staging environment
- [ ] For destructive operations: confirm backup taken and verified before running forward migration
- [ ] Time-to-rollback estimated — if > 5 min, escalate to zero-downtime migration strategy

---

## Step 3 — Safe Migration Patterns by Operation Type

**Adding a NOT NULL column:**
```sql
-- Step 1 (safe): Add nullable
ALTER TABLE orders ADD COLUMN approved_at TIMESTAMPTZ NULL;

-- Step 2 (data backfill): Populate existing rows
UPDATE orders SET approved_at = created_at WHERE approved_at IS NULL;

-- Step 3 (safe if backfill complete): Add NOT NULL constraint
ALTER TABLE orders ALTER COLUMN approved_at SET NOT NULL;
```

**Renaming a column (zero-downtime):**
```sql
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN full_name TEXT;

-- Step 2: Backfill
UPDATE users SET full_name = user_name;

-- Step 3: Deploy new code reading from full_name
-- Step 4 (next release): Drop old column user_name
```

**Changing column type:**
```sql
-- Step 1: Add a new column with the target type
-- Step 2: Backfill and keep in sync via trigger
-- Step 3: Deploy code to write to both columns
-- Step 4: Deploy code to read from new column
-- Step 5: Drop old column
```

**Adding a FK constraint:**
```sql
-- Add NOT VALID to skip validation of existing rows (fast, non-locking)
ALTER TABLE orders ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id)
  NOT VALID;

-- Validate in a separate migration (non-locking in PostgreSQL)
ALTER TABLE orders VALIDATE CONSTRAINT fk_customer;
```

Checklist:
- [ ] Multi-step patterns used for: NOT NULL columns, column renames, type changes, FK additions
- [ ] Large table backfills done in batches (not a single `UPDATE` with no `WHERE` limit)
- [ ] `NOT VALID` + `VALIDATE CONSTRAINT` pattern used for FK additions on large tables

---

## Step 4 — Migration Tooling and Versioning

**Recommended tools:**

| Tool | Ecosystem | Format |
|---|---|---|
| **Flyway** | JVM / multi-language | SQL files |
| **Liquibase** | JVM / multi-language | XML, YAML, SQL |
| **Alembic** | Python / SQLAlchemy | Python scripts |
| **golang-migrate** | Go | SQL files |
| **Prisma Migrate** | Node.js / TypeScript | Auto-generated SQL |
| **Rails ActiveRecord** | Ruby on Rails | Ruby DSL |

**Versioning convention:**
```
V001__create_users.sql
V002__add_orders_table.sql
V003__add_column_orders_approved_at.sql
```

Checklist:
- [ ] Migrations are numbered sequentially and never edited after merge
- [ ] Each file has a descriptive name — not `V010__fix.sql`
- [ ] Migration checksums validated on startup (Flyway/Liquibase do this automatically)
- [ ] Migrations committed to version control alongside the application code
- [ ] Branching conflicts resolved by renumbering — never by sharing a version number

---

## Step 5 — Migration Testing Pipeline

```yaml
migration-ci:
  steps:
    - name: Apply migrations to clean DB
      run: flyway migrate
    - name: Run migration idempotency check
      run: flyway migrate  # Second run must be no-op
    - name: Validate schema matches expected
      run: flyway validate
    - name: Run application tests against migrated schema
      run: go test ./...
    - name: Apply rollback
      run: flyway undo  # Or manual SQL
    - name: Verify rollback is clean
      run: flyway validate
```

Checklist:
- [ ] Migrations run against a clean DB in CI on every commit
- [ ] Application tests run **after** migration — confirm app code is compatible with new schema
- [ ] Rollback tested in CI — not just in theory
- [ ] Migration applied to staging 24–48h before production — catch environment-specific issues
- [ ] Migration execution time measured in staging — flag if > 30s (likely needs zero-downtime approach)

---

## Step 6 — Production Migration Checklist

Before applying any migration to production:

- [ ] Migration reviewed and approved by at least one other engineer
- [ ] Migration successfully applied and rolled back in staging
- [ ] Execution time on staging measured and acceptable
- [ ] Backup of the affected table(s) taken and verified
- [ ] Rollback plan documented and ready to execute (not written during the incident)
- [ ] Maintenance window scheduled or zero-downtime approach confirmed
- [ ] Monitoring alerts active for error rate, query latency, and connection pool usage
- [ ] Migration applied during low-traffic period (if not zero-downtime)
- [ ] Post-migration smoke test executed
- [ ] Rollback triggered if smoke test fails within 5 minutes of migration

---

## Output Report

### Critical
- Destructive migration (`DROP COLUMN`, `DROP TABLE`) with no backup taken
- Migration that locks a high-traffic table without a zero-downtime alternative
- Rollback script not written or not tested before forward migration applied to production
- Migration edits a previously applied version file — checksum mismatch will break startup

### High
- `NOT NULL` column added in a single step without backfill — application code breaks on deploy
- Large `UPDATE` without batch limits runs as a single transaction — locks table for minutes
- FK constraint added with validation on a large table — long lock held

### Medium
- Migration not tested in staging before production — environment-specific failures possible
- Execution time not measured before production apply — surprise lock duration
- No rollback plan documented — recovery improvised during incident

### Low
- Migration file names not descriptive — hard to navigate migration history
- Multiple unrelated changes in a single migration file — hard to roll back selectively
- Migration not committed alongside the application code change that requires it

### Passed
- Every migration classified by risk before review
- Rollback script written and tested in staging before production apply
- Multi-step patterns used for all risky operations (NOT NULL, rename, type change)
- Migrations run in CI against clean DB and tested for idempotency
- Production migration checklist completed before every apply

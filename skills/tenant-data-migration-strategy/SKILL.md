---
name: tenant-data-migration-strategy
description: >
  Safe tenant-specific data migrations in multi-tenant SaaS systems.
  Use when: tenant data migration, per-tenant migration, phased migration, tenant migration strategy,
  tenant migration rollout, data migration SaaS, migration per tenant, tenant migration rollback,
  tenant-specific schema migration, large data migration, tenant batch migration, migration safety,
  zero-downtime tenant migration, tenant migration audit, migration tracking, migration version,
  migration kill switch, migration pause, tenant migration window, data backfill tenant,
  migration dry run, migration verification, tenant migration progress, migration monitoring.
argument-hint: >
  Describe the migration (schema change, data transformation, backfill), whether it affects all
  tenants or a subset, the expected data volume per tenant, and the risk tolerance (zero-downtime
  required, rollback SLA, whether tenants can notice the migration in progress).
---

# Tenant Data Migration Strategy Specialist

## When to Use

Invoke this skill when you need to:
- Migrate data tenant-by-tenant with phased rollout and verification at each stage
- Track migration progress and status per tenant
- Define rollback for a per-tenant migration without affecting other tenants
- Implement a kill switch to halt a migration run instantly
- Run migrations during low-traffic windows per tenant
- Distinguish between global schema migrations (affect all tenants) and per-tenant data migrations

---

## Step 1 — Classify the Migration

**Two types of migrations in a multi-tenant SaaS:**

| Type | Scope | Risk | Approach |
|---|---|---|---|
| **Global schema migration** | All tenants; structural | High if table is large or locked | Zero-downtime DDL: ADD COLUMN CONCURRENTLY, expand-contract |
| **Per-tenant data migration** | One tenant at a time | Medium; isolatable | Phased rollout: N tenants per batch; pause and verify |

**Decision tree:**
```
Is this a DDL change (ADD COLUMN, CREATE INDEX, DROP COLUMN)?
  → Yes: Use zero-downtime DDL (see zero-downtime-migrations skill)
  → No: continue

Does the data transformation depend on per-tenant business logic?
  → Yes: per-tenant migration with tenant-scoped logic
  → No: global data migration (simpler; can be a single UPDATE with batching)

Does each tenant's migration need to be independently reversible?
  → Yes: per-tenant migration with per-tenant rollback
  → No: global migration with a single rollback
```

Checklist:
- [ ] Migration classified as global schema or per-tenant data migration
- [ ] DDL changes separated from data migrations — run DDL first; data migration after
- [ ] Per-tenant migrations designed to be independent — one failure does not block others
- [ ] Migration scope estimated: rows per tenant, total rows, expected duration per tenant

---

## Step 2 — Per-Tenant Migration Tracking

**Migration tracking table:**
```sql
CREATE TABLE tenant_migration_runs (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    migration_key   TEXT        NOT NULL,   -- 'normalize_order_currency_2026_01'
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    status          TEXT        NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','in_progress','completed','failed','skipped','rolled_back')),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    rows_processed  BIGINT      NOT NULL DEFAULT 0,
    rows_affected   BIGINT      NOT NULL DEFAULT 0,
    error_message   TEXT,
    checksum        TEXT,       -- hash of output state for verification
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (migration_key, tenant_id)
);

CREATE INDEX idx_tmr_migration_status ON tenant_migration_runs(migration_key, status);
CREATE INDEX idx_tmr_tenant ON tenant_migration_runs(tenant_id, migration_key);
```

**Initialize migration run for all tenants:**
```sql
-- Create pending rows for all active tenants at the start of a migration
INSERT INTO tenant_migration_runs (migration_key, tenant_id)
SELECT 'normalize_order_currency_2026_01', id
FROM tenants
WHERE status = 'active'
ON CONFLICT (migration_key, tenant_id) DO NOTHING;
```

Checklist:
- [ ] Tracking table uses `UNIQUE (migration_key, tenant_id)` — idempotent initialization
- [ ] Status captures all lifecycle states including `rolled_back` and `skipped`
- [ ] `rows_processed` and `rows_affected` tracked — enables monitoring and audit
- [ ] `checksum` field for pre/post state verification — detects silent corruption

---

## Step 3 — Phased Migration Runner

**Run the migration in batches, validating between stages:**
```go
type TenantMigration struct {
    Key         string
    Description string
    BatchSize   int           // tenants per batch (e.g., 10)
    BatchDelay  time.Duration // pause between batches (e.g., 5 minutes)
    Migrate     func(ctx context.Context, tenantID string) (rowsAffected int64, err error)
    Verify      func(ctx context.Context, tenantID string) error
    Rollback    func(ctx context.Context, tenantID string) error
}

type MigrationRunner struct {
    db          *pgxpool.Pool
    killSwitch  *killSwitchService
    metrics     MigrationMetrics
}

func (r *MigrationRunner) Run(ctx context.Context, m TenantMigration) error {
    pending, err := r.pendingTenants(ctx, m.Key)
    if err != nil {
        return err
    }

    for i := 0; i < len(pending); i += m.BatchSize {
        // Kill switch check before each batch
        if r.killSwitch.IsHalted(m.Key) {
            slog.Warn("migration halted by kill switch",
                slog.String("migration", m.Key),
                slog.Int("completed", i),
                slog.Int("remaining", len(pending)-i),
            )
            return ErrMigrationHalted
        }

        batch := pending[i:min(i+m.BatchSize, len(pending))]

        for _, tenantID := range batch {
            if err := r.migrateTenant(ctx, m, tenantID); err != nil {
                slog.Error("tenant migration failed",
                    slog.String("migration", m.Key),
                    slog.String("tenant_id", tenantID),
                    slog.String("err", err.Error()),
                )
                // Continue with next tenant — failure is isolated
                r.markFailed(ctx, m.Key, tenantID, err.Error())
                continue
            }
        }

        r.metrics.BatchCompleted(m.Key, i+len(batch), len(pending))

        // Pause between batches — watch for errors before continuing
        if i+m.BatchSize < len(pending) {
            time.Sleep(m.BatchDelay)
        }
    }

    return nil
}

func (r *MigrationRunner) migrateTenant(ctx context.Context, m TenantMigration, tenantID string) error {
    r.markInProgress(ctx, m.Key, tenantID)

    rows, err := m.Migrate(ctx, tenantID)
    if err != nil {
        return err
    }

    // Verify after migration
    if m.Verify != nil {
        if err := m.Verify(ctx, tenantID); err != nil {
            // Verification failed — roll back this tenant
            m.Rollback(ctx, tenantID)
            r.markRolledBack(ctx, m.Key, tenantID)
            return fmt.Errorf("verification failed: %w", err)
        }
    }

    r.markCompleted(ctx, m.Key, tenantID, rows)
    return nil
}
```

Checklist:
- [ ] Kill switch checked before every batch — migration can be halted within one batch's delay time
- [ ] One tenant's failure does not block the rest — continue with next tenant; log the failure
- [ ] Verification function runs after migration per tenant — silent corruption detected
- [ ] Batch size and delay configurable — adjust based on observed DB load during run
- [ ] Progress reported between batches — operator knows how far along the migration is

---

## Step 4 — Kill Switch and Pause

**Kill switch — halt an in-flight migration urgently:**
```go
type killSwitchService struct {
    db *pgxpool.Pool
}

// Halts the migration before the next batch — takes effect within BatchDelay seconds
func (k *killSwitchService) Halt(ctx context.Context, migrationKey string) error {
    _, err := k.db.Exec(ctx, `
        INSERT INTO migration_kill_switches (migration_key, halted_at, reason)
        VALUES ($1, NOW(), 'operator halt')
        ON CONFLICT (migration_key) DO UPDATE SET halted_at = NOW()`, migrationKey)
    return err
}

func (k *killSwitchService) IsHalted(migrationKey string) bool {
    // Cached check — does not add DB query per tenant
    return k.cache.Get(migrationKey) != nil
}

func (k *killSwitchService) Resume(ctx context.Context, migrationKey string) error {
    _, err := k.db.Exec(ctx,
        `DELETE FROM migration_kill_switches WHERE migration_key = $1`, migrationKey)
    k.cache.Delete(migrationKey)
    return err
}
```

```sql
CREATE TABLE migration_kill_switches (
    migration_key TEXT PRIMARY KEY,
    halted_at     TIMESTAMPTZ NOT NULL,
    reason        TEXT
);
```

Checklist:
- [ ] Kill switch can halt the migration without code deployment — API or database toggle
- [ ] Kill switch check is cached — does not add a DB query per tenant in the hot path
- [ ] After halt: migration state is preserved — resumption starts from where it stopped, not from scratch
- [ ] Resumption script available — run `WHERE status = 'pending' OR status = 'failed'`

---

## Step 5 — Per-Tenant Rollback

```go
func (r *MigrationRunner) RollbackTenant(ctx context.Context, m TenantMigration, tenantID string) error {
    run, err := r.getMigrationRun(ctx, m.Key, tenantID)
    if err != nil || run.Status == "pending" {
        return nil // nothing to roll back
    }

    if err := m.Rollback(ctx, tenantID); err != nil {
        return fmt.Errorf("rollback failed for tenant %s: %w", tenantID, err)
    }

    r.markRolledBack(ctx, m.Key, tenantID)
    return nil
}

// Roll back only failed tenants — others unaffected
func (r *MigrationRunner) RollbackFailed(ctx context.Context, migrationKey string, m TenantMigration) error {
    failed, _ := r.failedTenants(ctx, migrationKey)
    for _, tenantID := range failed {
        r.RollbackTenant(ctx, m, tenantID)
    }
    return nil
}
```

**Dry run support:**
```go
// Run the migration logic without committing — useful for pre-migration impact analysis
func (r *MigrationRunner) DryRun(ctx context.Context, m TenantMigration, tenantID string) (rowsAffected int64, err error) {
    tx, err := r.db.Begin(ctx)
    if err != nil {
        return 0, err
    }
    defer tx.Rollback(ctx) // always rollback — this is a dry run

    rows, err := m.Migrate(ctx, tenantID) // runs in a transaction that is rolled back
    return rows, err
}
```

Checklist:
- [ ] Rollback function defined for every per-tenant migration — not "rollback by hand"
- [ ] Rollback tested in staging before production rollout — not first exercised during an incident
- [ ] Per-tenant rollback does not affect completed tenants — only failed or explicitly selected tenants
- [ ] Dry run mode available — estimate impact and verify logic before committing

---

## Output Report

### Critical
- No per-tenant progress tracking — impossible to resume after a failure; must re-run from scratch and risk double-migration
- No kill switch — a buggy migration cannot be halted without a code deployment or killing the process

### High
- One tenant's failure aborts the entire migration — all remaining tenants blocked until failure is manually resolved
- No verification step after migration — silent data corruption goes undetected until a customer reports it
- Rollback not defined before running — during an incident, rollback must be improvised under pressure

### Medium
- Batch size and delay hardcoded — cannot tune without a code deployment when DB load is higher than expected
- Migration re-runs apply to completed tenants — idempotency failure; completed tenants re-migrated
- No dry run mode — impact not estimable before committing; surprise row counts discovered mid-run

### Low
- No monitoring dashboard for migration progress — operator must query the tracking table manually
- `rows_affected` not tracked — cannot determine if migration had the expected impact
- Tenant migration window not considered — some tenants receive migration during peak business hours

### Passed
- Tracking table with `UNIQUE (migration_key, tenant_id)`; all statuses captured including rolled_back
- Kill switch checked before every batch; cached check; halt takes effect within one batch delay
- One tenant failure isolated; remaining tenants continue; failed tenants recorded for retry/rollback
- Verification function validates per-tenant output; rollback function defined and tested pre-production
- Dry run mode available; batch size and delay configurable; progress metrics emitted per batch

---
name: single-database-tenant-isolation
description: >
  Tenant isolation strategies for SaaS products using a single shared database.
  Use when: single database multi-tenancy, shared database isolation, tenant_id column,
  row-level isolation, multi-tenant schema, tenant data separation, shared table multi-tenant,
  ORM global scope, ORM default scope, tenant filtering, mandatory tenant filter,
  missing tenant_id, tenant isolation review, cross-tenant query, tenant data leakage,
  tenant_id index, shared DB tenancy, pool model multi-tenant, tenant scoping,
  tenant column enforcement, tenant FK constraint, tenant composite index, tenant query audit.
argument-hint: >
  Describe your current data model (tables, ORM framework), whether you are starting fresh or
  retrofitting an existing single-tenant application, and where you suspect tenant isolation
  may be missing or incomplete. Include the ORM or query builder you use.
---

# Single-Database Tenant Isolation Specialist

## When to Use

Invoke this skill when you need to:
- Add `tenant_id` columns to every tenant-scoped table in a shared database
- Enforce that every query against tenant tables includes a `tenant_id` filter
- Configure ORM global/default scopes for automatic tenant filtering
- Review an existing data model for missing tenant isolation
- Design composite indexes that correctly include `tenant_id` as the leading column
- Write tests that prove cross-tenant isolation is enforced

---

## Step 1 — Audit the Data Model

**Categorize every table:**

| Category | Tenant-Scoped? | `tenant_id` Required? | Example |
|---|---|---|---|
| Tenant-owned data | Yes | Yes | `orders`, `invoices`, `projects` |
| User data (cross-tenant users) | Partial | Via membership table | `users` (global), `team_memberships` (tenant-scoped) |
| Platform/system tables | No | No | `plans`, `countries`, `currencies` |
| Audit log | Partially | Yes (nullable for platform) | `audit_log` |
| Billing | Yes | Yes | `subscriptions`, `billing_events` |

```sql
-- Quick audit: find tenant-data tables missing tenant_id
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE'
  AND table_name NOT IN ('tenants', 'plans', 'schema_migrations') -- exempt tables
  AND table_name NOT IN (
      SELECT table_name
      FROM information_schema.columns
      WHERE column_name = 'tenant_id'
  );
```

Checklist:
- [ ] Every domain table classified as tenant-scoped, partially scoped, or platform-level
- [ ] Audit query run — no tenant-scoped table missing `tenant_id`
- [ ] Exempt tables (plans, countries, feature flags) explicitly listed — not accidents
- [ ] `users` table itself is NOT tenant-scoped (users can belong to multiple tenants) — memberships table is

---

## Step 2 — Add `tenant_id` Correctly

**Adding `tenant_id` to an existing table (zero-downtime migration pattern):**
```sql
-- Phase 1: Add nullable column (non-blocking)
ALTER TABLE orders ADD COLUMN tenant_id UUID REFERENCES tenants(id);

-- Phase 2: Backfill (run in batches)
UPDATE orders SET tenant_id = (
    SELECT tenant_id FROM memberships WHERE user_id = orders.created_by LIMIT 1
)
WHERE tenant_id IS NULL;

-- Phase 3: Add NOT NULL constraint (after backfill verified)
ALTER TABLE orders ALTER COLUMN tenant_id SET NOT NULL;

-- Phase 4: Add index
CREATE INDEX CONCURRENTLY idx_orders_tenant_id ON orders(tenant_id);
```

**New table — include `tenant_id` from the start:**
```sql
CREATE TABLE projects (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        TEXT        NOT NULL,
    status      TEXT        NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by  UUID        REFERENCES users(id)
);

-- tenant_id is the FIRST column in every index used for tenant-scoped queries
CREATE INDEX idx_projects_tenant         ON projects(tenant_id);
CREATE INDEX idx_projects_tenant_status  ON projects(tenant_id, status);
CREATE INDEX idx_projects_tenant_created ON projects(tenant_id, created_at DESC);
```

**Why `tenant_id` must lead composite indexes:**
```sql
-- GOOD: tenant_id first — index used for both tenant-scoped and multi-column queries
CREATE INDEX idx_orders_tenant_status ON orders(tenant_id, status, created_at DESC);

-- BAD: status first — cannot use this index for tenant-scoped queries without a status filter
CREATE INDEX idx_orders_status ON orders(status, tenant_id);
```

Checklist:
- [ ] `tenant_id` added as `NOT NULL` + FK to `tenants(id)`
- [ ] `ON DELETE CASCADE` or `ON DELETE RESTRICT` — defined for every tenant FK (no orphaned rows)
- [ ] Single-column index on `tenant_id` exists on every tenant-scoped table
- [ ] `tenant_id` is the leading column in all composite indexes on tenant-scoped tables
- [ ] `CREATE INDEX CONCURRENTLY` used for retrofitting — avoids locking

---

## Step 3 — ORM Global Scopes

**GORM global scope:**
```go
type TenantScope struct {
    TenantID string
}

func (s TenantScope) Apply(db *gorm.DB) *gorm.DB {
    return db.Where("tenant_id = ?", s.TenantID)
}

// In repository — inject tenant_id at construction time, not call time
type OrderRepository struct {
    db *gorm.DB
}

func NewOrderRepository(db *gorm.DB, tenantID string) *OrderRepository {
    return &OrderRepository{
        db: db.Scopes(TenantScope{TenantID: tenantID}.Apply),
    }
}

func (r *OrderRepository) FindByStatus(ctx context.Context, status string) ([]*Order, error) {
    var orders []*Order
    // tenant_id filter is applied automatically by the scope
    err := r.db.WithContext(ctx).Where("status = ?", status).Find(&orders).Error
    return orders, err
}
```

**SQLx/pgx — tenant context in repository:**
```go
type OrderRepository struct {
    db       *pgxpool.Pool
    tenantID string  // injected; not retrieved from ctx per-query
}

func (r *OrderRepository) ListByStatus(ctx context.Context, status string) ([]*Order, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, tenant_id, status, total
         FROM orders
         WHERE tenant_id = $1 AND status = $2
         ORDER BY created_at DESC`,
        r.tenantID, status,  // tenant_id always explicit
    )
    // ...
}
```

**Detecting missing `tenant_id` in queries — integration test:**
```go
func TestOrderRepository_CannotAccessCrossTeant(t *testing.T) {
    tenant1 := createTenant(t, "acme")
    tenant2 := createTenant(t, "globex")
    order   := createOrder(t, tenant1.ID)

    // Repository scoped to tenant2 must not find tenant1's orders
    repo := NewOrderRepository(db, tenant2.ID)
    result, err := repo.Get(ctx, order.ID)

    assert.Nil(t, result, "should not find cross-tenant order")
    assert.ErrorIs(t, err, ErrNotFound)
}
```

Checklist:
- [ ] ORM scope or repository-level `tenant_id` injection applied to all tenant-scoped repositories
- [ ] No hand-written query omits `WHERE tenant_id = $N`
- [ ] Integration tests explicitly verify that cross-tenant row access returns not-found, not data
- [ ] Raw SQL queries audited for missing tenant filter — search codebase for queries without `tenant_id`

---

## Step 4 — Safe Escapes (Admin / Platform Operations)

Platform code sometimes needs to query across all tenants (cron jobs, billing reconciliation, admin UI).

```go
// Explicitly named "unscoped" repository — safe cross-tenant access
type GlobalOrderRepository struct {
    db *pgxpool.Pool
}

// This is intentionally not tenant-scoped — only use in admin/system contexts
func (r *GlobalOrderRepository) FindExpiredOrders(ctx context.Context) ([]*Order, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, tenant_id FROM orders
         WHERE status = 'pending' AND created_at < NOW() - INTERVAL '7 days'`)
    // ...
}
```

**Naming conventions to prevent confusion:**
- `OrderRepository` — always tenant-scoped; panics if `tenantID` is empty
- `GlobalOrderRepository` — explicitly unscoped; requires admin role in calling context
- Never export the unscoped version via the same interface — separate types for tenant vs. global access

Checklist:
- [ ] Global (cross-tenant) repositories are a separate type from tenant-scoped repositories
- [ ] Global repositories are only used in admin handlers and background jobs with elevated roles
- [ ] Admin handler that uses global repositories checks for admin role before calling
- [ ] Global repository usage searchable in codebase — naming convention makes it auditable

---

## Output Report

### Critical
- Query fetches records by `id` without `tenant_id` filter — any tenant can access any record by guessing an ID
- No `tenant_id` column on tenant-owned tables — all tenant data co-mingled; isolation impossible

### High
- `tenant_id` column exists but no FK constraint — orphaned rows from deleted tenants; data integrity violated
- ORM global scope not enforced — developer must remember to add `WHERE tenant_id = ?` on every query; easy to forget
- No cross-tenant isolation tests — assumption of correctness without verification

### Medium
- `tenant_id` column exists but no index — all tenant-scoped queries perform sequential scans; performance degrades with tenant count
- Composite index has `tenant_id` as second or later column — index unusable for queries filtering by tenant alone
- Global (cross-tenant) repository not separately typed — easy to use it accidentally in tenant-scoped context

### Low
- `tenant_id` added as nullable (historical migration) but never converted to NOT NULL — silently allows rows without a tenant
- Audit query for missing `tenant_id` tables never run — data model isolation gaps undiscovered until a breach
- `ON DELETE CASCADE` vs `ON DELETE RESTRICT` not consciously decided per table — accidental data deletion on tenant deprovisioning

### Passed
- All tenant-scoped tables have `tenant_id UUID NOT NULL REFERENCES tenants(id)`
- Single-column index on `tenant_id`; `tenant_id` leads all composite indexes
- ORM scope or constructor-injected `tenantID` enforced in all repositories
- Cross-tenant integration tests verify that mismatched tenant_id returns not-found
- Global repositories separately typed; used only in admin/system contexts with role enforcement

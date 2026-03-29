---
name: row-level-security-patterns
description: >
  Row-level security enforcement patterns for multi-tenant databases.
  Use when: row-level security, RLS, PostgreSQL RLS, row security policy, tenant isolation RLS,
  SET LOCAL tenant_id, USING clause, WITH CHECK clause, row-level access control,
  RLS policy, ALTER TABLE ENABLE ROW LEVEL SECURITY, CREATE POLICY, RLS bypass,
  RLS performance, RLS index, RLS testing, RLS cross-tenant, SECURITY DEFINER,
  SET ROLE bypass RLS, RLS admin, RLS superuser, row-level isolation PostgreSQL,
  database-enforced isolation, policy-based access control, RLS debugging.
argument-hint: >
  Describe your schema (tenant-scoped tables, shared tables), your application roles
  (app_user, admin, readonly), and whether you need read-only RLS, read-write RLS,
  or both. Include any performance concerns or bypass requirements for admin operations.
---

# Row-Level Security Patterns Specialist

## When to Use

Invoke this skill when you need to:
- Add PostgreSQL RLS policies to enforce tenant isolation at the database layer
- Set per-transaction tenant context via `SET LOCAL`
- Design separate read (`USING`) and write (`WITH CHECK`) policies
- Bypass RLS safely for admin, migration, and reporting queries
- Test RLS enforcement and verify no cross-tenant access is possible
- Optimize query plans when RLS policies are involved

---

## Step 1 — Enable RLS on Tenant-Scoped Tables

```sql
-- Enable RLS on every tenant-owned table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
-- Repeat for every tenant-scoped table

-- IMPORTANT: By default, table owners (superuser) bypass RLS.
-- Force RLS for all roles, including the table owner:
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

**Application role setup:**
```sql
-- Application role — cannot bypass RLS
CREATE ROLE app_user LOGIN PASSWORD '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
-- Do NOT grant BYPASSRLS to the application role

-- Admin role — explicitly allowed to bypass RLS for internal tools
CREATE ROLE admin_user LOGIN PASSWORD '...';
GRANT BYPASSRLS TO admin_user;  -- use sparingly; only for admin console and migrations
```

Checklist:
- [ ] `ENABLE ROW LEVEL SECURITY` applied to every tenant-scoped table
- [ ] `FORCE ROW LEVEL SECURITY` applied — table owners do not silently bypass policies
- [ ] Application DB role does NOT have `BYPASSRLS` privilege
- [ ] Separate admin role with `BYPASSRLS` for internal tooling — not the same role as the app

---

## Step 2 — Write RLS Policies

**Basic tenant isolation policy:**
```sql
-- The policy checks the current_setting for tenant_id
-- Matches rows where tenant_id equals the session-local setting

-- READ policy (USING clause — controls which rows are visible to SELECT/UPDATE/DELETE)
CREATE POLICY tenant_isolation_select ON orders
    AS PERMISSIVE
    FOR SELECT
    TO app_user
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::UUID);

-- WRITE policy (WITH CHECK clause — controls which rows can be inserted/updated)
CREATE POLICY tenant_isolation_write ON orders
    AS PERMISSIVE
    FOR ALL   -- covers INSERT, UPDATE, DELETE
    TO app_user
    USING (tenant_id = current_setting('app.tenant_id', TRUE)::UUID)
    WITH CHECK (tenant_id = current_setting('app.tenant_id', TRUE)::UUID);
```

**Policy explanation:**
- `USING`: filters rows for read and as a pre-condition for update/delete
- `WITH CHECK`: validates rows being inserted or that result from an update
- `current_setting('app.tenant_id', TRUE)`: returns `NULL` if the setting is not set (safe — `NULL != any UUID`)
- `TRUE` as second argument to `current_setting` — prevents error if setting is missing; returns NULL instead

**System/public tables — permissive policy:**
```sql
-- Plans, countries, etc. — readable by all tenants
CREATE POLICY allow_all_read ON plans
    AS PERMISSIVE
    FOR SELECT
    TO app_user
    USING (TRUE);  -- all rows visible; no tenant filter needed
```

Checklist:
- [ ] `USING` clause filters rows visible to SELECT, UPDATE, and DELETE
- [ ] `WITH CHECK` clause validates rows written by INSERT and UPDATE
- [ ] `current_setting('app.tenant_id', TRUE)` used — not `current_setting('...', FALSE)` which throws on missing setting
- [ ] Permissive policies on platform tables (plans, currencies) allow all reads without tenant filter
- [ ] No policy on a table + RLS enabled = no rows visible to app_user (safe default)

---

## Step 3 — Set Tenant Context Per Transaction

The application must inject the `tenant_id` into the PostgreSQL session before executing any query.

**Go — set tenant context at the start of every transaction:**
```go
func (r *TenantAwareDB) WithTenantTx(ctx context.Context, tenantID string, fn func(pgx.Tx) error) error {
    return pgx.BeginTxFunc(ctx, r.pool, pgx.TxOptions{}, func(tx pgx.Tx) error {
        // SET LOCAL — scoped to this transaction only; resets when transaction ends
        // Do NOT use SET (without LOCAL) — leaks across pooled connections
        if _, err := tx.Exec(ctx,
            `SELECT set_config('app.tenant_id', $1, TRUE)`,  // TRUE = local to transaction
            tenantID,
        ); err != nil {
            return fmt.Errorf("set tenant context: %w", err)
        }
        return fn(tx)
    })
}
```

**Why `SET LOCAL` and not `SET`:**
- `SET` persists for the connection lifetime — dangerous with connection pools (connection reused by different tenant)
- `SET LOCAL` (or `set_config('key', value, TRUE)`) scopes the setting to the current transaction — safe with pgBouncer

**For single queries outside a transaction:**
```go
// Use an explicit transaction even for single reads to scope SET LOCAL
func (r *OrderRepo) Get(ctx context.Context, tenantID, orderID string) (*Order, error) {
    var order Order
    err := r.db.WithTenantTx(ctx, tenantID, func(tx pgx.Tx) error {
        return tx.QueryRow(ctx,
            `SELECT id, tenant_id, status FROM orders WHERE id = $1`,
            orderID,
        ).Scan(&order.ID, &order.TenantID, &order.Status)
    })
    return &order, err
}
```

Checklist:
- [ ] `set_config('app.tenant_id', tenantID, TRUE)` used — the `TRUE` flag makes it transaction-local
- [ ] Every database operation wraps queries in a transaction that sets the tenant context first
- [ ] `SET` (without LOCAL) never used for tenant context — leaks across pgBouncer connection pool connections
- [ ] Tenant context not set from user-supplied HTTP input directly — extracted from validated JWT claim

---

## Step 4 — Bypass RLS for Admin and Migrations

```go
// Admin repository explicitly uses the admin_user DB role which has BYPASSRLS
type AdminOrderRepository struct {
    adminDB *pgxpool.Pool  // connects as admin_user, not app_user
}

func (r *AdminOrderRepository) ListAllTenants(ctx context.Context, filter OrderFilter) ([]*Order, error) {
    // No tenant context needed — admin_user bypasses RLS
    rows, err := r.adminDB.Query(ctx,
        `SELECT id, tenant_id, status FROM orders WHERE created_at > $1`,
        filter.After,
    )
    // ...
}
```

**Migration runner:**
```sql
-- Run migrations as the postgres superuser or a role with BYPASSRLS
-- Never run schema migrations as app_user
ALTER ROLE migration_runner BYPASSRLS;
```

**Temporary bypass within a transaction (SECURITY DEFINER):**
```sql
-- Function that runs as its definer (admin), bypassing RLS
CREATE OR REPLACE FUNCTION admin_get_tenant_stats(p_tenant_id UUID)
RETURNS TABLE(order_count BIGINT) AS $$
    SELECT COUNT(*) FROM orders WHERE tenant_id = p_tenant_id
$$ LANGUAGE sql SECURITY DEFINER;
-- Grant execute only to admin role
REVOKE EXECUTE ON FUNCTION admin_get_tenant_stats FROM PUBLIC;
GRANT EXECUTE ON FUNCTION admin_get_tenant_stats TO admin_user;
```

Checklist:
- [ ] Admin DB pool (BYPASSRLS role) never used in tenant request handlers — separate pool, separate role
- [ ] Admin pool credentials are different secrets from app_user credentials
- [ ] `SECURITY DEFINER` functions are restricted by GRANT — not accessible to `app_user`
- [ ] Migration runner uses a dedicated role with BYPASSRLS — not the application user

---

## Step 5 — Test RLS Enforcement

```go
func TestRLS_CrossTenantIsolation(t *testing.T) {
    tenant1 := fixtures.CreateTenant(t, "acme")
    tenant2 := fixtures.CreateTenant(t, "globex")
    order   := fixtures.CreateOrder(t, tenant1.ID)

    // Query as tenant2 — should see 0 rows due to RLS
    db := testAppDB(t) // connects as app_user (has RLS)

    var result *Order
    err := db.WithTenantTx(ctx, tenant2.ID, func(tx pgx.Tx) error {
        return tx.QueryRow(ctx,
            `SELECT id FROM orders WHERE id = $1`, order.ID,
        ).Scan(&result)
    })

    // RLS makes the row invisible — pgx returns ErrNoRows
    assert.ErrorIs(t, err, pgx.ErrNoRows)
    assert.Nil(t, result)
}

func TestRLS_TenantContextNotSet_NoRows(t *testing.T) {
    _ = fixtures.CreateOrder(t, "some-tenant-id")

    db := testAppDB(t)
    // Do NOT set tenant context — should see no rows
    var count int
    db.pool.QueryRow(ctx, `SELECT COUNT(*) FROM orders`).Scan(&count)
    assert.Equal(t, 0, count) // RLS returns 0 rows when app.tenant_id is not set
}
```

Checklist:
- [ ] Test: cross-tenant ID access returns `ErrNoRows` via RLS — not application-layer filtering
- [ ] Test: no tenant context set → 0 rows visible (policy returns false when `current_setting` is NULL)
- [ ] Test: tenant can read own rows when context is set correctly
- [ ] RLS tests run using the `app_user` connection — not superuser/admin which bypasses RLS

---

## Output Report

### Critical
- `SET` (not `SET LOCAL`) used for tenant context — tenant context leaks across pooled connections; cross-tenant queries possible
- Application DB role has `BYPASSRLS` — RLS policies are completely circumvented; tenant isolation is theater
- `FORCE ROW LEVEL SECURITY` not set — table owner (superuser) silently bypasses all policies

### High
- `current_setting('app.tenant_id', FALSE)` used — throws an error when setting is not initialized, rather than returning NULL; unhandled exception bypasses guard
- No transaction wrapping for single queries — `SET LOCAL` cannot be used safely; must use a transaction
- Admin DB pool (BYPASSRLS) shares credentials with app_user pool — no way to audit or restrict admin-level queries

### Medium
- No RLS cross-tenant isolation tests — policies assumed correct without verification
- `SECURITY DEFINER` functions grant access to PUBLIC — bypasses access restrictions; accessible to app_user
- No `WITH CHECK` policy — INSERT and UPDATE can create rows with any `tenant_id`, bypassing the intended isolation

### Low
- `ENABLE ROW LEVEL SECURITY` without `FORCE ROW LEVEL SECURITY` — table owner bypasses silently; may affect local developer testing
- No permissive policy on platform tables (plans, currencies) — app_user cannot read them even though they should be visible to all
- RLS policy not tested with unset session variable — behavior when `app.tenant_id` not initialized is unverified

### Passed
- `ENABLE ROW LEVEL SECURITY` + `FORCE ROW LEVEL SECURITY` on all tenant-scoped tables
- `USING` and `WITH CHECK` policies reference `current_setting('app.tenant_id', TRUE)::UUID`
- `set_config('app.tenant_id', id, TRUE)` called in every transaction before any query
- Application role lacks `BYPASSRLS`; admin role with `BYPASSRLS` is a separate credential
- Cross-tenant isolation and no-context tests pass using app_user connection; RLS verified at DB layer

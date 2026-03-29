---
name: multi-tenancy-design
description: >
  SaaS tenant modeling and isolation strategies — choosing and implementing the right
  multi-tenancy architecture for a SaaS product.
  Use when: multi-tenancy design, SaaS architecture, tenant isolation model, tenant modeling,
  database-per-tenant, schema-per-tenant, shared database multi-tenant, row-level isolation,
  tenant routing, tenant identification, subdomain routing, tenant slug, tenant JWT claim,
  SaaS data model, multi-tenant schema, tenant onboarding model, tenant separation,
  shared infrastructure, dedicated infrastructure, multi-tenant architecture review,
  tenant boundary, logical isolation, physical isolation, hybrid tenancy, silo model, pool model.
argument-hint: >
  Describe your SaaS product (type, scale, compliance requirements) and where you are in the
  design process (greenfield vs. retrofitting existing app). Include expected tenant count,
  data sensitivity, and whether tenants require data residency or dedicated infrastructure.
---

# Multi-Tenancy Design Specialist

## When to Use

Invoke this skill when you need to:
- Choose between database-per-tenant, schema-per-tenant, and shared-database isolation models
- Design the tenant data model (tenant table, membership, roles)
- Implement tenant identification and routing (subdomain, path, JWT claim)
- Retrofit multi-tenancy into an existing single-tenant application
- Evaluate isolation trade-offs for compliance and security requirements

---

## Step 1 — Choose the Right Isolation Model

| Model | Isolation | Cost | Operational Overhead | Best For |
|---|---|---|---|---|
| **Database-per-tenant (Silo)** | Full physical | Highest | High — each tenant has its own DB | Enterprise, regulated industries (healthcare, fintech), large tenants |
| **Schema-per-tenant** | Strong logical | Medium | Medium — DB migrations run per schema | Mid-market; PostgreSQL schemas provide good isolation; < 1,000 tenants |
| **Shared DB, shared schema (Pool)** | Logical via `tenant_id` | Lowest | Low — one schema, one migration path | High-volume SMB SaaS; startups; > 10,000 tenants |
| **Hybrid** | Mixed | Varies | High — two operational paths | When enterprise tier needs dedicated DB, SMB tier uses shared |

**Decision guide:**
```
Does any tenant require physical data separation (compliance, contract)?
  → YES: database-per-tenant (or schema-per-tenant)
  → NO:

Expected tenant count > 1,000?
  → YES: shared DB (pool model) — schema-per-tenant becomes operationally expensive
  → NO:

Do tenants require schema customization?
  → YES: schema-per-tenant
  → NO: shared DB with RLS
```

Checklist:
- [ ] Isolation model chosen based on compliance, scale, and operational capacity — not just convenience
- [ ] Hybrid model explicitly scoped — which tiers use which model; routing logic defined
- [ ] Migration strategy for each model defined before implementation begins
- [ ] Operational runbook exists: how to provision, migrate, and deprovision a tenant in the chosen model

---

## Step 2 — Tenant Data Model

**Core tenant table:**
```sql
CREATE TABLE tenants (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    slug          TEXT        NOT NULL UNIQUE,  -- URL-safe identifier
    display_name  TEXT        NOT NULL,
    plan_id       UUID        NOT NULL REFERENCES plans(id),
    status        TEXT        NOT NULL DEFAULT 'trialing'
                              CHECK (status IN ('trialing','active','past_due','suspended','canceled')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    canceled_at   TIMESTAMPTZ,
    settings      JSONB       NOT NULL DEFAULT '{}'
);

-- Tenant membership
CREATE TABLE tenant_memberships (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id     UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role        TEXT        NOT NULL CHECK (role IN ('owner','admin','member','viewer')),
    invited_by  UUID        REFERENCES users(id),
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, user_id)
);

CREATE INDEX idx_memberships_tenant ON tenant_memberships(tenant_id);
CREATE INDEX idx_memberships_user ON tenant_memberships(user_id);
```

**Every tenant-scoped table carries `tenant_id`:**
```sql
CREATE TABLE orders (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id),
    -- ... business columns
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_tenant ON orders(tenant_id);
-- Composite index for common queries
CREATE INDEX idx_orders_tenant_status ON orders(tenant_id, status, created_at DESC);
```

Checklist:
- [ ] `tenants` table is the single source of truth for tenant status, plan, and settings
- [ ] `tenant_memberships` (not embedded roles on `users`) — one user can belong to multiple tenants
- [ ] Every domain table has `tenant_id` column with `NOT NULL` constraint and an index
- [ ] `tenant_id` indexed alone and as the leading column of every composite index

---

## Step 3 — Tenant Identification and Routing

**Strategy options:**

| Strategy | URL Pattern | Pros | Cons |
|---|---|---|---|
| Subdomain | `acme.app.com` | Clean UX; natural brand | Wildcard cert required; local dev complexity |
| Path prefix | `app.com/t/acme/` | No wildcard cert; simpler | Uglier URLs; some apps handle this poorly |
| JWT claim | `app.com` with `tenant_id` in JWT | SPA-friendly; no DNS changes | Token refresh required on tenant switch |
| Custom domain | `app.acme.com` → `app.com` | White-label; premium UX | DNS + cert management per tenant |

**Subdomain extraction middleware (Go):**
```go
func TenantMiddleware(repo TenantRepository) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            slug := extractSubdomain(r.Host) // "acme" from "acme.app.com"
            if slug == "" {
                http.Error(w, "tenant not specified", http.StatusBadRequest)
                return
            }

            tenant, err := repo.GetBySlug(r.Context(), slug)
            if err != nil || tenant == nil {
                http.Error(w, "tenant not found", http.StatusNotFound)
                return
            }
            if tenant.Status == "suspended" || tenant.Status == "canceled" {
                http.Error(w, "tenant inactive", http.StatusForbidden)
                return
            }

            ctx := context.WithValue(r.Context(), tenantKey{}, tenant)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

func TenantFromContext(ctx context.Context) *Tenant {
    t, _ := ctx.Value(tenantKey{}).(*Tenant)
    return t
}
```

**JWT claim approach:**
```json
{
  "sub": "user_abc123",
  "tenant_id": "ten_xyz789",
  "tenant_slug": "acme",
  "role": "admin",
  "exp": 1743255600
}
```

Checklist:
- [ ] Tenant resolved once per request in middleware — not repeated in every handler
- [ ] Tenant status validated during resolution — suspended tenants receive 403, not 200
- [ ] Tenant context stored in `context.Context` — typed key (not string) prevents collision
- [ ] Tenant slug/ID validated against the authenticated user's membership — not blindly trusted

---

## Step 4 — Isolate Data Access

Regardless of isolation model, every data access function must be scoped to the current tenant.

**Repository pattern with mandatory `tenantID`:**
```go
type OrderRepository interface {
    // tenantID is a required parameter — not optional
    List(ctx context.Context, tenantID string, filter OrderFilter) ([]*Order, error)
    Get(ctx context.Context, tenantID, orderID string) (*Order, error)
    Create(ctx context.Context, tenantID string, order *Order) error
    Update(ctx context.Context, tenantID, orderID string, update OrderUpdate) error
}

type pgOrderRepository struct{ db *pgxpool.Pool }

func (r *pgOrderRepository) Get(ctx context.Context, tenantID, orderID string) (*Order, error) {
    // tenantID always in WHERE clause — never just WHERE id = $1
    row := r.db.QueryRow(ctx,
        `SELECT id, tenant_id, status, total FROM orders
         WHERE tenant_id = $1 AND id = $2`,
        tenantID, orderID,
    )
    // ...
}
```

Checklist:
- [ ] Repository methods require `tenantID` as an explicit parameter — not derived from global state
- [ ] No query selects by `id` alone without `tenant_id = $1` — prevents cross-tenant data access
- [ ] All repository implementations tested with mismatched tenant ID — must return not found, not data
- [ ] Global ORM scopes enforced where available (GORM's `Scopes`, Rails' `default_scope`)

---

## Step 5 — Schema-per-Tenant and DB-per-Tenant Specifics

**Schema-per-tenant routing (PostgreSQL):**
```go
func (r *Pool) connForTenant(ctx context.Context, tenantID string) (*pgx.Conn, error) {
    schema := "tenant_" + tenantID
    conn, err := r.pool.Acquire(ctx)
    if err != nil {
        return nil, err
    }
    // Set search_path for this connection
    _, err = conn.Exec(ctx, fmt.Sprintf("SET LOCAL search_path TO %s, public", schema))
    return conn, err
}
```

**DB-per-tenant connection routing:**
```go
type TenantRouter struct {
    mu      sync.RWMutex
    pools   map[string]*pgxpool.Pool  // tenantID → DB pool
    catalog CatalogDB                  // maps tenant → DSN
}

func (r *TenantRouter) GetPool(ctx context.Context, tenantID string) (*pgxpool.Pool, error) {
    r.mu.RLock()
    pool, ok := r.pools[tenantID]
    r.mu.RUnlock()
    if ok {
        return pool, nil
    }
    // Lazy-initialize pool from catalog
    return r.initPool(ctx, tenantID)
}
```

**Migration strategy for schema-per-tenant:**
```bash
# Run migrations for all tenant schemas
for schema in $(psql -c "SELECT schema_name FROM information_schema.schemata WHERE schema_name LIKE 'tenant_%'" -t); do
    migrate -path ./migrations -database "postgres://...?search_path=${schema}" up
done
```

Checklist:
- [ ] `SET LOCAL search_path` used (not `SET`) — scoped to transaction, not session (prevents leakage)
- [ ] New tenant provisioning includes running all current migrations on the new schema/DB
- [ ] Schema name derived from a stable, sanitized tenant ID — not user-supplied slug directly (SQL injection risk)
- [ ] Migration runner supports per-schema/per-DB execution in CI

---

## Output Report

### Critical
- No `tenant_id` on domain tables — all tenant data co-mingled; cross-tenant data access trivially possible
- Tenant resolved from untrusted user input without membership verification — tenant spoofing possible
- `SET search_path` (not `SET LOCAL`) used for schema routing — search_path leaks across transactions in pooled connections

### High
- Missing index on `tenant_id` column — all queries perform sequential scans; performance collapses with tenant count growth
- Repository methods do not include `tenant_id` in WHERE clause — query returns data from any tenant matching the ID
- Suspended/canceled tenants not checked during tenant resolution — inactive tenants continue to access the system

### Medium
- Tenant context stored with a string key in `context.Context` — collision risk; use a typed unexported struct key
- Schema name derived directly from user-supplied input — SQL injection via malicious tenant slug
- Migration strategy not defined for the chosen isolation model — schema drift risk as tenants are provisioned at different times

### Low
- Tenant slug not validated for URL safety — special characters cause routing failures
- No operational runbook for provisioning and deprovisioning a tenant in the chosen model
- Hybrid model tenant-tier mapping undocumented — routing logic inconsistently applied

### Passed
- Isolation model chosen based on compliance requirements, scale, and operational capacity
- `tenant_id` on every domain table; indexed; `NOT NULL`; foreign key to `tenants`
- Tenant resolved once per request in middleware; status validated; typed context key used
- All repository methods require explicit `tenantID` parameter; queries always include `tenant_id = $1`
- Schema/DB routing uses `SET LOCAL`; schema names derived from stable internal ID, not user input

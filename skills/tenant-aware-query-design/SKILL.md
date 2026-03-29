---
name: tenant-aware-query-design
description: >
  Query design ensuring tenant filtering is mandatory and cannot be omitted in multi-tenant SaaS.
  Use when: tenant-aware queries, tenant filtering, tenant_id WHERE clause, tenant scoped query,
  ORM tenant scope, global scope ORM, missing tenant filter, multi-tenant query safety,
  mandatory tenant filter, accidental cross-tenant query, GORM tenant scope, SQLAlchemy tenant,
  query review checklist, tenant-safe SQL, raw SQL tenant filter, parameterized tenant query,
  tenant filter enforcement, query-layer tenant isolation, repository tenant enforcement,
  query linting, missing tenant_id, query audit, tenant query pattern.
argument-hint: >
  Describe your query layer (raw SQL, GORM, SQLx, SQLAlchemy, Ecto, LINQ), where tenant filtering
  is currently applied (consistently / inconsistently / missing), and whether this is a greenfield
  design or a retrofit. Include any ORM-specific constraints or existing query patterns.
---

# Tenant-Aware Query Design Specialist

## When to Use

Invoke this skill when you need to:
- Design repository and query patterns that make tenant filtering mandatory
- Configure ORM global/default scopes for automatic tenant filtering
- Audit existing queries for missing `WHERE tenant_id = ?` clauses
- Prevent developers from accidentally writing cross-tenant queries
- Write raw SQL queries safely in a multi-tenant shared-database context
- Design a query review checklist for PR code review

---

## Step 1 — The Core Rule

**Every SELECT, UPDATE, and DELETE on a tenant-scoped table MUST include `WHERE tenant_id = $N`.**

This is not a guideline — it is a structural requirement. Enforce it through code structure, not discipline.

**Three enforcement layers (in order of preference):**

| Layer | Mechanism | When to Use |
|---|---|---|
| **Database** | PostgreSQL RLS policy | Strongest; enforced even if app bypasses it |
| **ORM scope** | GORM global scope, SQLAlchemy event listener, Rails `default_scope` | Catches ORM-generated queries |
| **Repository pattern** | Constructor-injected `tenantID`; explicit parameter | For raw SQL or when ORM scopes aren't available |

Apply all three for defense-in-depth.

---

## Step 2 — Safe Raw SQL Query Patterns

When writing raw SQL, `tenant_id` must be a named placeholder that is always populated.

**Go + pgx — correct pattern:**
```go
// CORRECT: tenant_id first in WHERE clause; typed parameter
func (r *OrderRepo) List(ctx context.Context, tenantID string, status string) ([]*Order, error) {
    const q = `
        SELECT id, tenant_id, status, total, created_at
        FROM orders
        WHERE tenant_id = $1
          AND status = $2
        ORDER BY created_at DESC
        LIMIT 100`

    rows, err := r.db.Query(ctx, q, tenantID, status)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    return pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[Order])
}

// CORRECT: Get by ID also includes tenant_id
func (r *OrderRepo) Get(ctx context.Context, tenantID, orderID string) (*Order, error) {
    const q = `
        SELECT id, tenant_id, status, total
        FROM orders
        WHERE tenant_id = $1 AND id = $2`

    var o Order
    err := r.db.QueryRow(ctx, q, tenantID, orderID).Scan(&o.ID, &o.TenantID, &o.Status, &o.Total)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, ErrNotFound
    }
    return &o, err
}

// WRONG: missing tenant_id in WHERE — cross-tenant access possible
func (r *OrderRepo) GetWrong(ctx context.Context, orderID string) (*Order, error) {
    const q = `SELECT id, status FROM orders WHERE id = $1` // MISSING tenant_id
    // ...
}
```

**Repository constructor injection pattern:**
```go
// TenantID injected at construction — never forgotten at call time
type OrderRepo struct {
    db       *pgxpool.Pool
    tenantID string  // injected — cannot be nil (string zero value is caught in tests)
}

func NewOrderRepo(db *pgxpool.Pool, tenantID string) *OrderRepo {
    if tenantID == "" {
        panic("NewOrderRepo: tenantID must not be empty")
    }
    return &OrderRepo{db: db, tenantID: tenantID}
}

// Every method uses r.tenantID — no chance to forget
func (r *OrderRepo) List(ctx context.Context, filter OrderFilter) ([]*Order, error) {
    return r.query(ctx, r.tenantID, filter)
}
```

Checklist:
- [ ] Every raw SQL query on tenant-scoped tables has `WHERE tenant_id = $N`
- [ ] `tenant_id` is always the FIRST parameter in parameterized queries — position makes it visually obvious
- [ ] GET-by-ID queries include `AND tenant_id = $2` — not just `WHERE id = $1`
- [ ] Repository panics on empty `tenantID` at construction — catches programming errors immediately

---

## Step 3 — ORM Global Scopes

**GORM:**
```go
// Register a global scope that applies to every query on the model
type OrderRepository struct {
    db *gorm.DB // pre-scoped at construction
}

func NewOrderRepository(db *gorm.DB, tenantID string) *OrderRepository {
    if tenantID == "" {
        panic("tenantID required")
    }
    return &OrderRepository{
        db: db.Where("tenant_id = ?", tenantID),
    }
}

func (r *OrderRepository) List(ctx context.Context) ([]*Order, error) {
    var orders []*Order
    // tenant_id filter is always applied — developer cannot forget it
    err := r.db.WithContext(ctx).Find(&orders).Error
    return orders, err
}

// Beware: .Unscoped() removes ALL scopes including tenant_id — never use in app code
// Only use explicitly in admin/migration contexts
```

**SQLAlchemy (Python):**
```python
from sqlalchemy.orm import with_loader_criteria

class TenantSession:
    def __init__(self, session, tenant_id: str):
        self.session = session.execution_options(
            populate_existing=True
        )
        self.tenant_id = tenant_id

    def query(self, model):
        # Apply tenant filter automatically
        return (
            self.session.query(model)
            .filter(model.tenant_id == self.tenant_id)
        )
```

**Rails (ActiveRecord):**
```ruby
class ApplicationRecord < ActiveRecord::Base
  # Override in tenant-scoped models
  def self.for_tenant(tenant_id)
    where(tenant_id: tenant_id)
  end
end

# In controllers — always use the scoped query
class OrdersController < ApplicationController
  def index
    @orders = Order.for_tenant(current_tenant.id).page(params[:page])
  end
end
```

Checklist:
- [ ] ORM scope applied at repository construction — not added per-query
- [ ] No `Unscoped()` / `unscoped` / `all` calls in tenant request handlers — only in admin/migration code
- [ ] Eager loading (includes/preload) also filtered to tenant — ORM sometimes bypasses scopes on includes
- [ ] ORM scope tested — verify it generates `WHERE tenant_id = ?` by inspecting SQL in tests

---

## Step 4 — Query Review Checklist

Use this checklist in PR review for any query touching tenant-scoped tables.

**Mandatory checks per query:**
```
[ ] Does this table have a tenant_id column?
    YES → continue; NO → skip (platform table)

[ ] Is tenant_id = $N in the WHERE clause?
    YES → pass; NO → BLOCK PR

[ ] Is tenant_id the first condition in the WHERE clause?
    Preferred — makes it visually obvious

[ ] Does GET-by-ID include AND tenant_id = $N?
    YES → pass; NO → BLOCK PR (IDOR vulnerability)

[ ] Can the query be triggered by user input without authentication?
    YES → also verify authentication and authorization middleware applied

[ ] Is this a raw SQL query bypassing ORM scopes?
    YES → apply extra scrutiny; prefer ORM-based query
```

Checklist:
- [ ] PR review checklist shared with the team and pinned in the pull request template
- [ ] Automated linting configured if possible (see Step 5)
- [ ] Raw SQL bypass of ORM layers requires reviewer sign-off from a senior engineer

---

## Step 5 — Automated Detection of Missing Tenant Filters

**Static analysis — search for queries missing tenant filter:**
```bash
# Search for repository methods that query without tenant_id
grep -rn "FROM orders" --include="*.go" | grep -v "tenant_id"
grep -rn "FROM projects" --include="*.go" | grep -v "tenant_id"

# Search for .Find() or .All() calls in GORM without scope
grep -rn "\.Find(&" --include="*.go" | grep -v "WithTenant\|tenantID\|tenant_id"
```

**Integration test — detect if any query can return cross-tenant data:**
```go
func TestNoQueryReturnsDataWithoutTenantContext(t *testing.T) {
    // Create data for tenant 1
    t1 := fixtures.CreateTenant(t)
    fixtures.CreateOrder(t, t1.ID)

    // Query with tenant 2 context — must return 0 rows
    t2 := fixtures.CreateTenant(t)
    repo := NewOrderRepo(db, t2.ID) // scoped to tenant 2
    orders, err := repo.List(ctx, OrderFilter{})

    assert.NoError(t, err)
    assert.Len(t, orders, 0, "tenant 2 must see 0 orders from tenant 1")
}
```

Checklist:
- [ ] Grep-based check added to CI for raw SQL queries missing tenant_id on known tenant-scoped tables
- [ ] Cross-tenant isolation integration test covers all resource types
- [ ] ORM SQL logging enabled in test mode — verify every query includes tenant_id in WHERE clause

---

## Output Report

### Critical
- `SELECT` or `UPDATE` or `DELETE` by ID alone without `tenant_id` in WHERE — IDOR vulnerability; any tenant can access any resource by ID
- ORM `.Unscoped()` called in a request handler — all scopes removed; unrestricted cross-tenant access

### High
- Repository methods pass `tenantID` as an optional parameter — callers forget to pass it; some queries run unscoped
- GET-by-ID queries missing `AND tenant_id = $2` — fetches by global ID; ownership not verified
- ORM eager loading bypasses tenant scope — related resources fetched without tenant filter

### Medium
- No automated check for missing tenant filter — CI does not detect when a new query omits tenant_id
- Cross-tenant isolation test not in test suite — correctness assumed, not verified
- Raw SQL used where ORM would enforce tenant scope — higher risk of developer oversight

### Low
- `tenant_id` not the first condition in WHERE — does not affect correctness but reduces visual clarity and code review effectiveness
- Repository panic on empty tenantID not implemented — empty string slips through; queries return all-tenant data
- ORM SQL not logged in tests — impossible to verify that the generated SQL includes tenant_id

### Passed
- Every raw SQL query on tenant-scoped tables includes `WHERE tenant_id = $1`; all GET-by-ID include `AND tenant_id = $2`
- ORM global scope applied at repository construction; `Unscoped()` absent from all request handlers
- Repository panics on empty `tenantID` at construction; typed error messages identify the offending repository
- PR review checklist enforced; grep-based CI check for missing tenant filter on known tables
- Cross-tenant isolation integration tests cover all resource types; ORM SQL verified in tests

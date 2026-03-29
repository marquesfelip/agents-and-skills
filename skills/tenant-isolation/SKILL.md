---
name: tenant-isolation
description: 'Logical isolation between tenants across queries, storage, and background processing in multi-tenant systems. Use when: tenant isolation, multi-tenant security, tenant data leakage, cross-tenant access, row-level security, tenant scoping, SaaS multi-tenancy, tenant boundary enforcement, tenant data separation, shared database multi-tenant, schema-per-tenant, database-per-tenant, tenant query filter, background job tenant isolation.'
argument-hint: 'Data access layer, ORM model, background job, or API endpoint — and the tenancy model (shared DB / schema-per-tenant / DB-per-tenant)'
---

# Tenant Isolation Specialist

## When to Use
- Designing or reviewing multi-tenant SaaS data access patterns
- Auditing queries for missing tenant scope filters
- Evaluating which tenancy model fits security and compliance requirements
- Hardening background jobs, queues, and scheduled tasks against cross-tenant data access
- Implementing row-level security at the database layer

---

## Step 1 — Choose and Verify Tenancy Model

Select and document the isolation model in use, then validate its specific risks:

| Model | Isolation level | Compliance fit | Risk profile |
|---|---|---|---|
| **Shared DB, shared schema** | Application-layer only | Low — harder to certify | Missing WHERE clause = full tenant bleed |
| **Shared DB, schema-per-tenant** | Schema-level | Medium | Misconfigured search_path or connection pool can cross schema |
| **Database-per-tenant** | DB-level | High — SOC 2 / ISO easier | Higher cost; connection pool management harder |
| **Silo (separate infra per tenant)** | Full infrastructure | Highest | Highest cost; used for enterprise/regulated tenants |

Most SaaS start with shared schema. Risk is proportional to application-layer correctness.

---

## Step 2 — Shared Schema: Tenant Column Audit

Every data table must have a `tenant_id` column, and **every query** must filter by it.

### Baseline Table Design

```sql
-- BAD — no tenant_id; full data accessible via any tenant's queries
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID,
    amount DECIMAL
);

-- GOOD — tenant_id on every row; composite index for performance
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID NOT NULL,
    amount DECIMAL NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_tenant ON orders (tenant_id, created_at DESC);
```

### Checklist — Per Table

- [ ] `tenant_id` column present (NOT NULL) on every entity table
- [ ] Composite index on `(tenant_id, <primary sort column>)` for query performance
- [ ] Foreign key constraint on `tenant_id` referencing `tenants` table
- [ ] No table that holds cross-tenant data without explicit documentation and access review

---

## Step 3 — Enforce Tenant Scope at Query Layer

### ORM-Level Automatic Scoping (Preferred)

Apply `tenant_id` filtering at the ORM/query-builder layer so individual developers cannot forget it:

```python
# Python / SQLAlchemy — Tenant-scoped session factory
class TenantSession:
    def __init__(self, db_session, tenant_id: str):
        self._db = db_session
        self._tenant_id = tenant_id

    def query(self, model):
        # Every query automatically filtered to current tenant
        return self._db.query(model).filter(model.tenant_id == self._tenant_id)

# Usage — cannot query cross-tenant accidentally
session = TenantSession(db, current_tenant_id)
orders = session.query(Order).filter(Order.status == 'pending').all()
# Generated SQL: WHERE tenant_id = $1 AND status = 'pending'
```

```go
// Go — Tenant-scoped repository pattern
type OrderRepository struct {
    db       *sql.DB
    tenantID string
}

func (r *OrderRepository) FindByID(ctx context.Context, orderID string) (*Order, error) {
    // tenant_id ALWAYS included — cannot be bypassed
    row := r.db.QueryRowContext(ctx,
        `SELECT id, user_id, amount FROM orders WHERE tenant_id = $1 AND id = $2`,
        r.tenantID, orderID,
    )
    // ...
}
```

### Code Review Rules

For shared-schema tenancy, **every** query that reads tenant data must:
- [ ] Include `WHERE tenant_id = $current_tenant` (or equivalent ORM filter)
- [ ] Use parameterized queries (never string interpolation)
- [ ] Return at most data belonging to the current tenant — verify with test

**Raw query audit pattern — search codebase for dangerous patterns:**

```bash
# Find SQL queries missing tenant filter in Go/Python/JS
grep -rn "SELECT\|FROM orders\|FROM users\|FROM invoices" src/ \
  | grep -v "tenant_id"
# Review every hit manually
```

---

## Step 4 — Row-Level Security (Database Enforcement)

For shared-schema systems, enforce tenant isolation at the database layer as a second line of defense using PostgreSQL RLS:

```sql
-- Enable RLS on every tenant table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- applies to table owner too

-- Policy: each connection can only see its tenant's rows
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Application sets the tenant context at the start of each request
SET LOCAL app.current_tenant_id = '<tenant_uuid>';
```

**Integration with connection pool:**

```python
def get_db_connection(tenant_id: str):
    conn = pool.getconn()
    with conn.cursor() as cur:
        cur.execute("SET LOCAL app.current_tenant_id = %s", (tenant_id,))
    return conn
```

RLS provides DB-layer enforcement even if application-layer filter is omitted.

---

## Step 5 — ID Enumeration and Cross-Tenant Object Access

A common bug: validate ownership (tenant scope) when accessing any resource by ID.

```python
# BAD — any authenticated user from any tenant can access any order by guessing ID
@app.get('/orders/{order_id}')
def get_order(order_id: str, user: User = Depends(get_current_user)):
    return db.query(Order).filter(Order.id == order_id).first()  # no tenant check!

# GOOD — tenant_id verified; returns 404 for cross-tenant IDs (not 403 — prevents enumeration)
@app.get('/orders/{order_id}')
def get_order(order_id: str, user: User = Depends(get_current_user)):
    order = db.query(Order).filter(
        Order.id == order_id,
        Order.tenant_id == user.tenant_id   # tenant scope enforced
    ).first()
    if not order:
        raise HTTPException(status_code=404)  # 404, not 403 — avoids confirming existence
    return order
```

**Use non-sequential, unpredictable IDs (UUID v4 / ULID)** to mitigate enumeration even if scope check is missing.

---

## Step 6 — Background Jobs and Async Processing

Background tasks frequently bypass the request context that carries `tenant_id`.

### Tenant Context Propagation Pattern

```python
# BAD — job processes ALL records without tenant filter
@celery_app.task
def send_invoice_reminders():
    unpaid = db.query(Invoice).filter(Invoice.status == 'unpaid').all()  # ALL tenants!
    for invoice in unpaid:
        send_reminder(invoice)

# GOOD — job is tenant-scoped; scheduled per-tenant or batch with explicit context
@celery_app.task
def send_invoice_reminders(tenant_id: str):
    unpaid = db.query(Invoice).filter(
        Invoice.tenant_id == tenant_id,
        Invoice.status == 'unpaid'
    ).all()
    for invoice in unpaid:
        send_reminder(invoice)

# Scheduler dispatches per-tenant
for tenant in active_tenants():
    send_invoice_reminders.delay(tenant.id)
```

### Message Queue Isolation

- [ ] Messages in queues include `tenant_id` in payload — consumers always scope queries by it
- [ ] Separate queues per tenant (for high-isolation tiers) or single queue with mandatory tenant field
- [ ] Queue consumers validate `tenant_id` is present and non-empty before processing
- [ ] Dead letter queue messages include `tenant_id` for safe replay without cross-contamination

---

## Step 7 — Feature Flags and Plan Limits Per Tenant

Tenant isolation extends beyond data to feature and resource access:

```python
def is_feature_enabled(tenant_id: str, feature: str) -> bool:
    plan = db.get_tenant_plan(tenant_id)
    return feature in plan.allowed_features

# Enforce in API
@app.post('/api/export')
def export_data(user: User = Depends(get_current_user)):
    if not is_feature_enabled(user.tenant_id, 'advanced_export'):
        raise HTTPException(403, "Feature not available on your plan")
```

- [ ] Resource quotas (API calls, storage, seats) enforced per tenant — cannot be exceeded even by application bugs
- [ ] Tenant plan checked before feature execution, not just at UI level

---

## Step 8 — Testing Tenant Isolation

Every data access path must have a cross-tenant test:

```python
def test_cannot_read_other_tenant_order():
    tenant_a = create_tenant()
    tenant_b = create_tenant()
    order = create_order(tenant=tenant_a)

    # tenant_b user attempts to access tenant_a's order
    response = client.get(
        f'/orders/{order.id}',
        headers=auth_headers(tenant=tenant_b)
    )
    assert response.status_code == 404  # NOT 200 — tenant isolation enforced
```

**Required test coverage:**
- [ ] Cross-tenant GET (read another tenant's resource)
- [ ] Cross-tenant PUT/PATCH (modify another tenant's resource)
- [ ] Cross-tenant DELETE (delete another tenant's resource)
- [ ] Background job scope (job for tenant A cannot process tenant B data)
- [ ] Bulk operations (e.g., export) scoped to requesting tenant only

---

## Output Report

```
## Tenant Isolation Review: <service>

### Critical
- GET /api/invoices/:id has no tenant_id filter — any authenticated user from any tenant
  can access any invoice by guessing the ID (sequential integer IDs make this trivial)
  Fix: add tenant_id = user.tenant_id to query; switch to UUID primary keys

- Background job `process_monthly_invoices` queries all Invoice records without tenant filter
  Entire invoice dataset processed without tenant scope; billing data from all tenants exposed
  Fix: pass tenant_id to job; filter all queries; schedule one job invocation per active tenant

### High
- Shared DB schema with no Row-Level Security policy configured
  Application-layer filter is the sole isolation mechanism; single buggy query breaches all tenants
  Fix: enable RLS via PostgreSQL policy; set app.current_tenant_id at connection setup

- User table missing tenant_id column; users identified only by email
  Cannot isolate user records by tenant; email-based lookup could collide across tenants
  Fix: add tenant_id column (NOT NULL); migrate existing records; add composite unique constraint on (tenant_id, email)

### Medium
- Celery tasks receive full ORM objects instead of tenant_id + IDs
  Serialization/deserialization may lose tenant context; objects contain cross-tenant relationships
  Fix: pass only (tenant_id, entity_id) in task payloads; reload DB object inside task with tenant filter

### Low
- Tenant IDs exposed in URL paths — not a vulnerability if scope check is correct, but leaks tenant existence
  Consider opaque slugs as alternative

### Passed
- All ORM query methods wrapped in TenantSession with automatic filter ✓
- Cross-tenant tests present for all CRUD operations ✓
- UUID v4 primary keys prevent enumeration ✓
```

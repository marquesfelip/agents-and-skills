---
name: tenant-data-leak-prevention
description: 'Protections against cross-tenant data exposure in multi-tenant SaaS systems. Use when: tenant data leak prevention, cross-tenant data exposure, tenant data leakage, multi-tenant data isolation, cross-tenant API response, tenant boundary enforcement, data isolation audit, IDOR multi-tenant, tenant data in response, PII cross-tenant, cross-tenant query, tenant-scoped response, tenant data exposure, excessive data exposure multi-tenant, API over-exposure tenant, cross-tenant field leak, missing tenant filter, tenant ownership gap, lateral tenant access.'
argument-hint: 'Describe the data types at risk (e.g., user profiles, documents, billing records), your tenancy model (shared DB with tenant_id, schema-per-tenant, DB-per-tenant), and any known or suspected exposure vectors (API responses, search, exports, notifications).'
---

# Tenant Data Leak Prevention

## When to Use

Invoke this skill when you need to:
- Audit API responses for fields that should not be visible across tenant boundaries
- Find and fix missing `tenant_id` filters in queries, exports, search indexes, and background jobs
- Design structural safeguards that make cross-tenant exposure architecturally impossible
- Respond to a cross-tenant data exposure security report or incident
- Build a pre-release checklist for any feature that touches multi-tenant data

---

## Threat Model: How Tenant Data Leaks

| Vector | Example | Root Cause |
|---|---|---|
| Missing WHERE filter | `SELECT * FROM orders WHERE id = $1` — no `tenant_id` | IDOR / BOLA at DB query layer |
| URL tenant ID trusted | `GET /api/tenants/{tenantId}/invoices` — `tenantId` from path, not token | Tenant scoping from untrusted input |
| Over-exposed response fields | User object returned with `internal_notes`, `stripe_customer_id` | No field-level serialization policy |
| Search index bleeding | Full-text search returns results from all tenants | Search index not scoped per tenant |
| Export / bulk download | CSV export unfiltered across tenants | Missing tenant filter in export query |
| Background job scope | Job processes all records globally to find "expired" ones | Missing tenant scope in scheduled query |
| Notification cross-contamination | Email recipient resolved from wrong tenant's member list | Tenant context lost in async job |
| Pagination escape | Cursor-based pagination encodes global row ID, not tenant-scoped offset | Cursor not tied to tenant context |
| Shared cache key | `cache.Get("user:{userID}")` without tenant prefix | Cache key does not include tenant scope |

---

## Step 1 — Enforce Tenant Isolation at the Repository Layer

Every query that retrieves tenant data MUST include `tenant_id`. This is a structural rule, not a case-by-case check.

```go
// WRONG — vulnerable to IDOR
func (r *OrderRepo) GetByID(ctx context.Context, id string) (*Order, error) {
    return r.db.QueryRow(ctx, `SELECT * FROM orders WHERE id = $1`, id)
}

// CORRECT — tenant-scoped
func (r *OrderRepo) GetByID(ctx context.Context, tenantID, id string) (*Order, error) {
    return r.db.QueryRow(ctx, `SELECT * FROM orders WHERE id = $1 AND tenant_id = $2`, id, tenantID)
}
```

**Pattern for all repository methods:**
- All `GetByID`, `Update`, `Delete` methods accept `tenantID` as first domain argument
- Tenant ID is ALWAYS sourced from the authenticated context — never from the URL, request body, or query parameter

```go
// Extract tenant from auth context — never from request input
tenantID := auth.TenantFromCtx(ctx)
order, err := orderRepo.GetByID(ctx, tenantID, orderID)
if errors.Is(err, sql.ErrNoRows) {
    // Return 404 — do NOT distinguish "not found" from "belongs to another tenant"
    return nil, ErrNotFound
}
```

---

## Step 2 — Response Serialization Policy

Define explicit serialization structs — never return raw DB models to API callers.

```go
// WRONG — exposes internal fields
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    user, _ := h.repo.GetByID(ctx, userID)
    json.NewEncoder(w).Encode(user) // leaks: stripe_customer_id, internal_notes, hashed_password...
}

// CORRECT — explicit response DTO
type UserResponse struct {
    ID          string `json:"id"`
    Email       string `json:"email"`
    DisplayName string `json:"display_name"`
    Role        string `json:"role"`
    CreatedAt   string `json:"created_at"`
    // No: StripeCustomerID, HashedPassword, InternalNotes, TenantID, DeletedAt
}

func toUserResponse(u *User) UserResponse {
    return UserResponse{
        ID: u.ID, Email: u.Email,
        DisplayName: u.DisplayName, Role: u.Role,
        CreatedAt: u.CreatedAt.Format(time.RFC3339),
    }
}
```

**Field exposure policy per actor type:**

| Field category | Public API user | Admin user | Platform ops |
|---|---|---|---|
| `id`, `name`, `created_at` | ✅ | ✅ | ✅ |
| `email`, `role` | Own record only | All in tenant | ✅ |
| `stripe_customer_id` | ❌ | ❌ | ✅ |
| `internal_notes` | ❌ | ❌ | ✅ |
| `hashed_password`, `mfa_secret` | ❌ | ❌ | ❌ (never in API) |
| `tenant_id` | ❌ | ❌ | ✅ |

---

## Step 3 — Tenant-Scoped Search

Search indexes must be namespace-isolated per tenant:

```go
// WRONG — Elasticsearch index shared across all tenants
esClient.Search("documents", query)

// CORRECT — Option A: tenant-prefixed index
indexName := fmt.Sprintf("documents_%s", tenantID)
esClient.Search(indexName, query)

// CORRECT — Option B: shared index with mandatory tenant_id filter
esClient.Search("documents", map[string]any{
    "query": map[string]any{
        "bool": map[string]any{
            "must": []any{
                map[string]any{"term": map[string]any{"tenant_id": tenantID}},
                userQuery,
            },
        },
    },
})
```

**PostgreSQL full-text search:**
```sql
-- Always include tenant_id in the WHERE clause before the full-text condition
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM documents, to_tsquery('english', $1) AS query
WHERE tenant_id = $2           -- tenant filter FIRST
  AND search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

---

## Step 4 — Scoped Cache Keys

Cache keys that contain tenant data must always be scoped with `tenant_id`:

```go
// WRONG — shared across tenants
cacheKey := fmt.Sprintf("user:%s", userID)

// CORRECT — tenant-namespaced
cacheKey := fmt.Sprintf("tenant:%s:user:%s", tenantID, userID)

// CORRECT — bulk tenant entitlement cache
cacheKey := fmt.Sprintf("tenant:%s:entitlements", tenantID)
```

Invalidation must also be tenant-scoped — flush `tenant:{tenantID}:*` not global patterns.

---

## Step 5 — Background Job Tenant Isolation

Background jobs that process data across tenants must always carry explicit tenant context:

```go
// WRONG — no tenant filter; processes all expired records globally
func (j *CleanupJob) Run(ctx context.Context) error {
    records, _ := j.repo.FindExpired(ctx) // returns ALL tenants' expired records
    for _, r := range records {
        j.process(ctx, r)  // no tenant context passed
    }
}

// CORRECT — process per-tenant, pass tenant context explicitly
func (j *CleanupJob) Run(ctx context.Context) error {
    tenants, _ := j.tenantRepo.ListActive(ctx)
    for _, t := range tenants {
        tenantCtx := auth.WithTenant(ctx, t.ID)
        records, _ := j.repo.FindExpiredForTenant(tenantCtx, t.ID)
        for _, r := range records {
            j.process(tenantCtx, r)
        }
    }
}
```

---

## Step 6 — Audit Sweep Queries

Run these SQL checks regularly (pre-release, in CI, or as a daily monitoring job):

```sql
-- 1. Tables without tenant_id column (surface area review)
SELECT table_name
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name NOT IN ('schema_migrations','plans','plan_features')  -- known global tables
GROUP BY table_name
HAVING COUNT(*) FILTER (WHERE column_name = 'tenant_id') = 0;

-- 2. Detect cross-tenant references: FK points to row in different tenant
-- Example: orders.user_id should resolve to a user in the same tenant
SELECT o.id AS order_id, o.tenant_id AS order_tenant, u.tenant_id AS user_tenant
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.tenant_id <> u.tenant_id;
-- Expected: 0 rows

-- 3. Orphaned records with no valid tenant
SELECT id FROM documents
WHERE tenant_id NOT IN (SELECT id FROM tenants);
-- Expected: 0 rows
```

---

## Quality Checks

- [ ] All repository `GetByID` / `Update` / `Delete` methods include `tenant_id` in the WHERE clause
- [ ] Tenant ID is always sourced from the authenticated context — never from URL, query param, or body
- [ ] API responses use explicit serialization DTOs — raw DB models are never encoded directly
- [ ] Fields scoped to ops/admin are excluded from user-facing response structs structurally (different type)
- [ ] Search indexes are tenant-scoped (either per-tenant index or mandatory `tenant_id` filter)
- [ ] Cache keys include `tenant_id` prefix — no bare `user:{id}` or `record:{id}` keys for tenant data
- [ ] Background jobs carry tenant context and use tenant-scoped repository methods
- [ ] Cross-tenant FK audit query returns 0 rows in CI
- [ ] "Not found" and "access denied" return identical responses (404) — do not distinguish them

## After Completion

You have structural tenant isolation across queries, responses, cache, search, and jobs. Recommended next skills:

- Use **`cross-tenant-access-prevention`** for in-depth IDOR/BOLA enforcement patterns
- Use **`row-level-security-patterns`** to enforce isolation at the PostgreSQL policy layer
- Use **`tenant-aware-query-design`** to audit every query in the codebase for missing tenant filters

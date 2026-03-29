---
name: cross-tenant-access-prevention
description: >
  Safeguards preventing cross-tenant data access in multi-tenant SaaS systems.
  Use when: cross-tenant access, tenant data leakage, tenant boundary enforcement,
  cross-tenant prevention, authorization tenant check, tenant isolation enforcement,
  data isolation, insecure direct object reference, IDOR multi-tenant, tenant bypass,
  resource ownership check, tenant ownership validation, access control multi-tenant,
  cross-tenant query prevention, broken object-level authorization, BOLA multi-tenant,
  tenant ID spoofing, resource tenant mismatch, tenant data leak, multi-tenant authorization.
argument-hint: >
  Describe the resource type vulnerable to cross-tenant access (order, project, invoice),
  where the authorization check is missing (handler, service, or repository layer), and
  whether the issue was discovered by code review, a security report, or a production incident.
---

# Cross-Tenant Access Prevention Specialist

## When to Use

Invoke this skill when you need to:
- Implement resource-level authorization checks that verify `resource.tenant_id == request.tenant_id`
- Prevent IDOR (Insecure Direct Object Reference) attacks across tenant boundaries
- Design an authorization pattern that makes cross-tenant access structurally impossible
- Review a codebase for authorization gaps where tenant ownership is not validated
- Write tests that prove cross-tenant access is rejected at every layer
- Respond to a security report of cross-tenant data leakage

---

## Step 1 — Understand the Threat Model

**How cross-tenant access occurs:**

| Attack Pattern | Example | Root Cause |
|---|---|---|
| Direct ID access | `GET /api/orders/order-id-from-another-tenant` | Fetch by ID without tenant filter |
| URL manipulation | `GET /api/tenants/victim-tenant-id/projects` | Tenant ID from URL, not from auth token |
| Relationship traversal | `GET /api/projects/proj-id/members` — project belongs to another tenant | Ownership not validated on related resources |
| Bulk listing escape | `GET /api/orders?all=true` bypasses tenant filter | Admin flag accepted from user input |
| JWT claim manipulation | JWT signed with weak secret; tenant_id in payload altered | Weak JWT secret; not validated against DB membership |

**Priority: fix structural issues, not symptoms:**
```
Do not add "if tenant mismatch, log and return 404" checks alone.
Make it structurally impossible to query cross-tenant by:
1. Always including tenant_id in WHERE clauses (repository layer)
2. Validating resource ownership at the service layer (defense in depth)
3. Testing with property-based tests using random tenant ID pairs
```

Checklist:
- [ ] All attack patterns reviewed against the current codebase
- [ ] Root cause identified — is it a single missing check or a systematic pattern?
- [ ] Defense-in-depth planned — both repository-level filtering AND service-level ownership validation
- [ ] JWT secret strength verified — weak secret allows token forgery; fix independently

---

## Step 2 — Service-Layer Ownership Validation

Every resource fetch must validate tenant ownership before returning data.

**Authorization check pattern — fetch → validate → return:**
```go
type OrderService struct {
    repo OrderRepository
}

func (s *OrderService) Get(ctx context.Context, orderID string) (*Order, error) {
    tenant := RequireTenant(ctx)

    // Option A: Repository enforces tenant — structurally impossible to fetch cross-tenant
    order, err := s.repo.Get(ctx, tenant.ID, orderID) // tenant.ID is mandatory param
    if err != nil {
        return nil, err // ErrNotFound if cross-tenant (not ErrForbidden — don't reveal existence)
    }

    return order, nil
}

// Option B: Explicit ownership validation (defense-in-depth with fallback)
func (s *OrderService) GetWithOwnershipCheck(ctx context.Context, orderID string) (*Order, error) {
    tenant := RequireTenant(ctx)

    order, err := s.repo.GetByID(ctx, orderID) // fetches without tenant filter
    if err != nil {
        return nil, err
    }

    // Explicit ownership check — defense in depth even if repo doesn't filter
    if order.TenantID != tenant.ID {
        // Return ErrNotFound — not ErrForbidden; do not reveal resource exists in another tenant
        return nil, ErrNotFound
    }

    return order, nil
}
```

**Why return `ErrNotFound` instead of `ErrForbidden` on ownership mismatch:**
- `403 Forbidden` confirms the resource exists in another tenant — information disclosure
- `404 Not Found` is the safe response — resource does not exist for THIS tenant's context
- This pattern is called "security through obscurity" only when it's the ONLY defense; here it supplements repository-layer filtering

Checklist:
- [ ] Service methods retrieve resources scoped to `tenant.ID` — not by global ID alone
- [ ] Explicit ownership check as defense-in-depth where repository cannot enforce tenant scoping
- [ ] `ErrNotFound` returned on tenant mismatch — not `ErrForbidden`
- [ ] Related resources validated too — if fetching `project.members`, verify the project belongs to the requesting tenant first

---

## Step 3 — Never Trust Client-Supplied Tenant IDs

**Anti-patterns to eliminate:**
```go
// BAD: tenant_id from URL path — user controls this
// GET /api/tenants/{tenantID}/orders
func (h *Handler) ListOrders(w http.ResponseWriter, r *http.Request) {
    tenantID := chi.URLParam(r, "tenantID")  // attacker puts any tenant ID here
    orders, _ := h.service.List(r.Context(), tenantID)
    json.NewEncoder(w).Encode(orders)
}

// BAD: tenant_id from request body
// POST /api/orders  body: {"tenant_id": "victim-tenant", ...}
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
    var req struct {
        TenantID string `json:"tenant_id"` // NEVER accept tenant_id from client
    }
    json.NewDecoder(r.Body).Decode(&req)
    // ...
}

// GOOD: tenant always from auth context
func (h *Handler) ListOrders(w http.ResponseWriter, r *http.Request) {
    // tenant derived from validated JWT claim — user cannot alter it
    tenant := TenantFromContext(r.Context())
    orders, _ := h.service.List(r.Context()) // service calls RequireTenant(ctx)
    json.NewEncoder(w).Encode(orders)
}
```

Checklist:
- [ ] No handler extracts `tenant_id` from URL parameters, query strings, or request body
- [ ] `tenant_id` only comes from the validated JWT claim via middleware
- [ ] URL patterns like `/api/tenants/{id}/...` replaced with implicit tenant from auth — or the `{id}` validated against JWT claim if multi-tenant user context requires it

---

## Step 4 — Relationship Traversal Authorization

Accessing a resource's children requires validating the parent's ownership first.

```go
// Vulnerability: anyone with a project's ID can fetch its members, even from another tenant
// WRONG
func (s *ProjectService) ListMembers(ctx context.Context, projectID string) ([]*Member, error) {
    return s.memberRepo.FindByProject(ctx, projectID)  // no ownership check on project
}

// CORRECT: validate parent ownership before accessing children
func (s *ProjectService) ListMembers(ctx context.Context, projectID string) ([]*Member, error) {
    tenant := RequireTenant(ctx)

    // First: verify the project belongs to this tenant
    project, err := s.projectRepo.Get(ctx, tenant.ID, projectID)
    if err != nil {
        return nil, ErrNotFound // project not in this tenant
    }

    // Now safe to fetch members — project ownership validated
    return s.memberRepo.FindByProject(ctx, project.ID)
}
```

**Multi-level traversal rule:**
```
For every resource fetch involving a relationship:
  Step 1: Fetch the root ancestor resource with tenant_id filter
  Step 2: Verify it belongs to the current tenant
  Step 3: Then access its children
```

Checklist:
- [ ] Every relationship traversal validates the parent resource's tenant ownership first
- [ ] No shortcut: "the parent ID in the JWT proves the child exists" — tenants are not transitive in shared DB
- [ ] Deeply nested resources validated at each level — not only the leaf

---

## Step 5 — Cross-Tenant Access Testing

**Property-based test approach — generate random tenant pairs, assert 0 cross-tenant reads:**
```go
func TestCrossTenantAccessPrevention(t *testing.T) {
    resourceTypes := []struct {
        name    string
        create  func(tenantID string) string  // returns resource ID
        fetch   func(ctx context.Context, id string) (interface{}, error)
    }{
        {
            name:   "order",
            create: func(tid string) string { return fixtures.CreateOrder(t, tid).ID },
            fetch:  func(ctx context.Context, id string) (interface{}, error) {
                return orderService.Get(ctx, id)
            },
        },
        // ... repeat for each resource type
    }

    for _, rt := range resourceTypes {
        t.Run(rt.name, func(t *testing.T) {
            tenant1 := fixtures.CreateTenant(t)
            tenant2 := fixtures.CreateTenant(t)
            resourceID := rt.create(tenant1.ID)

            // Fetch as tenant2 — should NOT find tenant1's resource
            ctx2 := ctxWithTenant(t, tenant2)
            result, err := rt.fetch(ctx2, resourceID)

            assert.Nil(t, result, "%s: cross-tenant access returned data", rt.name)
            assert.ErrorIs(t, err, ErrNotFound, "%s: expected not-found for cross-tenant", rt.name)
        })
    }
}
```

Checklist:
- [ ] Cross-tenant access test exists for every resource type
- [ ] Tests run as part of CI — not optional; must pass before merge
- [ ] Tests validate that the response is `not-found`, not `forbidden` (information disclosure)
- [ ] All relationship traversal endpoints tested — not just top-level resource endpoints

---

## Output Report

### Critical
- Direct object fetch by ID without tenant filter — any user can access any record by guessing or enumerating IDs
- `tenant_id` accepted from URL path or request body — attacker directly specifies target tenant
- JWT secret weak (< 256-bit) — token forgery allows tenant_id claim manipulation

### High
- Relationship traversal without parent ownership check — child resources (members, files) accessible via parent ID belonging to another tenant
- `403 Forbidden` returned on tenant mismatch — reveals resource existence in another tenant (information disclosure)
- No cross-tenant access tests in test suite — isolation guaranteed only by code inspection, not automated verification

### Medium
- Defense-in-depth missing — only one layer (repository OR service) enforces tenant; regression in one layer creates a gap
- Related-resource ownership validation missing — comment says "caller must validate ownership" but no enforcement
- Admin endpoints bypass tenant filter without role check — admin authorization not verified before cross-tenant query

### Low
- URL path includes `{tenantID}` that is compared to JWT claim — extra complexity; prefer always-from-context
- ErrNotFound indistinguishable from ErrForbidden in logs — makes security audit of access denials difficult
- Cross-tenant test only covers GET — PUT/DELETE/PATCH not tested; write operations also vulnerable

### Passed
- Repository layer always includes tenant_id in WHERE clause — structurally prevents cross-tenant fetch
- Service layer additionally validates ownership: `resource.TenantID == tenant.ID`; returns ErrNotFound on mismatch
- No handler accepts tenant_id from URL, query string, or request body — always from auth context
- Relationship traversal validates parent ownership before fetching children at every level
- Cross-tenant integration tests cover all resource types and HTTP methods; run in CI; must pass before merge

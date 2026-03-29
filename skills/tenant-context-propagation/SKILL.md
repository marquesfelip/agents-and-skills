---
name: tenant-context-propagation
description: >
  Safe tenant context propagation across requests, services, and background jobs in multi-tenant SaaS.
  Use when: tenant context, context propagation, tenant_id in context, request context tenant,
  middleware tenant extraction, tenant scoped context, context leaking, tenant context leak,
  background job tenant context, async tenant context, tenant context loss, goroutine tenant context,
  context cancellation tenant, tenant context injection, tenant context validation,
  service layer tenant, repository tenant context, context.Context tenant, tenant context key,
  impersonation context, multi-tenant context, context propagation middleware, tenant header.
argument-hint: >
  Describe your context propagation problem: where tenant context is extracted (JWT, header,
  subdomain), how it flows through layers (middleware → handler → service → repository),
  and where context is lost or incorrectly inherited (background jobs, goroutines, async calls).
---

# Tenant Context Propagation Specialist

## When to Use

Invoke this skill when you need to:
- Establish tenant context from an HTTP request (JWT claim, subdomain, path prefix)
- Propagate tenant context through all application layers without passing raw strings
- Prevent tenant context from leaking into background goroutines that outlive the request
- Pass tenant context safely into background job payloads
- Validate that tenant context is present at service boundaries
- Debug missing or incorrect tenant context in a multi-tenant application

---

## Step 1 — Extract Tenant Context in Middleware

Tenant context is established once, at the request boundary, and never re-derived later.

**Typed context key — prevents collision with other context values:**
```go
// unexported type — prevents collision with any other package using string keys
type contextKey struct{ name string }

var (
    tenantCtxKey = &contextKey{"tenant"}
    userCtxKey   = &contextKey{"user"}
    actorCtxKey  = &contextKey{"actor"}
)

type TenantContext struct {
    ID     string
    Slug   string
    PlanID string
    Status string
}

func WithTenant(ctx context.Context, tenant *TenantContext) context.Context {
    return context.WithValue(ctx, tenantCtxKey, tenant)
}

// Returns nil if not set — callers must handle this case
func TenantFromContext(ctx context.Context) *TenantContext {
    t, _ := ctx.Value(tenantCtxKey).(*TenantContext)
    return t
}

// RequireTenant panics if not set — use at service layer boundaries
func RequireTenant(ctx context.Context) *TenantContext {
    t := TenantFromContext(ctx)
    if t == nil {
        panic("tenant context not set: middleware not applied or request bypassed auth")
    }
    return t
}
```

**Middleware — extracts tenant from JWT claim:**
```go
func TenantMiddleware(repo TenantRepository) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims := JWTClaimsFromContext(r.Context()) // set by auth middleware upstream
            if claims == nil {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }

            tenantID := claims.TenantID
            if tenantID == "" {
                http.Error(w, "no tenant in token", http.StatusBadRequest)
                return
            }

            tenant, err := repo.Get(r.Context(), tenantID)
            if err != nil || tenant == nil {
                http.Error(w, "tenant not found", http.StatusNotFound)
                return
            }

            // Validate the authenticated user actually belongs to this tenant
            if !repo.IsMember(r.Context(), tenantID, claims.UserID) {
                http.Error(w, "forbidden", http.StatusForbidden)
                return
            }

            ctx := WithTenant(r.Context(), &TenantContext{
                ID:     tenant.ID,
                Slug:   tenant.Slug,
                PlanID: tenant.PlanID,
                Status: tenant.Status,
            })
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

Checklist:
- [ ] Typed unexported struct key used for context — not a string key such as `"tenant_id"`
- [ ] Tenant membership validated during extraction — not just JWT claim trusted without DB check
- [ ] Tenant status validated — suspended/canceled tenants rejected at the middleware boundary
- [ ] `TenantFromContext` returns nil safely; `RequireTenant` panics — use panic version at service entry points where it's a programming error

---

## Step 2 — Propagate Through Service and Repository Layers

**Service layer — receives `context.Context`; extracts tenant:**
```go
type OrderService struct {
    repo     OrderRepository
    entitl   EntitlementService
    auditLog AuditLogger
}

func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Extract at service boundary — panics early if middleware was not applied
    tenant := RequireTenant(ctx)

    // Check entitlements using tenant from context
    if err := s.entitl.CheckLimit(ctx, tenant.ID, "orders", 1); err != nil {
        return nil, err
    }

    order := &Order{
        TenantID: tenant.ID,  // always set from context — never from request body
        // ...
    }

    if err := s.repo.Create(ctx, order); err != nil {
        return nil, err
    }

    s.auditLog.Log(ctx, AuditEntry{
        TenantID:  tenant.ID,
        Action:    "order.created",
        ResourceID: order.ID,
    })

    return order, nil
}
```

**Anti-patterns to avoid:**
```go
// BAD: tenant_id passed as a function parameter throughout the call stack
func (s *OrderService) Create(ctx context.Context, tenantID string, req CreateOrderRequest) {}

// BAD: tenant_id extracted from a global variable or request struct
func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) {
    tenantID := req.TenantID  // request body should NEVER carry tenant_id
}

// GOOD: tenant always from context
func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) {
    tenant := RequireTenant(ctx)
    // use tenant.ID throughout
}
```

Checklist:
- [ ] `tenant_id` never accepted from the request body — always derived from validated context
- [ ] `tenant_id` not passed as an explicit parameter between service methods — always via context
- [ ] Repository `tenant_id` injected at construction time or extracted from context — not passed at call time
- [ ] `RequireTenant(ctx)` used at service layer entry points — catches missing middleware early

---

## Step 3 — Goroutine Context Safety

**Problem:** `context.Context` from an HTTP request is canceled when the request ends. Goroutines that inherit the request context will be canceled when the request completes.

```go
// BAD: goroutine inherits cancellable request context
func (s *Service) Process(ctx context.Context, orderID string) {
    tenant := RequireTenant(ctx)
    go func() {
        // ctx will be canceled when the HTTP request ends — this goroutine likely fails
        s.asyncWork(ctx, tenant.ID, orderID)  // WRONG
    }()
}

// GOOD: create a detached context with tenant value copied
func (s *Service) Process(ctx context.Context, orderID string) {
    tenant := RequireTenant(ctx)

    // Detach from request lifecycle; preserve tenant value
    detachedCtx := WithTenant(context.Background(), tenant)
    // Copy other needed values (logger, trace ID) if required
    detachedCtx = withLogger(detachedCtx, loggerFromCtx(ctx))

    go func() {
        s.asyncWork(detachedCtx, orderID)
    }()
}
```

**Even better — don't spawn goroutines from request handlers:**
- Enqueue the work into a job queue with the tenant_id serialized in the payload
- The worker reconstructs the full context from the payload (see `tenant-safe-background-jobs` skill)

Checklist:
- [ ] No goroutine inherits the HTTP request's `context.Context` without detaching
- [ ] Detached contexts use `context.Background()` as the base — not the canceled request context
- [ ] Tenant value explicitly copied into the detached context — not assumed to be inherited
- [ ] Prefer job queues over goroutines for async work — cleaner lifecycle management

---

## Step 4 — Context Validation at Boundaries

Add assertions to catch context propagation bugs early.

```go
// Middleware applied to all routes that require tenant context
func AssertTenantContextMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // This should never happen if middleware is applied correctly
        // Panics in dev; returns 500 in production
        if TenantFromContext(r.Context()) == nil {
            panic("AssertTenantContextMiddleware: tenant context missing on route " + r.URL.Path)
        }
        next.ServeHTTP(w, r)
    })
}

// Test helper — verify that all protected routes have tenant context set
func TestAllProtectedRoutesHaveTenantContext(t *testing.T) {
    protectedPaths := []string{"/api/orders", "/api/projects", "/api/team"}
    for _, path := range protectedPaths {
        req := httptest.NewRequest("GET", path, nil)
        req = req.WithContext(context.Background()) // no tenant context
        rr := httptest.NewRecorder()

        router.ServeHTTP(rr, req)

        // Routes must reject requests without tenant context (401/403, not 200/500)
        assert.NotEqual(t, http.StatusOK, rr.Code, "route %s must require tenant", path)
        assert.NotEqual(t, http.StatusInternalServerError, rr.Code, "route %s must not panic", path)
    }
}
```

Checklist:
- [ ] All protected routes mounted behind `TenantMiddleware` — not opted in per handler
- [ ] Test verifies all protected routes reject requests without tenant context
- [ ] Service-layer `RequireTenant` panics in development and staging — fast feedback on missing middleware
- [ ] In production, `RequireTenant` logs and returns 500 — no silent data leakage

---

## Output Report

### Critical
- `tenant_id` accepted from request body — attacker sends any tenant_id; accesses cross-tenant data
- Goroutine inherits canceled request context — async work runs under wrong tenant or no tenant after request ends

### High
- String key used for context value (`"tenant_id"`) — any package can read or overwrite the tenant context value; collision and leakage risk
- Tenant membership not validated in middleware — JWT claim for a deleted tenant or a user who left the tenant is trusted
- `tenant_id` passed as a parameter through service layers — bypasses context; inconsistent enforcement

### Medium
- No assertion middleware — missing `TenantMiddleware` on a route is silent; discovered only when a query fails or leaks data
- Detached goroutines use `context.Background()` without copying the tenant value — business logic runs without tenant context
- `RequireTenant` returns nil instead of panicking — callers forget nil check; nil dereference at data access

### Low
- Tenant status not checked in middleware — suspended tenant continues to access the system
- Context key is an exported string constant — other packages can accidentally use the same key
- No test verifying that all protected routes carry tenant context — coverage gap discovered at runtime

### Passed
- Typed unexported struct key used for context; `TenantFromContext` returns nil; `RequireTenant` panics
- JWT claim bound to DB lookup + membership check in middleware — tenant status and user membership validated
- `tenant_id` never from request body; never passed as function parameter; always from `RequireTenant(ctx)`
- Goroutines and async jobs detach from request context; tenant value explicitly copied into detached context
- Assertion middleware applied to all protected route groups; integration test verifies all protected paths require tenant

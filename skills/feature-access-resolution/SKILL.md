---
name: feature-access-resolution
description: 'Define and implement runtime feature permission resolution for SaaS entitlements. Use when: feature access resolution, runtime feature check, entitlement resolution, feature permission check, access resolution service, entitlement service implementation, feature gate runtime, IsEnabled check, HasFeature check, feature access service, entitlement lookup, feature flag resolution, plan feature check, runtime access check, feature permission resolution, gate point implementation, access check middleware.'
argument-hint: 'Describe where feature access is checked in your system (middleware, service layer, handler), whether you use a cache layer (Redis, in-memory), and which features need resolution (boolean flags, numeric limits, tier-based capabilities).'
---

# Feature Access Resolution

## When to Use

Invoke this skill when you need to:
- Implement the service/function that resolves "can tenant X use feature Y right now?"
- Design the resolution pipeline: cache → DB → default fallback
- Build a consistent gate-point API used across handlers, services, and middleware
- Handle degraded resolution gracefully (cache miss + DB unavailable)
- Differentiate between boolean feature flags, numeric limits, and tier-based capabilities
- Instrument access resolution for observability (latency, cache hit rate, denied access)

---

## Step 1 — Resolution Service Interface

Define a single, stable interface that ALL gate points use. No handler should perform raw DB lookups for access decisions.

```go
// EntitlementResolver is the single interface for all feature access decisions.
type EntitlementResolver interface {
    // IsEnabled returns true if the boolean feature is active for the tenant.
    IsEnabled(ctx context.Context, tenantID, featureKey string) (bool, error)

    // GetLimit returns the numeric limit for a resource key.
    // Returns -1 if unlimited. Returns ErrFeatureNotFound if the key is not in the plan.
    GetLimit(ctx context.Context, tenantID, limitKey string) (int64, error)

    // GetTier returns the tier value for a tiered capability.
    // e.g., featureKey="support_level" → "basic" | "advanced" | "enterprise"
    GetTier(ctx context.Context, tenantID, capabilityKey string) (string, error)

    // Resolve returns the raw Entitlement for full inspection.
    Resolve(ctx context.Context, tenantID, featureKey string) (*Entitlement, error)
}

type Entitlement struct {
    TenantID   string
    FeatureKey string
    BoolValue  *bool
    LimitValue *int64  // nil if not a limit feature; -1 = unlimited
    TierValue  *string
    Source     string  // 'plan' | 'override' | 'trial_bonus' | 'add_on'
    ValidUntil *time.Time
}

var (
    ErrFeatureNotFound = errors.New("feature not found for tenant plan")
    ErrAccessDenied    = errors.New("feature access denied")
)
```

---

## Step 2 — Resolution Pipeline

Resolution follows a strict pipeline: **cache → DB → default**. Never skip layers or query billing directly.

```go
type entitlementResolver struct {
    cache  EntitlementCache       // Redis or in-process TTL cache
    repo   EntitlementRepository  // Reads from tenant_entitlements table
    logger *slog.Logger
}

func (r *entitlementResolver) Resolve(ctx context.Context, tenantID, featureKey string) (*Entitlement, error) {
    // 1. Cache lookup (hot path — target < 1ms)
    if ent, ok := r.cache.Get(ctx, tenantID, featureKey); ok {
        return ent, nil
    }

    // 2. DB lookup (warm path — target < 10ms)
    ent, err := r.repo.Get(ctx, tenantID, featureKey)
    if err != nil {
        if errors.Is(err, ErrFeatureNotFound) {
            // Cache negative result to avoid DB hammering on unknown keys
            r.cache.SetMissing(ctx, tenantID, featureKey, 2*time.Minute)
            return nil, ErrFeatureNotFound
        }
        // 3. Degraded path: log and return safe default (deny)
        r.logger.Warn("entitlement resolution failed, defaulting to deny",
            slog.String("tenant_id", tenantID),
            slog.String("feature_key", featureKey),
            slog.String("err", err.Error()),
        )
        return nil, err
    }

    // Populate cache for subsequent requests
    r.cache.Set(ctx, tenantID, featureKey, ent, 5*time.Minute)
    return ent, nil
}

func (r *entitlementResolver) IsEnabled(ctx context.Context, tenantID, featureKey string) (bool, error) {
    ent, err := r.Resolve(ctx, tenantID, featureKey)
    if errors.Is(err, ErrFeatureNotFound) {
        return false, nil // Not in plan = disabled; not an error
    }
    if err != nil {
        return false, err
    }
    if ent.BoolValue == nil {
        return false, fmt.Errorf("feature %q is not a boolean feature", featureKey)
    }
    return *ent.BoolValue, nil
}

func (r *entitlementResolver) GetLimit(ctx context.Context, tenantID, limitKey string) (int64, error) {
    ent, err := r.Resolve(ctx, tenantID, limitKey)
    if err != nil {
        return 0, err
    }
    if ent.LimitValue == nil {
        return 0, fmt.Errorf("feature %q is not a limit feature", limitKey)
    }
    return *ent.LimitValue, nil // -1 = unlimited
}
```

---

## Step 3 — Gate-Point Patterns

### HTTP Middleware Gate (route-level)
```go
func RequireFeature(resolver EntitlementResolver, featureKey string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := TenantFromCtx(r.Context())
            enabled, err := resolver.IsEnabled(r.Context(), tenantID, featureKey)
            if err != nil {
                http.Error(w, "entitlement check failed", http.StatusInternalServerError)
                return
            }
            if !enabled {
                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(http.StatusForbidden)
                json.NewEncoder(w).Encode(map[string]any{
                    "error": "feature_not_available",
                    "feature": featureKey,
                    "upgrade_url": "/settings/billing",
                })
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Usage:
// router.With(RequireFeature(resolver, "api_access")).Handle("/api/v1/...", handler)
```

### Service-Layer Gate (inline)
```go
func (s *ExportService) ExportToPDF(ctx context.Context, tenantID string, ...) error {
    enabled, err := s.entitlements.IsEnabled(ctx, tenantID, "pdf_export")
    if err != nil {
        return fmt.Errorf("checking export entitlement: %w", err)
    }
    if !enabled {
        return ErrFeatureNotAvailable{Feature: "pdf_export"}
    }
    // ... export logic
}
```

### Tier-Based Gate
```go
func (s *AnalyticsService) GetAdvancedReport(ctx context.Context, tenantID string) (*Report, error) {
    tier, err := s.entitlements.GetTier(ctx, tenantID, "analytics_tier")
    if err != nil {
        return nil, err
    }
    if tier != "advanced" && tier != "enterprise" {
        return nil, ErrFeatureNotAvailable{Feature: "advanced_analytics", RequiredTier: "advanced"}
    }
    // ... analytics logic
}
```

---

## Step 4 — Cache Implementation

```go
type EntitlementCache interface {
    Get(ctx context.Context, tenantID, featureKey string) (*Entitlement, bool)
    Set(ctx context.Context, tenantID, featureKey string, ent *Entitlement, ttl time.Duration)
    SetMissing(ctx context.Context, tenantID, featureKey string, ttl time.Duration)
    InvalidateTenant(ctx context.Context, tenantID string) // Called after billing transition
}

// Redis implementation key pattern:
// entitlement:{tenantID}:{featureKey}  → msgpack/JSON serialized Entitlement
// entitlement:{tenantID}:*            → scan pattern for tenant invalidation

// Bulk prefetch — optimize for tenants with many feature checks per request
func (r *entitlementResolver) Prefetch(ctx context.Context, tenantID string) error {
    entitlements, err := r.repo.GetAll(ctx, tenantID)
    if err != nil {
        return err
    }
    for _, ent := range entitlements {
        e := ent
        r.cache.Set(ctx, tenantID, ent.FeatureKey, &e, 5*time.Minute)
    }
    return nil
}
```

**TTL Guidance:**

| Cache tier | TTL | Rationale |
|---|---|---|
| Per-feature positive hit | 5 min | Fast; invalidated on billing events |
| Per-feature negative miss | 2 min | Shorter: plan may be updated |
| Full tenant prefetch | 5 min | Useful for request-start warmup |
| After billing transition | Invalidate immediately | Sync must reach DB first |

---

## Step 5 — Resolution Decision Table

| Scenario | Resolution result |
|---|---|
| Feature found in cache — value `true` | Return `true`, no DB hit |
| Feature not in cache, found in DB — `true` | Populate cache, return `true` |
| Feature not in plan (DB miss) | Cache negative miss, return `false` (not an error) |
| DB unavailable | Log warning, return error → caller defaults to deny |
| `valid_until` has passed | Return `false`; syncer should have removed the row |
| Source is `override` and has no `valid_until` | Treat as permanent override — always return value |

---

## Quality Checks

- [ ] `EntitlementResolver` interface is the ONLY entry point for access decisions — no raw DB joins for feature gating
- [ ] Cache is invalidated immediately after every billing state transition
- [ ] DB unavailability results in deny-by-default (fail closed), not allow-by-default
- [ ] Negative cache results (feature not in plan) use a shorter TTL than positive results
- [ ] Error type `ErrFeatureNotFound` is distinguished from a generic DB error at every call site
- [ ] Middleware gate returns structured JSON error with `"upgrade_url"` for UX-friendly upgrade prompts
- [ ] `Prefetch` is called at request start for latency-sensitive multi-check paths

## After Completion

You now have a full resolution pipeline. Recommended next steps:

- Use **`runtime-permission-evaluation`** to add dynamic context (user role, IP, time) to access decisions
- Use **`quota-enforcement-runtime`** to enforce numeric limits against live usage counters
- Use **`plan-capability-mapping`** to populate the `plan_features` / `plan_limits` tables that feed this resolver

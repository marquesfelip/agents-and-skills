---
name: entitlement-synchronization
description: 'Synchronization between billing state and feature access for SaaS products. Use when: entitlement synchronization, feature access billing, billing entitlement sync, feature gate billing, plan feature matrix, entitlement table, feature flag billing, subscription entitlement, billing driven access, access control billing, feature access control, plan entitlements, billing to access sync, entitlement cache, feature gate plan tier, entitlement check, plan limit feature, access revocation billing, entitlement update webhook, stripe subscription entitlement, feature access stripe, plan feature gate, entitlement service, billing access layer.'
argument-hint: 'Describe your plan tiers and the features/limits they gate (e.g., Pro = 5 seats + API access, Enterprise = unlimited), how you currently check entitlements (database, cache, feature flags), and any known sync lag between billing state and feature access.'
---

# Entitlement Synchronization

## When to Use

Invoke this skill when you need to:
- Define the plan-to-feature mapping (entitlement matrix) and store it in your DB
- Sync entitlements whenever a billing event changes the subscription state
- Implement fast, low-latency entitlement checks for API and UI gate points
- Invalidate entitlement caches on subscription changes
- Handle entitlement state during grace periods, trials, and cancellations
- Prevent entitlement drift between billing system and feature access

---

## Step 1 — Entitlement Matrix Design

Define which features and limits each plan enables:

```sql
-- Plan-level feature definitions
CREATE TABLE plan_features (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID        NOT NULL REFERENCES plans(id),
    feature_key     TEXT        NOT NULL,           -- 'api_access', 'seats', 'export', 'custom_domain'
    feature_type    TEXT        NOT NULL            -- 'boolean', 'limit', 'tier'
                                CHECK (feature_type IN ('boolean', 'limit', 'tier')),
    bool_value      BOOLEAN,                        -- for boolean features
    limit_value     BIGINT,                         -- for numeric limits (-1 = unlimited)
    tier_value      TEXT,                           -- for tier features ('basic', 'advanced', 'enterprise')
    UNIQUE (plan_id, feature_key)
);

-- Example seed data:
-- Pro plan: api_access=true, seats=5, export=true, custom_domain=false
-- Enterprise plan: api_access=true, seats=-1 (unlimited), export=true, custom_domain=true

-- Tenant's effective entitlements (derived from subscription + plan)
CREATE TABLE tenant_entitlements (
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    feature_key     TEXT        NOT NULL,
    bool_value      BOOLEAN,
    limit_value     BIGINT,
    tier_value      TEXT,
    source          TEXT        NOT NULL DEFAULT 'plan'  -- 'plan', 'override', 'trial_bonus'
                                CHECK (source IN ('plan', 'override', 'trial_bonus')),
    valid_until     TIMESTAMPTZ,                    -- NULL = permanent for current subscription
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, feature_key)
);

CREATE INDEX idx_tenant_entitlements_tenant_id ON tenant_entitlements(tenant_id);
```

---

## Step 2 — Entitlement Sync on Billing Events

Rebuild tenant entitlements whenever a billing lifecycle event changes the subscription:

```go
type EntitlementSyncer struct {
    planFeatureRepo PlanFeatureRepository
    entitlementRepo EntitlementRepository
    cacheInvalidator CacheInvalidator
    db              DB
}

func (s *EntitlementSyncer) SyncForTenant(ctx context.Context, tenantID string) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil && !errors.Is(err, ErrNotFound) {
        return err
    }

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // If no active subscription — revoke all plan-derived entitlements
        if errors.Is(err, ErrNotFound) || !sub.HasAccess() {
            if err := s.entitlementRepo.RevokePlanEntitlementsTx(ctx, tx, tenantID); err != nil {
                return err
            }
            s.cacheInvalidator.Invalidate(ctx, tenantID)
            return nil
        }

        // Load plan features
        features, err := s.planFeatureRepo.GetForPlan(ctx, sub.PlanID)
        if err != nil {
            return err
        }

        // Upsert entitlements
        for _, f := range features {
            if err := s.entitlementRepo.UpsertTx(ctx, tx, TenantEntitlement{
                TenantID:   tenantID,
                FeatureKey: f.FeatureKey,
                BoolValue:  f.BoolValue,
                LimitValue: f.LimitValue,
                TierValue:  f.TierValue,
                Source:     "plan",
                ValidUntil: nil,
            }); err != nil {
                return err
            }
        }

        // Invalidate cache after transaction commits
        return nil
    })
}

// HasAccess: determines if a subscription state grants feature access
func (sub *Subscription) HasAccess() bool {
    switch sub.Status {
    case "trialing", "active", "past_due": // past_due in grace period
        return true
    default:
        return false
    }
}
```

---

## Step 3 — Trigger Sync from Billing State Machine

Wire the syncer into the state machine's transition effects:

```go
type EntitlementTransitionEffect struct {
    syncer *EntitlementSyncer
}

func (e *EntitlementTransitionEffect) OnTransition(ctx context.Context, sub *Subscription, from, to, reason string) {
    // Access-affecting transitions
    accessChanges := map[string]bool{
        "trialing→active":   true,
        "active→past_due":   true,
        "past_due→active":   true,
        "past_due→unpaid":   true,
        "unpaid→canceled":   true, // Can also trigger as part of canceled state
        "active→canceled":   true,
        "canceled→active":   true, // Reactivation
        "paused→active":     true,
        "active→paused":     true,
    }

    key := fmt.Sprintf("%s→%s", from, to)
    if accessChanges[key] {
        // Run outside the billing transaction — after DB commit
        go func() {
            if err := e.syncer.SyncForTenant(context.Background(), sub.TenantID); err != nil {
                slog.Error("entitlement sync failed",
                    slog.String("tenant_id", sub.TenantID),
                    slog.String("transition", key),
                    slog.String("err", err.Error()),
                )
            }
        }()
    }
}
```

---

## Step 4 — Entitlement Check at Gate Points

Provide a single, consistent entitlement check API used across all services:

```go
type EntitlementService struct {
    cache EntitlementCache      // Redis or in-memory with TTL
    repo  EntitlementRepository // DB fallback
}

func (s *EntitlementService) Check(ctx context.Context, tenantID, featureKey string) (*Entitlement, error) {
    // 1. Cache lookup (fast path)
    if cached, ok := s.cache.Get(ctx, tenantID, featureKey); ok {
        return cached, nil
    }

    // 2. DB fallback
    ent, err := s.repo.Get(ctx, tenantID, featureKey)
    if err != nil {
        return nil, err
    }

    s.cache.Set(ctx, tenantID, featureKey, ent, 5*time.Minute)
    return ent, nil
}

func (s *EntitlementService) IsEnabled(ctx context.Context, tenantID, featureKey string) (bool, error) {
    ent, err := s.Check(ctx, tenantID, featureKey)
    if errors.Is(err, ErrNotFound) {
        return false, nil // Feature not in plan = disabled
    }
    if err != nil {
        return false, err
    }
    return ent.BoolValue != nil && *ent.BoolValue, nil
}

func (s *EntitlementService) GetLimit(ctx context.Context, tenantID, featureKey string) (int64, error) {
    ent, err := s.Check(ctx, tenantID, featureKey)
    if errors.Is(err, ErrNotFound) {
        return 0, nil // Feature not in plan = 0 limit
    }
    if err != nil {
        return 0, err
    }
    if ent.LimitValue == nil {
        return 0, nil
    }
    return *ent.LimitValue, nil // -1 = unlimited
}
```

---

## Step 5 — Cache Invalidation

Invalidate the entitlement cache immediately after billing transitions:

```go
type RedisEntitlementCache struct {
    client *redis.Client
}

func (c *RedisEntitlementCache) Invalidate(ctx context.Context, tenantID string) error {
    // Invalidate all feature keys for this tenant using a pattern delete
    pattern := fmt.Sprintf("ent:%s:*", tenantID)
    keys, err := c.client.Keys(ctx, pattern).Result()
    if err != nil {
        return err
    }
    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }
    return nil
}

// Cache key format: "ent:{tenant_id}:{feature_key}"
func (c *RedisEntitlementCache) key(tenantID, featureKey string) string {
    return fmt.Sprintf("ent:%s:%s", tenantID, featureKey)
}
```

---

## Step 6 — Admin Entitlement Overrides

Allow customer success to grant temporary feature access outside of plan:

```sql
-- Override entitlements with expiry
INSERT INTO tenant_entitlements (tenant_id, feature_key, bool_value, source, valid_until)
VALUES ($1, $2, true, 'override', $3)  -- $3 = expiry timestamp
ON CONFLICT (tenant_id, feature_key) DO UPDATE
    SET bool_value  = EXCLUDED.bool_value,
        source      = EXCLUDED.source,
        valid_until = EXCLUDED.valid_until,
        updated_at  = NOW();
```

```go
// Scheduled job: expire overrides
func (j *EntitlementCleanupJob) Run(ctx context.Context) error {
    _, err := j.db.ExecContext(ctx, `
        DELETE FROM tenant_entitlements
        WHERE source = 'override'
          AND valid_until IS NOT NULL
          AND valid_until < NOW()
    `)
    return err
}
```

---

## Quality Checks

- [ ] `tenant_entitlements` table is the single source of truth for runtime access checks — no hardcoded plan logic in handlers
- [ ] Entitlement sync is triggered after every billing state transition that affects access
- [ ] Cache is invalidated immediately on entitlement change — not on TTL expiry alone
- [ ] Entitlement check returns `false` (not an error) for unknown feature keys — safe default
- [ ] `limit_value = -1` correctly interpreted as unlimited throughout the codebase
- [ ] Admin overrides have an expiry date and a cleanup job
- [ ] `past_due` status grants access (grace period) — not immediately locked
- [ ] Entitlement sync runs asynchronously after DB commit — not inside billing transaction

---
name: plan-capability-mapping
description: 'Define and implement the mapping between subscription plans and system capabilities for SaaS. Use when: plan capability mapping, plan feature matrix, plan to feature mapping, plan capabilities, feature matrix design, plan features table, capability catalog, plan entitlement matrix, feature-plan mapping, SaaS plan design, plan capability catalog, plan feature grid, plan feature definition, plan capability definition, plan feature seeding, capability per plan, feature availability per plan.'
argument-hint: 'List your plan tiers (e.g., Free/Starter/Pro/Enterprise), the features you need to gate (boolean, numeric limits, tier-based), and any special cases like add-ons, trial overrides, or grandfathered plans.'
---

# Plan-to-Capability Mapping

## When to Use

Invoke this skill when you need to:
- Build or audit the complete mapping between plans and the capabilities they unlock
- Decide which capability types to use: boolean feature flags, numeric limits, or tier values
- Seed the `plan_features`, `plan_limits`, and `plan_tiers` tables correctly
- Handle special mapping cases: add-ons, overrides, grandfathered plans, trial bonuses
- Design the capability key registry to prevent naming inconsistencies
- Document the canonical plan/capability matrix for product and engineering teams

---

## Step 1 — Capability Type Classification

Before mapping, classify every capability into one of three types:

| Type | Use for | DB table | Example keys |
|---|---|---|---|
| **Boolean** | Feature on/off | `plan_features` | `sso`, `api_access`, `audit_log`, `custom_domain`, `white_label` |
| **Limit** | Numeric quota | `plan_limits` | `seats`, `projects`, `api_calls_monthly`, `storage_gb`, `exports_monthly` |
| **Tier** | Graduated levels | `plan_tiers` | `support_level`, `analytics_tier`, `retention_days`, `export_format` |

**Decision rules:**

| Question | Answer | Type |
|---|---|---|
| Is it on or off? | Always binary | Boolean |
| Can the tenant use X amount? | Has a countable ceiling | Limit |
| Are there quality/level differences? | Can be basic/advanced/enterprise | Tier |
| Is it unlimited on higher plans? | Use limit with `-1` sentinel | Limit |

---

## Step 2 — Capability Key Registry

Define all capability keys in one canonical place. This prevents drift between product docs, DB seeds, and application code.

```go
// pkg/entitlements/keys.go — single source of truth for capability key constants
package entitlements

// Boolean feature keys
const (
    FeatureSSO          = "sso"
    FeatureAuditLog     = "audit_log"
    FeatureAPIAccess    = "api_access"
    FeatureCustomDomain = "custom_domain"
    FeatureWhiteLabel   = "white_label"
    FeaturePDFExport    = "pdf_export"
    FeatureWebhooks     = "webhooks"
    FeatureSAML         = "saml"
    FeatureAdvancedRoles = "advanced_roles"
)

// Numeric limit keys
const (
    LimitSeats             = "seats"
    LimitProjects          = "projects"
    LimitAPICallsMonthly   = "api_calls_monthly"
    LimitStorageGB         = "storage_gb"
    LimitExportsMonthly    = "exports_monthly"
    LimitWorkflowsActive   = "workflows_active"
)

// Tier capability keys
const (
    TierSupport    = "support_level"    // 'community' | 'email' | 'priority' | 'dedicated'
    TierAnalytics  = "analytics_tier"  // 'basic' | 'advanced' | 'enterprise'
    TierRetention  = "data_retention"  // '30d' | '90d' | '365d' | 'unlimited'
)
```

---

## Step 3 — Define the Plan Matrix

Build the matrix as a seed file. This is the authoritative entitlement configuration for all plans.

```sql
-- ── BOOLEAN FEATURES ───────────────────────────────────────────────────────────
INSERT INTO plan_features (plan_id, feature_key, enabled) VALUES
--                              free     starter   pro       enterprise
-- sso
('free-id',       'sso',              FALSE),
('starter-id',    'sso',              FALSE),
('pro-id',        'sso',              TRUE),
('enterprise-id', 'sso',              TRUE),
-- audit_log
('free-id',       'audit_log',        FALSE),
('starter-id',    'audit_log',        FALSE),
('pro-id',        'audit_log',        TRUE),
('enterprise-id', 'audit_log',        TRUE),
-- api_access
('free-id',       'api_access',       FALSE),
('starter-id',    'api_access',       TRUE),
('pro-id',        'api_access',       TRUE),
('enterprise-id', 'api_access',       TRUE),
-- webhooks
('free-id',       'webhooks',         FALSE),
('starter-id',    'webhooks',         TRUE),
('pro-id',        'webhooks',         TRUE),
('enterprise-id', 'webhooks',         TRUE),
-- custom_domain
('free-id',       'custom_domain',    FALSE),
('starter-id',    'custom_domain',    FALSE),
('pro-id',        'custom_domain',    FALSE),
('enterprise-id', 'custom_domain',    TRUE),
-- white_label
('free-id',       'white_label',      FALSE),
('starter-id',    'white_label',      FALSE),
('pro-id',        'white_label',      FALSE),
('enterprise-id', 'white_label',      TRUE);

-- ── NUMERIC LIMITS ─────────────────────────────────────────────────────────────
INSERT INTO plan_limits (plan_id, limit_key, limit_value, is_hard_limit) VALUES
--                              free  starter  pro    enterprise(-1=unlimited)
('free-id',       'seats',                3,    TRUE),
('starter-id',    'seats',               10,    TRUE),
('pro-id',        'seats',               50,    TRUE),
('enterprise-id', 'seats',               -1,    TRUE),   -- unlimited
-- projects
('free-id',       'projects',             1,    TRUE),
('starter-id',    'projects',             5,    TRUE),
('pro-id',        'projects',            25,    TRUE),
('enterprise-id', 'projects',            -1,    TRUE),
-- api_calls_monthly
('free-id',       'api_calls_monthly',    0,    TRUE),   -- 0 = feature disabled
('starter-id',    'api_calls_monthly',   5000,  TRUE),
('pro-id',        'api_calls_monthly',  50000,  FALSE),  -- soft: overage billing
('enterprise-id', 'api_calls_monthly',   -1,    FALSE),  -- unlimited, usage tracked
-- storage_gb
('free-id',       'storage_gb',           1,    TRUE),
('starter-id',    'storage_gb',          10,    TRUE),
('pro-id',        'storage_gb',         100,    TRUE),
('enterprise-id', 'storage_gb',          -1,    FALSE);  -- unlimited, usage tracked

-- ── TIER CAPABILITIES ──────────────────────────────────────────────────────────
INSERT INTO plan_tiers (plan_id, capability_key, tier_value) VALUES
-- support_level
('free-id',       'support_level',    'community'),
('starter-id',    'support_level',    'email'),
('pro-id',        'support_level',    'priority'),
('enterprise-id', 'support_level',    'dedicated'),
-- analytics_tier
('free-id',       'analytics_tier',   'basic'),
('starter-id',    'analytics_tier',   'basic'),
('pro-id',        'analytics_tier',   'advanced'),
('enterprise-id', 'analytics_tier',   'enterprise'),
-- data_retention
('free-id',       'data_retention',   '30d'),
('starter-id',    'data_retention',   '90d'),
('pro-id',        'data_retention',   '365d'),
('enterprise-id', 'data_retention',   'unlimited');
```

---

## Step 4 — Handle Special Mapping Cases

### Add-Ons (capability purchased separately)

```sql
CREATE TABLE plan_add_ons (
    id              UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT    NOT NULL UNIQUE,   -- 'extra_seats_10', 'extra_storage_50gb'
    display_name    TEXT    NOT NULL,
    price_cents     BIGINT  NOT NULL,
    billing_interval TEXT   CHECK (billing_interval IN ('monthly','annual','one_time')),
    capability_key  TEXT    NOT NULL,           -- which limit/feature it augments
    delta_value     BIGINT                      -- for limits: additive; NULL for new boolean feature
);

-- Tenant add-on subscriptions
CREATE TABLE tenant_add_ons (
    tenant_id       UUID    NOT NULL REFERENCES tenants(id),
    add_on_id       UUID    NOT NULL REFERENCES plan_add_ons(id),
    quantity        INT     NOT NULL DEFAULT 1,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    PRIMARY KEY (tenant_id, add_on_id)
);
```

**Entitlement syncer merges add-ons additively with the base plan limit:**
```go
// When syncing limits, sum the plan value + all active add-on deltas
effectiveSeatLimit = planLimit.seats + sum(addOn.delta_value * addOn.quantity)
```

### Grandfathered Plans

```sql
-- Mark grandfathered plans as inactive + non-public (still valid for existing subscribers)
UPDATE plans SET is_active = FALSE, is_public = FALSE WHERE slug = 'legacy_pro';

-- Existing subscriptions keep plan_id pointing to legacy_pro — entitlements still resolve correctly
```

### Trial Overrides

```go
// During trial, inject trial-bonus entitlements with valid_until = trial_end
func (s *EntitlementSyncer) applyTrialBonuses(ctx context.Context, tx pgx.Tx,
    tenantID string, sub *Subscription) error {
    if !sub.IsTrialing() {
        return nil
    }
    bonuses := []TenantEntitlement{
        {TenantID: tenantID, FeatureKey: "sso",       BoolValue: ptr(true),
         Source: "trial_bonus", ValidUntil: &sub.TrialEnd},
        {TenantID: tenantID, FeatureKey: "audit_log", BoolValue: ptr(true),
         Source: "trial_bonus", ValidUntil: &sub.TrialEnd},
    }
    for _, b := range bonuses {
        if err := s.entitlementRepo.UpsertTx(ctx, tx, b); err != nil {
            return err
        }
    }
    return nil
}
```

---

## Step 5 — Validate the Matrix

Run these checks after every plan matrix change:

```sql
-- 1. Every active plan has all required capability keys
SELECT p.slug, required.key
FROM plans p
CROSS JOIN (VALUES ('seats'),('api_calls_monthly'),('storage_gb')) AS required(key)
LEFT JOIN plan_limits pl ON pl.plan_id = p.id AND pl.limit_key = required.key
WHERE p.is_active = TRUE AND pl.limit_key IS NULL;
-- Expected: 0 rows (no missing required limits)

-- 2. No duplicate capability keys per plan
SELECT plan_id, feature_key, COUNT(*) FROM plan_features
GROUP BY plan_id, feature_key HAVING COUNT(*) > 1;
-- Expected: 0 rows

-- 3. Unlimited marker consistency (-1 only on enterprise-grade plans)
SELECT p.slug, pl.limit_key
FROM plan_limits pl JOIN plans p ON p.id = pl.plan_id
WHERE pl.limit_value = -1 AND p.slug NOT LIKE '%enterprise%';
-- Review any results — unlimited on non-enterprise may be intentional but worth auditing
```

---

## Quality Checks

- [ ] Every capability key is defined as a constant — no magic strings in application code
- [ ] Matrix is stored in the database (not in Go maps, YAML configs, or Stripe metadata)
- [ ] Every active plan has all required `limit_key` rows — validated by CI query above
- [ ] `-1` is the only unlimited sentinel — no `999999` or `MAX_INT` workarounds
- [ ] Add-ons merge additively with base plan limits via the syncer
- [ ] Grandfathered plans are `is_active=FALSE, is_public=FALSE` — not deleted
- [ ] Trial bonuses use `source='trial_bonus'` and `valid_until=trial_end`
- [ ] Matrix seed file is idempotent (uses `INSERT ... ON CONFLICT DO UPDATE`)

## After Completion

You have a complete, validated capability matrix. Recommended next steps:

- Use **`feature-access-resolution`** to implement the runtime lookup service over this matrix
- Use **`entitlement-system-design`** if you need to review the three-layer model
- Use **`billing-to-access-sync`** to ensure the matrix is re-applied as entitlements on every billing event

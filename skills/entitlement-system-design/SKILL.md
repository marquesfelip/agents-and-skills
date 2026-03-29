---
name: entitlement-system-design
description: 'Design the separation between subscription, plan, and access rights in SaaS. Use when: entitlement system design, subscription plan separation, access rights model, entitlement architecture, plan vs subscription design, entitlement data model, SaaS access control design, entitlement domain model, subscription access separation, plan entitlement schema, access rights layering, entitlement service design, billing access model, entitlement layer, access rights SaaS, subscription model design, plan model design.'
argument-hint: 'Describe your current or planned plan tiers (e.g., Free/Pro/Enterprise), what dimensions grant access (plan, add-ons, overrides, trial bonuses), and whether you have a single product or multiple modules/features to gate.'
---

# Entitlement System Design

## When to Use

Invoke this skill when you need to:
- Establish the conceptual and schema-level separation between **subscription**, **plan**, and **access rights**
- Design the entitlement domain model from scratch or audit an existing one
- Decide where the source of truth for feature access lives
- Define the entities, their responsibilities, and their relationships
- Prevent the common anti-pattern of mixing billing state with access logic in the same layer

---

## Core Concept: Three-Layer Model

Entitlements are not "billing". They are the *bridge* between billing state and application behavior.

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: BILLING                                        │
│   Subscription → status, price, trial dates, billing   │
│                  period, payment state                  │
├─────────────────────────────────────────────────────────┤
│ Layer 2: PLAN / CAPABILITY CATALOG                      │
│   Plan → feature matrix, limits, tier, add-ons          │
│   (Rarely changes. Cached aggressively.)               │
├─────────────────────────────────────────────────────────┤
│ Layer 3: EFFECTIVE ENTITLEMENTS                         │
│   TenantEntitlement → what the tenant can do RIGHT NOW  │
│   (Derived from billing + plan. Synced on transitions.) │
└─────────────────────────────────────────────────────────┘
```

**Rule:** Application code ONLY reads Layer 3. It NEVER derives access directly from Layer 1 (billing status).

---

## Step 1 — Define the Subscription Entity

The subscription belongs to billing. It carries payment state, not feature access.

```sql
CREATE TABLE subscriptions (
    id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID        NOT NULL REFERENCES tenants(id),
    plan_id             UUID        NOT NULL REFERENCES plans(id),
    external_id         TEXT        UNIQUE,          -- Stripe subscription ID
    status              TEXT        NOT NULL          -- 'trialing' | 'active' | 'past_due'
                                    CHECK (status IN (
                                        'trialing','active','past_due',
                                        'unpaid','canceled','paused','incomplete'
                                    )),
    trial_start         TIMESTAMPTZ,
    trial_end           TIMESTAMPTZ,
    current_period_start TIMESTAMPTZ NOT NULL,
    current_period_end   TIMESTAMPTZ NOT NULL,
    canceled_at         TIMESTAMPTZ,
    cancel_at_period_end BOOLEAN    NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_subscriptions_active_tenant
    ON subscriptions(tenant_id)
    WHERE status NOT IN ('canceled','incomplete');
```

**Responsibilities of subscription:**
- Tracks payment lifecycle (trials, renewals, failures, cancellations)
- Stores the `plan_id` binding — NOT feature values
- Does NOT gate features directly — that is Layer 3's job

---

## Step 2 — Define the Plan / Capability Catalog

Plans define *what* access looks like. Plans are product catalog data — owned by the business, not the billing provider.

```sql
CREATE TABLE plans (
    id              UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT    NOT NULL UNIQUE,        -- 'free', 'pro', 'enterprise'
    display_name    TEXT    NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    is_public       BOOLEAN NOT NULL DEFAULT TRUE,  -- FALSE = sales-only plan
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Boolean feature entitlements per plan
CREATE TABLE plan_features (
    plan_id         UUID    NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    feature_key     TEXT    NOT NULL,   -- 'sso', 'audit_log', 'api_access', 'custom_domain'
    enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (plan_id, feature_key)
);

-- Numeric limits per plan
CREATE TABLE plan_limits (
    plan_id         UUID    NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    limit_key       TEXT    NOT NULL,   -- 'seats', 'projects', 'api_calls_monthly', 'storage_gb'
    limit_value     BIGINT  NOT NULL,   -- -1 = unlimited
    is_hard_limit   BOOLEAN NOT NULL DEFAULT TRUE,
    PRIMARY KEY (plan_id, limit_key)
);

-- Tier/capability level per plan (for tiered features)
CREATE TABLE plan_tiers (
    plan_id         UUID    NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    capability_key  TEXT    NOT NULL,   -- 'support_level', 'analytics_tier', 'export_format'
    tier_value      TEXT    NOT NULL,   -- 'basic', 'advanced', 'enterprise'
    PRIMARY KEY (plan_id, capability_key)
);
```

**Responsibilities of plan catalog:**
- Defines the entitlement matrix: which features/limits/tiers each plan includes
- Is managed by product/business — not auto-synced from billing provider
- Cached aggressively (5–30 minute TTL) — plans change infrequently

---

## Step 3 — Define Effective Entitlements (Layer 3)

The `tenant_entitlements` table is the **single source of truth** for access decisions. Application code reads ONLY from here.

```sql
CREATE TABLE tenant_entitlements (
    tenant_id       UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_key     TEXT        NOT NULL,
    -- Value union: only one is populated per row (based on feature type)
    bool_value      BOOLEAN,
    limit_value     BIGINT,
    tier_value      TEXT,
    -- Provenance
    source          TEXT        NOT NULL DEFAULT 'plan'
                                CHECK (source IN ('plan', 'override', 'trial_bonus', 'add_on')),
    valid_until     TIMESTAMPTZ,              -- NULL = valid for current subscription period
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, feature_key)
);

CREATE INDEX idx_tenant_entitlements_tenant ON tenant_entitlements(tenant_id);
```

**Sources of entitlement:**

| Source | Description | Example |
|---|---|---|
| `plan` | Derived from active subscription's plan | Pro plan → `sso=true` |
| `override` | Manual override by ops/sales | Enterprise trial with extended limit |
| `trial_bonus` | Temporary bonus during trial | Free trial with pro features |
| `add_on` | Add-on purchase layered on top of plan | +50 seats add-on |

---

## Step 4 — Define the Entitlement Resolution Order

When multiple sources apply, define precedence:

```
override > add_on > trial_bonus > plan
```

Implementation rule: the syncer merges sources in order; higher-priority sources win at the `feature_key` level.

```go
type EntitlementSource int

const (
    SourcePlan       EntitlementSource = 0
    SourceTrialBonus EntitlementSource = 1
    SourceAddOn      EntitlementSource = 2
    SourceOverride   EntitlementSource = 3 // Highest priority
)

// Merge entitlements — higher source priority wins per feature_key
func MergeEntitlements(layers [][]Entitlement) map[string]Entitlement {
    result := make(map[string]Entitlement)
    for _, layer := range layers { // layers ordered low→high priority
        for _, ent := range layer {
            result[ent.FeatureKey] = ent // later overwrites earlier
        }
    }
    return result
}
```

---

## Step 5 — Audit the Separation

Verify the three layers are correctly separated in your codebase:

| Anti-pattern | Correct pattern |
|---|---|
| `if subscription.Status == "active"` in feature gate | `entitlementSvc.IsEnabled(ctx, tenantID, "feature_key")` |
| Feature matrix hardcoded in Go structs | Feature matrix in `plan_features` / `plan_limits` DB tables |
| Access check reads from `subscriptions` table | Access check reads from `tenant_entitlements` table |
| Entitlement built per-request from billing join | Entitlement pre-computed and cached after billing transitions |
| Plan features stored in the Stripe product metadata | Plan features stored in your own `plans` + `plan_features` DB |

---

## Quality Checks

- [ ] Subscription entity contains NO feature access logic — only billing state
- [ ] Plan catalog is in your own DB — not derived at runtime from Stripe objects
- [ ] `tenant_entitlements` is the sole read path for access decisions in application code
- [ ] Entitlement sources have a defined precedence order and are enforced by the syncer
- [ ] Adding a new feature requires only: a new `plan_features` row + a gate call — no code changes to the subscription or billing layer
- [ ] Schema reviewed: no direct JOIN from application queries across subscription → plan → feature in a single access check

## After Completion

You now have a clean three-layer entitlement model. Recommended next steps:

- Use **`feature-access-resolution`** to implement the runtime resolution service
- Use **`plan-capability-mapping`** to design the full plan-to-feature matrix
- Use **`billing-to-access-sync`** to wire billing events to entitlement updates

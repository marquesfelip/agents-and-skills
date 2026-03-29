---
name: plan-limiting-enforcement
description: >
  Enforcement of subscription plan limits and quotas for SaaS products.
  Use when: plan limits, subscription limits, feature gating, plan enforcement, quota enforcement,
  plan feature matrix, limit check, hard limit, soft limit, plan upgrade gate, resource limit,
  seat limit, API limit, storage limit, plan restriction, entitlement enforcement,
  limit bypass prevention, plan limit middleware, subscription feature check, plan tier,
  billing plan enforcement, limit exceeded response, upgrade prompt, plan comparison.
argument-hint: >
  Describe your plan structure (free/starter/pro/enterprise), what limits exist (seats, API calls,
  storage, features on/off), and where limits are currently enforced (or not). Include whether
  limits are hard (reject) or soft (warn/overage billing).
---

# Plan Limiting Enforcement Specialist

## When to Use

Invoke this skill when you need to:
- Design plan limits that are enforced reliably at the API layer
- Prevent tenants from exceeding their seat, storage, or feature limits
- Distinguish between hard limits (reject) and soft limits (warn or bill overage)
- Build an entitlement/feature matrix driving access control
- Return clear, actionable error responses when a limit is exceeded
- Test that limit enforcement cannot be bypassed

---

## Step 1 — Model Plans and Entitlements

**Plans table:**
```sql
CREATE TABLE plans (
    id          UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    slug        TEXT    NOT NULL UNIQUE,   -- 'free', 'starter', 'pro', 'enterprise'
    display_name TEXT   NOT NULL,
    price_cents BIGINT  NOT NULL DEFAULT 0,
    billing_interval TEXT CHECK (billing_interval IN ('monthly','annual','one_time')),
    is_active   BOOLEAN NOT NULL DEFAULT TRUE
);

-- Plan limits — one row per limit type per plan
CREATE TABLE plan_limits (
    plan_id         UUID    NOT NULL REFERENCES plans(id),
    limit_key       TEXT    NOT NULL,   -- 'seats', 'api_calls_monthly', 'storage_gb', 'projects'
    limit_value     BIGINT  NOT NULL,   -- -1 = unlimited
    is_hard_limit   BOOLEAN NOT NULL DEFAULT TRUE,
    overage_price_cents_per_unit BIGINT DEFAULT NULL,
    PRIMARY KEY (plan_id, limit_key)
);

-- Plan features — boolean entitlements
CREATE TABLE plan_features (
    plan_id     UUID    NOT NULL REFERENCES plans(id),
    feature_key TEXT    NOT NULL,  -- 'sso', 'audit_log', 'api_access', 'custom_domain'
    enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (plan_id, feature_key)
);
```

**Example data:**
```sql
-- Seat limits by plan
INSERT INTO plan_limits VALUES
    ('free-plan-id',    'seats',              5,   TRUE,  NULL),
    ('starter-plan-id', 'seats',              25,  TRUE,  NULL),
    ('pro-plan-id',     'seats',              100, TRUE,  NULL),
    ('enterprise-id',   'seats',              -1,  TRUE,  NULL),  -- unlimited
    ('free-plan-id',    'api_calls_monthly',  1000, TRUE, NULL),
    ('pro-plan-id',     'api_calls_monthly',  50000, FALSE, 1);   -- soft: $0.01 overage

-- Feature entitlements
INSERT INTO plan_features VALUES
    ('free-plan-id',    'sso',             FALSE),
    ('pro-plan-id',     'sso',             TRUE),
    ('enterprise-id',   'sso',             TRUE),
    ('pro-plan-id',     'audit_log',       TRUE),
    ('enterprise-id',   'audit_log',       TRUE);
```

Checklist:
- [ ] Plans and limits defined in the database — not hardcoded in application code
- [ ] `-1` represents unlimited — consistent sentinel value throughout the codebase
- [ ] Hard vs. soft limits distinguished — hard = reject; soft = allow + track for billing
- [ ] Feature entitlements are boolean flags per plan — separate from numeric limits
- [ ] Plan data cached aggressively (plans rarely change) — cache TTL 5–30 minutes

---

## Step 2 — Entitlement Service

Centralize all plan limit and feature checks in one service — never check limits ad-hoc in handlers.

```go
type EntitlementService interface {
    CheckLimit(ctx context.Context, tenantID, limitKey string, delta int64) error
    HasFeature(ctx context.Context, tenantID, featureKey string) bool
    GetLimit(ctx context.Context, tenantID, limitKey string) (*Limit, error)
}

type Limit struct {
    Key         string
    MaxValue    int64  // -1 = unlimited
    CurrentUsage int64
    IsHardLimit bool
}

type LimitExceededError struct {
    LimitKey    string
    MaxValue    int64
    CurrentUsage int64
    UpgradePlanSlug string // suggest which plan to upgrade to
}

func (e *LimitExceededError) Error() string {
    return fmt.Sprintf("limit exceeded: %s (%d/%d)", e.LimitKey, e.CurrentUsage, e.MaxValue)
}

func (s *entitlementService) CheckLimit(ctx context.Context, tenantID, limitKey string, delta int64) error {
    limit, err := s.GetLimit(ctx, tenantID, limitKey)
    if err != nil {
        return err
    }
    if limit.MaxValue == -1 {
        return nil // unlimited
    }
    if !limit.IsHardLimit {
        return nil // soft limit — allow and track; billed as overage
    }
    if limit.CurrentUsage+delta > limit.MaxValue {
        return &LimitExceededError{
            LimitKey:     limitKey,
            MaxValue:     limit.MaxValue,
            CurrentUsage: limit.CurrentUsage,
            UpgradePlanSlug: s.suggestUpgrade(ctx, tenantID),
        }
    }
    return nil
}
```

Checklist:
- [ ] Single `EntitlementService` — all limit checks go through it; no ad-hoc queries in handlers
- [ ] `CheckLimit` called BEFORE any state-changing operation — not after
- [ ] `HasFeature` returns false for unknown feature keys — denied by default
- [ ] `LimitExceededError` includes upgrade suggestion — enables actionable response to user
- [ ] `CheckLimit` and the operation that increments usage happen in the same database transaction

---

## Step 3 — Enforce Limits Before State Changes

**Seat limit enforcement:**
```go
func (h *TeamHandler) InviteMember(w http.ResponseWriter, r *http.Request) {
    tenant := TenantFromContext(r.Context())

    // Check limit BEFORE creating the invitation
    if err := h.entitlements.CheckLimit(r.Context(), tenant.ID, "seats", 1); err != nil {
        var limitErr *LimitExceededError
        if errors.As(err, &limitErr) {
            h.respondLimitExceeded(w, limitErr)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    // Now safe to proceed
    invitation, err := h.teamService.InviteMember(r.Context(), tenant.ID, req.Email)
    // ...
}

func (h *Handler) respondLimitExceeded(w http.ResponseWriter, err *LimitExceededError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusPaymentRequired) // 402 — upgrade required
    json.NewEncoder(w).Encode(map[string]interface{}{
        "error":            "plan_limit_exceeded",
        "limit_key":        err.LimitKey,
        "current_usage":    err.CurrentUsage,
        "max_allowed":      err.MaxValue,
        "upgrade_plan":     err.UpgradePlanSlug,
        "upgrade_url":      "/billing/upgrade",
        "message":          fmt.Sprintf("Your plan allows %d seats. Upgrade to add more.", err.MaxValue),
    })
}
```

**Feature gate enforcement:**
```go
func (h *SSOHandler) Configure(w http.ResponseWriter, r *http.Request) {
    tenant := TenantFromContext(r.Context())

    if !h.entitlements.HasFeature(r.Context(), tenant.ID, "sso") {
        w.WriteHeader(http.StatusPaymentRequired)
        json.NewEncoder(w).Encode(map[string]string{
            "error":   "feature_not_available",
            "feature": "sso",
            "message": "SSO is available on Pro and Enterprise plans.",
            "upgrade_url": "/billing/upgrade",
        })
        return
    }
    // proceed with SSO configuration
}
```

Checklist:
- [ ] Limit checks happen before any write operation — no "undo" needed on limit violation
- [ ] `402 Payment Required` returned for limit violations — not `403` (wrong semantics)
- [ ] Response body includes current usage, max allowed, and upgrade URL — actionable
- [ ] Feature gates checked on server side — client-side hiding is UX only, not enforcement
- [ ] Limit bypass via concurrent requests mitigated with database-level constraints or optimistic lock

---

## Step 4 — Prevent Limit Bypass via Concurrency

Concurrent requests can all pass the limit check simultaneously before any usage is incremented ("TOCTOU" race).

**Safe atomic usage increment with limit check:**
```go
// Atomic increment that enforces the limit at the database level
func (r *repo) ClaimSeat(ctx context.Context, tenantID string, maxSeats int64) error {
    result, err := r.db.Exec(ctx, `
        INSERT INTO tenant_usage (tenant_id, metric_key, current_value)
        VALUES ($1, 'seats', 1)
        ON CONFLICT (tenant_id, metric_key) DO UPDATE
            SET current_value = tenant_usage.current_value + 1
            WHERE tenant_usage.current_value < $2
        RETURNING current_value`,
        tenantID, maxSeats,
    )
    if err != nil {
        return err
    }
    if result.RowsAffected() == 0 {
        return ErrSeatLimitReached
    }
    return nil
}
```

**Alternative: database constraint (for small counters):**
```sql
-- Trigger or check constraint that prevents exceeding the seat limit
-- Better: use SELECT FOR UPDATE + check in a transaction
BEGIN;
SELECT current_value FROM tenant_usage
WHERE tenant_id = $1 AND metric_key = 'seats'
FOR UPDATE;

-- Check and increment in same transaction
UPDATE tenant_usage
SET current_value = current_value + 1
WHERE tenant_id = $1 AND metric_key = 'seats'
  AND current_value < (SELECT limit_value FROM plan_limits WHERE plan_id = $2 AND limit_key = 'seats');
COMMIT;
```

Checklist:
- [ ] Concurrency-safe limit enforcement — `SELECT FOR UPDATE` or atomic `UPDATE ... WHERE current < max`
- [ ] Database-level enforcement acts as the final safety net — application-level check is an optimization
- [ ] Load tested with concurrent requests to verify no race condition allows exceeding limits

---

## Output Report

### Critical
- Limits enforced only on the frontend — server accepts any operation regardless of plan
- Limit check done after the operation — resource created, then limit checked; cleanup logic needed
- No concurrency protection — concurrent requests bypass the limit check (TOCTOU race)

### High
- Limits hardcoded in application code — changing a plan limit requires a code deployment
- Feature gates not checked on the server — premium features accessible via direct API calls
- `403 Forbidden` returned instead of `402 Payment Required` — clients cannot distinguish auth failure from limit

### Medium
- Response does not include upgrade URL or upgrade plan suggestion — user doesn't know how to resolve
- Unlimited plans represented as `999999` — breaks if a customer actually approaches that value
- Soft limits tracked but overage never billed — billing and enforcement are inconsistent

### Low
- Plan and limit data not cached — DB query on every request for limit definitions that rarely change
- No monitoring of tenants approaching their limits — no proactive upgrade prompts
- Limits not tested with concurrent load — race conditions assumed not to occur

### Passed
- Plans and limits defined in database; `-1` for unlimited; hard and soft limits distinguished
- `EntitlementService` centralizes all checks; every write operation calls it before proceeding
- Concurrency-safe: atomic `UPDATE ... WHERE current < max` or `SELECT FOR UPDATE` in transaction
- `402 Payment Required` with current usage, max allowed, and upgrade URL in response
- Feature gates server-enforced; `HasFeature` defaults to false for unknown keys

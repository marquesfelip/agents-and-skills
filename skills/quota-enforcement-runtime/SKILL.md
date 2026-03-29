---
name: quota-enforcement-runtime
description: 'Define and implement runtime quota enforcement mechanisms for SaaS plan limits. Use when: quota enforcement runtime, runtime quota check, quota enforcement implementation, quota counter, usage counter, quota gate, limit enforcement runtime, quota check before action, atomic quota increment, quota exceeded response, quota overage, hard limit enforcement, soft limit enforcement, seats quota, API quota runtime, storage quota enforcement, monthly quota reset, quota usage tracking, quota service, concurrent quota check, quota atomicity.'
argument-hint: 'Describe the quota dimensions to enforce (e.g., seats, API calls/month, storage GB, active projects), whether they are hard (reject) or soft (warn/bill), reset cadence (monthly, never), and your current usage tracking mechanism (DB, Redis, Stripe metered billing).'
---

# Quota Enforcement Runtime

## When to Use

Invoke this skill when you need to:
- Enforce numeric limits at action time (before allowing seat creation, project creation, API call, file upload)
- Implement atomic "check-then-increment" to prevent race conditions on quota counters
- Distinguish hard limits (reject over-quota actions) from soft limits (allow + track overage)
- Reset monthly/periodic counters reliably at period boundaries
- Return actionable quota-exceeded errors with current usage and plan limit
- Track per-tenant usage for billing, analytics, and upgrade prompts

---

## Step 1 — Usage Tracking Schema

```sql
-- Tenant resource usage counters (for DB-persisted quotas)
CREATE TABLE tenant_usage (
    tenant_id       UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    usage_key       TEXT        NOT NULL,   -- matches plan_limits.limit_key
    current_value   BIGINT      NOT NULL DEFAULT 0,
    period_start    DATE,                   -- NULL = non-periodic (e.g., seats)
    period_end      DATE,                   -- NULL = non-periodic
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, usage_key, COALESCE(period_start, '1970-01-01'))
);

CREATE INDEX idx_tenant_usage_tenant ON tenant_usage(tenant_id);

-- Usage event log (for audit, analytics, and overage billing)
CREATE TABLE usage_events (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    user_id         UUID,
    usage_key       TEXT        NOT NULL,
    delta           BIGINT      NOT NULL,   -- positive = consumed, negative = released
    resource_id     TEXT,                   -- optional: which resource was created/deleted
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_usage_events_tenant_period
    ON usage_events(tenant_id, usage_key, recorded_at);
```

---

## Step 2 — Quota Service Interface

```go
type QuotaService interface {
    // CheckAndIncrement atomically checks the limit and increments by delta if allowed.
    // Returns ErrQuotaExceeded if the action would exceed the hard limit.
    // For soft limits, allows and returns a SoftLimitWarning.
    CheckAndIncrement(ctx context.Context, req QuotaRequest) (*QuotaResult, error)

    // Decrement releases quota when a resource is deleted.
    Decrement(ctx context.Context, tenantID, usageKey string, delta int64, resourceID string) error

    // GetUsage returns current usage for inspection / UI display.
    GetUsage(ctx context.Context, tenantID, usageKey string) (*Usage, error)

    // ResetPeriodic resets all periodic counters whose period_end has passed.
    // Called by a scheduled job at midnight UTC.
    ResetPeriodic(ctx context.Context) error
}

type QuotaRequest struct {
    TenantID   string
    UserID     string
    UsageKey   string   // e.g., LimitSeats, LimitAPICallsMonthly
    Delta      int64    // how many units this action consumes
    ResourceID string   // for audit trail (e.g., new project ID)
}

type QuotaResult struct {
    Allowed      bool
    CurrentUsage int64
    MaxValue     int64    // -1 = unlimited
    IsHardLimit  bool
    IsOverage    bool     // true when soft limit exceeded (allowed but billed)
}

var ErrQuotaExceeded = &QuotaExceededError{}

type QuotaExceededError struct {
    UsageKey     string
    CurrentUsage int64
    MaxValue     int64
    UpgradePlan  string
}

func (e *QuotaExceededError) Error() string {
    return fmt.Sprintf("quota exceeded: %s (%d/%d)", e.UsageKey, e.CurrentUsage, e.MaxValue)
}
```

---

## Step 3 — Atomic Check-and-Increment (PostgreSQL)

Race-condition-safe enforcement using a single atomic DB operation:

```go
func (s *quotaService) CheckAndIncrement(ctx context.Context, req QuotaRequest) (*QuotaResult, error) {
    // 1. Resolve the plan limit (cached)
    limitValue, isHard, err := s.entitlements.GetLimitWithPolicy(ctx, req.TenantID, req.UsageKey)
    if err != nil {
        return nil, fmt.Errorf("resolving quota limit: %w", err)
    }

    // 2. Unlimited — skip all enforcement
    if limitValue == -1 {
        if err := s.recordUsage(ctx, req, true); err != nil {
            return nil, err
        }
        return &QuotaResult{Allowed: true, MaxValue: -1, CurrentUsage: req.Delta}, nil
    }

    // 3. Atomic check + increment using DB CAS
    var newValue int64
    err = s.db.QueryRowContext(ctx, `
        INSERT INTO tenant_usage (tenant_id, usage_key, current_value, period_start, period_end, last_updated_at)
        VALUES ($1, $2, $3, $4, $5, NOW())
        ON CONFLICT (tenant_id, usage_key, COALESCE(period_start, '1970-01-01'))
        DO UPDATE SET
            current_value   = tenant_usage.current_value + $3,
            last_updated_at = NOW()
        WHERE ($6 = -1 OR tenant_usage.current_value + $3 <= $6)   -- hard limit guard
        RETURNING current_value
    `, req.TenantID, req.UsageKey, req.Delta,
       s.currentPeriodStart(req.UsageKey),
       s.currentPeriodEnd(req.UsageKey),
       limitValue,
    ).Scan(&newValue)

    if errors.Is(err, sql.ErrNoRows) {
        // WHERE clause rejected — hard limit would be exceeded
        current, _ := s.getCurrentUsage(ctx, req.TenantID, req.UsageKey)
        return &QuotaResult{Allowed: false, CurrentUsage: current, MaxValue: limitValue, IsHardLimit: true},
            &QuotaExceededError{
                UsageKey: req.UsageKey, CurrentUsage: current, MaxValue: limitValue,
            }
    }
    if err != nil {
        return nil, fmt.Errorf("atomic quota increment: %w", err)
    }

    // 4. Soft limit check (allowed but flag overage)
    isOverage := !isHard && newValue > limitValue
    s.recordUsage(ctx, req, true) // nolint: errcheck — best-effort audit

    return &QuotaResult{
        Allowed:      true,
        CurrentUsage: newValue,
        MaxValue:     limitValue,
        IsHardLimit:  isHard,
        IsOverage:    isOverage,
    }, nil
}
```

---

## Step 4 — Quota Enforcement Middleware

```go
// QuotaMiddleware checks quota BEFORE processing the request.
// Use for resource-creation endpoints (POST /projects, POST /members, etc.)
func QuotaMiddleware(quota QuotaService, usageKey string, delta int64) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := TenantFromCtx(r.Context())
            userID   := UserFromCtx(r.Context()).ID

            result, err := quota.CheckAndIncrement(r.Context(), QuotaRequest{
                TenantID: tenantID,
                UserID:   userID,
                UsageKey: usageKey,
                Delta:    delta,
            })
            if err != nil {
                var qerr *QuotaExceededError
                if errors.As(err, &qerr) {
                    w.Header().Set("Content-Type", "application/json")
                    w.WriteHeader(http.StatusPaymentRequired) // 402
                    json.NewEncoder(w).Encode(map[string]any{
                        "error":         "quota_exceeded",
                        "usage_key":     qerr.UsageKey,
                        "current_usage": qerr.CurrentUsage,
                        "max_value":     qerr.MaxValue,
                        "upgrade_url":   "/settings/billing",
                    })
                    return
                }
                http.Error(w, "quota check failed", http.StatusInternalServerError)
                return
            }

            // Propagate quota result to handler (for soft limit warnings in response headers)
            if result.IsOverage {
                w.Header().Set("X-Quota-Overage", "true")
                w.Header().Set("X-Quota-Usage", fmt.Sprintf("%d/%d", result.CurrentUsage, result.MaxValue))
            }

            // IMPORTANT: if the handler fails, decrement the reserved quota
            rw := newResponseWriter(w)
            next.ServeHTTP(rw, r)
            if rw.statusCode >= 400 {
                quota.Decrement(r.Context(), tenantID, usageKey, delta, "") //nolint: errcheck
            }
        })
    }
}
```

---

## Step 5 — Periodic Counter Reset

```go
// Scheduled job — runs daily at 00:05 UTC
func (s *quotaService) ResetPeriodic(ctx context.Context) error {
    now := time.Now().UTC()
    result, err := s.db.ExecContext(ctx, `
        UPDATE tenant_usage
        SET    current_value   = 0,
               period_start    = $1,
               period_end      = $2,
               last_updated_at = NOW()
        WHERE  period_end IS NOT NULL
          AND  period_end < CURRENT_DATE
    `, firstDayOfMonth(now), lastDayOfMonth(now))
    if err != nil {
        return fmt.Errorf("resetting periodic quotas: %w", err)
    }
    n, _ := result.RowsAffected()
    slog.Info("periodic quota reset complete", slog.Int64("rows_reset", n))
    return nil
}
```

---

## Step 6 — Quota Decision Table

| Scenario | Action |
|---|---|
| `limit_value = -1` (unlimited) | Record usage, return `Allowed=true` — skip all checks |
| Usage + delta ≤ max (hard limit) | Atomically increment, return `Allowed=true` |
| Usage + delta > max (hard limit) | No increment, return `Allowed=false` + HTTP 402 |
| Usage + delta > max (soft limit) | Increment, return `Allowed=true, IsOverage=true` + X-Quota-Overage header |
| Handler returns 4xx/5xx after increment | Decrement to release reserved quota |
| Monthly reset missed (grace) | Reset job is idempotent — re-run resets any expired rows |

---

## Quality Checks

- [ ] Check-and-increment is a single atomic DB statement — no separate read-then-write
- [ ] Hard limit rejection returns HTTP 402 with usage details and upgrade URL
- [ ] Soft limit overage is flagged in the response header without rejecting the request
- [ ] Quota is decremented when the consuming action fails (rollback on handler error)
- [ ] Unlimited (`-1`) fast-paths through all checks without DB overhead
- [ ] Periodic reset job is idempotent — no double-reset risk if run multiple times
- [ ] `usage_events` table captures every increment/decrement for audit and overage billing
- [ ] Quota service is the ONLY write path to `tenant_usage` — no direct SQL elsewhere

## After Completion

You have a complete runtime quota enforcement mechanism. Recommended next steps:

- Use **`billing-to-access-sync`** to reset or adjust quotas when plan changes occur
- Use **`usage-based-billing`** to report overage events to Stripe for metered billing
- Use **`plan-capability-mapping`** to review and adjust the `plan_limits` matrix that drives enforcement

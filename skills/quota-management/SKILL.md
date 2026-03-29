---
name: quota-management
description: >
  Resource quota tracking and enforcement for SaaS products.
  Use when: quota management, resource quota, usage tracking, quota counter, API quota,
  storage quota, seat quota, request quota, quota reset, monthly quota, daily quota,
  quota enforcement, quota overage, quota alert, quota exceeded, quota billing, quota monitoring,
  usage meter, metered billing, quota accumulation, quota rollover, quota enforcement point,
  quota atomicity, quota increment, quota decrement, quota audit, quota dashboard, usage report.
argument-hint: >
  Describe the resource being quota-tracked (API calls, storage bytes, active seats, exports),
  the reset schedule (monthly, daily, never), and whether overages are blocked (hard limit) or
  billed (soft limit / metered billing). Include current quota tracking approach if any.
---

# Quota Management Specialist

## When to Use

Invoke this skill when you need to:
- Design quota counters for API calls, storage, seats, or feature usage
- Implement quota reset on billing cycle boundaries
- Track usage accurately under concurrent writes without allowing bypass
- Alert tenants when approaching their quota limits
- Support metered billing for soft-quota resources
- Build a usage dashboard for tenants and for internal ops

---

## Step 1 — Design the Quota Data Model

**Quota usage table — the running counter:**
```sql
CREATE TABLE tenant_quota_usage (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    metric_key      TEXT        NOT NULL,   -- 'api_calls', 'storage_bytes', 'seats', 'exports'
    period_start    TIMESTAMPTZ NOT NULL,   -- start of the quota period
    period_end      TIMESTAMPTZ NOT NULL,   -- end of the quota period
    current_value   BIGINT      NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, metric_key, period_start)
);

CREATE INDEX idx_quota_tenant_metric ON tenant_quota_usage(tenant_id, metric_key, period_start);
```

**Quota period alignment:**

| Quota Type | Period Start | Period End |
|---|---|---|
| Monthly (billing cycle) | `subscription.current_period_start` | `subscription.current_period_end` |
| Calendar month | `date_trunc('month', NOW())` | `date_trunc('month', NOW()) + INTERVAL '1 month'` |
| Daily | `date_trunc('day', NOW())` | `date_trunc('day', NOW()) + INTERVAL '1 day'` |
| Lifetime | Tenant `created_at` | NULL (no reset) |

**Always align monthly quota to the billing cycle, not the calendar month** — a tenant who upgrades mid-month should not hit a pro-rated calendar limit.

Checklist:
- [ ] Quota table keyed on `(tenant_id, metric_key, period_start)` — unique per tenant, resource, period
- [ ] Period boundaries derived from the subscription billing cycle — not arbitrary calendar dates
- [ ] Lifetime quotas (seats) use a separate row type or a `NULL` period_end sentinel
- [ ] One quota row per period — new period created at reset, old row archived not deleted

---

## Step 2 — Atomic Quota Increment

Under concurrent loads, multiple requests may try to increment the quota simultaneously. The counter must be atomic.

**Atomic increment with limit check:**
```go
type QuotaService interface {
    // Increment increments usage by delta. Returns ErrQuotaExceeded if the result
    // would exceed the hard limit. For soft limits, increments and returns nil.
    Increment(ctx context.Context, tenantID, metricKey string, delta int64) (*QuotaResult, error)
    Current(ctx context.Context, tenantID, metricKey string) (*QuotaResult, error)
}

type QuotaResult struct {
    MetricKey    string
    CurrentValue int64
    MaxValue     int64   // -1 = unlimited
    IsHardLimit  bool
    PeriodEnd    time.Time
}

// Atomic increment — single SQL round-trip
func (s *pgQuotaService) Increment(ctx context.Context, tenantID, metricKey string, delta int64) (*QuotaResult, error) {
    limit, err := s.planLimits.Get(ctx, tenantID, metricKey)
    if err != nil {
        return nil, err
    }

    periodStart := s.currentPeriodStart(ctx, tenantID)

    var result QuotaResult
    err = s.db.QueryRow(ctx, `
        INSERT INTO tenant_quota_usage (tenant_id, metric_key, period_start, period_end, current_value)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (tenant_id, metric_key, period_start) DO UPDATE
            SET current_value   = tenant_quota_usage.current_value + $5,
                last_updated_at = NOW()
        RETURNING current_value`,
        tenantID, metricKey, periodStart, s.currentPeriodEnd(ctx, tenantID), delta,
    ).Scan(&result.CurrentValue)
    if err != nil {
        return nil, err
    }

    result.MetricKey = metricKey
    result.MaxValue = limit.MaxValue
    result.IsHardLimit = limit.IsHardLimit

    if limit.IsHardLimit && limit.MaxValue != -1 && result.CurrentValue > limit.MaxValue {
        // Roll back the increment
        s.db.Exec(ctx, `
            UPDATE tenant_quota_usage
            SET current_value = current_value - $1
            WHERE tenant_id = $2 AND metric_key = $3 AND period_start = $4`,
            delta, tenantID, metricKey, periodStart,
        )
        return nil, &QuotaExceededError{
            MetricKey:    metricKey,
            CurrentValue: result.CurrentValue - delta,
            MaxValue:     limit.MaxValue,
        }
    }

    return &result, nil
}
```

**Better pattern — check-then-increment in one atomic UPDATE:**
```go
// For hard limits: atomic single-statement approach
err = s.db.QueryRow(ctx, `
    WITH updated AS (
        INSERT INTO tenant_quota_usage (tenant_id, metric_key, period_start, period_end, current_value)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (tenant_id, metric_key, period_start) DO UPDATE
            SET current_value = tenant_quota_usage.current_value + $5
            WHERE tenant_quota_usage.current_value + $5 <= $6
                OR $6 = -1  -- unlimited
        RETURNING current_value
    )
    SELECT current_value FROM updated`,
    tenantID, metricKey, periodStart, periodEnd, delta, limit.MaxValue,
).Scan(&newValue)

if errors.Is(err, pgx.ErrNoRows) {
    return nil, ErrQuotaExceeded // WHERE clause rejected the update
}
```

Checklist:
- [ ] Quota increment is a single atomic database operation — no read-then-write pattern
- [ ] Hard limit check included in the `WHERE` clause of the `UPDATE` — no TOCTOU race
- [ ] `INSERT ... ON CONFLICT DO UPDATE` used — handles first-use initialization of the counter
- [ ] Quota decrement (releases) also handled — e.g., deleting a seat decrements the counter

---

## Step 3 — Quota Reset

**Quota reset on billing period rollover:**
```go
// Option A: Lazy reset — detect old period on read/increment; create new row for new period
// This is simpler — no scheduled job needed
func (s *pgQuotaService) currentPeriodStart(ctx context.Context, tenantID string) time.Time {
    sub, _ := s.subRepo.GetActive(ctx, tenantID)
    return sub.CurrentPeriodStart  // directly from subscription
}

// Option B: Proactive reset via scheduled job (for metered billing tally export)
// Run just after billing provider's invoice is finalized
func (s *ResetJob) ResetExpiredPeriods(ctx context.Context) error {
    _, err := s.db.Exec(ctx, `
        -- Archive expired quota rows; new period starts fresh via lazy init
        UPDATE tenant_quota_usage
        SET archived_at = NOW()
        WHERE period_end < NOW()
          AND archived_at IS NULL`)
    return err
}
```

**Never delete quota rows — archive them:**
- Historical quota rows are needed for billing reconciliation, audit, and dispute resolution
- Add `archived_at TIMESTAMPTZ NULL` column; queries filter `WHERE archived_at IS NULL`

Checklist:
- [ ] Lazy period initialization preferred — avoids a scheduled job and handles new tenants automatically
- [ ] Quota reset aligned to billing cycle — `subscription.current_period_start` from webhook sync
- [ ] Old quota rows archived, not deleted — needed for billing reconciliation
- [ ] Quota reset tested — tenant correctly starts at 0 after period rollover

---

## Step 4 — Quota Approaching Alerts

Proactive alerts at 80% and 100% prevent surprise limit hits and drive upgrade conversations.

**Threshold notification triggers:**
```go
type ThresholdEvent struct {
    TenantID   string
    MetricKey  string
    UsagePct   float64  // 0.80, 1.0, etc.
    Current    int64
    Max        int64
}

func (s *quotaService) checkThresholds(ctx context.Context, tenantID, metricKey string, result *QuotaResult) {
    if result.MaxValue <= 0 {
        return // unlimited — no threshold to check
    }
    pct := float64(result.CurrentValue) / float64(result.MaxValue)

    thresholds := []float64{0.80, 0.90, 1.0}
    for _, threshold := range thresholds {
        if pct >= threshold && !s.alreadyNotified(ctx, tenantID, metricKey, threshold) {
            s.notifier.Notify(ctx, ThresholdEvent{
                TenantID:  tenantID,
                MetricKey: metricKey,
                UsagePct:  pct,
                Current:   result.CurrentValue,
                Max:       result.MaxValue,
            })
            s.markNotified(ctx, tenantID, metricKey, threshold)
        }
    }
}
```

**Notification deduplication — only notify once per threshold per period:**
```sql
CREATE TABLE quota_threshold_notifications (
    tenant_id       UUID        NOT NULL,
    metric_key      TEXT        NOT NULL,
    threshold_pct   NUMERIC     NOT NULL,   -- 0.80, 0.90, 1.00
    period_start    TIMESTAMPTZ NOT NULL,
    notified_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, metric_key, threshold_pct, period_start)
);
```

Checklist:
- [ ] Threshold alerts fire at 80%, 90%, and 100% of the quota
- [ ] Each threshold notified at most once per quota period — no repeated emails
- [ ] Notification includes current usage, limit, and upgrade link
- [ ] Alert resets with the quota period — tenant is re-notified in the following period if they hit it again

---

## Step 5 — Usage Reporting API

Tenants need visibility into their quota consumption.

```go
// GET /api/quotas — returns current usage for all metrics
type QuotaUsageResponse struct {
    Metrics []MetricUsage `json:"metrics"`
    PeriodStart time.Time `json:"period_start"`
    PeriodEnd   time.Time `json:"period_end"`
}

type MetricUsage struct {
    Key         string  `json:"key"`            // "api_calls"
    DisplayName string  `json:"display_name"`   // "API Calls"
    Current     int64   `json:"current"`
    Max         int64   `json:"max"`            // -1 = unlimited
    UsagePct    float64 `json:"usage_pct"`      // 0.0–1.0; null if unlimited
    ResetAt     *time.Time `json:"reset_at"`
}
```

Checklist:
- [ ] Quota usage API returns current, max, and reset date for every tracked metric
- [ ] API response includes `usage_pct` — UI can render a progress bar without math
- [ ] Response indicates which limits are hard vs. soft — UI shows different messaging
- [ ] Internal ops dashboard shows all tenants approaching limits — proactive outreach opportunity

---

## Output Report

### Critical
- No atomic quota increment — concurrent requests all increment past the limit (TOCTOU race)
- Quota rows deleted on reset — historical usage lost; billing reconciliation and disputes impossible

### High
- Quota period aligned to calendar month, not billing cycle — misalignment causes confusing limit resets
- Quota not decremented when resources are deleted — counter grows indefinitely even as usage drops
- No quota usage API — tenants have no visibility into their consumption

### Medium
- Quota threshold notifications not deduplicated — tenants receive repeated emails every time they hit the limit
- Quota not enforced at the database layer — application-level check can be bypassed under concurrent load
- Soft-limit overages tracked but never exported to billing provider — revenue lost on metered resources

### Low
- Unlimited quota represented as `0` or `999999` instead of `-1` — inconsistent sentinel causes logic errors at edge cases
- No internal quota monitoring dashboard — ops team blind to tenants approaching limits
- Quota history archived correctly but not indexed — slow reconciliation queries after months of data

### Passed
- Atomic `INSERT ... ON CONFLICT DO UPDATE WHERE current + delta <= max` for hard limits
- Period aligned to billing cycle from subscription data; lazy initialization handles new periods
- Old quota rows archived with `archived_at`, not deleted; usable for billing reconciliation
- Threshold notifications fire at 80%, 90%, 100%; deduplicated per period via notification log
- `/api/quotas` endpoint returns current, max, usage_pct, and reset date for all tracked metrics

---
name: usage-based-billing
description: 'Metered billing and usage tracking for SaaS products using Stripe. Use when: usage based billing, metered billing, usage tracking stripe, stripe metered price, usage record stripe, stripe usage report, consumption billing, pay as you go billing, stripe meter, usage aggregation, billing meter stripe, usage cap, usage alert, usage overage, stripe usage record create, metered subscription item, usage based pricing, API call billing, seat usage billing, data usage billing, storage usage billing, usage reset, usage rollup, usage reporting stripe, stripe billing meter, event based billing, tiered usage pricing.'
argument-hint: 'Describe what you are metering (API calls, seats, storage GB, events), how you aggregate usage (sum, max, last_during_period), your billing period reset strategy, and whether you use Stripe Meters (new) or legacy usage records.'
---

# Usage-Based Billing

## When to Use

Invoke this skill when you need to:
- Set up Stripe Meters or legacy usage records for consumption-based pricing
- Aggregate usage events and report them to Stripe within billing periods
- Handle usage caps, soft limits, and overage billing
- Track per-tenant usage in your own database for UI dashboards
- Reset usage counters at billing period boundaries
- Send usage alerts when customers approach their limits

---

## Step 1 — Choose the Stripe API: Meters vs Legacy Usage Records

| Feature | Stripe Meters (new, recommended) | Legacy Usage Records |
|---|---|---|
| API style | Event-based (`stripe meters` resource) | Record per subscription item |
| Accuracy | High (deduplication support) | Manual deduplication needed |
| Real-time visible | Yes (in Dashboard) | Limited |
| Availability | GA as of 2024 | Deprecated path |
| Deduplication | Idempotency key per event | Manual |

**Use Stripe Meters** for new implementations.

---

## Step 2 — Stripe Meter Setup

**Create a Meter (once, in setup code or Stripe Dashboard):**
```go
import "github.com/stripe/stripe-go/v76/billing/meter"

// One-time: create the meter
m, err := meter.New(&stripe.BillingMeterParams{
    DisplayName:   stripe.String("API Calls"),
    EventName:     stripe.String("api_call"),    // Must match event_name when reporting
    DefaultAggregation: &stripe.BillingMeterDefaultAggregationParams{
        Formula: stripe.String("sum"),           // sum | count | max | last_during_period
    },
    CustomerMapping: &stripe.BillingMeterCustomerMappingParams{
        EventPayloadKey: stripe.String("stripe_customer_id"),
        Type:            stripe.String("by_id"),
    },
    ValueSettings: &stripe.BillingMeterValueSettingsParams{
        EventPayloadKey: stripe.String("value"), // field in event payload carrying usage quantity
    },
})
```

**Price using the Meter:**
```go
// Create a metered Price linked to the Meter ID
p, err := price.New(&stripe.PriceParams{
    Product:  stripe.String(productID),
    Currency: stripe.String("usd"),
    Recurring: &stripe.PriceRecurringParams{
        Interval:  stripe.String("month"),
        UsageType: stripe.String("metered"),
        Meter:     stripe.String(m.ID),
    },
    BillingScheme: stripe.String("per_unit"),
    UnitAmount:    stripe.Int64(1), // $0.01 per unit (1 cent per API call)
})
```

---

## Step 3 — Reporting Usage Events

Report usage to Stripe as billing events occur:

```go
import "github.com/stripe/stripe-go/v76/billing/meterevent"

func (u *UsageReporter) ReportAPICall(ctx context.Context, tenantID string, callCount int) error {
    tenant, err := u.tenantRepo.Get(ctx, tenantID)
    if err != nil {
        return err
    }

    // Idempotency: use a deterministic key for deduplication
    idempotencyKey := fmt.Sprintf("api_call_%s_%d", tenantID, time.Now().Truncate(time.Minute).Unix())

    _, err = meterevent.New(&stripe.BillingMeterEventParams{
        EventName: stripe.String("api_call"),
        Payload: map[string]string{
            "stripe_customer_id": tenant.StripeCustomerID,
            "value":              strconv.Itoa(callCount),
        },
        Identifier: stripe.String(idempotencyKey), // deduplication key
    })
    if err != nil {
        return fmt.Errorf("report usage to stripe: %w", err)
    }

    return nil
}
```

**Batching strategy — report usage in batches, not per-request:**
```go
// Buffer usage increments in Redis or local counter
// Flush every minute via a background job
func (j *UsageFlushJob) Run(ctx context.Context) error {
    buckets, err := j.usageBuffer.DrainAll(ctx)
    if err != nil {
        return err
    }

    for tenantID, count := range buckets {
        if err := j.reporter.ReportAPICall(ctx, tenantID, count); err != nil {
            slog.Error("usage flush failed",
                slog.String("tenant_id", tenantID),
                slog.Int("count", count),
                slog.String("err", err.Error()),
            )
            // Re-enqueue on failure — don't lose usage
            j.usageBuffer.Add(ctx, tenantID, count)
        }
    }
    return nil
}
```

---

## Step 4 — Local Usage Tracking (for dashboards and limits)

Track usage in your own database — Stripe is the billing source of truth but too slow for real-time limit checks:

```sql
-- Usage accumulation per billing period
CREATE TABLE tenant_usage (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    metric          TEXT        NOT NULL,   -- 'api_calls', 'storage_gb', 'seats'
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    quantity        BIGINT      NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, metric, period_start)
);

CREATE INDEX idx_tenant_usage_tenant_period ON tenant_usage(tenant_id, metric, period_start);
```

**Atomic usage increment (PostgreSQL UPSERT):**
```sql
INSERT INTO tenant_usage (tenant_id, metric, period_start, period_end, quantity)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (tenant_id, metric, period_start)
DO UPDATE SET
    quantity   = tenant_usage.quantity + EXCLUDED.quantity,
    updated_at = NOW();
```

---

## Step 5 — Usage Caps and Alerts

Enforce hard caps and notify customers approaching their limit:

```go
func (u *UsageService) CheckAndEnforce(ctx context.Context, tenantID, metric string, increment int) error {
    plan, err := u.planRepo.GetForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    limit := plan.GetLimit(metric) // e.g., 10_000 API calls/month; -1 = unlimited

    if limit < 0 {
        return nil // Unlimited plan
    }

    current, err := u.usageRepo.GetCurrentPeriod(ctx, tenantID, metric)
    if err != nil {
        return err
    }

    // Hard cap enforcement
    if current+int64(increment) > int64(limit) {
        return &UsageLimitExceededError{
            Metric:  metric,
            Current: current,
            Limit:   int64(limit),
        }
    }

    // Soft alert: 80% threshold
    pct := float64(current+int64(increment)) / float64(limit)
    if pct >= 0.80 && pct-float64(increment)/float64(limit) < 0.80 {
        u.notifier.SendUsageAlert(ctx, tenantID, metric, pct)
    }

    return nil
}
```

---

## Step 6 — Usage Reset at Billing Period Boundary

Reset local usage counters when the billing period renews (driven by `invoice.paid`):

```go
func (j *UsageResetJob) HandleRenewal(ctx context.Context, tenantID string, newPeriodStart, newPeriodEnd time.Time) error {
    metrics := []string{"api_calls", "storage_gb", "exports"}

    return j.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        for _, metric := range metrics {
            if err := j.usageRepo.ResetPeriodTx(ctx, tx, tenantID, metric, newPeriodStart, newPeriodEnd); err != nil {
                return err
            }
        }
        return nil
    })
}
```

---

## Quality Checks

- [ ] Stripe Meter created with the correct `formula` (sum/count/max) for the metric type
- [ ] Usage events reported with a deterministic `Identifier` for deduplication
- [ ] Usage is batched and flushed periodically — not reported on every request
- [ ] Local `tenant_usage` table tracks usage atomically via UPSERT — no lost increments
- [ ] Hard limit check happens before incrementing usage — rejected before Stripe is updated
- [ ] 80% usage threshold alert sent once per billing period — not on every request
- [ ] Usage counters reset when `invoice.paid` fires at the start of the new billing period
- [ ] Usage data retained for the billing history period (at minimum 90 days for dispute reference)

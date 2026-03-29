---
name: billing-observability
description: 'Monitoring of billing events and anomalies for SaaS products. Use when: billing observability, billing monitoring, billing event tracking, billing anomaly detection, billing alert, stripe webhook monitoring, payment failure alert, revenue metric, MRR monitoring, churn alert, failed payment rate, billing event log, invoice monitoring, subscription event metrics, billing error alert, billing event anomaly, dunning monitoring, payment success rate, billing pipeline health, subscription event tracking, billing event tracing, mrr drop alert, payment spike alert, stripe event monitoring.'
argument-hint: 'Describe the billing provider (Stripe, Paddle, Chargebee), the key billing events to monitor (payment failures, subscription cancellations, plan downgrades), the alerting destination (PagerDuty, Slack, email), and whether you need MRR/ARR tracking or just operational health.'
---

# Billing Observability

## When to Use

Invoke this skill when you need to:
- Monitor the billing pipeline health: are Stripe webhooks being processed? Are payments succeeding?
- Alert on abnormal billing patterns: sudden payment failure spike, MRR drop, cancellation surge
- Log all billing events with tenant context for audit and debugging
- Track key revenue metrics: MRR, ARR, trial conversion rate, churn rate, ARPU
- Detect stuck billing workflows: invoices that never finalized, subscriptions in limbo
- Verify idempotent webhook processing: detect duplicate billing event handling

---

## Step 1 — Billing Event Log

```go
// Every billing event (inbound from Stripe or outbound state change) is logged at INFO level
// with structured fields for queryability

func (h *StripeWebhookHandler) HandleSubscriptionUpdated(ctx context.Context, event stripe.Event) error {
    var sub stripe.Subscription
    if err := json.Unmarshal(event.Data.Raw, &sub); err != nil {
        return fmt.Errorf("%w: parse subscription", ErrUnrecoverable)
    }

    tenantID := sub.Metadata["tenant_id"]
    ctx = logctx.WithTenantID(ctx, tenantID)

    slog.InfoContext(ctx, "billing event received",
        "event_id", event.ID,
        "event_type", string(event.Type),
        "tenant_id", tenantID,
        "subscription_id", sub.ID,
        "status", string(sub.Status),
        "plan_id", sub.Items.Data[0].Price.ID,
        "current_period_end", sub.CurrentPeriodEnd,
    )

    // Process the event...

    slog.InfoContext(ctx, "billing event processed",
        "event_id", event.ID,
        "event_type", string(event.Type),
        "tenant_id", tenantID,
        "duration_ms", time.Since(start).Milliseconds(),
    )
    return nil
}
```

---

## Step 2 — Billing Metrics Registration

```go
package billing_metrics

import "github.com/prometheus/client_golang/prometheus"

var (
    // Operational health: webhook processing rate
    WebhookEventsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "saas_billing_webhook_events_total",
            Help: "Total Stripe webhook events received, by type and outcome.",
        },
        []string{"event_type", "outcome"}, // outcome: "processed" | "duplicate" | "failed"
    )

    // Payment outcomes
    PaymentOutcomesTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "saas_billing_payment_outcomes_total",
            Help: "Payment success and failure counts.",
        },
        []string{"outcome", "failure_code"}, // outcome: "succeeded" | "failed"
    )

    // Subscription state transitions
    SubscriptionTransitionsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "saas_billing_subscription_transitions_total",
            Help: "Total subscription state transitions.",
        },
        []string{"from_status", "to_status"},
    )

    // Active subscription gauge by plan
    ActiveSubscriptionsByPlan = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "saas_billing_active_subscriptions",
            Help: "Current count of active subscriptions per plan.",
        },
        []string{"plan_id", "plan_name"},
    )

    // Webhook processing latency
    WebhookProcessingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "saas_billing_webhook_duration_seconds",
            Help:    "Time to process a Stripe webhook event.",
            Buckets: []float64{0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
        },
        []string{"event_type"},
    )
)

func init() {
    prometheus.MustRegister(
        WebhookEventsTotal,
        PaymentOutcomesTotal,
        SubscriptionTransitionsTotal,
        ActiveSubscriptionsByPlan,
        WebhookProcessingDuration,
    )
}
```

---

## Step 3 — Emit Metrics in Webhook Handler

```go
func (h *BillingMetricsMiddleware) Wrap(handler BillingWebhookHandler) BillingWebhookHandler {
    return BillingWebhookHandlerFunc(func(ctx context.Context, event stripe.Event) error {
        start := time.Now()
        err := handler.Handle(ctx, event)
        duration := time.Since(start).Seconds()

        outcome := "processed"
        if errors.Is(err, ErrDuplicateEvent) {
            outcome = "duplicate"
        } else if err != nil {
            outcome = "failed"
        }

        billing_metrics.WebhookEventsTotal.WithLabelValues(string(event.Type), outcome).Inc()
        billing_metrics.WebhookProcessingDuration.WithLabelValues(string(event.Type)).Observe(duration)
        return err
    })
}

func (h *PaymentHandler) HandleInvoicePaymentSucceeded(ctx context.Context, event stripe.Event) error {
    billing_metrics.PaymentOutcomesTotal.WithLabelValues("succeeded", "").Inc()
    // ...
    return nil
}

func (h *PaymentHandler) HandleInvoicePaymentFailed(ctx context.Context, event stripe.Event) error {
    var invoice stripe.Invoice
    json.Unmarshal(event.Data.Raw, &invoice)

    failureCode := ""
    if invoice.PaymentIntent != nil && invoice.PaymentIntent.LastPaymentError != nil {
        failureCode = string(invoice.PaymentIntent.LastPaymentError.Code)
    }
    billing_metrics.PaymentOutcomesTotal.WithLabelValues("failed", failureCode).Inc()
    // ...
    return nil
}
```

---

## Step 4 — Key Billing Alerts

```yaml
# Prometheus alerting rules — billing pipeline health
groups:
  - name: billing
    rules:
      # Payment failure rate spike: > 5% of payments failing in 10-min window
      - alert: HighPaymentFailureRate
        expr: |
          rate(saas_billing_payment_outcomes_total{outcome="failed"}[10m])
          / rate(saas_billing_payment_outcomes_total[10m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Payment failure rate exceeded 5%"

      # Webhook processing failures: any failures in 5-min window
      - alert: BillingWebhookProcessingFailures
        expr: rate(saas_billing_webhook_events_total{outcome="failed"}[5m]) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Stripe webhook events failing to process"

      # Billing pipeline silent: no webhook events received in 30 minutes
      # (indicates Stripe delivery issue or signature mismatch)
      - alert: BillingWebhookSilent
        expr: rate(saas_billing_webhook_events_total[30m]) == 0
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "No Stripe webhooks received in 30 minutes — pipeline may be broken"

      # Cancellation surge: > 10 cancellations in 1 hour
      - alert: SubscriptionCancellationSurge
        expr: |
          increase(saas_billing_subscription_transitions_total{
            to_status="canceled"
          }[1h]) > 10
        labels:
          severity: warning
        annotations:
          summary: "Unusual cancellation volume in the last hour"
```

---

## Step 5 — Revenue Metrics: MRR Snapshot

```sql
-- Nightly MRR snapshot for trending (not real-time Prometheus)
CREATE TABLE mrr_snapshots (
    id           BIGSERIAL   PRIMARY KEY,
    snapshot_date DATE       NOT NULL UNIQUE,
    mrr_cents    BIGINT      NOT NULL,  -- sum of all active subscriptions' monthly amount
    arr_cents    BIGINT      NOT NULL,  -- mrr * 12
    active_count INT         NOT NULL,
    trial_count  INT         NOT NULL,
    churn_count  INT         NOT NULL,  -- cancellations in the period
    new_count    INT         NOT NULL,  -- new activations in the period
    recorded_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```go
// Nightly job: compute MRR from subscriptions table and store snapshot
func (j *MRRSnapshotJob) Run(ctx context.Context) error {
    snapshot, err := j.billingRepo.ComputeMRRSnapshot(ctx, time.Now().UTC().Truncate(24*time.Hour))
    if err != nil {
        return err
    }
    if err := j.snapshotRepo.Upsert(ctx, snapshot); err != nil {
        return err
    }
    slog.InfoContext(ctx, "MRR snapshot recorded",
        "date", snapshot.Date,
        "mrr_cents", snapshot.MRRCents,
        "active_count", snapshot.ActiveCount,
        "churn_count", snapshot.ChurnCount,
    )
    return nil
}
```

---

## Billing Observability Coverage Map

| Signal | Method | Alert? |
|---|---|---|
| Webhook receive rate | Prometheus counter | Yes — silent pipeline alert |
| Webhook processing success/failure | Prometheus counter | Yes — any failure |
| Payment success rate | Prometheus counter | Yes — > 5% failure rate |
| Payment failure codes | Prometheus label | No — dashbord only |
| Subscription state transitions | Prometheus counter | Yes — cancellation surge |
| Active subscriptions by plan | Prometheus gauge | No — dashboard only |
| Webhook processing latency | Prometheus histogram | Yes — p99 > 5s |
| MRR / ARR | PostgreSQL snapshot | Yes — > 5% daily drop |
| Trial conversion rate | PostgreSQL query | No — weekly report |

---

## Quality Checks

- [ ] Every Stripe webhook handler emits `WebhookEventsTotal` with correct `outcome` label
- [ ] Silent-pipeline alert fires when no webhooks arrive for 30 minutes
- [ ] Payment failure rate alert has a `for: 5m` window to avoid flapping on single failures
- [ ] All billing log entries include `event_id`, `tenant_id`, `event_type`, `subscription_id`
- [ ] MRR snapshot job runs nightly and stores output in `mrr_snapshots` table
- [ ] Duplicate webhook detection emits `outcome="duplicate"` — not silently ignored
- [ ] Cancellation surge alert threshold is calibrated to normal daily cancellation volume

## After Completion

- Use **`stripe-webhook-processing`** for the idempotent webhook handler this monitoring wraps
- Use **`alerting-strategy`** to configure routing of billing alerts to on-call or Slack
- Use **`tenant-aware-logging`** for structured billing log field conventions
- Use **`usage-anomaly-detection`** to detect unusual billing patterns beyond threshold alerts

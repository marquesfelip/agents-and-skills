---
name: subscription-cancellation-strategy
description: 'Cancellation flows and access revocation logic for SaaS subscription billing. Use when: subscription cancellation strategy, cancel subscription stripe, immediate cancellation, end of period cancellation, cancel at period end stripe, access revocation cancellation, cancellation flow, cancellation reasons, churn prevention, win back flow, cancellation survey, cancel subscription UX, voluntary cancellation, involuntary cancellation, subscription canceled webhook, customer.subscription.deleted, cancel billing, subscription end of period, cancel_at_period_end stripe, data retention cancellation, offboarding, reactivation after cancel, cancellation audit, exit interview billing.'
argument-hint: 'Describe your cancellation UX (immediate vs end-of-period), whether you collect cancellation reasons (churn survey), what happens to customer data after cancellation, and your data retention and reactivation policies.'
---

# Subscription Cancellation Strategy

## When to Use

Invoke this skill when you need to:
- Implement immediate and scheduled (end-of-period) cancellation flows
- Collect and store cancellation reasons for churn analysis
- Revoke feature access correctly after cancellation
- Define a data retention policy for canceled tenants
- Build a win-back / reactivation flow
- Handle voluntary vs involuntary (payment failure) cancellations distinctly

---

## Step 1 — Cancellation Types

| Type | Mechanism | Access ends | Stripe action |
|---|---|---|---|
| Immediate | Cancel now | Immediately | `subscription.cancel()` |
| End of period | Cancel at renewal | At `current_period_end` | `subscription.update(cancel_at_period_end=true)` |
| Scheduled cancel reversal | Undo before period ends | N/A | `subscription.update(cancel_at_period_end=false)` |
| Involuntary (payment) | All retries failed | Via grace period → `unpaid` | `customer.subscription.deleted` event |

---

## Step 2 — Immediate Cancellation

```go
func (s *SubscriptionService) CancelImmediately(ctx context.Context, tenantID string, reason CancellationReason) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    // 1. Cancel in Stripe (generates customer.subscription.deleted webhook)
    _, err = subscription.Cancel(sub.StripeSubscriptionID, &stripe.SubscriptionCancelParams{
        // Optional: collect final invoice or prorate
        // InvoiceNow: stripe.Bool(true),
        // Prorate:    stripe.Bool(false),
    })
    if err != nil {
        return fmt.Errorf("cancel stripe subscription: %w", err)
    }

    // 2. Record cancellation reason (before webhook arrives to avoid race)
    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        if err := s.subRepo.RecordCancellationTx(ctx, tx, sub.ID, reason, time.Now()); err != nil {
            return err
        }
        // State transition happens via `customer.subscription.deleted` webhook
        return nil
    })
}
```

---

## Step 3 — End-of-Period Cancellation (Preferred)

Allows customers to retain access through the paid period — reduces refund requests and improves churn data:

```go
func (s *SubscriptionService) CancelAtPeriodEnd(ctx context.Context, tenantID string, reason CancellationReason) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        CancelAtPeriodEnd: stripe.Bool(true),
    })
    if err != nil {
        return fmt.Errorf("schedule cancellation: %w", err)
    }

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        if err := s.subRepo.SetCancelAtPeriodEndTx(ctx, tx, sub.ID, true); err != nil {
            return err
        }
        return s.subRepo.RecordCancellationTx(ctx, tx, sub.ID, reason, sub.CurrentPeriodEnd)
    })
}

func (s *SubscriptionService) UndoCancellation(ctx context.Context, tenantID string) error {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil || !sub.CancelAtPeriodEnd {
        return errors.New("no scheduled cancellation to reverse")
    }

    _, err = subscription.Update(sub.StripeSubscriptionID, &stripe.SubscriptionParams{
        CancelAtPeriodEnd: stripe.Bool(false),
    })
    if err != nil {
        return fmt.Errorf("undo scheduled cancellation: %w", err)
    }

    return s.subRepo.SetCancelAtPeriodEnd(ctx, sub.ID, false)
}
```

---

## Step 4 — Cancellation Reason Tracking

```sql
-- Cancellation reasons for churn analysis
CREATE TABLE cancellation_reasons (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    subscription_id UUID        NOT NULL REFERENCES subscriptions(id),
    reason_code     TEXT        NOT NULL,   -- 'too_expensive', 'missing_feature', 'switching_competitor', etc.
    reason_detail   TEXT,                   -- free-text from exit survey
    cancellation_type TEXT      NOT NULL    -- 'voluntary', 'involuntary'
                                CHECK (cancellation_type IN ('voluntary', 'involuntary')),
    cancelled_at    TIMESTAMPTZ NOT NULL,
    access_ends_at  TIMESTAMPTZ NOT NULL,   -- immediate vs end of period
    plan_name       TEXT,                   -- snapshot for reporting
    mrr_lost_cents  BIGINT,                 -- Monthly Recurring Revenue at time of cancel
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cancellation_reasons_tenant_id ON cancellation_reasons(tenant_id);
CREATE INDEX idx_cancellation_reasons_reason_code ON cancellation_reasons(reason_code);
CREATE INDEX idx_cancellation_reasons_cancelled_at ON cancellation_reasons(cancelled_at);
```

**Standard reason codes:**
```go
const (
    ReasonTooExpensive       = "too_expensive"
    ReasonMissingFeature     = "missing_feature"
    ReasonSwitchingCompetitor= "switching_competitor"
    ReasonBusinessClosed     = "business_closed"
    ReasonTemporaryPause     = "temporary_pause"
    ReasonNotUsing           = "not_using_enough"
    ReasonPaymentIssue       = "payment_issue"        // involuntary
    ReasonOther              = "other"
)
```

---

## Step 5 — Handle `customer.subscription.deleted` Webhook

This webhook fires for both immediate cancellations and end-of-period expirations:

```go
func (h *SubscriptionHandler) HandleSubscriptionDeleted(ctx context.Context, event stripe.Event) error {
    var stripeSub stripe.Subscription
    if err := json.Unmarshal(event.Data.Raw, &stripeSub); err != nil {
        return err
    }

    sub, err := h.subRepo.GetByStripeID(ctx, stripeSub.ID)
    if err != nil {
        return err
    }

    return h.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // 1. Transition to canceled
        if err := h.stateMachine.TransitionTx(ctx, tx, sub.ID, "canceled",
            "stripe.subscription.deleted"); err != nil {
            return err
        }

        // 2. Set ended_at timestamp
        endedAt := time.Unix(stripeSub.EndedAt, 0)
        if err := h.subRepo.SetEndedAtTx(ctx, tx, sub.ID, endedAt); err != nil {
            return err
        }

        // 3. Determine cancellation type from subscription metadata or failure history
        cancelType := "voluntary"
        if sub.PaymentFailureCount > 0 {
            cancelType = "involuntary"
        }

        // 4. Record if not already recorded (idempotent)
        return h.cancellationRepo.CreateIfNotExistsTx(ctx, tx, CancellationRecord{
            TenantID:       sub.TenantID,
            SubscriptionID: sub.ID,
            CancellationAt: endedAt,
            AccessEndsAt:   endedAt,
            CancelType:     cancelType,
        })
    })
}
```

---

## Step 6 — Data Retention After Cancellation

Define what happens to tenant data after cancellation:

```go
type DataRetentionPolicy struct {
    GraceDays      int // Keep data accessible/downloadable for N days after cancellation
    PurgeDays      int // Hard delete after N days (GDPR/LGPD compliance)
}

var DefaultRetentionPolicy = DataRetentionPolicy{
    GraceDays: 30,  // 30-day download window
    PurgeDays: 90,  // 90-day hard delete
}
```

```sql
-- Track data lifecycle for canceled tenants
ALTER TABLE tenants
    ADD COLUMN data_retention_until TIMESTAMPTZ, -- when data becomes inaccessible
    ADD COLUMN data_purge_at        TIMESTAMPTZ, -- scheduled hard delete date
    ADD COLUMN offboarding_status   TEXT DEFAULT 'none'
                                    CHECK (offboarding_status IN ('none', 'grace', 'purge_scheduled', 'purged'));
```

```go
func (s *OffboardingService) StartOffboarding(ctx context.Context, tenantID string) error {
    now := time.Now()
    return s.tenantRepo.Update(ctx, tenantID, TenantUpdate{
        DataRetentionUntil: now.AddDate(0, 0, DefaultRetentionPolicy.GraceDays),
        DataPurgeAt:        now.AddDate(0, 0, DefaultRetentionPolicy.PurgeDays),
        OffboardingStatus:  "grace",
    })
}
```

---

## Step 7 — Win-Back / Reactivation Flow

Differentiate canceled customers who want to return:

```go
func (s *SubscriptionService) WinBackReactivate(ctx context.Context, tenantID, priceID, paymentMethodID string) (*Subscription, error) {
    // Ensure data purge hasn't started
    tenant, err := s.tenantRepo.Get(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    if tenant.OffboardingStatus == "purge_scheduled" || tenant.OffboardingStatus == "purged" {
        return nil, errors.New("tenant data no longer available for reactivation")
    }

    // Cancel any scheduled data purge
    if err := s.tenantRepo.CancelOffboarding(ctx, tenantID); err != nil {
        return nil, err
    }

    // Create a new Stripe subscription
    return s.CreateSubscription(ctx, tenantID, priceID, paymentMethodID)
}
```

---

## Quality Checks

- [ ] Both immediate and end-of-period cancellation flows are implemented; UX defaults to end-of-period
- [ ] Cancellation reason collected and stored at the time of cancellation — not deferred
- [ ] `customer.subscription.deleted` handler is idempotent — safe to receive twice
- [ ] Voluntary and involuntary (payment failure) cancellations are tracked separately in analytics
- [ ] Data retention policy is defined and enforced by a scheduled job — not ad-hoc
- [ ] Reactivation checks `offboarding_status` before allowing sign-up — purged tenants rejected
- [ ] Access revocation on immediate cancel is instant (not delayed until next billing check)
- [ ] Win-back campaign eligibility excludes involuntary churns caused by fraud

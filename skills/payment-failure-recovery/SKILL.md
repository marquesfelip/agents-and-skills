---
name: payment-failure-recovery
description: 'Retry and recovery strategies for failed payments in SaaS billing. Use when: payment failure recovery, failed payment handling, dunning strategy, stripe dunning, payment retry, smart retry stripe, invoice retry, payment failure email, dunning email sequence, past due recovery, payment method update, payment failure webhook, stripe invoice.payment_failed, dunning flow, payment retry schedule, exponential backoff billing, failed charge retry, smart dunning, payment recovery flow, payment failure grace period, update payment method link, hosted invoice page stripe, dunning management stripe, recover failed subscription.'
argument-hint: 'Describe your dunning strategy (how many retries, over how many days), which emails you send at each stage, and whether you use Stripe Smart Retries or a custom retry schedule.'
---

# Payment Failure Recovery

## When to Use

Invoke this skill when you need to:
- Design a dunning flow for failed subscription payments
- Configure Stripe Smart Retries or a custom retry schedule
- Send the right emails at each failure stage (first failure, retry, final warning)
- Give customers a self-service path to update their payment method
- Handle access during the failure recovery window (grace period)
- Track dunning metrics to optimize recovery rates

---

## Step 1 — Dunning Strategy

Choose between Stripe Smart Retries (ML-based) or a custom schedule:

**Option A — Stripe Smart Retries (recommended):**
```
Enable in Stripe Dashboard → Settings → Billing → Retry schedule → Smart Retries
```
Stripe optimizes retry timing based on issuer signals. Supports 3–4 attempts over ~4 weeks.

**Option B — Custom retry schedule:**
```
Day 0:  First failure    → immediate retry attempt #1
Day 3:  Retry #2
Day 5:  Retry #3
Day 7:  Retry #4         → if failed: move to unpaid / cancel
```

Configure in Stripe Dashboard → Settings → Billing → Retry schedule → Manual.

**Failure stage tracking in DB:**
```sql
ALTER TABLE subscriptions
    ADD COLUMN payment_failure_count   INT         NOT NULL DEFAULT 0,
    ADD COLUMN first_payment_failed_at TIMESTAMPTZ,
    ADD COLUMN last_payment_failed_at  TIMESTAMPTZ,
    ADD COLUMN last_dunning_email_sent TEXT;       -- e.g., 'first_failure', 'retry_2', 'final_warning'
```

---

## Step 2 — Webhook Handler for Payment Failure

```go
func (h *BillingEventHandler) HandlePaymentFailed(ctx context.Context, event stripe.Event) error {
    var invoice stripe.Invoice
    if err := json.Unmarshal(event.Data.Raw, &invoice); err != nil {
        return err
    }

    sub, err := h.subRepo.GetByStripeID(ctx, invoice.Subscription.ID)
    if err != nil {
        return err
    }

    return h.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // 1. Transition to past_due (if not already)
        if sub.Status != "past_due" {
            if err := h.stateMachine.TransitionTx(ctx, tx, sub.ID, "past_due", "invoice.payment_failed"); err != nil {
                return err
            }
        }

        // 2. Increment failure counter
        if err := h.subRepo.IncrementPaymentFailureCountTx(ctx, tx, sub.ID); err != nil {
            return err
        }

        // 3. Record failure times
        now := time.Now()
        update := PaymentFailureUpdate{LastPaymentFailedAt: now}
        if sub.FirstPaymentFailedAt == nil {
            update.FirstPaymentFailedAt = &now
        }
        if err := h.subRepo.UpdatePaymentFailureInfoTx(ctx, tx, sub.ID, update); err != nil {
            return err
        }

        // 4. Emit dunning event (email sent outside the TX)
        h.events.Emit(DunningEvent{
            TenantID:     sub.TenantID,
            SubscriptionID: sub.ID,
            FailureCount: sub.PaymentFailureCount + 1,
            InvoiceURL:   invoice.HostedInvoiceURL,
            AttemptCount: int(invoice.AttemptCount),
        })

        return nil
    })
}
```

---

## Step 3 — Dunning Email Sequence

Send the right email at each recovery stage:

```go
func (n *DunningNotifier) Send(ctx context.Context, event DunningEvent) error {
    switch {
    case event.FailureCount == 1:
        return n.mailer.Send(ctx, PaymentFailedFirstEmail{
            TenantID:   event.TenantID,
            InvoiceURL: event.InvoiceURL,
            UpdateURL:  n.buildUpdatePaymentURL(event.TenantID),
        })

    case event.FailureCount == 2:
        return n.mailer.Send(ctx, PaymentFailedRetryEmail{
            TenantID:   event.TenantID,
            InvoiceURL: event.InvoiceURL,
        })

    case event.FailureCount >= 3:
        return n.mailer.Send(ctx, PaymentFailedFinalWarningEmail{
            TenantID:     event.TenantID,
            InvoiceURL:   event.InvoiceURL,
            AccessCutoff: event.CurrentPeriodEnd,
        })
    }
    return nil
}
```

**Email timing recommendations:**
| Email | When to send | Content |
|---|---|---|
| First failure | Immediately on `invoice.payment_failed` (attempt 1) | "Payment failed — update card" + invoice link |
| Retry warning | On attempt 2-3 | "We tried again — still failing" + update link |
| Final warning | Last retry attempt | "Account will be suspended on [date]" |
| Access locked | When moved to `unpaid` | "Account suspended — pay now to restore" |
| Payment recovered | On `invoice.paid` after failure | "Payment recovered — account restored" |

---

## Step 4 — Self-Service Payment Method Update

Generate a secure, time-limited link for customers to update their payment method:

**Option A — Stripe Customer Portal (recommended):**
```go
func (h *BillingHandler) CreatePortalSession(ctx context.Context, tenantID string) (string, error) {
    tenant, err := h.tenantRepo.Get(ctx, tenantID)
    if err != nil {
        return "", err
    }

    session, err := portalsession.New(&stripe.BillingPortalSessionParams{
        Customer:  stripe.String(tenant.StripeCustomerID),
        ReturnURL: stripe.String("https://app.yourproduct.com/billing"),
    })
    if err != nil {
        return "", fmt.Errorf("create portal session: %w", err)
    }

    return session.URL, nil
}
```

**Option B — Stripe Setup Intent (custom card update form):**
```go
func (h *BillingHandler) CreateSetupIntent(ctx context.Context, tenantID string) (string, error) {
    tenant, err := h.tenantRepo.Get(ctx, tenantID)
    if err != nil {
        return "", err
    }

    intent, err := setupintent.New(&stripe.SetupIntentParams{
        Customer:           stripe.String(tenant.StripeCustomerID),
        Usage:              stripe.String("off_session"),
        PaymentMethodTypes: []*string{stripe.String("card")},
    })
    if err != nil {
        return "", fmt.Errorf("create setup intent: %w", err)
    }

    return intent.ClientSecret, nil // Send to frontend for Stripe.js
}
```

---

## Step 5 — Recovery on Payment Success

When a retry succeeds, restore access and reset failure state:

```go
func (h *BillingEventHandler) HandleInvoicePaid(ctx context.Context, invoice stripe.Invoice) error {
    sub, err := h.subRepo.GetByStripeID(ctx, invoice.Subscription.ID)
    if err != nil {
        return err
    }

    wasInFailure := sub.PaymentFailureCount > 0

    return h.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // Transition back to active
        if err := h.stateMachine.TransitionTx(ctx, tx, sub.ID, "active", "invoice.paid"); err != nil {
            return err
        }

        // Reset failure counters
        if err := h.subRepo.ResetPaymentFailuresTx(ctx, tx, sub.ID); err != nil {
            return err
        }

        // Update billing period
        if err := h.subRepo.UpdatePeriodTx(ctx, tx, sub.ID,
            time.Unix(invoice.PeriodStart, 0),
            time.Unix(invoice.PeriodEnd, 0),
        ); err != nil {
            return err
        }

        if wasInFailure {
            h.events.Emit(PaymentRecoveredEvent{TenantID: sub.TenantID})
        }

        return nil
    })
}
```

---

## Step 6 — Dunning Metrics

Track recovery performance to optimize your strategy:

```go
// Prometheus counters
var (
    paymentFailureTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "billing_payment_failure_total",
        Help: "Total payment failures by attempt number",
    }, []string{"attempt"})

    paymentRecoveryTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "billing_payment_recovery_total",
        Help: "Recovered payments by attempt number at which recovery succeeded",
    }, []string{"recovered_at_attempt"})

    subscriptionChurnTotal = promauto.NewCounter(prometheus.CounterOpts{
        Name: "billing_subscription_churn_total",
        Help: "Subscriptions canceled due to payment failure (involuntary churn)",
    })
)
```

---

## Quality Checks

- [ ] Stripe Smart Retries enabled OR custom retry schedule configured — not relying on manual retries
- [ ] `invoice.payment_failed` transitions subscription to `past_due` — not immediately to `canceled`
- [ ] Payment failure count tracked in DB; correct dunning email sent at each stage
- [ ] Stripe Customer Portal or Setup Intent flow is available for self-service card update
- [ ] `invoice.paid` after failure resets failure counters and restores access in the same transaction
- [ ] `invoice.payment_failed` when `attempt_count >= max` triggers `unpaid` transition, not another `past_due`
- [ ] Dunning emails include the Stripe Hosted Invoice URL for easy payment
- [ ] Involuntary churn (failed payment → cancel) is tracked as a separate metric from voluntary churn

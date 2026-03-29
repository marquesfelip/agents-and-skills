---
name: subscription-lifecycle-management
description: 'Full subscription lifecycle handling for SaaS products from creation to cancellation. Use when: subscription lifecycle management, full lifecycle billing, trial to paid conversion, subscription renewal, subscription activation, billing lifecycle, SaaS subscription flow, trial expiry handling, subscription reactivation, subscription end to end, billing lifecycle management, subscription stages, first payment subscription, subscription creation to cancellation, trial conversion flow, subscription renewal handling, subscription end of period, subscription period reset, lifecycle billing events, subscription customer journey.'
argument-hint: 'Describe your full subscription journey (trial → paid → renewal → cancel → reactivate), your billing intervals, and any lifecycle events that cause you problems today (e.g., trial expiry, failed first payment, reactivation after cancellation).'
---

# Subscription Lifecycle Management

## When to Use

Invoke this skill when you need to:
- Understand and implement every stage of a subscription from creation to cancellation
- Handle trial start, trial conversion, and trial expiry correctly
- Process recurring renewals and update billing period state
- Manage reactivation flows for previously canceled subscriptions
- Wire lifecycle events to notifications, access changes, and audit logs

---

## Step 1 — Lifecycle Stages Overview

```
[1. Welcome / Trial Start]
       ↓
[2. Trial Period Active]
       ↓ (trial_end reached)
[3. Trial Conversion Attempt]
       ↓ success               ↓ failure
[4. Active / Paid]         [5. Failed First Payment → past_due / canceled]
       ↓ renewal
[6. Renewal Attempt]
       ↓ success               ↓ failure
[7. New Billing Period]    [8. Dunning → past_due → unpaid / canceled]
       ↓ (cancel or cancel_at_period_end)
[9. Cancellation]
       ↓ (optional)
[10. Reactivation]
```

Each stage has: a **trigger**, **required state change**, **Stripe action**, and **downstream effects**.

---

## Step 2 — Stage Implementation Map

| Stage | Trigger | Stripe Event | DB Action | Effects |
|---|---|---|---|---|
| Trial Start | Subscription created w/ trial | `subscription.created` | Insert subscription (`trialing`) | Welcome email, onboarding |
| Trial Ends Soon | 3 days before `trial_end` | `subscription.trial_will_end` | No state change | Reminder email |
| Trial Convert (success) | First payment collected | `invoice.paid` + `subscription.updated` → `active` | Status → `active`, set `converted_at` | Activation email, full access |
| Trial Convert (failure) | Payment fails at trial end | `invoice.payment_failed` | Status → `past_due` or `canceled` | Dunning email |
| Renewal Success | Invoice paid | `invoice.paid` | Update `current_period_start/end` | Receipt email |
| Renewal Failure | Invoice payment fails | `invoice.payment_failed` | Status → `past_due` | First dunning email |
| Grace Period Expires | All retries exhausted | `subscription.updated` → `unpaid` | Status → `unpaid` | Final warning email, lock access |
| Cancel (immediate) | Customer cancels | `subscription.deleted` | Status → `canceled`, `ended_at` = now | Cancellation email, revoke access |
| Cancel (end of period) | Customer schedules cancel | `subscription.updated` with `cancel_at_period_end: true` | Set `cancel_at_period_end: true` | Scheduled cancellation email |
| Period End Cancel | `cancel_at_period_end` reached | `subscription.deleted` | Status → `canceled` | Access revocation |
| Reactivation | Cancel reversed before period end | `subscription.updated` with `cancel_at_period_end: false` | Clear `cancel_at_period_end` | Reactivation email |
| Reactivation (post-cancel) | New subscription created | `subscription.created` | Insert new subscription (`active`) | Welcome back email |

---

## Step 3 — Trial Conversion Handler

The most critical lifecycle moment — getting payment and unlocking full access atomically:

```go
func (s *SubscriptionService) HandleInvoicePaid(ctx context.Context, event stripe.Event) error {
    var invoice stripe.Invoice
    if err := json.Unmarshal(event.Data.Raw, &invoice); err != nil {
        return err
    }

    // Only act on subscription invoices
    if invoice.Subscription == nil {
        return nil
    }

    sub, err := s.subRepo.GetByStripeID(ctx, invoice.Subscription.ID)
    if err != nil {
        return err
    }

    // Determine new period from invoice
    newPeriodStart := time.Unix(invoice.PeriodStart, 0)
    newPeriodEnd := time.Unix(invoice.PeriodEnd, 0)

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // 1. Transition to active
        if err := s.stateMachine.TransitionTx(ctx, tx, sub.ID, "active", "invoice.paid"); err != nil {
            return err
        }

        // 2. Update billing period
        if err := s.subRepo.UpdatePeriodTx(ctx, tx, sub.ID, newPeriodStart, newPeriodEnd); err != nil {
            return err
        }

        // 3. Record converted_at if this was a trial conversion
        if sub.Status == "trialing" {
            if err := s.subRepo.SetConvertedAtTx(ctx, tx, sub.ID, time.Now()); err != nil {
                return err
            }
        }

        // 4. Store invoice record
        return s.invoiceRepo.UpsertTx(ctx, tx, mapStripeInvoice(sub.TenantID, &invoice))
    })
}
```

---

## Step 4 — Renewal Handling

Keep `current_period_start` and `current_period_end` accurate — entitlement checks depend on them:

```go
func (s *SubscriptionService) HandleSubscriptionUpdated(ctx context.Context, stripeSub stripe.Subscription) error {
    sub, err := s.subRepo.GetByStripeID(ctx, stripeSub.ID)
    if err != nil {
        return err
    }

    update := SubscriptionUpdate{
        Status:              string(stripeSub.Status),
        CurrentPeriodStart:  time.Unix(stripeSub.CurrentPeriodStart, 0),
        CurrentPeriodEnd:    time.Unix(stripeSub.CurrentPeriodEnd, 0),
        CancelAtPeriodEnd:   stripeSub.CancelAtPeriodEnd,
        CanceledAt:          nullableTime(stripeSub.CanceledAt),
    }

    return s.subRepo.Update(ctx, sub.ID, update)
}
```

---

## Step 5 — Reactivation Flow

When a customer reactivates after cancellation, a new Stripe subscription must be created (Stripe does not un-cancel subscriptions; they must be recreated):

```go
func (s *SubscriptionService) Reactivate(ctx context.Context, tenantID, priceID string) (*Subscription, error) {
    existing, err := s.subRepo.GetActiveForTenant(ctx, tenantID)

    // Case 1: Subscription is cancel_at_period_end — reverse the scheduled cancel
    if err == nil && existing.CancelAtPeriodEnd {
        _, err = subscription.Update(existing.StripeSubscriptionID, &stripe.SubscriptionParams{
            CancelAtPeriodEnd: stripe.Bool(false),
        })
        if err != nil {
            return nil, fmt.Errorf("reverse scheduled cancellation: %w", err)
        }
        return s.subRepo.ClearScheduledCancellation(ctx, tenantID)
    }

    // Case 2: Subscription is fully canceled — create a new one
    return s.CreateSubscription(ctx, tenantID, priceID, "")
}
```

---

## Step 6 — Lifecycle Event Notifications

Emit events at each lifecycle stage for email, metrics, and entitlements:

```go
type LifecycleEvent string

const (
    EventTrialStarted       LifecycleEvent = "trial.started"
    EventTrialWillEnd       LifecycleEvent = "trial.will_end"
    EventTrialConverted     LifecycleEvent = "trial.converted"
    EventTrialExpired       LifecycleEvent = "trial.expired"
    EventRenewalSucceeded   LifecycleEvent = "renewal.succeeded"
    EventRenewalFailed      LifecycleEvent = "renewal.failed"
    EventCanceled           LifecycleEvent = "subscription.canceled"
    EventCancelScheduled    LifecycleEvent = "subscription.cancel_scheduled"
    EventReactivated        LifecycleEvent = "subscription.reactivated"
    EventAccessLocked       LifecycleEvent = "access.locked"
    EventAccessRestored     LifecycleEvent = "access.restored"
)
```

---

## Quality Checks

- [ ] Every lifecycle stage is handled by a named event handler — no inline or anonymous logic in dispatch
- [ ] Trial conversion and renewal both update `current_period_start/end` and subscription `status` in the same transaction
- [ ] `converted_at` is set exactly once on the first `invoice.paid` following a `trialing` status
- [ ] Reactivation creates a new Stripe subscription when the old one is fully canceled
- [ ] Lifecycle notification events are emitted after DB commit — not inside the transaction
- [ ] `cancel_at_period_end` flag is tracked in DB and reflected in entitlement checks
- [ ] Access changes (lock/unlock) are triggered by state transitions — never by a cron job alone

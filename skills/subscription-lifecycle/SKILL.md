---
name: subscription-lifecycle
description: >
  Subscription state transitions and lifecycle handling for SaaS products.
  Use when: subscription lifecycle, subscription state, subscription status, trialing,
  subscription active, past due, dunning, subscription canceled, subscription paused,
  subscription resumed, trial conversion, trial expiry, subscription upgrade, plan change,
  downgrade subscription, subscription renewal, subscription reactivation, proration,
  billing cycle, subscription webhook, subscription state machine, subscription event,
  subscription transition, subscription model, subscription database, failed payment recovery.
argument-hint: >
  Describe your subscription model (trial → paid, monthly/annual, plan tiers), the lifecycle
  issue you're solving (trial expiry, failed payment, plan upgrade/downgrade), and your billing
  provider (Stripe, Paddle, etc.). Include current state transition logic if it exists.
---

# Subscription Lifecycle Specialist

## When to Use

Invoke this skill when you need to:
- Define all subscription states and valid transitions (state machine)
- Handle trial expiry, trial conversion, and trial extension
- Process plan upgrades, downgrades, and prorations
- Implement dunning flows for failed payments
- Handle subscription cancellation, pausing, and reactivation
- Build reliable event-driven state transitions from webhook events

---

## Step 1 — Define the State Machine

**Subscription states:**

| State | Meaning | Access |
|---|---|---|
| `trialing` | Trial period active; no payment on file required | Full or feature-limited |
| `active` | Paid and current | Full access |
| `past_due` | Payment failed; billing provider retrying | Grace period access |
| `unpaid` | All retries exhausted; no payment | Read-only or locked |
| `paused` | Customer voluntarily paused (if supported) | No access or frozen |
| `canceled` | Subscription ended | No access; data retention period |
| `incomplete` | Initial payment pending (async payment methods) | No access until confirmed |
| `incomplete_expired` | Initial payment not completed within 23h | Create new subscription |

**Valid transitions (state machine diagram in text form):**
```
trialing      → active          (trial converts: payment collected)
trialing      → canceled        (trial expired without conversion; or manual cancel)
active        → past_due        (payment failure; provider retries started)
active        → canceled        (cancel_at_period_end; immediately; or payment failure)
active        → paused          (voluntary pause; provider-dependent feature)
past_due      → active          (payment retry succeeded)
past_due      → unpaid          (all retries exhausted)
past_due      → canceled        (customer cancels during grace period)
unpaid        → active          (customer updates payment method and pays)
unpaid        → canceled        (customer gives up or admin cancels)
paused        → active          (customer resumes)
canceled      → active          (reactivation — creates a new subscription in provider)
```

**Implementation — state transition guard:**
```go
var validTransitions = map[string][]string{
    "trialing":  {"active", "canceled"},
    "active":    {"past_due", "canceled", "paused"},
    "past_due":  {"active", "unpaid", "canceled"},
    "unpaid":    {"active", "canceled"},
    "paused":    {"active", "canceled"},
    "canceled":  {"active"}, // reactivation
}

func (s *SubscriptionService) Transition(ctx context.Context, subID, toState string) error {
    sub, err := s.repo.Get(ctx, subID)
    if err != nil {
        return err
    }

    allowed := validTransitions[sub.Status]
    if !slices.Contains(allowed, toState) {
        return fmt.Errorf("invalid transition: %s → %s", sub.Status, toState)
    }

    return s.applyTransition(ctx, sub, toState)
}
```

Checklist:
- [ ] All subscription states explicitly defined in code and database constraint
- [ ] State machine enforced — invalid transitions rejected with an error, never silently accepted
- [ ] State transition function is the single place where status changes — no ad-hoc status updates
- [ ] Each transition triggers the appropriate downstream actions (access change, email, audit log)

---

## Step 2 — Trial Lifecycle

**Trial model:**
```sql
-- Trial configuration per plan
ALTER TABLE plans ADD COLUMN trial_days INT NOT NULL DEFAULT 14;

-- Subscription trial fields
ALTER TABLE subscriptions ADD COLUMN trial_start TIMESTAMPTZ;
ALTER TABLE subscriptions ADD COLUMN trial_end   TIMESTAMPTZ;
ALTER TABLE subscriptions ADD COLUMN converted_at TIMESTAMPTZ;
ALTER TABLE subscriptions ADD COLUMN trial_extended_count INT NOT NULL DEFAULT 0;
```

**Trial expiry job:**
```go
// Runs as a scheduled job — daily or hourly
func (j *TrialExpiryJob) Run(ctx context.Context) error {
    expiredTrials, err := j.repo.FindExpiredTrials(ctx)
    for _, sub := range expiredTrials {
        if sub.HasPaymentMethod() {
            // Collect payment and activate
            if err := j.billing.CreateInitialCharge(ctx, sub); err != nil {
                // Moves to past_due if charge fails
                j.transitionTo(ctx, sub, "past_due")
            } else {
                j.transitionTo(ctx, sub, "active")
            }
        } else {
            // No payment method — expire gracefully
            j.transitionTo(ctx, sub, "canceled")
            j.notifier.SendTrialExpiredEmail(ctx, sub)
        }
    }
    return nil
}
```

**Trial conversion triggers:**
- Customer adds payment method → immediately charge + activate (don't wait for trial_end)
- Customer clicks "Upgrade" → same flow
- `customer.subscription.updated` webhook: `status` changes from `trialing` to `active`

Checklist:
- [ ] Trial expiry handled idempotently — job can run multiple times without double-charging
- [ ] Trial expiry notified by email 7 days, 3 days, and 1 day before expiry
- [ ] Adding a payment method during trial triggers immediate conversion — not delayed to trial_end
- [ ] Extended trials tracked (`trial_extended_count`) — prevents abuse of trial extension

---

## Step 3 — Plan Upgrades and Downgrades

**Upgrade (immediate, proration):**
```
Customer upgrades from Starter ($50/mo) to Pro ($150/mo) on day 15 of 30.
Proration: customer owes ($150 - $50) × (15/30) = $50 for the remainder.
Next cycle: billed the full Pro price.
```

**Downgrade (at period end, no proration):**
```
Customer downgrades from Pro to Starter mid-cycle.
Downgrade effective at current_period_end — customer uses Pro features until then.
Next cycle: billed at Starter price.
```

**Handling plan change webhooks:**
```go
func (h *Processor) handleSubscriptionUpdated(ctx context.Context, event stripe.Event) error {
    var sub stripe.Subscription
    json.Unmarshal(event.Data.Raw, &sub)

    // Detect plan change
    if event.Data.PreviousAttributes != nil {
        prev, _ := event.Data.PreviousAttributes["items"]
        if prev != nil {
            // Plan changed — sync local plan reference
            return h.syncPlanChange(ctx, &sub)
        }
    }
    // Generic sync for all other fields
    return h.syncSubscription(ctx, &sub)
}

func (h *Processor) syncPlanChange(ctx context.Context, sub *stripe.Subscription) error {
    newPlanID := h.mapStripePriceToLocalPlan(sub.Items.Data[0].Price.ID)

    return h.db.Transaction(ctx, func(tx pgx.Tx) error {
        // Update subscription plan
        _, err := tx.Exec(ctx,
            `UPDATE subscriptions SET plan_id = $1, updated_at = NOW()
             WHERE provider_subscription_id = $2`,
            newPlanID, sub.ID,
        )
        if err != nil {
            return err
        }
        // Update tenant's active plan for entitlement checks
        tenantID := sub.Metadata["tenant_id"]
        _, err = tx.Exec(ctx,
            `UPDATE tenants SET plan_id = $1 WHERE id = $2`,
            newPlanID, tenantID,
        )
        return err
    })
}
```

Checklist:
- [ ] Upgrades are immediate — access to new plan features granted before next billing cycle
- [ ] Downgrades take effect at period end — customer retains higher plan until they've been billed for it
- [ ] `cancel_at_period_end = true` handled — subscription remains active until end of current period
- [ ] Local plan_id and tenant plan_id updated atomically in the same transaction

---

## Step 4 — Cancellation and Reactivation

**Cancellation modes:**

| Mode | Behavior | When |
|---|---|---|
| Immediate | Subscription ends now; credit issued for unused days | Customer requests; admin action |
| At period end | Access continues until current period end | Default customer cancellation via UI |
| Scheduled | `cancel_at` set to a future date | Annual plans with specific end dates |

**Cancellation handler:**
```go
func (s *SubscriptionService) Cancel(ctx context.Context, tenantID string, immediate bool) error {
    sub, err := s.repo.GetActive(ctx, tenantID)
    if err != nil {
        return err
    }

    if immediate {
        // Cancel in Stripe immediately
        _, err = s.stripe.Subscriptions.Cancel(sub.ProviderSubscriptionID, nil)
        // Webhook will arrive: customer.subscription.deleted → transition to canceled
    } else {
        // Schedule cancellation at period end
        params := &stripe.SubscriptionParams{CancelAtPeriodEnd: stripe.Bool(true)}
        _, err = s.stripe.Subscriptions.Update(sub.ProviderSubscriptionID, params)
        // Immediately reflect in local state
        s.repo.SetCancelAtPeriodEnd(ctx, sub.ID, true)
    }

    s.auditLog.Record(ctx, tenantID, "subscription.cancel_requested", map[string]any{
        "immediate": immediate,
    })

    return err
}
```

**Reactivation (before period end):**
```go
func (s *SubscriptionService) Reactivate(ctx context.Context, tenantID string) error {
    sub, err := s.repo.GetByTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    if sub.Status == "canceled" && time.Now().After(sub.CurrentPeriodEnd) {
        // Fully canceled — create new subscription in Stripe
        return s.createNewSubscription(ctx, tenantID, sub.PlanID)
    }

    if sub.CancelAtPeriodEnd {
        // Scheduled cancellation — undo it
        params := &stripe.SubscriptionParams{CancelAtPeriodEnd: stripe.Bool(false)}
        _, err = s.stripe.Subscriptions.Update(sub.ProviderSubscriptionID, params)
        s.repo.SetCancelAtPeriodEnd(ctx, sub.ID, false)
        return err
    }

    return ErrSubscriptionNotCancelable
}
```

Checklist:
- [ ] "Cancel at period end" is set in the provider AND reflected immediately in local `cancel_at_period_end`
- [ ] Reactivation before period end undoes the scheduled cancellation — does not create a new subscription
- [ ] Reactivation after full cancellation creates a new subscription — previous subscription not reused
- [ ] Off-boarding flow defined for canceled state: data export, deletion schedule, farewell email

---

## Output Report

### Critical
- No state machine enforcement — any code can set `status` to any value; invalid states reachable
- Cancellation webhook not handled — subscription status never updated to `canceled` in local DB

### High
- Trial expiry job is not idempotent — double-running charges some trials twice
- Plan change only updates subscriptions table, not tenants.plan_id — entitlement checks use stale plan
- `cancel_at_period_end` not reflected immediately in local state — entitlement shows wrong plan until webhook arrives

### Medium
- No trial expiry warning emails — customers surprised by trial end; conversion rate lower than possible
- Downgrade treated as immediate — customer loses paid-for plan features before the billing period ends
- Reactivation always creates a new subscription — avoidable churn on subscriptions still within period

### Low
- Subscription state transitions not logged — no audit trail for support and billing dispute investigations
- Trial extension count not tracked — customer support can extend trials indefinitely without visibility
- Past-due grace period duration not configurable — hardcoded value requires code deployment to adjust

### Passed
- All subscription states defined in DB check constraint; state machine enforced in code
- Trial expiry job is idempotent; warning emails sent at 7, 3, and 1 day before expiry
- Plan changes update subscription and tenant plan_id atomically; upgrades take effect immediately
- Cancel-at-period-end reflected immediately in local state; reactivation undoes vs. creates based on period
- All transitions logged to audit log; webhook handling covers all lifecycle events

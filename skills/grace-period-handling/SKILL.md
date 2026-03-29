---
name: grace-period-handling
description: 'Grace periods after payment failure for SaaS subscription billing. Use when: grace period handling, payment failure grace period, billing grace period, past due grace period, subscription grace period, access during payment failure, dunning grace period, grace period access control, grace period expiry, grace period configuration, stripe past due access, past due feature access, past due subscription access, grace period duration, grace period email, grace period notification, grace period job, grace period enforcement, billing access after failure, lock access past due, grace period unlock, grace period policy.'
argument-hint: 'Describe your current grace period duration (e.g., 14 days), what access customers retain during the grace period, and whether you use Stripe Smart Retries or a manual schedule to know when retries end.'
---

# Grace Period Handling

## When to Use

Invoke this skill when you need to:
- Define and configure the grace period duration after a payment failure
- Maintain correct feature access during the `past_due` state
- Track grace period start and end dates in your database
- Send staged communication at grace period milestones
- Enforce access revocation when the grace period expires
- Handle grace period extension for enterprise or high-value customers

---

## Step 1 — Grace Period Model

The grace period is the window between first payment failure (`past_due`) and full access revocation (`unpaid`):

```
[Invoice payment fails]
        │
        ▼
 Status = past_due ──────────────────────────────────┐
        │                                             │
        │  Grace period active                        │
        │  Access: maintained (with banners)          │
        │                                             │
        ├── Day 0: Stripe retry #1                    │
        ├── Day 3: Stripe retry #2 + 2nd email        │
        ├── Day 7: Stripe retry #3 + warning email    │
        │                                             │
        ▼ (all retries exhausted)                     ▼ (customer pays)
 Status = unpaid                               Status = active
 Access: revoked                               Access: restored
```

**Grace period stored in DB:**
```sql
ALTER TABLE subscriptions
    ADD COLUMN grace_period_ends_at   TIMESTAMPTZ,  -- set when entering past_due
    ADD COLUMN grace_period_extended  BOOLEAN NOT NULL DEFAULT false;
```

---

## Step 2 — Start Grace Period on Payment Failure

Set the grace period end date when transitioning to `past_due`:

```go
const DefaultGracePeriodDays = 14

func (s *SubscriptionService) StartGracePeriod(ctx context.Context, subID string) error {
    gracePeriodEnd := time.Now().Add(DefaultGracePeriodDays * 24 * time.Hour)

    return s.subRepo.SetGracePeriodEnd(ctx, subID, gracePeriodEnd)
}

// Call this in the state machine transition effect for past_due
func (e *GracePeriodTransitionEffect) OnTransition(ctx context.Context, sub *Subscription, from, to, reason string) {
    if to != "past_due" {
        return
    }
    if err := e.service.StartGracePeriod(ctx, sub.ID); err != nil {
        slog.Error("failed to start grace period",
            slog.String("sub_id", sub.ID),
            slog.String("err", err.Error()),
        )
    }
}
```

**Dynamic grace period from Stripe's retry schedule:**

Instead of a fixed duration, derive the grace period end from Stripe's next retry date:
```go
func (s *SubscriptionService) StartGracePeriodFromStripe(ctx context.Context, subID, stripeSubID string) error {
    stripeSub, err := subscription.Get(stripeSubID, nil)
    if err != nil {
        return err
    }

    var gracePeriodEnd time.Time
    if stripeSub.NextPendingInvoiceItemInvoice > 0 {
        // Use Stripe's scheduled retry as the grace period end
        gracePeriodEnd = time.Unix(stripeSub.NextPendingInvoiceItemInvoice, 0)
    } else {
        // Fallback to configured duration
        gracePeriodEnd = time.Now().Add(DefaultGracePeriodDays * 24 * time.Hour)
    }

    return s.subRepo.SetGracePeriodEnd(ctx, subID, gracePeriodEnd)
}
```

---

## Step 3 — Access Control During Grace Period

The `past_due` state must allow access — do not lock out paying customers who have a temporary failure:

```go
// Entitlement check including grace period awareness
func (s *EntitlementService) HasPlanAccess(ctx context.Context, tenantID string) (bool, error) {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if errors.Is(err, ErrNotFound) {
        return false, nil
    }
    if err != nil {
        return false, err
    }

    switch sub.Status {
    case "trialing", "active":
        return true, nil

    case "past_due":
        // Maintain access if within configured grace period
        if sub.GracePeriodEndsAt == nil {
            return true, nil // Grace period not explicitly set — allow access (safe default)
        }
        if time.Now().Before(*sub.GracePeriodEndsAt) {
            return true, nil
        }
        return false, nil // Grace period expired but status not yet updated

    default: // unpaid, canceled, paused, etc.
        return false, nil
    }
}
```

**Show in-app banners during grace period without blocking access:**
```go
func (s *SubscriptionService) GetBillingWarning(ctx context.Context, tenantID string) *BillingWarning {
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil || sub == nil {
        return nil
    }

    if sub.Status == "past_due" {
        daysLeft := 0
        if sub.GracePeriodEndsAt != nil {
            daysLeft = int(time.Until(*sub.GracePeriodEndsAt).Hours() / 24)
        }
        return &BillingWarning{
            Level:           "warning",
            Message:         "Payment failed. Please update your payment method.",
            DaysUntilLocked: daysLeft,
            UpdatePaymentURL: buildPortalURL(sub.TenantID),
        }
    }

    return nil
}
```

---

## Step 4 — Grace Period Milestone Notifications

Send escalating notifications as the grace period progresses:

```go
type GracePeriodNotifier struct {
    mailer  Mailer
    subRepo SubscriptionRepository
}

func (n *GracePeriodNotifier) SendMilestoneEmails(ctx context.Context) error {
    pastDueSubs, err := n.subRepo.GetPastDueWithGracePeriod(ctx)
    if err != nil {
        return err
    }

    for _, sub := range pastDueSubs {
        if sub.GracePeriodEndsAt == nil {
            continue
        }

        daysLeft := int(time.Until(*sub.GracePeriodEndsAt).Hours() / 24)

        switch daysLeft {
        case 7:
            n.sendIfNotSent(ctx, sub, "grace_period_7_days")
        case 3:
            n.sendIfNotSent(ctx, sub, "grace_period_3_days")
        case 1:
            n.sendIfNotSent(ctx, sub, "grace_period_1_day")
        case 0:
            n.sendIfNotSent(ctx, sub, "grace_period_final")
        }
    }
    return nil
}

func (n *GracePeriodNotifier) sendIfNotSent(ctx context.Context, sub Subscription, templateKey string) {
    if sub.LastDunningEmailSent == templateKey {
        return // Already sent this milestone
    }
    if err := n.mailer.Send(ctx, GracePeriodEmail{
        TenantID:    sub.TenantID,
        TemplateKey: templateKey,
        UpdateURL:   buildPortalURL(sub.TenantID),
    }); err != nil {
        slog.Error("grace period email failed",
            slog.String("template", templateKey),
            slog.String("tenant_id", sub.TenantID),
        )
        return
    }
    n.subRepo.SetLastDunningEmail(ctx, sub.ID, templateKey)
}
```

---

## Step 5 — Grace Period Expiry Enforcement

A scheduled job enforces access revocation when the grace period ends and Stripe retries are exhausted:

```go
// Runs every hour
func (j *GracePeriodExpiryJob) Run(ctx context.Context) error {
    expired, err := j.subRepo.FindExpiredGracePeriods(ctx)
    if err != nil {
        return err
    }

    for _, sub := range expired {
        // Verify Stripe still shows past_due (not already resolved)
        stripeSub, err := subscription.Get(sub.StripeSubscriptionID, nil)
        if err != nil {
            slog.Error("failed to verify stripe subscription",
                slog.String("sub_id", sub.ID))
            continue
        }

        if stripeSub.Status == stripe.SubscriptionStatusPastDue ||
            stripeSub.Status == stripe.SubscriptionStatusUnpaid {
            // Lock access — transition to unpaid
            if err := j.stateMachine.Transition(ctx, sub.ID, "unpaid",
                "grace_period_expired"); err != nil {
                slog.Error("grace period expiry transition failed",
                    slog.String("sub_id", sub.ID),
                    slog.String("err", err.Error()),
                )
            }
        }
        // If Stripe shows active/paid — webhook may have been delayed; skip
    }
    return nil
}
```

---

## Step 6 — Grace Period Extension (Enterprise)

Allow customer success to extend grace periods for high-value customers:

```go
func (s *SubscriptionService) ExtendGracePeriod(ctx context.Context, tenantID string, extensionDays int) error {
    sub, err := s.subRepo.GetByTenantID(ctx, tenantID)
    if err != nil {
        return err
    }

    if sub.Status != "past_due" {
        return errors.New("can only extend grace period for past_due subscriptions")
    }

    newEnd := time.Now().Add(time.Duration(extensionDays) * 24 * time.Hour)
    if sub.GracePeriodEndsAt != nil && sub.GracePeriodEndsAt.After(newEnd) {
        return errors.New("new grace period end is before current end")
    }

    return s.subRepo.UpdateGracePeriodEnd(ctx, sub.ID, newEnd, true /* grace_period_extended */)
}
```

---

## Quality Checks

- [ ] `grace_period_ends_at` is set the moment a subscription enters `past_due` — not deferred
- [ ] Access check for `past_due` status reads `grace_period_ends_at` — not a hardcoded duration
- [ ] In-app banner shown for all `past_due` tenants — access not silently suspended mid-session
- [ ] Milestone emails sent at 7 days, 3 days, 1 day, and 0 days remaining — tracked by `last_dunning_email_sent`
- [ ] Grace period expiry job verifies Stripe status before locking — accounts for delayed webhooks
- [ ] Access revocation happens via state machine transition, not direct DB update
- [ ] Grace period extension is audit-logged with actor and reason
- [ ] `past_due` → `active` (payment recovered) clears grace period fields and restores access

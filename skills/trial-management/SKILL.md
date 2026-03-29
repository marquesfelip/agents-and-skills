---
name: trial-management
description: 'Trial activation and expiration handling for SaaS products. Use when: trial management, trial activation, trial expiration, trial period, trial conversion, trial extension, trial expired, trial end, trial to paid, free trial, trial flow, trial state machine, trial duration, trial reminder email, trial grace period, trial without credit card, trial with credit card, trial quota, trial feature limits, trial signup, SaaS trial, trial expiry job.'
argument-hint: 'Describe your trial model (credit card required or not, trial duration, feature limits during trial), the lifecycle event you are implementing (start trial, expire trial, convert to paid, extend trial), and your billing provider (Stripe, Paddle, etc.).'
---

# Trial Management

## When to Use

Invoke this skill when you need to:
- Start a trial on tenant/user creation with configurable duration and feature scope
- Schedule and execute trial expiration — transitioning tenants to suspended or prompting conversion
- Convert a trialing tenant to a paid subscription atomically and safely
- Extend a trial manually (admin action, sales motion, marketing campaign)
- Send timed reminder emails at key trial milestones (day 1, mid-point, 3 days left, expiry)
- Enforce trial-specific feature limits distinct from paid plan access

---

## Trial State Machine

```
[signup] → trialing → active     (converted: payment collected)
                    → suspended  (trial expired without conversion)
                    → canceled   (user canceled during trial)
```

---

## Step 1 — Schema

```sql
CREATE TABLE tenant_trials (
    tenant_id           UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    started_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    ends_at             TIMESTAMPTZ NOT NULL,
    converted_at        TIMESTAMPTZ,           -- set when converted to paid
    expired_at          TIMESTAMPTZ,           -- set when expiry job runs
    extended_count      INT NOT NULL DEFAULT 0,
    extended_by_days    INT NOT NULL DEFAULT 0,
    original_ends_at    TIMESTAMPTZ NOT NULL,  -- preserved when extended
    credit_card_on_file BOOLEAN NOT NULL DEFAULT false,
    plan_id             TEXT NOT NULL          -- which plan to activate on conversion
);

CREATE INDEX idx_tenant_trials_ends_at ON tenant_trials(ends_at)
    WHERE converted_at IS NULL AND expired_at IS NULL;

CREATE TABLE trial_reminders (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   UUID      NOT NULL REFERENCES tenants(id),
    type        TEXT      NOT NULL,  -- 'day_1' | 'midpoint' | '3_days_left' | 'expired'
    sent_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, type)
);
```

---

## Step 2 — Starting a Trial

```go
type TrialConfig struct {
    DurationDays       int
    CreditCardRequired bool
    PlanID             string   // trial uses plan limits from this plan
}

func (s *trialService) StartTrial(ctx context.Context, tenantID uuid.UUID, cfg TrialConfig) error {
    now := time.Now().UTC()
    endsAt := now.AddDate(0, 0, cfg.DurationDays)

    return s.db.Transaction(ctx, func(tx DB) error {
        // Insert trial record
        if err := s.repo.Create(ctx, tx, TenantTrial{
            TenantID:         tenantID,
            StartedAt:        now,
            EndsAt:           endsAt,
            OriginalEndsAt:   endsAt,
            PlanID:           cfg.PlanID,
            CreditCardOnFile: cfg.CreditCardRequired, // false until card collected
        }); err != nil {
            return fmt.Errorf("create trial: %w", err)
        }

        // Ensure tenant is in 'trialing' status
        return s.tenantSvc.Transition(ctx, tx, tenantID, TenantStatusTrialing, TransitionOptions{
            Actor:       SystemActor,
            Reason:      "trial_started",
            TrialEndsAt: &endsAt,
        })
    })
}
```

---

## Step 3 — Trial Conversion to Paid

```go
type ConvertTrialInput struct {
    TenantID        uuid.UUID
    PaymentMethodID string  // Stripe payment method ID
    PlanID          string
    BillingInterval string  // 'monthly' | 'annual'
}

func (s *trialService) Convert(ctx context.Context, input ConvertTrialInput) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        trial, err := s.repo.LockByTenantID(ctx, tx, input.TenantID) // SELECT FOR UPDATE
        if err != nil {
            return err
        }
        if trial.ConvertedAt != nil {
            return ErrAlreadyConverted // idempotent — do not double-charge
        }

        // Create or retrieve Stripe subscription
        sub, err := s.billing.CreateSubscription(ctx, billing.CreateSubscriptionInput{
            TenantID:        input.TenantID,
            PaymentMethodID: input.PaymentMethodID,
            PlanID:          input.PlanID,
            BillingInterval: input.BillingInterval,
        })
        if err != nil {
            return fmt.Errorf("create subscription: %w", err)
        }

        // Mark trial converted
        now := time.Now().UTC()
        if err := s.repo.MarkConverted(ctx, tx, input.TenantID, now); err != nil {
            return err
        }

        // Transition tenant to active
        return s.tenantSvc.Transition(ctx, tx, input.TenantID, TenantStatusActive, TransitionOptions{
            Actor:          SystemActor,
            Reason:         "trial_converted",
            SubscriptionID: sub.ID,
        })
    })
}
```

---

## Step 4 — Trial Expiration Job

```go
// Run every hour — expire trials whose ends_at has passed
func (s *trialService) ExpireTrials(ctx context.Context) error {
    due, err := s.repo.ListExpired(ctx, time.Now().UTC())
    if err != nil {
        return err
    }

    for _, trial := range due {
        if err := s.expireOne(ctx, trial); err != nil {
            slog.Error("trial expiry failed", "tenant_id", trial.TenantID, "err", err)
            continue // process others — best-effort
        }
    }
    return nil
}

func (s *trialService) expireOne(ctx context.Context, trial TenantTrial) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        now := time.Now().UTC()
        if err := s.repo.MarkExpired(ctx, tx, trial.TenantID, now); err != nil {
            return err
        }
        if err := s.tenantSvc.Transition(ctx, tx, trial.TenantID, TenantStatusSuspended, TransitionOptions{
            Actor:  SystemActor,
            Reason: "trial_expired",
        }); err != nil {
            return err
        }
        // Publish for downstream handling (disable features, lock access)
        s.events.Publish(ctx, TrialExpiredEvent{TenantID: trial.TenantID})
        return nil
    })
}
```

---

## Step 5 — Trial Extension

```go
const MaxExtensions = 2
const MaxExtensionDays = 14

type ExtendTrialInput struct {
    TenantID  uuid.UUID
    Days      int
    Actor     Actor  // admin who approved the extension
    Reason    string // sales note or justification
}

func (s *trialService) Extend(ctx context.Context, input ExtendTrialInput) error {
    if input.Days <= 0 || input.Days > MaxExtensionDays {
        return ErrInvalidExtensionDays
    }

    return s.db.Transaction(ctx, func(tx DB) error {
        trial, err := s.repo.LockByTenantID(ctx, tx, input.TenantID)
        if err != nil {
            return err
        }
        if trial.ExtendedCount >= MaxExtensions {
            return ErrExtensionLimitReached
        }
        if trial.ConvertedAt != nil || trial.ExpiredAt != nil {
            return ErrTrialNoLongerActive
        }

        newEndsAt := trial.EndsAt.AddDate(0, 0, input.Days)
        return s.repo.Extend(ctx, tx, input.TenantID, newEndsAt, input.Days)
    })
}
```

---

## Step 6 — Reminder Email Schedule

```go
// Run daily — check which reminder milestones need sending
func (s *trialService) SendDueReminders(ctx context.Context) error {
    active, err := s.repo.ListActive(ctx)
    if err != nil {
        return err
    }

    now := time.Now().UTC()
    for _, trial := range active {
        daysLeft := int(trial.EndsAt.Sub(now).Hours() / 24)
        elapsed   := int(now.Sub(trial.StartedAt).Hours() / 24)
        midpoint  := int(trial.EndsAt.Sub(trial.StartedAt).Hours() / 24 / 2)

        reminderType := ""
        switch {
        case elapsed == 1:
            reminderType = "day_1"
        case elapsed == midpoint:
            reminderType = "midpoint"
        case daysLeft == 3:
            reminderType = "3_days_left"
        case daysLeft <= 0:
            reminderType = "expired"
        }

        if reminderType == "" {
            continue
        }

        if err := s.sendReminder(ctx, trial.TenantID, reminderType); err != nil {
            slog.Error("reminder send failed", "tenant_id", trial.TenantID, "type", reminderType, "err", err)
        }
    }
    return nil
}

func (s *trialService) sendReminder(ctx context.Context, tenantID uuid.UUID, reminderType string) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // INSERT ... ON CONFLICT DO NOTHING — guarantees exactly-once delivery
        sent, err := s.reminderRepo.InsertIfNotSent(ctx, tx, tenantID, reminderType)
        if err != nil {
            return err
        }
        if !sent {
            return nil // already sent — skip
        }
        return s.mailer.SendTrialReminder(ctx, tenantID, reminderType)
    })
}
```

---

## Trial Feature Limits

Trials typically run with a plan that has reduced limits. Do not hardcode trial limits — use the plan system:

| Approach | How | Advantage |
|---|---|---|
| Separate trial plan | `plan_id = "trial"` with its own limits | Clean separation; easy to change |
| Paid plan with overrides | Same plan + `tenant_entitlement_overrides` | Conversion with no entitlement work |
| Feature flag subset | Trial tenants get a subset of feature flags | Flexible; no plan config needed |

Use **`plan-capability-mapping`** to set trial-specific limits.

---

## Quality Checks

- [ ] Trial `ends_at` is set at creation — never left NULL
- [ ] Expiry job runs at least hourly; failures log per-trial and continue (best-effort)
- [ ] `Convert` is idempotent — second call returns `ErrAlreadyConverted`, not double-charges
- [ ] Trial reminders use `INSERT ... ON CONFLICT DO NOTHING` — exactly-once send
- [ ] Trial extension guarded by `MaxExtensions` and `MaxExtensionDays` constants
- [ ] `EndsAt` tracked alongside `OriginalEndsAt` for reporting and audit
- [ ] Suspended post-expiry tenants cannot access paid features (middleware checks tenant status)
- [ ] Credit card added during trial updates `credit_card_on_file = true` — used to send different reminder copy

## After Completion

- Use **`tenant-lifecycle-management`** for the status transitions that trial expiry triggers
- Use **`subscription-lifecycle`** for the full subscription state machine after trial conversion
- Use **`onboarding-state-machine`** for progress tracking during the trial period
- Use **`activation-flow-design`** for the activation events that often define trial success
- Use **`tenant-suspension-handling`** for what happens when a trial expires without conversion

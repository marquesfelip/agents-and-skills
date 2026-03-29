---
name: reactivation-flow
description: 'Safe account reactivation processes for SaaS tenants after suspension or cancellation. Use when: reactivation flow, account reactivation, reactivate tenant, reactivate subscription, unsuspend account, reactivate canceled account, win back customer, restore account, reactivation within retention window, reactivation after trial expiry, reactivation after billing failure, reactivation safety, reactivation idempotency, reactivation payment, reactivation access restore, re-onboarding, reactivation email, SaaS reactivation, account recovery.'
argument-hint: 'Describe the reactivation scenario (from suspension due to billing, from cancellation within retention window, from trial expiry), whether payment is required to reactivate, and which access rights should be restored (same plan, same data, same team members).'
---

# Reactivation Flow

## When to Use

Invoke this skill when you need to:
- Reactivate a suspended tenant who resolves the suspension cause (pays invoice, clears abuse review)
- Reactivate a canceled tenant who returns within the retention window
- Restore full access: re-enable sessions, restore API keys, resume billing
- Ensure reactivation is idempotent — double-submission does not double-charge or create two subscriptions
- Notify the tenant that access has been restored
- Prevent reactivation past the point of no return (after hard delete)

---

## Reactivation Paths

| From state | Scenario | Payment required | Data intact? | Notes |
|---|---|---|---|---|
| `suspended` (billing_failure) | Customer pays overdue invoice | Yes (implicit — payment triggers) | Yes | Auto-lift via webhook |
| `suspended` (trial_expired) | Customer adds payment method | Yes | Yes | Creates first subscription |
| `suspended` (admin/abuse) | Admin reviews and lifts | No | Yes | Manual admin action only |
| `pending_deletion` | Customer changes mind within retention window | Yes (if billing failure caused cancel) | Yes | Must happen before `deletion_scheduled_at` |
| `deleted` | Customer returns after hard delete | Yes | No (data purged) | Treat as new tenant provisioning |

---

## Step 1 — Reactivation Service

```go
type ReactivationInput struct {
    TenantID        uuid.UUID
    RequestedBy     uuid.UUID       // user initiating (or system for auto-lift)
    PaymentMethodID string          // required when payment needed
    PlanID          string          // plan to restore (defaults to last active plan)
    Actor           Actor
}

type ReactivationService interface {
    ReactivateFromSuspension(ctx context.Context, input ReactivationInput) error
    ReactivateFromPendingDeletion(ctx context.Context, input ReactivationInput) error
    CanReactivate(ctx context.Context, tenantID uuid.UUID) (*ReactivationEligibility, error)
}

type ReactivationEligibility struct {
    Eligible        bool
    CurrentStatus   TenantStatus
    Reason          string          // why not eligible (if Eligible = false)
    PaymentRequired bool
    DataIntact      bool
    ExpiresAt       *time.Time      // for pending_deletion: when reactivation is no longer possible
}
```

---

## Step 2 — Eligibility Check

```go
func (s *reactivationService) CanReactivate(ctx context.Context, tenantID uuid.UUID) (*ReactivationEligibility, error) {
    tenant, err := s.tenantSvc.GetByID(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    switch tenant.Status {
    case TenantStatusSuspended:
        // Abuse/fraud suspensions require admin review — not self-service
        if tenant.SuspensionReason == string(SuspensionAbuse) || tenant.SuspensionReason == string(SuspensionFraud) {
            return &ReactivationEligibility{
                Eligible: false,
                Reason:   "suspension_requires_admin_review",
            }, nil
        }
        return &ReactivationEligibility{
            Eligible:        true,
            CurrentStatus:   tenant.Status,
            PaymentRequired: true,
            DataIntact:      true,
        }, nil

    case TenantStatusPendingDeletion:
        if tenant.DeletionScheduledAt != nil && time.Now().UTC().After(*tenant.DeletionScheduledAt) {
            return &ReactivationEligibility{
                Eligible: false,
                Reason:   "retention_window_expired",
            }, nil
        }
        return &ReactivationEligibility{
            Eligible:        true,
            CurrentStatus:   tenant.Status,
            PaymentRequired: true,
            DataIntact:      true,
            ExpiresAt:       tenant.DeletionScheduledAt,
        }, nil

    case TenantStatusDeleted:
        return &ReactivationEligibility{
            Eligible:   false,
            Reason:     "account_permanently_deleted",
            DataIntact: false,
        }, nil

    default:
        return &ReactivationEligibility{
            Eligible: false,
            Reason:   fmt.Sprintf("not_reactivatable_from_%s", tenant.Status),
        }, nil
    }
}
```

---

## Step 3 — Reactivate from Suspension

```go
func (s *reactivationService) ReactivateFromSuspension(ctx context.Context, input ReactivationInput) error {
    eligibility, err := s.CanReactivate(ctx, input.TenantID)
    if err != nil {
        return err
    }
    if !eligibility.Eligible {
        return fmt.Errorf("%w: %s", ErrNotEligibleForReactivation, eligibility.Reason)
    }

    // Idempotency: check if already active before proceeding
    tenant, err := s.tenantSvc.GetByID(ctx, input.TenantID)
    if err != nil {
        return err
    }
    if tenant.Status == TenantStatusActive {
        return nil // already active — idempotent
    }

    return s.db.Transaction(ctx, func(tx DB) error {
        // 1. Restore or create subscription
        if input.PaymentMethodID != "" {
            if err := s.billing.RestoreSubscription(ctx, billing.RestoreInput{
                TenantID:        input.TenantID,
                PaymentMethodID: input.PaymentMethodID,
                PlanID:          input.PlanID,
            }); err != nil {
                return fmt.Errorf("restore billing: %w", err)
            }
        }

        // 2. Transition tenant to active
        if err := s.tenantSvc.Transition(ctx, tx, input.TenantID, TenantStatusActive, TransitionOptions{
            Actor:  input.Actor,
            Reason: "reactivation_from_suspension",
        }); err != nil {
            return fmt.Errorf("transition to active: %w", err)
        }

        // 3. Restore API keys that were suspended (not revoked)
        if err := s.apiKeyRepo.RestoreSuspended(ctx, tx, input.TenantID); err != nil {
            return fmt.Errorf("restore api keys: %w", err)
        }

        // 4. Clear cancellation record (if one was created during suspension)
        _ = s.cancellationRepo.Reactivate(ctx, tx, input.TenantID)

        return nil
    })
}
```

---

## Step 4 — Reactivate from Pending Deletion

```go
func (s *reactivationService) ReactivateFromPendingDeletion(ctx context.Context, input ReactivationInput) error {
    eligibility, err := s.CanReactivate(ctx, input.TenantID)
    if err != nil {
        return err
    }
    if !eligibility.Eligible {
        return fmt.Errorf("%w: %s", ErrNotEligibleForReactivation, eligibility.Reason)
    }

    return s.db.Transaction(ctx, func(tx DB) error {
        // 1. Clear scheduled deletion timestamps
        if err := s.tenantRepo.ClearDeletion(ctx, tx, input.TenantID); err != nil {
            return fmt.Errorf("clear deletion schedule: %w", err)
        }

        // 2. Restore sessions are NOT restored — user must log in fresh
        // (this is correct — we don't want to silently un-revoke sessions)

        // 3. Create new subscription (billing was canceled during offboarding)
        if input.PaymentMethodID != "" {
            if _, err := s.billing.CreateSubscription(ctx, billing.CreateSubscriptionInput{
                TenantID:        input.TenantID,
                PaymentMethodID: input.PaymentMethodID,
                PlanID:          input.PlanID,
            }); err != nil {
                return fmt.Errorf("create subscription: %w", err)
            }
        }

        // 4. Transition to active
        if err := s.tenantSvc.Transition(ctx, tx, input.TenantID, TenantStatusActive, TransitionOptions{
            Actor:  input.Actor,
            Reason: "reactivation_from_pending_deletion",
        }); err != nil {
            return err
        }

        // 5. Record win-back on cancellation record
        return s.cancellationRepo.Reactivate(ctx, tx, input.TenantID)
    })
}
```

---

## Step 5 — Post-Reactivation Hooks

```go
func (h *ReactivationEventHandler) Handle(ctx context.Context, event TenantReactivatedEvent) error {
    // 1. Send "welcome back" email
    if err := h.mailer.SendReactivationEmail(ctx, event.TenantID); err != nil {
        slog.Error("reactivation email failed", "tenant_id", event.TenantID, "err", err)
    }

    // 2. Notify CRM / sales (win-back tracking)
    h.crm.RecordWinBack(ctx, event.TenantID, event.FromStatus)

    // 3. Analytics
    h.analytics.Track(ctx, event.TenantID, "Tenant Reactivated", map[string]any{
        "from_status":     event.FromStatus,
        "days_since_cancel": event.DaysSinceCancel,
    })

    return nil
}
```

---

## Step 6 — API Keys: Suspend vs Revoke Distinction

Suspension should *suspend* (not revoke) API keys — they must be restorable on reactivation:

```sql
CREATE TYPE api_key_status AS ENUM ('active', 'suspended', 'revoked');

-- On tenant suspend: set status = 'suspended' (reversible)
UPDATE api_keys SET status = 'suspended' WHERE tenant_id = $1 AND status = 'active';

-- On tenant reactivate: restore only 'suspended' keys (not 'revoked' ones — those were intentionally revoked)
UPDATE api_keys SET status = 'active' WHERE tenant_id = $1 AND status = 'suspended';

-- On tenant delete/member remove: status = 'revoked' (irreversible)
UPDATE api_keys SET status = 'revoked' WHERE tenant_id = $1;
```

---

## Reactivation Decision Flowchart

```
User requests reactivation
        ↓
CanReactivate(tenantID)?
    ├─ status = 'deleted'       → "Account permanently deleted — contact support"
    ├─ status = 'active'        → Already active — no-op (idempotent)
    ├─ suspension_reason = 'abuse'/'fraud' → "Admin review required"
    ├─ pending_deletion after scheduled_at → "Retention window expired — re-register"
    ├─ suspended (billing/trial) → Prompt for payment → RestoreSubscription → Lift
    └─ pending_deletion (in window) → Prompt for payment → CreateSubscription → ClearDeletion → Active
```

---

## Quality Checks

- [ ] `CanReactivate` is called before any reactivation attempt — returns structured eligibility
- [ ] Reactivation is idempotent — calling twice when already `active` returns success, no double-charge
- [ ] Abuse/fraud suspensions return `eligible: false` — not self-service reactivatable
- [ ] `pending_deletion` reactivation checks `deletion_scheduled_at` is in the future
- [ ] API keys are *suspended* during tenant suspension — not revoked — so they restore cleanly
- [ ] Sessions are NOT restored on reactivation — users must log in fresh (security boundary)
- [ ] Win-back is recorded on `cancellation_records.reactivated_at` for churn analytics
- [ ] Post-reactivation event fires after the DB transaction commits — not inside it

## After Completion

- Use **`tenant-suspension-handling`** for the logic that suspends and the auto-lift webhook path
- Use **`tenant-lifecycle-management`** for the full state machine that reactivation transitions through
- Use **`customer-offboarding`** for the cancellation flow reactivation reverses
- Use **`payment-failure-recovery`** for the billing retry flow that may trigger reactivation automatically
- Use **`subscription-lifecycle`** for managing the subscription re-creation on reactivation

---
name: tenant-suspension-handling
description: 'Suspension logic tied to billing failure or abuse for SaaS tenants. Use when: tenant suspension, suspend tenant, suspension logic, billing suspension, abuse suspension, suspend on payment failure, suspend on dunning exhausted, suspension gate, suspended tenant access, suspension reason, lift suspension, unsuspend tenant, suspension audit, suspension notification, auto-suspend, suspension policy, grace period suspension, trial expiry suspension, admin suspend, suspension middleware, feature lock suspension.'
argument-hint: 'Describe the suspension trigger (billing failure, abuse detection, admin action, trial expiry), what access the tenant retains while suspended (read-only, export-only, none), and how suspension is lifted (pay invoice, admin review, add payment method).'
---

# Tenant Suspension Handling

## When to Use

Invoke this skill when you need to:
- Suspend a tenant automatically when dunning is exhausted or abuse is detected
- Define exactly what access a suspended tenant retains (read-only, export-only, none)
- Gate all write operations behind a suspension check in middleware
- Lift a suspension cleanly when the trigger is resolved (payment collected, abuse cleared)
- Notify the tenant about suspension cause, impact, and how to resolve it
- Audit all suspension events with reason, actor, and timestamps

---

## Suspension Trigger → Policy Table

| Trigger | Suspension type | Access retained | Lifted by | Auto-lift? |
|---|---|---|---|---|
| Dunning exhausted (all payment retries failed) | `billing_failure` | Export-only | Successful payment | Yes — on payment event |
| Trial expired without conversion | `trial_expired` | Export-only | Add payment method | Yes — on first payment |
| Abuse detected (ToS violation) | `abuse` | None | Admin review + lift | No — manual only |
| Admin manual suspension | `admin` | Admin-configurable | Admin lifts | No — manual only |
| Fraud signal (payment chargeback) | `fraud` | None | Admin review | No — manual only |
| Resource quota exceeded (overage) | `quota_exceeded` | Read-only within quota | Upgrade plan or reduce usage | Yes — on plan upgrade |

---

## Step 1 — Suspension Schema

```sql
-- Already on tenants table from tenant-lifecycle-management:
-- status          tenant_status NOT NULL
-- suspended_at    TIMESTAMPTZ
-- suspension_reason TEXT  -- 'billing_failure' | 'trial_expired' | 'abuse' | 'admin' | 'fraud' | 'quota_exceeded'

-- Detailed suspension events for audit
CREATE TABLE suspension_events (
    id               BIGSERIAL PRIMARY KEY,
    tenant_id        UUID      NOT NULL REFERENCES tenants(id),
    event_type       TEXT      NOT NULL,  -- 'suspended' | 'lifted' | 'access_denied'
    suspension_type  TEXT,               -- 'billing_failure' | 'abuse' | etc.
    actor_id         UUID      REFERENCES users(id),
    actor_type       TEXT      NOT NULL, -- 'system' | 'admin' | 'user'
    metadata         JSONB,
    occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_suspension_events_tenant ON suspension_events(tenant_id, occurred_at DESC);
```

---

## Step 2 — Suspension Service

```go
type SuspensionType string

const (
    SuspensionBillingFailure SuspensionType = "billing_failure"
    SuspensionTrialExpired   SuspensionType = "trial_expired"
    SuspensionAbuse          SuspensionType = "abuse"
    SuspensionAdmin          SuspensionType = "admin"
    SuspensionFraud          SuspensionType = "fraud"
    SuspensionQuotaExceeded  SuspensionType = "quota_exceeded"
)

type SuspendInput struct {
    TenantID  uuid.UUID
    Type      SuspensionType
    Actor     Actor
    Reason    string         // human-readable detail for admin UI
    Metadata  map[string]any // structured context (e.g., invoice ID, abuse signal)
}

func (s *suspensionService) Suspend(ctx context.Context, input SuspendInput) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // Use state machine guard from tenant-lifecycle-management
        if err := s.tenantSvc.Transition(ctx, tx, input.TenantID, TenantStatusSuspended, TransitionOptions{
            Actor:    input.Actor,
            Reason:   string(input.Type),
            Metadata: input.Metadata,
        }); err != nil {
            return fmt.Errorf("suspend transition: %w", err)
        }

        // Record suspension event
        if err := s.eventRepo.Insert(ctx, tx, SuspensionEvent{
            TenantID:       input.TenantID,
            EventType:      "suspended",
            SuspensionType: string(input.Type),
            ActorID:        input.Actor.ID,
            ActorType:      input.Actor.Type,
            Metadata:       input.Metadata,
        }); err != nil {
            return err
        }

        // Revoke all active sessions immediately
        if err := s.sessionStore.RevokeAllForTenant(ctx, input.TenantID); err != nil {
            return fmt.Errorf("revoke sessions: %w", err)
        }

        return nil
    })
}

func (s *suspensionService) Lift(ctx context.Context, tenantID uuid.UUID, actor Actor, reason string) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        if err := s.tenantSvc.Transition(ctx, tx, tenantID, TenantStatusActive, TransitionOptions{
            Actor:  actor,
            Reason: "suspension_lifted: " + reason,
        }); err != nil {
            return fmt.Errorf("lift suspension transition: %w", err)
        }

        return s.eventRepo.Insert(ctx, tx, SuspensionEvent{
            TenantID:  tenantID,
            EventType: "lifted",
            ActorID:   actor.ID,
            ActorType: actor.Type,
            Metadata:  map[string]any{"reason": reason},
        })
    })
}
```

---

## Step 3 — Suspension Middleware

Gate all incoming requests — applied before business logic:

```go
type SuspensionPolicy struct {
    AllowedMethods []string  // HTTP methods allowed while suspended
    AllowedPaths   []string  // path prefixes allowed while suspended
}

// Per suspension type: what is still accessible?
var suspensionPolicies = map[SuspensionType]SuspensionPolicy{
    SuspensionBillingFailure: {
        AllowedMethods: []string{"GET"},
        AllowedPaths:   []string{"/api/export", "/api/billing", "/api/account"},
    },
    SuspensionTrialExpired: {
        AllowedMethods: []string{"GET"},
        AllowedPaths:   []string{"/api/export", "/api/billing"},
    },
    SuspensionAbuse: {
        AllowedMethods: []string{},
        AllowedPaths:   []string{}, // no access
    },
    SuspensionAdmin: {
        AllowedMethods: []string{"GET"},
        AllowedPaths:   []string{"/api/export"},
    },
    SuspensionFraud: {
        AllowedMethods: []string{},
        AllowedPaths:   []string{},
    },
    SuspensionQuotaExceeded: {
        AllowedMethods: []string{"GET", "DELETE"}, // read + reduce usage
        AllowedPaths:   []string{},                // all paths
    },
}

func SuspensionGate(tenantSvc TenantService, eventRepo SuspensionEventRepo) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := auth.TenantFromCtx(r.Context())
            tenant, err := tenantSvc.GetByID(r.Context(), tenantID)
            if err != nil || tenant.Status != TenantStatusSuspended {
                next.ServeHTTP(w, r)
                return
            }

            suspType := SuspensionType(tenant.SuspensionReason)
            policy := suspensionPolicies[suspType]

            allowed := false
            for _, p := range policy.AllowedPaths {
                if strings.HasPrefix(r.URL.Path, p) {
                    for _, m := range policy.AllowedMethods {
                        if m == r.Method {
                            allowed = true
                            break
                        }
                    }
                }
            }
            // Empty AllowedPaths + non-empty AllowedMethods = all paths, method restricted
            if len(policy.AllowedPaths) == 0 && len(policy.AllowedMethods) > 0 {
                for _, m := range policy.AllowedMethods {
                    if m == r.Method {
                        allowed = true
                        break
                    }
                }
            }

            if !allowed {
                // Record access denied for audit
                _ = eventRepo.Insert(r.Context(), nil, SuspensionEvent{
                    TenantID:  tenantID,
                    EventType: "access_denied",
                    Metadata:  map[string]any{"path": r.URL.Path, "method": r.Method},
                })
                render.JSON(w, http.StatusForbidden, map[string]any{
                    "error":            "account_suspended",
                    "suspension_type":  tenant.SuspensionReason,
                    "resolve_url":      billingPortalURL(tenantID, suspType),
                })
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

---

## Step 4 — Auto-Lift on Billing Recovery

```go
// Called from billing webhook handler when payment succeeds
func (s *suspensionService) HandlePaymentSucceeded(ctx context.Context, tenantID uuid.UUID, invoiceID string) error {
    tenant, err := s.tenantSvc.GetByID(ctx, tenantID)
    if err != nil {
        return err
    }

    // Only auto-lift billing or trial suspensions — not abuse/fraud
    autoLiftTypes := map[SuspensionType]bool{
        SuspensionBillingFailure: true,
        SuspensionTrialExpired:   true,
    }

    if tenant.Status == TenantStatusSuspended && autoLiftTypes[SuspensionType(tenant.SuspensionReason)] {
        return s.Lift(ctx, tenantID, SystemActor, "payment_succeeded: "+invoiceID)
    }
    return nil
}
```

---

## Step 5 — Suspension Notification

```go
func (h *SuspensionEventHandler) Handle(ctx context.Context, event TenantSuspendedEvent) error {
    // Personalize message per suspension type
    templates := map[SuspensionType]string{
        SuspensionBillingFailure: "suspension_billing_failure",
        SuspensionTrialExpired:   "suspension_trial_expired",
        SuspensionAbuse:          "suspension_abuse",
        SuspensionAdmin:          "suspension_admin",
    }

    tmpl, ok := templates[event.Type]
    if !ok {
        tmpl = "suspension_generic"
    }

    return h.mailer.SendToTenantOwners(ctx, event.TenantID, tmpl, map[string]any{
        "suspension_type": event.Type,
        "resolve_url":     billingPortalURL(event.TenantID, event.Type),
    })
}
```

---

## Quality Checks

- [ ] `SuspensionGate` middleware applied globally — cannot be bypassed by individual route handlers
- [ ] Session revocation happens in the same DB transaction as the suspension state change
- [ ] Abuse and fraud suspensions are NOT auto-lifted — require explicit admin action
- [ ] Suspension type stored in `tenants.suspension_reason` — determines access policy without extra query
- [ ] `suspension_events` table records every `suspended`, `lifted`, and `access_denied` event
- [ ] Tenant owners receive a suspension email with the reason and a resolution URL
- [ ] `Lift` uses the state machine guard — cannot lift if tenant is in `deleted` or `pending_deletion`
- [ ] `HandlePaymentSucceeded` is idempotent — calling twice does not double-lift

## After Completion

- Use **`tenant-lifecycle-management`** for the full state machine that governs suspension transitions
- Use **`payment-failure-recovery`** for the dunning flow that precedes billing-failure suspension
- Use **`reactivation-flow`** for the full path back to `active` after suspension is lifted
- Use **`billing-to-access-sync`** to wire billing webhook events to suspension and lift actions
- Use **`abuse-detection`** to identify the signals that trigger abuse suspensions

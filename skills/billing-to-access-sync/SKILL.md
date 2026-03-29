---
name: billing-to-access-sync
description: 'Define and implement synchronization between billing state and system access rights for SaaS. Use when: billing to access sync, billing access synchronization, subscription access sync, billing state access rights, entitlement sync billing, billing event access update, access revocation billing, billing transition access, subscription event access, plan change access sync, trial access sync, cancellation access revocation, grace period access, billing state entitlement sync, Stripe subscription access sync, billing webhook access update, billing driven access control.'
argument-hint: 'Describe your billing provider (Stripe, Paddle, etc.), the subscription state transitions you handle (trial→active, past_due→canceled, etc.), and where your access control source of truth lives (tenant_entitlements table, cache, feature flags).'
---

# Billing-to-Access Sync

## When to Use

Invoke this skill when you need to:
- Wire billing lifecycle events (webhook, state machine transitions) to access right updates
- Ensure entitlements are updated immediately when billing state changes
- Handle access during edge cases: grace periods, trial end, reactivation, plan upgrades/downgrades
- Prevent access drift where a canceled subscription still has active feature access
- Design a reliable, idempotent sync pipeline that works even when events arrive out of order
- Test that billing transitions produce the correct access rights in all states

---

## Core Concept: Sync Architecture

```
Billing provider (Stripe)
        │
        ▼
[Webhook receiver] ─────────────────────────────────┐
        │                                           │
        ▼                                           ▼
[Billing State Machine]                    [Idempotency store]
  subscription.status updated                (processed_events)
        │
        ▼
[EntitlementSyncer.SyncForTenant(tenantID)]
        │
        ├── Load active subscription + plan
        ├── Rebuild tenant_entitlements rows
        ├── Invalidate entitlement cache
        └── Emit AccessSynced domain event (optional)
```

**Invariant:** Every subscription state transition MUST trigger an entitlement sync. No exception.

---

## Step 1 — Access State Decision Table

Map every subscription status to an access state:

| Subscription status | Access granted? | Rationale |
|---|---|---|
| `trialing` | ✅ Yes | Trial access (may include trial bonuses) |
| `active` | ✅ Yes | Paid, current |
| `past_due` | ✅ Yes (time-limited) | Grace period — keep access while retrying payment |
| `unpaid` | ❌ No | Grace period expired |
| `canceled` | ❌ No | Voluntarily or involuntarily canceled |
| `paused` | ❌ No | Voluntary pause |
| `incomplete` | ❌ No | Initial payment failed — never fully activated |
| *(no subscription)* | ❌ No | Free tier (use plan_id = 'free') or no plan at all |

```go
func (sub *Subscription) GrantsAccess() bool {
    switch sub.Status {
    case "trialing", "active", "past_due":
        return true
    default:
        return false
    }
}
```

---

## Step 2 — Entitlement Syncer

The syncer is the single function responsible for translating billing state into access rows:

```go
type EntitlementSyncer struct {
    subRepo         SubscriptionRepository
    planFeatureRepo PlanFeatureRepository
    planLimitRepo   PlanLimitRepository
    entitlementRepo EntitlementRepository
    addOnRepo       TenantAddOnRepository
    cache           EntitlementCache
    db              DB
    logger          *slog.Logger
}

func (s *EntitlementSyncer) SyncForTenant(ctx context.Context, tenantID string) error {
    // 1. Fetch active subscription (nil = no active sub)
    sub, err := s.subRepo.GetActiveForTenant(ctx, tenantID)
    if err != nil && !errors.Is(err, ErrNotFound) {
        return fmt.Errorf("fetching subscription: %w", err)
    }

    return s.db.WithTx(ctx, func(ctx context.Context, tx pgx.Tx) error {
        // 2. No access case — revoke all plan-derived entitlements
        if sub == nil || !sub.GrantsAccess() {
            if err := s.entitlementRepo.RevokePlanEntitlementsTx(ctx, tx, tenantID); err != nil {
                return err
            }
            s.logger.Info("entitlements revoked", slog.String("tenant_id", tenantID),
                slog.String("reason", statusReason(sub)))
            s.cache.InvalidateTenant(ctx, tenantID)
            return nil
        }

        // 3. Load plan capabilities
        features, err := s.planFeatureRepo.GetForPlan(ctx, sub.PlanID)
        if err != nil {
            return fmt.Errorf("loading plan features: %w", err)
        }
        limits, err := s.planLimitRepo.GetForPlan(ctx, sub.PlanID)
        if err != nil {
            return fmt.Errorf("loading plan limits: %w", err)
        }

        // 4. Build entitlement rows from plan
        ents := buildPlanEntitlements(tenantID, features, limits)

        // 5. Layer trial bonuses over plan entitlements (if trialing)
        if sub.IsTrialing() {
            ents = applyTrialBonuses(ents, tenantID, sub.TrialEnd)
        }

        // 6. Layer add-on entitlements
        addOns, err := s.addOnRepo.GetActiveForTenant(ctx, tenantID)
        if err != nil {
            return fmt.Errorf("loading add-ons: %w", err)
        }
        ents = applyAddOns(ents, tenantID, addOns)

        // 7. Upsert all computed entitlements atomically
        for _, ent := range ents {
            if err := s.entitlementRepo.UpsertTx(ctx, tx, ent); err != nil {
                return fmt.Errorf("upserting entitlement %s: %w", ent.FeatureKey, err)
            }
        }

        // 8. Remove stale entitlements no longer in the plan (e.g., downgrade)
        if err := s.entitlementRepo.RemoveStaleplanEntitlementsTx(ctx, tx, tenantID, planFeatureKeys(ents)); err != nil {
            return err
        }

        s.logger.Info("entitlements synced",
            slog.String("tenant_id", tenantID),
            slog.String("status", sub.Status),
            slog.Int("feature_count", len(ents)),
        )
        return nil
    })
    // Cache invalidation AFTER transaction commits
    s.cache.InvalidateTenant(ctx, tenantID)
    return nil
}
```

---

## Step 3 — Trigger Points

The syncer must be called from every billing state transition:

### A. Stripe Webhook Handler

```go
func (h *WebhookHandler) handleCustomerSubscriptionUpdated(ctx context.Context, event stripe.Event) error {
    var sub stripe.Subscription
    if err := json.Unmarshal(event.Data.Raw, &sub); err != nil {
        return err
    }

    tenantID, err := h.tenantRepo.GetByStripeCustomerID(ctx, sub.Customer.ID)
    if err != nil {
        return err
    }

    // Update local subscription record first
    if err := h.subRepo.SyncFromStripe(ctx, &sub); err != nil {
        return err
    }

    // Then sync entitlements from the new state
    return h.syncer.SyncForTenant(ctx, tenantID)
}

// Wire all relevant Stripe event types:
var syncTriggerEvents = []string{
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
    "customer.subscription.trial_will_end",
    "invoice.paid",
    "invoice.payment_failed",
}
```

### B. State Machine Transition Effect

```go
// EntitlementSyncEffect hooks into every state machine transition
type EntitlementSyncEffect struct {
    syncer *EntitlementSyncer
    logger *slog.Logger
}

func (e *EntitlementSyncEffect) OnTransition(ctx context.Context, sub *Subscription, from, to string) {
    // Run sync asynchronously after the billing DB transaction commits
    go func() {
        syncCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        if err := e.syncer.SyncForTenant(syncCtx, sub.TenantID); err != nil {
            e.logger.Error("entitlement sync after transition failed",
                slog.String("tenant_id", sub.TenantID),
                slog.String("from", from),
                slog.String("to", to),
                slog.String("err", err.Error()),
            )
            // TODO: enqueue retry via background job
        }
    }()
}
```

### C. Plan Change (Upgrade / Downgrade)

```go
func (s *SubscriptionService) ChangePlan(ctx context.Context, tenantID, newPlanID string) error {
    // 1. Update Stripe subscription item
    if err := s.stripeClient.UpdateSubscriptionPlan(ctx, tenantID, newPlanID); err != nil {
        return err
    }

    // 2. Update local subscription record
    if err := s.subRepo.UpdatePlan(ctx, tenantID, newPlanID); err != nil {
        return err
    }

    // 3. Sync entitlements immediately — plan change is instant access update
    return s.syncer.SyncForTenant(ctx, tenantID)
}
```

---

## Step 4 — Idempotency and Out-of-Order Safety

```go
// processed_webhook_events prevents double-processing of Stripe events
// (see stripe-webhook-processing skill for full implementation)

// Sync is safe to call multiple times — uses UPSERT semantics
// Re-running sync for the same tenant produces identical results (idempotent)

// Out-of-order webhook safety:
// Always derive access from the CURRENT subscription record in your DB,
// not from the event payload. By the time you process, a later event may
// have already updated the subscription — read your own state, not Stripe's event snapshot.

func (h *WebhookHandler) handleEvent(ctx context.Context, event stripe.Event) error {
    if h.isAlreadyProcessed(ctx, event.ID) {
        return nil // idempotent skip
    }

    tenantID, err := h.resolveTenant(ctx, event)
    if err != nil {
        return err
    }

    // Update subscription from Stripe (idempotent upsert)
    if err := h.subRepo.SyncFromStripe(ctx, event); err != nil {
        return err
    }

    // Sync entitlements from current DB state (not event payload)
    if err := h.syncer.SyncForTenant(ctx, tenantID); err != nil {
        return err
    }

    return h.markProcessed(ctx, event.ID)
}
```

---

## Step 5 — Grace Period Handling

For `past_due` subscriptions, access continues during the retry window:

```go
// Grace period is implicit in the access state table (past_due = access granted)
// When grace period expires, Stripe emits 'customer.subscription.updated' with status='unpaid'
// → SyncForTenant revokes entitlements at that point

// To enforce a custom grace period deadline (e.g., 7 days from first payment failure):
func (s *GracePeriodJob) ExpireGracePeriods(ctx context.Context) error {
    expired, err := s.subRepo.FindExpiredGracePeriods(ctx, time.Now())
    if err != nil {
        return err
    }
    for _, sub := range expired {
        if err := s.subRepo.SetStatus(ctx, sub.ID, "unpaid"); err != nil {
            s.logger.Error("failed to expire grace period", slog.String("sub_id", sub.ID))
            continue
        }
        s.syncer.SyncForTenant(ctx, sub.TenantID) //nolint: errcheck — logged inside
    }
    return nil
}
```

---

## Step 6 — Operational Validation

```sql
-- Detect entitlement drift: tenants with active subscriptions but no entitlements
SELECT s.tenant_id, s.status
FROM subscriptions s
LEFT JOIN tenant_entitlements te ON te.tenant_id = s.tenant_id
WHERE s.status IN ('trialing','active','past_due')
  AND te.tenant_id IS NULL;
-- Expected: 0 rows. Non-zero = sync missed a transition.

-- Detect stale entitlements: tenants with entitlements but no active subscription
SELECT te.tenant_id, COUNT(*) as entitlement_count
FROM tenant_entitlements te
LEFT JOIN subscriptions s ON s.tenant_id = te.tenant_id
    AND s.status IN ('trialing','active','past_due')
WHERE te.source = 'plan'
  AND s.tenant_id IS NULL
GROUP BY te.tenant_id;
-- Expected: 0 rows. Non-zero = access not revoked after cancellation.
```

Run these queries as part of a daily monitoring job and alert on non-zero results.

---

## Quality Checks

- [ ] Every Stripe subscription event type that affects access triggers `SyncForTenant`
- [ ] Sync reads from your DB subscription state — NOT directly from the Stripe event payload
- [ ] Sync is idempotent — safe to call multiple times without side effects
- [ ] Cache is invalidated AFTER the DB transaction commits, not before
- [ ] Grace period (`past_due`) is included in the "grants access" status set
- [ ] Plan downgrades remove stale entitlements from the old plan's feature set
- [ ] Add-on entitlements are re-layered on every sync — not just on add-on purchase
- [ ] Drift detection queries run daily and alert on non-zero results
- [ ] Sync failures are logged and enqueued for retry — not silently swallowed

## After Completion

You have a reliable billing-to-access sync pipeline. Recommended next steps:

- Use **`entitlement-system-design`** to review the three-layer separation model
- Use **`quota-enforcement-runtime`** to reset periodic usage counters on plan changes
- Use **`grace-period-handling`** for detailed dunning and access-during-failure patterns
- Use **`billing-state-machine`** for the full subscription state transition model

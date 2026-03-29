---
name: billing-state-machine
description: 'Billing state transitions and synchronization logic for SaaS products. Use when: billing state machine, subscription state transitions, subscription status sync, stripe status synchronization, billing state transition guard, subscription state model, billing state database, stripe subscription status, billing event driven state, billing state machine design, subscription status change, invalid transition prevention, billing state audit, concurrent state update, stripe state sync, subscription state diagram, billing status enum, state machine enforcement, optimistic lock billing state, billing transition trigger.'
argument-hint: 'Describe your subscription states (trialing, active, past_due, canceled, etc.), which Stripe events trigger transitions, and whether you have concurrency concerns with simultaneous webhook delivery.'
---

# Billing State Machine

## When to Use

Invoke this skill when you need to:
- Define all valid subscription/billing states and their allowed transitions
- Enforce transitions at the database and application layer (invalid transitions rejected)
- Synchronize Stripe subscription status to your local database state
- Handle concurrent webhook events without corrupting billing state
- Emit downstream effects (access changes, emails, audit logs) on each transition

---

## Step 1 — Define All States

**Canonical billing states (aligned with Stripe):**

| State | Stripe Equivalent | Meaning | Feature Access |
|---|---|---|---|
| `trialing` | `trialing` | Active trial; payment may or may not be on file | Full or feature-limited |
| `active` | `active` | Paid, current, within billing period | Full |
| `past_due` | `past_due` | Payment failed; provider retrying | Grace period |
| `unpaid` | `unpaid` | All retries exhausted | Locked / read-only |
| `incomplete` | `incomplete` | Initial payment pending (async methods) | None |
| `incomplete_expired` | `incomplete_expired` | Initial payment window expired (23h) | None |
| `paused` | `paused` | Voluntarily paused (if enabled) | None |
| `canceled` | `canceled` | Subscription ended | None |

**Valid transitions:**
```
trialing            → active              (trial converts: payment succeeds)
trialing            → canceled            (trial ends without conversion; or manual cancel)
trialing            → past_due            (trial ended, initial payment fails)
active              → past_due            (renewal payment failure)
active              → canceled            (cancel_at_period_end or immediate cancel)
active              → paused              (voluntary pause)
past_due            → active              (retry payment succeeds)
past_due            → unpaid              (all retries exhausted)
past_due            → canceled            (cancel during grace period)
unpaid              → active              (customer updates payment method + pays)
unpaid              → canceled            (admin or customer cancels)
paused              → active              (customer resumes)
canceled            → active             (reactivation — new Stripe subscription)
incomplete          → active              (initial async payment confirmed)
incomplete          → incomplete_expired  (23h window elapsed)
incomplete_expired  → (terminal)          (must create new subscription)
```

---

## Step 2 — Enforce Transitions in Code

```go
// billing/state_machine.go

var validTransitions = map[string]map[string]bool{
    "trialing":           {"active": true, "canceled": true, "past_due": true},
    "active":             {"past_due": true, "canceled": true, "paused": true},
    "past_due":           {"active": true, "unpaid": true, "canceled": true},
    "unpaid":             {"active": true, "canceled": true},
    "paused":             {"active": true, "canceled": true},
    "canceled":           {"active": true},
    "incomplete":         {"active": true, "incomplete_expired": true},
    "incomplete_expired": {},
}

type StateMachine struct {
    repo      SubscriptionRepository
    effects   TransitionEffects
    auditLog  AuditLogger
}

func (sm *StateMachine) Transition(ctx context.Context, subID, toState string, reason string) error {
    // Load with SELECT FOR UPDATE to prevent concurrent transitions
    sub, err := sm.repo.GetForUpdate(ctx, subID)
    if err != nil {
        return fmt.Errorf("load subscription: %w", err)
    }

    if sub.Status == toState {
        return nil // Already in target state — idempotent
    }

    allowed, ok := validTransitions[sub.Status]
    if !ok || !allowed[toState] {
        return fmt.Errorf("invalid billing transition: %s → %s (reason: %s)", sub.Status, toState, reason)
    }

    fromState := sub.Status
    sub.Status = toState
    sub.UpdatedAt = time.Now()

    if err := sm.repo.UpdateStatus(ctx, sub); err != nil {
        return fmt.Errorf("persist transition: %w", err)
    }

    // Emit downstream effects
    sm.effects.OnTransition(ctx, sub, fromState, toState, reason)

    // Audit log
    sm.auditLog.Record(ctx, AuditEvent{
        EntityType: "subscription",
        EntityID:   subID,
        Action:     fmt.Sprintf("status.%s→%s", fromState, toState),
        Actor:      "billing-state-machine",
        Metadata:   map[string]string{"reason": reason},
    })

    return nil
}
```

---

## Step 3 — Map Stripe Events to Transitions

```go
// Stripe event → local transition mapping
var stripeEventTransitions = map[string]func(stripe.Subscription) (string, string){
    "customer.subscription.updated": func(sub stripe.Subscription) (string, string) {
        return string(sub.Status), "stripe.subscription.updated"
    },
    "invoice.paid": func(_ stripe.Subscription) (string, string) {
        return "active", "invoice.paid"
    },
    "invoice.payment_failed": func(sub stripe.Subscription) (string, string) {
        if sub.Status == stripe.SubscriptionStatusUnpaid {
            return "unpaid", "invoice.payment_failed.all_retries_exhausted"
        }
        return "past_due", "invoice.payment_failed"
    },
    "customer.subscription.deleted": func(_ stripe.Subscription) (string, string) {
        return "canceled", "stripe.subscription.deleted"
    },
}

func (p *WebhookProcessor) syncSubscriptionState(ctx context.Context, event stripe.Event, stripeSub stripe.Subscription) error {
    sub, err := p.subRepo.GetByStripeID(ctx, stripeSub.ID)
    if err != nil {
        return err
    }

    mapper, ok := stripeEventTransitions[event.Type]
    if !ok {
        return nil
    }

    targetState, reason := mapper(stripeSub)
    return p.stateMachine.Transition(ctx, sub.ID, targetState, reason)
}
```

---

## Step 4 — Database-Level State Constraint

Enforce valid states at the DB layer as a final safety net:

```sql
ALTER TABLE subscriptions
    ADD CONSTRAINT chk_subscription_status
    CHECK (status IN (
        'trialing', 'active', 'past_due', 'unpaid',
        'incomplete', 'incomplete_expired', 'paused', 'canceled'
    ));
```

**Optimistic locking to prevent concurrent state corruption:**
```sql
ALTER TABLE subscriptions ADD COLUMN version INT NOT NULL DEFAULT 1;
```

```go
// Update with version check
const updateQuery = `
    UPDATE subscriptions
    SET status = $1, version = version + 1, updated_at = NOW()
    WHERE id = $2 AND version = $3
`

result, err := db.ExecContext(ctx, updateQuery, newStatus, subID, currentVersion)
if err != nil {
    return err
}
if rows, _ := result.RowsAffected(); rows == 0 {
    return ErrConcurrentModification // Retry or raise conflict error
}
```

---

## Step 5 — Transition Effects Interface

Decouple state transitions from downstream side-effects:

```go
type TransitionEffects interface {
    OnTransition(ctx context.Context, sub *Subscription, from, to, reason string)
}

type CompositeTransitionEffects struct {
    effects []TransitionEffects
}

func (c *CompositeTransitionEffects) OnTransition(ctx context.Context, sub *Subscription, from, to, reason string) {
    for _, e := range c.effects {
        e.OnTransition(ctx, sub, from, to, reason)
    }
}

// Registered effects:
// - EntitlementSyncer    → update feature access
// - EmailNotifier        → send dunning, cancellation, reactivation emails
// - MetricsRecorder      → increment billing_transitions_total counter
// - AuditLogger          → append to audit trail
```

---

## Quality Checks

- [ ] All valid states are defined in a constant or enum — no magic strings outside this module
- [ ] Transition guard table is the single source of truth — no ad-hoc `sub.Status = "active"` elsewhere
- [ ] `SELECT FOR UPDATE` (or equivalent) used to prevent concurrent transition races
- [ ] DB `CHECK` constraint enforces valid state values — application bugs caught at DB layer
- [ ] Stripe status values are mapped through a translation layer — Stripe's string is not stored directly
- [ ] Every transition emits to the audit log with `from`, `to`, and `reason`
- [ ] Invalid transition attempts are logged as warnings and return an error — never silently ignored
- [ ] Transition effects (entitlements, emails, metrics) are decoupled via an interface

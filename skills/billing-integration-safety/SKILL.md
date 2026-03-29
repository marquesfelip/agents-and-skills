---
name: billing-integration-safety
description: >
  Safe billing system integrations and state synchronization for SaaS products.
  Use when: billing integration, Stripe integration, payment provider, webhook handling,
  billing state sync, subscription sync, invoice webhook, payment webhook, idempotent billing,
  billing event processing, failed payment handling, dunning, proration, billing reconciliation,
  billing race condition, double charge prevention, billing idempotency key, billing event ordering,
  Stripe webhook signature, billing webhook security, subscription update sync, plan change,
  payment method update, refund handling, billing audit trail, billing retry, billing failure.
argument-hint: >
  Describe your billing provider (Stripe, Paddle, Chargebee, etc.), the current integration
  issue (e.g., "webhook events arriving out of order", "subscription state out of sync after
  failed payment", "double charging on retry"), and whether you are handling one-time or
  recurring payments.
---

# Billing Integration Safety Specialist

## When to Use

Invoke this skill when you need to:
- Design a safe webhook handler for a billing provider (Stripe, Paddle, Chargebee)
- Prevent double-charging or state corruption from duplicate or out-of-order events
- Synchronize subscription state between the billing provider and your database
- Handle failed payments, dunning, and grace periods
- Build a billing audit trail for dispute resolution and reconciliation
- Test billing flows without real charges

---

## Step 1 — Webhook Endpoint Security

The billing webhook endpoint receives real financial events — it must be hardened.

**Stripe webhook signature verification (Go):**
```go
import "github.com/stripe/stripe-go/v76/webhook"

func (h *BillingWebhookHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    const maxBodyBytes = int64(65536)
    r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)

    payload, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "request too large", http.StatusRequestEntityTooLarge)
        return
    }

    // Verify signature — reject anything not signed by Stripe
    event, err := webhook.ConstructEvent(
        payload,
        r.Header.Get("Stripe-Signature"),
        os.Getenv("STRIPE_WEBHOOK_SECRET"), // from Stripe Dashboard
    )
    if err != nil {
        slog.Warn("invalid webhook signature", slog.String("err", err.Error()))
        http.Error(w, "invalid signature", http.StatusBadRequest)
        return
    }

    // Respond 200 immediately — process asynchronously
    w.WriteHeader(http.StatusOK)
    go h.processEvent(context.Background(), event)
}
```

**Critical security rules:**
- Always verify the signature using the provider's SDK — never trust the payload body alone
- Return `200 OK` immediately after signature verification — processing failures trigger retries
- Never expose raw error details in the response — providers log failures; don't leak internals
- Restrict webhook endpoint access at the network level (allow only provider IP ranges in addition to signature verification)

Checklist:
- [ ] Webhook signature verified on every request before any processing
- [ ] Endpoint returns `200 OK` before processing — processing happens asynchronously or in a queue
- [ ] Request body size limited — prevents oversized payload attacks
- [ ] Webhook secret stored in environment variable or vault — never hardcoded

---

## Step 2 — Idempotent Event Processing

Billing providers retry webhook delivery on non-200 responses. Your handler must be idempotent.

**Idempotency with event deduplication:**
```sql
CREATE TABLE billing_events_processed (
    event_id       TEXT        PRIMARY KEY,  -- provider's event ID (e.g., evt_abc123)
    event_type     TEXT        NOT NULL,
    tenant_id      UUID,
    processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    result         JSONB       -- serialized outcome for debugging
);
```

```go
func (h *Processor) processEvent(ctx context.Context, event stripe.Event) {
    // Check if already processed
    if h.alreadyProcessed(ctx, event.ID) {
        slog.Info("duplicate webhook ignored", slog.String("event_id", event.ID))
        return
    }

    var processingErr error
    switch event.Type {
    case "customer.subscription.updated":
        processingErr = h.handleSubscriptionUpdated(ctx, event)
    case "invoice.payment_failed":
        processingErr = h.handlePaymentFailed(ctx, event)
    case "invoice.payment_succeeded":
        processingErr = h.handlePaymentSucceeded(ctx, event)
    case "customer.subscription.deleted":
        processingErr = h.handleSubscriptionDeleted(ctx, event)
    default:
        slog.Debug("unhandled webhook event type", slog.String("type", event.Type))
    }

    if processingErr != nil {
        slog.Error("billing event processing failed",
            slog.String("event_id", event.ID),
            slog.String("type", event.Type),
            slog.String("err", processingErr.Error()),
        )
        return // do NOT mark as processed — provider will retry
    }

    // Mark as processed only on success
    h.markProcessed(ctx, event.ID, event.Type)
}
```

**Mark processed atomically with the state change (database transaction):**
```go
func (h *Processor) handlePaymentSucceeded(ctx context.Context, event stripe.Event) error {
    var invoice stripe.Invoice
    if err := json.Unmarshal(event.Data.Raw, &invoice); err != nil {
        return err
    }

    return h.db.Transaction(ctx, func(tx pgx.Tx) error {
        // Both operations in same transaction — atomically marks processed + updates state
        _, err := tx.Exec(ctx,
            `INSERT INTO billing_events_processed(event_id, event_type, tenant_id)
             VALUES ($1, $2, $3)
             ON CONFLICT (event_id) DO NOTHING`,
            event.ID, event.Type, invoice.Customer.Metadata["tenant_id"],
        )
        if err != nil {
            return err
        }

        return h.activateTenantSubscription(ctx, tx, invoice.Customer.ID)
    })
}
```

Checklist:
- [ ] Event deduplication table keyed on provider event ID — `ON CONFLICT DO NOTHING` handles retries
- [ ] Deduplication check and state change happen in the same database transaction — no partial updates
- [ ] On processing failure, event is NOT marked processed — provider retries; handler is idempotent
- [ ] Unknown event types silently ignored and logged — not treated as errors

---

## Step 3 — Subscription State Synchronization

Your local subscription state must accurately mirror the billing provider's state.

**Local subscription table:**
```sql
CREATE TABLE subscriptions (
    id                    UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID        NOT NULL REFERENCES tenants(id),
    provider              TEXT        NOT NULL DEFAULT 'stripe',
    provider_subscription_id TEXT     NOT NULL UNIQUE,  -- e.g., sub_abc123
    provider_customer_id  TEXT        NOT NULL,
    status                TEXT        NOT NULL
                          CHECK (status IN ('trialing','active','past_due','unpaid','canceled','paused')),
    plan_id               UUID        NOT NULL REFERENCES plans(id),
    current_period_start  TIMESTAMPTZ NOT NULL,
    current_period_end    TIMESTAMPTZ NOT NULL,
    trial_end             TIMESTAMPTZ,
    cancel_at_period_end  BOOLEAN     NOT NULL DEFAULT FALSE,
    canceled_at           TIMESTAMPTZ,
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Authoritative state is in the billing provider — always sync from webhook events:**
```go
func (h *Processor) syncSubscription(ctx context.Context, tx pgx.Tx, sub *stripe.Subscription) error {
    tenantID := sub.Metadata["tenant_id"]

    _, err := tx.Exec(ctx, `
        INSERT INTO subscriptions (
            tenant_id, provider, provider_subscription_id, provider_customer_id,
            status, plan_id, current_period_start, current_period_end,
            trial_end, cancel_at_period_end, canceled_at, updated_at
        )
        VALUES ($1, 'stripe', $2, $3, $4, $5, $6, $7, $8, $9, $10, NOW())
        ON CONFLICT (provider_subscription_id) DO UPDATE SET
            status                = EXCLUDED.status,
            plan_id               = EXCLUDED.plan_id,
            current_period_start  = EXCLUDED.current_period_start,
            current_period_end    = EXCLUDED.current_period_end,
            trial_end             = EXCLUDED.trial_end,
            cancel_at_period_end  = EXCLUDED.cancel_at_period_end,
            canceled_at           = EXCLUDED.canceled_at,
            updated_at            = NOW()
        WHERE subscriptions.updated_at < EXCLUDED.updated_at`,
        tenantID,
        sub.ID, sub.Customer.ID,
        sub.Status, mapStripePlanToLocal(sub.Items.Data[0].Price.ID),
        time.Unix(sub.CurrentPeriodStart, 0),
        time.Unix(sub.CurrentPeriodEnd, 0),
        nullableTime(sub.TrialEnd),
        sub.CancelAtPeriodEnd,
        nullableTime(sub.CanceledAt),
    )
    return err
}
```

**Guard against out-of-order events:**
- The `WHERE subscriptions.updated_at < EXCLUDED.updated_at` condition prevents an older event from overwriting a newer state
- Use the provider event's `created` timestamp, not your processing timestamp, for ordering comparisons

Checklist:
- [ ] Local subscription table mirrors provider status exactly — no custom statuses that diverge
- [ ] All subscription updates go through the same `syncSubscription` function — no ad-hoc updates
- [ ] Out-of-order protection: do not update if the local record is newer than the incoming event
- [ ] Periodic reconciliation job compares local state to provider state — detects missed webhooks

---

## Step 4 — Failed Payment and Dunning Handling

**Payment failure flow:**
```
invoice.payment_failed webhook received
  → Set subscription status to 'past_due'
  → Start grace period (configurable — e.g., 7 days)
  → Send "payment failed" email to billing contact
  → Billing provider retries automatically (Smart Retries / Dunning)

If payment succeeds during retry:
  invoice.payment_succeeded received
    → Set subscription status to 'active'
    → Send "payment recovered" confirmation email

If payment fails final retry:
  customer.subscription.deleted OR subscription.status = 'unpaid'
    → Set subscription status to 'canceled'
    → Revoke access (or enter lockdown mode with data retention)
    → Send "subscription canceled" email
```

**Grace period enforcement:**
```go
func (s *AccessService) CheckAccess(ctx context.Context, tenantID string) error {
    sub, err := s.subRepo.GetActive(ctx, tenantID)
    if err != nil {
        return err
    }

    switch sub.Status {
    case "active", "trialing":
        return nil // full access
    case "past_due":
        // Allow access during grace period
        if time.Now().Before(sub.CurrentPeriodEnd.Add(s.gracePeriod)) {
            return nil // still within grace
        }
        return ErrSubscriptionPastDue
    case "canceled", "unpaid":
        return ErrSubscriptionCanceled
    default:
        return ErrSubscriptionInactive
    }
}
```

Checklist:
- [ ] Grace period defined and configured — past_due does not immediately revoke access
- [ ] Payment failure triggers email to billing contact — not account owner (may be different)
- [ ] Subscription status immediately reflected after webhook — not on next page load
- [ ] Data retention policy defined for canceled tenants — how long before deletion

---

## Step 5 — Billing Audit Trail

```sql
CREATE TABLE billing_audit_log (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID        REFERENCES tenants(id),
    event_type   TEXT        NOT NULL,  -- 'subscription.activated', 'payment.failed', etc.
    provider     TEXT        NOT NULL DEFAULT 'stripe',
    provider_event_id TEXT,
    amount_cents BIGINT,
    currency     TEXT,
    payload      JSONB       NOT NULL,  -- full provider event payload
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_billing_audit_tenant ON billing_audit_log(tenant_id, created_at DESC);
```

Checklist:
- [ ] Every billing webhook event logged to the audit table before processing
- [ ] Full raw payload stored — enables replay and dispute investigation
- [ ] Audit log is append-only — no updates or deletes
- [ ] Audit log queryable by tenant for support and compliance

---

## Output Report

### Critical
- Webhook signature not verified — any attacker can fake billing events; subscription state manipulated
- Event processing not idempotent — duplicate delivery causes double state changes (e.g., double activation)
- Local subscription state updated before the provider confirms payment — charge and access state diverge

### High
- No out-of-order event protection — delayed webhook overwrites newer state with older data
- Grace period missing — `past_due` immediately locks users out; churn spikes from transient payment failures
- Billing events not logged — no audit trail for dispute resolution or reconciliation

### Medium
- Periodic reconciliation job absent — missed webhooks cause subscription state drift indefinitely
- Payment failure emails sent to account owner, not billing contact — wrong person notified
- Webhook processing not transactional — event marked processed even if the DB update fails

### Low
- Webhook endpoint returns processing errors to provider — exposes internals; provider logs noise
- Stripe webhook secret hardcoded in source — secret rotation requires code deployment
- Plan change prorations not reflected in local state — billing and access control diverge on upgrade/downgrade

### Passed
- Webhook signature verified on every request using provider SDK; secret in vault
- Idempotency enforced via deduplication table; deduplication + state change in same transaction
- All subscription events processed through `syncSubscription`; out-of-order protection applied
- Grace period configured; `past_due` allows continued access while dunning retries
- All billing events appended to audit log before processing; full payload stored

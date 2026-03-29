---
name: billing-event-idempotency
description: 'Duplicate billing event prevention for SaaS Stripe integrations. Use when: billing event idempotency, duplicate billing event, billing deduplication, stripe webhook deduplication, idempotent billing handler, duplicate stripe event, webhook idempotency key, billing event deduplication table, prevent double billing, double payment prevention, idempotent webhook processing, stripe event id, stripe idempotency key, billing event processed table, duplicate subscription event, concurrent webhook billing, at least once billing delivery, webhook replay idempotency, billing event tracking, processed events table, billing event replay.'
argument-hint: 'Describe where duplicate billing events cause you problems (double charge, double state transition, duplicate emails) and your current processing model (sync or async, queue or direct handler).'
---

# Billing Event Idempotency

## When to Use

Invoke this skill when you need to:
- Prevent duplicate processing of the same Stripe webhook event
- Design the event deduplication table and lookup pattern
- Make billing handlers idempotent at each operation level
- Handle concurrent delivery of the same event safely
- Protect against billing state corruption from replayed events
- Audit which events were processed, skipped, or failed

---

## Step 1 — Why Idempotency is Non-Negotiable for Billing

Stripe guarantees **at-least-once** delivery, not exactly-once. The same event may arrive:
- Multiple times on non-2xx responses
- Simultaneously from Stripe's retry and your async retry
- After your server restart while an event was in-flight
- As a replay during incident recovery

Without idempotency, duplicates cause: double charges, double status transitions, double emails, double access grants/revocations, and corrupted usage counters.

---

## Step 2 — Event Deduplication Table

The foundation of billing idempotency:

```sql
CREATE TABLE billing_events_processed (
    -- Stripe's globally unique event ID (e.g., evt_1ABC123...)
    stripe_event_id     TEXT        PRIMARY KEY,
    event_type          TEXT        NOT NULL,
    tenant_id           UUID,                       -- NULL if tenant resolution fails
    status              TEXT        NOT NULL DEFAULT 'processing'
                                    CHECK (status IN ('processing', 'processed', 'failed', 'skipped')),
    idempotency_key     TEXT,                       -- for operations using Stripe idempotency keys
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ,
    error_message       TEXT,
    retry_count         INT         NOT NULL DEFAULT 0,
    payload_hash        TEXT        -- SHA-256 of raw payload for tamper detection
);

CREATE INDEX idx_billing_events_status       ON billing_events_processed(status);
CREATE INDEX idx_billing_events_tenant_id    ON billing_events_processed(tenant_id);
CREATE INDEX idx_billing_events_received_at  ON billing_events_processed(received_at);

-- Optional: cleanup index for expired events
CREATE INDEX idx_billing_events_processed_at ON billing_events_processed(processed_at)
    WHERE processed_at IS NOT NULL;
```

---

## Step 3 — Idempotent Processing Pattern

Use optimistic insert with conflict detection to prevent concurrent duplicate processing:

```go
func (p *IdempotentEventProcessor) Process(ctx context.Context, event stripe.Event) error {
    // Step 1: Attempt to claim the event with INSERT ... ON CONFLICT DO NOTHING
    claimed, err := p.claimEvent(ctx, event)
    if err != nil {
        return fmt.Errorf("claim event: %w", err)
    }

    if !claimed {
        // Another goroutine/instance is already processing or has processed this event
        slog.Info("billing event already claimed, skipping",
            slog.String("event_id", event.ID),
            slog.String("type", event.Type),
        )
        return nil
    }

    // Step 2: Process the event
    processingErr := p.dispatch(ctx, event)

    // Step 3: Record outcome
    if processingErr != nil {
        return p.markFailed(ctx, event.ID, processingErr)
    }

    return p.markProcessed(ctx, event.ID)
}

func (p *IdempotentEventProcessor) claimEvent(ctx context.Context, event stripe.Event) (bool, error) {
    const insertQuery = `
        INSERT INTO billing_events_processed
            (stripe_event_id, event_type, status, received_at, payload_hash)
        VALUES ($1, $2, 'processing', NOW(), $3)
        ON CONFLICT (stripe_event_id) DO NOTHING
    `

    payloadHash := sha256hex(event.Data.Raw)
    result, err := p.db.ExecContext(ctx, insertQuery, event.ID, event.Type, payloadHash)
    if err != nil {
        return false, err
    }

    rows, _ := result.RowsAffected()
    return rows == 1, nil // 0 rows = conflict = already processing or processed
}
```

---

## Step 4 — Handler-Level Idempotency

The deduplication table prevents double dispatch, but each handler must also be idempotent internally:

**Pattern: Check → Skip if already applied → Apply → Record**

```go
func (h *SubscriptionHandler) HandleSubscriptionActivated(ctx context.Context, sub *Subscription, event stripe.Event) error {
    // Idempotent state transition
    if sub.Status == "active" {
        slog.Debug("subscription already active, skipping",
            slog.String("sub_id", sub.ID),
            slog.String("event_id", event.ID),
        )
        return nil // Already in target state — safe to return without error
    }

    return h.stateMachine.Transition(ctx, sub.ID, "active", "invoice.paid")
}

func (h *InvoiceHandler) SyncInvoice(ctx context.Context, inv stripe.Invoice) error {
    // Idempotent upsert — safe to run multiple times
    return h.invoiceRepo.Upsert(ctx, mapStripeInvoice(inv))
    // INSERT ... ON CONFLICT (stripe_invoice_id) DO UPDATE SET ...
}

func (h *EmailHandler) SendTrialConversionEmail(ctx context.Context, tenantID, eventID string) error {
    // Track sent notifications by event ID to prevent double-sending
    sent, err := h.notifRepo.WasNotificationSent(ctx, tenantID, "trial_converted", eventID)
    if err != nil || sent {
        return err
    }

    if err := h.mailer.Send(ctx, TrialConversionEmail{TenantID: tenantID}); err != nil {
        return err
    }

    return h.notifRepo.RecordNotificationSent(ctx, tenantID, "trial_converted", eventID)
}
```

---

## Step 5 — Concurrent Event Handling

When two processes claim the same event simultaneously:

```sql
-- Second INSERT gets rejected by PRIMARY KEY constraint — no additional locking needed
-- The claiming pattern using ON CONFLICT DO NOTHING is safe under concurrency
```

**Handle the `processing` stuck state** (event claimed but process died before completion):

```go
// Recovery job: runs every 5 minutes
func (j *StuckEventRecoveryJob) Run(ctx context.Context) error {
    const query = `
        SELECT stripe_event_id, event_type, retry_count
        FROM billing_events_processed
        WHERE status = 'processing'
          AND received_at < NOW() - INTERVAL '10 minutes'
          AND retry_count < 3
    `

    stuckEvents, err := j.db.QueryContext(ctx, query)
    if err != nil {
        return err
    }
    defer stuckEvents.Close()

    for stuckEvents.Next() {
        var eventID, eventType string
        var retryCount int
        if err := stuckEvents.Scan(&eventID, &eventType, &retryCount); err != nil {
            continue
        }

        // Re-fetch from Stripe and re-process
        event, err := j.stripeClient.RetrieveEvent(ctx, eventID)
        if err != nil {
            slog.Error("failed to retrieve stuck event from stripe",
                slog.String("event_id", eventID),
                slog.String("err", err.Error()),
            )
            continue
        }

        // Reset to pending and dispatch
        j.db.ExecContext(ctx,
            "UPDATE billing_events_processed SET status='processing', retry_count=retry_count+1 WHERE stripe_event_id=$1",
            eventID,
        )
        j.processor.dispatch(ctx, *event)
    }

    return stuckEvents.Err()
}
```

---

## Step 6 — Stripe Idempotency Keys for Outbound Calls

Use idempotency keys when making Stripe API calls that could be retried:

```go
func (s *BillingService) CreateInvoiceItem(ctx context.Context, customerID string, amountCents int64, desc string) error {
    // Deterministic key: prevents duplicate invoice items if this function is retried
    idempotencyKey := fmt.Sprintf("invoice_item_%s_%d_%s",
        customerID, amountCents, slugify(desc))

    _, err := invoiceitem.New(&stripe.InvoiceItemParams{
        Customer:    stripe.String(customerID),
        Amount:      stripe.Int64(amountCents),
        Currency:    stripe.String("usd"),
        Description: stripe.String(desc),
    }, &stripe.RequestParams{
        IdempotencyKey: stripe.String(idempotencyKey),
    })

    return err
}
```

---

## Step 7 — Event Table Cleanup

Prevent unbounded table growth:

```sql
-- Cleanup job: retain last 90 days; delete older processed/skipped events
DELETE FROM billing_events_processed
WHERE status IN ('processed', 'skipped')
  AND processed_at < NOW() - INTERVAL '90 days';

-- Retain failed events for 1 year for audit
DELETE FROM billing_events_processed
WHERE status = 'failed'
  AND processed_at < NOW() - INTERVAL '365 days';
```

---

## Quality Checks

- [ ] `billing_events_processed` table exists with `stripe_event_id` as PRIMARY KEY — concurrent inserts safe
- [ ] Claim step uses `INSERT ... ON CONFLICT DO NOTHING` — no SELECT then INSERT race condition
- [ ] Each handler checks pre-conditions before applying changes (e.g., already in target state → skip)
- [ ] Upserts used for all write operations (invoice sync, entitlement updates) — not INSERT-only
- [ ] Notification sends tracked by event ID — emails never sent twice for the same event
- [ ] Stuck `processing` events older than 10 minutes are retried with exponential backoff
- [ ] Stripe outbound API calls use deterministic idempotency keys on retryable operations
- [ ] Processed events cleaned up after 90 days; failed events retained for 1 year

---
name: stripe-webhook-processing
description: 'Secure and idempotent Stripe webhook handling for SaaS products. Use when: stripe webhook processing, stripe webhook handler, stripe webhook security, stripe signature verification, webhook endpoint stripe, idempotent webhook stripe, stripe event processing, webhook deduplication stripe, stripe webhook retry, webhook event routing, stripe webhook queue, async webhook processing, stripe event type handler, stripe webhook secret, webhook signature check, stripe webhook best practices, stripe webhook failure, webhook replay stripe, stripe webhook endpoint security, webhook out of order stripe, webhook duplicate stripe.'
argument-hint: 'Describe which Stripe webhook events you handle (subscription.updated, invoice.payment_failed, etc.), your current processing approach (sync or async), and any known issues (duplicates, order sensitivity, timeouts).'
---

# Stripe Webhook Processing

## When to Use

Invoke this skill when you need to:
- Build a secure Stripe webhook endpoint with signature verification
- Implement idempotent event processing to handle retries safely
- Route Stripe events to the correct handler functions
- Process webhooks asynchronously to avoid Stripe timeout penalties
- Design a dead-letter queue for failed webhook processing
- Test webhook flows locally and in staging

---

## Step 1 — Endpoint Security

Stripe delivers webhook events signed with your endpoint's signing secret. Always verify before processing:

```go
import (
    "io"
    "net/http"
    "os"

    stripe "github.com/stripe/stripe-go/v76"
    "github.com/stripe/stripe-go/v76/webhook"
)

const maxWebhookBodyBytes = int64(65536) // 64KB — Stripe payloads are well under this

func (h *WebhookHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. Limit body size to prevent memory exhaustion
    r.Body = http.MaxBytesReader(w, r.Body, maxWebhookBodyBytes)
    payload, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "body read error", http.StatusBadRequest)
        return
    }

    // 2. Verify Stripe signature — reject if invalid
    event, err := webhook.ConstructEvent(
        payload,
        r.Header.Get("Stripe-Signature"),
        os.Getenv("STRIPE_WEBHOOK_SECRET"),
    )
    if err != nil {
        // Log without echoing error details — don't leak internals
        slog.Warn("stripe webhook signature invalid", slog.String("err", err.Error()))
        http.Error(w, "invalid signature", http.StatusBadRequest)
        return
    }

    // 3. Acknowledge immediately — Stripe retries on non-2xx after 30s
    w.WriteHeader(http.StatusOK)

    // 4. Process asynchronously — never block the HTTP response
    h.queue.Enqueue(event)
}
```

**Security rules:**
- Signature verification MUST happen before any processing or DB reads
- Return `200 OK` before processing — prevents Stripe timeout retries on slow handlers
- Store `STRIPE_WEBHOOK_SECRET` in a secrets manager (AWS Secrets Manager, Vault, etc.) — never hardcoded
- Use separate secrets per webhook endpoint (Dashboard → Webhooks → Endpoint → Signing Secret)

---

## Step 2 — Idempotency via Event Deduplication

Stripe guarantees at-least-once delivery. Your handler must be safe to call multiple times with the same event:

```sql
-- Event deduplication table
CREATE TABLE stripe_events_processed (
    stripe_event_id  TEXT        PRIMARY KEY,   -- e.g., evt_1ABC...
    event_type       TEXT        NOT NULL,
    tenant_id        UUID,                       -- resolved after processing
    received_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at     TIMESTAMPTZ,
    status           TEXT        NOT NULL DEFAULT 'pending'
                                 CHECK (status IN ('pending', 'processed', 'failed', 'skipped')),
    error_message    TEXT,
    retry_count      INT         NOT NULL DEFAULT 0
);

CREATE INDEX idx_stripe_events_received_at ON stripe_events_processed(received_at);
```

```go
func (p *EventProcessor) Process(ctx context.Context, event stripe.Event) error {
    // Check: already successfully processed?
    existing, err := p.repo.GetEvent(ctx, event.ID)
    if err == nil && existing.Status == "processed" {
        slog.Info("duplicate stripe event ignored", slog.String("event_id", event.ID))
        return nil
    }

    // Mark as pending (insert or update)
    if err := p.repo.UpsertEvent(ctx, event.ID, event.Type, "pending"); err != nil {
        return err
    }

    // Dispatch to handler
    handlerErr := p.dispatch(ctx, event)

    // Record outcome
    if handlerErr != nil {
        p.repo.MarkFailed(ctx, event.ID, handlerErr.Error())
        return handlerErr
    }

    return p.repo.MarkProcessed(ctx, event.ID)
}
```

---

## Step 3 — Event Type Routing

Route events to typed handlers to keep processing logic isolated and testable:

```go
func (p *EventProcessor) dispatch(ctx context.Context, event stripe.Event) error {
    switch event.Type {

    // Subscription lifecycle
    case "customer.subscription.created":
        return p.handleSubscriptionCreated(ctx, event)
    case "customer.subscription.updated":
        return p.handleSubscriptionUpdated(ctx, event)
    case "customer.subscription.deleted":
        return p.handleSubscriptionDeleted(ctx, event)
    case "customer.subscription.trial_will_end":
        return p.handleTrialWillEnd(ctx, event)

    // Invoice & payments
    case "invoice.paid":
        return p.handleInvoicePaid(ctx, event)
    case "invoice.payment_failed":
        return p.handlePaymentFailed(ctx, event)
    case "invoice.finalized":
        return p.handleInvoiceFinalized(ctx, event)

    // Customer
    case "customer.updated":
        return p.handleCustomerUpdated(ctx, event)

    // Payment method
    case "payment_method.attached":
        return p.handlePaymentMethodAttached(ctx, event)

    default:
        slog.Debug("unhandled stripe event type", slog.String("type", event.Type))
        return nil // Not an error — mark as skipped
    }
}
```

**Object deserialization pattern:**
```go
func (p *EventProcessor) handleSubscriptionUpdated(ctx context.Context, event stripe.Event) error {
    var sub stripe.Subscription
    if err := json.Unmarshal(event.Data.Raw, &sub); err != nil {
        return fmt.Errorf("unmarshal subscription event: %w", err)
    }

    // Retrieve tenant from DB by stripe_customer_id
    tenant, err := p.tenantRepo.GetByStripeCustomerID(ctx, sub.Customer.ID)
    if err != nil {
        return fmt.Errorf("resolve tenant for customer %s: %w", sub.Customer.ID, err)
    }

    return p.subscriptionService.SyncFromStripe(ctx, tenant.ID, &sub)
}
```

---

## Step 4 — Async Queue Processing

Process events from a queue, not inline in the HTTP handler:

```go
// Queue interface — backed by Redis, SQS, or in-process channel for tests
type WebhookQueue interface {
    Enqueue(event stripe.Event) error
    StartWorkers(ctx context.Context, concurrency int) error
}

// Worker loop
func (q *RedisWebhookQueue) StartWorkers(ctx context.Context, concurrency int) error {
    for i := 0; i < concurrency; i++ {
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return
                default:
                    event, err := q.dequeue(ctx)
                    if err != nil {
                        time.Sleep(1 * time.Second)
                        continue
                    }
                    if err := q.processor.Process(ctx, event); err != nil {
                        slog.Error("webhook processing failed",
                            slog.String("event_id", event.ID),
                            slog.String("type", event.Type),
                            slog.String("err", err.Error()),
                        )
                        q.requeueWithBackoff(event)
                    }
                }
            }
        }()
    }
    return nil
}
```

---

## Step 5 — Local Development & Testing

Test webhooks without a public endpoint:

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward to local server (creates a local webhook secret)
stripe listen --forward-to localhost:8080/webhooks/stripe

# Trigger specific events manually
stripe trigger customer.subscription.updated
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.trial_will_end
```

**Unit test pattern for handlers:**
```go
func TestHandlePaymentFailed(t *testing.T) {
    rawEvent := loadTestFixture(t, "testdata/invoice_payment_failed.json")
    event, err := webhook.ConstructEventWithOptions(rawEvent, "", "",
        webhook.ConstructEventOptions{IgnoreAPIVersionMismatch: true})
    require.NoError(t, err)

    processor := newTestProcessor(t)
    err = processor.dispatch(context.Background(), event)
    require.NoError(t, err)

    // Assert subscription moved to past_due
    sub := processor.subRepo.Get(t, testTenantID)
    assert.Equal(t, "past_due", sub.Status)
}
```

---

## Quality Checks

- [ ] Stripe signature verified on every request before any processing or DB reads
- [ ] `200 OK` returned before processing begins — handler never blocks the HTTP response
- [ ] Event deduplication table exists; handler is safe to call multiple times for the same event ID
- [ ] Each event type has its own handler function — no monolithic dispatch body
- [ ] `STRIPE_WEBHOOK_SECRET` sourced from environment / secrets manager — never hardcoded
- [ ] Failed events are retried with backoff; persistent failures land in a dead-letter queue
- [ ] Webhook endpoint excluded from CSRF middleware (raw body must reach signature verifier)
- [ ] Local development uses `stripe listen` for realistic event testing

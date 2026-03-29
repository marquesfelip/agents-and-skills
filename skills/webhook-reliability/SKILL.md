---
name: webhook-reliability
description: 'Reliable webhook ingestion and retry logic for SaaS systems. Use when: webhook reliability, webhook retry, webhook delivery, webhook ingestion, webhook queue, reliable webhook, webhook deduplication, webhook idempotency, webhook failure, webhook at-least-once, webhook backoff, webhook dead letter, inbound webhook reliability, outbound webhook delivery, webhook job, webhook processing, webhook consumer, webhook acknowledge, webhook ack, accepted immediately process async.'
argument-hint: 'Describe whether the webhook is inbound (from Stripe, GitHub, etc.) or outbound (you send to customers), the volume and latency sensitivity, and what failure modes to handle (provider retry storms, consumer crashes, slow processing).'
---

# Webhook Reliability

## When to Use

Invoke this skill when you need to:
- Ingest inbound webhooks from providers (Stripe, GitHub, Slack) without data loss when processing is slow
- Deliver outbound webhooks to customers with retry, backoff, and dead-letter handling
- Decouple webhook receipt (HTTP accept) from webhook processing (business logic)
- Implement retry with exponential backoff for failed deliveries without hammering endpoints
- Track delivery state: pending, delivered, failed, dead-lettered — per webhook per endpoint
- Handle the case where your consumer crashes mid-processing

---

## Inbound vs Outbound Reliability Patterns

| Direction | Key risk | Fix |
|---|---|---|
| **Inbound** (provider → you) | Long processing causes HTTP timeout → provider retries → duplicate | Accept immediately (202), process async, deduplicate |
| **Inbound** | Provider sends multiple events for one logical change | Dedup by provider's event/delivery ID |
| **Outbound** (you → customer) | Customer endpoint is slow/down → you give up → customer misses event | Retry with exponential backoff, DLQ after max attempts |
| **Outbound** | You resend → customer processes twice | Stable `X-Webhook-ID` header + customer-side idempotency |

---

## Step 1 — Inbound Webhook: Accept Immediately, Process Async

```go
// Handler: HTTP endpoint that accepts the raw payload and acknowledges
func (h *WebhookHandler) Receive(w http.ResponseWriter, r *http.Request) {
    // 1. Validate signature FIRST (security — see webhook-authentication-security)
    body, err := io.ReadAll(io.LimitReader(r.Body, 1<<20)) // 1 MB max
    if err != nil {
        http.Error(w, "read error", http.StatusBadRequest)
        return
    }
    if !h.sig.Verify(r.Header, body) {
        http.Error(w, "invalid signature", http.StatusUnauthorized)
        return
    }

    // 2. Extract deduplication ID from provider header
    deliveryID := r.Header.Get("X-Stripe-Event") // or "X-GitHub-Delivery", etc.
    if deliveryID == "" {
        deliveryID = uuid.New().String() // fallback if provider doesn't provide
    }

    // 3. Store raw payload in DB immediately — do not process inline
    if err := h.store.EnqueueWebhook(r.Context(), WebhookIngestion{
        DeliveryID: deliveryID,
        Source:     "stripe",
        EventType:  r.Header.Get("Stripe-Signature"), // or parsed from body
        RawPayload: body,
    }); err != nil {
        // On DB write failure: return 500 so provider retries
        http.Error(w, "enqueue failed", http.StatusInternalServerError)
        return
    }

    // 4. Acknowledge immediately — do NOT process inline
    w.WriteHeader(http.StatusAccepted) // 202
}
```

---

## Step 2 — Inbound Webhook Storage Schema

```sql
CREATE TYPE webhook_status AS ENUM ('pending', 'processing', 'processed', 'failed', 'dead_lettered');

CREATE TABLE inbound_webhooks (
    id           BIGSERIAL PRIMARY KEY,
    delivery_id  TEXT        NOT NULL UNIQUE,   -- provider's delivery ID (dedup key)
    source       TEXT        NOT NULL,          -- 'stripe' | 'github' | 'linear'
    event_type   TEXT        NOT NULL,
    raw_payload  BYTEA       NOT NULL,          -- store raw bytes — parse in worker
    status       webhook_status NOT NULL DEFAULT 'pending',
    attempts     INT         NOT NULL DEFAULT 0,
    last_error   TEXT,
    received_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ
);

CREATE INDEX idx_inbound_webhooks_pending ON inbound_webhooks(next_retry_at)
    WHERE status IN ('pending', 'failed');
CREATE UNIQUE INDEX idx_inbound_webhooks_delivery ON inbound_webhooks(delivery_id);
```

---

## Step 3 — Inbound Webhook Worker

```go
const (
    MaxWebhookAttempts = 5
    BaseBackoff        = 30 * time.Second
)

func (w *WebhookWorker) ProcessBatch(ctx context.Context) error {
    jobs, err := w.store.ClaimPending(ctx, 20) // SELECT FOR UPDATE SKIP LOCKED
    if err != nil {
        return err
    }
    for _, job := range jobs {
        w.processOne(ctx, job)
    }
    return nil
}

func (w *WebhookWorker) processOne(ctx context.Context, wh InboundWebhook) {
    err := w.dispatch(ctx, wh)

    if err == nil {
        w.store.MarkProcessed(ctx, wh.ID)
        return
    }

    slog.Error("webhook processing failed", "id", wh.ID, "attempt", wh.Attempts, "err", err)

    nextAttempt := wh.Attempts + 1
    if nextAttempt >= MaxWebhookAttempts {
        w.store.MarkDeadLettered(ctx, wh.ID, err.Error())
        w.alerts.NotifyDeadLetter(ctx, wh)
        return
    }

    // Exponential backoff: 30s, 60s, 120s, 240s, 480s
    delay := BaseBackoff * time.Duration(1<<nextAttempt)
    w.store.MarkFailed(ctx, wh.ID, err.Error(), nextAttempt, time.Now().Add(delay))
}

func (w *WebhookWorker) dispatch(ctx context.Context, wh InboundWebhook) error {
    switch wh.Source {
    case "stripe":
        return w.stripeHandler.Handle(ctx, wh.RawPayload)
    case "github":
        return w.githubHandler.Handle(ctx, wh.RawPayload)
    default:
        // Unknown source: dead-letter immediately — no point retrying
        return fmt.Errorf("%w: unknown source %q", ErrUnrecoverable, wh.Source)
    }
}
```

---

## Step 4 — Outbound Webhook Delivery

```sql
CREATE TABLE outbound_webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id),
    tenant_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    status          webhook_status NOT NULL DEFAULT 'pending',
    attempts        INT NOT NULL DEFAULT 0,
    last_status_code INT,
    last_error      TEXT,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    delivered_at    TIMESTAMPTZ
);

CREATE INDEX idx_outbound_deliveries_pending ON outbound_webhook_deliveries(next_retry_at)
    WHERE status IN ('pending', 'failed');
```

```go
func (d *WebhookDeliverer) Deliver(ctx context.Context, delivery OutboundDelivery) error {
    endpoint, err := d.endpointRepo.GetByID(ctx, delivery.EndpointID)
    if err != nil {
        return err
    }

    // Build payload with stable delivery ID for customer-side idempotency
    payload, err := json.Marshal(WebhookPayload{
        ID:        delivery.ID.String(), // stable across retries
        Type:      delivery.EventType,
        CreatedAt: delivery.CreatedAt,
        Data:      delivery.Payload,
    })
    if err != nil {
        return err
    }

    // Sign and send (see webhook-authentication-security for signing)
    req, _ := http.NewRequestWithContext(ctx, http.MethodPost, endpoint.URL, bytes.NewReader(payload))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Webhook-ID", delivery.ID.String()) // stable — same across retries
    req.Header.Set("X-Webhook-Signature", d.signer.Sign(payload, endpoint.Secret))
    req.Header.Set("X-Webhook-Timestamp", strconv.FormatInt(time.Now().Unix(), 10))

    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return &DeliveryError{Retriable: true, Err: err}
    }
    defer resp.Body.Close()

    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        return nil // success
    }
    if resp.StatusCode >= 400 && resp.StatusCode < 500 {
        // 4xx: customer configuration error — do not retry (except 429)
        if resp.StatusCode == 429 {
            return &DeliveryError{Retriable: true, StatusCode: resp.StatusCode}
        }
        return &DeliveryError{Retriable: false, StatusCode: resp.StatusCode}
    }
    // 5xx: server error — retry
    return &DeliveryError{Retriable: true, StatusCode: resp.StatusCode}
}
```

---

## Step 5 — Retry Schedule

```go
// Delivery retry schedule (outbound)
var retryDelays = []time.Duration{
    5 * time.Minute,   // attempt 1
    30 * time.Minute,  // attempt 2
    2 * time.Hour,     // attempt 3
    8 * time.Hour,     // attempt 4
    24 * time.Hour,    // attempt 5 — final
}

func nextRetryAt(attempt int) *time.Time {
    if attempt >= len(retryDelays) {
        return nil // no more retries — dead letter
    }
    t := time.Now().UTC().Add(retryDelays[attempt])
    return &t
}
```

---

## Webhook Reliability Checklist

| Concern | Inbound | Outbound |
|---|---|---|
| Deduplication | `UNIQUE (delivery_id)` | Stable `X-Webhook-ID` header |
| Signature validation | On receipt, before enqueue | On send, HMAC-SHA-256 |
| Async processing | 202 accept, process in worker | Delivery queue + worker |
| Retry with backoff | Exponential, 5 attempts max | Per-attempt delay table |
| Unrecoverable errors | Dead-letter immediately (4xx, bad parse) | Dead-letter on non-retriable 4xx |
| Observability | `status`, `attempts`, `last_error` columns | Same + `last_status_code` |
| Alert on DLQ | Yes — notify on-call | Yes — notify tenant |

---

## Quality Checks

- [ ] HTTP handler returns 202 within 500ms — all processing is asynchronous
- [ ] `delivery_id` UNIQUE constraint prevents double processing on provider retry storms
- [ ] Signature verified BEFORE storing payload — invalid payloads never enqueued
- [ ] Worker uses `SELECT FOR UPDATE SKIP LOCKED` — no two workers process the same webhook
- [ ] `ErrUnrecoverable` errors skip retry — go straight to DLQ (bad parse, unknown source, 4xx)
- [ ] Outbound: `X-Webhook-ID` is stable (same UUID) across all retry attempts — customer can dedup
- [ ] Outbound: 10-second HTTP timeout enforced on every delivery attempt
- [ ] Dead-lettered webhooks trigger an operational alert — not silently dropped

## After Completion

- Use **`webhook-authentication-security`** for HMAC-SHA-256 signing and timestamp replay prevention
- Use **`dead-letter-handling`** for managing dead-lettered deliveries and reprocessing
- Use **`job-idempotency`** for idempotent processing inside the webhook worker
- Use **`event-driven-saas-patterns`** for modeling the events that trigger outbound webhooks

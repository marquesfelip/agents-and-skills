---
name: dead-letter-handling
description: 'Dead-letter queue management and poison message handling for SaaS systems. Use when: dead letter queue, DLQ, dead letter handling, poison message, unprocessable event, DLQ requeue, DLQ inspection, DLQ alert, DLQ monitoring, DLQ retry, dead letter consumer, failed event, undeliverable event, dead letter schema, DLQ admin, DLQ depth alert, DLQ size alert, manual retry DLQ, discard dead letter, DLQ audit, dead letter strategy, SQS DLQ, Kafka DLQ, message dead letter, event dead letter, job dead letter, webhook dead letter, DLQ policy.'
argument-hint: 'Describe the message broker or job system (PostgreSQL outbox, SQS, Kafka, RabbitMQ), what events or jobs land in the DLQ, the desired DLQ inspection and requeue workflow, and any alert thresholds (number of events, DLQ age).'
---

# Dead-Letter Handling

## When to Use

Invoke this skill when you need to:
- Capture events, jobs, or webhook deliveries that exceed retry limits without silently discarding them
- Route poison messages (unparseable, schema-invalid) directly to DLQ without wasting retries
- Inspect DLQ contents via an admin interface and selectively requeue or discard
- Alert when DLQ depth exceeds a threshold (signals a systemic processing problem)
- Implement a discard policy for dead-lettered items that cannot be fixed
- Log all DLQ entries with full context for root-cause analysis

---

## Dead-Letter Routing Decision

```
Event processing fails
        │
        ▼
Is the error permanent? (non-retriable)
  • Unparseable payload (malformed JSON)
  • Unknown event type
  • Schema validation error
  • Missing required fields
        │YES                      │NO
        ▼                         ▼
Dead-letter immediately     Increment attempt counter
(skip retry queue)               │
                                 ▼
                         attempts >= max?
                           │YES        │NO
                           ▼           ▼
                     Dead-letter    Schedule retry
                     after max      with backoff
```

---

## Step 1 — Dead-Letter Table Schema

```sql
CREATE TYPE dlq_source AS ENUM ('event', 'job', 'webhook_inbound', 'webhook_outbound');

CREATE TABLE dead_letter_items (
    id             UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    source         dlq_source  NOT NULL,
    source_id      TEXT        NOT NULL,   -- original ID (event_id, job_id, delivery_id)
    topic          TEXT,                   -- original queue/topic/event type
    tenant_id      UUID,                   -- for tenant-scoped DLQ filtering
    raw_payload    BYTEA       NOT NULL,   -- original payload for reprocessing
    failure_reason TEXT        NOT NULL,   -- last error message
    failure_type   TEXT        NOT NULL,   -- 'permanent' | 'exhausted_retries'
    attempt_count  INT         NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_at    TIMESTAMPTZ,            -- set when an operator reviewed this item
    resolved_at    TIMESTAMPTZ,            -- set when requeued or discarded
    resolution     TEXT                   -- 'requeued' | 'discarded' | 'auto_fixed'
);

CREATE INDEX idx_dlq_unresolved ON dead_letter_items(created_at)
    WHERE resolved_at IS NULL;
CREATE INDEX idx_dlq_tenant ON dead_letter_items(tenant_id, created_at)
    WHERE resolved_at IS NULL;
CREATE INDEX idx_dlq_topic ON dead_letter_items(topic, created_at)
    WHERE resolved_at IS NULL;
```

---

## Step 2 — Dead-Letter Writer

```go
type DLQEntry struct {
    Source        string // "event" | "job" | "webhook_inbound" | "webhook_outbound"
    SourceID      string
    Topic         string
    TenantID      string
    RawPayload    []byte
    FailureReason string
    FailureType   string // "permanent" | "exhausted_retries"
    AttemptCount  int
}

type DLQRepository interface {
    Write(ctx context.Context, entry DLQEntry) error
    ListUnresolved(ctx context.Context, filter DLQFilter) ([]DLQItem, error)
    MarkResolved(ctx context.Context, id uuid.UUID, resolution string) error
    CountUnresolved(ctx context.Context) (int64, error)
}

// ErrUnrecoverable marks an error as non-retriable → immediate DLQ routing
var ErrUnrecoverable = errors.New("unrecoverable")

func IsUnrecoverable(err error) bool {
    return errors.Is(err, ErrUnrecoverable)
}

// Sentinel errors for common permanent failures
var (
    ErrInvalidPayload  = fmt.Errorf("%w: invalid payload", ErrUnrecoverable)
    ErrUnknownType     = fmt.Errorf("%w: unknown event type", ErrUnrecoverable)
    ErrSchemaViolation = fmt.Errorf("%w: schema violation", ErrUnrecoverable)
)
```

---

## Step 3 — Dead-Letter Integration in Worker

```go
func (w *EventWorker) processOne(ctx context.Context, event InboundEvent) {
    err := w.handler.Handle(ctx, event)
    if err == nil {
        w.store.MarkProcessed(ctx, event.ID)
        return
    }

    // Permanent error: route to DLQ immediately — no retry
    if IsUnrecoverable(err) {
        slog.Error("unrecoverable event — dead-lettering", "id", event.ID, "err", err)
        w.deadLetterNow(ctx, event, err, "permanent")
        return
    }

    // Transient error: increment retry counter
    event.Attempts++
    if event.Attempts >= MaxAttempts {
        slog.Error("max retries exceeded — dead-lettering", "id", event.ID, "attempts", event.Attempts)
        w.deadLetterNow(ctx, event, err, "exhausted_retries")
        return
    }

    // Schedule retry with exponential backoff
    delay := BaseBackoff * time.Duration(1<<event.Attempts)
    w.store.ScheduleRetry(ctx, event.ID, event.Attempts, time.Now().Add(delay), err.Error())
}

func (w *EventWorker) deadLetterNow(ctx context.Context, event InboundEvent, err error, failureType string) {
    entry := DLQEntry{
        Source:        "event",
        SourceID:      event.ID,
        Topic:         event.Topic,
        TenantID:      event.TenantID,
        RawPayload:    event.RawPayload,
        FailureReason: err.Error(),
        FailureType:   failureType,
        AttemptCount:  event.Attempts,
    }
    if writeErr := w.dlq.Write(ctx, entry); writeErr != nil {
        slog.Error("failed to write to DLQ", "id", event.ID, "err", writeErr)
    }
    w.store.MarkDeadLettered(ctx, event.ID)
}
```

---

## Step 4 — DLQ Alerting

```go
// Background goroutine: poll DLQ depth and alert when threshold exceeded
func (m *DLQMonitor) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            count, err := m.dlq.CountUnresolved(ctx)
            if err != nil {
                slog.Error("DLQ count failed", "err", err)
                continue
            }
            if count >= m.AlertThreshold {
                slog.Warn("DLQ depth alert", "count", count, "threshold", m.AlertThreshold)
                m.alerts.FireDLQDepthAlert(ctx, DLQAlert{
                    Count:     count,
                    Threshold: m.AlertThreshold,
                    CheckedAt: time.Now(),
                })
            }
        }
    }
}
```

---

## Step 5 — Admin: Inspect and Requeue

```go
// Admin GET /admin/dlq — list unresolved dead-letter items
func (h *DLQAdminHandler) List(w http.ResponseWriter, r *http.Request) {
    filter := DLQFilter{
        Source:   r.URL.Query().Get("source"),
        TenantID: r.URL.Query().Get("tenant_id"),
        Topic:    r.URL.Query().Get("topic"),
        Limit:    50,
    }
    items, err := h.dlq.ListUnresolved(r.Context(), filter)
    if err != nil {
        http.Error(w, "internal error", 500)
        return
    }
    writeJSON(w, items)
}

// Admin POST /admin/dlq/{id}/requeue — replay one item
func (h *DLQAdminHandler) Requeue(w http.ResponseWriter, r *http.Request) {
    id := mustParseUUID(r.PathValue("id"))

    item, err := h.dlq.GetByID(r.Context(), id)
    if err != nil {
        http.Error(w, "not found", 404)
        return
    }

    // Re-enqueue raw payload to original topic
    if err := h.broker.Publish(r.Context(), item.Topic, item.RawPayload); err != nil {
        http.Error(w, "requeue failed", 500)
        return
    }

    // Mark resolved before requeue completes — idempotent requeue required
    h.dlq.MarkResolved(r.Context(), id, "requeued")
    slog.Info("DLQ item requeued", "id", id, "topic", item.Topic, "operator", operatorFromCtx(r.Context()))
    w.WriteHeader(http.StatusNoContent)
}

// Admin POST /admin/dlq/{id}/discard — permanently discard with reason
func (h *DLQAdminHandler) Discard(w http.ResponseWriter, r *http.Request) {
    id := mustParseUUID(r.PathValue("id"))
    h.dlq.MarkResolved(r.Context(), id, "discarded")
    slog.Info("DLQ item discarded", "id", id, "operator", operatorFromCtx(r.Context()))
    w.WriteHeader(http.StatusNoContent)
}
```

---

## DLQ Failure Type Reference

| Failure type | Examples | Retry? | Action |
|---|---|---|---|
| `permanent` | Bad JSON, unknown event type, schema error | No | DLQ immediately |
| `exhausted_retries` | Flapping downstream, intermittent DB error | No (retries exhausted) | DLQ after max attempts |
| `dependency_unavailable` | Downstream service down | Yes | Retry with backoff |
| `business_rule_violation` | Constraint violation in business logic | Depends | Inspect: may need code fix |

---

## Quality Checks

- [ ] `ErrUnrecoverable` wrapping used for all permanent error categories — no retry wasted
- [ ] DLQ `Write` called inside the worker's error handler — item never silently dropped
- [ ] `source_id` column is unique enough to avoid DLQ duplicates on worker crash/retry
- [ ] DLQ monitor alerts when `CountUnresolved` >= threshold (e.g., 10 or 50)
- [ ] Admin requeue logs operator identity (audit trail) — not anonymous
- [ ] Requeue is safe to repeat: consumer is idempotent so double-requeue does no harm
- [ ] `resolved_at` set before marking in original queue — prevents losing the item on crash

## After Completion

- Use **`job-idempotency`** so requeued DLQ items are processed exactly once
- Use **`event-replay-strategy`** when bulk-replaying many DLQ items from a time window
- Use **`audit-trail-design`** to record DLQ operator actions (requeue, discard) for compliance
- Use **`alerting-strategy`** to configure DLQ depth alerts and escalation thresholds

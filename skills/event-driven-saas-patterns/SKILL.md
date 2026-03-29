---
name: event-driven-saas-patterns
description: 'SaaS event-driven architecture practices — event design, publishing, subscription, and delivery guarantees. Use when: event-driven architecture, EDA, event-based communication, pub/sub, event bus, domain events, integration events, reliable event publishing, outbox pattern, event schema design, event versioning, CloudEvents, event decoupling, event broker, Kafka, EventBridge, event fan-out, SaaS event architecture, event contract, event sourcing, async event, event store, event topic design, event consumer, event producer.'
argument-hint: 'Describe the event(s) you need to publish or consume (e.g., TenantActivated, PaymentFailed, UserCreated), the delivery guarantee required (at-least-once, exactly-once), and your infrastructure (Kafka, SQS, Redis Streams, PostgreSQL LISTEN/NOTIFY, in-process).'
---

# Event-Driven SaaS Patterns

## When to Use

Invoke this skill when you need to:
- Design the event taxonomy for a new domain area (naming, schema, versioning)
- Choose between domain events (in-process) and integration events (cross-service)
- Implement reliable event publishing using the Outbox Pattern (no dual-write risk)
- Design consumers that are idempotent, isolated, and independently deployable
- Version events safely so producers and consumers can evolve independently
- Choose the right broker (in-process, PostgreSQL, Kafka, SQS, Redis Streams) for your scale

---

## Event Type Taxonomy

| Type | Scope | Coupling | Transport |
|---|---|---|---|
| **Domain event** | Within a bounded context | Tight (same codebase) | In-process / in-memory |
| **Integration event** | Cross-service / cross-context | Loose (async) | Message broker (Kafka, SQS, etc.) |
| **Audit event** | Immutable history | None (append-only) | Event store / audit log table |
| **Webhook event** | External customer systems | Very loose | HTTP POST to customer URL |

---

## Step 1 — Event Naming and Schema Design

```go
// Convention: {Aggregate}.{PastTense} — noun.verb
// Examples: tenant.activated, payment.failed, user.invited, subscription.canceled

// Use CloudEvents envelope for cross-service events
type Event struct {
    // CloudEvents required fields
    ID          string    `json:"id"`           // UUID — used for deduplication
    Source      string    `json:"source"`       // "//myapp/tenants" — identifies producer
    SpecVersion string    `json:"specversion"`  // "1.0"
    Type        string    `json:"type"`         // "com.myapp.tenant.activated"
    Time        time.Time `json:"time"`         // RFC3339 — when event occurred
    // Custom fields
    TenantID    uuid.UUID `json:"tenantid"`     // always include for multi-tenant routing
    DataVersion string    `json:"dataversion"`  // "1" — for schema evolution
    Data        any       `json:"data"`         // event-specific payload
}

// Domain event payload — keep payload minimal; consumers fetch details if needed
type TenantActivatedPayload struct {
    TenantID    uuid.UUID `json:"tenant_id"`
    ActivatedAt time.Time `json:"activated_at"`
    PlanID      string    `json:"plan_id"`
    // DO NOT include: internal IDs, PII beyond what consumers need
}
```

**Naming rules:**
- Use past tense: `activated`, `failed`, `created` — not `activate`, `fail`, `create`
- Qualify with aggregate: `tenant.activated`, not just `activated`
- Use reverse-DNS for type: `com.myapp.{aggregate}.{event}`
- One event = one thing happened — do not bundle multiple state changes

---

## Step 2 — Outbox Pattern (Reliable Publishing)

The outbox pattern guarantees events are published exactly once when a DB transaction commits — no dual-write risk.

```sql
CREATE TABLE outbox_events (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topic        TEXT        NOT NULL,             -- Kafka topic or SQS queue
    event_type   TEXT        NOT NULL,
    tenant_id    UUID,
    aggregate_id UUID,
    payload      JSONB       NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ,                      -- NULL = pending
    attempts     INT         NOT NULL DEFAULT 0,
    last_error   TEXT
);

CREATE INDEX idx_outbox_pending ON outbox_events(created_at)
    WHERE published_at IS NULL;
```

```go
// Publish inside a DB transaction — atomically with the state change
func (s *tenantService) Activate(ctx context.Context, tenantID uuid.UUID) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // 1. Apply domain change
        if err := s.repo.MarkActivated(ctx, tx, tenantID); err != nil {
            return err
        }

        // 2. Write to outbox IN THE SAME TRANSACTION — no dual write
        return s.outbox.Enqueue(ctx, tx, OutboxEvent{
            Topic:       "tenant-events",
            EventType:   "com.myapp.tenant.activated",
            TenantID:    &tenantID,
            AggregateID: &tenantID,
            Payload: TenantActivatedPayload{
                TenantID:    tenantID,
                ActivatedAt: time.Now().UTC(),
            },
        })
    })
}

// Outbox relay: runs every second, publishes pending events
func (r *OutboxRelay) Run(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-time.After(1 * time.Second):
            if err := r.publishBatch(ctx); err != nil {
                slog.Error("outbox relay error", "err", err)
            }
        }
    }
}

func (r *OutboxRelay) publishBatch(ctx context.Context) error {
    events, err := r.repo.ListPending(ctx, 100) // batch of 100
    if err != nil {
        return err
    }
    for _, e := range events {
        if err := r.broker.Publish(ctx, e.Topic, e); err != nil {
            r.repo.RecordFailure(ctx, e.ID, err.Error())
            continue
        }
        r.repo.MarkPublished(ctx, e.ID)
    }
    return nil
}
```

---

## Step 3 — Consumer Design

```go
type EventConsumer interface {
    Handle(ctx context.Context, event Event) error
}

// Idempotent consumer — safe to call multiple times with the same event
type TenantActivatedConsumer struct {
    processed ProcessedEventsRepo // deduplication store
    mailer    Mailer
}

func (c *TenantActivatedConsumer) Handle(ctx context.Context, event Event) error {
    // Deduplication: check if already processed by event ID
    alreadyProcessed, err := c.processed.Has(ctx, event.ID)
    if err != nil {
        return fmt.Errorf("dedup check: %w", err)
    }
    if alreadyProcessed {
        return nil // already processed — idempotent
    }

    var payload TenantActivatedPayload
    if err := json.Unmarshal(event.Data, &payload); err != nil {
        return fmt.Errorf("unmarshal: %w", err) // DO NOT retry malformed events → DLQ
    }

    // Business logic
    if err := c.mailer.SendActivationEmail(ctx, payload.TenantID); err != nil {
        return err // retriable error — broker will retry
    }

    // Mark processed AFTER successful handling
    return c.processed.Insert(ctx, event.ID)
}
```

---

## Step 4 — Event Versioning

```go
// Version in the CloudEvents type field: "com.myapp.tenant.activated.v2"
// Or in a dedicated dataversion field (preferred — keeps type stable)

// V1 payload
type TenantActivatedV1 struct {
    TenantID string `json:"tenant_id"` // was string in V1
}

// V2 payload — UUID type added; new optional field
type TenantActivatedV2 struct {
    TenantID    uuid.UUID `json:"tenant_id"`
    ActivatedAt time.Time `json:"activated_at"` // new field in V2
    PlanID      string    `json:"plan_id,omitempty"` // optional — safe to add
}

// Consumer: handle both versions during migration
func (c *consumer) Handle(ctx context.Context, event Event) error {
    switch event.DataVersion {
    case "1":
        var p TenantActivatedV1
        if err := json.Unmarshal(event.Data, &p); err != nil {
            return err
        }
        return c.handleV1(ctx, p)
    case "2", "": // "" = no version field = treat as latest
        var p TenantActivatedV2
        if err := json.Unmarshal(event.Data, &p); err != nil {
            return err
        }
        return c.handleV2(ctx, p)
    default:
        return fmt.Errorf("%w: unsupported version %q", ErrUnknownEventVersion, event.DataVersion)
    }
}
```

**Safe schema changes (backward-compatible):**
- Add optional fields with `omitempty` — old consumers ignore them
- Never remove or rename fields while consumers are still reading them
- Never change field types — use a new field name instead

---

## Step 5 — Broker Selection Guide

| Scale | Approach | Tradeoffs |
|---|---|---|
| Single service, low volume | In-process Go channel / `sync` | Simple; no infra; lost on crash without outbox |
| Small SaaS, PostgreSQL already used | PostgreSQL outbox + relay | No new infra; transactional; polling overhead |
| Multiple services, moderate volume | AWS SQS + SNS fan-out | Managed; at-least-once; no ordering guarantee |
| High throughput, ordering required | Apache Kafka / Confluent | Partitioned ordering; complex ops; high throughput |
| Serverless / AWS-native | EventBridge | Low ops; schema registry; limited throughput |
| Redis already in stack | Redis Streams | Good middle ground; at-least-once; consumer groups |

---

## Quality Checks

- [ ] All events have a stable, unique `ID` (UUID) — used for consumer deduplication
- [ ] Events are published inside the same DB transaction as the state change (outbox pattern)
- [ ] Consumer `Handle` is idempotent — processing the same event twice produces the same outcome
- [ ] Malformed/unparseable events are routed to DLQ — not retried indefinitely
- [ ] Event schema uses past-tense naming: `tenant.activated`, not `tenant.activate`
- [ ] New fields are added as optional (`omitempty`) — old consumers do not break
- [ ] Outbox relay runs with concurrency=1 (or uses distributed lock) — avoids duplicate publish
- [ ] Consumer failures are retried by the broker — consumer does not implement its own retry loop

## After Completion

- Use **`job-idempotency`** for idempotent processing patterns across all background jobs
- Use **`webhook-reliability`** for delivering integration events to external customer endpoints
- Use **`dead-letter-handling`** for managing unprocessable events after max retries
- Use **`event-replay-strategy`** for replaying events after consumer bugs are fixed
- Use **`event-ordering-strategy`** for guaranteeing ordered processing of related events

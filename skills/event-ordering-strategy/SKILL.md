---
name: event-ordering-strategy
description: 'Ordered event processing guarantees for SaaS systems. Use when: event ordering, ordered events, event sequence, out-of-order events, event ordering guarantee, Kafka partition key, partition ordering, aggregate ordering, version ordering, sequence number, optimistic concurrency events, inbox table, sequential inbox, event ordering pattern, ordered consumer, event processing order, FIFO, out-of-order detection, deferred event, event reorder, ordering by tenant, ordering by aggregate, ordering consistency, event version.'
argument-hint: 'Describe the aggregate or entity for which ordering matters (e.g., subscription state transitions, payment events), the event volume, the event source (Kafka, PostgreSQL outbox, SQS), and what incorrect ordering would cause (state machine corruption, double charge).'
---

# Event Ordering Strategy

## When to Use

Invoke this skill when you need to:
- Guarantee that events for the same aggregate are always processed in sequence (e.g., subscription state transitions)
- Detect out-of-order event delivery before applying state changes
- Design Kafka partition keys so events for the same aggregate land on the same partition
- Use an inbox table to serialize events for one aggregate on a single consumer
- Implement optimistic concurrency on aggregates so stale events are rejected
- Handle deferred processing: hold an out-of-order event until its predecessor arrives

---

## Why Ordering Fails and How to Detect It

| Root cause | Symptom | Fix |
|---|---|---|
| Multiple Kafka partitions, no partition key | Events for same tenant on different partitions, consumed in parallel | Use `tenant_id` or `aggregate_id` as partition key |
| Concurrent workers processing same aggregate | Two workers race on same subscription | Inbox table with `FOR UPDATE` or optimistic concurrency |
| Provider sends webhooks in parallel threads | Stripe sends `invoice.created` after `invoice.paid` | HMAC-verified dedup + version check on aggregate |
| Replay runs alongside production | Old events arrive after new state is applied | Replay consumer checks version before applying |
| Network retry delivers reordered messages | SQS visibility timeout causes redelivery after newer event | Version field on aggregate + `ON CONFLICT DO NOTHING` |

---

## Step 1 — Kafka: Partition Key Design

```go
// Choose partition key to guarantee same-partition delivery for related events
// Rule: all events for one aggregate MUST share the same partition key

// Option A: aggregate_id as partition key (finest granularity)
func partitionKey(event CloudEvent) string {
    return event.AggregateID // e.g., subscription UUID
}

// Option B: tenant_id as partition key (coarser — all tenant events ordered together)
func partitionKey(event CloudEvent) string {
    return event.TenantID
}

// Produce to Kafka with explicit key
func (p *KafkaProducer) Publish(ctx context.Context, topic string, event CloudEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return err
    }
    msg := &sarama.ProducerMessage{
        Topic: topic,
        Key:   sarama.StringEncoder(partitionKey(event)),
        Value: sarama.ByteEncoder(payload),
    }
    _, _, err = p.producer.SendMessage(msg)
    return err
}
```

---

## Step 2 — Aggregate Version for Optimistic Concurrency

```sql
-- Add a monotonically increasing version to the aggregate table
ALTER TABLE subscriptions ADD COLUMN version BIGINT NOT NULL DEFAULT 1;

-- Optimistic update: only apply if version matches expected
UPDATE subscriptions
SET    status = $1,
       version = version + 1,
       updated_at = now()
WHERE  id = $2
  AND  version = $3;   -- expected version MUST match

-- 0 rows updated = optimistic lock conflict → event out of order
```

```go
func (r *SubscriptionRepository) Apply(ctx context.Context, id uuid.UUID, expectedVersion int64, update SubscriptionUpdate) error {
    result, err := r.db.ExecContext(ctx, `
        UPDATE subscriptions
        SET    status = $1, plan_id = $2, version = version + 1, updated_at = now()
        WHERE  id = $3 AND version = $4
    `, update.Status, update.PlanID, id, expectedVersion)
    if err != nil {
        return err
    }
    rows, _ := result.RowsAffected()
    if rows == 0 {
        return ErrVersionConflict // caller decides: retry or dead-letter
    }
    return nil
}

var ErrVersionConflict = errors.New("version conflict: event is out of order or already applied")
```

---

## Step 3 — Sequential Inbox Table (In-Process Ordering)

When broker partitioning is not available (e.g., SQS, HTTP webhooks), use a per-aggregate inbox:

```sql
CREATE TABLE aggregate_inbox (
    id           BIGSERIAL   PRIMARY KEY,
    aggregate_id UUID        NOT NULL,  -- subscription_id, tenant_id, etc.
    aggregate_type TEXT      NOT NULL,
    event_id     TEXT        NOT NULL UNIQUE,
    event_type   TEXT        NOT NULL,
    expected_version BIGINT  NOT NULL,  -- version the event expects to find
    payload      JSONB       NOT NULL,
    status       TEXT        NOT NULL DEFAULT 'pending',  -- 'pending' | 'applied' | 'skipped'
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Worker claims all pending events for a specific aggregate IN ORDER
CREATE INDEX idx_inbox_aggregate_pending ON aggregate_inbox(aggregate_id, id)
    WHERE status = 'pending';
```

```go
// InboxProcessor serializes event application per aggregate
func (p *InboxProcessor) ProcessAggregate(ctx context.Context, aggregateID uuid.UUID) error {
    // Claim this aggregate exclusively — no concurrent processing
    return p.db.WithTx(ctx, func(tx *sql.Tx) error {
        // Lock the aggregate row
        var currentVersion int64
        err := tx.QueryRowContext(ctx, `
            SELECT version FROM subscriptions WHERE id = $1 FOR UPDATE
        `, aggregateID).Scan(&currentVersion)
        if err != nil {
            return err
        }

        // Fetch all pending inbox events for this aggregate, in arrival order
        rows, err := tx.QueryContext(ctx, `
            SELECT id, event_id, event_type, expected_version, payload
            FROM aggregate_inbox
            WHERE aggregate_id = $1 AND status = 'pending'
            ORDER BY id ASC
            FOR UPDATE SKIP LOCKED
        `, aggregateID)
        if err != nil {
            return err
        }
        defer rows.Close()

        for rows.Next() {
            var item InboxItem
            rows.Scan(&item.ID, &item.EventID, &item.EventType, &item.ExpectedVersion, &item.Payload)

            if item.ExpectedVersion != currentVersion {
                // Out of order: mark as deferred and stop processing — wait for predecessor
                slog.Warn("out-of-order inbox event — deferring",
                    "event_id", item.EventID,
                    "expected_version", item.ExpectedVersion,
                    "current_version", currentVersion,
                )
                break
            }

            // Apply the event to the aggregate
            if err := p.applyEvent(ctx, tx, aggregateID, item); err != nil {
                return err
            }

            // Update inbox status and aggregate version atomically
            tx.ExecContext(ctx, `UPDATE aggregate_inbox SET status = 'applied' WHERE id = $1`, item.ID)
            currentVersion++
        }
        return nil
    })
}
```

---

## Step 4 — Out-of-Order Detection in Consumers

```go
// Consumer that checks event sequence number before applying
func (c *SubscriptionEventConsumer) Handle(ctx context.Context, event CloudEvent) error {
    var data SubscriptionEventPayload
    if err := json.Unmarshal(event.Data, &data); err != nil {
        return fmt.Errorf("%w: %s", ErrUnrecoverable, err)
    }

    err := c.repo.Apply(ctx, data.SubscriptionID, data.ExpectedVersion, SubscriptionUpdate{
        Status: data.NewStatus,
        PlanID: data.PlanID,
    })
    if errors.Is(err, ErrVersionConflict) {
        // Check if already applied (idempotency — version is now higher)
        current, _ := c.repo.GetVersion(ctx, data.SubscriptionID)
        if current > data.ExpectedVersion {
            return nil // event already applied at this version — safe to skip
        }
        // True out-of-order: predecessor not yet applied — schedule retry
        return fmt.Errorf("out-of-order event, will retry: %w", err)
    }
    return err
}
```

---

## Ordering Strategy Selection Table

| Scenario | Recommended pattern |
|---|---|
| Kafka + high volume | Partition key = `aggregate_id` |
| Kafka + low volume (few aggregates) | Partition key = `tenant_id` |
| SQS / HTTP webhooks | Inbox table + aggregate `FOR UPDATE` |
| In-process events (single instance) | Ordered channel buffered by `aggregate_id` |
| Multi-instance PostgreSQL workers | Inbox table + version column + `FOR UPDATE SKIP LOCKED` |
| Replay / recovery | Version check before apply — skip already-applied events |

---

## Quality Checks

- [ ] All Kafka producers set `Key` to `aggregate_id` or `tenant_id` — not empty, not random
- [ ] Aggregate table has `version BIGINT NOT NULL DEFAULT 1` column
- [ ] `UPDATE ... WHERE version = expected` returns 0 rows on conflict — caller handles it
- [ ] Out-of-order events are scheduled for retry (not dead-lettered immediately)
- [ ] Inbox processor uses `FOR UPDATE` on the aggregate row — never two workers at once
- [ ] Version conflict is distinguishable from "already applied" — check current version
- [ ] Consumers declare their expected ordering dependency in comments

## After Completion

- Use **`job-idempotency`** for idempotent event application (precondition for safe retry on version conflict)
- Use **`dead-letter-handling`** for events that remain out-of-order after max retry attempts
- Use **`event-driven-saas-patterns`** for the outbox that produces correctly-keyed events
- Use **`data-consistency-patterns`** for deeper coverage of isolation levels and optimistic locking

---
name: event-driven-architecture
description: 'Event-based communication, decoupling strategies, and reliable event publishing. Use when: event-driven architecture, EDA, event-based communication, pub/sub, event bus, event sourcing, domain events, reliable event publishing, outbox pattern, event ordering, event schema design, event versioning, CloudEvents, integration events, event broker, Kafka, EventBridge, event decoupling.'
argument-hint: 'System or integration to review — describe what events exist, what produces them, and what consumes them'
---

# Event-Driven Architecture Specialist

## When to Use
- Designing event-driven communication between services or bounded contexts
- Reviewing event publishing reliability (events lost on crashes, dual-write problem)
- Designing event schema for evolvability and consumer compatibility
- Evaluating when to use events vs direct API calls vs async queues
- Implementing the outbox pattern for guaranteed event delivery

---

## Step 1 — Event Classification

Not all "events" are equal. Distinguish by purpose:

| Type | Purpose | Consumer | Example |
|---|---|---|---|
| **Domain event** | Records a business fact that occurred | Internal services/handlers in same bounded context | `OrderConfirmed`, `PaymentReceived` |
| **Integration event** | Communicates across bounded context or service boundaries | External services, other teams | `order.confirmed` published to event bus |
| **System event** | Infrastructure/operational state | Ops, monitoring, automation | `DeploymentCompleted`, `HealthCheckFailed` |
| **Notification event** | Triggers user-facing notification | Notification service | `InvoiceReady`, `ShipmentTracked` |

Domain events stay internal. Integration events cross service boundaries and require schema stability guarantees.

---

## Step 2 — Event Schema Design

Events must be **self-describing, versioned, and stable**:

```json
{
  "id":           "01H9XVJK4M3S-...",
  "specversion":  "1.0",
  "type":         "com.myapp.order.confirmed",
  "source":       "/services/order-service",
  "time":         "2024-03-15T14:32:01.342Z",
  "datacontenttype": "application/json",
  "dataschema":   "https://schemas.myapp.com/order.confirmed/v1.json",
  "subject":      "orders/ord_abc123",
  "data": {
    "order_id":      "ord_abc123",
    "customer_id":   "cust_xyz456",
    "tenant_id":     "ten_qrs789",
    "total_amount":  "149.90",
    "total_currency":"BRL",
    "item_count":    3,
    "confirmed_at":  "2024-03-15T14:32:01.342Z"
  }
}
```

**Schema aligned with [CloudEvents spec](https://cloudevents.io/) — use as baseline for all integration events.**

### Schema Design Rules

- [ ] `type` uses reverse-DNS namespacing: `com.myapp.order.confirmed` (avoids collision)
- [ ] `time` always UTC ISO 8601 — never local time
- [ ] `id` is globally unique (ULID or UUID v4) — used for deduplication
- [ ] `subject` identifies the primary resource affected (`orders/{id}`)
- [ ] `data` is **minimal** — include only what consumers need; avoid dumping entire aggregate state
- [ ] **Past tense** event names: `order.confirmed`, not `confirm.order` or `order.confirm`
- [ ] Schema registered and versioned in a schema registry (Confluent Schema Registry, AWS Glue, custom)

---

## Step 3 — Reliable Event Publishing: The Outbox Pattern

**The dual-write problem:** publishing an event to a message broker at the same time as a DB commit is not atomic. If the service crashes between the DB write and the broker publish, the event is lost.

```
# UNSAFE — event can be lost if crash occurs between steps 2 and 3
def confirm_order(order_id: str):
    order.confirm()                    # step 1: update DB
    db.commit()                        # step 2: commit
    event_bus.publish(OrderConfirmed)  # step 3: publish — crash here = event lost!
```

### Outbox Pattern Implementation

```python
# SAFE — event stored atomically with business data; separate relay publishes
def confirm_order(order_id: str, db: Session):
    order = db.get(Order, order_id)
    order.confirm()

    # Store event in same transaction as business data
    event = OutboxEvent(
        id=ulid.new(),
        aggregate_type='order',
        aggregate_id=order_id,
        event_type='com.myapp.order.confirmed',
        payload=json.dumps(OrderConfirmedPayload.from_order(order)),
        created_at=datetime.utcnow(),
        published_at=None,              # will be set by relay when published to broker
    )
    db.add(event)
    db.commit()   # ← order update + event stored atomically
```

```python
# Outbox relay — polls unpublished events and publishes to broker
@scheduler.task('*/5 * * * * *')  # every 5 seconds
def relay_outbox_events():
    events = db.query(OutboxEvent).filter(
        OutboxEvent.published_at.is_(None)
    ).order_by(OutboxEvent.created_at).limit(100).all()

    for event in events:
        try:
            event_bus.publish(topic=event.event_type, payload=event.payload)
            event.published_at = datetime.utcnow()
            db.commit()
        except Exception as e:
            logger.error("Outbox relay failed", event_id=event.id, error=str(e))
```

```sql
-- Outbox table schema
CREATE TABLE outbox_events (
    id            TEXT PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    aggregate_id   TEXT NOT NULL,
    event_type     TEXT NOT NULL,
    payload        JSONB NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at   TIMESTAMPTZ,
    attempts       INT NOT NULL DEFAULT 0
);
CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at) WHERE published_at IS NULL;
```

**Outbox guarantees at-least-once delivery.** Consumers must be idempotent (use `event.id` as deduplication key).

---

## Step 4 — Consumer Design Patterns

### Idempotent Consumer

```python
class OrderConfirmedHandler:
    def handle(self, event: CloudEvent) -> None:
        event_id = event['id']

        # Idempotency: skip if already processed
        if self.event_store.already_processed(event_id):
            logger.info("Duplicate event skipped", event_id=event_id)
            return

        data = event['data']
        self.notification_service.send_order_confirmation(
            customer_id=data['customer_id'],
            order_id=data['order_id'],
        )

        self.event_store.mark_processed(event_id, ttl=timedelta(days=7))
```

### Competing Consumers (Fan-Out)

```
[Event Bus / Topic]
       │
       ├──→ [Queue A] → [Consumer Group A: Notification Service Workers]
       ├──→ [Queue B] → [Consumer Group B: Fulfillment Service Workers]
       └──→ [Queue C] → [Consumer Group C: Analytics Workers]
```

Each downstream service has its own queue/subscription — processes at its own pace; failure in one does not affect others.

---

## Step 5 — Event Ordering and Causality

### When Order Matters

```python
# Problem: events for the same order arrive out of order
# OrderShipped arrives before OrderConfirmed — shipping service processes in wrong state

# Solution 1: Partition key — all events for same order go to same partition (Kafka)
producer.produce(
    topic='orders',
    key=order_id,           # partition key → same order always same partition → ordered
    value=event_json,
)

# Solution 2: Version/sequence number — consumers detect and handle out-of-order
class OrderEventHandler:
    def handle(self, event: OrderEvent):
        current_version = order_state.get_version(event.order_id)
        if event.sequence <= current_version:
            logger.info("Out-of-order or duplicate event ignored", seq=event.sequence)
            return
        # process...
        order_state.update_version(event.order_id, event.sequence)
```

### Saga Pattern for Multi-Step Processes

When a business process spans multiple services, use a saga instead of a distributed transaction:

```
Order Saga:
  1. OrderService         → publishes OrderCreated
  2. InventoryService     → reserves stock → publishes StockReserved
  3. PaymentService       → charges card  → publishes PaymentCaptured
  4. FulfillmentService   → ships order   → publishes ShipmentDispatched

# On failure at step 3:
  3. PaymentService       → fails         → publishes PaymentFailed
  2. InventoryService     → hears PaymentFailed → releases stock (compensating transaction)
  1. OrderService         → hears PaymentFailed → marks order as failed → notifies customer
```

**Compensating transactions** are the rollback mechanism in event-driven sagas.

---

## Step 6 — Event Schema Evolution

Events are public contracts. Breaking changes must be managed carefully:

| Change type | Safe? | Strategy |
|---|---|---|
| Add optional field to `data` | Yes | Add with null/default; consumers ignore unknown fields |
| Remove a field from `data` | No | Deprecate first; maintain for min 1 version cycle |
| Rename a field | No | Add new name; keep old for transition period |
| Change field type | No | New version of event type; dual-publish during transition |
| Change event type name | No | New type; deprecate old; transition period |

```json
// Versioning via type namespace
"type": "com.myapp.order.confirmed.v2"   // breaking change → new type version

// Consumers subscribe to both during migration:
// "com.myapp.order.confirmed.v1" and "com.myapp.order.confirmed.v2"
```

---

## Step 7 — Observability for Event-Driven Systems

```python
# Trace events through the system using correlation_id
logger.info("Event published", extra={
    "event_id":       event.id,
    "event_type":     event.type,
    "correlation_id": event.correlation_id,  # trace back to originating request
    "aggregate_id":   event.subject,
    "topic":          topic,
})

# Consumer logging
logger.info("Event consumed", extra={
    "event_id":         event.id,
    "event_type":       event.type,
    "correlation_id":   event.correlation_id,
    "consumer":         self.__class__.__name__,
    "processing_ms":    elapsed_ms,
    "outcome":          "processed" | "duplicate" | "failed",
})
```

| Metric | Alert |
|---|---|
| Outbox relay lag | > 30s of unpublished events |
| Consumer lag per subscription | > 5 min behind |
| Event processing error rate | > 1% |
| DLQ depth per subscription | > 0 |

---

## Output Report

```
## Event-Driven Architecture Review: <system>

### Critical
- Events published directly to broker in same code path as DB commit (dual-write)
  Service crash between commit and publish causes silent event loss with no recovery path
  Fix: implement outbox pattern; store event in DB atomically with business data; relay separately

- Consumer performs duplicate actions on redelivery (sends duplicate emails, double-charges)
  At-least-once delivery guarantees redelivery on crash; idempotency required
  Fix: use event.id as idempotency key; check Redis before processing; mark done after success

### High
- No schema registry — event structure evolves without coordination with consumers
  Consumers break silently when producer adds/renames fields
  Fix: adopt CloudEvents spec; register event schemas; enforce backward compatibility in CI

- Events published to a single topic with all event types — no partition key set
  Events for same Order arrive out of order; fulfillment processes shipped before confirmed
  Fix: set partition key to order_id; all order events on same partition → ordered delivery

### Medium
- No outbox cleanup job — outbox_events table grows unboundedly
  Fix: add scheduled job to delete published events older than 30 days

- Integration events include full aggregate state (all 40 fields) — consumers only need 5 fields
  Increases coupling; schema changes in aggregate break consumers unnecessarily
  Fix: design event payload with only the fields consumers need; consumer fetches additional data if required

### Low
- Event type names use present tense ("order.confirm") vs past tense convention
  Fix: rename to "order.confirmed" — events record what happened, not what to do

### Passed
- Consumer groups configured per downstream service — independent failure isolation ✓
- Correlation IDs propagated through all event payloads ✓
- DLQ configured with depth alerting for all subscriptions ✓
```

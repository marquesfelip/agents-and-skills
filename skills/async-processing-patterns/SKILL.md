---
name: async-processing-patterns
description: 'Queues, workers, retries, and safe asynchronous execution models. Use when: async processing, message queue design, worker design, retry strategy, job queue, background processing, queue consumer, dead letter queue, DLQ, message broker, RabbitMQ, SQS, Kafka, async worker, exponential backoff, async execution, task queue, Celery, async job.'
argument-hint: 'Async processing flow or worker code to review — or describe the operation being moved to async processing and the infrastructure available (SQS, RabbitMQ, Kafka, Redis Queue, etc.)'
---

# Async Processing Patterns Specialist

## When to Use
- Designing a new asynchronous processing flow (queue + worker)
- Reviewing existing workers for safety, reliability, and correctness
- Debugging message loss, duplicate processing, or queueing failures
- Selecting the right messaging pattern for a given use case
- Implementing retry and dead letter queue strategies

---

## Step 1 — When to Use Async Processing

Not every operation needs to be async. Apply this decision framework:

| Use async when | Keep synchronous when |
|---|---|
| Operation takes > 500ms (non-user-facing) | User expects immediate result |
| Operation can tolerate eventual consistency | Strong consistency required |
| Work can be retried independently | All-or-nothing with DB transaction needed |
| Multiple downstream systems must be notified | Single downstream system, simple flow |
| Decoupling producers from consumers is valuable | Complexity of async outweighs decoupling benefit |
| Peak load exceeds synchronous capacity | Load is steady and predictable |

---

## Step 2 — Message Schema Design

Messages must be self-contained and versioned:

```python
@dataclass
class SendInvoiceEmailMessage:
    # Identity
    message_id: str        # UUID v4 — used for deduplication
    schema_version: str    # "1.0" — for graceful evolution
    # Context
    tenant_id: str         # always include for multi-tenant systems
    correlation_id: str    # trace back to originating request
    # Payload
    order_id: str
    customer_email: str
    invoice_url: str
    locale: str
    # Metadata
    enqueued_at: str       # ISO 8601 UTC — when was it queued
    max_attempts: int = 3  # processing policy carried with message

# Serialized as JSON with explicit field names (no positional encoding)
```

### Message Design Rules

- [ ] Messages are **immutable** after enqueue — workers read-only
- [ ] `message_id` included in every message for idempotency key
- [ ] `schema_version` present for future evolution
- [ ] Payload is **self-contained** — worker should not need to re-fetch from DB to process
- [ ] Messages are **small** (< 256 KB for SQS) — store large payloads in S3/blob; pass reference
- [ ] Sensitive data (PII, payment info) in message payload should follow data classification rules (encrypt field if required)
- [ ] `enqueued_at` timestamp for queue latency monitoring

---

## Step 3 — Worker Design Patterns

### Idempotent Consumer (Critical)

Workers will receive the same message more than once. Design every consumer to be idempotent:

```python
def process_send_invoice_email(message: SendInvoiceEmailMessage) -> None:
    # Check if already processed — idempotency gate
    if processed_messages.exists(message.message_id):
        logger.info("Skipping duplicate message", message_id=message.message_id)
        return   # already done; safe to acknowledge without reprocessing

    # Perform the operation
    email_client.send(
        to=message.customer_email,
        template="invoice",
        invoice_url=message.invoice_url,
    )

    # Mark as processed AFTER successful operation
    processed_messages.mark_done(message.message_id, ttl=timedelta(days=7))
```

```python
# Idempotency store — Redis with TTL
class RedisMessageIdempotencyStore:
    def exists(self, message_id: str) -> bool:
        return bool(self._redis.get(f"processed:{message_id}"))

    def mark_done(self, message_id: str, ttl: timedelta) -> None:
        self._redis.setex(f"processed:{message_id}", int(ttl.total_seconds()), "1")
```

### Worker Concurrency Model

```python
# Pattern: bounded concurrent consumers
import asyncio
from asyncio import Semaphore

async def run_worker(queue: AsyncQueue, max_concurrent: int = 10):
    semaphore = Semaphore(max_concurrent)

    async def handle_with_limit(message):
        async with semaphore:
            await process_message(message)
            await queue.acknowledge(message)

    async for message in queue.stream():
        asyncio.create_task(handle_with_limit(message))
```

---

## Step 4 — Retry Strategy Design

### Exponential Backoff with Jitter

```python
import random
import time

def calculate_backoff(attempt: int, base_delay_s: float = 1.0, max_delay_s: float = 300.0) -> float:
    """Exponential backoff with full jitter to prevent thundering herd."""
    exponential = base_delay_s * (2 ** attempt)
    capped = min(exponential, max_delay_s)
    jitter = random.uniform(0, capped)   # full jitter
    return jitter

# Attempt 0 → 0–1s
# Attempt 1 → 0–2s
# Attempt 2 → 0–4s
# Attempt 3 → 0–8s
# Attempt 5 → 0–32s
# Attempt 8 → 0–300s (capped)
```

### Retry Policy Decision Table

| Error type | Retry? | Strategy |
|---|---|---|
| Transient (network timeout, 503) | Yes | Exponential backoff |
| Rate limited (429) | Yes | Respect `Retry-After` header |
| Resource not found (404) | No | Dead-letter immediately |
| Validation error (400/422) | No | Dead-letter immediately — retrying won't fix it |
| Concurrency conflict (409) | Yes | Short backoff; small number of retries |
| Database unavailable | Yes | Long backoff; high retry count |
| Business rule violation | No | Dead-letter; notify for review |

```python
class RetryPolicy:
    def should_retry(self, error: Exception, attempt: int, max_attempts: int) -> bool:
        if attempt >= max_attempts:
            return False
        if isinstance(error, (ValidationError, NotFoundError, BusinessRuleError)):
            return False   # non-transient — retrying won't help
        return True        # transient — retry with backoff
```

---

## Step 5 — Dead Letter Queue Pattern

Messages that fail all retries go to a Dead Letter Queue (DLQ) for inspection and recovery:

```
[Producer] → [Main Queue] → [Worker]
                               ↓ (all retries exhausted)
                           [Dead Letter Queue] → [DLQ Monitor / Alert]
                                                    ↓ (manual or automated)
                                               [Replay / Discard / Fix]
```

### DLQ Requirements

- [ ] DLQ configured for every processing queue — no unconditional message loss
- [ ] DLQ messages retain original payload + failure metadata:
  ```python
  class DLQMessage:
      original_message: dict
      failure_reason: str        # last exception message
      failure_type: str          # exception class name
      attempt_count: int
      first_failed_at: str       # ISO 8601 UTC
      last_failed_at: str        # ISO 8601 UTC
      original_queue: str
  ```
- [ ] DLQ depth alerted on (CloudWatch alarm / Datadog monitor / PagerDuty)
- [ ] DLQ provides replay mechanism — requeue to main queue after fix deployed
- [ ] DLQ messages have a longer TTL than main queue (human needs time to investigate)

---

## Step 6 — Queue Infrastructure Selection

| Broker | Best for | Watch out for |
|---|---|---|
| AWS SQS (Standard) | Simple fan-out, at-least-once, serverless | No ordering; deduplication window only 5 min |
| AWS SQS (FIFO) | Ordered, exactly-once within group | Lower throughput (3000 msg/s); higher cost |
| RabbitMQ | Complex routing, multiple exchange patterns | Self-managed ops complexity |
| Apache Kafka | High-throughput event streaming, log replay | High operational overhead; not a task queue |
| Redis (Streams / Lists) | Low-latency ephemeral queuing | Data loss risk if not configured persistently |
| Google Pub/Sub | GCP-native fan-out, at-least-once | No ordering by default |

### Visibility Timeout / Ack Deadline Sizing

Set visibility timeout > max expected processing time + safety margin:

```python
# SQS VisibilityTimeout = max_processing_time * 1.5 + buffer
# If job takes up to 30s: VisibilityTimeout = 60s (30 * 1.5 + 15)

# Extend visibility while long jobs run
async def process_long_job(message: SQSMessage):
    async def extend_visibility():
        while not done:
            await asyncio.sleep(20)
            await sqs.change_message_visibility(
                message.receipt_handle,
                visibility_timeout=60,
            )
    extend_task = asyncio.create_task(extend_visibility())
    try:
        await do_work(message)
    finally:
        done = True
        extend_task.cancel()
```

---

## Step 7 — Monitoring and Observability

| Metric | Alert threshold | Action |
|---|---|---|
| Queue depth (main) | > 1000 messages | Scale up workers |
| Queue depth (DLQ) | > 0 | Investigate failures |
| Consumer lag | > 5 min | Scale or debug worker |
| Message processing duration | P99 > 30s | Optimize or increase timeout |
| Worker error rate | > 1% of messages | Investigate error patterns |
| Retry rate | > 10% of messages | Check downstream service health |

```python
# Structured metrics on every message processed
logger.info("Message processed", extra={
    "message_id": message.message_id,
    "queue": queue_name,
    "attempt": attempt_number,
    "duration_ms": elapsed_ms,
    "outcome": "success" | "failed" | "skipped_duplicate",
})
```

---

## Output Report

```
## Async Processing Review: <worker/queue>

### Critical
- Worker acknowledges the message BEFORE processing — message lost on crash mid-processing
  Fix: acknowledge only AFTER successful processing (or failed+DLQ'd); use ack-on-completion pattern

- No idempotency check — every duplicate delivery re-sends the invoice email
  Customers receive multiple identical emails on queue redelivery
  Fix: implement message_id deduplication using Redis with 7-day TTL

### High
- No Dead Letter Queue configured — messages that fail all retries are silently discarded
  No visibility into failed messages; data loss on transient failures after max retries
  Fix: configure DLQ on main queue; set up DLQ depth alert; implement replay mechanism

- Retry uses fixed 5s delay — thundering herd when external service recovers from outage
  All backed-up messages retry simultaneously, overwhelming the recovering service
  Fix: implement exponential backoff with full jitter; respect Retry-After headers

### Medium
- Visibility timeout set to 30s; worker takes up to 45s on large payloads
  Messages reappear in queue before processing completes; duplicate processing occurs
  Fix: set VisibilityTimeout to 90s; extend visibility heartbeat for long-running jobs

- Validation errors retried 3 times before DLQ — malformed messages waste retries
  Fix: detect ValidationError; dead-letter immediately without retrying

### Low
- No structured logging on message processing — cannot measure per-message latency
  Fix: log message_id, queue, attempt, duration_ms, outcome on every message

### Passed
- Schema versioning included in all message payloads ✓
- Worker concurrency bounded (max 10 concurrent) ✓
- DLQ replay mechanism automated via CLI tool ✓
```

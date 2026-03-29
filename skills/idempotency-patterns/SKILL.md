---
name: idempotency-patterns
description: 'Idempotent operations, request replay safety, and duplicate execution prevention. Use when: idempotency, idempotency key, request deduplication, duplicate execution prevention, replay safety, at-least-once delivery, payment idempotency, API idempotency, safe retry, duplicate request, idempotent consumer, idempotent API, double payment prevention, webhook idempotency.'
argument-hint: 'Operation or endpoint to make idempotent — describe the action, data flow, and infrastructure available (Redis, DB, etc.)'
---

# Idempotency Patterns Specialist

## When to Use
- Making an API endpoint safe to retry without causing duplicate side effects
- Designing consumers that handle at-least-once delivery from queues/brokers
- Preventing double charges, double sends, or duplicate record creation
- Implementing idempotency keys for payment or financial operations
- Reviewing existing operations for duplicate execution risks

---

## Step 1 — Identify Non-Idempotent Operations

Idempotency matters most for operations with side effects. Audit every operation:

| Idempotent by nature | Not idempotent — must be made safe |
|---|---|
| GET, HEAD (read-only) | POST /orders (creates new record) |
| PUT (replace full resource) | POST /payments/charge (charges card) |
| DELETE (second call = 404) | POST /emails/send (sends email) |
| Configuration updates (SET) | POST /notifications/push (sends push) |
| Cache writes (upsert) | POST /invoices (generates invoice) |
| | PATCH /account/balance +N (increment — not idempotent!) |

**Rule:** Any operation that creates a new record, sends a message, charges money, or increments a counter is non-idempotent and must be protected.

---

## Step 2 — Idempotency Key Design

An idempotency key uniquely identifies a request or operation. If the same key is seen again, return the original result.

### Key Source Options

| Source | Strategy | Use case |
|---|---|---|
| Client-provided | Client generates UUID/ULID; sends in `Idempotency-Key` header | Payment APIs, financial APIs |
| Derived from payload | Hash of deterministic payload fields | Event consumers, background jobs |
| Request content hash | SHA-256 of canonical payload | When client cannot provide a key |
| Business field | Natural business key (order_id + action) | Internal operations with known uniqueness |

```python
# Client-provided header (Stripe-style)
@app.post('/payments')
def charge_payment(request: Request, body: ChargeRequest):
    idempotency_key = request.headers.get('Idempotency-Key')
    if not idempotency_key:
        raise HTTPException(400, "Idempotency-Key header required")
    if len(idempotency_key) > 255:
        raise HTTPException(400, "Idempotency-Key too long")
    ...

# Derived from payload (event consumer)
def derive_idempotency_key(event: dict) -> str:
    return f"order_confirmed:{event['order_id']}:{event['event_id']}"
```

### Key Scoping

Always scope the key per operation type:

```python
# BAD — same key space for different operations; collision risk
stored_result = redis.get(idempotency_key)

# GOOD — namespaced by operation
stored_result = redis.get(f"payments:charge:{idempotency_key}")
stored_result = redis.get(f"emails:invoice:{idempotency_key}")
```

---

## Step 3 — Implementation Patterns

### Pattern 1: Redis-Based Idempotency Store

```python
class IdempotencyStore:
    def __init__(self, redis: Redis, namespace: str, ttl: timedelta = timedelta(days=7)):
        self._redis = redis
        self._namespace = namespace
        self._ttl = ttl

    def _key(self, idempotency_key: str) -> str:
        return f"idempotency:{self._namespace}:{idempotency_key}"

    def get_cached_result(self, idempotency_key: str) -> dict | None:
        raw = self._redis.get(self._key(idempotency_key))
        return json.loads(raw) if raw else None

    def store_result(self, idempotency_key: str, result: dict) -> None:
        self._redis.setex(
            self._key(idempotency_key),
            int(self._ttl.total_seconds()),
            json.dumps(result),
        )

# Usage at API handler
@app.post('/payments/charge')
def charge_payment(request: Request, body: ChargeRequest):
    key = request.headers.get('Idempotency-Key')
    store = IdempotencyStore(redis, namespace='payments:charge')

    # Return cached result if key was already processed
    cached = store.get_cached_result(key)
    if cached:
        return JSONResponse(cached, status_code=200)

    # Process the operation
    result = payment_gateway.charge(body.amount, body.card_token)

    # Cache the result for replay
    store.store_result(key, result.dict())
    return result
```

### Pattern 2: Database-Based Idempotency (Stronger Consistency)

```sql
CREATE TABLE idempotency_keys (
    key_hash     TEXT PRIMARY KEY,          -- SHA-256 of namespace + key
    namespace    TEXT NOT NULL,
    status       TEXT NOT NULL DEFAULT 'processing',  -- processing | completed | failed
    response     JSONB,                     -- cached response payload
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    expires_at   TIMESTAMPTZ NOT NULL       -- enforce TTL via scheduled cleanup
);
```

```python
def execute_idempotent(namespace: str, key: str, operation: Callable) -> dict:
    key_hash = hashlib.sha256(f"{namespace}:{key}".encode()).hexdigest()

    with db.transaction():
        try:
            # INSERT or conflict — ensures only one execution in-progress
            db.execute("""
                INSERT INTO idempotency_keys (key_hash, namespace, expires_at)
                VALUES (%s, %s, now() + interval '7 days')
                ON CONFLICT (key_hash) DO NOTHING
            """, (key_hash, namespace))

            row = db.fetchone("SELECT status, response FROM idempotency_keys WHERE key_hash = %s", key_hash)

            if row['status'] == 'completed':
                return row['response']          # replay cached result
            if row['status'] == 'processing':
                raise ConcurrentRequestError()  # another request is processing this key

            # Execute the operation
            result = operation()

            # Mark complete with result
            db.execute("""
                UPDATE idempotency_keys SET status = 'completed', response = %s, completed_at = now()
                WHERE key_hash = %s
            """, (json.dumps(result), key_hash))

            return result
        except Exception as e:
            db.execute("UPDATE idempotency_keys SET status = 'failed' WHERE key_hash = %s", key_hash)
            raise
```

---

## Step 4 — Handling Concurrent Duplicate Requests

When two identical requests arrive simultaneously (network retry storm), only one must succeed:

```python
# Race condition: two requests with same key arrive at same millisecond
# Both check cache: both see "not found"
# Both execute operation: double charge!

# Solution: distributed lock + idempotency check
def process_with_lock(idempotency_key: str, operation: Callable) -> dict:
    lock_key = f"lock:payment:{idempotency_key}"

    # Acquire distributed lock — only one process proceeds
    with redis_lock(lock_key, timeout=30):
        # Check again under lock
        cached = idempotency_store.get_cached_result(idempotency_key)
        if cached:
            return cached   # another thread completed it while we waited for lock

        result = operation()
        idempotency_store.store_result(idempotency_key, result)
        return result
```

---

## Step 5 — Idempotent Consumer for Queues

```python
class IdempotentConsumer:
    """Wraps any handler to make it idempotent under at-least-once delivery."""

    def __init__(self, handler: Callable, store: IdempotencyStore):
        self._handler = handler
        self._store = store

    def process(self, event: dict) -> None:
        # Derive unique key from event identity
        event_id = event.get('id') or self._derive_key(event)
        namespace = f"consumer:{self._handler.__class__.__name__}"

        cached = self._store.get_cached_result(f"{namespace}:{event_id}")
        if cached:
            logger.info("Duplicate event skipped", event_id=event_id)
            return   # already processed; do nothing

        # Process the event
        self._handler.handle(event)

        # Mark as processed (store minimal marker, not full result)
        self._store.store_result(f"{namespace}:{event_id}", {"processed": True})

    def _derive_key(self, event: dict) -> str:
        # Fallback: hash stable fields to derive a key
        stable = {k: event[k] for k in sorted(event.keys()) if k != 'timestamp'}
        return hashlib.sha256(json.dumps(stable, sort_keys=True).encode()).hexdigest()
```

---

## Step 6 — Idempotency Key TTL Strategy

| Operation type | Recommended TTL | Reason |
|---|---|---|
| Payment / financial | 30 days | Long enough to handle delayed retries and disputes |
| Email send | 7 days | Covers standard retry windows |
| Account creation | 1 day | Short-lived; creation attempt is single window |
| Event consumer | 7 days | At-least-once window; message retention period |
| Webhook delivery | 24 hours | Typical webhook retry window |
| Background job | Job schedule interval + 50% | Prevents re-run if job fires twice |

After TTL expiry, the same key can be reused — this is intentional (the original request is considered safely expired).

---

## Step 7 — API Contract for Idempotency

Expose idempotency key requirements explicitly in your API:

```http
# Request
POST /v1/payments/charge
Idempotency-Key: 7f3d9a2e-1234-4abc-8def-56789abcdef0
Content-Type: application/json

{ "amount": 4990, "currency": "BRL", "card_token": "tok_..." }

# First call (new key) → 201 Created
HTTP/1.1 201 Created
Idempotency-Key: 7f3d9a2e-1234-4abc-8def-56789abcdef0
{ "payment_id": "pay_xyz", "status": "captured" }

# Second call (same key, replayed) → 200 OK with same body
HTTP/1.1 200 OK
Idempotency-Key: 7f3d9a2e-1234-4abc-8def-56789abcdef0
{ "payment_id": "pay_xyz", "status": "captured" }
```

```http
# Concurrent in-flight conflict → 409
HTTP/1.1 409 Conflict
{ "error": { "code": "IDEMPOTENCY_CONFLICT", "message": "Request with this key is in progress" } }

# Different payload with same key → 422
HTTP/1.1 422 Unprocessable Entity
{ "error": { "code": "IDEMPOTENCY_PAYLOAD_MISMATCH", "message": "Idempotency key used with different request body" } }
```

---

## Output Report

```
## Idempotency Review: <operation/endpoint>

### Critical
- POST /payments/charge has no idempotency protection
  Client retries on timeout cause duplicate card charges
  Fix: require Idempotency-Key header; store result in Redis with 30-day TTL; replay on repeat key

- Order creation endpoint creates a new DB row on every call with same client payload
  Network retry on 500 creates duplicate orders for same cart
  Fix: derive idempotency key from (user_id, cart_id, timestamp_bucket); deduplicate before insert

### High
- Event consumer processes OrderConfirmed without checking for duplicate events
  Queue redelivery on worker crash causes duplicate invoice emails to customers
  Fix: use event.id as idempotency key; mark processed in Redis after successful handling

- Idempotency check uses Redis GET then SET without atomic locking
  Race condition: two simultaneous retries both see "not processed" and both execute charge
  Fix: use Redis SET NX (atomic check-and-set) or Redis lock wrapping the check+process+store

### Medium
- Idempotency keys stored without TTL — Redis memory grows unboundedly
  Fix: set EXPIRE = 7 days on all idempotency keys; add scheduled cleanup for DB-based store

- Same key namespace shared across different operations (charge, refund, cancel)
  Collision: refund idempotency key could match a charge key if UUIDs collide (low but non-zero risk)
  Fix: namespace keys by operation: "payments:charge:{key}", "payments:refund:{key}"

### Low
- Idempotency-Key not documented in OpenAPI spec — clients unaware it is required
  Fix: add header to OpenAPI spec with description and example; return 400 if missing

### Passed
- Background job deduplication using (job_type, entity_id, schedule_date) composite key ✓
- Event consumers use event.id for deduplication with 7-day TTL ✓
- Idempotency key scoped per namespace ✓
```

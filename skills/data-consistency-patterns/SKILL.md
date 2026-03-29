---
name: data-consistency-patterns
description: >
  Consistency models and data integrity guarantees in distributed and relational systems.
  Use when: data consistency, consistency model, ACID, BASE, eventual consistency, strong consistency,
  linearizability, serializability, isolation level, read committed, repeatable read, serializable,
  dirty read, phantom read, non-repeatable read, optimistic locking, pessimistic locking,
  lost update, write skew, double booking, distributed consistency, saga pattern, outbox pattern,
  two phase commit, compensating transaction, idempotency, concurrent update, distributed transaction,
  consistency guarantee, data integrity, causality, monotonic read, read your own writes.
argument-hint: >
  Describe the consistency problem or scenario (e.g., "two users can book the same seat concurrently",
  "order creation must update inventory and emit an event atomically"), and specify the database engine,
  isolation level in use, and whether the system is distributed across services.
---

# Data Consistency Patterns Specialist

## When to Use

Invoke this skill when you need to:
- Choose the correct transaction isolation level for a concurrent access pattern
- Prevent lost updates, phantom reads, write skew, and double-booking anomalies
- Design pessimistic or optimistic locking strategies
- Maintain consistency across multiple services without a distributed transaction
- Implement the outbox pattern for atomic event publishing
- Design saga-based compensation for distributed multi-step operations

---

## Step 1 — Identify the Consistency Problem

Classify the scenario before choosing a solution:

| Anomaly | Description | Example |
|---|---|---|
| **Dirty read** | Reading uncommitted data from another transaction | Reading a balance before a concurrent transfer commits |
| **Non-repeatable read** | Same row returns different value within one transaction | Row updated by another TX between two SELECTs |
| **Phantom read** | Range query returns different rows on re-execution | New row inserted into a range between two range queries |
| **Lost update** | Two transactions read-then-write the same row; one overwrites the other | Concurrent increment: both read 10, both write 11, result is 11 not 12 |
| **Write skew** | Two transactions read overlapping data and write disjoint rows based on a shared invariant | Both doctors go on leave, assuming the other is on-call |
| **Double booking** | Two transactions concurrently claim the same resource | Two users book the last seat |

Checklist:
- [ ] Anomaly type identified before choosing a solution
- [ ] Frequency of concurrent access to the affected rows estimated
- [ ] Whether the system is single-DB or multi-service determines the approach

---

## Step 2 — Transaction Isolation Levels

Match the isolation level to the anomalies that must be prevented:

| Isolation Level | Prevents | Allows | Performance Impact |
|---|---|---|---|
| **Read Uncommitted** | — | Dirty reads, non-repeatable, phantom | Fastest; rarely useful |
| **Read Committed** (default PG/MySQL) | Dirty reads | Non-repeatable reads, phantom reads, write skew | Low overhead |
| **Repeatable Read** | Dirty + non-repeatable reads | Phantom reads (in some DBs), write skew | Moderate overhead |
| **Serializable** | All anomalies | — | Highest overhead; may cause serialization failures |

**PostgreSQL notes:**
- Repeatable Read in PG also prevents phantom reads using MVCC snapshots
- Serializable uses SSI (Serializable Snapshot Isolation) — detects and aborts conflicting transactions

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... operations
COMMIT;
```

Checklist:
- [ ] Default `READ COMMITTED` used unless a specific anomaly requires higher isolation
- [ ] `SERIALIZABLE` used for scenarios with write skew or phantom reads (shift scheduling, seat booking)
- [ ] Application handles serialization failure (`ERROR 40001` in PG, `ER_LOCK_DEADLOCK` in MySQL) with retry logic
- [ ] Long-running transactions at high isolation levels minimized — they increase lock contention

---

## Step 3 — Pessimistic Locking

Explicitly lock rows to prevent concurrent modification:

```sql
-- Lock rows for update (others must wait until transaction commits)
SELECT * FROM seats WHERE id = $1 FOR UPDATE;

-- Shared lock (others can read, not write)
SELECT * FROM inventory WHERE product_id = $1 FOR SHARE;

-- Skip locked rows (non-blocking — for queue processing)
SELECT * FROM jobs WHERE status = 'pending' FOR UPDATE SKIP LOCKED LIMIT 1;
```

**When to use:**
- High contention on specific rows (booking, inventory reservation)
- Short transactions where wait time is acceptable
- Operations that must not be retried (irreversible side effects)

Checklist:
- [ ] `FOR UPDATE` used when the selected row will be modified in the same transaction
- [ ] `SKIP LOCKED` used for task queue patterns — prevents worker pile-up
- [ ] Lock acquisition order consistent across transactions — prevents deadlocks
- [ ] Timeout set to prevent indefinite waiting: `SET lock_timeout = '5s'`

---

## Step 4 — Optimistic Locking

Detect conflicts at commit time using a version counter:

**Schema:**
```sql
ALTER TABLE products ADD COLUMN version INT NOT NULL DEFAULT 0;
```

**Application flow:**
```sql
-- Read:
SELECT id, stock, version FROM products WHERE id = $1;
-- Returns: { id: 42, stock: 10, version: 7 }

-- Write (only succeeds if version hasn't changed):
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = $1 AND version = $2;
-- Check rows affected: if 0, conflict detected → retry or abort
```

**When to use:**
- Low contention — conflicts are rare
- Long user sessions where holding a database lock is impractical
- Read-heavy workloads with occasional writes

Checklist:
- [ ] `version` column present and incremented on every `UPDATE`
- [ ] Application checks `rows affected == 1` — if 0, conflict occurred
- [ ] Conflict handling strategy defined: retry, merge, or reject with user error
- [ ] ORM-level optimistic locking configured correctly (Hibernate `@Version`, ActiveRecord `lock_version`)

---

## Step 5 — Preventing Double Booking

**Database-enforced uniqueness (strongest):**
```sql
-- Only one active booking per seat per event:
CREATE UNIQUE INDEX uq_bookings_seat_event_active
  ON bookings (seat_id, event_id)
  WHERE cancelled_at IS NULL;

-- Insert will fail with unique violation if already booked:
INSERT INTO bookings (seat_id, event_id, user_id) VALUES ($1, $2, $3);
```

**Advisory locks (PostgreSQL):**
```sql
-- Acquire a session-level advisory lock keyed by the resource ID:
SELECT pg_advisory_xact_lock(seat_id::bigint);
-- ... proceed with booking logic
COMMIT;  -- lock released automatically
```

**Inventory decrement pattern:**
```sql
UPDATE seats SET status = 'booked', booked_by = $user_id
WHERE id = $seat_id AND status = 'available';
-- Check rows affected: 0 = already booked by someone else
```

Checklist:
- [ ] Unique constraint or unique partial index used as the final safety net
- [ ] Application logic does not rely solely on prior SELECT to detect availability — the UPDATE must be the arbiter
- [ ] Advisory locks used when the uniqueness constraint alone is insufficient (multi-step workflows)

---

## Step 6 — Distributed Consistency (Multi-Service)

When consistency spans multiple services, two-phase commit (2PC) is rarely practical. Use these patterns instead:

**Outbox Pattern (Atomic write + event publish):**
```sql
-- In one transaction: write business record + insert event into outbox
BEGIN;
INSERT INTO orders (id, customer_id, total) VALUES ($1, $2, $3);
INSERT INTO outbox (aggregate_id, event_type, payload)
  VALUES ($1, 'OrderCreated', '{"total": 99.90}');
COMMIT;

-- Separate process reads outbox and publishes to message broker (at-least-once)
```

**Saga Pattern (Compensating transactions):**
- Each step in a multi-service operation is a local transaction
- On failure, compensating transactions undo previous steps
- Orchestration saga: a central orchestrator directs each step
- Choreography saga: services react to events from each other

```
OrderService → emit OrderCreated
InventoryService → reserve stock → emit StockReserved
PaymentService → charge card → emit PaymentCaptured
If PaymentService fails → emit PaymentFailed
InventoryService → release stock (compensation)
OrderService → cancel order (compensation)
```

Checklist:
- [ ] Outbox table per service — never dual-write to DB and message broker in application code
- [ ] Outbox relay process is idempotent — message published at-least-once; consumer deduplicates
- [ ] Saga compensating transactions defined for every non-trivial step
- [ ] Compensating transactions are themselves idempotent and retryable
- [ ] Saga state machine persisted to DB — recovery possible after crash mid-saga

---

## Step 7 — Consistency Guarantees to Expose to Clients

Checklist:
- [ ] **Read your own writes**: user always sees their own writes immediately (route to primary or use sessions)
- [ ] **Monotonic reads**: user never sees data go backwards (stick to same replica within session)
- [ ] **Causal consistency**: operations that are causally related appear in order to all readers
- [ ] API responses include an `ETag` or `version` field to enable client-side optimistic locking
- [ ] Retry-safe operations use idempotency keys — retried requests do not double-apply

---

## Output Report

### Critical
- Double booking possible because availability checked in SELECT before a separate UPDATE — race condition
- Write skew causing an invariant violation (e.g., both on-call doctors simultaneously go off duty)
- Lost update: concurrent read-then-write sequence overwrites another transaction's changes silently
- Outbox pattern not used — dual write to DB and broker creates inconsistency on partial failure

### High
- `READ COMMITTED` used for a scenario that requires `SERIALIZABLE` — phantom reads possible
- No retry logic for serialization failures — `SERIALIZABLE` transactions abort unnecessarily
- Distributed saga has no compensation logic — partial failures leave system in inconsistent state

### Medium
- Optimistic locking version mismatch not handled — application silently drops conflicting writes
- `FOR UPDATE` lock held across a network call — long lock duration increases contention
- Advisory lock not released on failure — requires session expiry to clear

### Low
- Causal consistency not guaranteed on replica reads — monotonic read contract violated
- API does not expose `version`/`ETag` — clients cannot implement optimistic concurrency
- Idempotency keys not implemented — retried API requests double-apply state changes

### Passed
- Isolation level matched to the specific anomaly being prevented
- Pessimistic or optimistic locking applied correctly based on contention profile
- Double booking prevented at the database level via unique constraints or atomic UPDATE
- Outbox pattern used for atomic DB write + event publish
- Saga compensating transactions defined and idempotent for all distributed operations

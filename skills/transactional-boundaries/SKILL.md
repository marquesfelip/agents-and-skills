---
name: transactional-boundaries
description: 'Transaction scope, consistency guarantees, and safe state transitions. Use when: transaction design, transactional boundaries, ACID transactions, database transaction, transaction scope, consistency guarantee, distributed transaction, saga pattern, two-phase commit, isolation level, optimistic locking, pessimistic locking, transaction rollback, state transition safety, transaction per request.'
argument-hint: 'Business operation or data flow requiring transaction design — describe what entities change, what must be consistent, and whether single or multiple databases are involved'
---

# Transactional Boundaries Specialist

## When to Use
- Designing the transaction scope for a business operation that touches multiple entities
- Debugging partial writes, inconsistent state, or missing rollbacks
- Choosing between optimistic and pessimistic locking for concurrent updates
- Designing distributed operations across multiple databases or services
- Defining the correct database isolation level for a given workload

---

## Step 1 — Define What Must Be Consistent

Before writing code, identify the consistency units:

**Questions to ask:**
1. Which records must all change together, or all remain unchanged?
2. What is the minimum set of operations that must be atomic?
3. What is the business consequence if only part of the operation succeeds?
4. Are all entities in the same database / same service?

```
Example: Place an order

Must be atomic:
  ✓ decrement stock quantity          ← DB: inventory service
  ✓ create order record               ← DB: order service
  ✓ create payment intent             ← External: payment gateway (not transactional!)

Actually atomic (single DB):     → decrement stock + create order
Eventually consistent (events):  → payment captured event → order status updated
```

---

## Step 2 — Transaction Scope Design

### Rule: Transactions Should Be Short and Narrow

```python
# BAD — long transaction: holds locks across external API call
def place_order(cart_id: str):
    with db.transaction():                   # transaction starts
        items = db.query(CartItem, cart_id)
        order = Order.create(items)
        db.save(order)
        payment = stripe.charge(...)         # ← BLOCKS: external API call inside transaction!
        # If Stripe takes 5s, DB locks held for 5s — throughput destroyed
        db.save(payment_record)
    # transaction ends

# GOOD — external call outside transaction
def place_order(cart_id: str):
    # Step 1: business validation within short transaction
    with db.transaction():
        items = db.query(CartItem, cart_id)
        order = Order.create_draft(items)
        reserve_stock(items)                 # DB operation — inside transaction
        db.save(order)
    # lock released here

    # Step 2: external call outside transaction
    payment = stripe.charge(order.total, order.payment_token)  # no DB lock held

    # Step 3: update status in short follow-up transaction
    with db.transaction():
        order.mark_paid(payment.id)
        db.save(order)
        db.add(OutboxEvent(OrderConfirmed.from_order(order)))   # outbox inside transaction
```

### Transaction Scope Checklist

- [ ] Transactions contain only DB operations — no external HTTP calls, no file I/O
- [ ] Transactions are as short as possible — data fetched before transaction, results processed after
- [ ] No user interaction (waiting for input) inside transaction scope
- [ ] Large batch operations processed in smaller chunks within separate transactions
- [ ] Events and audit records written within the same transaction as business data (outbox pattern)

---

## Step 3 — Isolation Levels

| Level | What it prevents | Allows | Use case |
|---|---|---|---|
| READ UNCOMMITTED | Nothing | Dirty reads | Almost never; avoid |
| READ COMMITTED | Dirty reads | Non-repeatable reads, phantoms | Default for most apps |
| REPEATABLE READ | Dirty reads, non-repeatable reads | Phantom reads | Financial calculations, inventory |
| SERIALIZABLE | All anomalies | Nothing | Strictest; highest contention; auditing |

```python
# Raise isolation level for sensitive financial operations
with db.transaction(isolation_level='REPEATABLE READ'):
    account = db.query(Account).filter_by(id=account_id).with_for_update().first()
    if account.balance < amount:
        raise InsufficientFundsError()
    account.balance -= amount
    db.save(account)
```

### Isolation Anomalies to Be Aware Of

```sql
-- Non-repeatable read: same row read twice, different values (another tx committed between reads)
-- Example: read balance = 100 → another tx withdraws 50 → read balance again = 50
-- Prevented by: REPEATABLE READ

-- Phantom read: range query returns different rows on second read
-- Example: COUNT of pending orders = 5 → another tx inserts order → COUNT = 6
-- Prevented by: SERIALIZABLE or explicit range lock

-- Lost update: two transactions read-modify-write the same row; second write overwrites first
-- Example: T1 reads balance=100, T2 reads balance=100; T1 writes 90, T2 writes 95 (T1's debit lost!)
-- Prevented by: SELECT FOR UPDATE (pessimistic) or version-check (optimistic)
```

---

## Step 4 — Optimistic vs Pessimistic Locking

### Pessimistic Locking (SELECT FOR UPDATE)

```sql
-- Lock the row immediately on read; block other writers until transaction commits
BEGIN;
SELECT * FROM accounts WHERE id = $1 FOR UPDATE;   -- other writers block here
-- ... validate, update ...
UPDATE accounts SET balance = $2 WHERE id = $1;
COMMIT;
```

```python
# SQLAlchemy
account = db.query(Account).filter_by(id=account_id).with_for_update().first()
```

**Use when:** contention is expected; cost of retrying is high; operation must not fail.

### Optimistic Locking (Version Column)

```python
# Schema: add version column to tracked entities
class Account(Base):
    id      = Column(String, primary_key=True)
    balance = Column(Decimal)
    version = Column(Integer, nullable=False, default=1)

# Read with version
account = db.query(Account).filter_by(id=account_id).first()
old_version = account.version

# Validate and update, including version check
rows_updated = db.query(Account).filter(
    Account.id == account_id,
    Account.version == old_version,  # fails if another transaction modified it
).update({
    'balance': new_balance,
    'version': old_version + 1,
})

if rows_updated == 0:
    raise OptimisticLockError("Account was modified by another transaction — retry")
```

**Use when:** contention is low; reads are far more frequent than writes; retries are acceptable.

| | Pessimistic | Optimistic |
|---|---|---|
| Strategy | Lock before read | Version check on write |
| Concurrency | Lower — readers block | Higher — no blocking |
| Retry needed? | No | Yes, on conflict |
| Best for | High contention, critical sections | Low contention, read-heavy |
| Risk | Deadlock | Lost update if no retry |

---

## Step 5 — Distributed Transactions (Multi-Service)

**Two-phase commit (2PC) is rarely correct in microservices — too fragile, too coupled.**

Use **Saga pattern** instead:

### Choreography Saga (event-driven)

Each service listens for events and publishes its own outcomes. On failure, compensating events trigger rollback:

```
OrderService       → publishes  OrderCreated
  InventoryService → hears OrderCreated → reserves stock → publishes StockReserved
  PaymentService   → hears StockReserved → charges card → publishes PaymentCaptured
  ShippingService  → hears PaymentCaptured → creates shipment → publishes ShipmentCreated

On failure:
  PaymentService fails → publishes PaymentFailed
  InventoryService → hears PaymentFailed → releases reserved stock → publishes StockReleased
  OrderService → hears StockReleased → marks order cancelled
```

### Orchestration Saga (central coordinator)

```python
class PlaceOrderSaga:
    """Central coordinator for the multi-step order placement saga."""

    def execute(self, command: PlaceOrderCommand) -> OrderId:
        order_id = self._create_order(command)
        try:
            self._reserve_stock(order_id, command.items)
        except StockUnavailableError:
            self._cancel_order(order_id)
            raise

        try:
            self._capture_payment(order_id, command.payment)
        except PaymentFailedError:
            self._release_stock(order_id, command.items)    # compensate
            self._cancel_order(order_id)
            raise

        self._confirm_order(order_id)
        return order_id
```

### Saga Design Rules

- [ ] Every step has a defined compensating action
- [ ] Compensating actions are idempotent (can be called multiple times safely)
- [ ] Saga state is persisted — recoverable if coordinator crashes mid-saga
- [ ] Final state is always consistent: either fully completed or fully compensated (no partial success)

---

## Step 6 — Common Transaction Anti-Patterns

```python
# ANTI-PATTERN 1: Transaction per-loop-iteration
for order in orders:
    with db.transaction():    # N separate transactions — N separate round trips
        order.process()

# FIX: batch in a single transaction (if all must succeed together)
with db.transaction():
    for order in orders:
        order.process()

# Or: chunk into bounded transactions
CHUNK_SIZE = 100
for i in range(0, len(orders), CHUNK_SIZE):
    with db.transaction():
        for order in orders[i:i+CHUNK_SIZE]:
            order.process()

# ANTI-PATTERN 2: Swallowing transaction errors
def save_user(user):
    try:
        with db.transaction():
            db.save(user)
    except Exception:
        pass   # silently discards the transaction — user not actually saved!

# FIX: never swallow transaction exceptions; let them propagate or handle explicitly

# ANTI-PATTERN 3: Reading outside transaction, writing inside — stale read
user = db.query(User).first()              # read outside transaction
with db.transaction():
    user.balance += amount                  # may write stale value!
    db.save(user)

# FIX: read inside transaction with locking
with db.transaction():
    user = db.query(User).filter_by(id=id).with_for_update().first()
    user.balance += amount
    db.save(user)
```

---

## Output Report

```
## Transactional Boundary Review: <operation>

### Critical
- PlaceOrder makes a Stripe HTTP call inside the database transaction
  Stripe API takes 2–8s; DB row locks held for entire duration; throughput drops by ~10x at load
  Fix: release transaction after reserving stock; call Stripe outside transaction; open new short transaction for payment record

- Balance debit reads account balance then writes new value without locking
  Lost update: two concurrent withdrawals both read balance=100; both write 50; net balance is 50 (debit lost)
  Fix: use SELECT FOR UPDATE to lock row; or optimistic locking with version column and retry on conflict

### High
- Saga has no compensating action for InventoryService step — if PaymentService fails, stock remains reserved
  Orders remain in limbo; inventory shows understated availability
  Fix: implement ReleaseReservation compensating action; trigger on PaymentFailed event

- Transaction spans 500-item batch — holds row locks for 45 seconds
  Long lock causes timeouts and deadlocks for concurrent requests on same rows
  Fix: process in chunks of 50 with separate transactions per chunk; accept eventual consistency

### Medium
- Isolation level READ COMMITTED allows non-repeatable read in balance check + debit flow
  Balance read at start of function may differ from balance at time of lock acquisition
  Fix: upgrade to REPEATABLE READ for financial operations; acquire row lock at first read

- Outbox event written in separate transaction after business transaction commits
  Service crash between commits orphans the business change without a published event
  Fix: write OutboxEvent in same transaction as business data (atomic)

### Low
- Exception in loop silently swallowed — failed items not retried or reported
  Fix: collect errors; re-raise after loop; caller decides retry strategy

### Passed
- Short-lived transactions — average duration < 50ms ✓
- Saga state persisted to DB — recoverable after coordinator restart ✓
- Compensating transactions defined and tested for all saga steps ✓
```

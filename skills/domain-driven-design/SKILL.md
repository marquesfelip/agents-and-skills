---
name: domain-driven-design
description: 'Domain modeling, aggregates, bounded contexts, and business-driven architecture decisions. Use when: domain-driven design, DDD, domain modeling, aggregate design, aggregate root, bounded context, ubiquitous language, value objects, domain events, context map, business-driven architecture, DDD tactical patterns, DDD strategic patterns, entity vs value object, domain service.'
argument-hint: 'Domain area or business problem to model — describe the business rules, entities involved, and consistency requirements'
---

# Domain-Driven Design Specialist

## When to Use
- Modeling a new business domain before writing code
- Reviewing existing code for DDD alignment (anemic models, missing invariants, wrong aggregate boundaries)
- Identifying bounded context boundaries within a monolith or set of services
- Establishing a ubiquitous language shared by the team and domain experts
- Designing aggregate consistency rules for complex business processes

---

## Step 1 — Establish Ubiquitous Language

The ubiquitous language is the shared vocabulary used by developers and domain experts — it must match the code.

### Language Audit

- [ ] Every entity name in code matches the term used by domain experts — no synonyms (`Customer` vs `Client` vs `User` in same context)
- [ ] Operations named as business verbs: `PlaceOrder`, `CancelReservation`, `ActivateSubscription` — not CRUD (`UpdateOrder`, `SetStatus`)
- [ ] Enum/constant values use business terms: `OrderStatus.AWAITING_PAYMENT` not `ORDER_STATUS_1`
- [ ] Comments, variable names, method parameters use ubiquitous language consistently

```python
# VIOLATION — technical naming, not ubiquitous language
def process_record(data: dict) -> bool:
    data['flag'] = 1
    ...

# CORRECT — domain language in code
def place_order(command: PlaceOrderCommand) -> Order:
    order = Order.create(customer=command.customer, items=command.items)
    ...
```

---

## Step 2 — Identify and Classify Domain Objects

### Entity vs Value Object Decision Table

| Characteristic | Entity | Value Object |
|---|---|---|
| Has unique identity? | Yes (ID-based equality) | No (value-based equality) |
| Mutates over time? | Yes | No (immutable) |
| Example | Order, Customer, Account | Money, Address, DateRange, Email, SKU |
| Equality | By ID | By all field values |

```python
# Value Object — immutable, equality by value
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

    def add(self, other: 'Money') -> 'Money':
        if self.currency != other.currency:
            raise CurrencyMismatchError()
        return Money(self.amount + other.amount, self.currency)

# Entity — has ID, mutable lifecycle
@dataclass
class Order:
    id: OrderId
    customer_id: CustomerId
    items: list[OrderItem]
    status: OrderStatus
    total: Money  # ← value object as attribute

    def add_item(self, item: OrderItem) -> None:
        if self.status != OrderStatus.DRAFT:
            raise OrderNotEditableError(self.id)
        self.items.append(item)
        self.total = self._calculate_total()
```

---

## Step 3 — Design Aggregate Boundaries

An aggregate is a cluster of domain objects treated as a single unit for data changes. Only the **aggregate root** is accessible from outside.

### Aggregate Design Rules

1. **Single aggregate root** — only one class is the entry point; external code never holds references to internal entities
2. **Transactional boundary** — one aggregate = one transaction; never span a transaction across aggregates
3. **Consistency enforced within** — all invariants of an aggregate enforced by the root
4. **Reference other aggregates by ID only** — never hold object references to external aggregates

```python
# CORRECT — Order aggregate with enforced invariants
class Order:          # ← Aggregate Root
    def __init__(self, id: OrderId, customer_id: CustomerId):
        self._id = id
        self._customer_id = customer_id   # reference to Customer by ID, not object
        self._items: list[OrderItem] = []   # internal entities — not exposed directly
        self._status = OrderStatus.DRAFT
        self._events: list[DomainEvent] = []

    def add_item(self, product_id: str, quantity: int, unit_price: Money) -> None:
        """Enforces: cannot add items once order is confirmed."""
        self._assert_status(OrderStatus.DRAFT)
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        self._items.append(OrderItem(product_id, quantity, unit_price))

    def confirm(self) -> None:
        """Enforces: order must have items; total must be positive."""
        self._assert_status(OrderStatus.DRAFT)
        if not self._items:
            raise EmptyOrderError()
        self._status = OrderStatus.CONFIRMED
        self._events.append(OrderConfirmed(order_id=self._id, total=self.total))

    def _assert_status(self, expected: OrderStatus) -> None:
        if self._status != expected:
            raise InvalidOrderStateError(expected, self._status)

    @property
    def pending_events(self) -> list[DomainEvent]:
        return list(self._events)

    def clear_events(self) -> None:
        self._events.clear()

# VIOLATION — exposing internal entity; caller can mutate without going through root
order._items.append(raw_item)     # bypasses aggregate invariants!
```

### Aggregate Size Heuristics

- **Too large**: aggregate locks too much data; long transactions; frequent conflicts
- **Too small**: requires multi-step cross-aggregate coordination for basic operations
- **Right size**: all fields in the aggregate change together as a result of one business operation

```
✓ Order aggregate: items, status, totals, discounts — all change together on a single order action
✗ CustomerWithAllOrders: Customer + List[Order] — orders change without touching Customer; too large
```

---

## Step 4 — Domain Events Design

Domain events capture what happened in the business domain — past tense, immutable.

```python
@dataclass(frozen=True)
class OrderConfirmed:
    """Raised when a customer confirms their order."""
    order_id: str
    customer_id: str
    total_amount: Decimal
    total_currency: str
    item_count: int
    occurred_at: datetime = field(default_factory=datetime.utcnow)

# Event naming rules:
# ✓ OrderConfirmed, PaymentReceived, ShipmentDispatched  (past tense, business language)
# ✗ OrderUpdated, StatusChanged  (too generic — what happened specifically?)
```

### Domain Event Checklist

- [ ] Events named in past tense, business vocabulary
- [ ] Events are immutable (frozen dataclass / record type)
- [ ] Events contain only the data consumers need — not the full aggregate state
- [ ] Aggregate collects events during operation; they are published AFTER the transaction commits
- [ ] Event field names use ubiquitous language (not column names or DB IDs)

---

## Step 5 — Map Bounded Contexts

A bounded context is an explicit boundary within which a domain model applies and has a consistent meaning.

### Context Mapping Exercise

For each entity that appears in multiple parts of the system, ask:
- Does the word mean the same thing in both contexts?
- Does it have the same attributes and lifecycle?

If no → they are in different bounded contexts and should each have their own model.

| Entity | Ordering context | Shipping context | Billing context |
|---|---|---|---|
| Order | Has items, status (DRAFT→CONFIRMED) | Has ship-to address, tracking | Has invoice reference, payment status |
| Customer | Has preferences, order history | Has delivery addresses | Has billing address, payment methods |
| Product | Has price, description | Has weight, dimensions | Has SKU, tax category |

Each bounded context has its own classes — do not share a single `Order` class across all three.

### Context Integration Patterns

| Relationship type | Pattern | Use when |
|---|---|---|
| Upstream / Downstream | Open Host Service (OHS) + Published Language | One context publishes stable API for others |
| Independent with sync | Anti-Corruption Layer (ACL) | Translating upstream model to local model |
| Event-based decoupling | Event-driven integration | Contexts don't need to know each other |
| Legacy integration | Conformist or ACL | Cannot change upstream model |

```python
# Anti-Corruption Layer — translates external Shipping model to local domain
class ShippingACL:
    def __init__(self, shipping_client: ShippingServiceClient):
        self._client = shipping_client

    def get_delivery_estimate(self, order_id: str, address: Address) -> DeliveryEstimate:
        # Translate our domain model → external API format
        ext_request = self._to_shipping_request(order_id, address)
        ext_response = self._client.estimate(ext_request)
        # Translate external response → our domain model
        return self._to_domain_estimate(ext_response)
```

---

## Step 6 — Domain Service vs Application Service

| Question | Answer → |
|---|---|
| Does it involve business logic that doesn't naturally belong to one entity? | Domain Service |
| Does it orchestrate multiple aggregates or infrastructure? | Application Service (Use-Case) |
| Does it perform a calculation on domain objects? | Domain Service |
| Does it send an email or call an external API? | Application Service |

```python
# Domain Service — pure business logic, no I/O
class PricingService:
    def calculate_total(
        self,
        items: list[OrderItem],
        discount: Discount,
        tax_rules: TaxRules,
    ) -> Money:
        subtotal = sum(item.subtotal for item in items)
        discounted = discount.apply(subtotal)
        return tax_rules.apply(discounted)

# Application Service — orchestration + I/O
class PlaceOrderUseCase:
    def execute(self, command: PlaceOrderCommand) -> OrderId:
        customer = self.customer_repo.get_or_raise(command.customer_id)
        pricing = self.pricing_service.calculate_total(command.items, command.discount, ...)
        order = Order.create(customer_id=customer.id, items=command.items, total=pricing)
        self.order_repo.save(order)
        self.event_bus.publish_all(order.pending_events)
        return order.id
```

---

## Output Report

```
## DDD Review: <domain area>

### Critical
- Order entity is an anemic model — no business logic, all rules in OrderService (procedural style)
  Invariants (cannot add items after confirmation) not enforced on the model; any caller bypasses them
  Fix: move business rules into Order aggregate; guard state transitions on the entity itself

- OrderItem list exposed as public mutable list on Order aggregate root
  External code appends items directly, bypassing quantity validation and status checks
  Fix: expose add_item() method on Order; make _items list private/readonly from outside

### High
- Single Order class used by ordering, shipping, and billing contexts with different meanings
  Fields added for each context collide; class grows unboundedly; contexts coupled
  Fix: define separate Order model per bounded context; integrate via events or ACL

- Payment and Shipment updated in same transaction as Order — cross-aggregate transaction
  Risk: long locks, high contention, distributed partial failure
  Fix: use domain events; Payment and Shipment react to OrderConfirmed asynchronously

### Medium
- Domain events not defined — state changes (order confirmed, payment received) inferred from polling
  Tight coupling between contexts; latency and missed events
  Fix: define OrderConfirmed, PaymentReceived events; emit from aggregate on state change

- "Customer" and "User" used interchangeably in code but have different meanings to domain experts
  Fix: align naming with domain — codify distinction in ubiquitous language glossary

### Low
- Money amounts stored as float — rounding errors in financial calculations
  Fix: use Decimal type; define Money value object with currency; enforce in domain layer

### Passed
- Value objects (Address, Money) defined as immutable dataclasses ✓
- Aggregate root enforces all status transitions via named methods ✓
- Bounded context for ordering isolated from other modules ✓
```

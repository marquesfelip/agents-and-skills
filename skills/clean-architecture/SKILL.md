---
name: clean-architecture
description: 'Layered architecture, dependency inversion, domain separation, and framework isolation in backend systems. Use when: clean architecture review, layered architecture, dependency inversion, domain separation, framework isolation, hexagonal architecture, ports and adapters, onion architecture, architecture review, SOLID architecture, framework coupling, architecture refactor, testable architecture, infrastructure abstraction.'
argument-hint: 'Service or module to review — or describe the architecture being designed and the language/framework in use'
---

# Clean Architecture Specialist

## When to Use
- Reviewing a codebase for architecture violations (framework leakage, inverted dependencies)
- Designing a new service and choosing the correct layer structure
- Refactoring a monolith toward a testable, framework-independent architecture
- Establishing architecture conventions for a team or project
- Evaluating whether domain logic can be tested without infrastructure

---

## Step 1 — Understand the Dependency Rule

The single most important rule in clean architecture:

> **Source code dependencies must point inward — toward higher-level policy. Outer layers depend on inner layers. Inner layers know nothing about outer layers.**

```
┌────────────────────────────────────────┐
│  Infrastructure / Frameworks           │  ← knows about domain (via interfaces)
│  ┌──────────────────────────────────┐  │
│  │  Interface Adapters (API, Repos) │  │  ← translates between layers
│  │  ┌────────────────────────────┐  │  │
│  │  │  Application (Use-Cases)   │  │  │  ← orchestrates domain + ports
│  │  │  ┌──────────────────────┐  │  │  │
│  │  │  │  Domain (Entities)   │  │  │  │  ← pure business rules; no external deps
│  │  │  └──────────────────────┘  │  │  │
│  │  └────────────────────────────┘  │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
Dependency arrows: all point INWARD ←
```

---

## Step 2 — Audit Each Layer for Responsibility Violations

### Domain Layer (Innermost)

**Allowed:** Entities, value objects, domain events, business rules, domain interfaces (ports).
**Forbidden:** ORM imports, HTTP libraries, external service SDK imports, environment variables, I/O of any kind.

```python
# VIOLATION — domain entity imports SQLAlchemy (framework leakage)
from sqlalchemy import Column, String
class User(Base):          # Base = SQLAlchemy DeclarativeBase
    id = Column(String)
    email = Column(String)
    def can_place_order(self): ...

# CORRECT — pure Python domain entity; zero external imports
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class User:
    id: str
    email: str
    status: str
    created_at: datetime

    def can_place_order(self) -> bool:
        return self.status == 'active'
```

### Application Layer (Use-Cases)

**Allowed:** Orchestration of domain objects and ports; use-case-specific input/output DTOs; domain errors.
**Forbidden:** HTTP status codes, SQL queries, framework annotations, direct instantiation of infrastructure classes.

```python
# VIOLATION — use-case knows about HTTP
from fastapi import HTTPException
def create_order(req):
    if not customer_exists(req.customer_id):
        raise HTTPException(status_code=404)   # HTTP concept in use-case layer!

# CORRECT — use-case raises domain errors; caller maps to HTTP
def create_order(req: CreateOrderInput) -> CreateOrderOutput:
    customer = self.customer_repo.get(req.customer_id)
    if not customer:
        raise CustomerNotFoundError(req.customer_id)   # domain error only
```

### Interface Adapters Layer

**Allowed:** Request/response translation, controller logic, repository implementations, serialization.
**Allowed here (forbidden deeper):** ORM models, HTTP framework decorators, SQL queries, JSON serialization.

```python
# Repository implementation lives here — domain sees only the interface (port)
class PostgresOrderRepository(OrderRepository):   # implements domain port
    def __init__(self, session: Session):
        self._session = session

    def get(self, order_id: str) -> Order | None:
        row = self._session.query(OrderRow).filter_by(id=order_id).first()
        return self._to_domain(row) if row else None

    def _to_domain(self, row: OrderRow) -> Order:
        return Order(id=row.id, customer_id=row.customer_id, ...)  # ORM → domain mapping
```

### Infrastructure / Frameworks Layer (Outermost)

**Allowed:** DI wiring, main entry points, framework bootstrapping, config loading.
**This layer is allowed to know everything** — it assembles the application.

---

## Step 3 — Port and Adapter Audit

Every external system interaction must go through a port (interface):

| External dependency | Port (interface) | Adapter (implementation) |
|---|---|---|
| Relational DB | `OrderRepository` | `PostgresOrderRepository` |
| Email service | `Mailer` | `SendGridMailer`, `SMTPMailer` |
| Message queue | `EventPublisher` | `RabbitMQPublisher`, `SQSPublisher` |
| External REST API | `PaymentGateway` | `StripePaymentGateway` |
| File storage | `FileStorage` | `S3FileStorage`, `LocalFileStorage` |
| Cache | `CacheStore` | `RedisCache`, `InMemoryCache` |

### Port Interface Design Rules

```python
# Port defined in domain or application layer
class OrderRepository(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def find_by_customer(self, customer_id: str) -> list[Order]: ...

# Rule: port methods use domain types only (Order, not OrderRow or dict)
# Rule: port interface has no infrastructure imports
```

**Checklist:**
- [ ] Every external system accessed via a port interface — no direct instantiation in use-cases
- [ ] Port interfaces defined in domain/application layer — NOT in infrastructure layer
- [ ] Each port has at least one test double (stub, mock, fake) enabling use-case testing without infrastructure
- [ ] Adapters map between infrastructure types and domain types — no infrastructure types leak into domain

---

## Step 4 — Dependency Inversion Validation

Dependency inversion = high-level modules define the interfaces they need; low-level modules implement them.

```python
# VIOLATION — use-case depends on concrete class (wrong direction)
from infrastructure.postgres import PostgresOrderRepository

class CreateOrderUseCase:
    def __init__(self):
        self.repo = PostgresOrderRepository()   # concrete dep in use-case!

# CORRECT — use-case depends on abstraction; concrete wired from outside
from domain.ports import OrderRepository

class CreateOrderUseCase:
    def __init__(self, repo: OrderRepository):  # depends on interface, not impl
        self.repo = repo

# Wiring happens in composition root (main / DI container) — outermost layer
use_case = CreateOrderUseCase(repo=PostgresOrderRepository(session))
```

### Import Direction Audit

```bash
# Python — check for inward violations (domain importing infrastructure)
grep -rn "from infrastructure\|import sqlalchemy\|import redis\|import boto3" src/domain/
# Any hit is an architecture violation

# Go
grep -rn "\"github.com/app/infrastructure\"" internal/domain/
```

---

## Step 5 — Framework Isolation Test

The ultimate test of clean architecture: **can you run all use-case and domain tests without starting any external service?**

```python
# Fast use-case test — no DB, no HTTP, no queue
def test_create_order_insufficient_stock():
    # Arrange: in-memory fake repo — no Postgres needed
    repo = InMemoryOrderRepository()
    product_repo = InMemoryProductRepository()
    product_repo.save(Product(id='p1', stock=0))

    use_case = CreateOrderUseCase(
        order_repo=repo,
        product_repo=product_repo,
        event_bus=FakeEventBus(),
    )

    # Act + Assert
    with pytest.raises(InsufficientStockError):
        use_case.execute(CreateOrderInput(product_id='p1', quantity=1))
```

### Framework Isolation Checklist

- [ ] Domain tests run with `pytest` / `go test` only — no Docker, no `testcontainers`
- [ ] Every port has an in-memory fake implementation for testing
- [ ] Use-case tests inject fakes — never real infrastructure classes
- [ ] Integration/adapter tests are separate and explicitly labeled

---

## Step 6 — Cross-Cutting Concerns Placement

| Concern | Where it lives | Anti-pattern |
|---|---|---|
| Logging | Domain raises events; middleware/adapter logs them | Domain calling `logger.info()` directly |
| Tracing | Middleware injects trace context; propagated via context param | Use-case creating spans directly |
| Validation | Handler validates shape; domain validates business rules | Framework validator annotations inside domain entities |
| Auth/authz | Middleware authenticates; use-case checks permissions (via domain role model) | DB query in domain to check user roles |
| Config | Assembled at composition root; passed as constructor args | Use-case reading from `os.environ` directly |

---

## Output Report

```
## Clean Architecture Review: <service>

### Critical
- Domain entity Order imports SQLAlchemy Column — framework leakage into innermost layer
  Domain tests require a running Postgres; business rules coupled to ORM technology
  Fix: define pure dataclass Order in domain layer; create OrderRow ORM model in infrastructure

- Use-case CreateOrderUseCase imports PostgresOrderRepository directly and calls its constructor
  Dependency direction is inverted; use-case controls its own dependencies
  Fix: define OrderRepository protocol in domain; inject concrete impl at composition root

### High
- Use-case layer raises HTTPException(404) for not-found cases
  HTTP concepts (status codes) inside application logic; layer boundary violated
  Fix: raise CustomerNotFoundError (domain error); map to 404 in exception handler at API layer

- No port interface for email delivery — SendGridMailer imported directly in use-case
  Cannot test order confirmation without calling real SendGrid API
  Fix: define Mailer protocol; inject SendGrid adapter; use FakeMailer in tests

### Medium
- In-memory test doubles for repositories not implemented — all tests require Postgres
  Slow feedback loop; local development requires Docker Compose running
  Fix: implement InMemoryOrderRepository; use in use-case unit tests

- Config (DATABASE_URL, API keys) read from os.environ inside PostgresOrderRepository
  Infrastructure class controls its own instantiation; cannot vary config in tests
  Fix: pass config as constructor arguments from composition root

### Low
- Repository query results (SQLAlchemy Row objects) returned directly from use-case methods
  Infrastructure type leaks through use-case return value
  Fix: map Row → domain entity in repository; return domain type from use-case

### Passed
- Domain layer has zero external imports ✓
- All use-case tests run without Docker (using in-memory fakes) ✓
- Composition root isolates all DI wiring in main.py ✓
```

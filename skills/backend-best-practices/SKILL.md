---
name: backend-best-practices
description: 'Backend service organization, clean boundaries, validation, error handling, and maintainable code structure. Use when: backend best practices, service organization, code structure review, validation layer, error handling design, clean backend, maintainable code, backend code review, service boundaries, input validation, backend refactor, layered backend, backend code quality, separation of concerns.'
argument-hint: 'Service, module, or file to review — or describe the backend feature being built and the language/framework in use'
---

# Backend Best Practices Specialist

## When to Use
- Reviewing a backend service for structural quality and maintainability
- Designing a new service or module and validating the approach before implementation
- Refactoring messy backend code toward clean, testable structure
- Onboarding a new service pattern to an existing codebase
- Code review of backend PRs for systematic structural issues

---

## Step 1 — Assess Service Organization

A well-organized backend separates concerns into distinct layers with clear responsibilities:

### Recommended Layer Structure

```
src/
  api/           # HTTP handlers / controllers — only decode request, call use-case, encode response
  usecases/      # Application logic — orchestrates domain and infrastructure
  domain/        # Business rules, entities, value objects — zero framework dependencies
  infrastructure/ # DB access, external HTTP clients, message queues, email adapters
  config/         # Environment, feature flags, DI wiring
  middleware/     # Cross-cutting: auth, logging, tracing, rate limiting
```

### Layer Responsibility Rules

| Layer | Allowed | Forbidden |
|---|---|---|
| API handler | Decode request, authenticate, validate input shape, call use-case, encode response | Business logic, DB queries, sending emails |
| Use-case | Orchestrate domain + infrastructure; enforce business rules | HTTP concepts (status codes, headers), raw SQL |
| Domain | Model invariants, value object validation, business events | Framework imports, DB drivers, HTTP clients |
| Infrastructure | DB queries, external API calls, queue producers | Business rules, use-case orchestration |

### Anti-Patterns to Flag

- [ ] Business logic inside HTTP handler functions
- [ ] Raw SQL queries called from use-case layer
- [ ] HTTP status codes decided inside domain/use-case layer
- [ ] Circular imports between modules at the same layer
- [ ] "God file" — single file over 500 lines with multiple distinct concerns

---

## Step 2 — Input Validation at System Boundaries

**Validate at the edge — never trust incoming data from any external source.**

### Validation Layers

```python
# Layer 1: Shape validation — is the structure correct? (HTTP handler)
class CreateOrderRequest(BaseModel):
    customer_id: str = Field(..., min_length=1, max_length=64)
    items: List[OrderItemRequest] = Field(..., min_items=1, max_items=100)
    shipping_address: AddressRequest

# Layer 2: Business rule validation — is it semantically valid? (use-case / domain)
def create_order(req: CreateOrderRequest) -> Order:
    customer = customer_repo.get_or_raise(req.customer_id)  # existence validated here
    if not customer.is_active():
        raise CustomerInactiveError(req.customer_id)
    for item in req.items:
        product = product_repo.get_or_raise(item.product_id)
        if product.stock < item.quantity:
            raise InsufficientStockError(item.product_id, product.stock, item.quantity)
```

### Validation Checklist

- [ ] All handler inputs validated before reaching use-case layer
- [ ] String lengths bounded (not just required)
- [ ] Numeric ranges validated (no negative quantities, no price = 0 where not allowed)
- [ ] Enum fields validated against allowed values
- [ ] IDs validated for format (UUID format check) before DB lookup
- [ ] List sizes bounded (prevent large payload abuse)
- [ ] Business rule validation (entity existence, state preconditions) in use-case/domain — not handler

---

## Step 3 — Error Handling Design

### Typed Error Hierarchy

```python
# Base domain errors — carry semantic meaning
class DomainError(Exception):
    pass

class NotFoundError(DomainError):
    def __init__(self, entity: str, id: str):
        self.entity = entity
        self.id = id
        super().__init__(f"{entity} not found: {id}")

class ConflictError(DomainError):
    pass

class ValidationError(DomainError):
    def __init__(self, field: str, reason: str):
        self.field = field
        self.reason = reason

# Mapping to HTTP at the handler boundary — only the API layer knows about HTTP
ERROR_TO_STATUS = {
    NotFoundError:   404,
    ConflictError:   409,
    ValidationError: 422,
    PermissionError: 403,
}

@app.exception_handler(DomainError)
def handle_domain_error(request, exc: DomainError):
    status = ERROR_TO_STATUS.get(type(exc), 500)
    return JSONResponse(status_code=status, content={"error": str(exc)})
```

### Error Handling Rules

- [ ] Domain errors carry semantic meaning — not HTTP status codes
- [ ] HTTP status code mapping at handler/middleware boundary only
- [ ] Errors logged with context (request_id, user_id, entity details) — never swallowed silently
- [ ] Infrastructure errors (DB down, timeout) caught and wrapped into domain errors at the repo boundary
- [ ] No `except Exception: pass` — every `except` either re-raises or logs and propagates
- [ ] Client errors (4xx) not logged as ERROR — use WARN or INFO to avoid alert fatigue
- [ ] Stack traces never returned in HTTP responses to clients

---

## Step 4 — Dependency Management and Injection

### Constructor Injection Pattern (All Languages)

```python
# BAD — hard dependency on concrete class; untestable
class OrderService:
    def __init__(self):
        self.repo = PostgresOrderRepository()     # cannot swap in tests
        self.mailer = SendGridMailer()             # calls real SendGrid in unit tests

# GOOD — depends on interfaces; injectable
class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,    # interface / protocol
        event_bus: EventBus,            # interface / protocol
        mailer: Mailer,                 # interface / protocol
    ):
        self.order_repo = order_repo
        self.event_bus = event_bus
        self.mailer = mailer
```

```go
// Go — interface injection
type OrderService struct {
    repo     OrderRepository  // interface
    events   EventPublisher   // interface
}

func NewOrderService(repo OrderRepository, events EventPublisher) *OrderService {
    return &OrderService{repo: repo, events: events}
}
```

### DI Checklist

- [ ] Use-cases receive their dependencies via constructor — no `new Concrete()` inside business logic
- [ ] All external dependencies (DB, HTTP client, queue) accessed via interfaces
- [ ] Dependency instantiation centralized in a single composition root (main / DI container)
- [ ] No global mutable state used as implicit dependency

---

## Step 5 — Module Boundaries and Coupling

### Coupling Audit

```bash
# Detect circular imports (Python)
pip install pydeps
pydeps src/ --max-bacon=3

# Detect circular imports (Go)
go vet ./...
# Or use: golang.org/x/tools/cmd/goimports
```

### Rules

- [ ] No import of a sibling module's internal (e.g., `orders` importing `users.internal.repository` directly)
- [ ] Inter-module communication via defined interfaces or shared DTOs, never shared DB tables
- [ ] Shared utilities (logging, tracing) in a dedicated `shared/` or `pkg/` layer — not reimplemented per module
- [ ] Module has one clear owner/purpose — rename it if you can't describe its purpose in one sentence

---

## Step 6 — Code Maintainability Standards

- [ ] Functions under ~30 lines; complex orchestration split into named helpers with descriptive names
- [ ] No magic numbers — constants named and defined at package scope
- [ ] No commented-out code committed — use version control history instead
- [ ] No deep nesting (>3 levels) — extract functions or use early returns / guard clauses
- [ ] No hard-coded environment-specific config (URLs, credentials, keys) in code — use environment variables
- [ ] Boolean parameters replaced with option structs or named constants (no `createUser(true, false, true)`)

---

## Output Report

```
## Backend Best Practices Review: <service>

### Critical
- Business logic (pricing calculation, discount rules) implemented inside HTTP handler functions
  Cannot unit test business rules without spinning up HTTP server
  Fix: extract PricingService use-case; handler only decodes request and calls service

- Exception `DatabaseError` propagated directly to API response exposing SQL error text
  Fix: catch at repository boundary; wrap as DomainError; log SQL detail server-side

### High
- Domain layer imports SQLAlchemy models directly — framework leaked into business rules
  Fix: define domain entities as plain dataclasses; map to/from ORM model at repository boundary

- Input validation absent on integer fields: quantity=-1, price=0 not rejected
  Fix: add Field(gt=0) constraints on Pydantic models for all numeric business fields

### Medium
- OrderService creates PostgresOrderRepository() internally — untestable without DB
  Fix: inject OrderRepository interface via constructor; wire concrete impl in composition root

- Error logging uses print() and missing request_id context
  Fix: use structured logger; include request_id, user_id, and operation name in all error logs

### Low
- ProcessOrder function is 210 lines with 4 levels of nesting
  Fix: extract ValidateStock(), ReserveItems(), PublishOrderCreated() helper methods

### Passed
- Typed error hierarchy with HTTP mapping at handler boundary ✓
- No circular imports between modules ✓
- All config from environment variables ✓
```

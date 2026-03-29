---
name: integration-testing
description: >
  Real dependency testing and system interaction validation strategies.
  Use when: integration testing, real dependency test, database integration test, repository test,
  HTTP client integration, message broker test, Testcontainers, Docker Compose test, embedded database,
  in-memory database, SQLite test, H2 test, fake server, WireMock, httptest, system interaction test,
  external service integration, third-party API test, integration test isolation, test data setup,
  integration test teardown, test fixture, integration test CI, integration test strategy.
argument-hint: >
  Describe the component and its real dependencies (e.g., "UserRepository with PostgreSQL and Redis cache"),
  and specify the infrastructure, isolation strategy, or specific integration boundary to validate.
---

# Integration Testing Specialist

## When to Use

Invoke this skill when you need to:
- Test a component against its real infrastructure dependencies (DB, cache, queue, external API)
- Design a strategy for integration test isolation and data management
- Configure Testcontainers, Docker Compose, or embedded alternatives
- Validate repository queries, HTTP clients, message consumers, and external adapters
- Prevent state pollution between tests in a shared environment

---

## Step 1 — Define Integration Test Scope

Integration tests verify that a component correctly interacts with its dependencies. They **do not** test business logic — that belongs in unit tests.

| Component Under Test | Real Dependency | What to Validate |
|---|---|---|
| Repository / DAO | PostgreSQL, MySQL, MongoDB | CRUD, query, pagination, constraint enforcement |
| HTTP Client / Adapter | External REST API | Serialization, auth headers, retries, timeouts |
| Cache Layer | Redis | Read/write, TTL, miss behavior |
| Message Consumer | Kafka, RabbitMQ, SQS | Message deserialization, processing, DLQ routing |
| File Storage Adapter | S3-compatible | Upload, download, signed URL |
| Email / SMS Gateway | SMTP, Twilio, SES | Request format, error handling |

Checklist:
- [ ] Scope is limited to infrastructure contract, not business behavior
- [ ] Each test covers one integration boundary
- [ ] Business logic is not re-tested here (already covered by unit tests)

---

## Step 2 — Choose Infrastructure Strategy

**Testcontainers (recommended):**
- Spins up real Docker containers per test suite
- Works in CI without pre-existing services
- Supported in: Java, Go, Python, .NET, Node.js, Rust

**Docker Compose:**
- Declarative multi-service setup
- Good for local dev; requires Docker Compose available in CI
- Less flexible per-test lifecycle management

**Embedded / In-Memory Alternatives:**
- SQLite in-memory (for SQL repositories — limited dialect compatibility)
- H2 (Java) — risks false positives if dialect differs from production DB
- **Prefer Testcontainers** over in-memory DBs for fidelity

**Fake Servers:**
- WireMock — HTTP API mocking with stubbed responses
- `net/http/httptest` (Go) — inline test server
- `nock` (Node.js) — HTTP request interceptor
- MockServer — Java/polyglot HTTP mock

Checklist:
- [ ] Real DB engine used (same version as production) — not a dialect substitute
- [ ] Testcontainers or Docker Compose preferred over in-memory DBs
- [ ] Fake HTTP servers used to stub external APIs with realistic response shapes
- [ ] Container reuse enabled across tests in a suite for performance

---

## Step 3 — Test Isolation and Data Management

Integration tests are stateful — isolation is critical:

**Strategy A: Transaction Rollback (recommended for DB tests)**
- Begin transaction before each test
- Run the test inside the transaction
- Roll back after the test — no data committed

**Strategy B: Truncate Between Tests**
- Truncate tables after each test
- Slower but works when transaction rollback is not possible (e.g., stored procedures with implicit commits)

**Strategy C: Dedicated Schema Per Test**
- Each test gets its own schema/database (parallel-safe)
- Higher overhead — use for parallel test suites

Checklist:
- [ ] No test relies on data left by a previous test
- [ ] Test setup (`beforeEach`) seeds its own required data
- [ ] Cleanup (`afterEach`) is guaranteed even when the test fails (use `defer` / `finally`)
- [ ] Tests can run in any order and in parallel without interference
- [ ] Unique identifiers used in seeded data to prevent key conflicts

---

## Step 4 — Repository / DAO Tests

Test the database adapter in isolation:

Checklist:
- [ ] `Create` → record exists, returned fields match
- [ ] `FindByID` → existing record found; non-existent ID returns `nil`/`null`/not-found error
- [ ] `List` → pagination, filters, sort order validated
- [ ] `Update` → changed fields persisted; unchanged fields unaffected
- [ ] `Delete` → record removed; cascades applied as expected
- [ ] Unique constraint violations return correct error type
- [ ] Nullable fields tested with both `nil`/`null` and populated values
- [ ] Migrations applied before suite; no schema drift between test and production

---

## Step 5 — HTTP Client / Adapter Tests

Test the outbound HTTP adapter against a fake server:

Checklist:
- [ ] Request URL, method, and headers match expected contract
- [ ] Request body serialized correctly (JSON, XML, form data)
- [ ] Successful response correctly deserialized into domain type
- [ ] 4xx/5xx responses mapped to appropriate error types
- [ ] Timeout triggers a client-side error (not hang)
- [ ] Retry logic re-attempts on transient 5xx or network error
- [ ] Circuit breaker opens after repeated failures (if applicable)
- [ ] Auth headers (Bearer, API key, Basic) sent correctly

---

## Step 6 — Message Consumer Tests

Test the message consumer with a real or containerized broker:

Checklist:
- [ ] Message deserialized from raw format (JSON, Avro, Protobuf) correctly
- [ ] Valid message triggers expected processing action
- [ ] Invalid message routed to DLQ, does not crash consumer
- [ ] Duplicate message handled idempotently
- [ ] Offset/ack committed only after successful processing
- [ ] Poison pill does not block queue — consumer continues after DLQ routing

---

## Step 7 — CI Configuration

```yaml
integration-tests:
  services:
    postgres:
      image: postgres:16
      env: { POSTGRES_PASSWORD: test }
    redis:
      image: redis:7
  steps:
    - run: go test ./internal/... -tags=integration -timeout=5m
```

Checklist:
- [ ] Integration tests tagged/grouped separately from unit tests
- [ ] CI service containers or Testcontainers used — not mocks of infrastructure
- [ ] Integration tests run after unit tests in CI pipeline
- [ ] Total integration suite completes in < 5 minutes
- [ ] Test output includes failure context (seeded data, query, response received)

---

## Output Report

### Critical
- Repository tests run against in-memory substitutes with different SQL dialect — false confidence
- No cleanup between tests — state pollution causes random failures
- Business logic re-tested in integration tests — duplication without extra coverage value

### High
- Production DB credentials used in integration tests — security risk
- No isolation strategy — tests fail when run in parallel or out of order
- HTTP client tests use manual `fetch` mocks instead of fake servers — drift from real behavior

### Medium
- Testcontainers not used in CI — tests require pre-provisioned services
- Missing test cases for 4xx/5xx responses and timeout scenarios
- Migration not applied before test suite — schema drift undetected

### Low
- Container startup repeated per test instead of per suite — slow test runs
- No test for null/optional fields in DB schema
- Test seed data uses hardcoded IDs that conflict with parallel runs

### Passed
- Real infrastructure used (same DB engine version as production)
- Tests isolated via transaction rollback or dedicated schema
- Suite runs in < 5 minutes and is stable across parallel runs
- HTTP clients tested against fake servers with realistic response shapes
- CI provisions all dependencies automatically via Testcontainers or service containers

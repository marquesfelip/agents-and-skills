---
name: backend-testing-strategy
description: >
  Unit, integration, and end-to-end testing strategies for backend systems.
  Use when: backend testing strategy, test pyramid backend, unit test design, integration test design,
  end-to-end test backend, e2e backend, backend test coverage, API test, service test, repository test,
  use case test, test doubles, mock strategy, stub strategy, spy, fake, test isolation, test boundary,
  backend test suite, test organization, pytest backend, Go testing, JUnit, NUnit, test architecture,
  backend TDD, test first, coverage report, test quality, backend QA.
argument-hint: >
  Describe the backend system or layer to be tested (e.g., "REST API service with PostgreSQL and Redis"),
  and specify the coverage goal, testing framework, or layer in focus.
---

# Backend Testing Strategy Specialist

## When to Use

Invoke this skill when you need to:
- Design or review a backend test suite from scratch
- Apply the test pyramid to a specific backend architecture
- Choose the right test type (unit / integration / E2E) for a given scenario
- Improve test isolation, coverage quality, or maintainability
- Select and configure testing frameworks and tools for a backend stack
- Define mocking and stubbing boundaries

---

## Step 1 — Map the System Under Test

Identify the layers and boundaries of the backend system:

- **Entry points**: HTTP handlers, gRPC endpoints, CLI commands, message consumers
- **Application layer**: use cases, services, orchestrators
- **Domain layer**: entities, value objects, domain services
- **Infrastructure layer**: repositories, DB adapters, external HTTP clients, caches, queues

Classify each layer using the test pyramid:
- **Unit tests** → domain logic, use cases with fakes, pure functions
- **Integration tests** → repositories against a real or containerized DB, HTTP clients against real endpoints or WireMock
- **E2E tests** → full request-to-response flows covering critical user paths

---

## Step 2 — Define Unit Test Boundaries

Unit tests must be:
- **Fast**: no I/O, no network, no file system
- **Isolated**: one unit under test; all collaborators replaced by test doubles
- **Deterministic**: same input → same output always

Checklist:
- [ ] Domain entities and value objects tested without any infrastructure
- [ ] Use cases tested with fakes (in-memory implementations) for repositories and external services
- [ ] No `time.Now()` / `Date.now()` calls without injection; use clock abstraction
- [ ] No environment variables read inside unit-tested code
- [ ] Test doubles categorized: **stub** (returns canned data), **mock** (verifies calls), **fake** (real behavior, fake implementation), **spy** (wraps real with observation)
- [ ] Table-driven or parameterized tests for edge cases and boundary values

---

## Step 3 — Define Integration Test Boundaries

Integration tests verify that real dependencies behave as expected:

- **Repository / DAO tests**: run against a real DB (Testcontainers, Docker Compose, SQLite in-memory)
  - Test CRUD operations, query correctness, migrations compatibility
  - Use per-test transaction rollback or isolated schemas to prevent state leakage
- **HTTP client tests**: run against a real server stub (WireMock, httptest, nock)
  - Verify serialization, headers, retries, timeout handling
- **Message broker tests**: run against embedded broker (Testcontainers Kafka, in-process RabbitMQ)
  - Test consumer processing, message format, DLQ behavior

Checklist:
- [ ] Containerized or embedded dependencies spun up in `TestMain` / setup fixture
- [ ] Each test uses isolated state (own schema, own topic, rollback after test)
- [ ] Migrations applied before suite, rolled back or dropped after suite
- [ ] No business logic tested here — infrastructure contract only
- [ ] Tests are slower but still run in CI (target < 5 min total)

---

## Step 4 — Define End-to-End Test Boundaries

E2E tests cover complete flows through the fully assembled stack:

- Deploy the service (in-process, Docker, or staging environment)
- Use real or seeded DB state
- Drive via the public API (HTTP, gRPC, CLI)
- Assert on final state (response body, DB record, emitted event)

Checklist:
- [ ] Cover only the **critical paths** (happy path + highest-risk failure paths)
- [ ] Seed data before each scenario; clean up after
- [ ] Do not test every edge case here — delegate to unit/integration layers
- [ ] E2E suite is kept small (< 20 scenarios) to stay maintainable
- [ ] Run in CI on merge to main or before release, not on every commit

---

## Step 5 — Test Organization and Naming

Enforce a consistent structure:

```
tests/
  unit/
    domain/
    usecases/
  integration/
    repositories/
    clients/
  e2e/
    scenarios/
```

Naming convention:
- Unit: `<Unit>_<Method>_<Scenario>_<ExpectedResult>`
- Integration: `<Adapter>_<Operation>_<Condition>`
- E2E: `<Feature>_<UserAction>_<ExpectedOutcome>`

Checklist:
- [ ] Tests are co-located with source or in a parallel `tests/` mirror
- [ ] Test file names match the unit under test
- [ ] No test uses `sleep()` for timing — use polling, channels, or event-driven assertions
- [ ] CI runs unit then integration then E2E — fail fast on cheaper tests first

---

## Step 6 — Coverage and Quality Gates

- Target **≥ 80% line/branch coverage** on domain and use case layers
- Do **not** target 100% coverage — untestable infra code is acceptable
- Use mutation testing to detect weak assertions (see `mutation-testing` skill)
- Block merges on coverage regression > 5%

Checklist:
- [ ] Coverage report generated on every CI run
- [ ] Coverage gate enforced at the domain + use case layers only
- [ ] Tests fail fast if a critical path is untested
- [ ] No test exists _only_ to inflate coverage (test must assert meaningful behavior)

---

## Output Report

### Critical
- No tests exist for core business logic or critical API paths
- Tests pass without asserting outcomes (empty `assert` blocks, always-true conditions)
- Test suite randomly fails due to shared mutable state between tests

### High
- Use cases tested with real DB instead of fakes (slow, coupled)
- No test isolation — tests depend on execution order
- Missing test coverage for error paths, boundary values, and failure modes

### Medium
- E2E suite covers edge cases that belong at the unit layer (inverted pyramid)
- Test names are generic (`test1`, `testSuccess`) and not self-documenting
- No coverage gate enforced in CI

### Low
- Test file organization is inconsistent
- Missing `TestMain` teardown leaving orphaned containers
- Test utilities are duplicated across test files instead of shared helpers

### Passed
- Test pyramid is balanced and each layer covers its correct scope
- Unit tests run in < 1 second, integration in < 5 min, E2E in < 15 min
- CI runs layers in order and fails fast on cheap tests
- Coverage gate is enforced on domain and use case layers

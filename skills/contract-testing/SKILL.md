---
name: contract-testing
description: >
  API contract validation between services and clients to prevent integration regressions.
  Use when: contract testing, API contract, consumer-driven contract, provider contract,
  Pact testing, Pact framework, consumer contract, provider verification, breaking change detection,
  API compatibility, schema validation contract, OpenAPI contract, service integration contract,
  microservice contract, contract test CI, CDC testing, consumer-driven development, Pact broker,
  contract drift, API versioning safety, backwards compatibility test, integration contract.
argument-hint: >
  Describe the services involved (e.g., "OrderService as consumer, PaymentService as provider")
  and specify the protocol (REST, gRPC, event/messages) and the testing tool or framework in use.
---

# Contract Testing Specialist

## When to Use

Invoke this skill when you need to:
- Prevent breaking changes between a service and its consumers
- Design consumer-driven contract (CDC) tests with Pact or equivalent
- Validate provider compliance against existing consumer contracts
- Replace fragile shared-environment integration tests with isolated contract tests
- Integrate contract verification into CI/CD pipelines
- Manage a Pact Broker or contract registry

---

## Step 1 — Identify Contracts

Map all inter-service communication boundaries:

| Source | Protocol | Provider | Contract Type |
|---|---|---|---|
| Frontend | REST/HTTP | Backend API | Consumer-driven |
| ServiceA | REST/HTTP | ServiceB | Consumer-driven |
| ServiceA | Async/Events | ServiceB | Message contract |
| API Gateway | REST/HTTP | Microservice | Provider-published |

Checklist:
- [ ] All service-to-service HTTP calls are mapped
- [ ] All consumed event/message formats are mapped
- [ ] Each boundary has a designated **consumer** and **provider** role
- [ ] Shared schemas or OpenAPI specs are identified

---

## Step 2 — Choose Contract Testing Model

**Consumer-Driven Contract (CDC) — Recommended for HTTP:**
- Consumer defines what it needs from the provider
- Provider verifies it satisfies all registered consumer contracts
- Tool: **Pact** (multi-language), **Spring Cloud Contract**

**Provider-Published Schema — For event/message protocols:**
- Provider publishes the schema (AsyncAPI, Avro, JSON Schema)
- Consumers validate they can deserialize the published schema
- Tool: **Pact Message**, **Confluent Schema Registry**, **AsyncAPI validator**

**OpenAPI Contract Validation — For REST:**
- Use OpenAPI spec as the contract source of truth
- Validate requests/responses against the spec in tests
- Tool: **Schemathesis**, **Dredd**, **OpenAPI Validator**

Checklist:
- [ ] Contract model chosen for each service boundary
- [ ] Pact chosen for consumer-driven flows
- [ ] AsyncAPI or Schema Registry used for event contracts
- [ ] OpenAPI spec kept synchronized with implementation

---

## Step 3 — Write Consumer Contract Tests (Pact)

Consumer-side test defines the **minimum interaction** it depends on:

```
// Example structure (language-agnostic pseudocode)
given: "an order with ID 123 exists"
upon receiving: GET /orders/123
with headers: { Accept: application/json }
will respond with:
  status: 200
  body: { id: 123, status: "PAID", total: 99.90 }
  // Only include fields the consumer actually uses
```

Checklist:
- [ ] Consumer test only declares fields it actually consumes — **no overfitting**
- [ ] Pact matchers used instead of exact values for dynamic fields (UUIDs, dates)
- [ ] Each consumer interaction has a `providerState` describing preconditions
- [ ] Pact file generated and published to Pact Broker after consumer CI passes
- [ ] Consumer tests run without calling the real provider

---

## Step 4 — Verify Provider Compliance

Provider verification runs the consumer contract against the real provider code:

1. Pull consumer contracts from Pact Broker
2. For each consumer interaction, set up `providerState` (seed data, mocks)
3. Replay the interaction against the actual provider
4. Assert response matches the consumer's contract

Checklist:
- [ ] Provider verification runs in CI on every provider code change
- [ ] `providerState` handlers are implemented for every consumer state
- [ ] Provider does **not** need to deploy consumer code to verify
- [ ] Verification results published back to Pact Broker
- [ ] Provider cannot merge if any consumer contract is broken

---

## Step 5 — Integrate with CI/CD (can-i-deploy)

Use **can-i-deploy** to gate deployments on contract compatibility:

```bash
pact-broker can-i-deploy \
  --pacticipant OrderService \
  --version $CI_COMMIT_SHA \
  --to-environment production
```

Checklist:
- [ ] `can-i-deploy` runs as a deployment gate for both consumer and provider
- [ ] Pact Broker (or PactFlow) deployed and used as central contract registry
- [ ] Webhook triggers provider verification when new consumer contract is published
- [ ] Contract versions tagged with environment they are deployed to
- [ ] Breaking changes blocked automatically before they reach shared environments

---

## Step 6 — Message and Event Contracts

For async communication (Kafka, RabbitMQ, SNS/SQS):

Checklist:
- [ ] Message contracts defined using Pact Message or AsyncAPI schema
- [ ] Producer tests verify the message it emits matches the contract
- [ ] Consumer tests verify it can process a message matching the contract
- [ ] Schema compatible with backward-compatible evolution rules:
  - Adding optional fields: allowed
  - Removing or renaming fields: breaking change → version bump
  - Changing field type: breaking change → version bump
- [ ] Schema Registry enforces compatibility mode (`BACKWARD`, `FULL`)

---

## Output Report

### Critical
- No contract tests exist between services that share APIs or events
- Provider merges code that breaks existing consumer contracts undetected
- Shared staging environment used as the only integration validation — not isolated

### High
- Consumer contracts over-specify exact values instead of using matchers — tests are brittle
- `providerState` handlers not implemented — contract verification cannot run
- Pact Broker not used — contracts stored in version control without publisher/verifier tracking

### Medium
- `can-i-deploy` not integrated into deployment pipeline — contract status not gating releases
- Message contracts not defined for async event communication
- Consumer tests call the real provider instead of using Pact mock

### Low
- Pact Broker webhook not configured — provider must be manually re-run after consumer publishes
- Contract descriptions are not human-readable — hard to review in Pact Broker UI

### Passed
- Consumer-driven contracts defined for all HTTP service boundaries
- Provider verification runs automatically on every provider code change
- `can-i-deploy` gates deployment for both consumer and provider
- Message contracts defined and validated against Schema Registry
- Pact Broker tracks contract status across environments

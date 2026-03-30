# agents-and-skills

A curated library of **AI agent personas** and **reusable skill modules** for software engineering. Agents define specialized roles with structured philosophies, responsibilities, and output formats. Skills are self-contained knowledge modules covering specific engineering domains — from security and multi-tenancy to billing, observability, and testing.

---

## 📁 Repository Structure

```
agents-and-skills/
├── agents/   # 16 specialized AI agent definitions
└── skills/   # 158 domain-specific skill modules
```

---

## 🤖 Agents

Each agent is a Markdown file that defines a specialized AI persona, including its role, philosophy, responsibilities, tools, and output format.

| Agent | Description |
|-------|-------------|
| **appsec-reviewer** | Security reviewer for pull requests. Detects secrets exposure, injection flaws, broken auth/authz, and misconfiguration. Produces structured reports with severity ratings. |
| **backend-architect** | Distributed systems architect. Advises on service boundaries, inter-service communication, consistency patterns, API design, and async/event-driven strategies. |
| **backend-engineer** | Hands-on backend implementation engineer. Writes services, APIs, integrations, queues, and background jobs following clean architecture principles. |
| **cicd-engineer** | CI/CD and release engineering specialist. Designs pipelines, automated QA gates, security scanning, branching strategies, and deployment workflows (rolling/blue-green/canary). |
| **code-reviewer** | Technical code reviewer focused on correctness, clarity, complexity, and maintainability. Uses a structured feedback severity system: `[MUST]`, `[SHOULD]`, `[CONSIDER]`, `[QUESTION]`, `[PRAISE]`. |
| **database-engineer** | Data layer expert. Handles schema design, migrations, query optimization, indexing, data integrity, and zero-downtime patterns. |
| **devops-engineer** | Infrastructure and deployment specialist. Writes IaC (Terraform), manages containers (Docker/Kubernetes), cloud resources, networking, and observability setup. |
| **frontend-architect** | Frontend systems architect. Designs rendering models (SPA/SSR/SSG/hybrid), design systems, state management, data fetching strategies, and team scalability. |
| **frontend-engineer** | Frontend implementation engineer. Builds components, integrates APIs, manages state, and ensures accessibility (a11y) following project patterns. |
| **observability-engineer** | Logs, metrics, and tracing specialist. Designs structured logging, metrics instrumentation, distributed tracing, SLI metrics, and alerting quality. |
| **performance-engineer** | Performance and optimization specialist. Identifies bottlenecks, designs caching, optimizes throughput/latency, and performs cost-vs-performance analysis. |
| **platform-architect** | SaaS platform architect. Designs multi-tenancy models, customer isolation, billing boundaries, global scalability, and cloud strategy. |
| **privacy-compliance-engineer** | LGPD/GDPR compliance specialist. Audits for privacy violations, implements data subject rights, retention enforcement, and consent management. |
| **security-engineer** | Comprehensive security engineer. Covers threat modeling (STRIDE), OWASP Top 10, attack surface review, data leak prevention, and infrastructure hardening. |
| **sre-engineer** | Site Reliability Engineer. Defines SLOs/SLIs, leads incident response, post-mortems, capacity planning, chaos engineering, and toil reduction. |
| **test-engineer** | Test strategy and implementation specialist. Designs unit, integration, contract, E2E, load, and security test suites with a focus on high-value behavioral tests. |

---

## 🧠 Skills

Each skill lives in its own directory and contains a `SKILL.md` file with step-by-step guidance, checklists, decision matrices, code examples, and architectural patterns.

### Authentication & Authorization
| Skill | Description |
|-------|-------------|
| `authentication-patterns` | Sessions, tokens, OAuth, passwordless, and MFA patterns |
| `authorization-models` | RBAC, ABAC, and policy-based access control |
| `jwt-security` | JWT validation, signing, expiry, and rotation best practices |
| `oauth-security` | Secure OAuth 2.0 flows and token handling |
| `secure-session-management` | Session lifecycle, fixation prevention, and secure cookies |
| `account-takeover-protection` | Detecting and preventing credential stuffing and ATO attacks |
| `brute-force-protection` | Rate limiting, lockout strategies, and CAPTCHA integration |
| `password-storage-security` | Hashing algorithms, salting, and upgrade strategies |
| `organization-membership-security` | Secure invite flows and membership validation |
| `secure-invite-system` | Token-based invite generation, expiry, and claim logic |
| `runtime-permission-evaluation` | Evaluating permissions at request time |

### Security & Vulnerability Prevention
| Skill | Description |
|-------|-------------|
| `owasp-top-10-defense` | Countermeasures for all OWASP Top 10 vulnerabilities |
| `xss-prevention` | Output encoding, CSP, and DOM-based XSS defenses |
| `csrf-protection` | Token-based and SameSite cookie CSRF mitigations |
| `sql-injection-prevention` | Parameterized queries, ORM safety, and input validation |
| `ssrf-protection` | Blocking server-side request forgery attacks |
| `rce-prevention` | Preventing remote code execution via untrusted input |
| `deserialization-security` | Safe deserialization patterns and type validation |
| `encryption-best-practices` | Key management, algorithm selection, and data-at-rest/transit encryption |
| `secrets-management` | Vault integration, secret rotation, and env variable hygiene |
| `secrets-exposure-detection` | Static scanning and runtime detection of leaked secrets |
| `api-security` | Authentication, authorization, and rate limiting for APIs |
| `api-key-management` | Lifecycle management, rotation, and scoping of API keys |
| `data-leak-prevention` | Detecting and blocking sensitive data in egress |
| `secure-logging` | Redacting PII and secrets from log output |
| `webhook-authentication-security` | Signature verification and replay protection for webhooks |

### Multi-tenancy & Tenant Isolation
| Skill | Description |
|-------|-------------|
| `multi-tenancy-design` | Silo, pool, and hybrid tenancy architecture patterns |
| `tenant-isolation` | Enforcing strict data boundaries between tenants |
| `single-database-tenant-isolation` | Row-level and schema-level isolation in shared databases |
| `row-level-security-patterns` | PostgreSQL RLS policies and enforcement patterns |
| `tenant-aware-query-design` | Ensuring all queries are scoped to the correct tenant |
| `tenant-context-propagation` | Passing tenant identity through service layers and async jobs |
| `tenant-aware-logging` | Including tenant context in all log entries |
| `tenant-aware-metrics` | Emitting per-tenant metrics for observability |
| `tenant-data-leak-prevention` | Preventing cross-tenant data exposure |
| `tenant-data-migration-strategy` | Moving data between tenancy models safely |
| `tenant-file-isolation` | Scoping file storage to individual tenants |
| `tenant-lifecycle-management` | Provisioning, suspension, reactivation, and deletion flows |
| `tenant-provisioning` | Automated onboarding and resource allocation for new tenants |
| `tenant-suspension-handling` | Gracefully suspending tenant access without data loss |
| `tenant-safe-background-jobs` | Ensuring background workers respect tenant boundaries |
| `cross-tenant-access-prevention` | Guards against accidental or malicious cross-tenant access |

### Billing & Subscriptions
| Skill | Description |
|-------|-------------|
| `stripe-subscription-architecture` | Full Stripe subscription model design |
| `stripe-webhook-processing` | Reliable Stripe webhook ingestion and processing |
| `subscription-lifecycle` | States and transitions in a subscription lifecycle |
| `subscription-lifecycle-management` | Managing upgrades, downgrades, cancellations, and renewals |
| `subscription-cancellation-strategy` | Cancellation UX, data retention, and access revocation |
| `billing-state-machine` | Modeling billing states and transitions reliably |
| `billing-event-idempotency` | Handling duplicate billing events safely |
| `billing-integration-safety` | Ensuring consistency between billing provider and internal state |
| `billing-observability` | Metrics and alerts for billing health |
| `billing-to-access-sync` | Keeping subscription state and feature access in sync |
| `payment-failure-recovery` | Dunning flows, retry logic, and grace periods |
| `grace-period-handling` | Allowing continued access during payment recovery windows |
| `invoice-reconciliation` | Matching invoices with payments and usage records |
| `trial-management` | Free trial lifecycle, conversion, and expiry handling |
| `usage-based-billing` | Metered billing, usage recording, and aggregation |
| `plan-capability-mapping` | Mapping subscription plans to feature sets |
| `plan-change-handling` | Proration, immediate vs. deferred plan changes |
| `plan-limiting-enforcement` | Enforcing feature and usage limits per plan |
| `entitlement-system-design` | Designing a flexible entitlement engine |
| `entitlement-synchronization` | Keeping entitlements in sync with billing state |
| `feature-access-resolution` | Resolving whether a user/tenant has access to a feature |
| `quota-management` | Defining and tracking resource quotas per tenant/plan |
| `quota-enforcement-runtime` | Enforcing quotas at request time without performance degradation |
| `reactivation-flow` | Re-enabling access after cancellation or payment recovery |

### Architecture & Design Patterns
| Skill | Description |
|-------|-------------|
| `api-design-guidelines` | REST/gRPC API design conventions and best practices |
| `clean-architecture` | Layered architecture, dependency inversion, and separation of concerns |
| `domain-driven-design` | Bounded contexts, aggregates, entities, and value objects |
| `event-driven-architecture` | Event sourcing, pub/sub, and eventual consistency patterns |
| `event-driven-saas-patterns` | Event-driven patterns specific to SaaS products |
| `async-processing-patterns` | Task queues, workers, and async job orchestration |
| `distributed-workflows` | Orchestrating multi-step distributed processes |
| `distributed-locking` | Preventing concurrent access conflicts across services |
| `idempotency-patterns` | Making operations safe to retry without side effects |
| `data-consistency-patterns` | Saga, outbox, and eventual consistency strategies |
| `transactional-boundaries` | Defining safe transaction scopes across services |
| `feature-flag-architecture` | Flag management, rollout strategies, and kill switches |
| `event-ordering-strategy` | Handling out-of-order events in distributed systems |
| `event-replay-strategy` | Replaying events for recovery and backfilling |
| `sla-slo-sli-design` | Defining reliability targets and error budgets |

### Database
| Skill | Description |
|-------|-------------|
| `schema-design` | Normalization, relationships, and modeling best practices |
| `indexing-strategies` | Choosing and managing indexes for performance |
| `query-optimization` | Analyzing and rewriting slow queries |
| `database-performance` | Connection pooling, read replicas, and tuning |
| `migration-safety` | Safe schema migration strategies for live systems |
| `zero-downtime-migrations` | Applying schema changes without service interruptions |
| `replication-strategies` | Primary/replica setups, failover, and consistency trade-offs |
| `soft-delete-strategy` | Implementing logical deletion with retention and recovery |

### Observability
| Skill | Description |
|-------|-------------|
| `structured-logging` | JSON logging, log levels, and context enrichment |
| `metrics-design` | Choosing and naming metrics for dashboards and alerts |
| `distributed-tracing` | Propagating trace context across services |
| `alerting-strategy` | Alert thresholds, on-call routing, and reducing noise |
| `audit-trail-design` | Immutable audit logs for compliance and debugging |
| `auditability-patterns` | Designing systems to be auditable by default |
| `incident-response-playbooks` | Runbooks and response procedures for production incidents |

### Performance & Reliability
| Skill | Description |
|-------|-------------|
| `caching-strategies` | Cache placement, invalidation, and consistency trade-offs |
| `rate-limiting-strategies` | Token bucket, sliding window, and distributed rate limiting |
| `load-shedding` | Gracefully degrading under overload |
| `backpressure-patterns` | Propagating and handling backpressure in pipelines |
| `ddos-mitigation` | Protecting services from volumetric and application-layer attacks |
| `api-performance` | Reducing API latency through batching, pagination, and caching |
| `frontend-performance` | Core Web Vitals, bundle optimization, and lazy loading |
| `database-performance` | Query and connection tuning for high-throughput workloads |
| `concurrency-safety` | Avoiding race conditions and deadlocks in concurrent systems |
| `avoid-race-conditions` | Patterns for safe concurrent state modification |
| `load-testing` | Load test design, tooling, and result analysis |
| `chaos-testing` | Fault injection and resilience validation |
| `cost-aware-architecture` | Designing systems with cloud cost efficiency in mind |
| `compute-cost-guardrails` | Preventing runaway compute spend |
| `storage-cost-control` | Lifecycle policies and tiering for object storage cost management |
| `storage-cost-optimization` | Compression, deduplication, and access pattern optimization |

### CI/CD & DevOps
| Skill | Description |
|-------|-------------|
| `pipeline-design` | CI/CD pipeline structure, stages, and best practices |
| `preview-environments` | Ephemeral per-PR environments for integration testing |
| `release-management` | Versioning, changelogs, and release coordination |
| `automated-versioning` | Semantic versioning and automated release tagging |
| `artifact-signing` | Signing build artifacts for supply chain integrity |
| `sbom-generation` | Generating Software Bill of Materials for compliance |
| `dependency-security-scanning` | Automated CVE scanning for third-party dependencies |

### Privacy & Compliance
| Skill | Description |
|-------|-------------|
| `pii-handling` | Classifying, storing, and processing personally identifiable information |
| `data-retention-policies` | Defining and enforcing data retention and deletion schedules |
| `data-export-compliance` | Implementing data portability and export requests (GDPR/LGPD) |

### Storage & File Handling
| Skill | Description |
|-------|-------------|
| `object-storage-architecture` | Designing scalable object storage systems |
| `s3-compatible-storage-patterns` | Patterns for working with S3-compatible APIs |
| `object-lifecycle-management` | Automating object expiry, archival, and deletion |
| `private-object-access-control` | Scoping read access to authorized users only |
| `signed-url-management` | Generating and validating pre-signed URLs |
| `secure-file-upload` | Validating, scanning, and storing user-uploaded files |
| `virus-scan-pipeline` | Integrating antivirus scanning into upload workflows |
| `media-processing-pipeline` | Transcoding, resizing, and processing media at upload |
| `backup-and-restore` | Designing backup schedules and tested restore procedures |

### Background Jobs & Async Processing
| Skill | Description |
|-------|-------------|
| `background-job-safety` | Ensuring background jobs are safe, idempotent, and observable |
| `job-idempotency` | Designing jobs that can be retried without side effects |
| `dead-letter-handling` | Processing failed messages and alerting on poison pills |
| `webhook-reliability` | Reliable webhook delivery with retries and dead-letter queues |

### Testing
| Skill | Description |
|-------|-------------|
| `backend-testing-strategy` | Unit, integration, and contract testing for backend services |
| `frontend-testing-strategy` | Component, E2E, and accessibility testing for frontends |
| `integration-testing` | Testing service interactions and external dependencies |
| `contract-testing` | Consumer-driven contract tests for API compatibility |
| `mutation-testing` | Measuring test suite effectiveness with mutation analysis |
| `security-testing` | SAST, DAST, and penetration testing integration |

### User & Account Flows
| Skill | Description |
|-------|-------------|
| `onboarding-flows` | Designing user and tenant onboarding experiences |
| `onboarding-state-machine` | State machine design for multi-step onboarding |
| `activation-flow-design` | Guiding users to their first "aha" moment |
| `customer-offboarding` | Graceful account deletion, data export, and churn handling |
| `reactivation-flow` | Re-engaging lapsed users and restoring access |

### Analytics & Usage
| Skill | Description |
|-------|-------------|
| `usage-analytics-architecture` | Capturing and aggregating usage events for analytics |
| `usage-anomaly-detection` | Detecting abnormal usage patterns and triggering alerts |

### Abuse Prevention
| Skill | Description |
|-------|-------------|
| `abuse-detection` | Identifying abusive behavior patterns across users and tenants |
| `abuse-prevention-system` | Building a comprehensive anti-abuse system |
| `bot-protection` | Detecting and blocking automated bot traffic |

### Go Language
| Skill | Description |
|-------|-------------|
| `go-best-practices` | Idiomatic Go patterns, error handling, and project structure |
| `go-unit-tests` | Writing effective unit tests in Go |

### Tooling
| Skill | Description |
|-------|-------------|
| `skill-creator` | A meta-skill for generating new skill modules in this format |

---

## 📄 License

MIT License — Copyright © 2026 Felipe Marques

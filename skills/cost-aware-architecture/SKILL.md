---
name: cost-aware-architecture
description: 'Architecture decisions guided by infrastructure cost for SaaS products. Use when: cost-aware architecture, infrastructure cost optimization, cost-driven design, cloud cost architecture, cost tradeoff, architecture cost review, compute cost, storage cost, egress cost, database cost, cache cost, queue cost, cloud spend architecture, cost per tenant, unit economics architecture, cost vs performance tradeoff, right-sizing, cloud resource optimization, serverless vs container cost, cost allocation, cost tagging, cost attribution, AWS cost optimization, GCP cost optimization, Azure cost optimization, FinOps architecture.'
argument-hint: 'Describe the cloud provider (AWS, GCP, Azure), the primary cost drivers you want to address (compute, storage, egress, database, queues), the current monthly spend range, and whether cost optimization is urgent (runaway spend) or proactive (architecture review).'
---

# Cost-Aware Architecture

## When to Use

Invoke this skill when you need to:
- Evaluate architectural choices with explicit cost tradeoffs (e.g., Redis vs PostgreSQL for caching, SQS vs Kafka, Lambda vs ECS)
- Identify the top cost drivers in a SaaS infrastructure and target the highest-impact optimizations first
- Design cost allocation: tag resources by tenant, feature, or team so you know what costs what
- Apply right-sizing: detect over-provisioned compute and database instances
- Choose the cheapest data path for a given SLA — cache hits, CDN, read replicas
- Implement cost guardrails: alerts and circuit breakers before runaway spend occurs

---

## Step 1 — Identify Cost Drivers (Pareto Analysis)

```
Cost Driver           Typical % of cloud bill    Optimization lever
────────────────────  ─────────────────────────  ─────────────────────────────────────────
Compute (EC2/GKE)     30–50%                     Right-size, Spot/Preemptible, autoscale
Database (RDS/Cloud)  20–35%                     Connection pooling, read replicas, cache
Egress (data transfer) 10–25%                   CDN, same-region routing, compression
Object Storage (S3)   5–15%                      Lifecycle rules, intelligent tiering
Message queues        2–10%                      Batch processing, avoid fan-out excess
Logging/Monitoring    3–8%                       Log sampling, metric cardinality control

Rule: Fix the top 2 cost drivers before touching anything else.
```

---

## Step 2 — Cost Tagging Strategy

```go
// All cloud resources must carry cost allocation tags at creation time
// Without tags, spend cannot be attributed to tenant, feature, or team

// AWS example: tag every resource at creation
type ResourceTags struct {
    Environment string // "production" | "staging" | "dev"
    Service     string // "api" | "worker" | "db" | "cache"
    Feature     string // "billing" | "analytics" | "core" | "notifications"
    Team        string // "platform" | "product" | "data"
    CostCenter  string // "saas-core" | "data-infra"
}

// Tag convention enforcement in Terraform:
// default_tags in provider block ensures all resources inherit base tags
// resource-specific tags override where needed

// For multi-tenant cost attribution:
// Do NOT tag individual resources per tenant (too many resources)
// Instead: measure usage per tenant in application and multiply by unit cost
// unit_cost = total_service_cost / total_service_units (requests, GB, transactions)
// tenant_cost = tenant_units * unit_cost
```

---

## Step 3 — Architectural Cost Tradeoff Reference

```
Decision: Caching layer
  Option A: Redis (ElastiCache)
    Cost: ~$0.04/GB/hr (r6g.large ≈ $100/mo)
    Benefit: Sub-ms reads, reduces DB load by 80%+
    Use when: hot read path, frequently-read shared data
  Option B: In-process cache (Go sync.Map / bigcache)
    Cost: $0 additional — uses existing compute RAM
    Benefit: Zero network hop
    Limitation: Not shared across instances — stale on deploy
    Use when: immutable reference data, per-instance config
  Option C: PostgreSQL with UNLOGGED table
    Cost: $0 additional
    Use when: write-through cache with persistence, small dataset
  Verdict: Use in-process cache first. Redis only when data is shared across instances.

Decision: Message broker
  Option A: SQS
    Cost: $0.40 per million requests
    Use when: < 1M messages/day, simple fan-out, managed retries needed
  Option B: Kafka (MSK)
    Cost: $500–$2000+/mo for managed cluster
    Benefit: Log retention, replay, ordering by partition
    Use when: > 10M events/day, replay required, ordering critical
  Option C: PostgreSQL LISTEN/NOTIFY + outbox
    Cost: $0 additional
    Use when: < 100k events/day, reliable outbox processing acceptable
  Verdict: Start with PostgreSQL outbox. Migrate to SQS when outbox becomes bottleneck.

Decision: Compute model
  Option A: Always-on containers (ECS/GKE)
    Cost: Constant — pay for reserved capacity
    Use when: p99 latency < 100ms required, stateful, > 100 req/min baseline
  Option B: Serverless (Lambda/Cloud Run)
    Cost: Pay per invocation — near zero at low traffic
    Use when: bursty, event-driven, < 10 concurrent executions, cold start acceptable
  Option C: Spot/Preemptible instances
    Cost: 60–90% cheaper than on-demand
    Use when: stateless workers, batch jobs, fault-tolerant workloads
  Verdict: API on always-on (Spot where possible). Async workers on Spot/Preemptible.
```

---

## Step 4 — Unit Economics: Cost Per Tenant

```go
// CostPerTenantCalculator estimates infrastructure cost attribution per tenant
// Uses usage signals from the analytics pipeline

type TenantCostComponents struct {
    TenantID      string
    Period        time.Time
    ComputeCents  int64 // pro-rated by API request share
    StorageCents  int64 // actual S3 bytes * unit price
    DatabaseCents int64 // pro-rated by query count share
    EgressCents   int64 // actual bytes transferred
    TotalCents    int64
}

// Inputs: from usage analytics + billing snapshot
// compute_cost_cents * (tenant_requests / total_requests)
func (c *CostCalculator) ComputeTenantShare(ctx context.Context, tenantID string, month time.Time) (TenantCostComponents, error) {
    totalCost, _ := c.cloudBilling.GetMonthlyCost(ctx, month)
    tenantUsage, _ := c.usageRepo.GetTenantUsage(ctx, tenantID, month)
    totalUsage, _ := c.usageRepo.GetTotalUsage(ctx, month)

    share := float64(tenantUsage.RequestCount) / float64(totalUsage.RequestCount)
    computeShare := int64(float64(totalCost.ComputeCents) * share)
    storageActual := tenantUsage.StorageGB * totalCost.StorageCentsPerGB

    return TenantCostComponents{
        TenantID:     tenantID,
        Period:       month,
        ComputeCents: computeShare,
        StorageCents: storageActual,
        TotalCents:   computeShare + storageActual,
    }, nil
}
```

---

## Step 5 — Cost Alerts and Guardrails

```go
// DailyCostAlert fires when projected monthly spend exceeds threshold
func (m *CostMonitor) CheckDailySpend(ctx context.Context) {
    daily, err := m.cloudBilling.GetTodaySpend(ctx)
    if err != nil {
        slog.Error("cost check failed", "err", err)
        return
    }

    projectedMonthly := daily * 30
    if projectedMonthly > m.MonthlyBudgetCents {
        slog.Error("COST ALERT: projected monthly spend exceeds budget",
            "daily_cents", daily,
            "projected_monthly_cents", projectedMonthly,
            "budget_cents", m.MonthlyBudgetCents,
        )
        m.alerts.FireCostAlert(ctx, CostAlert{
            DailyCents:    daily,
            ProjectedCents: projectedMonthly,
            BudgetCents:   m.MonthlyBudgetCents,
        })
    }
}
```

---

## Cost-Aware Design Principles

| Principle | Implementation |
|---|---|
| **Measure before optimizing** | Tag all resources; run 1 month to establish baseline before tuning |
| **Cheapest data path first** | Check in-process cache → Redis → read replica → primary DB (in that order) |
| **Batch beats per-request** | Aggregate writes; flush in batches every N seconds instead of on each event |
| **Egress is expensive** | Keep compute and storage in the same region; use CDN for static assets |
| **Idle capacity costs money** | Autoscale down after traffic; use Spot for workers |
| **Logs and metrics have cost** | Sample DEBUG logs; limit Prometheus label cardinality; use log retention TTLs |
| **Right-size before scaling out** | A larger instance is often cheaper than two smaller ones |

---

## Quality Checks

- [ ] All cloud resources carry `environment`, `service`, `feature`, and `team` tags
- [ ] Top 2 cost drivers identified from billing dashboard before any optimization work starts
- [ ] Unit economics (cost per tenant) calculated at least monthly
- [ ] Daily cost alert defined with threshold at 110% of expected daily average
- [ ] Architectural decisions document the cost rationale (not just technical rationale)
- [ ] Egress costs reviewed — storage and compute in same region; no cross-region reads on hot paths
- [ ] Log retention policy set: 30 days hot, 90 days warm, archive or delete after 1 year

## After Completion

- Use **`storage-cost-optimization`** for S3/GCS lifecycle rules and tiering decisions
- Use **`compute-cost-guardrails`** for autoscaling, Spot usage, and compute budget enforcement
- Use **`caching-strategies`** to implement the cheapest data path (in-process → Redis → DB)
- Use **`usage-analytics-architecture`** as the source of per-tenant usage data for cost attribution

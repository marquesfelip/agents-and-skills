---
name: compute-cost-guardrails
description: 'Compute usage protection mechanisms for SaaS products. Use when: compute cost guardrails, compute cost protection, autoscaling cost control, Spot instance, Preemptible VM, compute budget alert, EC2 cost guardrail, GKE cost control, ECS cost optimization, compute right-sizing, idle compute, Lambda cost control, serverless budget, container compute cost, worker pool cost, compute budget, CPU budget, compute spend alert, runaway compute, autoscale cap, max instance count, compute quota, resource limit compute, CPU throttle, worker concurrency limit.'
argument-hint: 'Describe the compute platform (EC2, ECS, GKE, Lambda, Cloud Run), the workload type (always-on API, bursty async workers, scheduled jobs), whether Spot/Preemptible instances are feasible, and the monthly compute budget you are targeting.'
---

# Compute Cost Guardrails

## When to Use

Invoke this skill when you need to:
- Cap autoscaling groups to prevent runaway cost from traffic spikes or infinite loops
- Use Spot/Preemptible instances for stateless async workers to cut compute cost 60–90%
- Right-size over-provisioned containers or EC2 instances using real CPU/memory utilization data
- Implement worker concurrency limits to prevent a single tenant from consuming all compute
- Set up budget alerts that fire before the month's compute spend gets out of control
- Design job queues so background work cannot grow unboundedly and consume all capacity

---

## Step 1 — Autoscaling with Cost Cap

```go
// Application-level: cap batch worker pool to bound compute consumption
// (Cloud-level: set max instance count in ASG / GKE HPA / ECS Service)

type WorkerPool struct {
    maxWorkers int        // hard upper bound — do not exceed this regardless of queue depth
    sem        chan struct{}
    wg         sync.WaitGroup
}

func NewWorkerPool(maxWorkers int) *WorkerPool {
    return &WorkerPool{
        maxWorkers: maxWorkers,
        sem:        make(chan struct{}, maxWorkers),
    }
}

func (p *WorkerPool) Submit(ctx context.Context, task func(context.Context)) error {
    select {
    case p.sem <- struct{}{}: // acquire slot
        p.wg.Add(1)
        go func() {
            defer p.wg.Done()
            defer func() { <-p.sem }() // release slot
            task(ctx)
        }()
        return nil
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Pool full — apply backpressure: caller should slow down enqueue
        return ErrPoolFull
    }
}

func (p *WorkerPool) Wait() { p.wg.Wait() }
```

---

## Step 2 — Spot/Preemptible Instance Pattern for Workers

```go
// Spot instances can be terminated with 2-minute notice (AWS)
// Design async workers to handle termination gracefully

type SpotWorker struct {
    queue  JobQueue
    pool   *WorkerPool
    done   chan struct{}
}

func (w *SpotWorker) Run(ctx context.Context) {
    // Listen for spot termination signal (AWS IMDS endpoint)
    go w.watchSpotTermination(ctx)

    for {
        select {
        case <-w.done:
            slog.Info("spot termination signal — draining worker")
            w.pool.Wait() // finish in-flight jobs
            return
        case <-ctx.Done():
            return
        default:
            job, err := w.queue.Claim(ctx)
            if err != nil {
                time.Sleep(1 * time.Second)
                continue
            }
            w.pool.Submit(ctx, func(ctx context.Context) {
                w.process(ctx, job)
            })
        }
    }
}

// AWS: poll IMDS for spot termination notice (appears 2 minutes before termination)
func (w *SpotWorker) watchSpotTermination(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            resp, err := http.Get("http://169.254.169.254/latest/meta-data/spot/termination-time")
            if err == nil && resp.StatusCode == 200 {
                close(w.done) // signal graceful drain
                return
            }
        }
    }
}
```

---

## Step 3 — Right-Sizing: Detect Over-Provisioned Containers

```go
// Emit actual CPU and memory usage metrics for right-sizing analysis
// Compare reported usage against requested limits every 5 minutes

type ResourceUsageReporter struct {
    interval time.Duration
}

func (r *ResourceUsageReporter) Run(ctx context.Context) {
    ticker := time.NewTicker(r.interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // Read from /sys/fs/cgroup inside container
            cpuUsage := readCgroupCPU()
            memUsage := readCgroupMemory()

            slog.Info("resource utilization",
                "cpu_millicores_used", cpuUsage,
                "cpu_millicores_requested", env.CPULimit(),
                "memory_mb_used", memUsage/1024/1024,
                "memory_mb_requested", env.MemLimit()/1024/1024,
                "cpu_utilization_pct", cpuUsage*100/env.CPULimit(),
                "mem_utilization_pct", memUsage*100/env.MemLimit(),
            )

            // Alert if under-utilized — candidate for right-sizing
            if cpuUsage*100/env.CPULimit() < 20 {
                slog.Warn("CPU under-utilized — consider reducing CPU request",
                    "cpu_utilization_pct", cpuUsage*100/env.CPULimit(),
                    "pod", os.Getenv("POD_NAME"),
                )
            }
        }
    }
}
```

---

## Step 4 — Per-Tenant Compute Fairness (Concurrency Limit)

```go
// Prevent one tenant from monopolizing the worker pool
// Each tenant gets a concurrency slot limit

type TenantConcurrencyLimiter struct {
    mu       sync.Mutex
    active   map[string]int
    maxPerTenant int
}

func (l *TenantConcurrencyLimiter) Acquire(tenantID string) error {
    l.mu.Lock()
    defer l.mu.Unlock()
    if l.active[tenantID] >= l.maxPerTenant {
        return ErrTenantConcurrencyExceeded
    }
    l.active[tenantID]++
    return nil
}

func (l *TenantConcurrencyLimiter) Release(tenantID string) {
    l.mu.Lock()
    defer l.mu.Unlock()
    if l.active[tenantID] > 0 {
        l.active[tenantID]--
    }
}

// Usage in worker — wraps job execution
func (w *Worker) processWithFairness(ctx context.Context, job Job) error {
    if err := w.limiter.Acquire(job.TenantID); err != nil {
        // Re-enqueue for later — this tenant is at their concurrency cap
        return w.queue.Requeue(ctx, job, 5*time.Second)
    }
    defer w.limiter.Release(job.TenantID)
    return w.process(ctx, job)
}
```

---

## Step 5 — Compute Budget Alert

```yaml
# AWS Budgets (Terraform equivalent)
# Alert when compute spend exceeds 80% of monthly budget

resource "aws_budgets_budget" "compute" {
  name         = "compute-monthly"
  budget_type  = "COST"
  limit_amount = "2000"    # $2000 USD monthly budget
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Service"
    values = ["Amazon Elastic Compute Cloud - Compute"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["oncall@example.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["oncall@example.com"]
  }
}
```

---

## Compute Cost Optimization Reference

| Strategy | Saving potential | Effort | Risk |
|---|---|---|---|
| Spot/Preemptible workers | 60–90% | Medium | Requires graceful drain |
| Right-sizing over-provisioned pods | 20–40% | Low | Re-test stability after resize |
| Autoscale down at night/weekend | 15–30% | Low | Only for non-latency-sensitive workloads |
| Committed use discounts (1-yr/3-yr) | 30–55% | Low | Commit risk if usage drops |
| Cap max instances in ASG | 0% saving, cost protection | Very low | May cause latency under extreme spike |
| Per-tenant concurrency limit | 0% saving, fairness | Low | Noisy neighbors get queued, not dropped |

---

## Quality Checks

- [ ] Autoscaling group / HPA has a `maxReplicas` cap — unbounded scale-up is impossible
- [ ] Spot/Preemptible instances used for all stateless async workers
- [ ] Spot termination watcher drains in-flight jobs gracefully — no incomplete job data loss
- [ ] CPU and memory utilization logged every 5 minutes — right-sizing candidates visible in dashboard
- [ ] Per-tenant concurrency limit set — single tenant cannot consume > `maxPerTenant` worker slots
- [ ] Compute budget alert at 80% of monthly budget with 48 hours lead time
- [ ] Worker pool `maxWorkers` is configured via env var — can be tuned without redeploy

## After Completion

- Use **`cost-aware-architecture`** for overall FinOps strategy including storage and egress costs
- Use **`backpressure-patterns`** to design the queue/worker interaction that prevents runaway job accumulation
- Use **`load-shedding`** to protect compute under overload conditions
- Use **`tenant-aware-metrics`** to emit per-tenant CPU and memory consumption signals

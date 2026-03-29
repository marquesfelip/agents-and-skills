---
name: tenant-safe-background-jobs
description: >
  Tenant isolation during async job execution in multi-tenant SaaS systems.
  Use when: background job tenant isolation, tenant-safe jobs, job queue tenant_id,
  worker tenant context, async job multi-tenant, background worker isolation,
  job payload tenant, cron job tenant, batch job tenant, scheduled task tenant,
  worker tenant reconstruction, job retry tenant context, platform job tenant job distinction,
  global job tenant job, job queue design multi-tenant, job deduplication tenant,
  job context propagation, job payload design, tenant job queue, job scoping.
argument-hint: >
  Describe your job queue system (Sidekiq, Celery, Faktory, Go workers, Temporal, etc.),
  the type of job affected (per-tenant operation, platform-wide batch, scheduled sync),
  and the tenant context problem (context missing in worker, wrong tenant, context from request).
---

# Tenant-Safe Background Jobs Specialist

## When to Use

Invoke this skill when you need to:
- Design job payloads that carry tenant context explicitly
- Reconstruct complete tenant context in a worker from the job payload
- Separate platform (global) jobs from tenant-scoped jobs clearly
- Prevent shared mutable state from leaking tenant data across concurrent job executions
- Ensure job retries re-establish tenant context correctly
- Check entitlements inside a job (tenant may have downgraded since job was enqueued)

---

## Step 1 — The Core Principle

**Tenant context is NOT inherited from the enqueuing request — it must be serialized into the job payload.**

```go
// BAD: tenant context from request context is lost when the worker picks up the job
func (h *OrderHandler) ExportOrders(w http.ResponseWriter, r *http.Request) {
    tenant := RequireTenant(r.Context())

    // WRONG: enqueue job without tenant context
    // Worker gets an empty context.Background() — no tenant
    h.queue.Enqueue(r.Context(), ExportOrdersJob{})
}

// GOOD: tenant_id explicitly in the job payload
func (h *OrderHandler) ExportOrders(w http.ResponseWriter, r *http.Request) {
    tenant := RequireTenant(r.Context())

    h.queue.Enqueue(r.Context(), ExportOrdersJob{
        TenantID: tenant.ID,   // explicitly serialized
        UserID:   UserFromContext(r.Context()).ID,
        Format:   req.Format,
    })
}
```

**Checklist:**
- [ ] Every tenant-scoped job payload contains `TenantID string` as a mandatory field
- [ ] Job payload structs are validated — `TenantID` cannot be empty when enqueuing
- [ ] No job reads `tenant_id` from `context.Background()` — it doesn't exist there

---

## Step 2 — Job Payload Design

**Tenant-scoped job payload template:**
```go
type ExportOrdersJob struct {
    // Routing and identity
    TenantID  string `json:"tenant_id"`  // mandatory — worker reconstructs context from this
    UserID    string `json:"user_id"`    // who triggered this job
    RequestID string `json:"request_id"` // for distributed tracing

    // Business payload
    Format    string    `json:"format"`   // "csv", "xlsx"
    DateFrom  time.Time `json:"date_from"`
    DateTo    time.Time `json:"date_to"`

    // Retry metadata
    Attempt   int    `json:"attempt"`    // incremented on each retry
    EnqueuedAt time.Time `json:"enqueued_at"` // when originally enqueued (not retry time)
}

func (j ExportOrdersJob) Validate() error {
    if j.TenantID == "" {
        return errors.New("ExportOrdersJob: TenantID is required")
    }
    if j.UserID == "" {
        return errors.New("ExportOrdersJob: UserID is required")
    }
    return nil
}
```

**Distinguishing tenant jobs from platform jobs:**
```go
// Platform/global job — operates across all tenants; no single tenant_id
type InvoiceReminderJob struct {
    // No tenant_id — this job queries all tenants
    ScheduledAt time.Time `json:"scheduled_at"`
}

// Tenant-scoped job — one job per tenant
type TenantInvoiceReminderJob struct {
    TenantID    string    `json:"tenant_id"`  // processes a single tenant
    ScheduledAt time.Time `json:"scheduled_at"`
}

// Pattern: the global job fans out into per-tenant jobs
func (w *InvoiceReminderWorker) Process(ctx context.Context, job InvoiceReminderJob) error {
    tenants, err := w.tenantRepo.ListActive(ctx)
    for _, tenant := range tenants {
        // Enqueue one scoped job per tenant
        w.queue.Enqueue(ctx, TenantInvoiceReminderJob{TenantID: tenant.ID})
    }
    return nil
}
```

Checklist:
- [ ] Job types documented as either "platform" (global) or "tenant-scoped"
- [ ] Platform jobs fan out into per-tenant jobs — do not process all tenants in a single job execution
- [ ] Tenant job payloads validated before enqueueing — empty TenantID caught at enqueue time
- [ ] `EnqueuedAt` vs. `ProcessedAt` tracked separately — useful for diagnosing queue backlogs

---

## Step 3 — Worker Tenant Context Reconstruction

The worker reconstructs a complete, authenticated tenant context from the job payload before any business logic runs.

```go
type ExportOrdersWorker struct {
    tenantRepo  TenantRepository
    orderRepo   OrderRepository  // unscoped — receives tenant from context
    exporter    ExportService
    entitl      EntitlementService
    auditLog    AuditLogger
}

func (w *ExportOrdersWorker) Process(ctx context.Context, job ExportOrdersJob) error {
    if err := job.Validate(); err != nil {
        // Programming error in the enqueuing code — do not retry
        return fmt.Errorf("invalid job payload: %w (permanent failure)", err)
    }

    // Step 1: Reconstruct tenant context from DB (not from job payload directly)
    // Re-fetch from DB — tenant status may have changed since job was enqueued
    tenant, err := w.tenantRepo.Get(ctx, job.TenantID)
    if err != nil || tenant == nil {
        return fmt.Errorf("tenant not found: %s", job.TenantID)
    }

    // Step 2: Check tenant is still active
    if tenant.Status == "canceled" || tenant.Status == "suspended" {
        // Skip gracefully — tenant is no longer active
        slog.Info("skipping job for inactive tenant",
            slog.String("tenant_id", job.TenantID),
            slog.String("status", tenant.Status),
        )
        return nil // do not retry
    }

    // Step 3: Check entitlement — tenant may have downgraded since job was enqueued
    if !w.entitl.HasFeature(ctx, job.TenantID, "data_export") {
        slog.Warn("tenant no longer has export entitlement",
            slog.String("tenant_id", job.TenantID))
        return nil // do not retry; notify user
    }

    // Step 4: Build tenant context for downstream calls
    tenantCtx := WithTenant(ctx, &TenantContext{
        ID:     tenant.ID,
        Slug:   tenant.Slug,
        PlanID: tenant.PlanID,
        Status: tenant.Status,
    })

    // Business logic runs with proper tenant context
    return w.doExport(tenantCtx, job)
}
```

Checklist:
- [ ] Tenant re-fetched from DB in worker — not blindly trusted from payload (tenant may be deleted)
- [ ] Tenant status checked: skip job for canceled/suspended tenants — do not error/retry
- [ ] Entitlements re-checked in worker — tenant may have downgraded since job was enqueued
- [ ] Full tenant context reconstructed before any service or repository calls
- [ ] Invalid payload (empty TenantID) flagged as permanent failure — not retried

---

## Step 4 — Prevent Cross-Tenant Contamination in Workers

Concurrent job workers must not share mutable state that could carry tenant context.

```go
// BAD: global/module-level state that workers share
var currentTenantID string // worker 1 sets "acme", worker 2 reads "acme" instead of "globex"

// GOOD: per-job, immutable context propagated via context.Context
func (w *Worker) Process(ctx context.Context, job ExportOrdersJob) error {
    tenantCtx := WithTenant(ctx, ...) // scoped to this job's goroutine/goroutine stack
    return w.doExport(tenantCtx, job) // all calls use this context
}
```

**Pooled repository safety:**
```go
// GOOD: repository scoped per-job, not shared across jobs
func (w *ExportOrdersWorker) doExport(ctx context.Context, job ExportOrdersJob) error {
    // Repository is constructed per-job with this job's tenant_id
    // (alternatively, use context-based tenant injection)
    repo := NewOrderRepo(w.db, job.TenantID)
    orders, err := repo.List(ctx, OrderFilter{})
    // ...
}
```

Checklist:
- [ ] No global or package-level variables hold tenant context
- [ ] Repository instances created per-job (or use context-based tenant injection) — not shared
- [ ] Worker goroutines terminate with the job — no goroutine leaks carrying stale tenant context
- [ ] Concurrent job stress test: run N workers with different tenants simultaneously; verify no data cross-contamination

---

## Step 5 — Job Retry Tenant Safety

On retry, the worker must re-establish tenant context from the job payload — not from any retained state.

```go
// Safe retry: job payload carries all needed context; worker is stateless
// Each retry goes through the full context reconstruction in Step 3

// Retry limits by failure type:
// - Transient (DB timeout, network): retry up to 5 times with exponential backoff
// - Permanent (missing tenant, invalid payload, entitlement denied): do not retry
// - Soft failure (tenant inactive): do not retry; log and drop
func classifyError(err error) RetryPolicy {
    switch {
    case errors.Is(err, ErrTenantNotFound):
        return RetryPolicy{ShouldRetry: false, Permanent: true}
    case errors.Is(err, ErrEntitlementDenied):
        return RetryPolicy{ShouldRetry: false, Permanent: true}
    case errors.Is(err, ErrTenantInactive):
        return RetryPolicy{ShouldRetry: false, Permanent: false}
    default:
        return RetryPolicy{ShouldRetry: true, MaxAttempts: 5, Backoff: ExponentialBackoff}
    }
}
```

Checklist:
- [ ] Retry re-runs full tenant context reconstruction — not cached from previous attempt
- [ ] Permanent failures (invalid tenant, denied entitlement) do not retry — sent to DLQ or logged
- [ ] DLQ entries include `tenant_id` — enables per-tenant DLQ analysis
- [ ] Retry backoff does not cause resource starvation for other tenants in the same queue

---

## Output Report

### Critical
- Job enqueued without `tenant_id` in payload — worker has no tenant context; processes data globally or fails silently
- Worker reads tenant context from request `context.Context` — context is canceled at request end; worker runs with nil tenant

### High
- Tenant not re-fetched from DB in worker — deleted or suspended tenants processed; operations on canceled accounts
- Global variable holds tenant context — concurrent workers contaminate each other's tenant context
- Entitlement not re-checked in worker — downgraded tenant continues to execute premium operations asynchronously

### Medium
- Platform fan-out job processes all tenants in a single execution — one tenant's failure cancels processing for all; retry retries all tenants
- Invalid payload not classified as permanent failure — retried indefinitely; waste queue capacity
- DLQ entries do not include `tenant_id` — DLQ analysis cannot filter by affected tenant

### Low
- Job payload does not contain `EnqueuedAt` — cannot diagnose how long jobs sat in queue before processing
- Per-job repository not scoped — repository reuse across jobs risks tenant context bleed via ORM scope caching
- No concurrent worker multi-tenant stress test — cross-contamination assumed not to occur

### Passed
- Every tenant-scoped job payload contains `TenantID` validated as non-empty at enqueue time
- Worker re-fetches tenant from DB; checks status and entitlement; skips gracefully for inactive tenants
- Platform jobs fan out into per-tenant jobs; each tenant job is an independent unit of work
- No global mutable state; repositories scoped per-job via constructor injection; context.Context carries tenant value
- Retry classification: transient (retry); permanent (DLQ); inactive tenant (drop); DLQ entries include tenant_id

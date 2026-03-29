---
name: job-idempotency
description: 'Idempotent background job execution patterns for SaaS systems. Use when: job idempotency, idempotent job, background job deduplication, job replay safety, duplicate job execution, at-least-once job delivery, job key, idempotency key job, job deduplication table, job processed table, safe retry job, background worker idempotency, cron idempotency, scheduled job idempotency, job concurrency safety, double execution prevention, job uniqueness, job exactly-once.'
argument-hint: 'Describe the job type (event-driven vs scheduled), what makes two invocations a duplicate (same event ID, same entity ID + date, etc.), and whether partial execution must be rolled back or can be resumed.'
---

# Job Idempotency

## When to Use

Invoke this skill when you need to:
- Ensure re-delivering a job message produces the same outcome as delivering it once
- Build a deduplication store that prevents double billing, double email, or double provisioning
- Design idempotency keys for jobs that are driven by external events (webhooks, billing events)
- Handle partial execution — decide whether to roll back or continue from a checkpoint
- Prevent concurrent duplicate execution of the same logical job
- Make scheduled jobs safe to run multiple times in the same window (cron overlap protection)

---

## Idempotency Failure Modes

| Failure mode | Example | Fix |
|---|---|---|
| Message re-delivery | Broker retries after ACK timeout | Deduplication by event/message ID |
| Double cron trigger | Two cron workers start at same time | Distributed lock by job name |
| Retry after partial success | Job succeeded 50%, crashed, retried from zero | Checkpoint or idempotent operations |
| Concurrent duplicate | Two workers pick the same job simultaneously | `SELECT ... FOR UPDATE SKIP LOCKED` |
| Webhook replay | Provider sends same webhook twice | Dedup by webhook delivery ID |
| At-least-once guarantees | SQS, Kafka — messages delivered ≥1 time | Consumer-side deduplication |

---

## Step 1 — Processed Events Table (Event-Driven Jobs)

```sql
CREATE TABLE processed_events (
    event_id     TEXT        NOT NULL,  -- broker message ID or idempotency key
    consumer     TEXT        NOT NULL,  -- consumer name — same event may be consumed by multiple
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (event_id, consumer)
);

-- TTL index: purge records older than 30 days (events cannot be redelivered after broker retention)
CREATE INDEX idx_processed_events_age ON processed_events(processed_at);
```

```go
// IdempotencyGuard wraps any job handler with deduplication
type IdempotencyGuard struct {
    repo     ProcessedEventsRepo
    consumer string
}

func (g *IdempotencyGuard) WrapHandler(h JobHandler) JobHandler {
    return func(ctx context.Context, job Job) error {
        // Check: has this job been processed by this consumer?
        done, err := g.repo.Has(ctx, job.ID, g.consumer)
        if err != nil {
            return fmt.Errorf("idempotency check: %w", err)
        }
        if done {
            return nil // already processed — discard
        }

        // Execute business logic
        if err := h(ctx, job); err != nil {
            return err // do NOT mark processed — broker will retry
        }

        // Mark processed ONLY after success
        return g.repo.Insert(ctx, job.ID, g.consumer)
    }
}
```

---

## Step 2 — Transactional Idempotency (DB-Backed Jobs)

For jobs that write to the database, mark processed in the same transaction as the business write:

```go
func (w *BillingWorker) Handle(ctx context.Context, job InvoicePaidJob) error {
    return w.db.Transaction(ctx, func(tx DB) error {
        // Dedup: INSERT ... ON CONFLICT DO NOTHING — atomic with business logic
        inserted, err := w.processed.InsertIfAbsent(ctx, tx, job.EventID, "billing-worker")
        if err != nil {
            return fmt.Errorf("dedup: %w", err)
        }
        if !inserted {
            return nil // duplicate — discard safely
        }

        // Business logic runs only once
        if err := w.entitlements.ActivateForTenant(ctx, tx, job.TenantID, job.PlanID); err != nil {
            return err // TX rolls back — including the processed marker
        }
        if err := w.invoiceRepo.MarkPaid(ctx, tx, job.InvoiceID); err != nil {
            return err
        }
        return nil
    })
}
```

**Pattern:** dedup INSERT and business INSERT in the same transaction — if business logic fails, both roll back. On retry, dedup INSERT succeeds and business logic runs again.

---

## Step 3 — Idempotency Keys for Scheduled Jobs

Scheduled jobs are stateless by default. Use a "lease" table to prevent overlap:

```sql
CREATE TABLE job_leases (
    job_name     TEXT        PRIMARY KEY,
    locked_at    TIMESTAMPTZ NOT NULL,
    locked_until TIMESTAMPTZ NOT NULL,  -- TTL prevents stale locks on crash
    worker_id    TEXT        NOT NULL    -- hostname:pid for debugging
);
```

```go
type JobLease struct {
    repo     JobLeaseRepo
    workerID string
}

func (l *JobLease) TryAcquire(ctx context.Context, jobName string, ttl time.Duration) (bool, error) {
    now := time.Now().UTC()
    // Upsert: insert new or take over expired lease
    acquired, err := l.repo.TryInsertOrTakeover(ctx, JobLeaseRow{
        JobName:     jobName,
        LockedAt:    now,
        LockedUntil: now.Add(ttl),
        WorkerID:    l.workerID,
    })
    return acquired, err
}

func (l *JobLease) Release(ctx context.Context, jobName string) error {
    return l.repo.Delete(ctx, jobName, l.workerID)
}

// Usage in a scheduled job
func (w *TrialExpiryWorker) Run(ctx context.Context) error {
    acquired, err := w.lease.TryAcquire(ctx, "trial-expiry", 10*time.Minute)
    if err != nil {
        return err
    }
    if !acquired {
        return nil // another worker holds the lock — skip this run
    }
    defer w.lease.Release(ctx, "trial-expiry")

    return w.processExpiredTrials(ctx)
}
```

---

## Step 4 — Idempotent Operations Reference

Design each operation to be safe when called multiple times:

```go
// WRONG — not idempotent: duplicate email on retry
func sendWelcomeEmail(tenantID uuid.UUID) error {
    return mailer.Send(tenantID, "welcome")
}

// CORRECT — idempotent: track sends in DB
func sendWelcomeEmailIfNotSent(ctx context.Context, tx DB, tenantID uuid.UUID) error {
    sent, err := emailLog.InsertIfAbsent(ctx, tx, tenantID, "welcome")
    if err != nil || !sent {
        return err
    }
    return mailer.Send(ctx, tenantID, "welcome")
}

// WRONG — external API: creates duplicate subscription on retry
stripe.CreateSubscription(customerID, priceID)

// CORRECT — use Stripe idempotency key
stripe.CreateSubscription(customerID, priceID,
    stripe.WithIdempotencyKey(fmt.Sprintf("sub-%s-%s", tenantID, priceID)))

// CORRECT — use INSERT ... ON CONFLICT for DB writes
_, err = db.Exec(ctx,
    `INSERT INTO tenant_features (tenant_id, feature)
     VALUES ($1, $2)
     ON CONFLICT (tenant_id, feature) DO NOTHING`,
    tenantID, featureKey)
```

---

## Step 5 — Concurrent Execution Prevention

```sql
-- Queue table with advisory lock pattern (PostgreSQL)
CREATE TABLE job_queue (
    id          BIGSERIAL PRIMARY KEY,
    job_type    TEXT        NOT NULL,
    payload     JSONB       NOT NULL,
    status      TEXT        NOT NULL DEFAULT 'pending',  -- pending | processing | done | failed
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    claimed_at  TIMESTAMPTZ,
    worker_id   TEXT
);

-- Claim a job atomically — prevents two workers from running the same job
UPDATE job_queue SET status = 'processing', claimed_at = now(), worker_id = $1
WHERE id = (
    SELECT id FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
RETURNING *;
```

```go
// In Go: claim and process
func (w *Worker) claimAndProcess(ctx context.Context) error {
    job, err := w.queue.Claim(ctx, w.workerID)
    if errors.Is(err, ErrNoJobsAvailable) {
        return nil // queue empty
    }
    if err != nil {
        return err
    }

    if err := w.handle(ctx, job); err != nil {
        w.queue.MarkFailed(ctx, job.ID, err.Error())
        return err
    }
    return w.queue.MarkDone(ctx, job.ID)
}
```

---

## Idempotency Strategy Decision Table

| Job trigger | Idempotency key source | Dedup scope | Storage |
|---|---|---|---|
| Kafka / SQS message | Message ID (broker-assigned) | Per message ID + consumer | `processed_events` table |
| Stripe webhook | Stripe Event ID | Per event ID | `processed_events` table |
| Scheduled cron | Job name | Per job name (one at a time) | `job_leases` table |
| HTTP retry w/ client key | `Idempotency-Key` header | Per key + endpoint | `idempotency_keys` table |
| Outbox relay publish | Outbox row ID | Per row ID | `outbox_events.published_at` |

---

## Quality Checks

- [ ] Every job handler checks a deduplication store before executing business logic
- [ ] Deduplication INSERT and business logic write happen in the same DB transaction
- [ ] Processed marker is written AFTER success — never before (prevents masking failures)
- [ ] External API calls (Stripe, etc.) use idempotency keys derived from stable job inputs
- [ ] Scheduled jobs use `job_leases` with TTL — stale locks expire automatically
- [ ] `SELECT ... FOR UPDATE SKIP LOCKED` used in queue workers — prevents concurrent claim
- [ ] `processed_events` rows are purged after broker retention period (prevent unbounded growth)
- [ ] Job failure does NOT write the processed marker — retry will re-execute cleanly

## After Completion

- Use **`event-driven-saas-patterns`** for the outbox pattern that ensures at-least-once delivery
- Use **`dead-letter-handling`** for jobs that fail after max retries
- Use **`async-processing-patterns`** for queue worker design and retry backoff
- Use **`distributed-workflows`** for long-running workflows that span multiple jobs

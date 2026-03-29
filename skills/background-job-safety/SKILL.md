---
name: background-job-safety
description: 'Safe job execution, retries, deduplication, and failure recovery for background jobs. Use when: background job safety, job deduplication, job retry, cron job safety, scheduled job, worker safety, job failure recovery, job idempotency, job timeout, job cancellation, Celery job, Sidekiq job, Hangfire job, job monitoring, job queue, job orchestration, at-most-once job, at-least-once job.'
argument-hint: 'Background job or scheduled task to review — describe what the job does, how it is triggered, and what infrastructure runs it'
---

# Background Job Safety Specialist

## When to Use
- Reviewing a background job for correctness, safety, and failure handling
- Designing a new scheduled or queue-triggered job from scratch
- Debugging jobs that run multiple times, fail silently, or leave partial state
- Implementing retry and deduplication for worker consumers
- Establishing job safety standards for a team or codebase

---

## Step 1 — Job Safety Property Checklist

Every background job should satisfy these properties before going to production:

| Property | What it means | How to verify |
|---|---|---|
| **Idempotent** | Running the job N times produces the same result as running it once | Test: trigger job twice; assert no duplicate side effects |
| **Retryable** | Failure at any point can be retried safely without corruption | Test: simulate failure mid-job; retry; assert clean final state |
| **Bounded** | Job has a maximum execution time; cannot run forever | Timeout configured; monitored |
| **Observable** | Success, failure, and duration are logged and tracked | Metrics/alerts configured |
| **Scoped** | Processes only the data it should (tenant, date range) | Review query filters |
| **Deduplicable** | Two instances triggered for the same work produce one execution | Distributed lock or idempotency key in place |

---

## Step 2 — Idempotent Job Design

The most critical property. Every job must produce the same outcome regardless of how many times it runs:

```python
# BAD — not idempotent: sends duplicate emails on retry
@celery_app.task
def send_monthly_invoices():
    customers = db.query(Customer).filter(Customer.is_active == True).all()
    for customer in customers:
        invoice_pdf = generate_invoice(customer)
        email_client.send(to=customer.email, attachment=invoice_pdf)
        # If crash here after sending N emails → retry sends all N again!

# GOOD — idempotent: tracks sent status; skips already-processed customers
@celery_app.task
def send_monthly_invoices(billing_month: str):
    # billing_month = "2024-03" — uniquely identifies the job run
    customers = db.query(Customer).filter(
        Customer.is_active == True,
        ~db.query(InvoiceSent).filter(          # exclude already processed
            InvoiceSent.customer_id == Customer.id,
            InvoiceSent.billing_month == billing_month,
        ).exists()
    ).all()

    for customer in customers:
        invoice_pdf = generate_invoice(customer, billing_month)
        email_client.send(to=customer.email, attachment=invoice_pdf)
        # Record completion immediately after success
        db.save(InvoiceSent(
            customer_id=customer.id,
            billing_month=billing_month,
            sent_at=datetime.utcnow(),
        ))
        db.commit()   # commit per customer — partial progress preserved on crash
```

---

## Step 3 — Deduplication: Preventing Duplicate Concurrent Runs

### Approach 1: Distributed Lock (For Cron Jobs)

```python
@celery_app.task
def daily_report_job():
    lock_key = f"job:daily-report:{date.today().isoformat()}"

    try:
        with RedisDistributedLock(redis, lock_key, ttl_seconds=3600):
            run_daily_report()
    except LockNotAcquiredError:
        logger.info("Daily report already running or completed for today; skipping")
```

### Approach 2: Unique Constraint (Database-Level)

```python
class JobRun(Base):
    job_type    = Column(String, nullable=False)
    job_key     = Column(String, nullable=False)    # e.g., "2024-03" for monthly job
    status      = Column(String, default='running') # running | completed | failed
    started_at  = Column(DateTime, default=datetime.utcnow)
    completed_at = Column(DateTime, nullable=True)
    __table_args__ = (
        UniqueConstraint('job_type', 'job_key', name='uq_job_run'),
    )

def run_deduplicated_job(job_type: str, job_key: str, job_fn: callable):
    try:
        db.execute(
            "INSERT INTO job_runs (job_type, job_key) VALUES (%s, %s)",
            (job_type, job_key)
        )
        db.commit()         # commit the run record first (lock in DB)
    except UniqueViolationError:
        logger.info("Job already ran or is running", job_type=job_type, job_key=job_key)
        return

    try:
        job_fn()
        db.execute("UPDATE job_runs SET status='completed', completed_at=now() WHERE job_type=%s AND job_key=%s",
                   (job_type, job_key))
        db.commit()
    except Exception as e:
        db.execute("UPDATE job_runs SET status='failed' WHERE job_type=%s AND job_key=%s",
                   (job_type, job_key))
        db.commit()
        raise
```

---

## Step 4 — Retry Strategy for Jobs

```python
# Celery retry configuration
@celery_app.task(
    bind=True,
    max_retries=5,
    default_retry_delay=60,       # base delay in seconds
    autoretry_for=(TransientError, requests.Timeout),
    retry_backoff=True,           # exponential backoff
    retry_backoff_max=600,        # cap at 10 minutes
    retry_jitter=True,            # randomize to avoid thundering herd
)
def process_payment(self, payment_id: str):
    try:
        result = payment_gateway.charge(payment_id)
        save_result(payment_id, result)
    except PaymentDeclinedError:
        # Non-transient: do not retry; dead-letter for review
        logger.error("Payment declined", payment_id=payment_id)
        self.update_state(state='DECLINED')
        return  # do not raise — no retry
    except requests.Timeout as exc:
        # Transient: retry with backoff
        raise self.retry(exc=exc)
```

### Error Classification for Retry

| Error type | Retry? | Action |
|---|---|---|
| Network timeout | Yes | Retry with exponential backoff |
| 503 Service Unavailable | Yes | Retry; respect Retry-After if present |
| Rate limited (429) | Yes | Wait for Retry-After; then retry |
| 400 Bad Request / ValidationError | No | Dead-letter; fix the data or code |
| 404 Not Found | No | Entity was deleted; move on |
| Business rule violation | No | Dead-letter; human review |
| DB constraint violation | No | Indicates a bug; do not retry blindly |
| Duplicate key error | No | Idempotency worked correctly; skip |

---

## Step 5 — Timeout and Cancellation

```python
import signal
from contextlib import contextmanager

@contextmanager
def job_timeout(seconds: int):
    """Raise TimeoutError if job exceeds allowed duration."""
    def _handler(signum, frame):
        raise TimeoutError(f"Job exceeded {seconds}s time limit")
    signal.signal(signal.SIGALRM, _handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)  # cancel alarm

# Usage
@celery_app.task(soft_time_limit=300, time_limit=360)  # Celery-level timeout
def export_report(report_id: str):
    with job_timeout(290):   # application-level graceful timeout before hard kill
        data = fetch_all_data(report_id)
        pdf = generate_pdf(data)
        upload_to_storage(pdf)
```

### Timeout Configuration

| Job type | Soft timeout | Hard timeout |
|---|---|---|
| Webhook delivery | 10s | 15s |
| Export (small) | 60s | 90s |
| Export (large) | 300s | 360s |
| ML inference | 120s | 180s |
| Nightly batch | 3600s | 4200s |

**Soft timeout:** triggers cleanup; allows graceful shutdown.
**Hard timeout:** kills the process; no cleanup; use a safe higher value.

---

## Step 6 — Failure Recovery and Dead Letter Jobs

```python
# Dead letter job handler
@celery_app.task
def handle_dead_letter(original_task_name: str, original_args: list, failure_reason: str, attempts: int):
    """Called when a job exhausts all retries."""
    logger.error("Job exhausted retries — dead-lettered",
                 task=original_task_name,
                 attempts=attempts,
                 reason=failure_reason)

    # Persist for investigation and potential replay
    db.save(DeadLetterJob(
        task_name=original_task_name,
        args=json.dumps(original_args),
        failure_reason=failure_reason,
        attempts=attempts,
        failed_at=datetime.utcnow(),
    ))

    # Alert on-call if job is critical
    if is_critical_job(original_task_name):
        pagerduty.trigger(
            severity='error',
            summary=f"Critical job failed after {attempts} attempts: {original_task_name}",
        )
```

### Dead Letter Job Features Required

- [ ] Failed jobs persisted to dead letter table with failure reason and original args
- [ ] Dead letter job count alerted (> 0 triggers notification)
- [ ] Admin interface or CLI to inspect and replay dead letter jobs
- [ ] Replay is idempotent — safe to run dead letter jobs again after fix deployed

---

## Step 7 — Observability and Monitoring

```python
# Structured job lifecycle logging
def log_job_lifecycle(job_name: str, job_key: str):
    start = time.monotonic()
    logger.info("Job started", job=job_name, key=job_key)
    try:
        yield
        duration_ms = (time.monotonic() - start) * 1000
        logger.info("Job completed", job=job_name, key=job_key, duration_ms=duration_ms)
        metrics.increment('job.completed', tags={'job': job_name})
        metrics.histogram('job.duration_ms', duration_ms, tags={'job': job_name})
    except Exception as e:
        duration_ms = (time.monotonic() - start) * 1000
        logger.error("Job failed", job=job_name, key=job_key, error=str(e), duration_ms=duration_ms)
        metrics.increment('job.failed', tags={'job': job_name})
        raise
```

### Required Alerts

| Alert | Condition | Severity |
|---|---|---|
| Job not running | Cron job missed expected execution window | Warning |
| Job too slow | Duration > P95 threshold | Warning |
| Job failure rate | > 5% of job invocations fail | Error |
| Dead letter depth | > 0 dead letter jobs | Error |
| Job queue backlog | > N jobs pending | Warning |

---

## Output Report

```
## Background Job Safety Review: <job name>

### Critical
- send_monthly_invoices sends emails before marking them as sent; crash mid-loop causes duplicate emails on retry
  Customers receive same invoice email 2–5 times (retry count) on transient failure
  Fix: save InvoiceSent record atomically with each send; skip already-sent customers on retry

- No job timeout configured — stuck jobs from external API hangs run indefinitely
  Worker thread pool exhausted; all other jobs blocked; service degradation
  Fix: set Celery soft_time_limit=300, time_limit=360; add application-level timeout context

### High
- No deduplication: two cron triggers within same minute both run full job concurrently
  Duplicate invoice generation; double sends; database constraint violations
  Fix: acquire Redis distributed lock keyed by (job_type, date); second trigger skips execution

- No dead letter handling — jobs exhausting retries are silently dropped
  Critical payment settlement failures disappear; no alerting; no audit trail
  Fix: configure dead_letter_task; persist to dead_letter_jobs table; alert on queue depth > 0

### Medium
- Retry uses fixed 60s delay on all errors including non-transient (ValidationError)
  ValidationErrors retried 5 times; wastes retries that will always fail
  Fix: do not retry validation or business rule errors; dead-letter immediately

- Entire batch committed in single transaction — failure on item 400/500 rolls back all 400 successes
  Long re-run from scratch after partial failure
  Fix: commit per item or per chunk (100 items); idempotent design preserves progress

### Low
- Job completion not logged with duration — cannot monitor performance regression
  Fix: log duration_ms on completion; emit job.duration_ms histogram metric

- Job scheduled without jitter — all instances of distributed system trigger simultaneously
  DB spike at exact cron time (e.g., 00:00:00 every day)
  Fix: add random startup delay (0–300s) offset to job schedule

### Passed
- Job key includes billing_month — re-run for same month is safely deduplicated ✓
- Soft and hard timeouts both configured ✓
- Dead letter jobs emit PagerDuty alert ✓
```

---
name: tenant-aware-logging
description: 'Tenant-scoped structured logging practices for SaaS systems. Use when: tenant-aware logging, tenant scoped logs, tenant_id in logs, tenant log context, multi-tenant logging, log tenant isolation, structured logging tenant, tenant log field, request log tenant, background job log tenant, tenant log enrichment, tenant log filtering, log correlation tenant, tenant context log, PII in logs tenant, log scrubbing tenant, tenant log query, Grafana Loki tenant, Datadog tenant logs, tenant log namespace, log per tenant, tenant log field propagation, middleware log tenant.'
argument-hint: 'Describe the logging library in use (slog, zap, zerolog, logrus), where tenant context originates (JWT claim, HTTP header, session), and whether logs flow to Loki, Datadog, CloudWatch, or another backend. Mention any PII sensitivity constraints.'
---

# Tenant-Aware Logging

## When to Use

Invoke this skill when you need to:
- Attach `tenant_id` to every log entry across HTTP handlers, background workers, and async jobs
- Ensure logs are filterable per tenant in Grafana Loki, Datadog, or CloudWatch
- Prevent PII leakage into logs — mask emails, names, and payment identifiers
- Propagate tenant context through `context.Context` across goroutines and service calls
- Design log levels and sampling strategies that don't generate excessive per-tenant noise
- Audit which logs are queryable by tenant for compliance and incident response

---

## Step 1 — Tenant Context Key and Extraction

```go
// pkg/logctx/logctx.go — single definition, used across all packages
package logctx

type contextKey string

const tenantKey contextKey = "tenant_id"

func WithTenantID(ctx context.Context, tenantID string) context.Context {
    return context.WithValue(ctx, tenantKey, tenantID)
}

func TenantID(ctx context.Context) string {
    v, _ := ctx.Value(tenantKey).(string)
    return v
}
```

---

## Step 2 — HTTP Middleware: Inject Tenant into Context and Logger

```go
// Middleware extracts tenant_id from validated JWT and enriches context + logger
func TenantLogMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        claims, ok := auth.ClaimsFromContext(r.Context())
        if !ok {
            next.ServeHTTP(w, r)
            return
        }

        tenantID := claims.TenantID
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Enrich context
        ctx := logctx.WithTenantID(r.Context(), tenantID)
        ctx = logctx.WithRequestID(ctx, requestID)

        // Attach fields to a logger stored in context
        logger := slog.With(
            "tenant_id", tenantID,
            "request_id", requestID,
            "user_id", claims.UserID,
        )
        ctx = logctx.WithLogger(ctx, logger)

        // Add tenant_id to response for client-side correlation
        w.Header().Set("X-Request-ID", requestID)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## Step 3 — Logger Helper: Always Include Tenant Fields

```go
// Logger retrieves the enriched logger from context, with tenant fields pre-attached
// Fallback to default slog if no tenant context is present (e.g., startup logs)
func Logger(ctx context.Context) *slog.Logger {
    if l := logctx.LoggerFromContext(ctx); l != nil {
        return l
    }
    // Not in a tenant request — attach whatever context fields exist
    attrs := []any{}
    if tid := logctx.TenantID(ctx); tid != "" {
        attrs = append(attrs, "tenant_id", tid)
    }
    if rid := logctx.RequestID(ctx); rid != "" {
        attrs = append(attrs, "request_id", rid)
    }
    if len(attrs) > 0 {
        return slog.Default().With(attrs...)
    }
    return slog.Default()
}

// Usage in any service layer — context carries all log fields
func (s *SubscriptionService) Activate(ctx context.Context, id uuid.UUID) error {
    Logger(ctx).Info("activating subscription", "subscription_id", id)

    if err := s.repo.Activate(ctx, id); err != nil {
        Logger(ctx).Error("subscription activation failed", "subscription_id", id, "err", err)
        return err
    }

    Logger(ctx).Info("subscription activated", "subscription_id", id)
    return nil
}
```

---

## Step 4 — Background Jobs: Reconstruct Tenant Context

```go
// Background workers don't have an HTTP request — reconstruct tenant context from job payload
func (w *BillingWorker) Handle(ctx context.Context, job BillingJob) error {
    // Reconstruct tenant context from durable job payload
    ctx = logctx.WithTenantID(ctx, job.TenantID)
    ctx = logctx.WithJobID(ctx, job.ID)

    logger := slog.With(
        "tenant_id", job.TenantID,
        "job_id", job.ID,
        "job_type", "billing",
    )
    ctx = logctx.WithLogger(ctx, logger)

    Logger(ctx).Info("billing job started")
    defer Logger(ctx).Info("billing job finished")

    return w.processInvoice(ctx, job)
}
```

---

## Step 5 — PII Masking: Never Log Raw Sensitive Fields

```go
// MaskEmail masks all but domain: john.doe@example.com → ***@example.com
func MaskEmail(email string) string {
    idx := strings.Index(email, "@")
    if idx < 0 {
        return "***"
    }
    return "***" + email[idx:]
}

// MaskCard masks everything but last 4: 4111111111111111 → ****1111
func MaskCard(card string) string {
    if len(card) <= 4 {
        return "****"
    }
    return "****" + card[len(card)-4:]
}

// CORRECT: mask before logging
Logger(ctx).Info("user email updated", "masked_email", MaskEmail(newEmail))

// WRONG — never log raw PII
// Logger(ctx).Info("user email updated", "email", newEmail) // ❌
```

---

## Step 6 — Log Level Policy per Tenant Action

| Action | Level | Notes |
|---|---|---|
| Request received / response sent | `DEBUG` | Sample at 1% in production to avoid noise |
| Business event (subscription created, plan changed) | `INFO` | Always log with tenant_id |
| Validation failure (user input error) | `WARN` | Include field name, not value |
| Dependency error (DB timeout, API error) | `ERROR` | Include err, tenant_id, operation |
| Security event (auth failure, rate limit hit) | `WARN` or `ERROR` | Emit to security log stream |
| Panic / unhandled error | `ERROR` + alert | Include stack trace, tenant_id |
| Background job started / finished | `INFO` | Include job_id, tenant_id |

---

## Step 7 — Log Query Patterns (Grafana Loki)

```logql
# All errors for one tenant in the last hour
{app="api"} | json | tenant_id="t_abc123" | level="ERROR" | last 1h

# High error rate per tenant — detect noisy tenants
sum by (tenant_id) (
  rate({app="api"} | json | level="ERROR" [5m])
)

# Specific operation failures for a tenant
{app="worker"} | json | tenant_id="t_abc123" | operation="billing" | level="ERROR"

# All logs for a request ID (cross-service tracing)
{app=~"api|worker|billing"} | json | request_id="req_xyz987"
```

---

## Tenant Log Field Reference

| Field | Type | Required | Source |
|---|---|---|---|
| `tenant_id` | string | **Yes** | JWT claim, job payload |
| `request_id` | string | **Yes** | `X-Request-ID` header or generated |
| `user_id` | string | Yes (on user requests) | JWT claim |
| `job_id` | string | Yes (on background jobs) | Job payload |
| `trace_id` | string | Yes (if tracing enabled) | OpenTelemetry span |
| `operation` | string | Recommended | Call site label |
| `err` | string | On error only | `err.Error()` |

---

## Quality Checks

- [ ] `tenant_id` present on every log line that reaches production — verified by log sampling in CI
- [ ] No raw email, payment card, or personal name appears in any log entry
- [ ] Background job workers reconstruct tenant context from job payload before first log line
- [ ] `Logger(ctx)` used throughout service and repository layers — no bare `slog.Info` with hardcoded fields
- [ ] Log levels follow the policy table — `DEBUG` not emitted in production by default
- [ ] Loki / Datadog: `tenant_id` is an indexed label — queries by tenant are fast
- [ ] Security events (auth failures, rate limit hits) routed to a separate log stream or alert

## After Completion

- Use **`tenant-aware-metrics`** to add per-tenant Prometheus counters alongside logs
- Use **`secure-logging`** for broader sensitive-data-in-logs coverage
- Use **`distributed-tracing`** to correlate `trace_id` with `tenant_id` across services
- Use **`structured-logging`** for overall log schema and field naming conventions

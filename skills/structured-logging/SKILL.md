---
name: structured-logging
description: >
  Structured log design, correlation identifiers, and safe log practices for observable systems.
  Use when: structured logging, structured logs, JSON logs, log format design, log fields, correlation ID,
  request ID, trace ID, log context propagation, log levels, log sampling, log enrichment,
  contextual logging, log schema, log standardization, log correlation, log aggregation,
  ELK stack logging, OpenTelemetry logs, Datadog logs, Grafana Loki, log filtering,
  sensitive data in logs, PII in logs, log injection, log cardinality, log verbosity,
  Go zerolog, Go zap, Go slog, structured log Go, log best practices, log review.
argument-hint: >
  Describe the service stack and log destination (e.g., "Go microservices sending JSON logs to
  Loki via Promtail", "Node.js API writing to stdout consumed by Datadog"), plus any existing
  log issues (e.g., unstructured, missing correlation IDs, PII exposure).
---

# Structured Logging Specialist

## When to Use

Invoke this skill when you need to:
- Design or audit a structured log schema for a service or system
- Add or improve correlation IDs across service boundaries
- Define log levels, sampling, and verbosity rules
- Prevent sensitive data (PII, secrets, tokens) from appearing in logs
- Configure a logging library (zap, zerolog, slog, Winston, structlog)
- Establish log standards across a multi-service architecture

---

## Step 1 — Choose Log Format and Library

All logs must be machine-parseable. Use JSON as the universal wire format.

**Library selection:**

| Language | Library | Notes |
|---|---|---|
| Go | `log/slog` (stdlib ≥ 1.21) | Preferred for new services; structured by default |
| Go | `go.uber.org/zap` | High performance, production battle-tested |
| Go | `github.com/rs/zerolog` | Zero-allocation, chainable API |
| Node.js | `pino` | JSON, fast, low overhead |
| Python | `structlog` | Composable processors, stdlib compatible |
| Java | `logback` + `logstash-logback-encoder` | JSON encoder for Logback |

**Example — Go `slog` with JSON output:**
```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

slog.Info("order created",
    slog.String("order_id", orderID),
    slog.String("customer_id", customerID),
    slog.Int("item_count", len(items)),
)
```

**Output:**
```json
{"time":"2026-03-29T14:00:00Z","level":"INFO","msg":"order created","order_id":"ord_123","customer_id":"cust_456","item_count":3}
```

Checklist:
- [ ] All services emit JSON logs to stdout — no file-based logging in containers
- [ ] Logging library configured at startup — not scattered `fmt.Println` calls
- [ ] Log output goes to stdout/stderr — collection is the infrastructure's responsibility
- [ ] Log library not initialized multiple times — single global logger with per-request child loggers

---

## Step 2 — Define the Standard Log Schema

Every log entry should include a consistent set of fields so log aggregators can index and filter them uniformly.

**Mandatory fields:**

| Field | Type | Example | Purpose |
|---|---|---|---|
| `time` | RFC3339 UTC | `2026-03-29T14:00:00Z` | Timestamp for ordering |
| `level` | string | `info`, `warn`, `error` | Severity routing |
| `msg` | string | `"order created"` | Human-readable summary |
| `service` | string | `orders-service` | Which service emitted this |
| `version` | string | `v1.2.3` | Deployed version for correlation |
| `env` | string | `production`, `staging` | Environment for filtering |
| `trace_id` | string | `4bf92f3577b34da6...` | OpenTelemetry trace ID |
| `span_id` | string | `00f067aa0ba902b7` | OpenTelemetry span ID |
| `request_id` | string | `req_abc123` | Per-request unique identifier |

**Context fields (add per domain):**

| Field | When to Add |
|---|---|
| `user_id` | Authenticated user context |
| `tenant_id` | Multi-tenant systems |
| `order_id`, `payment_id` | Domain entity being processed |
| `http_method`, `http_path`, `http_status` | HTTP request logs |
| `duration_ms` | Operation timing |
| `error` | Error message (structured, not stack trace blob) |

**Schema enforcement — Go middleware example:**
```go
func LoggingMiddleware(logger *slog.Logger, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.NewString()
        }

        log := logger.With(
            slog.String("request_id", requestID),
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
        )

        ctx := context.WithValue(r.Context(), loggerKey, log)
        rw := &responseWriter{ResponseWriter: w}
        next.ServeHTTP(rw, r.WithContext(ctx))

        log.Info("request completed",
            slog.Int("status", rw.status),
            slog.Int64("duration_ms", time.Since(start).Milliseconds()),
        )
    })
}
```

Checklist:
- [ ] Mandatory fields present in every log entry — enforced at the middleware/handler layer
- [ ] `service`, `version`, `env` injected at logger initialization — not per log call
- [ ] `trace_id` and `span_id` extracted from OpenTelemetry context — propagated automatically
- [ ] Schema documented and shared across all services — consistency enables cross-service log queries

---

## Step 3 — Correlation Identifiers

Correlation IDs connect a single user action across multiple services, logs, and traces.

**ID hierarchy:**
```
Browser request
  └─ X-Request-ID: req_abc123         ← generated at the edge/API gateway
       └─ trace_id: 4bf92f3577b34da6  ← OpenTelemetry W3C traceparent
            └─ span_id: 00f067aa0ba902b7 (per service hop)
```

**Propagation rules:**
- API gateway or ingress generates `X-Request-ID` if absent; always forwards it downstream
- All downstream HTTP calls include `X-Request-ID` and `traceparent` headers
- Background jobs generate a `job_id` at enqueue time; workers log it on pickup
- Async events carry a `correlation_id` in the message envelope — never lost mid-pipeline

**Go: inject and propagate:**
```go
// Extract from incoming request
requestID := r.Header.Get("X-Request-ID")

// Propagate to outgoing HTTP call
req, _ := http.NewRequestWithContext(ctx, "GET", upstreamURL, nil)
req.Header.Set("X-Request-ID", requestID)

// Propagate to message queue payload
msg := Message{
    CorrelationID: requestID,
    Payload:       payload,
}
```

Checklist:
- [ ] `X-Request-ID` generated at the edge and forwarded to every downstream service
- [ ] All outbound HTTP calls include `X-Request-ID` and W3C `traceparent` headers
- [ ] Message queue events include `correlation_id` in the envelope — not the payload
- [ ] Logs always include `request_id` or `correlation_id` — log queries can reconstruct a full flow
- [ ] `trace_id` and `span_id` auto-injected from the active OpenTelemetry span context

---

## Step 4 — Log Levels and Verbosity Rules

| Level | When to Use | Examples |
|---|---|---|
| `DEBUG` | Detailed internal state during development | Variable values, loop iterations |
| `INFO` | Normal operational events | Request received, record created, job started |
| `WARN` | Unexpected but recoverable condition | Retry attempt, deprecated API called, slow query |
| `ERROR` | Operation failed; action required | DB connection lost, payment declined, dependency unavailable |
| `FATAL` / `PANIC` | Unrecoverable — process must exit | Config missing, port bind failed |

**Rules:**
- `DEBUG` disabled in production — only enabled via dynamic log level change or feature flag
- `INFO` is the default production level — meaningful events, not noise
- `ERROR` must always include: `error` field, affected entity ID, and context to reproduce
- Never log success for every row in a bulk operation — log a summary: `"processed 5000 records in 2.3s"`
- Avoid logging the full HTTP request body — log the operation name and outcome instead

**Dynamic log level (Go slog example):**
```go
var level = new(slog.LevelVar) // default INFO
level.Set(slog.LevelInfo)

// Change at runtime via admin endpoint:
// PUT /admin/log-level {"level":"debug"}
func setLogLevel(l string) {
    switch l {
    case "debug": level.Set(slog.LevelDebug)
    case "warn":  level.Set(slog.LevelWarn)
    default:      level.Set(slog.LevelInfo)
    }
}
```

Checklist:
- [ ] `DEBUG` disabled in production by default — enableable without restart
- [ ] `ERROR` logs include enough context to reproduce and investigate the failure
- [ ] Bulk operations log summaries, not per-item events — prevents log volume explosion
- [ ] Log levels documented in the service README — clear contract for operators

---

## Step 5 — Sensitive Data Protection

Never log secrets, credentials, full payment card numbers, passwords, or raw PII.

**Fields that must never appear in logs:**
- Passwords, PINs, security questions
- API keys, tokens, OAuth credentials, session IDs
- Full credit card numbers, CVVs, bank account numbers
- Government IDs (CPF, SSN, passport)
- Unmasked email (context-dependent — often necessary; mask in sensitive flows)

**Masking strategy:**
```go
// Mask a value: show first 4 chars, mask the rest
func mask(s string) string {
    if len(s) <= 4 {
        return "****"
    }
    return s[:4] + strings.Repeat("*", len(s)-4)
}

// Log a card number safely
slog.Info("payment initiated",
    slog.String("card_last4", card[len(card)-4:]),   // ✅ last 4 only
    // slog.String("card_number", card),              // ❌ never
)
```

**Log injection prevention:**
- Never interpolate raw user input directly into log messages — use structured fields
- Sanitize newlines (`\n`, `\r`) from user-supplied strings before logging
- Do not log raw request bodies — only log parsed, validated, and filtered fields

```go
// ❌ Vulnerable to log injection
slog.Info("user action: " + userInput)

// ✅ Safe — user input is a structured field, not part of the message
slog.Info("user action", slog.String("input", sanitize(userInput)))
```

Checklist:
- [ ] Secrets, tokens, and passwords never appear in any log line — verified by log scanning tool
- [ ] Credit card numbers masked to last 4; government IDs masked or omitted
- [ ] User input sanitized before logging — newlines stripped, length capped
- [ ] Log scanning (`detect-secrets`, `trufflehog`, or similar) runs in CI on log output samples
- [ ] Log aggregator configured to redact patterns matching common secret formats

---

## Step 6 — Log Sampling and Cardinality

High-traffic services can produce millions of log lines per minute. Control cost and noise.

**Sampling strategy:**
| Scenario | Strategy |
|---|---|
| Health check endpoints | Log only errors — suppress 200 responses |
| High-throughput background jobs | Log 1-in-N successes, all failures |
| Slow queries | Log all queries > 500ms; sample < 500ms at 1% |
| External API calls | Log all errors; sample successes at 10% |

**Cardinality rules — avoid high-cardinality fields as log labels:**
- Do NOT use `user_id`, `order_id`, or `request_id` as Loki/Prometheus **labels** — use them as log **fields**
- Labels are for low-cardinality values: `service`, `env`, `level`, `http_method`
- High-cardinality values belong inside the log line body — searchable via log query language

```go
// ❌ High-cardinality label — kills Loki indexer
logger = logger.With("user_id", userID)  // as a label

// ✅ High-cardinality value as a log field — safe
slog.Info("checkout started", slog.String("user_id", userID))
```

Checklist:
- [ ] Health check successes suppressed or sampled at < 1%
- [ ] High-cardinality identifiers are log fields, not aggregator labels
- [ ] Log volume budgeted per service — stay within log aggregator tier limits
- [ ] Sampling implemented server-side (not client-side) — prevents biased samples

---

## Output Report

### Critical
- Passwords, API keys, or tokens logged in plaintext — immediate secret rotation required
- User input interpolated into log messages without sanitization — log injection vulnerability
- PII (government IDs, full card numbers) visible in log aggregator — compliance violation

### High
- No correlation ID — impossible to trace a user request across service boundaries
- `trace_id` not propagated to downstream services — distributed traces broken
- `DEBUG` level active in production — log volume overload and potential secret exposure

### Medium
- Log schema inconsistent across services — cross-service queries require field name guessing
- `ERROR` logs missing context fields — on-call engineer cannot reproduce or triage from logs alone
- Bulk operations logging per-item events — log volume spike during high load

### Low
- High-cardinality values used as aggregator labels — index cardinality explosion over time
- Health check endpoints not sampled — significant fraction of log volume is noise
- Log level not dynamically adjustable — debug investigation requires a service restart

### Passed
- All logs emitted as JSON to stdout; schema includes mandatory fields consistently
- Correlation IDs generated at the edge and propagated through all HTTP and async boundaries
- `trace_id` and `span_id` auto-injected from OpenTelemetry context
- Sensitive data masked or omitted; log injection prevented via structured fields
- Log levels follow defined rules; `DEBUG` disabled in production, dynamically enableable
- Sampling configured for high-volume endpoints; cardinality managed at the aggregator

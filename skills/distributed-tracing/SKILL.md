---
name: distributed-tracing
description: >
  Distributed tracing design and instrumentation across services for debugging and performance analysis.
  Use when: distributed tracing, OpenTelemetry tracing, trace propagation, span design, trace context,
  W3C traceparent, Jaeger, Zipkin, Tempo, Datadog APM, AWS X-Ray, tracing instrumentation,
  service trace, cross-service trace, request tracing, latency tracing, trace sampling,
  parent span, child span, span attributes, trace visualization, bottleneck identification,
  slow request debugging, span events, trace correlation, tracing middleware, auto-instrumentation,
  manual instrumentation, trace cardinality, trace backend, OpenTelemetry SDK, OTEL Go, OTEL Node.
argument-hint: >
  Describe the service topology (e.g., "API gateway → 3 Go microservices → PostgreSQL + Redis"),
  current tracing state (none, partial, broken), and target backend (Jaeger, Grafana Tempo,
  Datadog, AWS X-Ray).
---

# Distributed Tracing Specialist

## When to Use

Invoke this skill when you need to:
- Instrument services with OpenTelemetry distributed tracing from scratch
- Fix broken trace propagation across service boundaries
- Design a span hierarchy and attribute schema
- Configure trace sampling to control cost and volume
- Connect traces to logs and metrics (unified observability)
- Debug latency problems or identify bottlenecks across services

---

## Step 1 — Understand the Trace Model

**Core concepts:**

| Concept | Definition |
|---|---|
| **Trace** | The complete journey of a single request across all services |
| **Span** | A single unit of work within a trace (one service, one operation) |
| **Parent span** | The span that caused the current span to be created |
| **Root span** | The top-level span (e.g., the incoming HTTP request at the edge) |
| **Trace context** | `trace_id` + `span_id` + flags — propagated across service boundaries |
| **W3C traceparent** | Standard HTTP header for trace context: `00-<trace_id>-<span_id>-01` |

**Trace anatomy:**
```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
│
├─ [0ms]  Span: HTTP POST /orders (orders-service)               [root, 142ms total]
│    ├─ [2ms]  Span: validate_order (orders-service)             [12ms]
│    ├─ [14ms] Span: HTTP GET /inventory (inventory-service)     [38ms]
│    │    └─ [16ms] Span: db.query SELECT inventory (postgres)   [35ms]  ← bottleneck
│    ├─ [52ms] Span: publish order.created (kafka)               [5ms]
│    └─ [57ms] Span: db.insert orders (postgres)                 [82ms]
```

Checklist:
- [ ] Team understands the difference between a trace (full journey) and a span (single operation)
- [ ] Target tracing backend selected (Jaeger, Grafana Tempo, Datadog, AWS X-Ray)
- [ ] OpenTelemetry (OTEL) chosen as the vendor-neutral instrumentation layer
- [ ] OTEL Collector deployed as the centralized telemetry pipeline — services export to the collector, not directly to backends

---

## Step 2 — Install and Initialize the OpenTelemetry SDK

**Go — OTEL SDK setup:**
```go
// tracing/tracing.go
package tracing

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func InitTracer(ctx context.Context, serviceName, serviceVersion string) (func(context.Context) error, error) {
    exp, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("otel-collector:4318"),
        otlptracehttp.WithInsecure(), // use TLS in production
    )
    if err != nil {
        return nil, err
    }

    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName(serviceName),
        semconv.ServiceVersion(serviceVersion),
        semconv.DeploymentEnvironment("production"),
    )

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exp),
        trace.WithResource(res),
        trace.WithSampler(trace.ParentBased(trace.TraceIDRatioBased(0.1))), // 10% sampling
    )

    otel.SetTracerProvider(tp)
    return tp.Shutdown, nil
}

// Usage in main.go:
shutdown, err := tracing.InitTracer(ctx, "orders-service", version)
defer shutdown(ctx)
```

**Node.js — OTEL SDK setup:**
```js
// tracing.js — must be required BEFORE all other imports
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  serviceName: 'orders-service',
  traceExporter: new OTLPTraceExporter({ url: 'http://otel-collector:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

Checklist:
- [ ] OTEL SDK initialized at process startup — before any other code runs
- [ ] `resource` attributes include `service.name`, `service.version`, `deployment.environment`
- [ ] Exporter targets the OTEL Collector — not a backend directly (collector provides buffering and routing)
- [ ] TLS enabled on the exporter in production — traces contain sensitive timing and metadata
- [ ] Graceful shutdown calls `TracerProvider.Shutdown()` — flushes pending spans before exit

---

## Step 3 — Auto-Instrumentation and Span Creation

**Auto-instrumentation covers (zero code change):**
- Incoming HTTP requests → root span created automatically
- Outgoing HTTP calls → child span with `http.method`, `http.url`, `http.status_code`
- Database queries (pgx, database/sql, gorm) → span with `db.statement`, `db.system`
- Message queue publish/consume (Kafka, RabbitMQ) → spans with `messaging.system`
- gRPC calls → spans with `rpc.method`, `rpc.service`

**Manual span creation (for business logic):**
```go
tracer := otel.Tracer("orders-service")

func processOrder(ctx context.Context, order Order) error {
    ctx, span := tracer.Start(ctx, "processOrder",
        trace.WithAttributes(
            attribute.String("order.id", order.ID),
            attribute.String("order.customer_id", order.CustomerID),
            attribute.Int("order.item_count", len(order.Items)),
        ),
    )
    defer span.End()

    if err := validateOrder(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    span.AddEvent("order_validated")

    if err := persistOrder(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to persist order")
        return err
    }

    span.SetStatus(codes.Ok, "")
    return nil
}
```

**Naming conventions:**
| Operation Type | Span Name Pattern | Example |
|---|---|---|
| HTTP server | `HTTP <METHOD> <route template>` | `HTTP POST /orders/{id}` |
| HTTP client | `HTTP <METHOD>` | `HTTP GET` |
| Database | `<db.operation> <db.name>.<table>` | `SELECT orders.orders` |
| Message publish | `<topic> publish` | `order.created publish` |
| Message consume | `<topic> receive` | `order.created receive` |
| Business logic | Verb + noun | `processOrder`, `validatePayment` |

Checklist:
- [ ] Auto-instrumentation enabled for HTTP, DB, and messaging libraries
- [ ] Manual spans added for meaningful business logic — not every function call
- [ ] Span names use stable templates — no dynamic values (IDs, user data) in the name
- [ ] `span.RecordError(err)` called for all errors — populates the trace timeline
- [ ] `span.SetStatus(codes.Error, ...)` set alongside `RecordError` — marks the span as failed
- [ ] `span.End()` called via `defer` — spans are never left open

---

## Step 4 — Trace Context Propagation

Context must flow through every service boundary — HTTP, gRPC, message queues, and async jobs.

**HTTP propagation (Go):**
```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Server: auto-extract traceparent from incoming request
handler := otelhttp.NewHandler(myHandler, "orders-service")

// Client: auto-inject traceparent into outgoing request
client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport)}
```

**Message queue propagation:**
```go
import "go.opentelemetry.io/otel/propagation"

// Publisher: inject trace context into message headers
carrier := propagation.MapCarrier{}
otel.GetTextMapPropagator().Inject(ctx, carrier)
msg.Headers = carrier  // attach to the message envelope

// Consumer: extract trace context from message headers
ctx = otel.GetTextMapPropagator().Extract(ctx, propagation.MapCarrier(msg.Headers))
ctx, span := tracer.Start(ctx, "order.created receive")
defer span.End()
```

**W3C traceparent header format:**
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^  ^                                ^                ^
              version  trace_id (128-bit hex)     span_id (64-bit) flags
```

Checklist:
- [ ] `otelhttp` or equivalent OTEL middleware used for HTTP — manual header extraction is error-prone
- [ ] Outgoing HTTP client wrapped with `otelhttp.NewTransport` — propagates automatically
- [ ] Message queue publishers inject context into message headers — not the payload
- [ ] Consumers extract context from message headers before creating child spans
- [ ] gRPC services use `otelgrpc` interceptors — propagates `grpc-trace-bin` or `traceparent`
- [ ] Context (`ctx`) passed through all function calls — never stored in global state

---

## Step 5 — Span Attributes and Events

Attributes make traces searchable and debuggable. Events capture what happened inside a span.

**Attribute guidelines:**
```go
// ✅ Good attributes — low cardinality values for indexing, IDs for correlation
span.SetAttributes(
    attribute.String("order.status", "pending"),        // low cardinality
    attribute.String("order.id", order.ID),             // ID for correlation
    attribute.String("customer.tier", "premium"),        // business context
    attribute.Int("order.item_count", len(order.Items)),
)

// ❌ Avoid putting sensitive data in span attributes
// span.SetAttributes(attribute.String("card.number", card)) // never
```

**Semantic conventions (use OTEL standard attribute names):**
| Category | Attribute | Value Example |
|---|---|---|
| HTTP | `http.method`, `http.route`, `http.status_code` | `POST`, `/orders/{id}`, `201` |
| Database | `db.system`, `db.name`, `db.operation`, `db.statement` | `postgresql`, `orders`, `SELECT` |
| Messaging | `messaging.system`, `messaging.destination` | `kafka`, `order.created` |
| Error | `exception.type`, `exception.message` | `ValidationError`, `"invalid email"` |

**Span events for key moments:**
```go
span.AddEvent("cache_miss", trace.WithAttributes(attribute.String("cache.key", cacheKey)))
span.AddEvent("retry_attempt", trace.WithAttributes(attribute.Int("attempt", attemptNum)))
span.AddEvent("circuit_breaker_open")
```

Checklist:
- [ ] Span attributes follow OpenTelemetry semantic conventions — enables vendor-agnostic dashboards
- [ ] Entity IDs included in attributes for correlation with logs (`order.id`, `user.id`)
- [ ] Sensitive data (card numbers, passwords, tokens) never in span attributes
- [ ] Span events used for state transitions and retries — not as a replacement for logging
- [ ] Attribute count per span bounded — avoid thousands of attributes on a single span

---

## Step 6 — Sampling Strategy

Sampling controls the fraction of traces captured. Too much = cost explosion. Too little = missing incidents.

**Sampling options:**

| Sampler | Description | Use When |
|---|---|---|
| `AlwaysOn` | 100% of traces captured | Low traffic, dev/staging |
| `TraceIDRatioBased(0.1)` | 10% of traces sampled | Medium traffic baseline |
| `ParentBased(ratio)` | Follows parent's decision | Microservices — respect upstream sampling |
| **Tail-based sampling** | Decision made after trace completes | Capture 100% of slow/errored traces |

**Recommended production strategy:**
```go
// Head sampling: ParentBased(10%) + always sample errors via tail sampling in the collector
trace.WithSampler(
    trace.ParentBased(trace.TraceIDRatioBased(0.10)), // follow parent; 10% for new traces
)
```

**OTEL Collector tail sampling (captures all slow + error traces):**
```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      - name: always-sample-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: always-sample-slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic-baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
```

Checklist:
- [ ] `ParentBased` sampler used in services — respects upstream sampling decisions
- [ ] Tail-based sampling configured in the OTEL Collector — captures 100% of errors and slow requests
- [ ] Sampling rate documented and reviewed against trace backend costs
- [ ] Health check and metrics scrape spans excluded from sampling

---

## Step 7 — Connect Traces to Logs and Metrics

**Correlate traces with logs — inject `trace_id` and `span_id` into every log line:**
```go
func logFromSpan(ctx context.Context, msg string) {
    span := trace.SpanFromContext(ctx)
    sc := span.SpanContext()
    slog.InfoContext(ctx, msg,
        slog.String("trace_id", sc.TraceID().String()),
        slog.String("span_id", sc.SpanID().String()),
    )
}
```

**Grafana — exemplar linking (histogram → trace):**
```go
// Attach trace ID as a Prometheus/OTEL exemplar on latency histograms
histogram.Record(ctx, durationMs,
    metric.WithAttributeSet(attribute.NewSet(
        attribute.String("trace_id", trace.SpanFromContext(ctx).SpanContext().TraceID().String()),
    )),
)
```

Checklist:
- [ ] `trace_id` and `span_id` present in every log line — enables jump from log to trace in UI
- [ ] Log aggregator (Loki, Datadog, CloudWatch) configured to parse and index `trace_id`
- [ ] Metrics histograms emit exemplars with `trace_id` — enables drill-down from dashboard to trace
- [ ] Unified search available: log query → click trace_id → open trace in Jaeger/Tempo

---

## Output Report

### Critical
- No trace context propagated across service boundaries — traces are single-service islands; cross-service debugging impossible
- Sensitive data (tokens, card numbers) stored in span attributes — visible to anyone with tracing backend access

### High
- Context (`ctx`) not threaded through function calls — spans complete immediately, recording no meaningful duration
- `span.End()` missing on error paths — spans left open, corrupting the trace and leaking resources
- `AlwaysOn` sampler in production at high traffic — trace backend costs explode

### Medium
- No tail-based sampling — error traces dropped at the same rate as healthy traces; incidents miss traces
- Span names contain user IDs or order IDs — high-cardinality span names break trace indexers
- `trace_id` not injected into logs — traces and logs cannot be correlated during incidents

### Low
- Manual header propagation instead of OTEL middleware — fragile, inconsistent, likely to break on library updates
- Span events missing — state transitions and retries invisible in the trace timeline
- No attribute semantic convention — each service uses different field names for the same concept

### Passed
- OTEL SDK initialized with correct resource attributes; TracerProvider gracefully shuts down
- Auto-instrumentation covers HTTP, database, and messaging; manual spans added for business logic
- Trace context propagated via W3C traceparent through HTTP and message queue boundaries
- Tail-based sampling in OTEL Collector captures 100% of errors and slow traces; 10% baseline otherwise
- `trace_id` injected into all log lines; exemplars link metrics dashboards to traces

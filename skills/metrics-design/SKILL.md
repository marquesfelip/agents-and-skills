---
name: metrics-design
description: >
  Service metrics design, instrumentation, and monitoring for observable production systems.
  Use when: metrics design, instrumentation, Prometheus metrics, OTEL metrics, service metrics,
  counter metric, gauge metric, histogram metric, summary metric, metric naming, metric labels,
  cardinality, Grafana dashboard, metric instrumentation, RED metrics, USE metrics, DORA metrics,
  throughput rate error duration, request rate, error rate, latency histogram, p95 latency, p99 latency,
  business metrics, SLI metrics, metric review, Datadog metrics, CloudWatch metrics, InfluxDB,
  metric aggregation, metric alert, recording rules, metric cardinality explosion, Go prometheus,
  promhttp, OpenTelemetry metrics SDK, counter increment, histogram observe, metric exemplar.
argument-hint: >
  Describe the service and its critical operations (e.g., "Go HTTP API serving order creation,
  inventory checks, and payment processing"), the metrics backend (Prometheus + Grafana, Datadog,
  OTEL Collector), and which metrics are already in place (if any).
---

# Metrics Design Specialist

## When to Use

Invoke this skill when you need to:
- Design a metrics instrumentation plan for a service or system
- Choose the right metric types (counter, gauge, histogram, summary)
- Define metric naming conventions and label schemas
- Apply the RED and USE methods to cover standard operational signals
- Add business and domain metrics beyond infrastructure defaults
- Prevent label cardinality explosions

---

## Step 1 — Choose Metric Types

| Type | Description | Use For | Example |
|---|---|---|---|
| **Counter** | Monotonically increasing integer | Events that accumulate | Requests total, errors total, bytes sent |
| **Gauge** | Instantaneous value, can go up or down | Current state | Active connections, queue depth, memory usage |
| **Histogram** | Samples distributed into configurable buckets | Latency, request sizes | Request duration in seconds |
| **Summary** | Pre-computed quantiles on the client | When server-side aggregation is impossible | (prefer histograms; summaries cannot be aggregated across replicas) |

**Decision guide:**
- "How many times did X happen?" → **Counter**
- "How many of X exist right now?" → **Gauge**
- "How long does X take?" → **Histogram**
- Never use **Summary** in new code — it cannot be aggregated across multiple service instances

---

## Step 2 — Apply the RED Method (Request-Oriented Services)

RED covers the most important signals for any service that handles requests.

| Signal | Metric | PromQL |
|---|---|---|
| **R**ate | Requests per second | `rate(http_requests_total[5m])` |
| **E**rrors | Error rate (4xx + 5xx) | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| **D**uration | p50, p95, p99 latency | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |

**Go — Prometheus implementation:**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "orders",
        Subsystem: "http",
        Name:      "requests_total",
        Help:      "Total number of HTTP requests by method, route, and status code.",
    }, []string{"method", "route", "status_code"})

    httpRequestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "orders",
        Subsystem: "http",
        Name:      "request_duration_seconds",
        Help:      "HTTP request latency in seconds.",
        Buckets:   []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
    }, []string{"method", "route"})
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)
        duration := time.Since(start).Seconds()

        route := chi.RouteContext(r.Context()).RoutePattern() // use template, not actual path
        httpRequestsTotal.WithLabelValues(r.Method, route, strconv.Itoa(rw.status)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, route).Observe(duration)
    })
}
```

Checklist:
- [ ] Rate, Errors, and Duration instrumented for every external-facing service
- [ ] Histogram buckets cover the expected latency distribution — not just the defaults
- [ ] Route label uses the path template (`/orders/{id}`), never the actual path (`/orders/123`)
- [ ] Status code grouped at counter level (`2xx`, `4xx`, `5xx`) OR kept as raw integer for detail

---

## Step 3 — Apply the USE Method (Resource-Oriented Services)

USE covers infrastructure and resource-level signals. Often provided by node exporters, but custom services with resource workers need it too.

| Signal | Metric | Example |
|---|---|---|
| **U**tilization | Fraction of time resource is busy | Worker pool utilization: active/total |
| **S**aturation | Queue depth or backpressure | Job queue depth, pending tasks |
| **E**rrors | Resource error rate | Connection errors, disk I/O errors |

**Worker pool example:**
```go
var (
    workerPoolActive = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "orders",
        Subsystem: "worker",
        Name:      "active_total",
        Help:      "Number of currently active worker goroutines.",
    })
    workerPoolCapacity = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "orders",
        Subsystem: "worker",
        Name:      "capacity_total",
        Help:      "Total worker pool capacity.",
    })
    jobQueueDepth = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "orders",
        Subsystem: "queue",
        Name:      "depth",
        Help:      "Current depth of the job queue by type.",
    }, []string{"queue_name"})
)
```

Checklist:
- [ ] Worker pool utilization (active/capacity) exposed as a ratio gauge
- [ ] Queue depth exposed as a gauge — critical for saturation alerting
- [ ] Connection pool: active, idle, and waiting connections tracked
- [ ] Database and external dependency error rates tracked separately from application errors

---

## Step 4 — Metric Naming Conventions

**Pattern:** `<namespace>_<subsystem>_<name>_<unit>`

| Part | Rule | Example |
|---|---|---|
| `namespace` | Service or team name | `orders`, `payments`, `platform` |
| `subsystem` | Component within the service | `http`, `db`, `worker`, `cache` |
| `name` | What is being measured | `requests`, `duration`, `errors`, `queue_depth` |
| `unit` | Always include for dimensional metrics | `_seconds`, `_bytes`, `_total` |

**Rules:**
- Counters end in `_total` (Prometheus convention)
- Duration metrics use `_seconds` — never milliseconds as the base unit
- Boolean states use a gauge with value 0 or 1: `feature_flag_enabled{flag="dark_mode"} 1`
- No abbreviations: `req` → `requests`, `dur` → `duration`, `err` → `errors`
- Lowercase snake_case only — no camelCase or hyphens

**Good vs bad naming:**
```
✅ orders_http_request_duration_seconds
✅ orders_db_query_errors_total
✅ orders_worker_active_total
✅ orders_cache_hits_total

❌ orderReqDur           — camelCase, no unit
❌ orders_http_ms        — wrong unit suffix
❌ orders_err            — too vague, no subsystem or unit
❌ orders_http_requests  — counter missing _total
```

---

## Step 5 — Label Design and Cardinality Control

Labels enable slice-and-dice but cause cardinality explosion when misused.

**Safe (low-cardinality) labels:**
- `method`: `GET`, `POST`, `PUT`, `DELETE` (5 values)
- `route`: `/orders`, `/orders/{id}` (bounded by route count)
- `status_code`: `200`, `201`, `400`, `500` (bounded)
- `queue_name`: `orders`, `payments` (bounded by queue count)
- `env`: `production`, `staging` (2-3 values)
- `region`: `us-east-1`, `eu-west-1` (bounded)

**Dangerous (high-cardinality) labels — never use:**
- `user_id` — millions of distinct values → millions of time series
- `order_id`, `request_id`, `session_id` — unbounded
- Raw URL paths (`/orders/12345`) — one time series per unique ID
- Timestamps as labels

**Cardinality estimation:**
```
Total time series = product of all label value counts
example: method(5) × route(20) × status_code(10) = 1,000 series   ✅ safe
example: method(5) × user_id(1,000,000)           = 5,000,000 series ❌ explosion
```

Checklist:
- [ ] All label values are bounded — no user IDs, order IDs, or request IDs as labels
- [ ] Route label uses the path template — middleware extracts it, never the raw request path
- [ ] Total time series per service estimated before deployment
- [ ] Prometheus cardinality limit configured (`--storage.tsdb.max-block-chunk-seg-size`)
- [ ] `cardinality` endpoint (`/api/v1/status/tsdb`) monitored for unexpected growth

---

## Step 6 — Business and Domain Metrics

Infrastructure metrics tell you HOW the system is behaving. Business metrics tell you WHAT is happening.

**Examples:**
```go
var (
    ordersCreatedTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "orders", Name: "created_total",
        Help: "Total orders created by payment method and customer tier.",
    }, []string{"payment_method", "customer_tier"})

    orderValueHistogram = promauto.NewHistogram(prometheus.HistogramOpts{
        Namespace: "orders", Name: "value_dollars",
        Help:    "Distribution of order values in USD.",
        Buckets: []float64{10, 25, 50, 100, 250, 500, 1000, 5000},
    })

    checkoutFunnelSteps = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "orders", Name: "checkout_funnel_total",
        Help: "Count of users reaching each step of the checkout funnel.",
    }, []string{"step"}) // "cart_viewed", "address_entered", "payment_submitted", "confirmed"

    activeSubscriptions = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "billing", Name: "active_subscriptions",
        Help: "Current count of active subscriptions by plan.",
    }, []string{"plan"})
)
```

Checklist:
- [ ] At least one business metric per critical user journey instrumented
- [ ] Revenue-impacting operations (orders created, payments processed) have dedicated counters
- [ ] Funnel metrics expose where users drop off — not just whether a step succeeded
- [ ] Business metric dashboards owned by the product/engineering team — not just SRE

---

## Step 7 — Expose and Scrape Metrics

**Go — expose Prometheus metrics endpoint:**
```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

mux.Handle("/metrics", promhttp.Handler())
```

**Kubernetes — annotate pods for Prometheus scraping:**
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

**Prometheus scrape config:**
```yaml
scrape_configs:
  - job_name: orders-service
    scrape_interval: 15s
    static_configs:
      - targets: ["orders-service:8080"]
```

**Recording rules (pre-compute expensive queries):**
```yaml
groups:
  - name: orders_recording_rules
    rules:
      - record: orders:http_request_rate5m
        expr: rate(orders_http_requests_total[5m])
      - record: orders:http_error_rate5m
        expr: rate(orders_http_requests_total{status_code=~"5.."}[5m]) / rate(orders_http_requests_total[5m])
      - record: orders:http_p99_latency5m
        expr: histogram_quantile(0.99, rate(orders_http_request_duration_seconds_bucket[5m]))
```

Checklist:
- [ ] `/metrics` endpoint exposed and accessible to Prometheus scraper
- [ ] Kubernetes pod annotations set — auto-discovered by Prometheus Operator
- [ ] Recording rules defined for frequently-queried aggregations — reduces dashboard query time
- [ ] `/metrics` endpoint not exposed publicly — firewall or network policy restricts to internal scrape

---

## Output Report

### Critical
- User IDs, order IDs, or request IDs used as metric labels — cardinality explosion that crashes the metrics backend
- Raw request paths used as the route label — one time series per unique URL destroys the system

### High
- No duration histogram for critical endpoints — SLO breaches invisible until users complain
- Histogram buckets not calibrated — all requests fall in the last bucket; quantile estimates meaningless
- Summary metric used instead of histogram — cannot be aggregated across multiple instances

### Medium
- No business metrics — dashboards show infrastructure health but not business impact
- Metric names violate conventions — teams cannot discover or share metrics across services
- Queue depth not instrumented — saturation goes undetected until the queue backs up fatally

### Low
- No recording rules for expensive aggregations — dashboard queries time out at high scale
- `/metrics` endpoint accessible without network restriction — internal metrics details exposed externally
- Worker pool utilization not tracked — overload detected only via cascading failures

### Passed
- RED signals (Rate, Errors, Duration) instrumented for all external-facing operations
- USE signals (Utilization, Saturation, Errors) instrumented for all resource workers and pools
- Metric naming follows `<namespace>_<subsystem>_<name>_<unit>` convention consistently
- All labels are low-cardinality; route labels use path templates, not actual paths
- Business metrics instrumented for critical user journeys and revenue-impacting operations
- Recording rules defined for SLO-critical queries; `/metrics` endpoint restricted to internal access

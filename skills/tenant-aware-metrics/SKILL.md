---
name: tenant-aware-metrics
description: 'Per-tenant metrics monitoring for SaaS systems. Use when: tenant-aware metrics, per-tenant metrics, tenant metric label, tenant Prometheus, tenant Grafana, tenant metric cardinality, tenant counter, tenant histogram, tenant gauge, tenant metric aggregation, tenant usage metric, tenant error rate, tenant latency, multi-tenant observability, tenant metric dashboard, tenant SLO metric, tenant billing metric, metric per tenant, tenant request rate, tenant quota metric, tenant seat metric, tenant storage metric, metric tenant isolation, cardinality explosion per tenant.'
argument-hint: 'Describe the metrics backend (Prometheus, Datadog, CloudWatch, InfluxDB), whether the tenant count is small (<100) or large (>1000 — cardinality risk), which tenant-level signals matter most (error rate, latency, quota usage, billing events), and the alerting tool (Alertmanager, PagerDuty, Grafana).'
---

# Tenant-Aware Metrics

## When to Use

Invoke this skill when you need to:
- Add `tenant_id` as a Prometheus label to track error rates, latencies, and request counts per tenant
- Design label cardinality safely — avoid one label value per user (cardinality explosion)
- Build Grafana dashboards that allow drilling down to a specific tenant
- Define SLO metrics at both the platform level and the per-tenant level
- Alert when a single tenant drives disproportionate resource usage (noisy neighbor)
- Track quota consumption metrics, seat counts, and billing-relevant signals per tenant

---

## Cardinality Strategy

```
Tenant count    Prometheus label strategy
─────────────   ───────────────────────────────────────────────────────
< 100           Safe to use tenant_id as a label — full cardinality OK
100 – 1 000     Use tenant_id only on critical metrics; aggregate others
> 1 000         Do NOT use tenant_id as a Prometheus label
                → Write per-tenant aggregates to a separate TSDB or DB
                → Use Loki LogQL for per-tenant error queries instead
```

---

## Step 1 — Metric Registration with Tenant Label

```go
package metrics

import "github.com/prometheus/client_golang/prometheus"

var (
    // HTTP request rate per tenant — only viable if tenant count < 1000
    TenantRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "saas_tenant_requests_total",
            Help: "Total HTTP requests per tenant and status class.",
        },
        []string{"tenant_id", "status_class", "endpoint"},
    )

    // Tenant error rate
    TenantErrorsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "saas_tenant_errors_total",
            Help: "Total errors per tenant and error category.",
        },
        []string{"tenant_id", "error_category"},
    )

    // Quota consumption — gauge showing % of plan limit used
    TenantQuotaUsage = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "saas_tenant_quota_usage_ratio",
            Help: "Current quota usage ratio (0.0–1.0) per tenant and resource.",
        },
        []string{"tenant_id", "resource"}, // resource: "seats" | "api_calls" | "storage_gb"
    )

    // Request latency histogram — without tenant label (too high cardinality)
    // Use exemplars to link high-latency requests to tenant_id
    RequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "saas_http_request_duration_seconds",
            Help:    "HTTP request latency.",
            Buckets: prometheus.DefBuckets,
        },
        []string{"endpoint", "method", "status_class"},
    )
)

func init() {
    prometheus.MustRegister(
        TenantRequestsTotal,
        TenantErrorsTotal,
        TenantQuotaUsage,
        RequestDuration,
    )
}
```

---

## Step 2 — HTTP Middleware: Emit Per-Tenant Metrics

```go
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(rw, r)

        duration := time.Since(start).Seconds()
        statusClass := fmt.Sprintf("%dxx", rw.statusCode/100)
        endpoint := routePattern(r) // e.g., "/api/subscriptions/{id}"

        // Platform-level latency (no tenant label — safe at any scale)
        metrics.RequestDuration.WithLabelValues(endpoint, r.Method, statusClass).
            Observe(duration)

        // Per-tenant request count (only if tenant count < 1000)
        tenantID := logctx.TenantID(r.Context())
        if tenantID != "" {
            metrics.TenantRequestsTotal.WithLabelValues(tenantID, statusClass, endpoint).Inc()

            if rw.statusCode >= 500 {
                metrics.TenantErrorsTotal.WithLabelValues(tenantID, "server_error").Inc()
            }
        }
    })
}
```

---

## Step 3 — Quota Usage Metrics (Periodic Export)

```go
// QuotaMetricsExporter runs periodically and exports current quota usage for all tenants
type QuotaMetricsExporter struct {
    quotaRepo QuotaRepository
    interval  time.Duration
}

func (e *QuotaMetricsExporter) Run(ctx context.Context) {
    ticker := time.NewTicker(e.interval) // e.g., every 60 seconds
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            e.export(ctx)
        }
    }
}

func (e *QuotaMetricsExporter) export(ctx context.Context) {
    usages, err := e.quotaRepo.GetAllCurrentUsage(ctx)
    if err != nil {
        slog.Error("quota metrics export failed", "err", err)
        return
    }
    for _, u := range usages {
        ratio := float64(u.Used) / float64(u.Limit)
        metrics.TenantQuotaUsage.WithLabelValues(u.TenantID, u.Resource).Set(ratio)

        // Alert signal: emit high-cardinality data to a separate store
        // (do not add >1000 unique tenant labels to Prometheus)
        if ratio >= 0.9 {
            slog.Warn("tenant quota near limit",
                "tenant_id", u.TenantID,
                "resource", u.Resource,
                "used", u.Used,
                "limit", u.Limit,
                "ratio", ratio,
            )
        }
    }
}
```

---

## Step 4 — Large-Scale Alternative: Per-Tenant Aggregates in PostgreSQL

```sql
-- For > 1000 tenants: write usage snapshots to DB instead of Prometheus labels
CREATE TABLE tenant_metric_snapshots (
    id           BIGSERIAL   PRIMARY KEY,
    tenant_id    UUID        NOT NULL,
    metric_name  TEXT        NOT NULL,   -- 'request_count' | 'error_count' | 'api_calls'
    value        BIGINT      NOT NULL,
    period_start TIMESTAMPTZ NOT NULL,
    period_end   TIMESTAMPTZ NOT NULL,
    recorded_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_metrics ON tenant_metric_snapshots(tenant_id, metric_name, period_start);
CREATE INDEX idx_tenant_metrics_window ON tenant_metric_snapshots(period_start, metric_name);
```

```go
// Aggregate counter: increment in Redis, flush to DB on schedule
func (s *MetricsSink) IncrRequest(ctx context.Context, tenantID string) {
    key := fmt.Sprintf("metrics:req:%s:%s", tenantID, currentHour())
    s.redis.Incr(ctx, key)
    s.redis.Expire(ctx, key, 25*time.Hour)
}

func (s *MetricsSink) Flush(ctx context.Context) {
    // Scan all tenant metric keys, write to DB, reset counters
    keys, _ := s.redis.Keys(ctx, "metrics:req:*").Result()
    for _, key := range keys {
        val, _ := s.redis.GetDel(ctx, key).Int64()
        parts := strings.Split(key, ":")
        tenantID, hour := parts[2], parts[3]
        s.db.UpsertMetricSnapshot(ctx, tenantID, "request_count", val, hour)
    }
}
```

---

## Step 5 — Grafana Dashboard Design

```
Panel: Tenant Request Rate (top 10 noisy tenants)
  PromQL: topk(10, sum by (tenant_id) (rate(saas_tenant_requests_total[5m])))

Panel: Tenant Error Rate % (single tenant drill-down)
  PromQL: rate(saas_tenant_errors_total{tenant_id="$tenant"}[5m])
           / rate(saas_tenant_requests_total{tenant_id="$tenant"}[5m])

Panel: Quota Usage by Resource (gauge panel)
  PromQL: saas_tenant_quota_usage_ratio{tenant_id="$tenant"}
  Threshold: 0.8 = yellow, 0.95 = red

Panel: Noisy Neighbors (tenants above 10% of total traffic)
  PromQL: sum by (tenant_id) (rate(saas_tenant_requests_total[5m]))
           / scalar(sum(rate(saas_tenant_requests_total[5m]))) > 0.1
```

---

## Metric Design Reference

| Signal | Metric type | Labels | Cardinality note |
|---|---|---|---|
| Request count | Counter | tenant_id, status_class, endpoint | Safe if < 1000 tenants |
| Error count | Counter | tenant_id, error_category | Safe if < 1000 tenants |
| Latency | Histogram | endpoint, method, status_class | No tenant label — use exemplars |
| Quota usage ratio | Gauge | tenant_id, resource | Safe if < 1000 tenants |
| Seat count | Gauge | tenant_id | Safe if < 1000 tenants |
| Storage bytes used | Gauge | tenant_id | Use DB snapshot for scale |
| Billing events | Counter | plan_id, event_type | No tenant label |

---

## Quality Checks

- [ ] Tenant count verified before adding `tenant_id` as a Prometheus label — budget cardinality
- [ ] Latency histograms do NOT include `tenant_id` label — use exemplars for drill-down instead
- [ ] Quota usage exported on a fixed schedule (not on every request) — avoids label churn
- [ ] Grafana dashboard has a `$tenant` variable for single-tenant drill-down
- [ ] Alert defined: `saas_tenant_quota_usage_ratio > 0.95` for any tenant
- [ ] Alert defined: single tenant driving > 20% of total request rate (noisy neighbor)
- [ ] For > 1000 tenants: per-tenant aggregates go to DB or Datadog custom metrics — not Prometheus

## After Completion

- Use **`tenant-aware-logging`** for per-tenant log-based error analysis complementing metrics
- Use **`alerting-strategy`** to configure quota and noisy-neighbor alert routing
- Use **`sla-slo-sli-design`** to define per-tenant SLOs built on these metrics
- Use **`usage-anomaly-detection`** to detect abnormal per-tenant usage patterns

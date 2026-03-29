---
name: usage-analytics-architecture
description: 'Usage analytics collection architecture for SaaS products. Use when: usage analytics, usage data collection, usage event tracking, product analytics, feature usage tracking, usage pipeline, analytics architecture, event collection pipeline, usage funnel, feature adoption, usage retention analytics, analytics ingestion, clickstream, usage event schema, analytics data warehouse, SaaS product metrics, DAU MAU, feature engagement, usage event bus, analytics sink, BigQuery analytics, Redshift analytics, usage tracking pipeline, usage data model, analytics event, product usage metrics.'
argument-hint: 'Describe the types of usage events to track (feature clicks, API calls, file uploads, report views), the expected event volume, the analytics destination (BigQuery, Redshift, ClickHouse, Mixpanel, PostHog), and whether the analytics pipeline must be real-time or batch is acceptable.'
---

# Usage Analytics Architecture

## When to Use

Invoke this skill when you need to:
- Design the pipeline that captures product usage events (feature clicks, API calls, page views) without impacting application latency
- Define a canonical analytics event schema that works across frontend, backend, and mobile
- Route events to a data warehouse (BigQuery, Redshift, ClickHouse) or product analytics tool (Mixpanel, PostHog)
- Track feature adoption, DAU/MAU, funnel completion, and retention cohorts
- Separate the analytics write path from the application hot path — analytics must never slow down the product
- Model usage data per tenant for product-led growth signals and billing (metered features)

---

## Analytics Pipeline Architecture

```
Application (backend/frontend)
        │
        │ fire-and-forget event
        ▼
  Analytics Event Bus
  (Kafka | SQS | Redis Stream | HTTP batch)
        │
   ┌────┴─────┐
   │          │
   ▼          ▼
Real-time   Batch sink
consumer    (hourly/nightly)
(alerts,      │
 quotas)      ▼
        Data Warehouse
        (BigQuery / Redshift / ClickHouse)
              │
              ▼
        BI Tool / Dashboard
        (Metabase, Grafana, Looker)
```

---

## Step 1 — Canonical Analytics Event Schema

```go
// AnalyticsEvent is the canonical event envelope for all usage tracking
// This schema is shared across backend services, frontend SDKs, and mobile
type AnalyticsEvent struct {
    // Identity
    EventID    string    `json:"event_id"`    // UUID — dedup key
    EventName  string    `json:"event_name"`  // e.g., "report.viewed", "api.called", "seat.invited"
    OccurredAt time.Time `json:"occurred_at"` // UTC timestamp of the action

    // Tenant context
    TenantID   string `json:"tenant_id"`
    TenantPlan string `json:"tenant_plan"` // "free" | "starter" | "pro" | "enterprise"

    // User context (optional — background jobs may not have a user)
    UserID     string `json:"user_id,omitempty"`
    UserRole   string `json:"user_role,omitempty"` // "owner" | "admin" | "member"

    // Session context (frontend events only)
    SessionID  string `json:"session_id,omitempty"`
    PageURL    string `json:"page_url,omitempty"`
    Referrer   string `json:"referrer,omitempty"`

    // Event-specific properties (flexible per event type)
    Properties map[string]any `json:"properties,omitempty"`

    // Source
    Source     string `json:"source"` // "backend" | "frontend" | "mobile"
    AppVersion string `json:"app_version,omitempty"`
}

// Event name conventions: {noun}.{verb} or {feature}.{action}
// Examples:
//   "subscription.activated"    — billing event
//   "report.exported"           — feature usage
//   "api_key.created"           — admin action
//   "file.uploaded"             — engagement
//   "dashboard.viewed"          — engagement
//   "seat.invited"              — growth signal
```

---

## Step 2 — Backend Event Emission (Fire-and-Forget)

```go
// AnalyticsClient sends events to the pipeline without blocking the caller
type AnalyticsClient interface {
    Track(ctx context.Context, event AnalyticsEvent)
}

// AsyncClient buffers events in memory and flushes in batches
type AsyncAnalyticsClient struct {
    queue  chan AnalyticsEvent
    sink   AnalyticsSink
    logger *slog.Logger
}

func NewAsyncClient(sink AnalyticsSink, bufferSize int) *AsyncAnalyticsClient {
    c := &AsyncAnalyticsClient{
        queue: make(chan AnalyticsEvent, bufferSize), // non-blocking buffer
        sink:  sink,
    }
    go c.flush()
    return c
}

// Track is non-blocking: drops the event if buffer is full (analytics must not block the product)
func (c *AsyncAnalyticsClient) Track(ctx context.Context, event AnalyticsEvent) {
    event.EventID = uuid.New().String()
    event.OccurredAt = time.Now().UTC()
    event.TenantID = logctx.TenantID(ctx)
    event.UserID = logctx.UserID(ctx)

    select {
    case c.queue <- event:
        // enqueued
    default:
        // Buffer full — drop and log. Analytics loss is acceptable; product latency is not.
        c.logger.Warn("analytics event dropped — buffer full", "event_name", event.EventName)
    }
}

func (c *AsyncAnalyticsClient) flush() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    batch := make([]AnalyticsEvent, 0, 100)
    for {
        select {
        case event := <-c.queue:
            batch = append(batch, event)
            if len(batch) >= 100 {
                c.sink.WriteBatch(context.Background(), batch)
                batch = batch[:0]
            }
        case <-ticker.C:
            if len(batch) > 0 {
                c.sink.WriteBatch(context.Background(), batch)
                batch = batch[:0]
            }
        }
    }
}
```

---

## Step 3 — Usage Event Tracking in Service Layer

```go
// Track feature usage after the business operation succeeds — never before
func (s *ReportService) Export(ctx context.Context, reportID uuid.UUID, format string) ([]byte, error) {
    data, err := s.repo.GetReportData(ctx, reportID)
    if err != nil {
        return nil, err
    }

    output, err := s.renderer.Render(data, format)
    if err != nil {
        return nil, err
    }

    // Track AFTER success — do not track failed operations as usage
    s.analytics.Track(ctx, AnalyticsEvent{
        EventName: "report.exported",
        Properties: map[string]any{
            "report_id": reportID,
            "format":    format,
            "row_count": len(data.Rows),
        },
    })

    return output, nil
}
```

---

## Step 4 — Analytics Sink: BigQuery

```go
// BigQuerySink writes batches to a BigQuery streaming insert
type BigQuerySink struct {
    inserter *bigquery.Inserter
}

type bqRow struct {
    EventID    string    `bigquery:"event_id"`
    EventName  string    `bigquery:"event_name"`
    OccurredAt time.Time `bigquery:"occurred_at"`
    TenantID   string    `bigquery:"tenant_id"`
    TenantPlan string    `bigquery:"tenant_plan"`
    UserID     string    `bigquery:"user_id"`
    Properties string    `bigquery:"properties"` // JSON string — flexible querying via JSON_EXTRACT
    Source     string    `bigquery:"source"`
}

func (s *BigQuerySink) WriteBatch(ctx context.Context, events []AnalyticsEvent) error {
    rows := make([]*bqRow, 0, len(events))
    for _, e := range events {
        propsJSON, _ := json.Marshal(e.Properties)
        rows = append(rows, &bqRow{
            EventID:    e.EventID,
            EventName:  e.EventName,
            OccurredAt: e.OccurredAt,
            TenantID:   e.TenantID,
            TenantPlan: e.TenantPlan,
            UserID:     e.UserID,
            Properties: string(propsJSON),
            Source:     e.Source,
        })
    }
    if err := s.inserter.Put(ctx, rows); err != nil {
        slog.Error("BigQuery analytics write failed", "count", len(rows), "err", err)
        return err
    }
    return nil
}
```

---

## Step 5 — Key Analytics Queries

```sql
-- DAU per tenant (last 30 days)
SELECT
    DATE(occurred_at) AS date,
    tenant_id,
    COUNT(DISTINCT user_id) AS dau
FROM analytics_events
WHERE occurred_at >= CURRENT_DATE - INTERVAL '30 days'
  AND user_id IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- Feature adoption: % of tenants that used each feature in the last 30 days
SELECT
    event_name,
    COUNT(DISTINCT tenant_id) AS tenants_used,
    COUNT(DISTINCT tenant_id) * 100.0 / (SELECT COUNT(DISTINCT tenant_id) FROM tenants WHERE status = 'active') AS adoption_pct
FROM analytics_events
WHERE occurred_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY event_name
ORDER BY tenants_used DESC;

-- Retention cohort: tenants active in week 1 who returned in week 4
WITH week1 AS (
    SELECT DISTINCT tenant_id FROM analytics_events
    WHERE occurred_at BETWEEN '2026-01-01' AND '2026-01-07'
),
week4 AS (
    SELECT DISTINCT tenant_id FROM analytics_events
    WHERE occurred_at BETWEEN '2026-01-22' AND '2026-01-28'
)
SELECT COUNT(w4.tenant_id)::float / COUNT(w1.tenant_id) AS week4_retention
FROM week1 w1 LEFT JOIN week4 w4 USING (tenant_id);
```

---

## Sink Selection Guide

| Volume | Latency need | Recommended sink |
|---|---|---|
| < 1M events/day | Batch OK | PostgreSQL partitioned table |
| 1M–100M events/day | Near real-time | BigQuery streaming insert or ClickHouse |
| > 100M events/day | Real-time | Kafka → ClickHouse or Kafka → BigQuery |
| Need product analytics UI | Any | PostHog self-hosted or Mixpanel |
| Need SQL only | Batch OK | S3 Parquet → Athena / Redshift Spectrum |

---

## Quality Checks

- [ ] `Track()` is non-blocking — uses buffered channel with DROP on full (no `chan <- event` without `select`)
- [ ] Events are tracked AFTER the business operation succeeds — not on attempt
- [ ] `event_id` (UUID) present on every event — enables deduplication in the warehouse
- [ ] No PII in `properties`: no raw email, name, or payment data
- [ ] Event names follow `{noun}.{verb}` convention and are documented in a registry
- [ ] Analytics sink failures are logged but do NOT return error to the caller
- [ ] Warehouse table is partitioned by `occurred_at` (date) — queries over time windows are fast

## After Completion

- Use **`tenant-aware-metrics`** for real-time operational signals alongside batch analytics
- Use **`usage-anomaly-detection`** to build alerts on top of the analytics pipeline
- Use **`quota-management`** to use usage analytics as the source for metered billing counters
- Use **`abuse-prevention-system`** to detect suspicious usage patterns in the analytics stream

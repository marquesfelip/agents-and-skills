---
name: usage-anomaly-detection
description: 'Detection of abnormal usage or abuse patterns in SaaS systems. Use when: usage anomaly detection, abnormal usage, usage spike detection, API abuse detection, usage outlier, anomaly alert, unusual usage pattern, traffic anomaly, request spike anomaly, tenant usage anomaly, per-tenant anomaly, rate spike, sudden usage increase, usage deviation, behavioral anomaly, statistical anomaly, Z-score usage, moving average anomaly, usage threshold alert, anomaly detection SaaS, product analytics anomaly, API call spike, storage anomaly, feature abuse detection, velocity anomaly.'
argument-hint: 'Describe the usage signals to monitor (API call rate, storage growth, file uploads, login attempts, exports), the tenant scale (number of active tenants), whether anomalies should trigger automated action (block, throttle) or just alert, and the latency requirement for detection (real-time vs hourly batch).'
---

# Usage Anomaly Detection

## When to Use

Invoke this skill when you need to:
- Detect a tenant suddenly making 10× their normal API call volume (scraping, infinite loop, abuse)
- Alert when a tenant's storage usage grows abnormally fast within a short window
- Identify tenants whose usage pattern deviates significantly from their historical baseline
- Distinguish between legitimate growth and anomalous bursts that may indicate abuse or runaway automation
- Implement a detection pipeline that evaluates signals in near-real-time or on a rolling schedule
- Route detected anomalies to alerts, automated throttling, or a manual review queue

---

## Detection Strategies Matrix

| Signal | Method | Latency | Use case |
|---|---|---|---|
| API call rate spike | Sliding window counter vs rolling average | Real-time | Detect runaway loops, scraping |
| Storage growth rate | Hourly snapshot diff | Near real-time (1h) | Detect mass upload abuse |
| Failed auth attempts | Count in 10-min window | Real-time | Account takeover detection |
| Export frequency | Daily count vs historical avg | Daily | Data exfiltration detection |
| Seat invitation rate | Count in 1-hour window | Real-time | Referral abuse, spam |
| Unusual geographic access | First-seen country in JWT | Per-request | Account takeover signal |
| Feature usage pattern | Z-score vs tenant cohort | Hourly batch | Broad behavioral anomaly |

---

## Step 1 — Sliding Window Rate Anomaly (Real-Time)

```go
// SlidingWindowAnomalyDetector detects when a tenant's request rate
// in the last N minutes exceeds K× their rolling 7-day average

type SlidingWindowDetector struct {
    redis     *redis.Client
    threshold float64 // multiplier: e.g., 5.0 = 5× normal rate
}

func (d *SlidingWindowDetector) RecordAndCheck(ctx context.Context, tenantID string, endpoint string) (bool, error) {
    now := time.Now().UTC()
    minuteKey := fmt.Sprintf("usage:rate:%s:%s:%s", tenantID, endpoint, now.Format("2006-01-02T15:04"))
    dailyKey  := fmt.Sprintf("usage:daily:%s:%s:%s", tenantID, endpoint, now.Format("2006-01-02"))

    pipe := d.redis.Pipeline()
    pipe.Incr(ctx, minuteKey)
    pipe.Expire(ctx, minuteKey, 10*time.Minute)
    pipe.Incr(ctx, dailyKey)
    pipe.Expire(ctx, dailyKey, 8*24*time.Hour) // keep 8 days of daily data
    pipe.Exec(ctx)

    // Current rate: sum of last 5 minutes
    var currentRate int64
    for i := 0; i < 5; i++ {
        t := now.Add(-time.Duration(i) * time.Minute)
        key := fmt.Sprintf("usage:rate:%s:%s:%s", tenantID, endpoint, t.Format("2006-01-02T15:04"))
        val, _ := d.redis.Get(ctx, key).Int64()
        currentRate += val
    }

    // Historical average: sum of same 5-minute windows over past 7 days
    var historicalSum int64
    dayCount := 7
    for day := 1; day <= dayCount; day++ {
        for i := 0; i < 5; i++ {
            t := now.AddDate(0, 0, -day).Add(-time.Duration(i) * time.Minute)
            key := fmt.Sprintf("usage:rate:%s:%s:%s", tenantID, endpoint, t.Format("2006-01-02T15:04"))
            val, _ := d.redis.Get(ctx, key).Int64()
            historicalSum += val
        }
    }
    historicalAvg := float64(historicalSum) / float64(dayCount)

    // Anomaly: current rate is more than threshold× the historical average
    if historicalAvg > 10 && float64(currentRate) > historicalAvg*d.threshold {
        slog.Warn("usage rate anomaly detected",
            "tenant_id", tenantID,
            "endpoint", endpoint,
            "current_5min", currentRate,
            "historical_avg_5min", historicalAvg,
            "multiplier", float64(currentRate)/historicalAvg,
        )
        return true, nil // anomaly detected
    }
    return false, nil
}
```

---

## Step 2 — Z-Score Anomaly (Batch, Hourly)

```go
// Z-score anomaly: detect tenants whose usage is N standard deviations
// above the mean for their plan cohort

type ZScoreDetector struct {
    db *sql.DB
}

type AnomalyResult struct {
    TenantID   string
    Metric     string
    Value      float64
    Mean       float64
    StdDev     float64
    ZScore     float64
    Severity   string // "low" | "medium" | "high"
}

func (d *ZScoreDetector) DetectCohortAnomalies(ctx context.Context, metric string, windowHours int) ([]AnomalyResult, error) {
    // Compare each tenant's usage in the window against the mean/stddev for their plan cohort
    rows, err := d.db.QueryContext(ctx, `
        WITH cohort_stats AS (
            SELECT
                plan_id,
                AVG(metric_value)    AS mean,
                STDDEV(metric_value) AS stddev
            FROM tenant_metric_snapshots
            WHERE metric_name = $1
              AND period_start >= now() - ($2 || ' hours')::interval
            GROUP BY plan_id
        )
        SELECT
            s.tenant_id,
            s.metric_name,
            s.value,
            c.mean,
            c.stddev,
            CASE WHEN c.stddev > 0
                 THEN (s.value - c.mean) / c.stddev
                 ELSE 0
            END AS z_score
        FROM (
            SELECT tenant_id, metric_name,
                   SUM(value) AS value
            FROM tenant_metric_snapshots
            WHERE metric_name = $1
              AND period_start >= now() - ($2 || ' hours')::interval
            GROUP BY tenant_id, metric_name
        ) s
        JOIN tenants t ON t.id = s.tenant_id
        JOIN cohort_stats c ON c.plan_id = t.plan_id
        WHERE CASE WHEN c.stddev > 0
                   THEN (s.value - c.mean) / c.stddev
                   ELSE 0
              END > 2.5  -- flag > 2.5 standard deviations above mean
        ORDER BY z_score DESC
    `, metric, windowHours)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []AnomalyResult
    for rows.Next() {
        var r AnomalyResult
        rows.Scan(&r.TenantID, &r.Metric, &r.Value, &r.Mean, &r.StdDev, &r.ZScore)
        r.Severity = zScoreSeverity(r.ZScore)
        results = append(results, r)
    }
    return results, nil
}

func zScoreSeverity(z float64) string {
    switch {
    case z >= 5.0:
        return "high"
    case z >= 3.5:
        return "medium"
    default:
        return "low"
    }
}
```

---

## Step 3 — Anomaly Alert and Action Table

```go
// AnomalyRouter decides what to do with a detected anomaly
type AnomalyRouter struct {
    alerts   AlertService
    limiter  RateLimiter
    reviewer ReviewQueueService
}

type Anomaly struct {
    TenantID  string
    Signal    string  // "api_rate_spike" | "storage_surge" | "auth_failures"
    Severity  string  // "low" | "medium" | "high"
    ZScore    float64
    Multiplier float64
}

func (r *AnomalyRouter) Handle(ctx context.Context, a Anomaly) {
    slog.Warn("anomaly routed", "tenant", a.TenantID, "signal", a.Signal, "severity", a.Severity)

    switch a.Severity {
    case "low":
        // Log only — no action. Review in weekly anomaly report.
        r.reviewer.RecordForReport(ctx, a)

    case "medium":
        // Alert on-call team + add to review queue
        r.alerts.FireAnomalyAlert(ctx, a)
        r.reviewer.AddToQueue(ctx, a, "manual_review")

    case "high":
        // Alert + throttle the tenant immediately + add to review queue
        r.alerts.FireAnomalyAlert(ctx, a)
        r.limiter.ThrottleTenant(ctx, a.TenantID, 60*time.Minute)
        r.reviewer.AddToQueue(ctx, a, "urgent_review")
        slog.Error("high-severity anomaly: tenant throttled",
            "tenant_id", a.TenantID,
            "signal", a.Signal,
        )
    }
}
```

---

## Step 4 — Anomaly Log Table for Review Queue

```sql
CREATE TYPE anomaly_severity AS ENUM ('low', 'medium', 'high');
CREATE TYPE anomaly_status AS ENUM ('open', 'investigating', 'resolved', 'false_positive');

CREATE TABLE usage_anomalies (
    id           UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID          NOT NULL,
    signal       TEXT          NOT NULL,   -- 'api_rate_spike' | 'storage_surge' | etc.
    severity     anomaly_severity NOT NULL,
    z_score      NUMERIC(8,2),
    multiplier   NUMERIC(8,2), -- e.g., 8.3 = 8.3× normal
    details      JSONB         NOT NULL DEFAULT '{}',
    status       anomaly_status NOT NULL DEFAULT 'open',
    detected_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    resolved_at  TIMESTAMPTZ,
    resolved_by  UUID,         -- operator user_id
    resolution_note TEXT
);

CREATE INDEX idx_anomalies_open ON usage_anomalies(detected_at) WHERE status = 'open';
CREATE INDEX idx_anomalies_tenant ON usage_anomalies(tenant_id, detected_at);
```

---

## Quality Checks

- [ ] Sliding window detector only fires anomaly when `historicalAvg > 10` — prevents false positives on new tenants with zero baseline
- [ ] Z-score computed within plan cohort — free tenants not compared against enterprise usage patterns
- [ ] Low-severity anomalies logged only — no alert fatigue from minor deviations
- [ ] High-severity anomalies trigger automatic throttle AND human review queue — not just an alert
- [ ] `usage_anomalies` table has `false_positive` resolution to calibrate thresholds over time
- [ ] Anomaly detector runs hourly for batch signals; sliding window checked on every N requests for real-time signals
- [ ] Thresholds (Z-score cutoff, rate multiplier) are configurable via env vars — no hardcoded values in code

## After Completion

- Use **`abuse-prevention-system`** to build automated responses on top of anomaly detection
- Use **`tenant-aware-metrics`** as the Prometheus-side counterpart for real-time usage monitoring
- Use **`rate-limiting-strategies`** for the throttling mechanism invoked on high-severity anomalies
- Use **`alerting-strategy`** to configure anomaly alert routing and severity-based escalation

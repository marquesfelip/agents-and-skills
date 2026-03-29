---
name: load-shedding
description: >
  Overload protection and graceful degradation strategies for backend services.
  Use when: load shedding, overload protection, graceful degradation, service overload,
  traffic spike protection, request dropping, priority queuing, admission control,
  concurrency limiting, request queue depth, server overload, capacity protection,
  CPU-based load shedding, latency-based shedding, queue-based admission, token bucket rejection,
  load shedding middleware, HTTP 503 overload, HTTP 429 overload, fail fast, shedding strategy,
  non-critical request dropping, degraded mode, feature degradation, partial availability,
  overload detection, adaptive load shedding, load shedding Go, load shedding policy.
argument-hint: >
  Describe the service and the overload scenario (e.g., "Go HTTP API that collapses under 3×
  normal traffic", "background worker queue that backs up during peak hours"), current behavior
  (crash, timeout cascade, queue backup), and which operations are critical vs. non-critical.
---

# Load Shedding Specialist

## When to Use

Invoke this skill when you need to:
- Protect a service from collapsing under traffic spikes
- Implement graceful degradation — serve partial functionality rather than failing completely
- Define a priority hierarchy for requests (critical vs. non-critical)
- Configure CPU, latency, or queue-depth based admission control
- Design a load shedding policy that preserves the most important user flows
- Prevent cascading failures when one system is overloaded

---

## Step 1 — Understand What to Protect and What to Shed

**The key question:** When the system cannot serve all requests, which ones matter most?

**Define a request priority hierarchy:**

| Priority | Category | Examples | Under Overload |
|---|---|---|---|
| **P0 — Critical** | Revenue-critical or safety | Payment processing, checkout, auth | Never shed |
| **P1 — Important** | Core user actions | Order placement, profile save | Shed last |
| **P2 — Normal** | Standard reads | Product listing, search | Shed at moderate load |
| **P3 — Non-critical** | Analytics, recommendations | "You may also like", tracking events | Shed first |
| **P4 — Background** | Internal jobs | Report generation, email digest | Shed aggressively |

**Rules:**
- Define the priority hierarchy BEFORE a crisis — not during one
- Critical path must work even when the system is shedding 80% of traffic
- Side effects of shedding must be acceptable: P3/P4 requests dropped means analytics data gaps — that is acceptable
- Never shed auth or health checks — they are required for recovery

Checklist:
- [ ] Request priority hierarchy defined and documented
- [ ] Critical paths identified — guaranteed to be served under overload
- [ ] Non-critical endpoints identified — safe to return 503 or degraded response
- [ ] Shedding behavior communicated to product — degraded mode is a deliberate trade-off

---

## Step 2 — Choose a Load Shedding Trigger

Triggering too early wastes capacity. Triggering too late fails to protect the system.

**Trigger options:**

| Trigger | How It Works | Use When |
|---|---|---|
| **Queue depth** | Reject when request queue > N | Service has an explicit request queue |
| **Active request count** | Reject when concurrent requests > N | Compute-bound services |
| **Latency-based** | Reject when p99 latency > threshold | Latency SLO is critical |
| **CPU utilization** | Reject when CPU > 80% | CPU-bound services |
| **Error rate** | Reject when error rate spikes | Indicates downstream overload |
| **Adaptive** | Combine multiple signals | Most robust; avoids single-signal false positives |

**Active request concurrency limiter (Go):**
```go
type ConcurrencyLimiter struct {
    active  atomic.Int64
    maxConc int64
}

func (l *ConcurrencyLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        current := l.active.Add(1)
        defer l.active.Add(-1)

        if current > l.maxConc {
            w.Header().Set("Retry-After", "5")
            http.Error(w, `{"error":"overloaded","code":"SERVICE_OVERLOAD"}`,
                http.StatusServiceUnavailable)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

**Latency-based adaptive shedding:**
```go
type AdaptiveShedder struct {
    mu             sync.RWMutex
    p99Latency     time.Duration
    latencyBudget  time.Duration // SLO target, e.g., 500ms
    shedProbability float64
}

func (s *AdaptiveShedder) ShouldShed() bool {
    s.mu.RLock()
    prob := s.shedProbability
    s.mu.RUnlock()
    return rand.Float64() < prob
}

func (s *AdaptiveShedder) Update(latency time.Duration) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.p99Latency = latency
    if latency > s.latencyBudget {
        overage := float64(latency-s.latencyBudget) / float64(s.latencyBudget)
        s.shedProbability = min(overage*0.5, 0.95) // max 95% shed rate
    } else {
        s.shedProbability = max(s.shedProbability-0.05, 0) // recover gradually
    }
}
```

Checklist:
- [ ] Shedding trigger chosen based on the service's bottleneck (CPU, concurrency, latency)
- [ ] Trigger threshold calibrated from load testing — not guessed
- [ ] Shedding activates before the service fails — at 80% capacity, not 100%
- [ ] Trigger recovery is gradual — rapid oscillation between shed/no-shed avoided

---

## Step 3 — Priority-Based Request Admission

Not all requests are shed equally — critical requests are admitted even under load.

**HTTP header-based priority (client signals priority):**
```go
func priorityFromRequest(r *http.Request) int {
    switch r.Header.Get("X-Request-Priority") {
    case "critical": return 0
    case "high":     return 1
    case "normal":   return 2
    default:         return 3
    }
}

func (l *PriorityLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        priority := priorityFromRequest(r)
        capacity := l.active.Load()

        // Shed P3+ requests at 60% capacity; P2+ at 80%; P1+ at 90%
        shed := (priority >= 3 && capacity > l.maxConc*60/100) ||
                (priority >= 2 && capacity > l.maxConc*80/100) ||
                (priority >= 1 && capacity > l.maxConc*90/100)

        if shed {
            http.Error(w, `{"error":"overloaded"}`, http.StatusServiceUnavailable)
            return
        }

        l.active.Add(1)
        defer l.active.Add(-1)
        next.ServeHTTP(w, r)
    })
}
```

**Route-based priority (server assigns priority by path):**
```go
func routePriority(path string) int {
    switch {
    case strings.HasPrefix(path, "/checkout"),
         strings.HasPrefix(path, "/payment"):
        return 0 // critical
    case strings.HasPrefix(path, "/orders"),
         strings.HasPrefix(path, "/cart"):
        return 1 // high
    case strings.HasPrefix(path, "/search"),
         strings.HasPrefix(path, "/products"):
        return 2 // normal
    default:
        return 3 // non-critical — shed first
    }
}
```

Checklist:
- [ ] Priority assigned by the server (route-based) — not trusted from client headers in untrusted contexts
- [ ] Critical paths (checkout, payment, auth) always at priority 0 — never shed
- [ ] Graduated shedding thresholds protect P0/P1 much earlier than P2/P3
- [ ] Client-provided priority headers only trusted from authenticated internal services

---

## Step 4 — Graceful Degradation (Partial Availability)

Instead of failing completely, return a degraded but useful response.

**Degradation strategies:**

| Feature | Normal | Degraded |
|---|---|---|
| Product recommendations | Personalized ML-generated | Static "popular items" list (cached) |
| Search | Full-text + filters | Cached top results or simplified query |
| Inventory check | Real-time stock check | Cached stock status (5 min old) |
| User profile | Full profile from DB | Cached profile or minimal (name + email only) |
| Analytics (/track) | Record event to DB | Drop silently — return 200 OK with no-op |
| Email on order | Send immediately | Queue for later delivery |

**Go — feature flag driven degradation:**
```go
func (h *ProductHandler) GetRecommendations(ctx context.Context, userID string) ([]Product, error) {
    // Try the expensive personalized path first
    if !h.isOverloaded() {
        recs, err := h.mlService.GetRecommendations(ctx, userID)
        if err == nil {
            return recs, nil
        }
        // Log the error but fall through to degraded path
        slog.Warn("recommendations service unavailable, using fallback", slog.String("user_id", userID))
    }

    // Degraded path — return cached popular items
    return h.cache.GetPopularItems(ctx)
}
```

**Always-available degraded endpoints:**
- Pre-compute and cache degraded responses at startup (popular items, static configs)
- Degraded responses must not require the failing component — they are the fallback when it fails
- Degraded mode is transparent to the client — same API shape; lower data quality

Checklist:
- [ ] Every non-critical feature has a defined degraded fallback
- [ ] Degraded responses pre-computed and cached — available without the failing dependency
- [ ] Degraded mode logged — operators know when the system is running in degraded mode
- [ ] Degraded mode does not cascade — fallback path is simpler and faster than the primary path

---

## Step 5 — Load Shedding Response Protocol

When shedding, the response format must be correct so clients can distinguish overload from other errors.

**HTTP response for shed requests:**
```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json
Retry-After: 5
X-Shed-Reason: concurrency-limit-exceeded

{
  "error": "service temporarily overloaded",
  "code": "OVERLOAD",
  "retry_after_seconds": 5
}
```

**Status code guidance:**
| Code | Meaning | Client Action |
|---|---|---|
| `503 Service Unavailable` | Server overloaded; try again | Retry with backoff |
| `429 Too Many Requests` | Rate limit exceeded (per-client) | Retry with backoff; check quota |
| `504 Gateway Timeout` | Upstream dependency timeout | Retry; may be transient |

**`Retry-After` header — mandatory for shed requests:**
- Always include `Retry-After` — tell the client how long to wait
- For overload: `Retry-After: 5` (seconds) is a reasonable starting point
- Well-behaved clients use this to implement exponential backoff without overwhelming the recovering service

Checklist:
- [ ] Shed requests return `503` with `Retry-After` header — not `500` (which implies a bug)
- [ ] Response body includes `code: "OVERLOAD"` — clients distinguish overload from other errors
- [ ] `Retry-After` value calibrated — long enough for service to recover, short enough for users
- [ ] Shed metrics recorded: `requests_shed_total` counter by reason and endpoint

---

## Step 6 — Monitor and Tune Load Shedding

```go
// Metrics for load shedding behavior
var (
    requestsShedTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "orders", Name: "requests_shed_total",
        Help: "Total requests shed by reason.",
    }, []string{"reason", "priority"})

    activeConcurrency = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "orders", Name: "active_concurrency",
        Help: "Current number of in-flight requests.",
    })

    shedRate = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "orders", Name: "shed_rate",
        Help: "Current fraction of requests being shed (0-1).",
    })
)
```

**Alerting on load shedding:**
```yaml
- alert: OrdersLoadSheddingActive
  expr: rate(orders_requests_shed_total[5m]) > 0
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Orders service is shedding requests — capacity may be insufficient"
    runbook: "https://runbooks.internal/orders/load-shedding"

- alert: OrdersLoadSheddingCritical
  expr: rate(orders_requests_shed_total[5m]) / rate(orders_http_requests_total[5m]) > 0.1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Orders service shedding > 10% of requests — scale up immediately"
```

**Tuning the concurrency limit:**
```
max_concurrency = (target_RPS) × (p99_latency_at_target_RPS)
# Little's Law: L = λ × W
# Example: 200 RPS target × 0.5s p99 latency = 100 concurrent requests
```

Checklist:
- [ ] `requests_shed_total` counter instrumented by reason and request priority
- [ ] Alert fires when shedding starts — signals capacity shortfall before user impact becomes severe
- [ ] Shedding threshold tuned via load testing — not arbitrary
- [ ] Shedding rate visible on the operations dashboard — not a hidden protection mechanism

---

## Output Report

### Critical
- No load shedding in place — traffic spike causes complete service collapse instead of graceful degradation
- Critical paths (checkout, payment) shed at the same rate as non-critical paths — revenue impact under load

### High
- Shedding trigger fires too late (at 100% capacity) — service already failing before shedding starts
- No `Retry-After` header on 503 responses — clients retry immediately, amplifying the overload
- No degraded fallbacks — when a feature is shed, users receive an error instead of a reduced response

### Medium
- No priority differentiation — all requests are treated equally; low-priority requests consume capacity needed by critical paths
- Shedding not monitored — operators unaware the service is shedding until user complaints arrive
- Concurrency limit not calibrated via load testing — threshold is guessed, not measured

### Low
- Shed responses return `500` instead of `503` — clients interpret as a bug, not a transient overload
- Recovery from overload is abrupt — shedding stops instantly when the trigger falls below threshold, causing oscillation
- Degraded mode not logged — operators cannot distinguish normal operation from degraded mode

### Passed
- Priority hierarchy defined; critical paths (P0) never shed; non-critical paths (P3/P4) shed first
- Shedding trigger calibrated from load testing; activates at 80% capacity; recovers gradually
- 503 + `Retry-After` returned for shed requests; response body includes `code: "OVERLOAD"`
- Degraded fallbacks defined and pre-cached for all non-critical features
- `requests_shed_total` counter instrumented; alert fires when shedding rate exceeds 1%

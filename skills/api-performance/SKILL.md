---
name: api-performance
description: >
  API latency optimization and throughput strategies for backend services.
  Use when: API performance, API latency optimization, API throughput, slow API, p95 latency,
  p99 latency, response time, HTTP performance, REST performance, gRPC performance,
  GraphQL performance, payload optimization, serialization performance, connection reuse,
  keep-alive, HTTP/2, HTTP compression, response caching, ETag, conditional requests,
  pagination performance, cursor pagination, offset pagination, API concurrency,
  goroutine pool, worker pool API, database query in API, N+1 API, timeout strategy,
  circuit breaker, API gateway performance, rate limiting performance, fanout API,
  parallel requests, request coalescing, field selection, sparse fieldsets, response size.
argument-hint: >
  Describe the API (framework, language), the slow endpoint (route, method, p95/p99 latency),
  and the suspected bottleneck (DB query, serialization, downstream call, payload size).
  Include profiling data or APM traces if available.
---

# API Performance Specialist

## When to Use

Invoke this skill when you need to:
- Diagnose the root cause of high API latency (p95/p99 > target)
- Reduce payload size without breaking API contracts
- Parallelize independent downstream calls within a single request
- Implement response caching with correct cache headers
- Tune timeouts, retries, and circuit breakers for downstream dependencies
- Improve throughput without scaling up infrastructure

---

## Step 1 — Profile and Locate the Bottleneck

**Never optimize without measurement. Latency has a single dominant cause — find it first.**

**Latency breakdown — where is the time spent?**
```
Total request latency:
  ├─ Network + TLS handshake (client to server)         → optimize with HTTP/2, keep-alive
  ├─ Request parsing + routing                          → usually negligible
  ├─ Auth / middleware                                  → cache token validation results
  ├─ Business logic                                     → profiling needed
  │    ├─ Database queries                              → EXPLAIN ANALYZE (see database-performance skill)
  │    ├─ External API calls                            → measure independently
  │    ├─ Serialization / deserialization               → benchmark the serializer
  │    └─ Computation                                   → profiling / flame graph
  └─ Response serialization + network write             → payload size optimization
```

**Go — pprof profiling:**
```go
import _ "net/http/pprof"

// Register pprof endpoints (restrict to internal network in production)
go http.ListenAndServe("localhost:6060", nil)

// Then: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// Or: curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
//     go tool trace trace.out
```

**OpenTelemetry spans — measure each sub-operation:**
```go
ctx, span := tracer.Start(ctx, "createOrder")
defer span.End()

ctx, dbSpan := tracer.Start(ctx, "db.insertOrder")
err := db.InsertOrder(ctx, order)
dbSpan.End()

ctx, notifySpan := tracer.Start(ctx, "notification.send")
err = notifier.Send(ctx, order)
notifySpan.End()
```

Checklist:
- [ ] Distributed trace available for the slow endpoint — spans show time breakdown per sub-operation
- [ ] Database query time isolated from application logic time
- [ ] External API call latency measured independently — determines if it is the bottleneck
- [ ] Profiling (pprof, async-profiler, py-spy) run under realistic load — not idle

---

## Step 2 — Parallelize Independent Operations

Sequential calls to independent resources are the most impactful source of avoidable latency.

```
❌ Sequential: 20ms (auth) + 80ms (product DB) + 60ms (inventory API) + 40ms (pricing API) = 200ms

✅ Parallel:  20ms (auth) + max(80ms, 60ms, 40ms) = 100ms  → 50% reduction
```

**Go — parallel fanout with errgroup:**
```go
import "golang.org/x/sync/errgroup"

func GetOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    g, ctx := errgroup.WithContext(ctx)

    var order *Order
    var inventory *Inventory
    var pricing *Pricing

    g.Go(func() error {
        var err error
        order, err = orderRepo.Get(ctx, orderID)
        return err
    })

    g.Go(func() error {
        var err error
        inventory, err = inventoryClient.Get(ctx, orderID)
        return err
    })

    g.Go(func() error {
        var err error
        pricing, err = pricingClient.Get(ctx, orderID)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return assemble(order, inventory, pricing), nil
}
```

**Bounded concurrency — prevent fanout from overwhelming downstreams:**
```go
sem := make(chan struct{}, 10) // max 10 concurrent calls

for _, id := range ids {
    id := id
    g.Go(func() error {
        sem <- struct{}{}
        defer func() { <-sem }()
        return processItem(ctx, id)
    })
}
```

Checklist:
- [ ] All independent downstream calls within a handler are parallelized
- [ ] `errgroup.WithContext` used — cancels remaining calls if one fails
- [ ] Semaphore applied to bounded fanout — no unbounded goroutine creation
- [ ] Total latency reduced to `max(parallel_calls)` — not `sum(sequential_calls)`

---

## Step 3 — Optimize Payload Size

Large response payloads increase serialization time, network transfer time, and client parsing time.

**Strategies:**

### Sparse Fieldsets (return only requested fields)
```
GET /orders?fields=id,status,total
```
```go
func (h *OrderHandler) Get(w http.ResponseWriter, r *http.Request) {
    fields := parseFields(r.URL.Query().Get("fields")) // ["id", "status", "total"]
    order, _ := h.orderRepo.Get(ctx, id)

    response := filterFields(order, fields) // only include requested fields
    json.NewEncoder(w).Encode(response)
}
```

### Pagination — cursor over offset
```go
// ❌ OFFSET pagination — DB scans all preceding rows on each page
SELECT * FROM orders ORDER BY created_at DESC OFFSET 10000 LIMIT 20; -- slow at high offsets

// ✅ Cursor pagination — O(log n) regardless of page depth
SELECT * FROM orders
WHERE created_at < $cursor_timestamp
ORDER BY created_at DESC
LIMIT 21; -- fetch N+1 to detect if there's a next page
```

### HTTP Compression
```go
import "github.com/klauspost/compress/gzhttp"

// Wrap all handlers with Brotli/gzip compression
handler := gzhttp.GzipHandler(mux)
// Reduces JSON payloads by 70-90%
```

### Efficient Serialization
```go
// ❌ encoding/json (reflection-based, slower)
json.Marshal(response)

// ✅ jsoniter — drop-in replacement, 2-5× faster
import jsoniter "github.com/json-iterator/go"
var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Marshal(response)
// Or use code-generated serializers: easyjson, sonic
```

**Payload size targets:**
| Endpoint Type | Maximum Recommended Payload |
|---|---|
| API list response (paginated) | < 100KB |
| Single resource detail | < 50KB |
| Search results | < 200KB |
| Bulk export | Stream; do not buffer in memory |

Checklist:
- [ ] Response compression enabled (Brotli preferred, gzip fallback) — 70–90% bandwidth reduction
- [ ] Sparse fieldset support for list endpoints — clients request only fields they need
- [ ] Cursor-based pagination used for all list endpoints — offset pagination removed for large datasets
- [ ] Large payloads streamed rather than buffered entirely in memory before sending

---

## Step 4 — HTTP Caching for GET Endpoints

Proper HTTP cache headers eliminate redundant processing for unchanged resources.

**ETag + Conditional GET:**
```go
func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    product, _ := h.repo.Get(ctx, id)

    etag := fmt.Sprintf(`"%x"`, md5.Sum([]byte(product.UpdatedAt.String())))

    // Return 304 Not Modified if client has current version
    if r.Header.Get("If-None-Match") == etag {
        w.WriteHeader(http.StatusNotModified)
        return
    }

    w.Header().Set("ETag", etag)
    w.Header().Set("Cache-Control", "private, max-age=60, must-revalidate")
    json.NewEncoder(w).Encode(product)
}
```

**Cache-Control directives:**

| Directive | Meaning |
|---|---|
| `public, max-age=3600` | CDN + browser can cache for 1 hour |
| `private, max-age=60` | Browser only (user-specific data); 60s TTL |
| `no-store` | Never cache — for sensitive or real-time data |
| `stale-while-revalidate=3600` | Serve stale for 1h while fetching fresh in background |
| `immutable` | Content never changes at this URL (use with content-addressed URLs) |

**Vary header — cache variants:**
```
Vary: Accept-Language, Accept-Encoding
```

Checklist:
- [ ] `Cache-Control` header set explicitly on every GET response — no implicit browser caching behavior
- [ ] `ETag` or `Last-Modified` + conditional request handling on frequently-polled endpoints
- [ ] `no-store` used for sensitive or real-time data — not the default for all endpoints
- [ ] Public API responses served via CDN with `public, max-age` header
- [ ] `Vary: Accept-Encoding` set whenever compression is enabled

---

## Step 5 — Timeout and Circuit Breaker Strategy

Slow downstream calls propagate latency to users. Timeouts and circuit breakers bound the impact.

**Timeout hierarchy:**
```
Client timeout (e.g., 30s)
  └─ API gateway timeout (e.g., 25s)
       └─ Service handler timeout (e.g., 20s)
            └─ DB query timeout (e.g., 5s)
            └─ External API call timeout (e.g., 3s)
```

**Rule:** Each layer's timeout must be shorter than its parent's — otherwise the parent times out first with no useful error context.

**Go — per-request deadline propagation:**
```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    // All downstream calls use this ctx — automatically cancelled at deadline
    result, err := h.service.Process(ctx, ...)
    if errors.Is(err, context.DeadlineExceeded) {
        http.Error(w, "request timeout", http.StatusGatewayTimeout)
        return
    }
}
```

**Circuit breaker (gobreaker):**
```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "inventory-api",
    MaxRequests: 5,                 // allow 5 requests in half-open state
    Interval:    60 * time.Second,  // reset counts every 60s
    Timeout:     30 * time.Second,  // wait 30s before attempting recovery
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5 // open after 5 consecutive failures
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return inventoryClient.Get(ctx, id)
})
if err == gobreaker.ErrOpenState {
    // Return cached/degraded response — don't wait
    return fallbackInventory(id), nil
}
```

Checklist:
- [ ] Per-request context deadline set at the handler level — all sub-calls inherit it
- [ ] DB query timeout configured separately — a slow query does not block the handler indefinitely
- [ ] Circuit breaker wrapping all external service calls — open state returns degraded response immediately
- [ ] Retry strategy uses exponential backoff with jitter — linear retries cause synchronized thundering herds
- [ ] Timeout values documented and consistent with the client's expected SLA

---

## Step 6 — Connection Management

**HTTP client reuse — never create a new client per request:**
```go
// ❌ New client per request — no connection pooling, TLS handshake every time
func callDownstream() {
    client := &http.Client{Timeout: 3 * time.Second}
    client.Get(url)
}

// ✅ Shared client — connection pool reused across requests
var httpClient = &http.Client{
    Timeout: 3 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 2 * time.Second,
        ForceAttemptHTTP2:   true,           // enable HTTP/2 multiplexing
    },
}
```

**HTTP/2 — multiplexed connections:**
- HTTP/2 multiplexes many requests over a single TCP connection — eliminates head-of-line blocking
- Enable on both server and client — most TLS-enabled Go HTTP servers support it automatically
- HTTP/2 is especially impactful when making many small parallel requests to the same host

**gRPC — persistent streaming connections:**
- gRPC over HTTP/2 inherently multiplexes and streams — ideal for high-frequency internal service calls
- Use gRPC connection pools (`grpc.Dial` with `grpc.WithBlock()` at startup, shared across handlers)

Checklist:
- [ ] HTTP client created once at startup — shared across all requests (not created per request)
- [ ] `MaxIdleConnsPerHost` tuned to match expected parallelism — default of 2 is too low
- [ ] HTTP/2 enabled for both the server and all outbound HTTP clients
- [ ] TLS session resumption enabled — reduces TLS handshake overhead for recurring connections

---

## Output Report

### Critical
- Downstream calls made sequentially when they are independent — x3 latency avoidable with parallelism
- HTTP client instantiated per request — TLS handshake overhead on every call; connection pool not used
- No request timeout set — one slow downstream dependency blocks all goroutines/threads indefinitely

### High
- No circuit breaker on external calls — a slow dependency causes cascading latency to all users
- Response compression disabled — JSON payloads 5–10× larger than necessary over the wire
- Offset pagination on large datasets — page 500 queries scan and discard 10,000 rows

### Medium
- All response fields always returned — clients receive irrelevant data; payload unnecessarily large
- ETag / conditional GET not implemented on frequently-polled endpoints — cache miss on every request
- Timeout hierarchy violated — handler timeout longer than client timeout; client disconnects with no error logged

### Low
- `Cache-Control` header absent on public GET endpoints — CDN cannot cache; origin serves every request
- Serialization using reflection-based JSON for hot paths — 3–5× slower than code-generated alternatives
- Profiling endpoint (`/debug/pprof`) exposed without network restriction — internal metrics publicly accessible

### Passed
- Distributed traces available for all endpoints; latency attributed to specific sub-operations
- Independent downstream calls parallelized with `errgroup`; bounded concurrency with semaphore
- Response compression enabled (Brotli/gzip); cursor pagination implemented; sparse fieldsets supported
- Per-request context deadlines propagated to all sub-calls; circuit breakers on all external dependencies
- HTTP clients shared and pooled; HTTP/2 enabled; `Cache-Control` and `ETag` headers set correctly

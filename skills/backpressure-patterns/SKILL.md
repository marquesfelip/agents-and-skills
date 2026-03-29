---
name: backpressure-patterns
description: >
  Flow control and system stability under load through backpressure mechanisms.
  Use when: backpressure, flow control, system stability under load, producer consumer,
  channel buffer, queue backpressure, consumer backpressure, message queue overflow,
  Kafka consumer lag, RabbitMQ queue depth, worker pool backpressure, goroutine backpressure,
  reactive streams, bounded queue, unbounded queue, memory pressure, producer throttling,
  consumer rate limiting, pipeline throttling, batch processing backpressure,
  streaming backpressure, TCP backpressure, HTTP/2 flow control, gRPC flow control,
  I/O backpressure, disk write backpressure, upstream throttling, slow consumer fast producer,
  queue depth alert, consumer lag alert.
argument-hint: >
  Describe the data flow (e.g., "Go HTTP API producing to Kafka → batch consumer writing to
  PostgreSQL"), the symptom (e.g., "queue depth growing unboundedly", "OOM on the consumer",
  "producer crashes when consumer is slow"), and the volumes involved.
---

# Backpressure Patterns Specialist

## When to Use

Invoke this skill when you need to:
- Prevent a fast producer from overwhelming a slow consumer
- Design bounded queues and buffer constraints
- Implement backpressure in Go channels, message queues, or HTTP pipelines
- Detect and alert on queue depth and consumer lag
- Throttle producers based on downstream capacity signals
- Stabilize a pipeline that collapses under burst load

---

## Step 1 — Understand the Backpressure Model

**Backpressure** means the consumer signals its capacity upstream, and the producer slows down or stops rather than overwhelming the consumer.

**Without backpressure — the failure cascade:**
```
Producer (10,000 events/s)
  → Unbounded queue grows → OOM
  → Consumer can only process 1,000 events/s → queue depth: 9,000/s growth
  → Memory exhausted in ~2 minutes
  → System crash
```

**With backpressure — stable operation:**
```
Producer (10,000 events/s)
  → Bounded queue (capacity: 500 items)
  → Consumer processes at 1,000 events/s
  → Queue full → Producer blocks OR drops OR slows down
  → Producer throttled to ~1,000 events/s
  → System stable at consumer capacity
```

**Backpressure propagation strategies:**

| Strategy | Producer Behavior When Consumer Full | Trade-off |
|---|---|---|
| **Block** | Producer waits until capacity available | Zero data loss; latency at producer |
| **Drop** | Producer discards new items | No blocking; data loss |
| **Throttle** | Producer slows down to match consumer | Smooth flow; requires producer cooperation |
| **Reject** | Producer signals error to upstream | Upstream can retry or route elsewhere |
| **Spill** | Overflow to disk or secondary store | Higher durability; disk I/O added |

Checklist:
- [ ] Backpressure strategy chosen before implementation — not left as "we'll see"
- [ ] Consumer capacity measured at realistic data volumes — establishes the throughput ceiling
- [ ] Producer behavior under full queue defined explicitly — block, drop, or throttle

---

## Step 2 — Bounded Channels and Goroutine Pools (Go)

**Unbounded goroutine creation is the most common Go backpressure failure:**
```go
// ❌ Unbounded — every incoming message spawns a goroutine; OOM on burst
for msg := range messages {
    go processMessage(msg) // goroutine count = queue depth → memory exhaustion
}

// ✅ Bounded worker pool — fixed concurrency, backpressure via channel
func NewWorkerPool(workers int) (chan<- Message, func()) {
    jobs := make(chan Message, workers*2) // bounded buffer

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for msg := range jobs {
                processMessage(msg)
            }
        }()
    }

    shutdown := func() {
        close(jobs)
        wg.Wait()
    }
    return jobs, shutdown
}

// Usage: send blocks when pool is saturated (backpressure to caller)
jobs <- msg
```

**Semaphore-based bounded concurrency:**
```go
sem := make(chan struct{}, 20) // max 20 concurrent workers

for _, item := range items {
    item := item
    sem <- struct{}{}           // acquire — blocks if 20 already running
    go func() {
        defer func() { <-sem }() // release
        process(item)
    }()
}
// Drain remaining work
for i := 0; i < cap(sem); i++ { sem <- struct{}{} }
```

**errgroup with bounded concurrency:**
```go
import "golang.org/x/sync/errgroup"
import "golang.org/x/sync/semaphore"

g, ctx := errgroup.WithContext(ctx)
sem := semaphore.NewWeighted(20)

for _, item := range items {
    item := item
    if err := sem.Acquire(ctx, 1); err != nil {
        break // context cancelled
    }
    g.Go(func() error {
        defer sem.Release(1)
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    return err
}
```

Checklist:
- [ ] No unbounded goroutine creation in loops — worker pool or semaphore always bounds concurrency
- [ ] Channel buffers sized based on consumer throughput × acceptable latency (Little's Law)
- [ ] `select` with a `default` case used for non-blocking sends — explicit drop, not silent block
- [ ] Worker pool size configurable — tunable without code changes (env var or config)

---

## Step 3 — Message Queue Backpressure

**Kafka — consumer lag as the backpressure signal:**
```go
// Consumer: process messages at a controlled rate
func runConsumer(ctx context.Context, reader *kafka.Reader, workers int) {
    pool, shutdown := NewWorkerPool(workers)
    defer shutdown()

    for {
        msg, err := reader.ReadMessage(ctx) // blocks when no messages
        if err != nil {
            break
        }

        select {
        case pool <- msg:     // send to worker pool
        default:              // pool buffer full — backpressure from workers
            // Option A: Block until space available (strongest backpressure)
            pool <- msg
            // Option B: Pause consumption briefly
            // time.Sleep(10 * time.Millisecond)
        }
    }
}
```

**Kafka partition assignment — control consumer parallelism:**
```go
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"kafka:9092"},
    Topic:          "orders",
    GroupID:        "order-processor",
    MaxBytes:       10e6,           // 10MB max batch
    CommitInterval: time.Second,    // async commit every 1s
    // Match partition count to worker pool size for even distribution
})
```

**RabbitMQ — prefetch count as backpressure:**
```go
// Consumer prefetch: only buffer N unacknowledged messages per consumer
ch.Qos(
    20,     // prefetch count — consumer never holds more than 20 unacked messages
    0,      // prefetch size (not used)
    false,  // apply per-consumer (not per-channel)
)
// Without prefetch: consumer tries to load the entire queue into memory
```

**Consumer lag monitoring (Kafka):**
```bash
# Check lag for all consumer groups
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --all-groups

# Alert when lag grows — means consumers are falling behind
```

**Prometheus metric for lag:**
```yaml
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag{topic="orders"} > 10000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Orders Kafka consumer lag > 10,000 — consumer is falling behind"
    runbook: "https://runbooks.internal/kafka/consumer-lag"
```

Checklist:
- [ ] Kafka prefetch / RabbitMQ QoS set — consumer does not buffer the entire queue in memory
- [ ] Consumer worker pool size matches partition count — all partitions consumed in parallel
- [ ] Consumer lag monitored and alerted — lag growth is the primary backpressure health signal
- [ ] Manual lag offset reset procedure documented — recovery path when lag is enormous

---

## Step 4 — Pipeline Backpressure (Stage-to-Stage)

For multi-stage processing pipelines, each stage must propagate backpressure to the previous stage.

```
Ingest → Parse → Validate → Enrich → Persist
  ↑         ↑        ↑         ↑
  backpressure propagates upstream when downstream is slow
```

**Go pipeline with bounded channels:**
```go
func buildPipeline(ctx context.Context, input <-chan RawEvent) <-chan error {
    // Each channel is bounded — backpressure propagates stage to stage
    parsed  := make(chan ParsedEvent, 100)
    valid   := make(chan ValidEvent, 100)
    enriched := make(chan EnrichedEvent, 50)

    go parseStage(ctx, input, parsed)
    go validateStage(ctx, parsed, valid)
    go enrichStage(ctx, valid, enriched)

    return persistStage(ctx, enriched)
}

func parseStage(ctx context.Context, in <-chan RawEvent, out chan<- ParsedEvent) {
    defer close(out)
    for raw := range in {
        parsed, err := parse(raw)
        if err != nil {
            continue // or send to dead letter channel
        }
        select {
        case out <- parsed: // blocks if next stage is full — backpressure
        case <-ctx.Done():
            return
        }
    }
}
```

**Monitoring pipeline stage health:**
```go
var pipelineChannelDepth = promauto.NewGaugeVec(prometheus.GaugeOpts{
    Namespace: "pipeline", Name: "channel_depth",
    Help: "Current depth of each pipeline stage channel.",
}, []string{"stage"})

// Report channel depths periodically
go func() {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        pipelineChannelDepth.WithLabelValues("parse").Set(float64(len(parsed)))
        pipelineChannelDepth.WithLabelValues("validate").Set(float64(len(valid)))
        pipelineChannelDepth.WithLabelValues("enrich").Set(float64(len(enriched)))
    }
}()
```

Checklist:
- [ ] All inter-stage channels are bounded — no `make(chan T)` with unbounded buffer in pipelines
- [ ] Slowest stage identified and its throughput used to size upstream buffers
- [ ] Pipeline stage channel depths instrumented — visible in monitoring before backup occurs
- [ ] Dead letter channel or error sink defined — malformed records don't stall the pipeline

---

## Step 5 — HTTP and gRPC Flow Control

**HTTP/2 — built-in flow control:**
HTTP/2 has per-stream and per-connection flow control windows. When the client is slow, the server automatically stops writing.

```go
// Go HTTP server — HTTP/2 enabled automatically with TLS
srv := &http.Server{
    Addr:    ":8443",
    Handler: mux,
    // HTTP/2 configured via TLSConfig
}
// For streaming responses, check client disconnection to stop producing
func (h *Handler) StreamLargeResponse(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", 500)
        return
    }

    for _, chunk := range generateChunks(r.Context()) {
        select {
        case <-r.Context().Done(): // client disconnected — stop producing
            return
        default:
        }
        fmt.Fprintf(w, "%s\n", chunk)
        flusher.Flush()
    }
}
```

**gRPC streaming — flow control via Send blocking:**
```go
// Server streaming — gRPC buffers and blocks Send when client is slow
func (s *Service) StreamOrders(req *pb.StreamRequest, stream pb.OrderService_StreamOrdersServer) error {
    for {
        order, err := s.repo.Next(stream.Context())
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        // Send blocks when the client's receive buffer is full — natural backpressure
        if err := stream.Send(order); err != nil {
            return err // client disconnected or buffer permanently full
        }
    }
}
```

Checklist:
- [ ] HTTP streaming handlers check `r.Context().Done()` — stop producing when client disconnects
- [ ] gRPC server streaming uses `Send` return value — stop on error (client buffer full / disconnected)
- [ ] Large response bodies streamed via `http.Flusher` — not buffered entirely in server memory
- [ ] HTTP/2 enabled on all services — inherits built-in flow control; outperforms HTTP/1.1 for streaming

---

## Step 6 — Producer Throttling Based on Consumer Signals

When the consumer cannot signal backpressure directly (e.g., fire-and-forget producers), use explicit throttling.

**Token bucket producer throttling:**
```go
import "golang.org/x/time/rate"

// Allow 100 events/s with a burst of 50
limiter := rate.NewLimiter(rate.Limit(100), 50)

for _, event := range events {
    // Block until a token is available
    if err := limiter.Wait(ctx); err != nil {
        return err // context cancelled
    }
    producer.Send(event)
}
```

**Adaptive throttling — slow down when consumer lag grows:**
```go
type AdaptiveProducer struct {
    limiter     *rate.Limiter
    lagMonitor  func() int64
    baseRate    float64
    currentRate float64
}

func (p *AdaptiveProducer) adjustRate() {
    lag := p.lagMonitor()
    switch {
    case lag > 100000:
        p.currentRate = p.baseRate * 0.1  // 10% of base rate
    case lag > 10000:
        p.currentRate = p.baseRate * 0.5  // 50%
    case lag < 1000:
        p.currentRate = min(p.currentRate*1.1, p.baseRate) // recover 10% per cycle
    }
    p.limiter.SetLimit(rate.Limit(p.currentRate))
}
```

Checklist:
- [ ] Producer rate is bounded — no fire-and-forget without a rate limit
- [ ] Adaptive throttling adjusts producer rate based on consumer lag — not static
- [ ] Rate limiter state shared across all producer goroutines — not per-goroutine (defeats the purpose)
- [ ] Rate limit values configurable — adjustable without code deployment

---

## Output Report

### Critical
- Unbounded goroutine creation in a consumer loop — OOM crash under moderate burst load
- No consumer prefetch limit (RabbitMQ/Kafka) — consumer buffers the entire queue; OOM during replay
- Pipeline inter-stage channels unbounded — fast stage fills memory of downstream slow stage

### High
- No backpressure from consumers to producers — fast producer + slow consumer → queue grows unboundedly
- Worker pool size not tuned to consumer throughput — pool too small wastes available capacity; too large causes downstream overload
- Consumer lag not monitored — lag growth detected only when system is already in distress

### Medium
- Drop strategy used without metrics — events silently lost with no visibility into loss rate
- Pipeline channel depth not instrumented — bottleneck stage invisible until the pipeline backs up
- Rate limiter not shared across goroutines — each goroutine has its own limit; effective rate = N × limit

### Low
- Consumer lag alert threshold too high (> 1M) — meaningful backlog before alert fires
- Worker pool size hardcoded — capacity cannot be adjusted without a code deployment
- HTTP streaming handler does not check client disconnect — continues generating data after client leaves

### Passed
- Bounded worker pool used for all consumer loops; no unbounded goroutine creation
- Kafka prefetch / RabbitMQ QoS configured; consumer lag monitored and alerted
- Pipeline inter-stage channels bounded; slowest stage throughput measured and used to size buffers
- Producer rate limiter applied; adaptive throttling adjusts rate based on consumer lag signal
- HTTP streaming checks context cancellation; gRPC Send errors handled to stop on slow/disconnected clients

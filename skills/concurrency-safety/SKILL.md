---
name: concurrency-safety
description: 'Race condition prevention, locking strategies, and concurrent execution safety. Use when: race condition, concurrency safety, shared state, mutex, lock, atomic operation, goroutine safety, thread safety, data race, concurrent access, Go race detector, sync.Mutex, sync.RWMutex, channel safety, concurrent map write, concurrent execution, parallelism safety.'
argument-hint: 'Concurrent code, shared data structure, or goroutine/thread pattern to review — include the language and concurrency model'
---

# Concurrency Safety Specialist

## When to Use
- Reviewing code with shared mutable state accessed from multiple goroutines or threads
- Debugging a data race detected by `go test -race` or similar tools
- Choosing between mutex, channel, atomic, or other synchronization primitives
- Designing concurrent data pipelines that must be safe and efficient
- Auditing code for deadlock potential or live-lock conditions

---

## Step 1 — Identify Shared Mutable State

Concurrency bugs only occur when goroutines/threads share mutable state. Start by identifying:

- [ ] Global or package-level variables written by more than one goroutine
- [ ] Structs passed by pointer to multiple goroutines
- [ ] Maps written concurrently (Go maps are not goroutine-safe)
- [ ] Slices appended to from multiple goroutines
- [ ] Counters incremented without atomics or mutex
- [ ] Caches read and written concurrently
- [ ] Channels managed from multiple goroutines without coordination

```go
// RACE CONDITION — map written by multiple goroutines
var results = make(map[string]int)  // shared map

func processItems(items []Item) {
    for _, item := range items {
        go func(i Item) {
            results[i.ID] = compute(i)  // concurrent map write — data race!
        }(item)
    }
}
```

---

## Step 2 — Select the Right Synchronization Primitive

| Scenario | Best primitive | Why |
|---|---|---|
| Shared mutable state (general) | `sync.Mutex` | Simple, clear ownership |
| Read-heavy, write-rare shared state | `sync.RWMutex` | Concurrent reads; exclusive write |
| Single integer/bool counter | `sync.atomic` / `atomic.Int64` | No lock overhead |
| One-time initialization | `sync.Once` | Guaranteed single execution |
| Passing data between goroutines | Channel | Ownership transfer — no sharing |
| Fan-out work, collect results | `errgroup` + channel | Bounded concurrency + error collection |
| Concurrent-safe map | `sync.Map` | Designed for concurrent access |
| Wait for goroutines to complete | `sync.WaitGroup` | Coordination without data sharing |

### Mutex vs Channel Decision

```
"Do not communicate by sharing memory; share memory by communicating." — Go proverb

Use mutex when:      - protecting a shared data structure in place
                     - multiple operations on the same object must be atomic
                     - cache, counter, registry, in-memory store

Use channel when:    - transferring ownership of data from one goroutine to another
                     - pipeline flow (producer → consumer)
                     - signaling events (done signals, work items, results)
```

---

## Step 3 — Mutex Patterns

### Basic Mutex Protection

```go
// SAFE — mutex protects the cache map
type SafeCache struct {
    mu    sync.RWMutex
    store map[string]CachedValue
}

func (c *SafeCache) Get(key string) (CachedValue, bool) {
    c.mu.RLock()         // shared read lock — multiple readers allowed concurrently
    defer c.mu.RUnlock()
    val, ok := c.store[key]
    return val, ok
}

func (c *SafeCache) Set(key string, val CachedValue) {
    c.mu.Lock()          // exclusive write lock — blocks all readers and writers
    defer c.mu.Unlock()
    c.store[key] = val
}
```

### Critical Mutex Rules

- [ ] **Always use `defer mu.Unlock()`** — prevents forgetting on error paths
- [ ] **Never call external functions while holding a mutex** — risk of deadlock and performance degradation
- [ ] **Never pass a mutex by value** — always use pointer receiver or embed by value at the owner level
- [ ] **Minimize lock scope** — hold lock only for the critical section, not surrounding logic
- [ ] **Document which fields a mutex protects** in a comment

```go
// DEADLOCK — holding mu while calling a function that acquires mu again
func (c *Cache) GetOrCreate(key string, create func() Value) Value {
    c.mu.Lock()
    defer c.mu.Unlock()
    if val, ok := c.store[key]; ok {
        return val
    }
    val := create()           // if create() calls c.Get() or c.Set() → deadlock!
    c.store[key] = val
    return val
}

// SAFE — release lock before calling external functions
func (c *Cache) GetOrCreate(key string, create func() Value) Value {
    c.mu.RLock()
    val, ok := c.store[key]
    c.mu.RUnlock()
    if ok {
        return val
    }
    val = create()            // called without lock — no deadlock risk
    c.mu.Lock()
    c.store[key] = val        // double-check pattern
    c.mu.Unlock()
    return val
}
```

---

## Step 4 — Atomic Operations

Use `sync/atomic` for simple counters and flags — zero overhead vs mutex:

```go
import "sync/atomic"

type Metrics struct {
    requestCount atomic.Int64   // Go 1.19+ typed atomics
    errorCount   atomic.Int64
    activeConns  atomic.Int32
}

func (m *Metrics) RecordRequest(isError bool) {
    m.requestCount.Add(1)
    if isError {
        m.errorCount.Add(1)
    }
}

func (m *Metrics) ActiveConnections() int32 {
    return m.activeConns.Load()
}
```

```go
// Atomic compare-and-swap for state machine transitions
type State = int32
const (
    StateIdle    State = 0
    StateRunning State = 1
    StateStopped State = 2
)

type Worker struct {
    state atomic.Int32
}

func (w *Worker) Start() error {
    if !w.state.CompareAndSwap(StateIdle, StateRunning) {
        return errors.New("worker already running")
    }
    go w.run()
    return nil
}
```

---

## Step 5 — Channel Patterns for Safe Concurrency

```go
// Fan-out / Fan-in with bounded concurrency (errgroup pattern)
func processAll(ctx context.Context, items []Item) ([]Result, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Result, len(items))

    sem := make(chan struct{}, 8)  // max 8 concurrent workers

    for i, item := range items {
        i, item := i, item  // capture loop vars — Go < 1.22
        g.Go(func() error {
            sem <- struct{}{}
            defer func() { <-sem }()

            result, err := process(ctx, item)
            if err != nil {
                return fmt.Errorf("processing item %d: %w", i, err)
            }
            results[i] = result   // safe: each goroutine writes to distinct index
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

```go
// Pipeline pattern — each stage is a goroutine; data flows via channels
func pipeline(ctx context.Context, input <-chan RawData) <-chan ProcessedData {
    out := make(chan ProcessedData, 100)   // buffered to avoid blocking
    go func() {
        defer close(out)
        for raw := range input {
            select {
            case <-ctx.Done():
                return
            case out <- transform(raw):
            }
        }
    }()
    return out
}
```

---

## Step 6 — Deadlock Prevention

Deadlock occurs when goroutine A holds lock X and waits for lock Y, while goroutine B holds lock Y and waits for lock X.

### Prevention Rules

1. **Consistent lock ordering** — always acquire multiple locks in the same order across all goroutines
2. **Lock leveling** — assign a numeric level to each lock; only acquire lock with higher level while holding lower
3. **Avoid nested locks** — if possible, complete one operation before acquiring another lock
4. **Use timeouts** — `context.WithTimeout` to prevent goroutines waiting forever

```go
// DEADLOCK RISK — inconsistent ordering
// Goroutine 1: mu1.Lock() then mu2.Lock()
// Goroutine 2: mu2.Lock() then mu1.Lock()

// SAFE — consistent ordering: always mu1 before mu2
func safeTransfer(from, to *Account, amount Money) {
    // Always lock lower ID first — deterministic ordering regardless of call order
    first, second := from, to
    if from.ID > to.ID {
        first, second = to, from
    }
    first.mu.Lock()
    defer first.mu.Unlock()
    second.mu.Lock()
    defer second.mu.Unlock()
    // transfer...
}
```

---

## Step 7 — Race Detector Usage (Go)

Always run the race detector in CI:

```bash
# Run tests with race detection
go test -race ./...

# Build binary with race detection (for staging/integration environments)
go build -race -o ./app .

# Run specific test with race detection
go test -race -run TestConcurrentCache ./internal/cache/

# Verify with additional thread sanitization
GORACE="halt_on_error=1 history_size=7" go test -race ./...
```

### Race Detector Output Interpretation

```
DATA RACE
Read at 0x00c0001e4010 by goroutine 8:
  main.processItems.func1()
      /app/main.go:23 +0x44      ← goroutine reading

Previous write at 0x00c0001e4010 by goroutine 7:
  main.processItems.func1()
      /app/main.go:23 +0x57      ← another goroutine writing same address

# Action: identify the shared variable at the address; add mutex or use channel
```

---

## Output Report

```
## Concurrency Safety Review: <package/service>

### Critical
- Concurrent map writes to `results map[string]int` from worker goroutines — DATA RACE
  Go maps are not goroutine-safe; concurrent write causes panic or corrupted data
  Fix: protect with sync.RWMutex or replace with sync.Map; or collect results via channel

- Counter `requestCount int` incremented with `++` from multiple goroutines — DATA RACE
  Non-atomic increment; reads and writes interleave; counter under-counts
  Fix: replace with atomic.Int64; use .Add(1) for increment, .Load() for read

### High
- Mutex mu.Lock() called without `defer mu.Unlock()` — unlock skipped on error paths
  Goroutines calling the function after an error will deadlock waiting for the lock
  Fix: always use `defer mu.Unlock()` immediately after `mu.Lock()`

- GetOrCreate() calls external user-provided function while holding the mutex
  If user function calls any cache method, deadlock occurs; also blocks all cache reads
  Fix: copy data under lock; call external function outside lock; re-acquire to write

### Medium
- Fan-out with `go processItem(item)` launches unbounded goroutines for large input
  Memory exhaustion on large batch input; OS thread pressure
  Fix: bound with semaphore channel or errgroup with worker pool pattern (max 8 workers)

- sync.WaitGroup.Add() called inside launched goroutine instead of before
  Race: main goroutine may exit wg.Wait() before all goroutines have called Add()
  Fix: call wg.Add(1) in the loop before launching the goroutine

### Low
- Tests do not use `go test -race` in CI pipeline
  Races only reliably detected by the race detector
  Fix: add `go test -race ./...` to CI; add integration test with artificial concurrent load

### Passed
- SafeCache uses sync.RWMutex with correct RLock/RUnlock for reads ✓
- errgroup used for bounded fan-out with context cancellation ✓
- No global mutable state outside of explicitly synchronized types ✓
```

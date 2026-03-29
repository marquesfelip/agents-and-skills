---
name: avoid-race-conditions
description: 'Find, fix, and prevent race conditions in Go code. Use when: data race in golang, race detector failure, concurrent map writes, goroutine synchronization bug, mutex vs channel decision, atomic operations in go, errgroup shared state issue, flaky concurrency test. Produces a safe concurrency fix plus regression tests and race-check commands.'
argument-hint: 'Target package or file and the race symptom'
---

# Avoid Race Conditions (Go)

## When to Use
- Race detector reports conflicts in tests or runtime
- Concurrent writes to shared slices, maps, counters, or structs
- Goroutine orchestration with wait groups, channels, or errgroup appears flaky
- Concurrency code is being introduced and needs prevention checks before merge

## Step 1 - Reproduce the Race Reliably

1. Run the target tests with race detector enabled.
2. If no deterministic failure appears, increase repetition and reduce cache effects.
3. Capture one concrete conflict trace before changing code.

Recommended commands:
- go test -race ./...
- go test -race -count=1 ./...
- go test -race -run <TestName> -count=50 ./...

## Step 2 - Classify Shared-State Risk

Inspect the code path and classify the shared state:

| Shared State | Typical Symptom | Preferred Guard |
|---|---|---|
| map | concurrent map writes panic or race report | sync.Mutex or single-owner goroutine |
| slice append | missing or reordered elements | sync.Mutex or buffered channel collector |
| numeric counters | wrong totals | sync/atomic |
| struct fields | torn reads / stale values | sync.Mutex or immutable copy-per-goroutine |
| error aggregation | dropped errors / racy append | channel fan-in or mutex-protected collector |

## Step 3 - Choose the Synchronization Strategy

Use this decision table:

| Decision Point | Choose |
|---|---|
| Single scalar value only | sync/atomic |
| Composite mutable data (map/slice/struct) | sync.Mutex / sync.RWMutex |
| Ownership can be centralized in one goroutine | channels (message passing) |
| Read-heavy with rare writes and simple consistency | sync.RWMutex |
| Need bounded worker parallelism | errgroup with SetLimit plus protected shared state |

Implementation rules:
- Do not mix multiple synchronization primitives on the same variable unless strictly necessary.
- Keep lock scope minimal and explicit.
- Never read shared mutable state outside the chosen guard.

## Step 4 - Apply the Fix Safely

1. Introduce synchronization in the smallest surface area needed.
2. Preserve existing behavior and output format.
3. Prefer dependency injection over global mutable state.
4. For file and XML pipelines, avoid shared append without protection.

Common safe patterns:
- mutex.Lock -> mutate shared map/slice -> mutex.Unlock
- atomic.AddInt64 and atomic.LoadInt64 for counters
- results channel with one collector goroutine

## Step 5 - Add Regression Tests

Add or update tests to lock in the fix:

- Unit test for correctness under concurrent execution
- Subtest loop with parallel workers
- Race validation run included in verification

Example verification set:
- go test -race ./...
- go test -race -count=1 ./...
- go test -run <TargetTest> -race -count=50 ./...

## Step 6 - Final Quality Checks

Mark complete only if all checks pass:

- [ ] Race detector reports no conflicts in touched package(s)
- [ ] No new flaky tests introduced
- [ ] Shared mutable state has one clear synchronization strategy
- [ ] Lock scope is minimal and no deadlock risk is introduced
- [ ] Error handling and output behavior remain unchanged
- [ ] Code comments explain non-obvious concurrency decisions

## After Completion

Report back with:
- Root cause category (map/slice/counter/struct/channel misuse)
- Exact synchronization strategy chosen and why
- Commands executed and outcome summary
- Any residual risk (for example, integration paths not covered by tests)

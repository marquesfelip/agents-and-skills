---
name: event-replay-strategy
description: 'Event replay and recovery mechanisms for SaaS systems. Use when: event replay, replay events, event recovery, consumer replay, replay from offset, Kafka replay, replay after bug fix, event backfill, reprocess events, replay mode, new consumer bootstrap, replay gate, replay window, selective replay, tenant replay, aggregate replay, replay safety, outbox replay, event store replay, idempotent replay, event driven recovery, replay consumer, replay trigger, replay source.'
argument-hint: 'Describe why replay is needed (bug fix, new consumer, data corruption), the event source (PostgreSQL outbox, Kafka, event store), the event volume and time window to replay, and whether replay should be selective (by tenant, aggregate type, or event type).'
---

# Event Replay Strategy

## When to Use

Invoke this skill when you need to:
- Replay historical events to a new consumer that needs to bootstrap state
- Rerun a time window of events after a bug fix to correct derived data
- Recover from consumer data corruption by replaying source-of-truth events
- Perform selective replay: only for one tenant, aggregate type, or event type
- Design consumers to be safe for replay (idempotent handling of already-applied events)
- Control replay execution without blocking production event processing

---

## Replay Triggers and Strategies

| Trigger | Source | Replay scope |
|---|---|---|
| New consumer bootstrapping | Outbox table, event store | Full history or from anchor date |
| Bug fix in consumer | Outbox table, event store | Time window containing corrupted events |
| Data corruption recovery | Outbox or snapshot | Affected tenants, affected aggregate type |
| Schema migration with compute | Outbox table | All events of one type |
| Kafka consumer group reset | Kafka broker offset | Offset range or time-based |

---

## Step 1 — Prerequisites: Consumer Idempotency

Replay is only safe if every consumer is idempotent. Verify each consumer:

```go
// Required before enabling replay — consumer must handle duplicates gracefully
func (c *SubscriptionActivatedConsumer) Handle(ctx context.Context, event Event) error {
    tenantID := event.TenantID

    // Idempotency guard: skip if already applied (see job-idempotency skill)
    applied, err := c.db.HasProcessed(ctx, event.ID, "subscription-activated-consumer")
    if err != nil {
        return err
    }
    if applied {
        return nil // already handled — safe to skip on replay
    }

    // Business logic — safe to run because we checked above
    if err := c.entitlementsRepo.Activate(ctx, tenantID, event.Data.PlanID); err != nil {
        return err
    }

    return c.db.MarkProcessed(ctx, event.ID, "subscription-activated-consumer")
}
```

---

## Step 2 — Outbox-Based Replay Source

```sql
-- Outbox table is the replay source of truth for PostgreSQL-based event sourcing
-- (created in event-driven-saas-patterns skill)
-- Query events for replay with flexible filtering:

-- All events in time window:
SELECT * FROM outbox_events
WHERE published_at BETWEEN $1 AND $2
ORDER BY published_at ASC, id ASC;

-- Selective replay: one tenant, one event type:
SELECT * FROM outbox_events
WHERE tenant_id = $1
  AND event_type = $2
  AND published_at BETWEEN $3 AND $4
ORDER BY published_at ASC, id ASC;

-- New consumer bootstrap: all events of one or more types:
SELECT * FROM outbox_events
WHERE event_type = ANY($1::text[])
ORDER BY published_at ASC, id ASC;
```

---

## Step 3 — Replay Gate: Separate Mode from Production

```go
// ReplayConfig controls replay execution without touching production flow
type ReplayConfig struct {
    Enabled    bool
    Source     string    // "outbox" | "kafka" | "event-store"
    StartTime  time.Time
    EndTime    time.Time
    TenantID   string    // empty = all tenants
    EventTypes []string  // empty = all types
    DryRun     bool      // if true, log only — no DB writes
    BatchSize  int       // default 100
    ThrottleMs int       // ms between batches to avoid overloading DB
}

// Read config from environment or admin endpoint — never hardcoded
func LoadReplayConfig() (ReplayConfig, bool) {
    if os.Getenv("REPLAY_ENABLED") != "true" {
        return ReplayConfig{}, false
    }
    start, _ := time.Parse(time.RFC3339, os.Getenv("REPLAY_START"))
    end, _ := time.Parse(time.RFC3339, os.Getenv("REPLAY_END"))
    return ReplayConfig{
        Enabled:    true,
        Source:     os.Getenv("REPLAY_SOURCE"),
        StartTime:  start,
        EndTime:    end,
        TenantID:   os.Getenv("REPLAY_TENANT_ID"),
        EventTypes: strings.Split(os.Getenv("REPLAY_EVENT_TYPES"), ","),
        DryRun:     os.Getenv("REPLAY_DRY_RUN") == "true",
        BatchSize:  100,
        ThrottleMs: 50,
    }, true
}
```

---

## Step 4 — Replay Runner

```go
type ReplayRunner struct {
    outbox   OutboxRepository
    consumer EventConsumer
    cfg      ReplayConfig
}

func (r *ReplayRunner) Run(ctx context.Context) error {
    slog.Info("replay started",
        "start", r.cfg.StartTime,
        "end", r.cfg.EndTime,
        "tenant", r.cfg.TenantID,
        "types", r.cfg.EventTypes,
        "dry_run", r.cfg.DryRun,
    )

    var cursor *time.Time
    total := 0

    for {
        batch, err := r.outbox.FetchReplayBatch(ctx, OutboxReplayQuery{
            StartTime:  r.cfg.StartTime,
            EndTime:    r.cfg.EndTime,
            After:      cursor,
            TenantID:   r.cfg.TenantID,
            EventTypes: r.cfg.EventTypes,
            Limit:      r.cfg.BatchSize,
        })
        if err != nil {
            return fmt.Errorf("fetch replay batch: %w", err)
        }
        if len(batch) == 0 {
            break
        }

        for _, event := range batch {
            if r.cfg.DryRun {
                slog.Info("[dry-run] would replay", "id", event.ID, "type", event.Type, "tenant", event.TenantID)
                continue
            }
            if err := r.consumer.Handle(ctx, event); err != nil {
                slog.Error("replay event failed", "id", event.ID, "err", err)
                // Continue replay — consumer idempotency means it's safe to continue
                // Failed events are logged for manual review
            }
        }

        last := batch[len(batch)-1]
        lastTime := last.PublishedAt
        cursor = &lastTime
        total += len(batch)

        slog.Info("replay batch complete", "total_replayed", total, "last_event", last.ID)

        // Throttle to avoid overwhelming the DB
        if r.cfg.ThrottleMs > 0 {
            time.Sleep(time.Duration(r.cfg.ThrottleMs) * time.Millisecond)
        }

        if err := ctx.Err(); err != nil {
            return err // respect cancellation
        }
    }

    slog.Info("replay complete", "total_events", total)
    return nil
}
```

---

## Step 5 — Kafka Replay (Offset Reset)

```go
// Kafka replay: set consumer group offset to replay from a point in time
// This is done via admin API — not inline in the consumer

type KafkaReplayConfig struct {
    Topic         string
    ConsumerGroup string
    FromTime      time.Time
}

func ResetConsumerGroupOffset(cfg KafkaReplayConfig, admin sarama.ClusterAdmin) error {
    // 1. Get partitions
    offsets, err := admin.ListConsumerGroupOffsets(cfg.ConsumerGroup, map[string][]int32{
        cfg.Topic: nil, // nil = all partitions
    })
    if err != nil {
        return fmt.Errorf("list offsets: %w", err)
    }

    // 2. For each partition, find the offset at cfg.FromTime
    // Use OffsetForTimes API — requires Kafka >= 0.10
    newOffsets := map[string]map[int32]*sarama.OffsetAndMetadata{}
    newOffsets[cfg.Topic] = map[int32]*sarama.OffsetAndMetadata{}
    for partition := range offsets.Blocks[cfg.Topic] {
        // In practice: use admin.GetOffset for time-to-offset mapping
        newOffsets[cfg.Topic][partition] = &sarama.OffsetAndMetadata{
            Offset: sarama.OffsetOldest, // replay from beginning as an example
        }
    }

    return admin.ResetConsumerGroupOffset(cfg.ConsumerGroup, cfg.Topic, newOffsets)
}
```

---

## Replay Safety Rules

| Rule | Why |
|---|---|
| Every consumer must be idempotent before enabling replay | Replay events will arrive as duplicates alongside production events |
| Never replay in production broker from day 1 — use dry-run first | Dry-run reveals missing idempotency before data mutation |
| Throttle batch processing | Replay batches can saturate the DB if unconstrained |
| Log failed events during replay — continue processing | A single bad event should not abort the full replay |
| Replay specific tenant before doing all tenants | Validates idempotency and correctness on small blast radius |
| Disable replay gate after completion | Env vars left set cause accidental re-replay on next deploy |

---

## Quality Checks

- [ ] All consumers in scope implement idempotency guard before replay is enabled
- [ ] Dry-run executed and logged with 0 errors before live replay
- [ ] Replay started for one tenant, verified results, then widened to all tenants
- [ ] Batch throttle (`ThrottleMs` > 0) prevents DB saturation
- [ ] Replay runner respects context cancellation — safe to stop mid-run
- [ ] Replay gate (env var or flag) is disabled after replay completes
- [ ] Post-replay: verify derived state matches expected values via assertion query

## After Completion

- Use **`job-idempotency`** to implement the idempotency guard required in every consumer
- Use **`dead-letter-handling`** for events that fail repeatedly during replay
- Use **`event-ordering-strategy`** if ordered replay is required for correctness
- Use **`event-driven-saas-patterns`** for the outbox schema that serves as replay source

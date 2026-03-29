---
name: replication-strategies
description: >
  Read replicas, replication safety, and consistency tradeoffs in database replication.
  Use when: replication strategy, read replica, database replication, primary replica, leader follower,
  replication lag, eventual consistency, read after write consistency, replication safety,
  synchronous replication, asynchronous replication, streaming replication, logical replication,
  PostgreSQL replication, MySQL replication, failover, promote replica, read scaling,
  replica routing, replication slot, WAL, binlog, replica delay, stale read, replica monitoring.
argument-hint: >
  Describe the system's read/write pattern and consistency requirements (e.g., "high read traffic with
  user profile lookups — reads can tolerate 1-2s lag, but order status must be read-after-write consistent"),
  and specify the database engine and current infrastructure.
---

# Replication Strategies Specialist

## When to Use

Invoke this skill when you need to:
- Design a read scaling strategy using read replicas
- Choose between synchronous and asynchronous replication
- Handle replication lag and stale read scenarios safely
- Route queries between primary and replicas
- Design failover and promotion procedures
- Monitor replication health and lag

---

## Step 1 — Choose Replication Mode

| Mode | How It Works | Durability | Latency Impact | Use When |
|---|---|---|---|---|
| **Asynchronous** | Primary commits without waiting for replica acknowledgment | Replica may lag — data loss risk on leader failure | No write latency added | Default; tolerable lag; high write throughput needed |
| **Synchronous** | Primary waits for ≥1 replica to confirm WAL receipt before commit | Zero data loss (synchronous replica is up-to-date) | Added write latency = replica RTT | Financial data, critical records where data loss is unacceptable |
| **Quorum / Semi-sync** | Commit after majority of replicas acknowledge (MySQL: `SEMISYNC_MASTER`) | Reduced data loss risk | Moderate latency | Middle ground for distributed clusters |

**PostgreSQL synchronous replication:**
```sql
-- postgresql.conf
synchronous_standby_names = 'replica1'   -- wait for this replica before commit
synchronous_commit = on                  -- default: wait for local WAL flush + replica confirm
```

**MySQL semi-synchronous replication:**
```sql
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- fall back to async after 1s
```

Checklist:
- [ ] Replication mode chosen based on acceptable data loss window (RPO)
- [ ] Synchronous replication used for any data where loss is unacceptable
- [ ] Write latency impact of synchronous replication accepted by stakeholders
- [ ] Semi-sync timeout configured to prevent synchronous replication from blocking indefinitely

---

## Step 2 — Read Replica Routing Strategy

**Route by query type:**
| Query Type | Route To | Reason |
|---|---|---|
| Application writes (`INSERT`, `UPDATE`, `DELETE`) | Primary only | Replicas are read-only |
| Reads immediately after a write (read-after-write) | Primary | Replica may not have received the write yet |
| General reads (user listings, search, reports) | Replica | Offloads primary |
| Long-running analytical queries | Dedicated analytics replica | Prevents interference with OLTP |
| Admin / schema migrations | Primary only | Never run DDL on replicas |

**Read-after-write consistency patterns:**

1. **Always read from primary for the current user's own data** — simplest; highest primary load
2. **Session routing**: after a write, route that user's reads to primary for 1–5 seconds
3. **Replication token**: write returns a LSN/position; reader checks replica has reached that position before serving read
4. **Sticky session**: pin the user to primary for the session duration after any write

Checklist:
- [ ] Writes never routed to replicas
- [ ] Read-after-write scenarios identified and handled (profile update, order placement)
- [ ] Analytics/reporting queries routed to a dedicated replica — not the primary
- [ ] Connection pool maintains separate pools for primary and replica connections

---

## Step 3 — Replication Lag Monitoring

Lag is the delay between a write on the primary and its application on the replica.

**PostgreSQL — measure lag:**
```sql
-- On the replica:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- On the primary (requires pg_stat_replication):
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS bytes_lag
FROM pg_stat_replication;
```

**MySQL:**
```sql
SHOW REPLICA STATUS\G
-- Look at: Seconds_Behind_Source, Relay_Log_Space, Exec_Master_Log_Pos
```

**Alerting thresholds:**

| Alert Level | Lag Threshold | Action |
|---|---|---|
| Warning | > 10 seconds | Investigate write volume or replica capacity |
| Critical | > 60 seconds | Route all reads to primary; investigate immediately |
| Emergency | Replica stopped | Promote or replace replica; verify data integrity |

Checklist:
- [ ] Replication lag metric exported to monitoring (Prometheus, Datadog) with alerts configured
- [ ] Application has a lag-aware fallback: if lag > threshold, route to primary
- [ ] Replication slot lag monitored separately (PostgreSQL) — stale slots cause WAL accumulation and disk exhaustion
- [ ] Alert on `pg_replication_slots.active = false` for any slot

---

## Step 4 — Replication Slot Safety (PostgreSQL)

Replication slots prevent the primary from discarding WAL segments that replicas haven't consumed yet. If a replica falls behind or disconnects, the slot grows unboundedly.

```sql
-- Check slot lag:
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;

-- Drop a stale slot (CAUTION: the replica must reconnect and re-sync):
SELECT pg_drop_replication_slot('stale_slot_name');
```

Checklist:
- [ ] `max_slot_wal_keep_size` set to prevent runaway disk usage (PostgreSQL 13+)
- [ ] Stale replication slots detected by monitoring and automatically dropped after a grace period
- [ ] Logical replication slots monitored separately from physical replication slots

---

## Step 5 — Failover and Promotion

**Manual promotion (PostgreSQL):**
```bash
# On the replica:
pg_ctl promote -D /var/lib/postgresql/data
# or touch a trigger file:
touch /tmp/postgresql.trigger.5432
```

**Automated failover tools:**

| Tool | Database | Notes |
|---|---|---|
| **Patroni** | PostgreSQL | Uses etcd/ZooKeeper/Consul for leader election |
| **Repmgr** | PostgreSQL | Simpler; manual intervention for split-brain |
| **pg_auto_failover** | PostgreSQL | Microsoft-maintained; monitor-based |
| **Orchestrator** | MySQL | Topology-aware; handles complex replication graphs |
| **AWS RDS Multi-AZ** | MySQL, PostgreSQL | Managed synchronous failover |

**Failover runbook checklist:**
- [ ] Failover trigger defined: health check failures, lag > threshold, primary unreachable
- [ ] Fencing mechanism in place — old primary must be prevented from becoming a replica of itself
- [ ] Application connection strings use a virtual IP or cluster endpoint (not the primary IP directly)
- [ ] Failover tested in staging using simulated primary failure (chaos testing)
- [ ] RTO (Recovery Time Objective) measured during testing
- [ ] After failover: old primary inspected for data divergence before re-joining as replica

---

## Step 6 — Logical Replication Use Cases

Logical replication replicates row-level changes (not WAL binary) — useful for:

- Replicating a subset of tables (not the full DB)
- Replicating across major version boundaries (PG upgrade path)
- CDC (Change Data Capture) for event streaming to Kafka via Debezium
- Zero-downtime version upgrades

```sql
-- Publisher (source):
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- Subscriber (target):
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=primary dbname=myapp user=repl password=...'
  PUBLICATION my_pub;
```

Checklist:
- [ ] Logical replication used when column/table filtering is required
- [ ] Logical replication slots monitored for lag — same disk exhaustion risk as physical slots
- [ ] Logical replication not used as a replacement for physical streaming replication (WAL-based) in HA setups

---

## Output Report

### Critical
- Asynchronous replication used for financial/critical data with no synchronous standby — data loss on primary failure
- Replication slots not monitored — disk exhaustion possible when a replica falls behind
- No automated failover — hours of manual recovery time on primary failure

### High
- Read-after-write consistency not handled — users see stale data immediately after writing
- Replication lag alert not configured — lag grows undetected until queries return stale data
- Analytics queries running on primary — OLAP load competes with OLTP write path

### Medium
- Connection pool not split between primary and replica — all reads hit primary even though replicas are available
- Failover not tested in staging — RTO is unknown and runbook may fail in a real incident
- `max_slot_wal_keep_size` not set — WAL disk usage unbounded if replica disconnects

### Low
- Replica promotion procedure not documented — engineers improvise during an incident
- Logical replication slots not separated from physical in monitoring dashboards
- No dedicated analytics replica — reporting queries share the read replica with application reads

### Passed
- Synchronous replication configured for all critical data paths
- Replication lag monitored with alerting; application falls back to primary on high lag
- Read replicas used for analytics and general reads; primary for writes and read-after-write
- Automated failover tested with measured RTO < SLO
- Replication slots monitored and bounded with `max_slot_wal_keep_size`

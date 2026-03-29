---
name: distributed-locking
description: 'Distributed coordination and locking mechanisms across services. Use when: distributed lock, distributed locking, Redis lock, Redlock, distributed mutex, cross-service coordination, leader election, distributed coordination, database lock, advisory lock, fencing token, lock expiry, distributed concurrency, prevent concurrent execution, single active worker.'
argument-hint: 'Operation or resource requiring distributed coordination — describe the services involved, infrastructure available (Redis, PostgreSQL, etc.), and what would go wrong without a lock'
---

# Distributed Locking Specialist

## When to Use
- Ensuring only one instance of a background job runs at a time across multiple nodes
- Coordinating access to a shared external resource (API with rate limits, file, database row)
- Implementing leader election in a cluster
- Preventing duplicate processing when multiple consumers can pick up the same work
- Designing distributed mutual exclusion in a microservices environment

---

## Step 1 — Assess Whether a Distributed Lock Is Needed

Distributed locks are complex and introduce failure modes. Consider alternatives first:

| Alternative | When it's sufficient |
|---|---|
| Database row-level lock (SELECT FOR UPDATE) | Lock needed only for DB operations in same DB |
| Optimistic locking (version column) | Low contention; conflicts are rare |
| Idempotency key | Deduplication is the goal, not mutual exclusion |
| Queue with single consumer | Serial processing sufficient; queue provides ordering |
| Leader election (k8s, etcd) | Long-lived single-active-instance pattern |

**Use distributed lock when:** multiple processes on different machines must be prevented from executing the same critical section concurrently — and none of the simpler alternatives above apply.

---

## Step 2 — Redis-Based Distributed Lock (SET NX)

The simplest and most commonly correct approach for most use cases:

```python
import redis
import uuid
import time

class RedisDistributedLock:
    """
    Single-node Redis distributed lock.
    Safe for use when Redis is a single instance or primary-replica with acceptable risk.
    """
    def __init__(self, redis_client: redis.Redis, key: str, ttl_seconds: int):
        self._redis = redis_client
        self._key = f"dlock:{key}"
        self._ttl = ttl_seconds
        self._token = str(uuid.uuid4())   # unique token per lock acquisition

    def acquire(self, retry_count: int = 0, retry_delay_s: float = 0.1) -> bool:
        for attempt in range(retry_count + 1):
            # SET NX EX — atomic: set only if not exists; with expiry
            acquired = self._redis.set(
                self._key,
                self._token,          # store our token to verify on release
                nx=True,              # only set if key does not exist
                ex=self._ttl,         # auto-expire: prevents lock being held forever on crash
            )
            if acquired:
                return True
            if attempt < retry_count:
                time.sleep(retry_delay_s)
        return False

    def release(self) -> bool:
        """Release lock only if we are the current holder (using Lua script for atomicity)."""
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        # Atomic check-and-delete: prevents releasing another holder's lock
        result = self._redis.eval(script, 1, self._key, self._token)
        return bool(result)

    def __enter__(self):
        if not self.acquire():
            raise LockNotAcquiredError(self._key)
        return self

    def __exit__(self, *args):
        self.release()

# Usage
with RedisDistributedLock(redis_client, key="monthly-billing-job", ttl_seconds=300):
    run_monthly_billing()   # only one process executes this at a time
```

---

## Step 3 — TTL and Lock Expiry Design

TTL is critical: too short → lock expires while work is in progress; too long → crash leaves lock held for too long.

### TTL Sizing

```
TTL = (expected_max_operation_duration * 2) + safety_margin

Examples:
- Short DB query (< 1s)    → TTL = 10s
- API call (< 5s)          → TTL = 30s
- Batch job (< 60s)        → TTL = 180s
- Long report (< 300s)     → TTL = 900s
```

### Lock Extension (Heartbeat) for Long Operations

```python
import threading

class ExtendableLock:
    def __init__(self, redis_client, key: str, ttl: int):
        self._lock = RedisDistributedLock(redis_client, key, ttl)
        self._key = f"dlock:{key}"
        self._ttl = ttl
        self._token = self._lock._token
        self._redis = redis_client
        self._stop_event = threading.Event()
        self._heartbeat_thread = None

    def _heartbeat(self):
        """Extend TTL periodically while work is in progress."""
        # Extend every 1/3 of TTL to ensure we renew well before expiry
        interval = self._ttl / 3
        while not self._stop_event.wait(timeout=interval):
            # Only extend if we still hold the lock (token matches)
            script = """
            if redis.call("GET", KEYS[1]) == ARGV[1] then
                return redis.call("EXPIRE", KEYS[1], ARGV[2])
            else
                return 0
            end
            """
            result = self._redis.eval(script, 1, self._key, self._token, self._ttl)
            if not result:
                break  # lock was taken by someone else or expired

    def __enter__(self):
        self._lock.acquire()
        self._heartbeat_thread = threading.Thread(target=self._heartbeat, daemon=True)
        self._heartbeat_thread.start()
        return self

    def __exit__(self, *args):
        self._stop_event.set()
        self._heartbeat_thread.join(timeout=5)
        self._lock.release()
```

---

## Step 4 — Fencing Tokens (Preventing Stale Lock Actions)

A process can be paused (GC, network partition) long enough for its lock to expire. Another process acquires the lock. The original process resumes and takes actions with a stale lock. **Fencing tokens prevent stale actions from taking effect.**

```python
# Concept: monotonically increasing token issued on each lock acquisition
# Resource server rejects requests with token lower than the highest seen

class FencingTokenLock:
    def acquire(self) -> int:
        # Atomic increment — each lock grant gets a unique, increasing token
        token = self._redis.incr(f"fence:{self._key}")
        acquired = self._redis.set(
            self._key, token, nx=True, ex=self._ttl
        )
        return token if acquired else None

# Resource operation carries the token
def update_resource(resource_id: str, data: dict, fence_token: int):
    current_token = resource_db.get_fence_token(resource_id)
    if fence_token <= current_token:
        raise StaleOperationError(f"Token {fence_token} is outdated (current: {current_token})")
    resource_db.update(resource_id, data, fence_token=fence_token)
```

---

## Step 5 — Database Advisory Locks (PostgreSQL)

For operations that are anyway DB-bound, PostgreSQL advisory locks are simpler and eliminate the need for Redis:

```python
# Session-level advisory lock — released when session closes or explicitly
def process_with_pg_lock(db, lock_id: int, operation: callable):
    with db.transaction():
        # pg_try_advisory_lock returns true only if lock acquired
        is_locked = db.execute(
            "SELECT pg_try_advisory_lock(%s)", (lock_id,)
        ).scalar()

        if not is_locked:
            logger.info("Another instance holds the lock; skipping", lock_id=lock_id)
            return

        try:
            result = operation()
        finally:
            db.execute("SELECT pg_advisory_unlock(%s)", (lock_id,))

        return result

# Consistent lock ID derivation from string key
def derive_lock_id(key: str) -> int:
    # pg_try_advisory_lock takes int8; derive from hash, fit into int64
    return int(hashlib.sha256(key.encode()).hexdigest()[:15], 16)

# Usage
lock_id = derive_lock_id("monthly-billing-job")
process_with_pg_lock(db, lock_id, run_monthly_billing)
```

**Advisory lock advantages:** no TTL management; automatically released on DB connection close; integrated with transactions.

---

## Step 6 — Redlock (Multi-Node Redis)

For higher availability requirements, Redlock acquires the lock on N Redis nodes — the lock is granted only if a majority (N/2 + 1) respond:

```python
# Using the redlock-py library
from redlock import RedLock, RedLockError

redis_instances = [
    redis.Redis(host='redis-1', port=6379),
    redis.Redis(host='redis-2', port=6379),
    redis.Redis(host='redis-3', port=6379),
]

def acquire_redlock(lock_name: str, ttl_ms: int = 10000):
    dlm = RedLock(lock_name, connection_details=redis_instances, ttl=ttl_ms)
    try:
        with dlm:
            execute_critical_section()
    except RedLockError:
        logger.warning("Failed to acquire Redlock", lock=lock_name)
```

**When to use Redlock:** when a single Redis node failover leaving the lock field in an inconsistent state would be a critical business issue. For most applications, single-node Redis with acceptable TTL is sufficient.

---

## Step 7 — Common Failure Modes

| Failure mode | Cause | Mitigation |
|---|---|---|
| Lock held forever | Process crash; no TTL | Always set TTL on lock |
| Two holders simultaneously | TTL too short; process pauses (GC/network) | Heartbeat extension; fencing tokens |
| Lock never acquired | Always contended; TTL too long after crash | Monitor lock acquisition failure rate |
| Deadlock | A holds X, waits Y; B holds Y, waits X | Consistent lock ordering; never hold multiple locks if avoidable |
| Thundering herd | Many processes retry at exact intervals | Add jitter to retry delay |
| Stale operation | Process resumes after lock expired | Fencing token on resource writes |

---

## Output Report

```
## Distributed Locking Review: <job/operation>

### Critical
- No distributed lock on billing job — multiple app instances run it simultaneously
  Double billing: each instance charges every customer once → customers charged 3x (3 instances)
  Fix: acquire Redis lock "billing-job:{month}" with TTL=900s before executing; skip if lock not acquired

- Lock released by token string comparison using == instead of Lua atomic check-and-delete
  Race: process A releases lock, process B acquires, process A then deletes B's lock via second call
  Fix: use Lua script to atomically check token and delete in one operation

### High
- Lock TTL set to 30s; batch job takes up to 2 minutes on large datasets
  Lock expires mid-job; second instance acquires lock and starts duplicate execution
  Fix: set TTL=600s; implement heartbeat extension every 60s; monitor for lock loss

- Lock key does not include environment/tenant identifier
  Production and staging share same Redis; staging job can steal production lock
  Fix: prefix lock key with environment: "prod:billing-job" / "staging:billing-job"

### Medium
- Retry on failed lock acquisition uses fixed 100ms delay across all instances
  Thundering herd: all 10 instances retry simultaneously when lock holder finishes
  Fix: add jitter: delay = random.uniform(0.05, 0.3) seconds

- Lock acquisition failure logged as ERROR — PagerDuty fires on every scheduled run where lock is already held
  This is expected behavior (another instance is running); should not alert
  Fix: log as INFO when lock is not acquired (expected); ERROR only when work fails after acquiring lock

### Low
- Advisory lock ID derived by casting string to int — collision possible with other lock names
  Fix: derive from SHA-256 hash truncated to int64 range

### Passed
- PostgreSQL advisory lock auto-released on connection close (crash safety) ✓
- Lock tokens are UUID v4 — cannot be guessed by other processes ✓
- Lock acquisition failure metrics tracked and dashboarded ✓
```

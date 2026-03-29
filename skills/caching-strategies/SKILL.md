---
name: caching-strategies
description: >
  Caching layers, invalidation, and performance optimization for backend and frontend systems.
  Use when: caching strategy, cache design, cache invalidation, Redis cache, Memcached, CDN cache,
  HTTP cache, cache-aside pattern, write-through cache, write-behind cache, read-through cache,
  cache TTL, cache eviction, LRU cache, cache warming, cache stampede, thundering herd, dogpile effect,
  distributed cache, in-memory cache, application cache, cache miss, cache hit ratio,
  cache key design, cache partitioning, cache sharding, full-page cache, fragment cache,
  object cache, query result cache, cache coherence, stale-while-revalidate, cache busting.
argument-hint: >
  Describe the system and the performance problem (e.g., "Go API where every request queries
  PostgreSQL for product data that changes once per hour", "CDN not caching API responses correctly").
  Specify the cache backend in use or under consideration (Redis, Memcached, Varnish, CloudFront).
---

# Caching Strategies Specialist

## When to Use

Invoke this skill when you need to:
- Design a caching layer to reduce database load or external API calls
- Choose between cache-aside, read-through, write-through, or write-behind patterns
- Fix cache invalidation bugs causing stale data
- Prevent cache stampede (thundering herd) on cold starts or expired keys
- Design cache key schemas that are correct, consistent, and evictable
- Tune TTLs, eviction policies, and cache sizing

---

## Step 1 — Choose the Right Cache Layer

Multiple caching layers can be composed. Apply them from closest to the user inward.

| Layer | Location | Latency | Best For |
|---|---|---|---|
| **Browser cache** | User's browser | 0ms (memory hit) | Static assets, API responses with Cache-Control |
| **CDN cache** | Edge (CloudFront, Fastly, Cloudflare) | 1–10ms | Public content, images, JS/CSS, API GET responses |
| **Reverse proxy cache** | Varnish, Nginx proxy_cache | 1–5ms | Full-page HTML, semi-public API responses |
| **Application cache** | In-process memory (sync.Map, Ristretto) | < 1ms | Frequently read, rarely changed config or lookups |
| **Distributed cache** | Redis, Memcached (shared across instances) | 0.5–2ms | Session data, computed results, hot DB rows |
| **Database query cache** | DB-level (pg_bouncer, query plan cache) | 5–50ms | Repeated identical queries |

**Layer selection decision:**
- Static assets served publicly → CDN cache (no application code needed)
- Shared user session state → distributed cache (Redis)
- Hot reference data (product catalog, config) → application-level + distributed cache with L1/L2 pattern
- Expensive computed aggregations → distributed cache with long TTL

Checklist:
- [ ] Cache layer chosen based on the nature of the data (public vs. private, shared vs. per-user)
- [ ] Multiple layers composed where appropriate — CDN in front of app cache in front of DB
- [ ] Cache layer placement documented — every team member knows what is cached where

---

## Step 2 — Select a Cache Pattern

### Cache-Aside (Lazy Loading) — Most Common

```go
func GetProduct(ctx context.Context, id string) (*Product, error) {
    cacheKey := "product:" + id

    // 1. Try cache first
    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var p Product
        json.Unmarshal([]byte(cached), &p)
        return &p, nil
    }

    // 2. Cache miss — load from DB
    product, err := db.GetProduct(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. Populate cache
    data, _ := json.Marshal(product)
    redis.Set(ctx, cacheKey, data, 15*time.Minute)

    return product, nil
}
```

**When to use:** Most general-purpose scenarios. Cache only stores what's requested. Application controls cache logic.

### Read-Through

Cache layer sits in front of the DB and loads data automatically on miss. Application never talks to the DB directly.

**When to use:** When using a cache framework with built-in read-through support (e.g., Caffeine + loadingCache in Java, or a cache service abstraction).

### Write-Through

On every write → update both the DB and cache atomically.

```go
func UpdateProduct(ctx context.Context, p *Product) error {
    if err := db.UpdateProduct(ctx, p); err != nil {
        return err
    }
    data, _ := json.Marshal(p)
    redis.Set(ctx, "product:"+p.ID, data, 15*time.Minute)
    return nil
}
```

**When to use:** Strong consistency required; read-after-write must be fresh; write path can tolerate the extra latency.

### Write-Behind (Write-Back)

Write to cache immediately → flush to DB asynchronously in batches.

**When to use:** Write-heavy workloads where latency matters more than immediate durability. Risk: data loss on cache crash before flush.

**Pattern summary table:**

| Pattern | Reads | Writes | Consistency | Complexity |
|---|---|---|---|---|
| Cache-Aside | App checks cache, fetches DB on miss | Write to DB; invalidate or update cache | Eventual | Low |
| Read-Through | Cache fetches DB on miss automatically | Write to DB | Eventual | Medium |
| Write-Through | From cache | To DB + cache atomically | Strong | Medium |
| Write-Behind | From cache | To cache; async flush to DB | Eventual | High |

Checklist:
- [ ] Cache pattern chosen based on consistency requirements and write frequency
- [ ] Cache-aside used as the default unless strong consistency or write-heavy patterns require otherwise
- [ ] Write-behind avoided unless the team can handle cache-to-DB flush failure recovery

---

## Step 3 — Cache Key Design

Key design determines correctness, collisions, and eviction granularity.

**Key structure:** `<service>:<entity>:<identifier>[:<variant>]`

```
product:detail:prod_123
product:detail:prod_123:locale=pt-BR      # locale variant
user:session:sess_abc123
orders:list:user_id=456:page=1:size=20
rate_limit:user_id=789:minute=20260329T1400
```

**Rules:**
- Include all dimensions that affect the value — missing a dimension causes stale data across variants
- Use `:` as the separator — consistent, redis-friendly
- Hash long keys if they exceed 128 characters — `sha256(key)[:16]`
- Never include secrets or credentials in cache keys — keys are often visible in logs and monitoring
- Namespace keys by service — prevents accidental cross-service collisions in shared caches

**Anti-patterns:**
```
# ❌ Missing locale variant — all locales get the same cached product name
product:prod_123

# ❌ Pagination params omitted — page 1 returned for all pages
orders:list:user_id=456

# ❌ Timestamp in key — unique key per second; cache is useless
user:profile:789:ts=1743255600
```

Checklist:
- [ ] All dimensions that change the cached value are encoded in the key
- [ ] Keys namespaced by service — shared Redis has no cross-service collisions
- [ ] Pagination, locale, and filter parameters included in list query cache keys
- [ ] Key schema documented — developers know how to construct and evict keys manually

---

## Step 4 — TTL and Eviction Policy

**TTL selection:**

| Data Type | Volatility | Suggested TTL |
|---|---|---|
| Product catalog (rarely changes) | Low | 15–60 minutes |
| User profile | Medium | 5–15 minutes |
| Search results | Medium | 2–5 minutes |
| Session data | Controlled | Session lifetime (e.g., 24h with sliding TTL) |
| Rate limit counters | Fixed window | Window duration (60s) |
| Real-time inventory | High | 30–60 seconds |
| Auth tokens | Fixed | Token TTL minus buffer |
| Aggregated analytics | Low | 1–24 hours |

**Redis eviction policies (maxmemory-policy):**

| Policy | Behavior | Use When |
|---|---|---|
| `allkeys-lru` | Evict least-recently-used from all keys | General-purpose cache |
| `volatile-lru` | Evict LRU only from keys with TTL | Mix of cache + persistent data in same Redis |
| `allkeys-lfu` | Evict least-frequently-used | Hot/cold data with popularity skew |
| `noeviction` | Return error when full | When Redis is used as a primary store, not a cache |

**Recommendation:** Use `allkeys-lru` for a dedicated cache Redis instance. Never use `noeviction` for a cache.

Checklist:
- [ ] TTL set on every cached key — no keys cached indefinitely
- [ ] TTL reflects actual data change frequency — not a generic "1 hour"
- [ ] Redis `maxmemory-policy` configured explicitly — not left as `noeviction` default
- [ ] Sliding TTL used for session data — `EXPIRE` reset on access, not on write only
- [ ] TTLs monitored — alert if cache hit rate drops significantly (indicates TTL too short)

---

## Step 5 — Cache Invalidation

Cache invalidation is the hardest cache problem. Use the simplest strategy that is correct.

**Strategies:**

| Strategy | How | When |
|---|---|---|
| **TTL expiration** | Key auto-expires | Data eventually consistent is acceptable |
| **Write invalidation** | Delete key on write | Cache-aside + strong read-after-write |
| **Write-through update** | Update key on write | Write-through pattern |
| **Event-driven** | Subscribe to domain events; invalidate on change | Decoupled services; event bus available |
| **Tag-based** | Group keys under a tag; invalidate tag = evict all | CDN (Cloudflare cache tags), Varnish |

**Write invalidation (Go):**
```go
func UpdateProduct(ctx context.Context, p *Product) error {
    if err := db.UpdateProduct(ctx, p); err != nil {
        return err
    }
    // Invalidate — next read will fetch fresh from DB
    redis.Del(ctx, "product:detail:"+p.ID)
    // Also invalidate list caches that might contain this product
    redis.Del(ctx, "product:list:category:"+p.CategoryID)
    return nil
}
```

**Anti-patterns:**
- Invalidating by pattern (`KEYS product:*`) — blocks Redis for large keyspaces; use `SCAN` instead
- Not invalidating list caches when an item changes — lists show stale items after updates
- Invalidating cache before the DB write — if the DB write fails, cache is empty but data is unchanged

Checklist:
- [ ] On every write, all affected cache keys are invalidated or updated
- [ ] List caches invalidated when items in the list change
- [ ] Never use `KEYS pattern` in production — use `SCAN` for batch invalidation
- [ ] Cache invalidated AFTER a successful DB write — not before
- [ ] Event-driven invalidation used when the writer and cache owner are different services

---

## Step 6 — Cache Stampede Prevention

A cache stampede (thundering herd) occurs when a popular key expires and many concurrent requests all hit the DB simultaneously.

**Prevention strategies:**

### 1. Probabilistic Early Recomputation (PER)
Recompute the cache slightly before it expires to avoid a stampede:
```go
func GetWithEarlyRecompute(ctx context.Context, key string, ttl time.Duration, fetch func() (interface{}, error)) (interface{}, error) {
    // Check remaining TTL; if < 10% of original TTL, recompute with 1-in-N probability
    remaining, _ := redis.TTL(ctx, key).Result()
    if remaining < ttl/10 && rand.Intn(10) == 0 {
        // Proactively refresh
        val, err := fetch()
        if err == nil {
            data, _ := json.Marshal(val)
            redis.Set(ctx, key, data, ttl)
            return val, nil
        }
    }
    // Normal cache-aside
    ...
}
```

### 2. Mutex / Single-Flight (most reliable)
Only one goroutine fetches from the DB; others wait for the result:
```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func GetProduct(ctx context.Context, id string) (*Product, error) {
    result, err, _ := group.Do("product:"+id, func() (interface{}, error) {
        // Only one goroutine enters here per unique key
        return fetchFromDBAndCache(ctx, id)
    })
    if err != nil {
        return nil, err
    }
    return result.(*Product), nil
}
```

### 3. Stale-While-Revalidate
Serve stale content immediately; refresh in the background:
```
Cache-Control: max-age=60, stale-while-revalidate=3600
```

Checklist:
- [ ] `singleflight` or equivalent mutex used for high-traffic cache keys
- [ ] `stale-while-revalidate` used for HTTP and CDN responses where freshness is not critical
- [ ] Hot keys identified via cache monitoring — apply stampede protection selectively
- [ ] Cache warming strategy on deploy — pre-populate hot keys before serving traffic

---

## Output Report

### Critical
- Cache used for user-specific private data without per-user key isolation — data leaks between users
- Write-behind pattern used without crash-recovery flush — data loss on cache restart
- Cache invalidation missing after writes — users see stale data indefinitely until TTL expires

### High
- Cache stampede unmitigated on high-traffic keys — DB overwhelmed on TTL expiration
- `KEYS pattern` used in production Redis — blocks event loop; causes latency spikes for all clients
- No maxmemory-policy set — Redis silently rejects writes when full; application errors spike

### Medium
- TTL not set on some cache keys — keys accumulate indefinitely; memory grows unbounded
- List caches not invalidated on item update — stale lists served after mutations
- Cache key missing key dimensions (locale, pagination) — wrong data returned for some request variants

### Low
- No cache monitoring (hit ratio, memory usage, eviction rate) — cache effectiveness invisible
- `noeviction` policy on a cache Redis shared with a primary store — cache and store compete for memory
- Cache TTL chosen arbitrarily (1h for everything) — not aligned to actual data change frequency

### Passed
- Cache layer selected appropriate to data visibility, sharing, and consistency requirements
- Cache-aside pattern implemented correctly: check → miss → fetch → populate; invalidate on write
- Cache keys include all dimensions; namespaced by service; TTL set on all keys
- Redis maxmemory-policy set to `allkeys-lru`; cache sizing monitored and alerted
- Stampede prevention via `singleflight` on hot keys; stale-while-revalidate on CDN responses
- Invalidation covers item caches AND affected list caches; invalidation occurs after successful writes

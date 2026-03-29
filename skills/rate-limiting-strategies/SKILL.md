---
name: rate-limiting-strategies
description: 'API rate limiting, throttling, abuse prevention, and adaptive request control. Use when: rate limiting, API throttling, request throttling, rate limit design, abuse prevention, quota management, token bucket, sliding window, fixed window, leaky bucket, per-user rate limit, per-IP rate limit, Redis rate limiting, rate limit middleware, API quota, burst limit, DDoS prevention, endpoint protection.'
argument-hint: 'API endpoints, middleware, or gateway config to review — or describe the rate limiting strategy being designed'
---

# Rate Limiting Strategies Specialist

## When to Use
- Designing rate limiting for a new API or service
- Reviewing existing rate limit configuration for gaps or bypasses
- Choosing between rate limiting algorithms for a specific use case
- Protecting specific endpoints (login, SMS, payment, search) from abuse
- Implementing quota management for multi-tenant or SaaS APIs

---

## Step 1 — Understand the Protection Goals

Rate limiting serves multiple purposes — identify which apply:

| Goal | Focus |
|---|---|
| Abuse prevention | Stop credential stuffing, scraping, spam |
| Fair usage | Prevent one user from monopolizing shared resources |
| Cost control | Cap expensive operations (email, SMS, external API calls) |
| Service availability | Prevent resource exhaustion from excessive requests |
| Business rules | Enforce plan-based quotas (free: 100/day, pro: 10k/day) |

---

## Step 2 — Choose the Right Algorithm

| Algorithm | How it works | Best for |
|---|---|---|
| **Fixed Window** | Count resets every N seconds (e.g., per minute) | Simple; can burst at window boundary |
| **Sliding Window Log** | Track timestamps of each request; count within rolling window | Accurate; memory-intensive at scale |
| **Sliding Window Counter** | Interpolate between fixed windows; approximation | Good balance; Redis-friendly |
| **Token Bucket** | Tokens refill at fixed rate; burst allowed up to bucket size | APIs that allow short bursts |
| **Leaky Bucket** | Requests queued; processed at fixed rate; overflow dropped | Smooth output rate; queuing systems |

**Recommendation guide:**
- **Login / auth endpoints**: Sliding window (accurate, burst-resistant)
- **General API**: Token bucket (allows legitimate bursts)
- **Expensive ops (SMS, email, payment)**: Fixed window with small limits
- **Streaming / real-time**: Leaky bucket

---

## Step 3 — Rate Limit Granularity

Apply rate limits at multiple levels simultaneously:

| Level | Key | Use |
|---|---|---|
| Per IP | `limit:{ip}` | First line of defense; anonymous abuse |
| Per user | `limit:{user_id}` | Authenticated abuse; per-account fairness |
| Per API key | `limit:{api_key}` | B2B / machine-to-machine quotas |
| Per tenant | `limit:{tenant_id}` | SaaS multi-tenant fairness |
| Per endpoint | `limit:{user_id}:{endpoint}` | Protect specific expensive or sensitive paths |
| Global | `limit:global:{endpoint}` | Total throughput cap; service protection |

**Combine layers**: a request passes both per-IP AND per-user limits; the stricter one wins.

---

## Step 4 — Endpoint-Specific Limits

Critical endpoints need stricter, purpose-built limits:

| Endpoint | Recommended limit | Notes |
|---|---|---|
| POST /login | 5/15min per account + 20/min per IP | Prevent credential stuffing |
| POST /register | 3/hour per IP | Prevent mass account creation |
| POST /password-reset | 3/hour per account | Prevent enumeration + abuse |
| POST /send-sms / /send-email | 5/hour per user | Prevent SMS/email bombing |
| POST /payment / /charge | 10/hour per user | Prevent card testing |
| GET /search | 60/min per user | Prevent data scraping |
| POST /export | 5/hour per user | Prevent bulk data exfiltration |
| File upload | 20/hour per user + size quota | Prevent storage abuse |
| Public API (free tier) | 100/day per key | Plan-based quota |

---

## Step 5 — Implementation Patterns

### Redis-Based Sliding Window (recommended)

```lua
-- Redis Lua script for sliding window rate limit
local key = KEYS[1]
local now = tonumber(ARGV[1])      -- current timestamp in ms
local window = tonumber(ARGV[2])   -- window size in ms
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now)
    redis.call('PEXPIRE', key, window)
    return 1   -- allowed
else
    return 0   -- rejected
end
```

### Middleware Integration Example (Go)

```go
func RateLimitMiddleware(limiter *ratelimit.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := limiter.KeyFor(r)   // extract user ID or IP
            result, err := limiter.Allow(r.Context(), key)
            if err != nil {
                http.Error(w, "service unavailable", 503)
                return
            }
            // Always set rate limit headers
            w.Header().Set("X-RateLimit-Limit", strconv.Itoa(result.Limit))
            w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(result.Remaining))
            w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(result.ResetAt.Unix(), 10))
            if !result.Allowed {
                w.Header().Set("Retry-After", strconv.Itoa(result.RetryAfter))
                http.Error(w, "Too Many Requests", 429)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## Step 6 — Response Headers

Always communicate rate limit status to clients:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1711656000

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1711656000
Retry-After: 47
Content-Type: application/json

{"error": "rate_limit_exceeded", "retry_after": 47}
```

---

## Step 7 — Bypass Prevention

| Bypass technique | Defense |
|---|---|
| Rotating IPs | Rate limit by account, not just IP; require auth for sensitive endpoints |
| Distributed attack (many IPs, one account) | Per-account rate limits independent of IP |
| X-Forwarded-For header spoofing | Use trusted proxy config; only trust X-Forwarded-For from known proxies |
| IPv6 rotation | Limit by /64 prefix, not individual IPv6 address |
| Multi-account abuse | Rate limit by device fingerprint, email domain; require phone verification |
| Header manipulation | Extract real IP from validated proxy chain; never trust client headers directly |

**Never trust client-supplied IP headers unless they come from a trusted reverse proxy.**

---

## Step 8 — Adaptive Rate Limiting

For advanced abuse prevention, implement adaptive limits:

| Signal | Adaptive action |
|---|---|
| High failure rate (repeated 401/403) | Tighten limit for that key; require CAPTCHA |
| Unusual access pattern (scraping) | Slow down responses; add latency penalty |
| Known bad ASN/datacenter IP | Apply stricter limits automatically |
| Account age < 24h | Apply stricter limits than established accounts |
| Anomalous request volume spike | Trigger alert and tighten global limits |

---

## Step 9 — Output Report

```
## Rate Limiting Review: <service/API>

### Critical
- POST /auth/login — no rate limiting
  Credential stuffing possible without resistance
  Fix: 5 attempts per account per 15 min; 20 per IP per minute; lockout with exponential backoff

- POST /sms/send — no limit per user
  SMS bombing attack possible; per-SMS cost unbounded
  Fix: 5 SMS per user per hour; require verified phone for additional quota

### High
- Rate limiting uses X-Forwarded-For from client directly
  Attacker spoofs header to bypass IP-based limits
  Fix: configure trusted proxy list; derive real IP from trusted proxy tier only

- No per-account limit on GET /api/export
  Attacker with valid account can bulk-scrape all data
  Fix: 5 exports per user per hour; async with queue for large exports

### Medium
- Rate limit window is fixed (not sliding) — users can burst 2x at window boundary
  Fix: migrate to sliding window counter for sensitive endpoints

- 429 responses include no Retry-After header
  Fix: add Retry-After and X-RateLimit-Reset to all 429 responses

### Low
- Public endpoints have no rate limiting at all
  Fix: apply 100 req/min per IP on all unauthenticated endpoints

### Passed
- Redis sliding window used for auth endpoints ✓
- Per-user AND per-IP limits applied in combination ✓
- Rate limit headers present on all responses ✓
```

---
name: ddos-mitigation
description: 'Service protection strategies against denial-of-service attacks and traffic floods. Use when: DDoS mitigation, denial of service, traffic flood, volumetric attack, layer 7 DDoS, application DDoS, slowloris, amplification attack, DDoS protection, HTTP flood, SYN flood, CDN DDoS protection, WAF DDoS, connection limit, request throttling DDoS, service availability.'
argument-hint: 'Service, infrastructure config, or architecture to review for DDoS resilience — or describe the availability protection strategy being designed'
---

# DDoS Mitigation Specialist

## When to Use
- Designing DDoS resilience for a web service or API
- Reviewing infrastructure and application configuration for flood attack exposure
- Choosing a DDoS protection strategy (CDN, WAF, on-premise scrubbing)
- Hardening against application-layer (L7) DDoS and resource exhaustion attacks
- Defining capacity, rate limiting, and failover strategies for high-availability services

---

## Step 1 — Classify the DDoS Type

| Layer | Attack type | Target | Defense focus |
|---|---|---|---|
| L3/L4 — Network | Volumetric flood (UDP/ICMP), SYN flood, TCP exhaustion | Bandwidth, connection tables | ISP scrubbing, Anycast, upstream provider protection |
| L7 — Application | HTTP flood, Slowloris, RUDY, search/heavy endpoint abuse | CPU, DB, memory, application threads | Rate limiting, request validation, WAF, caching |
| Amplification | DNS amplification, NTP amplification (spoofed source) | Bandwidth | BCP38 filtering, response rate limiting (RRL) at DNS |
| Resource exhaustion | Expensive DB queries, file parsing, search indexing | Backend compute | Request throttling, async queuing, circuit breakers |

> **Application-layer (L7) DDoS** is the primary concern for most application teams — volumetric L3/L4 is typically handled by CDN/ISP providers.

---

## Step 2 — Defense Architecture Layers

Use **defense in depth** — no single layer is sufficient:

```
Internet
  ↓
[CDN / DDoS scrubbing] ← absorb volumetric; anycast; global traffic distribution
  ↓
[WAF / Edge rate limiting] ← block L7 floods; challenge suspicious IPs; filter bad UA/patterns
  ↓
[Load balancer] ← connection limits; TCP syn cookie; health checks; sticky sessions management
  ↓
[Application server] ← request timeouts; concurrency limits; graceful degradation
  ↓
[Database / backend] ← query timeouts; connection pool limits; read replicas; circuit breakers
```

---

## Step 3 — CDN and Anycast Protection

| Provider | Capabilities |
|---|---|
| Cloudflare | Anycast DDoS scrubbing, Bot Management, WAF, rate limiting, Under Attack Mode |
| AWS Shield + CloudFront | L3/L4 scrubbing (Shield Standard free); L7 with Shield Advanced + WAF |
| Akamai / Prolexic | Enterprise scrubbing; high-capacity volumetric mitigation |
| GCP Cloud Armor | L7 WAF + rate limiting; DDoS protection on Google Front End |
| Fastly | Edge rate limiting; WAF; image/cache optimization reduces origin load |

**Anycast advantage**: traffic is absorbed across global PoPs — no single origin is overwhelmed.

**Enable these CDN features:**
- [ ] DDoS mode / Under Attack mode (activates JS challenge globally)
- [ ] Bot management / challenge pages for suspicious traffic
- [ ] Edge rate limiting per IP and per path
- [ ] WAF ruleset enabled with appropriate sensitivity
- [ ] Cache everything cacheable — reduces origin requests per flood

---

## Step 4 — Application-Layer (L7) Hardening

Even with CDN protection, harden the application itself:

### Request Rate Limiting
- Apply rate limits at the edge AND the application (see `rate-limiting-strategies` skill).
- Stricter limits on expensive endpoints (search, export, reporting).
- Global concurrency cap — reject new requests if worker threads/goroutines exceed limit.

### Connection and Timeout Limits

```nginx
# Nginx — connection and timeout hardening
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
limit_conn conn_per_ip 20;              # max 20 concurrent connections per IP

limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=30r/s;
limit_req zone=req_per_ip burst=50;     # 30 req/s; burst of 50; queue others

client_body_timeout   10s;              # disconnect clients that stop sending body
client_header_timeout 10s;
keepalive_timeout      15s;
send_timeout           10s;
```

```
# Application server: always set request read + response write timeouts
# Go
srv := &http.Server{
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       120 * time.Second,
    ReadHeaderTimeout: 2 * time.Second,
}
```

### Slowloris Defense
Slowloris sends slow, partial HTTP requests to exhaust connection limits:
- [ ] Set `client_header_timeout` and `client_body_timeout` short (5–10s)
- [ ] Limit max concurrent connections per IP
- [ ] Use non-blocking I/O (async) servers — Go, Node.js, nginx are naturally resistant

### Expensive Endpoint Protection
- [ ] Cache results of expensive queries (search, aggregations) with short TTLs
- [ ] Move expensive work to async job queues — return 202 Accepted immediately
- [ ] Apply per-endpoint stricter rate limits before DB/cache is hit
- [ ] Return cached "degraded" responses under high load rather than hitting the DB

---

## Step 5 — Infrastructure Hardening

| Control | Implementation |
|---|---|
| Connection pool limits | DB connection pool capped; excess requests queued not rejected immediately |
| Query timeouts | DB queries time out after N seconds; never run indefinitely |
| Circuit breakers | Stop calling degraded downstream services; return cached fallback |
| Autoscaling | Horizontal scale-out on CPU/request count; but don't scale into a DDoS |
| Resource isolation | Separate DB instances for read/write; reporting DB isolated from OLTP |
| Health checks | Load balancer removes unhealthy instances; attack doesn't cascade |
| Graceful degradation | Non-critical features disabled under load (recommendations, suggestions) |

---

## Step 6 — DNS DDoS Considerations

- Use a managed DNS provider with built-in DDoS protection (Cloudflare DNS, AWS Route 53, NS1).
- Enable DNSSEC to prevent cache poisoning.
- Set low TTLs on critical records — allows fast failover if origin is attacked.
- Use multiple DNS providers (secondary DNS) for redundancy.
- Never expose origin server IPs — route all traffic through CDN to prevent direct-to-origin attacks.

---

## Step 7 — Monitoring and Response

### Signals to Monitor
| Signal | Threshold | Action |
|---|---|---|
| Requests per second (global) | > 5× baseline | Alert; activate CDN Under Attack mode |
| 5xx error rate | > 1% | Alert; check capacity and upstream |
| CPU / memory spike | > 80% | Alert; autoscale; circuit break |
| New IP range flooding | Any | Block at edge; null route if volumetric |
| DB connection saturation | > 90% of pool | Alert; shed non-essential traffic |

### Incident Response Playbook
1. **Detect**: automated alert from monitoring.
2. **Classify**: volumetric (L3/L4) vs application (L7)?
3. **Engage CDN**: activate Under Attack mode; enable challenge pages.
4. **Block patterns**: identify attack fingerprint (UA, header pattern, endpoint); add WAF rule.
5. **Shed load**: disable non-critical features; enable degraded mode.
6. **Notify**: inform stakeholders; document timeline.
7. **Restore**: gradually disable mitigations as traffic normalizes.
8. **Post-mortem**: document attack type, detection time, response time, improvements.

---

## Step 8 — Output Report

```
## DDoS Mitigation Review: <service/infrastructure>

### Critical
- No CDN or DDoS scrubbing in front of origin servers
  Direct HTTP flood to origin can saturate bandwidth/CPU without any mitigation layer
  Fix: deploy behind Cloudflare or AWS CloudFront + Shield; never expose origin IP publicly

- No HTTP server timeouts configured — Slowloris attack will exhaust connections
  Fix: set client_header_timeout=5s, client_body_timeout=10s in nginx; ReadHeaderTimeout in Go

### High
- GET /search — no rate limiting; runs unbounded DB full-text query
  10 concurrent requests per attacker IP exhaust DB CPU
  Fix: rate limit to 20/min per IP; cache results for 30 seconds; add query timeout of 3s

- DB connection pool has no limit — DDoS of endpoint can exhaust all DB connections
  Fix: cap pool at 80% of DB max_connections; queue excess; return 503 under extreme load

### Medium
- No autoscaling configured — sustained attack consumes all fixed capacity
  Fix: configure horizontal autoscaling on CPU > 60%; set max instance cap to control costs

- Monitoring has no DDoS-specific alert thresholds
  Fix: add alert for req/s > 5× baseline and 5xx rate > 1%

### Low
- DNS TTL set to 86400s (24 hours) — failover during attack is slow
  Fix: reduce critical DNS records to 300s TTL to enable fast failover

### Passed
- Cloudflare WAF enabled with default ruleset ✓
- Rate limiting applied at edge (30 req/s per IP) ✓
- Circuit breaker in place for downstream payment service ✓
```

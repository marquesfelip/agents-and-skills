---
name: load-testing
description: >
  Performance and scalability testing practices for APIs and backend systems.
  Use when: load testing, performance testing, scalability testing, stress testing, soak testing,
  spike testing, k6, Gatling, JMeter, Locust, Artillery, throughput testing, latency testing,
  p95 p99 latency, requests per second, RPS, TPS, performance baseline, performance regression,
  performance CI gate, API performance, load test scenario, virtual users, VUs, ramp-up,
  capacity planning, bottleneck identification, database under load, connection pool exhaustion,
  memory leak under load, performance benchmark, load test report.
argument-hint: >
  Describe the target system and performance goal (e.g., "checkout API must handle 500 RPS at p99 < 300ms"),
  and specify the tool, load profile (ramp-up/steady/spike), or environment.
---

# Load Testing Specialist

## When to Use

Invoke this skill when you need to:
- Establish a performance baseline for an API or service
- Detect performance regressions before releasing to production
- Validate that the system meets SLOs under expected and peak load
- Identify bottlenecks in the stack (DB, cache, CPU, network)
- Plan capacity for anticipated traffic growth
- Design and execute stress, soak, or spike test scenarios

---

## Step 1 — Define Performance Goals

Before writing a single test, define measurable targets:

| Metric | Example Target |
|---|---|
| **Throughput** | ≥ 500 RPS sustained |
| **p50 latency** | < 50ms |
| **p95 latency** | < 200ms |
| **p99 latency** | < 500ms |
| **Error rate** | < 0.1% |
| **Availability** | ≥ 99.9% during peak |

Checklist:
- [ ] SLOs defined for each critical endpoint
- [ ] Expected traffic profile documented (steady, seasonal peaks, burst)
- [ ] Acceptable degradation levels defined (e.g., p99 may increase 2x under 2x traffic)
- [ ] Metrics agreed with product/engineering stakeholders before testing

---

## Step 2 — Choose the Right Load Test Type

| Test Type | Goal | Duration | Load Level |
|---|---|---|---|
| **Baseline** | Measure current performance at normal load | 5–15 min | Expected average RPS |
| **Load** | Validate SLOs under expected peak | 15–30 min | Expected peak RPS |
| **Stress** | Find the breaking point | 30–60 min | Gradually increase until degradation |
| **Spike** | Validate behavior under sudden burst | 10–20 min | Instant 10x–50x of baseline |
| **Soak** | Detect memory leaks and resource exhaustion | 2–24 hours | Sustained moderate load |
| **Breakpoint** | Identify max capacity | Varies | Binary search to failure point |

Checklist:
- [ ] Test type chosen based on the specific risk being validated
- [ ] Baseline test run before stress or spike tests
- [ ] Soak test included before major releases if service runs long-lived processes

---

## Step 3 — Choose Load Testing Tool

| Tool | Best For | Language |
|---|---|---|
| **k6** | Developer-friendly, CI-native, scripted scenarios | JavaScript |
| **Gatling** | High-throughput, Scala/Java ecosystems | Scala / Java |
| **Locust** | Python ecosystem, swarm simulation | Python |
| **Artillery** | YAML-first, quick setup | YAML / JS |
| **JMeter** | Enterprise, complex correlation, GUI | Java |
| **wrk / wrk2** | HTTP microbenchmarks (single endpoint) | Lua |
| **hey / vegeta** | Simple CLI load generation | Go |

Recommendation: **k6** for most teams — scriptable, CI-friendly, cloud option available.

Checklist:
- [ ] Tool selected matches team's scripting language preference
- [ ] Tool supports the protocol used (HTTP, WebSocket, gRPC)
- [ ] Tool can output metrics in a format compatible with CI and dashboards

---

## Step 4 — Design Load Test Scenarios

Realistic scenarios must reflect production traffic patterns:

```javascript
// k6 example structure
export const options = {
  scenarios: {
    ramp_up: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },   // ramp-up
        { duration: '5m', target: 100 },   // steady state
        { duration: '2m', target: 0 },     // ramp-down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    http_req_failed: ['rate<0.001'],
  },
};
```

Checklist:
- [ ] Scenario includes **ramp-up, steady state, and ramp-down** phases
- [ ] Virtual user think time modeled (real users don't fire requests instantly)
- [ ] Traffic mix reflects production distribution (e.g., 60% GET, 30% POST, 10% PUT)
- [ ] Authentication tokens / session management handled in test setup
- [ ] Unique data used per VU to prevent artificial cache hit rates
- [ ] Thresholds defined and test **fails** if thresholds are breached

---

## Step 5 — Configure Test Environment

Checklist:
- [ ] Load test runs against a **dedicated performance environment** — never against production
- [ ] Environment is production-equivalent in topology (same instance types, DB, cache)
- [ ] DB is pre-seeded with production-scale data (or representative volume)
- [ ] Monitoring enabled in test environment (APM, DB slow query log, infra metrics)
- [ ] No other traffic sharing the environment during the test
- [ ] Connection pool sizes, timeouts, and rate limits match production config

---

## Step 6 — Analyze Results

After the test, analyze:

**Throughput & Latency:**
- Did throughput plateau before target RPS? → Bottleneck exists
- Did p95/p99 latency spike at a threshold? → Find the inflection point

**Error Analysis:**
- What error types appeared (timeout, 503, connection refused)?
- At what RPS did errors start?

**Infrastructure Signals:**
- CPU saturation (> 80%): compute bottleneck
- Memory growth (not returning to baseline): memory leak
- DB connection pool exhausted: increase pool or optimize query count
- Disk I/O saturation: slow queries, missing indexes

**Flame graphs / profiling:**
- Run CPU profiler during the load test to identify hot code paths

Checklist:
- [ ] Latency percentiles (p50, p95, p99, p999) captured and compared to targets
- [ ] Error rate tracked over time — did it increase gradually or spike suddenly?
- [ ] DB slow query log reviewed for queries that degrade under concurrent load
- [ ] APM trace analyzed for the slowest request paths
- [ ] Infrastructure metrics exported (CPU, memory, disk, network) and correlated with load

---

## Step 7 — CI Integration

Integrate performance baseline into CI to catch regressions:

```yaml
performance-test:
  stage: integration
  script:
    - k6 run --out json=results.json scenarios/load.js
  artifacts:
    paths:
      - results.json
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

Checklist:
- [ ] Baseline thresholds committed to version control alongside the test script
- [ ] CI fails the build if thresholds are breached
- [ ] Performance test results stored as artifacts for trend analysis
- [ ] Trend compared across commits to detect gradual regression

---

## Output Report

### Critical
- System cannot meet target RPS even with no concurrency — fundamental bottleneck
- Error rate exceeds 1% under expected load — SLO breach in production
- Memory grows continuously during soak test without returning to baseline — memory leak

### High
- p99 latency breaches SLO threshold under expected peak load
- DB connection pool exhausted under load — queries queuing
- No performance environment available — test cannot be run without affecting production

### Medium
- Load test runs against production-unlike environment — results are not representative
- No ramp-up phase — results include cold-start latency artifacts
- Virtual users use the same test data record, creating artificial cache hits

### Low
- Test lacks think time between requests — overestimates real throughput
- No CI integration — performance regressions are detected manually
- Load test results not archived — no trend analysis possible

### Passed
- System meets all SLO thresholds under expected peak load
- Error rate < 0.1% at target RPS
- Soak test shows stable memory and CPU over 2+ hours
- CI performance gate catches regressions before merge
- Bottleneck analysis documented with infrastructure metrics

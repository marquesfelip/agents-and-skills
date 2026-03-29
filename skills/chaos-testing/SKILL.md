---
name: chaos-testing
description: >
  Failure injection and resilience validation strategies for distributed systems.
  Use when: chaos testing, chaos engineering, failure injection, resilience testing, fault injection,
  Chaos Monkey, Chaos Toolkit, Gremlin, Litmus, Toxiproxy, network failure simulation,
  service unavailability test, dependency failure, latency injection, packet loss simulation,
  circuit breaker test, retry test, bulkhead test, timeout validation, graceful degradation test,
  resilience validation, site reliability, SRE chaos, GameDay, blast radius, steady state,
  hypothesis driven chaos, failure mode, chaos experiment, pod kill, node failure, disk failure.
argument-hint: >
  Describe the system and the failure scenario to explore (e.g., "e-commerce checkout service — validate behavior when PaymentService is unavailable"),
  and specify the tool, environment, and blast radius constraints.
---

# Chaos Testing Specialist

## When to Use

Invoke this skill when you need to:
- Validate that a system degrades gracefully when dependencies fail
- Test circuit breakers, retries, bulkheads, and fallbacks under real failure conditions
- Run a chaos experiment or GameDay on a production or staging environment
- Identify unknown failure modes before they become incidents
- Build confidence in resilience patterns through empirical evidence

---

## Step 1 — Define the Steady State

Before injecting failures, define what "healthy" looks like:

| Signal | Healthy Threshold | Measurement Method |
|---|---|---|
| Error rate | < 0.5% | APM / metrics |
| p99 latency | < 300ms | Prometheus / Datadog |
| Availability | ≥ 99.9% | Uptime monitor |
| Queue depth | < 1000 messages | Broker metrics |
| CPU / Memory | < 70% | Infra metrics |

Checklist:
- [ ] Steady state metrics defined before any experiment
- [ ] Steady state can be measured automatically (not manually)
- [ ] Baseline captured immediately before the experiment starts
- [ ] Rollback criteria defined: revert the experiment if any steady state metric breaches threshold

---

## Step 2 — Formulate a Hypothesis

Every chaos experiment is a **hypothesis test**:

```
"When [failure condition], we believe [system component] will [expected behavior]
because [resilience mechanism], measured by [observable metric]."
```

Examples:
- "When PaymentService returns 503, CheckoutService will use the cached payment result and return 200 to the user, measured by checkout error rate remaining < 1%."
- "When 50% of database connections are dropped, the connection pool will recover within 30 seconds, measured by p99 query latency returning to baseline."

Checklist:
- [ ] Hypothesis is falsifiable (has a measurable outcome)
- [ ] The resilience mechanism being tested is explicitly named
- [ ] Experiment scope (blast radius) is understood and bounded
- [ ] Confidence level required before running in production is defined

---

## Step 3 — Choose Chaos Tools

| Tool | Failure Types | Environment |
|---|---|---|
| **Chaos Monkey** | Instance termination | AWS, Netflix OSS |
| **Gremlin** | Network, CPU, memory, state attacks | Any cloud / k8s |
| **Chaos Toolkit** | Open source, multi-driver | Any |
| **Litmus** | Kubernetes-native chaos | Kubernetes |
| **Toxiproxy** | Network proxy: latency, bandwidth, disconnect | Local / CI / staging |
| **tc (Linux)** | Kernel-level network degradation | Linux hosts |
| **Pumba** | Docker container chaos | Docker |
| **AWS FIS** | AWS-native failure injection | AWS |
| **Azure Chaos Studio** | Azure-native | Azure |

Checklist:
- [ ] Tool selected matches the infrastructure (bare metal, Docker, Kubernetes, cloud)
- [ ] Tool can be automated and integrated with CI/CD or GameDay runbook
- [ ] Blast radius can be scoped (single pod, single AZ, % of traffic)

---

## Step 4 — Design the Experiment

Structure each chaos experiment as a runbook:

```yaml
# Chaos Toolkit experiment structure (pseudocode)
title: "PaymentService unavailability handling"
description: "Verify CheckoutService degrades gracefully when PaymentService is down"

steady-state-hypothesis:
  - probe: checkout_error_rate < 0.01
  - probe: p99_checkout_latency < 500ms

method:
  - action: toxiproxy_disconnect(target=payment-service, duration=60s)
  - action: generate_load(rps=100, duration=60s)

rollbacks:
  - action: toxiproxy_reconnect(target=payment-service)

observations:
  - probe: checkout_error_rate
  - probe: p99_latency
  - probe: fallback_cache_hit_rate
```

Checklist:
- [ ] Experiment has a clear title and success/failure definition
- [ ] Steady state probes run before and after the failure injection
- [ ] Automatic rollback configured to restore normal conditions
- [ ] Blast radius is bounded (single service, single region, canary % of traffic)
- [ ] Load generation included to stress the system during failure
- [ ] All team members are notified before running in production (GameDay)

---

## Step 5 — Failure Scenarios Catalog

Run experiments across these failure categories:

**Service Failures:**
- [ ] Downstream dependency returns 503 (all requests)
- [ ] Downstream dependency returns 503 (50% of requests)
- [ ] Downstream dependency responds with 30-second delay (timeout trigger)
- [ ] Downstream dependency returns malformed response body

**Network Failures:**
- [ ] Network partition between two services (full disconnect)
- [ ] 200ms added latency on all requests to a dependency
- [ ] 30% packet loss to a dependency
- [ ] DNS resolution failure for a dependency

**Infrastructure Failures:**
- [ ] Kill one pod / container in a replicated service
- [ ] Kill all pods in one AZ (AZ failure simulation)
- [ ] Reduce node CPU to 5% (resource starvation)
- [ ] Fill disk to 95% on a DB host

**Data Failures:**
- [ ] DB primary fails — verify replica promotion and reconnect
- [ ] Cache eviction — verify DB fallback under increased load
- [ ] Message broker unavailable — verify queue durability and replay

---

## Step 6 — Analyze Results

After each experiment:

1. **Hypothesis confirmed?** Resilience mechanism worked as designed
2. **Hypothesis rejected?** System degraded unexpectedly — document the failure mode
3. **Steady state restored?** System recovered within expected time after failure removed

Checklist:
- [ ] Error rate during failure recorded and compared to hypothesis
- [ ] Time to detect failure (TTD) and time to recover (TTR) measured
- [ ] Unexpected side effects documented (e.g., cascading failures to unrelated services)
- [ ] Circuit breaker opened and closed at the expected thresholds
- [ ] Retry storms and thundering herd not triggered during recovery
- [ ] Experiment result stored in a shared knowledge base (GameDay report)

---

## Step 7 — Iterate and Schedule

Chaos testing is continuous, not a one-time event:

Checklist:
- [ ] Failed experiments trigger a resilience improvement backlog item
- [ ] Experiments that pass are scheduled for periodic re-execution (quarterly)
- [ ] New experiments designed for every new external dependency introduced
- [ ] Chaos suite runs automatically in staging on every release candidate
- [ ] GameDay sessions scheduled before major traffic events (product launches, sale events)

---

## Output Report

### Critical
- System becomes completely unavailable when a single non-critical dependency fails (no fallback)
- Retry loops during dependency failure cause a thundering herd that worsens the outage
- Circuit breaker not implemented — every request waits for timeout during dependency failure

### High
- System recovers but takes > 5 minutes after dependency is restored — too slow for SLO
- Error messages during failure expose internal stack traces or infrastructure details to users
- No automatic rollback in the experiment — manual intervention required to restore

### Medium
- Circuit breaker opens correctly but does not re-probe and close after recovery
- Bulkhead not configured — dependency failure threads block unrelated request processing
- No load generated during failure injection — experiment does not reflect real conditions

### Low
- Experiments run only in local environments — production failure modes not validated
- Chaos experiments not documented — team has no shared knowledge of tested scenarios
- No periodic re-execution — passing experiments may regress silently

### Passed
- All tested failure scenarios result in graceful degradation within hypothesis thresholds
- Circuit breakers, retries, and fallbacks verified to function under real failure conditions
- System recovers automatically within SLO after failure is removed
- GameDay results documented and tracked in improvement backlog

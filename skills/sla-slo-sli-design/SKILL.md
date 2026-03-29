---
name: sla-slo-sli-design
description: >
  Reliability objectives design, SLI measurement, SLO target setting, and error budget management.
  Use when: SLA SLO SLI design, service level agreement, service level objective, service level indicator,
  error budget, reliability target, availability target, uptime target, reliability engineering,
  SRE SLO, latency SLO, availability SLO, error rate SLO, SLO burn rate, error budget policy,
  SLO dashboard, SLI measurement, SLO alert, reliability metrics, four nines uptime, five nines,
  99.9% availability, 99.99% availability, reliability objective design, SLO review, SLO violation,
  SLO tracking, uptime SLA, customer-facing SLA, user journey SLO, SLO instrumentation,
  SLO Prometheus, SLO Grafana, SLO Datadog, SLO framework, reliability budget, downtime budget.
argument-hint: >
  Describe the service and user-facing operations to protect (e.g., "order placement, inventory lookup,
  payment processing"), the current reliability posture (is there any SLO defined?), and the business
  context (customer-facing SaaS, internal tool, regulated industry).
---

# SLA / SLO / SLI Design Specialist

## When to Use

Invoke this skill when you need to:
- Define SLIs (what to measure), SLOs (what targets to set), and SLAs (what to promise customers)
- Instrument and track SLOs in Prometheus/Grafana, Datadog, or OTEL
- Calculate error budgets and define error budget policies
- Design SLO-based alerts that page on meaningful degradation
- Review and calibrate existing SLOs that are too tight, too loose, or unmeasured
- Communicate reliability objectives to product, business, and customers

---

## Step 1 — Understand the Hierarchy

```
SLA (Service Level Agreement)
  └─ The contractual promise to customers: "99.9% uptime per month"
       └─ Backed by SLOs (Service Level Objectives)
            └─ Internal engineering targets: "99.95% availability — stricter than SLA"
                 └─ Measured by SLIs (Service Level Indicators)
                      └─ The metric: "fraction of successful HTTP requests"
```

| Term | What it is | Who sets it | Consequence of breach |
|---|---|---|---|
| **SLI** | The measurement (ratio, latency value) | Engineering (SRE) | None — it's just a metric |
| **SLO** | The target for the SLI | Engineering + Product | Error budget consumed → policy triggers |
| **SLA** | The contractual commitment to the customer | Business + Legal | Credits, penalties, churn |

**Key principle:** SLO is always stricter than SLA. If the SLO is breached → engineering fixes it before it becomes an SLA breach.

```
SLA target: 99.9% → SLO target: 99.95% → engineering buffer: 0.05% (extra error budget)
```

---

## Step 2 — Choose the Right SLI Type

SLIs must directly measure the user experience — not proxy metrics like CPU or memory.

| SLI Type | Measures | Formula |
|---|---|---|
| **Availability** | Fraction of successful requests | `good_requests / total_requests` |
| **Latency** | Fraction of requests below a threshold | `requests < 500ms / total_requests` |
| **Throughput** | Rate of successful completions | Successful operations per second |
| **Error rate** | Fraction of requests that errored | `error_requests / total_requests` |
| **Freshness** | Data or content recency | Age of most recent record |
| **Durability** | Data preserved over time | Data retained / data stored (for storage systems) |

**User journey SLIs — pick SLIs per critical user action:**
```
User journey: "Place an Order"
  SLI 1: 99.95% of POST /orders requests succeed (availability)
  SLI 2: 95% of POST /orders requests complete in < 2 seconds (latency)
  SLI 3: 0% of successfully placed orders lost within 24 hours (durability)
```

**Good vs. bad SLI choices:**
```
✅ Fraction of /checkout requests returning 2xx
✅ Fraction of /orders/{id} responses with latency < 500ms
✅ Fraction of payment events processed within 30 seconds

❌ CPU utilization < 80%           — not user-facing
❌ Pod restart count               — cause, not symptom
❌ Deployment success rate         — internal, not user-facing
```

Checklist:
- [ ] SLIs defined per critical user journey — not generic "service availability"
- [ ] SLIs are ratios (0 to 1) — not absolute numbers or resource metrics
- [ ] Latency SLIs defined as threshold-based ratios — not raw averages (averages hide long tails)
- [ ] SLIs measured from the user's perspective — ideally at the API gateway or load balancer

---

## Step 3 — Define SLO Targets

**Starting point: measure before you commit.**
Instrument the SLI for 2–4 weeks before setting a target. Set the SLO at a level slightly better than current actual performance.

**Availability SLO table:**

| SLO | Monthly Error Budget | Downtime/Month | Appropriate For |
|---|---|---|---|
| 99.0% | 1.0% (7.3 hours) | 7.3 hours | Non-critical internal tools |
| 99.5% | 0.5% (3.65 hours) | 3.65 hours | Moderate impact internal services |
| 99.9% | 0.1% (43.8 min) | 43.8 minutes | Customer-facing services |
| 99.95% | 0.05% (21.9 min) | 21.9 minutes | SLO target behind a 99.9% SLA |
| 99.99% | 0.01% (4.4 min) | 4.4 minutes | Critical payment / auth services |
| 99.999% | 0.001% (26 sec) | 26 seconds | Extremely rare — very high cost |

**Latency SLO examples:**
```
99% of API requests complete in < 500ms (measured as a ratio over a 28-day rolling window)
95% of database queries complete in < 100ms
99% of background jobs complete within 30 seconds of being enqueued
```

**Setting the right target:**
- Too tight (99.99%) → error budget exhausted by deployments; engineers cannot ship
- Too loose (99.0%) → users experience poor service; SLO feels meaningless
- Target: set at a level where exhausting the error budget is a meaningful signal, not a routine occurrence

Checklist:
- [ ] SLO target set based on measured baseline — not guessed
- [ ] SLO reviewed with product and business — ensure it reflects what users actually need
- [ ] SLO stricter than the SLA by a meaningful margin (50–100 basis points)
- [ ] Latency SLO uses a threshold ratio — not average latency
- [ ] SLO window defined: rolling 28 days is standard (avoids month-end cliff effects)

---

## Step 4 — Calculate and Track Error Budget

**Error budget = the allowed amount of unreliability within the SLO window.**

```
Monthly error budget = (1 - SLO) × window_in_minutes
For 99.9% SLO: (1 - 0.999) × 43,200 min = 43.2 minutes of allowed downtime/month

Error budget remaining = total_budget - budget_consumed
Budget consumed = (bad_requests / total_requests) × window_duration
```

**Prometheus — track SLO compliance:**
```yaml
# Recording rules for SLI and error budget
groups:
  - name: orders_slo
    rules:
      # SLI: fraction of good requests (availability)
      - record: orders:sli:availability:rate28d
        expr: |
          1 - (
            sum(rate(orders_http_requests_total{status_code=~"5.."}[28d]))
            / sum(rate(orders_http_requests_total[28d]))
          )

      # Error budget remaining (fraction)
      - record: orders:error_budget:remaining:rate28d
        expr: |
          (orders:sli:availability:rate28d - 0.999) / (1 - 0.999)

      # Burn rate: how fast is budget being consumed right now?
      - record: orders:error_budget:burn_rate:1h
        expr: |
          (
            sum(rate(orders_http_requests_total{status_code=~"5.."}[1h]))
            / sum(rate(orders_http_requests_total[1h]))
          ) / (1 - 0.999)
```

**Grafana SLO dashboard panels:**
- Error budget remaining (% of month left as a gauge — green > 50%, yellow 10-50%, red < 10%)
- SLI trend over 28 days (time series vs. SLO line)
- Current burn rate (number — > 1 means burning faster than budget allows)
- Annotated deployment and incident markers

Checklist:
- [ ] Error budget calculated and displayed on the team's primary dashboard
- [ ] Budget remaining reviewed in weekly engineering sync — not only during incidents
- [ ] Budget burn rate available in real-time — not only as a post-facto calculation
- [ ] SLO compliance reported to product on a monthly cadence

---

## Step 5 — Error Budget Policy

An error budget policy defines what happens when the budget is consumed.

**Budget consumption thresholds and responses:**

| Budget Remaining | State | Engineering Response |
|---|---|---|
| > 50% | Healthy | Normal development velocity; feature work unrestricted |
| 25–50% | Cautious | Reduce risky deploys; prioritize reliability improvements |
| 10–25% | At risk | Feature freeze on risky changes; reliability work prioritized |
| < 10% | Critical — freeze | No new feature deployments; all effort on reliability |
| 0% | Exhausted | SLA breach risk — escalate to leadership; incident review |

**Policy document (stored in the repo):**
```markdown
## Orders Service Error Budget Policy

SLO: 99.9% of requests succeed (28-day rolling window)
Monthly error budget: 43.2 minutes

| Budget Remaining | Action Required |
|---|---|
| > 50% | Normal operations |
| 25–50% | Flag at weekly review; avoid risky deploys |
| < 25% | Tech lead approval required for all deploys |
| < 10% | Freeze on new features; reliability sprint activated |
| Exhausted | Escalate to VP Engineering; no feature deploys until budget restored |

Budget restores naturally as the 28-day window rolls forward from the incident period.
```

Checklist:
- [ ] Error budget policy written and shared with the team and product
- [ ] Policy thresholds trigger actual changes in behavior — not just dashboard colors
- [ ] Product stakeholders understand that feature velocity slows when budget is low — agreed upfront
- [ ] Budget restoration timeline communicated — engineers know when they can resume normal velocity

---

## Step 6 — SLO-Based Alerting (Burn Rate)

See also: the `alerting-strategy` skill for full Prometheus Alertmanager configuration.

**Multi-window burn rate alert recap:**

```yaml
# Alert fires when budget is burning 14.4× faster than normal (≈ exhausted in 2 hours)
- alert: OrdersSLOHighBurnRate
  expr: |
    orders:error_budget:burn_rate:5m > 14.4
    and orders:error_budget:burn_rate:1h > 14.4
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Orders SLO: burning error budget at 14.4× — 2% budget consumed in 1h"
    runbook: "https://runbooks.internal/orders/slo-breach"
```

**Burn rate to alert mapping:**
| Burn Rate | Time to Exhaustion | Alert | Severity |
|---|---|---|---|
| 14.4× | ~2 hours | Fast burn — 5m + 1h windows | Critical / P1 |
| 6× | ~5 days | Slow burn — 30m + 6h windows | Warning / P2 |
| 3× | ~10 days | Low and slow — 2h + 24h windows | Info / P3 |

Checklist:
- [ ] Multi-window burn rate alerts configured for every SLO
- [ ] Alerts link to the SLO dashboard and runbook
- [ ] Burn rate alerts reviewed for false positives after the first month of deployment
- [ ] Alert thresholds adjusted if budget is exhausted more than once per quarter — SLO may be too tight

---

## Step 7 — SLO Review and Governance

SLOs are not set-and-forget. They require regular review.

**Monthly SLO review checklist:**
- [ ] SLI measurements working — no gaps in data, no stale recording rules
- [ ] SLO met this month? If not: post-mortem action items on track?
- [ ] Error budget consumed this month — is the consumption expected (deploys) or unexpected (bugs)?
- [ ] Feature requests blocked by budget freeze? Review whether SLO target is appropriately set.
- [ ] Any SLO that was never in danger of breach? — SLO may be too loose; consider tightening.

**Quarterly SLO calibration:**
- Review all SLOs against actual user-reported issues — are the right things being measured?
- Check whether new user journeys need SLOs defined
- Validate SLA commitments match the SLO targets — no SLA tighter than the SLO

**SLO ownership:**
| Artifact | Owner |
|---|---|
| SLI instrumentation | Engineering (backend team) |
| SLO target setting | Engineering + Product |
| Error budget policy | Engineering + Product |
| SLA commitments | Business + Legal (informed by SLO) |
| SLO dashboard and alerts | SRE / Platform |
| Monthly SLO reporting | Team lead |

---

## Output Report

### Critical
- SLA commitments set tighter than the SLO target — SLA can breach even when engineering meets its SLO
- No SLI instrumentation — SLO is a number on a slide deck with no measurement backing it
- Error budget exhausted but no engineering response policy — budget means nothing; team ships freely

### High
- SLIs measure resource proxies (CPU, memory) instead of user-facing outcomes — wrong signals trigger responses
- No latency SLO — users experience slow service with no alerting or accountability
- SLO target set too tight (99.99%) for the current architecture — every deploy exhausts the budget; team paralyzed

### Medium
- Error budget not reviewed regularly — budget exhaustion discovered only at SLA breach
- Multi-window burn rate alerting not configured — slow-burn issues detected only after significant budget consumed
- SLOs defined per service, not per user journey — service health looks fine while a critical user path is broken

### Low
- SLO dashboard not shared with product — product makes feature velocity decisions without reliability context
- Error budget policy not written or agreed with product — freeze decisions become political, not objective
- SLI uses raw average latency — hides long tail; 1% of users experience 10× worse latency, invisible in the average

### Passed
- SLIs defined per critical user journey; measured as ratios from user-facing signals
- SLO targets set based on measured baseline; stricter than SLA by a meaningful margin
- Error budget calculated and displayed in real-time; burn rate alerts configured for fast and slow burn
- Error budget policy written, agreed with product, and enforced — feature freeze when budget is critical
- Monthly SLO review held; SLOs recalibrated quarterly; SLA commitments validated against SLO targets

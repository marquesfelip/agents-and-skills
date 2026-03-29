---
name: alerting-strategy
description: >
  Actionable alert configuration, routing, and escalation practices for production systems.
  Use when: alerting strategy, alert design, alert fatigue, actionable alerts, alert routing,
  PagerDuty alerting, OpsGenie, alertmanager, Prometheus alerting, Grafana alerts, alert rules,
  alert thresholds, on-call policy, alert severity, alert runbook, alert escalation,
  symptom-based alerting, cause-based alerting, SLO-based alerting, burn rate alert,
  error budget alert, alert noise, false positive alert, alert grouping, alert suppression,
  dead man's switch, watchdog alert, multi-window alerting, alert annotation, alert inhibition,
  alert silencing, alert deduplication, alert routing tree, on-call rotation.
argument-hint: >
  Describe the current alerting pain (e.g., "alert fatigue — 50+ alerts per night, most non-actionable",
  "no alerts at all", "alerts fire but don't link to runbooks"), the alerting stack
  (Prometheus Alertmanager, Grafana, PagerDuty, OpsGenie), and the on-call structure (teams, rotations).
---

# Alerting Strategy Specialist

## When to Use

Invoke this skill when you need to:
- Design or audit an alerting strategy for a service or platform
- Reduce alert fatigue by eliminating noisy, non-actionable alerts
- Define alert severity levels, routing rules, and escalation paths
- Implement SLO-based burn rate alerting
- Write alert runbooks that guide on-call engineers to resolution
- Set up dead man's switch (watchdog) alerts and heartbeat monitoring

---

## Step 1 — Alert Philosophy: Symptoms Over Causes

**Principle:** Alert on WHAT the user experiences (symptoms), not WHY it is happening (causes).

| Cause-based alert (bad) | Symptom-based alert (good) |
|---|---|
| CPU > 80% | p99 latency > 2s (users experiencing slowness) |
| Memory > 75% | Error rate > 1% (users receiving errors) |
| Database connections > 80% | Checkout success rate < 99% |
| Disk I/O wait high | Payment processing delay > 5s |

**Rules:**
- Every alert must be **actionable** — the on-call engineer can do something about it right now
- Every alert must have a **runbook link** — no alert without documented response steps
- Every alert must have a **severity** — not all alerts wake someone up at 3 AM
- Eliminate alerts that **always fire** (broken) or **never fire** (useless)
- An alert that fires and gets silenced more than 3 times in a month needs to be fixed or deleted

---

## Step 2 — Define Alert Severity Levels

| Severity | Urgency | Response Time | Notification Channel |
|---|---|---|---|
| **P1 — Critical** | Immediate | < 5 minutes | PagerDuty page (wakes on-call) |
| **P2 — High** | Urgent | < 30 minutes | Slack #incidents + PagerDuty |
| **P3 — Medium** | Needs attention today | < 4 hours | Slack #alerts |
| **P4 — Low / Info** | Informational | Next business day | Ticket / email digest |

**P1 examples:** Payment processing down, user login broken, data loss detected, SLO burn rate critical
**P2 examples:** Error rate elevated but not breaching SLO threshold, response time degraded, dependency latency spike
**P3 examples:** Memory growing (but not OOMing), background job queue depth elevated, non-critical external dependency slow
**P4 examples:** Certificate expiring in 30 days, disk usage at 60%, deprecated API being called

Checklist:
- [ ] Every alert has an explicit severity label in the alert rule
- [ ] Only P1 alerts page on-call — P2 and below route to Slack
- [ ] P1 alert count reviewed monthly — more than 5 P1 alerts per week indicates miscalibration
- [ ] P4 alerts reviewed and actioned within a sprint — not ignored indefinitely

---

## Step 3 — SLO-Based Burn Rate Alerting

Burn rate alerting is the most reliable method for SLO-driven on-call. It catches issues early while avoiding alert fatigue.

**Concept:**
- Your error budget is the allowed downtime/errors over a period (e.g., 1 month)
- Burn rate = how fast you are consuming the budget relative to normal
- Burn rate of 1 = consuming budget at exactly the rate that exhausts it in 30 days
- Burn rate of 14.4 = budget exhausted in 2 hours → page immediately

**Multi-window, multi-burn-rate alerts (Google SRE Book recommended approach):**

| Alert | Short Window | Long Window | Burn Rate | Severity | Budget Consumed |
|---|---|---|---|---|---|
| Fast burn | 5m | 1h | 14.4× | P1 | 2% in 1h |
| Slow burn | 30m | 6h | 6× | P2 | 5% in 6h |
| Low and slow | 2h | 24h | 3× | P3 | 10% in 24h |

**Prometheus rules:**
```yaml
groups:
  - name: orders_slo_alerts
    rules:
      # P1: fast burn — exhausts monthly budget in < 2 hours
      - alert: OrdersHighErrorBurnRate
        expr: |
          (
            rate(orders_http_requests_total{status_code=~"5.."}[5m])
            / rate(orders_http_requests_total[5m])
          ) > (14.4 * 0.01)   # 14.4× burn rate on 99% SLO (1% error budget)
          and
          (
            rate(orders_http_requests_total{status_code=~"5.."}[1h])
            / rate(orders_http_requests_total[1h])
          ) > (14.4 * 0.01)
        for: 2m
        labels:
          severity: critical
          team: orders
        annotations:
          summary: "Orders service burning error budget at 14.4× rate"
          runbook: "https://runbooks.internal/orders/high-error-rate"
          dashboard: "https://grafana.internal/d/orders-slo"

      # P2: slow burn — exhausts monthly budget in ~5 days
      - alert: OrdersSlowErrorBurnRate
        expr: |
          (
            rate(orders_http_requests_total{status_code=~"5.."}[30m])
            / rate(orders_http_requests_total[30m])
          ) > (6 * 0.01)
          and
          (
            rate(orders_http_requests_total{status_code=~"5.."}[6h])
            / rate(orders_http_requests_total[6h])
          ) > (6 * 0.01)
        for: 15m
        labels:
          severity: warning
          team: orders
        annotations:
          summary: "Orders service burning error budget at 6× rate"
          runbook: "https://runbooks.internal/orders/elevated-error-rate"
```

Checklist:
- [ ] Multi-window burn rate rules defined for each SLO — fast burn (P1) and slow burn (P2)
- [ ] The `for` clause set appropriately — avoids false positives from transient spikes
- [ ] `runbook` annotation URL present on every alert — on-call has immediate guidance
- [ ] `dashboard` annotation URL present — on-call can visualize context instantly

---

## Step 4 — Alert Rule Authoring Standards

Every alert rule must include:

```yaml
- alert: <PascalCase name that describes the symptom>
  expr: <PromQL expression>
  for: <minimum duration before firing — avoids flapping>
  labels:
    severity: critical | warning | info
    team: <owning team>
    service: <service name>
  annotations:
    summary: "<one-line description of the symptom>"
    description: "<detailed context — current value, threshold, impact>"
    runbook: "<URL to runbook>"
    dashboard: "<URL to relevant Grafana dashboard>"
```

**`for` duration guidelines:**
| Severity | Minimum `for` |
|---|---|
| Critical | 2–5 minutes |
| Warning | 10–15 minutes |
| Info | 30+ minutes |

**Alert naming convention:**
- Format: `<Service><Symptom><Condition>` in PascalCase
- Examples: `OrdersHighErrorRate`, `PaymentsLatencyBreached`, `InventoryQueueDepthHigh`
- Bad: `HTTPErrors`, `HighCPU`, `Alert1`

Checklist:
- [ ] All alerts follow naming convention and include all required annotations
- [ ] `for` duration is non-zero — no instant-firing alerts
- [ ] PromQL expression validated in Prometheus UI before merging alert rules
- [ ] Alert rules stored in version control — reviewed like code, not configured in the UI

---

## Step 5 — Alertmanager Routing and Grouping

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname", "team", "service"]
  group_wait: 30s        # wait before sending first notification in a group
  group_interval: 5m     # wait before re-notifying about new alerts in the same group
  repeat_interval: 4h    # re-notify if alert is still firing after this duration
  receiver: default-slack

  routes:
    # Critical alerts → PagerDuty (wakes on-call)
    - match:
        severity: critical
      receiver: pagerduty-critical
      group_wait: 0s     # page immediately for critical

    # Team-specific routing
    - match:
        team: payments
      receiver: payments-slack
      routes:
        - match:
            severity: critical
          receiver: payments-pagerduty

    # Suppress noisy alerts during known maintenance
    - match:
        alertname: DeploymentInProgress
      receiver: blackhole

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: "<PD_ROUTING_KEY>"
        severity: critical
        description: "{{ .CommonAnnotations.summary }}"
        links:
          - href: "{{ .CommonAnnotations.runbook }}"
            text: "Runbook"

  - name: default-slack
    slack_configs:
      - api_url: "<SLACK_WEBHOOK_URL>"
        channel: "#alerts"
        title: "[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}"
        text: "{{ .CommonAnnotations.description }}\nRunbook: {{ .CommonAnnotations.runbook }}"

inhibit_rules:
  # Suppress warning if critical is already firing for the same service
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["service", "team"]
```

Checklist:
- [ ] Routing tree configured — right alert reaches the right team via the right channel
- [ ] Grouping configured — related alerts grouped into one notification, not 50 individual pages
- [ ] Inhibition rules prevent warning-severity spam when critical is already firing
- [ ] `repeat_interval` set to 4h or more — prevents continuous re-paging for sustained incidents
- [ ] Secrets (PagerDuty keys, Slack webhooks) stored in Kubernetes Secrets — not in the config file

---

## Step 6 — Dead Man's Switch (Watchdog Alert)

A watchdog alert ensures the alerting pipeline itself is healthy.

```yaml
# Always-firing alert — if this stops being received, the pipeline is broken
- alert: Watchdog
  expr: vector(1)       # always evaluates to 1
  labels:
    severity: none
    type: watchdog
  annotations:
    summary: "Alerting pipeline heartbeat — always firing"
    description: "If this alert stops, Prometheus or Alertmanager is down."
```

```yaml
# In Alertmanager: route Watchdog to a dead man's switch service (e.g., healthchecks.io)
- match:
    alertname: Watchdog
  receiver: dead-mans-switch
  group_wait: 0s
  repeat_interval: 60s   # send heartbeat every 60s

receivers:
  - name: dead-mans-switch
    webhook_configs:
      - url: "https://hc-ping.com/<UUID>"  # healthchecks.io ping URL
```

Checklist:
- [ ] Watchdog alert defined and routing to a dead man's switch service
- [ ] Dead man's switch service (healthchecks.io, Cronitor, or equivalent) configured to alert if heartbeat stops
- [ ] Watchdog tested by intentionally stopping Alertmanager — verify dead man's switch fires
- [ ] Watchdog alert filtered from normal on-call routing — it should only notify if it STOPS arriving

---

## Step 7 — Runbook Standards

Every alert must have a runbook. A runbook is a written procedure, not a troubleshooting novel.

**Runbook template:**
```markdown
# Alert: OrdersHighErrorRate

## What is happening
The orders service error rate has exceeded 1% over the past 5 minutes.
Users may be seeing error responses when placing orders.

## Impact
- Orders failing: users cannot complete purchases
- Revenue impact: ~$X per minute at current order rate

## Immediate actions (do these first)
1. Open the [Orders SLO Dashboard](https://grafana.../d/orders-slo)
2. Check recent deployments: `kubectl rollout history deployment/orders-service`
3. Check logs for errors: Loki query → `{service="orders-service"} |= "error" | json`
4. Check downstream dependencies: [Inventory service status](...)

## Investigation
- If errors spiked after a deploy → rollback: `kubectl rollout undo deployment/orders-service`
- If database errors → check [DB dashboard](...)
- If inventory service errors → check [Inventory runbook](...)

## Escalation
- If unresolved after 15 minutes: escalate to `#orders-engineering` and ping the tech lead on call
- If data loss suspected: immediately page the Database team

## Resolution
- Declare incident resolved when error rate returns below 0.1% for 5 minutes
- Run a post-mortem if this incidents lasted > 30 minutes or caused > $1K revenue impact
```

Checklist:
- [ ] Runbook exists for every P1 and P2 alert
- [ ] Runbook includes impact statement — helps on-call assess urgency
- [ ] Runbook includes specific commands and dashboard links — not generic advice
- [ ] Runbooks stored in version control — reviewed and updated after every incident
- [ ] Runbook URL present in the alert annotation — accessible with one click from the alert

---

## Output Report

### Critical
- No runbooks for P1 alerts — on-call engineers improvise under pressure, making incidents worse
- No dead man's switch — alerting pipeline failure goes undetected; incidents silently missed
- P1 alerts routing to Slack instead of pager — engineers miss critical alerts overnight

### High
- Alert fatigue: > 10 P1 alerts per week from the same service — most are ignored
- Cause-based alerts (CPU, memory) fire constantly without guidance on whether user impact exists
- `for: 0s` on alerts — every transient spike pages on-call; trust in alerting system erodes

### Medium
- No inhibition rules — a single outage triggers 20 separate alert notifications
- Alert rules not in version control — configuration drift between environments, no review process
- Burn rate alerting absent — SLO breaches discovered by users, not by monitoring

### Low
- `repeat_interval` too short (< 1h) — sustained incidents generate continuous re-pages
- Alert names are generic (`HighError`, `Alert1`) — on-call cannot identify impact from the notification title alone
- `group_wait` not configured — every alert sends an immediate notification before grouping

### Passed
- All alerts are symptom-based with actionable runbook links; severity levels defined and routing configured
- Multi-window burn rate alerts defined for all SLOs; P1 fast-burn pages immediately, slow-burn notifies via Slack
- Alertmanager routing sends P1 to PagerDuty; grouping and inhibition reduce notification noise
- Dead man's switch configured; watchdog alert tests the full pipeline end-to-end
- Alert rules stored in version control; reviewed and tested before merging; `for` durations prevent flapping

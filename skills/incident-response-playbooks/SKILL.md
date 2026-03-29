---
name: incident-response-playbooks
description: >
  Incident handling workflows, escalation procedures, and post-incident recovery practices.
  Use when: incident response, incident management, incident playbook, incident handling,
  on-call response, incident commander, incident severity, SEV1 SEV2 SEV3, P0 P1 P2 incident,
  incident runbook, escalation procedure, war room, incident communication, status page update,
  incident timeline, incident mitigation, rollback decision, incident post-mortem, blameless postmortem,
  root cause analysis, RCA, five whys, incident retrospective, MTTR, MTTD, MTTI, incident metrics,
  PagerDuty incident, OpsGenie incident, incident Slack, incident on-call rotation, incident drill,
  chaos drill, game day, incident on-call, incident recovery, service degradation, outage management.
argument-hint: >
  Describe the incident being handled or the playbook being designed: service affected, severity,
  team structure (solo on-call vs. incident commander + responders), and current gaps
  (e.g., "no defined severity levels", "no post-mortem culture", "slow MTTR").
---

# Incident Response Playbook Specialist

## When to Use

Invoke this skill when you need to:
- Design or audit an incident response process from detection to resolution
- Define incident severity levels and escalation paths
- Write or improve an incident playbook for a specific service or failure mode
- Establish a blameless post-mortem culture and process
- Reduce MTTR (Mean Time to Resolve) and MTTD (Mean Time to Detect)
- Run an incident drill (GameDay) to rehearse response procedures

---

## Step 1 — Define Incident Severity Levels

Consistent severity definitions prevent over-escalating minor issues and under-escalating critical ones.

| Severity | Definition | Example | Response SLA |
|---|---|---|---|
| **SEV1 / P0** | Complete outage of a critical service; significant data loss; security breach | Payment processing down, user data exposed | Page immediately; all hands response |
| **SEV2 / P1** | Major feature degraded; SLO breach in progress; significant user impact | Checkout failing for 20% of users, p99 > 10s | Page on-call; incident commander assigned |
| **SEV3 / P2** | Minor feature degraded; SLO burn rate elevated; no immediate data risk | Search slow, export feature unavailable | Notify via Slack; response within 30 min |
| **SEV4 / P3** | Performance degradation below alert threshold; non-customer-facing | Internal dashboard slow, batch job delayed | Ticket created; next business day |

**Severity criteria (used to classify during triage):**
- Is a customer-facing feature completely unavailable? → SEV1
- Are > 5% of users impacted? → SEV2
- Is revenue directly affected? → SEV1 or SEV2
- Is the SLO error budget burning at > 10× rate? → SEV2

Checklist:
- [ ] Severity levels published and accessible to all engineers — not just SRE
- [ ] Severity classification criteria are objective, not subjective — anyone can apply them
- [ ] Downgrade path defined — when does a SEV1 drop to SEV2? (when primary symptom resolves)
- [ ] Severity influences escalation and communication — not just priority label

---

## Step 2 — Incident Response Roles

**Keep it simple — three roles cover most incidents:**

| Role | Responsibility | Who |
|---|---|---|
| **Incident Commander (IC)** | Drives the process, delegates tasks, makes go/no-go decisions | On-call lead or senior engineer |
| **Technical Lead (TL)** | Investigates, diagnoses, implements fixes | Domain expert or on-call engineer |
| **Communications Lead (CL)** | Updates status page, notifies stakeholders, manages the Slack war room | Can be the IC for smaller incidents |

**For SEV1/P0 — additional roles:**
- **Executive Sponsor**: Informed but not in the war room — makes resource decisions only
- **Customer Success**: Handles customer-facing communication during extended outages

**Role handoff (for long incidents):**
- Formally hand off the IC role with a written handoff summary — who did what, what's in progress, what's unknown
- Handoff every 4 hours maximum — fatigued incident commanders make poor decisions

Checklist:
- [ ] IC assigned within 2 minutes of SEV1 declaration
- [ ] IC role is clear — no single person trying to investigate AND communicate simultaneously
- [ ] On-call rotation documented — who is IC this week vs. next week
- [ ] Escalation contact list current — who to call when the first on-call is unreachable

---

## Step 3 — Incident Lifecycle

```
Detection → Triage → Declaration → Investigation → Mitigation → Resolution → Post-mortem
```

**Phase 1: Detection** (target: < 5 minutes for SEV1)
- Alert fires in PagerDuty / OpsGenie → on-call paged
- IC acknowledges the page within 5 minutes (SLA)
- If no alert: report from customer, Slack message, monitoring dashboard → SEV process triggered manually

**Phase 2: Triage** (target: < 10 minutes)
- IC assesses scope: which services affected? which users? how many?
- IC assigns severity using the criteria from Step 1
- IC opens an incident Slack channel: `#inc-<date>-<short-description>` (e.g., `#inc-20260329-orders-down`)
- IC posts initial summary in the channel: what we know, what we don't, who is working it

**Phase 3: Declaration**
- IC declares the incident formally: `@here Declaring SEV2 incident — orders checkout degraded`
- IC starts an incident timeline document (Google Doc or incident management tool)
- Status page updated: "Investigating — we are aware of an issue affecting order placement"

**Phase 4: Investigation**
- TL investigates while IC manages comms and coordination
- Key questions: When did it start? What changed? (deploys, config, traffic spike?) What's the error?
- Timeline entries logged for every finding: `14:32 — Error rate spike to 12% began`

**Phase 5: Mitigation** (stop the bleeding — not the same as fixing the root cause)
- Rollback the latest deploy if correlated
- Toggle a feature flag off
- Scale up the affected service
- Redirect traffic to a backup region
- Enable rate limiting to protect a struggling dependency

**Phase 6: Resolution**
- Customer impact confirmed resolved — error rate back below threshold, SLO consuming normally
- Status page updated: "Resolved — service restored as of 15:47 UTC"
- Monitoring window: observe for 30 minutes minimum before closing the incident

Checklist:
- [ ] Incident Slack channel opened within 5 minutes of SEV1 declaration
- [ ] Timeline document created and updated throughout the incident
- [ ] Status page updated at declaration, at mitigation, and at resolution
- [ ] Mitigation and root cause are treated separately — restore first, fix root cause after
- [ ] IC explicitly declares the incident resolved — not quietly abandoned

---

## Step 4 — Communication Templates

**Incident Slack opening message:**
```
🔴 SEV2 INCIDENT DECLARED — 2026-03-29 14:28 UTC

Summary: Orders checkout is failing for ~15% of users. Payment step returning 500 errors.
Severity: SEV2
IC: @alice
TL: @bob
Status page: https://status.myapp.com

Timeline doc: <link>

Current hypothesis: Correlated with deploy v1.4.2 at 14:20 UTC
Next update: in 15 minutes or when we have new findings.
```

**Status page updates:**

| Phase | Message |
|---|---|
| Investigating | "We are investigating reports of elevated error rates on order placement. Engineers are actively engaged." |
| Identified | "We have identified the cause of the issue: a database connection pool exhaustion. Mitigation is in progress." |
| Monitoring | "A fix has been applied and we are monitoring to confirm stability. Some users may still experience issues." |
| Resolved | "The incident is resolved. Orders are processing normally as of 15:47 UTC. A post-incident review will follow." |

**Stakeholder update (email/Slack to leadership for extended SEV1):**
```
Subject: [INCIDENT UPDATE] Order Processing Degraded — 14:28 UTC

Duration: 47 minutes (still ongoing)
Customer impact: ~15% of checkout attempts failing
Revenue impact: Estimated $X,XXX loss per hour
Current status: Root cause identified (DB connection pool), fix deploying now
ETA to resolution: ~15 minutes
Next update: in 30 minutes or at resolution
```

Checklist:
- [ ] First Slack update sent within 5 minutes of declaring the incident
- [ ] Status page updated promptly — customers self-serve rather than flooding support
- [ ] Stakeholder updates sent every 30 minutes for SEV1 — silence breeds panic
- [ ] All communications factual, calm, and customer-focused — no finger-pointing or speculation

---

## Step 5 — Mitigation Decision Framework

```
Is the issue correlated with a recent deploy?
  Yes → Rollback immediately; restore first, investigate later
  No  → Continue investigation

Can a feature flag isolate the affected feature?
  Yes → Toggle it off immediately

Is the issue caused by traffic volume?
  Yes → Enable rate limiting / throttling; scale up if feasible

Is a downstream dependency failing?
  Yes → Apply circuit breaker; fallback to degraded mode if available

Is data at risk of corruption or loss?
  Yes → Stop writes immediately; escalate to data team; preserve state first
```

**Rollback triggers (act immediately, no committee needed):**
- Deploy within the last 2 hours + correlated error spike → rollback
- Database migration ran + new errors → stop migration + assess rollback
- Config change deployed + service crash → revert config

**Rollback is not failure. It is the right engineering decision.**

Checklist:
- [ ] On-call engineer has permission to rollback without approval — no bureaucratic blocker during active outage
- [ ] Rollback procedure is documented and rehearsed — not improvised during the incident
- [ ] Circuit breaker or feature flag available for critical paths — fast mitigation tool
- [ ] Data integrity checked before resuming full operation after a mitigation

---

## Step 6 — Post-Incident Review (Blameless Post-Mortem)

**Blameless means:** the system failed, not the person. The goal is to find systemic gaps, not to assign blame.

**When to hold a post-mortem:**
- Every SEV1 (mandatory)
- SEV2 lasting > 30 minutes (recommended)
- SEV3 with customer complaints (optional but valuable)

**Timeline for post-mortem:**
- Draft within 24 hours of incident resolution (while memory is fresh)
- Review meeting within 48–72 hours
- Action items tracked in Jira/Linear with owners and due dates

**Post-mortem document template:**
```markdown
# Post-Mortem: Orders Checkout Degraded
**Date:** 2026-03-29
**Duration:** 47 minutes (14:28 – 15:15 UTC)
**Severity:** SEV2
**Authors:** Alice, Bob

## Impact
- 15% of checkout attempts failed
- Estimated 2,300 affected users
- Estimated revenue impact: $12,000

## Timeline
| Time | Event |
|------|-------|
| 14:20 | Deploy v1.4.2 released |
| 14:28 | Alert fired: error rate 12% |
| 14:30 | IC Alice declared SEV2 |
| 14:35 | Identified correlation with deploy |
| 14:52 | Rollback initiated |
| 15:10 | Error rate normalized |
| 15:15 | Incident resolved |

## Root Cause
Deploy v1.4.2 introduced a connection pool size reduction (from 50 to 10) as an optimization.
Under production load, the pool exhausted within 8 minutes, causing DB timeout errors on the checkout path.

## Contributing Factors
1. No staging load test was run for this configuration change
2. Connection pool size was not included in the change review checklist
3. Alert threshold (1% errors) took 8 minutes to trigger — mitigation delayed

## Five Whys
1. Why did checkout fail? → DB connections timed out
2. Why did connections time out? → Pool exhausted
3. Why was it exhausted? → Pool reduced to 10 connections
4. Why was it reduced? → Performance optimization merged without load testing
5. Why was there no load test? → Load testing not required for config changes (gap in process)

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| Require load tests for connection pool config changes | eng-process | 2026-04-05 |
| Add DB connection pool saturation alert (< 20% free) | SRE | 2026-04-03 |
| Reduce alert detection threshold to 0.5% error rate | SRE | 2026-04-01 |
| Document rollback steps in the orders runbook | Bob | 2026-04-02 |

## What Went Well
- IC role was clear; no confusion about who was driving
- Rollback decision made quickly once correlation was found
- Status page updated throughout

## What Could Be Improved
- 8-minute detection lag was too slow for a SEV2
- Communication to leadership was delayed — no template for stakeholder updates
```

Checklist:
- [ ] Post-mortem document drafted within 24 hours — not weeks later
- [ ] Timeline reconstructed from logs, alerts, and Slack history — accurate, not reconstructed from memory
- [ ] Root cause identified using Five Whys or equivalent — not "human error" as the final answer
- [ ] Action items are specific, assigned, and time-bound — `"improve monitoring"` is not an action item
- [ ] At least one `What Went Well` section — reinforce what to preserve, not just what to fix
- [ ] Blameless framing maintained — the document names the system failure, not the person who deployed

---

## Step 7 — Incident Drills (GameDay)

Drills validate that the process and tooling actually work before a real incident.

**GameDay types:**
| Type | Method | Goal |
|---|---|---|
| **Tabletop** | Discuss a scenario verbally | Validate process, find gaps in runbooks |
| **Chaos injection** | Kill a pod, block a port, throttle CPU | Validate detection, alert, and rotation |
| **Communication drill** | Simulate a SEV1 Slack channel | Validate communication templates and escalation |

**Quarterly drill schedule:**
- Q1: Tabletop — new team members walk through a SEV1 scenario
- Q2: Chaos injection — kill primary DB replica, validate failover and alert
- Q3: Communication drill — simulated outage with full status page update exercise
- Q4: Full end-to-end — detection → declaration → mitigation → post-mortem for a synthetic incident

Checklist:
- [ ] At least one incident drill per quarter
- [ ] Drill debrief held within 24 hours — gaps found and tracked as action items
- [ ] New on-call engineers participate in a drill before their first solo rotation
- [ ] Runbooks updated based on drill findings — drills without follow-up are waste

---

## Output Report

### Critical
- No defined severity levels — every incident treated as P1; team always in crisis mode
- On-call engineers cannot rollback without a manager's approval — every minute of delay costs users
- No status page — users flood support during incidents; support team has no information to share

### High
- No post-mortem process — the same failures recur every few months
- IC and TL roles not separated — one engineer investigating while also writing all communications burns out
- No incident timeline document — post-mortem reconstruction is guesswork

### Medium
- Post-mortem action items not tracked — identified gaps are forgotten within a week
- Communication templates missing — ad-hoc messages sent under pressure cause confusion
- No dead man's switch or alert pipeline test — alerting fails silently, incidents detected by customer reports

### Low
- No incident drill program — team discovers process gaps during real incidents, not controlled exercises
- Runbooks not updated after incidents — same investigation steps re-discovered each time
- Status page updates infrequent — users assume the worst during silence

### Passed
- Severity levels defined with objective criteria; escalation paths configured for each level
- IC, TL, and CL roles clearly separated; on-call rotation current and accessible
- Lifecycle phases defined: detection SLA < 5 min for SEV1; mitigation before root cause; status page updated at each phase
- Blameless post-mortem held within 48h for every SEV1; action items assigned with owners and due dates
- Incident drills run quarterly; runbooks updated based on drill findings and real incident learnings

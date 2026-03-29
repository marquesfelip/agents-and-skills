---
name: abuse-detection
description: 'Anomaly detection, suspicious usage monitoring, and automated abuse response. Use when: abuse detection, anomaly detection, suspicious usage, fraud detection, behavioral monitoring, account abuse, API abuse monitoring, usage anomaly, suspicious login, impossible velocity, geo anomaly, abuse response, account flagging, automated abuse, usage spike detection.'
argument-hint: 'Service, user activity, or monitoring system to review for abuse detection capabilities — or describe the abuse prevention strategy being designed'
---

# Abuse Detection Specialist

## When to Use
- Designing a monitoring strategy to detect and respond to platform abuse
- Reviewing logging and alerting pipelines for coverage of suspicious behavior
- Building fraud detection or account abuse signals for an authentication or transactional system
- Implementing automated response workflows (flag, challenge, suspend, block)
- Auditing gaps in visibility between security events and operational response

---

## Step 1 — Define the Abuse Threat Model

Before modeling detection, identify what abuse looks like for the platform:

| Abuse category | Examples |
|---|---|
| Account takeover (ATO) | Credential stuffing, session hijacking, unauthorized login |
| Fraudulent account creation | Mass fake registrations, disposable email accounts |
| Data exfiltration | Bulk API access, excessive downloads, scraping |
| Financial fraud | Card testing, promo code abuse, chargeback fraud |
| Platform policy violations | Spam, hate content, impersonation |
| Resource abuse | Compute/storage/API quota violations |
| Insider threats | Privileged users accessing data outside their scope |

---

## Step 2 — Detection Signal Taxonomy

### Authentication & Session Signals

| Signal | Threshold / Rule | Abuse indicated |
|---|---|---|
| Login failure rate | > 5 failures / 15 min per account | Credential stuffing / brute force |
| Login from new country | First time in 30+ days | Account takeover attempt |
| Impossible velocity | Same account, 2 geolocations > 500km apart, < 1h | Session sharing or ATO |
| Multiple accounts, same IP | > 3 new accounts from 1 IP in 24h | Fake account creation |
| Password reset frequency | > 2 resets / week | Account access abuse |
| MFA challenge failure rate | > 3 failures / session | ATO attempt on MFA-protected account |

### API & Data Access Signals

| Signal | Threshold | Abuse indicated |
|---|---|---|
| Requests per minute | > 10× account baseline | Scraping or automation |
| Unique endpoint calls | > 80% of API surface in 1h | Automated enumeration |
| Data volume exported | > 3× account baseline | Bulk data exfiltration |
| Sequential ID traversal | Incrementing resource IDs in pattern | BOLA/IDOR enumeration |
| Error rate | > 20% 4xx in session | Probing / fuzzing |
| Time-of-day anomaly | Activity at unusual hours vs account history | Compromised account |

### Financial / Transaction Signals

| Signal | Threshold | Abuse indicated |
|---|---|---|
| Card decline rate | > 2 declines per hour per account | Card testing |
| Multiple cards on one account | > 3 new cards in 24h | Fraud ring |
| Chargeback rate | > 0.5% of transactions | Friendly fraud |
| Promo code usage rate | > 10 unique codes per account | Promo abuse |
| High-value transaction, new account | First transaction > 2× average | Fraud |

---

## Step 3 — Risk Scoring Model

Aggregate signals into a risk score per entity (user, IP, device, session):

```
risk_score = 0

# Authentication signals
if login_failures_last_15min > 5:       risk_score += 40
if new_country_login:                    risk_score += 20
if impossible_velocity:                  risk_score += 60

# Behavioral signals
if request_rate > 10x_baseline:          risk_score += 30
if sequential_id_access_detected:        risk_score += 50
if new_device_fingerprint:               risk_score += 10

# Context signals
if datacenter_ip:                        risk_score += 20
if disposable_email_domain:              risk_score += 15
if account_age_hours < 24:              risk_score += 10

# Thresholds → actions
0–30:   allow (normal)
31–60:  log + monitor closely
61–80:  require step-up auth (MFA re-prompt)
81–100: block and require manual review
```

---

## Step 4 — Logging Requirements for Abuse Detection

Detection is only possible with complete, structured logs:

| Event | Required fields |
|---|---|
| Login attempt | timestamp, user_id, ip, user_agent, success/failure, country, device_fingerprint |
| MFA challenge | timestamp, user_id, method, result |
| Resource access | timestamp, user_id, resource_id, action (read/write/delete) |
| API request | timestamp, api_key/user_id, endpoint, method, response_code, duration |
| Account change | timestamp, user_id, field_changed, old_value_hash, ip |
| Payment attempt | timestamp, user_id, amount, currency, status, card_fingerprint |
| Export / download | timestamp, user_id, data_type, record_count, bytes |
| Admin action | timestamp, admin_id, target_user_id, action, reason |

**All logs must be:**
- Structured (JSON) for machine-parseable querying
- Tamper-evident (append-only log store, WORM storage for critical logs)
- Centralized (SIEM, Splunk, Datadog, Elastic)
- Retained for ≥90 days for investigation; 1+ year for compliance

---

## Step 5 — Automated Response Tiers

Match response to risk level — avoid over-blocking legitimate users:

| Risk tier | Automated response | Example signals |
|---|---|---|
| Low | Log only; increase monitoring sensitivity | New country login |
| Medium | Send user notification; require MFA re-prompt | New device + unusual hour |
| High | Temporary account lock; force password reset | Impossible velocity + high failure rate |
| Critical | Immediate account suspend + alert security team | Bulk data exfiltration + card testing pattern |

**User communication on abuse detection:**
```
Low:    "We noticed a login from a new location. Was this you?"
Medium: "Please verify your identity to continue."
High:   "Your account has been locked for security. Please reset your password."
```

**Never auto-permanently-ban** without human review — false positives permanently harm legitimate users.

---

## Step 6 — Detection Architecture

### Real-Time Detection
- Apply rules inline in the request path (low latency, simple rules only).
- Use a rules engine (Redis Streams + rule evaluation, Drools, custom middleware).
- Best for: rate limiting, IP blocking, simple threshold rules.

### Near-Real-Time Detection (stream processing)
- Process event streams asynchronously (Kafka, Kinesis, Pub/Sub → Spark/Flink).
- Evaluate complex patterns over time windows.
- Best for: behavioral anomalies, multi-event correlations.

### Batch Detection (ML-based anomaly detection)
- Run periodic batch jobs on historical data; generate risk scores fed back to real-time layer.
- Use unsupervised anomaly detection (Isolation Forest, DBSCAN) for novelty detection.
- Best for: fraud rings, slow-burn attacks, behavioral baselines.

---

## Step 7 — Insider Threat Detection

Privileged users (admins, support agents, engineers) require separate monitoring:

| Signal | Detection |
|---|---|
| Bulk data access outside normal role scope | Alert when an admin views > N customer records in 1 day |
| Off-hours privileged access | Alert on admin actions at unusual hours without on-call justification |
| Exfiltration indicators | Large exports from admin tools; unusual download patterns |
| Privilege escalation attempts | Attempts to access endpoints above assigned role |
| Break-glass access | Alert immediately on emergency access; require mandatory justification |

---

## Step 8 — Output Report

```
## Abuse Detection Review: <platform/service>

### Critical
- No logging of authentication events (login success/failure)
  Impossible to detect credential stuffing; no forensic baseline exists
  Fix: log all auth events with timestamp, IP, user_id, success/failure, user_agent

- No automated response to any abuse signal
  Detected anomalies are not acted upon; attackers face no resistance after detection
  Fix: implement automated lockout on > 10 consecutive login failures; notify user

### High
- No impossible velocity check across login geolocations
  Account takeover via credential stuffing leaves no velocity anomaly alert
  Fix: flag accounts logging in from locations > 500km apart within 1 hour; trigger MFA

- Logs stored only 7 days — insufficient for incident investigation
  Fix: retain auth and API access logs for ≥ 90 days; 1 year for financial events

### Medium
- No risk score model for new account creation
  Fraud rings creating hundreds of accounts daily go undetected
  Fix: score new accounts on: email domain, IP reputation, registration velocity, device fingerprint

- Admin actions not logged separately or monitored
  Insider threat activity not visible until post-incident
  Fix: log all admin actions to append-only audit log; alert on bulk access

### Low
- User notifications not sent on new-device login
  Legitimate users not alerted to potential ATO
  Fix: send email notification on first login from new device fingerprint

### Passed
- Centralized SIEM with structured JSON logs ingested ✓
- Alert on > 5 login failures / 15 min per account ✓
- Sequential resource ID access monitored ✓
```

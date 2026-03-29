---
name: data-retention-policies
description: 'Data lifecycle rules, retention enforcement, deletion policies, and compliance safety. Use when: data retention policy, data lifecycle, retention enforcement, data deletion policy, GDPR retention, LGPD retention, right to erasure, data purge, automated deletion, data expiry, record retention, data minimization lifecycle, retention compliance, scheduled deletion, storage cost, data archival, TTL policy.'
argument-hint: 'Data entity or category requiring a retention policy — include applicable regulations (GDPR, LGPD, PCI, HIPAA, or local tax/financial rules) and business context'
---

# Data Retention Policy Specialist

## When to Use
- Designing retention rules for a new data entity or feature
- Auditing existing systems for data that is held longer than necessary
- Implementing automated deletion or archival pipelines
- Responding to GDPR/LGPD right-to-erasure requests that intersect legal hold obligations
- Building compliance evidence that data is not retained beyond its lawful period

---

## Step 1 — Inventory Data Entities and Their Sensitivity

Map every significant data entity to its category and regulatory context:

| Entity | Category | Regulatory driver | Business driver |
|---|---|---|---|
| User account (PII) | Personal data | GDPR, LGPD | Active user lifecycle |
| Authentication logs | Security logs | SOC 2, ISO 27001 | Fraud/incident investigation |
| Order / transaction records | Financial | Tax law (typically 5–7 years) | Dispute resolution |
| Payment card data | PCI-DSS data | PCI-DSS Req. 3 | Must not be retained beyond transaction |
| Health records | PHI / sensitive | HIPAA, LGPD Art. 11 | Clinical need |
| Session tokens and cookies | Credential | Privacy law | Security hygiene |
| Application error logs | Operational | None formal | Debug/incident response |
| Audit events | Compliance | SOC 2, ISO 27001, LGPD | Compliance, forensics |
| Marketing consents | Consent records | GDPR Art. 7, LGPD Art. 8 | Proof of lawful processing |
| Backup snapshots | All categories | Inherits from contained data | DR / recovery |

---

## Step 2 — Define Retention Rules by Regulatory and Business Driver

### Regulatory Minimums (Reference Table)

| Framework | Data type | Minimum retention | Maximum (if specified) |
|---|---|---|---|
| GDPR / LGPD | Personal data / PII | Only as long as purpose requires | Purpose limitation applies |
| Brazilian Tax (Lei 9.430/96) | Financial / fiscal records | 5 years | — |
| Brazilian CLT | Employment records | 5 years post-termination | — |
| PCI-DSS | Cardholder data (CHD) | Defined by business need | Do not retain PANs beyond necessity |
| PCI-DSS | Transaction audit logs | 1 year (most recent 3 months accessible) | — |
| HIPAA | Protected health information | 6 years from creation or last effective date | — |
| SOC 2 | Security audit logs | 1 year | — |
| ISO 27001 | Security event logs | 1 year | — |
| US IRS | Tax-related records | 3–7 years | — |

### Retention Policy Template

Define per entity:

```yaml
entity: user_account
category: personal_data
regulation: LGPD, GDPR
retention:
  active:          # while user has active account
    period: indefinite
    condition: account_status == 'active'
  post_closure:    # after account deletion request
    period: 30 days
    purpose: dispute_resolution, outstanding_balance_check
  legal_hold:      # overrides deletion if legal dispute exists
    trigger: legal_notice_received
    period: until_resolved
deletion_method: anonymize_then_purge
archival: none
```

---

## Step 3 — Design the Retention Lifecycle State Machine

Every data entity should have a lifecycle with defined transitions:

```
[Created] → [Active] → [Closed/Expired] → [Grace Period] → [Archived] → [Purged]
                                                ↕
                                         [Legal Hold]
```

| State | Description | Transition trigger |
|---|---|---|
| Active | Data in active use | Entity is in use by business process |
| Closed/Expired | Primary purpose ended | Account deleted, subscription cancelled, session expired |
| Grace period | Short window for dispute resolution or reversal | Configurable per entity (7–90 days typical) |
| Archived | Moved to cold storage; restricted access | Grace period elapsed; regulatory period requires preservation |
| Purged | Irreversibly deleted or anonymized | Retention period fully elapsed; no active legal hold |
| Legal hold | Deletion suspended by regulation or litigation | Manual flag; cleared by legal team |

---

## Step 4 — Implement Automated Retention Enforcement

### Soft Delete + Scheduled Hard Delete Pattern

```python
# Schema — includes lifecycle tracking columns
class UserAccount(Base):
    id               = Column(String, primary_key=True)
    status           = Column(String)               # active | closed | archived
    closed_at        = Column(DateTime, nullable=True)
    scheduled_purge_at = Column(DateTime, nullable=True)
    legal_hold       = Column(Boolean, default=False)
    purged_at        = Column(DateTime, nullable=True)

# On account closure: mark closed and schedule deletion
def close_account(user_id: str):
    user = db.get(user_id)
    user.status = 'closed'
    user.closed_at = datetime.utcnow()
    user.scheduled_purge_at = datetime.utcnow() + timedelta(days=30)
    db.save(user)
    audit_log.record('account.closed', user_id)

# Scheduled daily job: purge accounts past retention window
@scheduler.task('0 2 * * *')  # daily at 02:00 UTC
def purge_expired_accounts():
    candidates = db.query(UserAccount).filter(
        UserAccount.status == 'closed',
        UserAccount.scheduled_purge_at <= datetime.utcnow(),
        UserAccount.legal_hold == False,
        UserAccount.purged_at.is_(None),
    ).all()

    for user in candidates:
        anonymize_user(user)       # replace PII fields with anonymized values
        cascade_purge(user.id)     # delete related records per policy
        user.purged_at = datetime.utcnow()
        audit_log.record('account.purged', user.id)

def anonymize_user(user: UserAccount):
    """Replace PII with anonymized values — preserves referential integrity."""
    user.email      = f"deleted_{user.id}@anon.invalid"
    user.full_name  = "Deleted User"
    user.phone      = None
    user.address    = None
    # Preserve: id (for FK integrity), closed_at, created_at (for analytics aggregates)
```

### TTL-Based Expiry (Redis, DynamoDB, MongoDB)

```python
# Redis — session token expires automatically
redis.setex(
    name=f"session:{session_id}",
    time=timedelta(hours=24),   # TTL enforces retention without scheduled job
    value=session_data,
)

# DynamoDB — TTL attribute
table.put_item(Item={
    'pk': f"session#{session_id}",
    'data': session_data,
    'ttl': int((datetime.utcnow() + timedelta(days=30)).timestamp()),
})
```

### SQL: Retention Enforcement Pattern

```sql
-- Automated purge: delete records past retention window (run as scheduled job)
DELETE FROM application_logs
WHERE created_at < now() - INTERVAL '90 days'
  AND log_level IN ('DEBUG', 'INFO');  -- preserve ERROR logs longer for forensics

-- Anonymize PII in orders older than 5 years (tax retention: keep record, remove PII)
UPDATE orders
SET
    customer_name    = 'ANONYMIZED',
    customer_email   = NULL,
    billing_address  = NULL
WHERE
    created_at < now() - INTERVAL '5 years'
    AND customer_name != 'ANONYMIZED';
```

---

## Step 5 — Handle Legal Holds

Legal holds must override all automated deletion:

```python
def apply_legal_hold(entity_type: str, entity_id: str, reason: str, applied_by: str):
    hold = LegalHold(
        entity_type=entity_type,
        entity_id=entity_id,
        reason=reason,
        applied_by=applied_by,
        applied_at=datetime.utcnow(),
    )
    db.save(hold)
    audit_log.record('legal_hold.applied', entity_id, {'reason': reason})

# Deletion job always checks
def is_eligible_for_deletion(entity_id: str) -> bool:
    holds = db.query(LegalHold).filter(
        LegalHold.entity_id == entity_id,
        LegalHold.released_at.is_(None),   # active hold
    ).count()
    return holds == 0
```

---

## Step 6 — Backup and Replica Retention

Data deleted from the primary database may persist in backups longer than the retention policy allows:

| Concern | Mitigation |
|---|---|
| Deleted PII lingers in backups | Document that backups retain data up to backup age; disclose in privacy policy |
| GDPR erasure request | For regulatory erasure: note in erasure record that backup copies will be overwritten as backups age per retention schedule |
| Long-lived backups (3+ years) | Encrypt backups; maintain separate audit of which backup snapshots may contain specific subjects' data |
| Read replicas | Replication lag; deletion applied within minutes; acceptable for most policies |
| Long-running analytics imports | ETL pipelines may hold copies; include analytics tables in purge scope |

**Best practice:** Define a "data subject erasure acknowledgement" that clearly states which stores may hold data for up to N days after erasure (equal to backup age) and that this is within regulatory tolerance.

---

## Step 7 — Audit and Verification

Retention policies are only effective if they actually run and are verifiable:

```python
# Verification job: detect data that should have been purged but wasn't
@scheduler.task('0 6 * * 1')  # weekly
def verify_retention_compliance():
    # Find closed accounts past purge date with PII still present
    violations = db.query(UserAccount).filter(
        UserAccount.status == 'closed',
        UserAccount.scheduled_purge_at < datetime.utcnow() - timedelta(days=7),
        UserAccount.purged_at.is_(None),
        UserAccount.legal_hold == False,
        UserAccount.email.notlike('%@anon.invalid'),  # still has real email
    ).count()

    if violations > 0:
        alert.send_critical(f"Retention violation: {violations} accounts past purge date with PII")
        audit_log.record('retention.violation_detected', count=violations)
```

### Compliance Evidence Checklist

- [ ] Retention policy document version-controlled and approved by legal/DPO
- [ ] Automated purge job runs on schedule — monitored and alerted on failure
- [ ] Purge events recorded in audit log (who, what, when, why)
- [ ] Legal hold mechanism tested — deletion does not occur while hold is active
- [ ] Annual retention policy review scheduled
- [ ] Privacy policy accurately describes retention periods for each data category
- [ ] Backup retention periods documented and aligned with data retention policy

---

## Output Report

```
## Data Retention Policy Review: <domain>

### Critical
- Deleted user accounts retain full PII (email, name, phone, address) indefinitely
  No purge job runs; LGPD Art. 16 violation — data retained beyond lawful purpose
  Fix: implement scheduled purge job with 30-day grace period; anonymize PII on expiry

- Payment card PANs stored in orders table with no expiry
  PCI-DSS Req. 3.1: do not retain sensitive authentication data post-authorization
  Fix: delete PANs immediately post-auth; store only last 4 + scheme; use tokenization

### High
- No legal hold mechanism — legal team cannot prevent data deletion during litigation
  Risk: data subject to deletion could be needed as evidence; spoliation liability
  Fix: implement LegalHold table; purge job checks for active holds before deletion

- Application logs retained indefinitely (DB table with no TTL or scheduled delete)
  Debug logs accumulate PII from request contexts; no business justification for retention
  Fix: add created_at index; scheduled job deletes logs older than 90 days

### Medium
- Backup snapshots retained 3 years; no documentation of PII scope per snapshot
  Cannot fulfill GDPR data subject access or erasure requests for backup-specific data
  Fix: document retention overlap in privacy policy; add metadata tagging per snapshot

- Session tokens in Redis without TTL — sessions never expire server-side
  Fix: set Redis EXPIRE of 24h for regular sessions; 30 min for sensitive operations

### Low
- Retention policy document last reviewed 3 years ago — regulatory landscape changed
  Fix: schedule annual review; version-control the policy document

### Passed
- Authentication logs retained 2 years per SOC 2 evidence requirements ✓
- Automated weekly compliance verification job configured and alerting ✓
- Legal hold flag present on all entity tables ✓
```

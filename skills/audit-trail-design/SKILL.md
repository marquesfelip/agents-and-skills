---
name: audit-trail-design
description: 'Immutable audit logging, traceability, and compliance-ready event tracking. Use when: audit trail, audit log design, immutable audit log, event traceability, compliance logging, SOC 2 audit, ISO 27001 logs, who did what when, change tracking, data access log, admin action log, GDPR audit, LGPD audit, audit event schema, tamper-proof log, forensic logging, accountability log.'
argument-hint: 'Domain or system to audit (e.g., user management, financial transactions, data access) — include compliance requirements if known (SOC 2, GDPR, PCI, etc.)'
---

# Audit Trail Design Specialist

## When to Use
- Designing audit logging for a new feature or service from scratch
- Reviewing existing audit logging for completeness and tamper-resistance
- Preparing audit evidence for SOC 2, ISO 27001, HIPAA, PCI-DSS, LGPD, or GDPR
- Implementing "who did what, when, and from where" traceability requirements
- Building compliance-ready event tracking for regulated data operations

---

## Step 1 — Define What Must Be Audited

Use this matrix to determine audit event coverage:

| Event category | Must audit | Priority |
|---|---|---|
| Authentication | Login (success + failure), logout, MFA events, password change | Critical |
| Authorization | Permission denied events, privilege escalation attempts | Critical |
| Data access (PII/sensitive) | Read access to sensitive records (health, financial, government ID) | Critical |
| Data modification | Create / Update / Delete on regulated entities | Critical |
| Admin actions | User management, role changes, config changes, API key creation/revocation | Critical |
| Data export | Bulk export, CSV download, report generation containing PII | High |
| System events | Service start/stop, backup, restore, deploy, infra changes | High |
| Integration events | Webhook delivery, external API calls with sensitive data | Medium |
| Search / Query | Searches over sensitive data sets | Medium |

**Default rule:** If an action can affect another person's data or system state, it must be audited.

---

## Step 2 — Design the Audit Event Schema

Each audit event must be **self-contained** — reconstructable without joining other tables:

```json
{
  "event_id":       "01H9XVJK-...",          // ULID or UUID v7 — sortable, unique
  "event_type":     "user.password_changed",  // category.action — verb in past tense
  "timestamp":      "2024-03-15T14:32:01.342Z", // ISO 8601 UTC — never local time
  "actor": {
    "type":         "user",                   // user | service | system | anonymous
    "id":           "usr_abc123",             // internal actor ID
    "email":        "admin@example.com",      // at capture time — may change later
    "ip_address":   "203.0.113.42",           // remote IP of the action
    "user_agent":   "Mozilla/5.0 ...",        // browser/client fingerprint
    "session_id":   "sess_xyz789"             // links event to login session
  },
  "tenant_id":      "ten_qrs456",             // always present for multi-tenant systems
  "resource": {
    "type":         "user_account",           // entity type
    "id":           "usr_def456",             // entity ID affected
    "display":      "Jane Doe"                // human-readable label at capture time
  },
  "action":         "update",                 // create | read | update | delete | login | export | ...
  "outcome":        "success",                // success | failure | partial
  "changes": {                                // diff for mutations — before and after
    "email": { "before": "old@x.com", "after": "new@x.com" },
    "role":  { "before": "member",    "after": "admin" }
  },
  "request_id":     "req_uvw321",             // correlation with application logs
  "metadata": {                               // additional context, schema-extensible
    "reason":       "admin_self_service",
    "source":       "web_ui"
  }
}
```

### Field Requirements Checklist

- [ ] `event_id`: globally unique, sortable (ULID preferred over UUID v4 for time-ordered queries)
- [ ] `event_type`: namespaced, consistent vocabulary (`entity.action` pattern)
- [ ] `timestamp`: UTC only; never local time or timezone-dependent
- [ ] `actor.id`: identifies WHO performed the action (never anonymous for authenticated actions)
- [ ] `actor.ip_address`: source IP at event time (crucial for forensics)
- [ ] `tenant_id`: present on every event in multi-tenant systems
- [ ] `resource`: WHAT was acted upon
- [ ] `outcome`: was the action successful?
- [ ] `changes`: for mutations — the before/after diff (not just "something changed")

---

## Step 3 — Implement Audit Logging in Code

### Centralized Audit Logger Pattern

```python
class AuditLogger:
    def __init__(self, storage: AuditStorage):
        self._storage = storage

    def log(
        self,
        event_type: str,
        actor: Actor,
        resource: Resource,
        action: str,
        outcome: str,
        changes: dict | None = None,
        metadata: dict | None = None,
    ) -> None:
        event = AuditEvent(
            event_id=ulid.new(),
            event_type=event_type,
            timestamp=datetime.utcnow().isoformat() + 'Z',
            actor=actor,
            tenant_id=actor.tenant_id,
            resource=resource,
            action=action,
            outcome=outcome,
            changes=changes or {},
            metadata=metadata or {},
        )
        # Write MUST be synchronous for audit integrity — do not fire-and-forget
        self._storage.write(event)

# Usage at service layer
def change_user_role(current_user: User, target_user_id: str, new_role: str):
    old_role = get_user_role(target_user_id)
    update_role(target_user_id, new_role)
    audit.log(
        event_type='user.role_changed',
        actor=Actor.from_user(current_user),
        resource=Resource(type='user', id=target_user_id),
        action='update',
        outcome='success',
        changes={'role': {'before': old_role, 'after': new_role}},
    )
```

### Write Strategy

| Strategy | When to use | Risk |
|---|---|---|
| Synchronous inline write | Default — ensures event is recorded before response | Adds latency; if storage is down, action may fail |
| Transactional outbox (DB → event) | When audit storage is separate from application DB | Slightly more complex; guarantees atomicity |
| Fire-and-forget async (queue) | Never for critical audit events | Events lost if queue fails; audit gap |

**Never** use fire-and-forget for audit events. The application action and the audit event must be durably linked.

---

## Step 4 — Immutability and Tamper Resistance

Audit logs are only useful if they cannot be altered after the fact:

### Storage Controls

| Layer | Control |
|---|---|
| Database | Audit table: INSERT only; no UPDATE, no DELETE permissions for application role |
| File storage | WORM storage (S3 Object Lock, Immutable Azure Blob); access logs enabled |
| Log aggregation | Append-only log index; retention lock prevents deletion before expiry |
| Backup | Encrypted, hash-verified, stored in separate account/subscription |

### Hash Chain (Rolling Integrity)

```python
def compute_chain_hash(event: AuditEvent, previous_hash: str) -> str:
    """Each event's hash includes the previous event's hash — chain breakage detects tampering."""
    payload = json.dumps({
        'previous_hash': previous_hash,
        'event_id': event.event_id,
        'timestamp': event.timestamp,
        'event_type': event.event_type,
        'actor_id': event.actor.id,
        'resource_id': event.resource.id,
        'action': event.action,
        'outcome': event.outcome,
    }, sort_keys=True)
    return hashlib.sha256(payload.encode()).hexdigest()
```

Store `chain_hash` on each audit_events row. A periodic integrity verifier recomputes and detects gaps.

---

## Step 5 — Queryability and Compliance Reports

Audit data is only useful if it can be queried efficiently for compliance reviews:

### Recommended Indexes

```sql
CREATE TABLE audit_events (
    event_id     TEXT PRIMARY KEY,           -- ULID: time-ordered, lexicographically sortable
    event_type   TEXT NOT NULL,
    timestamp    TIMESTAMPTZ NOT NULL,
    actor_id     TEXT NOT NULL,
    tenant_id    TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id  TEXT NOT NULL,
    outcome      TEXT NOT NULL,
    payload      JSONB NOT NULL,
    chain_hash   TEXT NOT NULL
);

-- Queries required by compliance
CREATE INDEX idx_audit_tenant_ts    ON audit_events (tenant_id, timestamp DESC);
CREATE INDEX idx_audit_actor        ON audit_events (actor_id, timestamp DESC);
CREATE INDEX idx_audit_resource     ON audit_events (resource_type, resource_id, timestamp DESC);
CREATE INDEX idx_audit_event_type   ON audit_events (event_type, timestamp DESC);
```

### Standard Compliance Queries

```sql
-- All actions by a specific user (GDPR data access request)
SELECT * FROM audit_events
WHERE actor_id = 'usr_abc123'
  AND tenant_id = 'ten_xyz'
ORDER BY timestamp DESC;

-- All access to a specific record (data breach investigation)
SELECT * FROM audit_events
WHERE resource_type = 'patient_record'
  AND resource_id = 'pat_789'
ORDER BY timestamp DESC;

-- All failed authentication events in last 24h (security monitoring)
SELECT * FROM audit_events
WHERE event_type = 'user.login'
  AND outcome = 'failure'
  AND timestamp > now() - INTERVAL '24 hours';

-- Admin role escalations (privilege abuse detection)
SELECT * FROM audit_events
WHERE event_type = 'user.role_changed'
  AND payload->'changes'->'role'->>'after' = 'admin';
```

---

## Step 6 — Access Control for Audit Logs

Audit logs themselves contain sensitive metadata and must be protected:

- [ ] Audit log read access restricted to: compliance officers, security team, system admins
- [ ] Regular engineers cannot query audit logs in production (only via request to on-call)
- [ ] Access to the audit log system itself is... audited (meta-audit)
- [ ] Tenant admins can query their own tenant's audit log; cannot query other tenants
- [ ] Audit log export (bulk download) itself generates an audit event

---

## Step 7 — Retention Policy

| Compliance framework | Minimum retention | Recommended |
|---|---|---|
| SOC 2 Type II | 1 year | 2 years |
| ISO 27001 | 1 year | 2 years |
| PCI-DSS | 1 year (90 days online) | 2 years |
| HIPAA | 6 years | 6 years |
| GDPR / LGPD | Duration of processing + margin | 3–5 years for auth/access events |
| Financial regulations | 5–7 years | 7 years |

- [ ] Retention period defined per event category and documented
- [ ] Automated archival to cold storage after online retention window
- [ ] Automated deletion after full retention period (GDPR minimization)
- [ ] Retention policy version-controlled and reviewed annually

---

## Output Report

```
## Audit Trail Review: <domain>

### Critical
- No audit events recorded for admin user management actions
  Role changes, account creation/deletion, and permission grants are untracked
  Fix: instrument all admin endpoints with AuditLogger.log(); cover all CRUD operations

- Audit events deleted when associated user account is deleted (CASCADE DELETE)
  Audit trail destroyed on account closure — prevents forensic reconstruction
  Fix: remove CASCADE on audit_events FK; retain events per retention policy

### High
- Audit table has UPDATE permission for application DB user
  Application code (or SQL injection) could alter past audit records
  Fix: revoke UPDATE/DELETE on audit_events; grant INSERT only to app role

- Missing `changes` diff on data mutation events — only records "entity updated"
  Cannot determine what field values were changed without joining snapshot system
  Fix: capture before/after diff in changes column for all UPDATE operations

### Medium
- IP address not recorded in audit events
  Cannot determine origin of action for forensic analysis
  Fix: include actor.ip_address from request context in all audit events

- No index on (actor_id, timestamp) — GDPR data access queries exceed 30 s on production data
  Fix: add composite index; review query plan

### Low
- Audit events use local server time instead of UTC
  Correlation across services in different timezones is ambiguous
  Fix: use datetime.utcnow().isoformat() + 'Z'; enforce UTC at schema level (TIMESTAMPTZ)

### Passed
- Audit table is append-only in application role ✓
- ULID event IDs ensure time-ordered uniqueness ✓
- Chain hash implemented and verified nightly ✓
```

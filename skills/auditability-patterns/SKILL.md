---
name: auditability-patterns
description: >
  Auditability requirements across SaaS operations — designing immutable audit trails
  for compliance, security, and operational accountability.
  Use when: auditability, audit trail, audit log, audit design, who did what when,
  change tracking, immutable log, append-only log, event sourcing audit, compliance audit,
  SOC 2 audit, ISO 27001, GDPR audit, LGPD audit, admin action log, user action log,
  impersonation tracking, admin impersonation audit, data access log, privilege audit,
  sensitive operation tracking, audit schema, audit query, audit retention, audit export,
  audit hash chain, tamper-evident log, forensic logging, accountability log, change history.
argument-hint: >
  Describe the compliance requirement (SOC 2, GDPR, internal policy) and the operations that
  need auditing (user login, data exports, admin actions, billing changes, permission changes).
  Specify the expected audit retention period and whether the logs need to be tamper-evident.
---

# Auditability Patterns Specialist

## When to Use

Invoke this skill when you need to:
- Design an audit log schema covering who, what, when, where, and result
- Make the audit log tamper-evident and append-only
- Track admin impersonation sessions
- Audit sensitive operations (data exports, permission changes, billing)
- Build an audit log API for tenant admins and compliance exports
- Define audit retention policy and archival strategy

---

## Step 1 — Audit Log Schema

**Core audit log table — append-only, never updated:**
```sql
CREATE TABLE audit_log (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),

    -- WHO performed the action
    actor_id        UUID        REFERENCES users(id),   -- NULL for system actions
    actor_type      TEXT        NOT NULL,               -- 'user', 'system', 'api_key', 'webhook'
    actor_ip        INET,                               -- client IP at time of action
    actor_user_agent TEXT,

    -- Impersonation tracking
    impersonated_by UUID        REFERENCES users(id),   -- admin who triggered on behalf of actor_id
    impersonation_session_id UUID,

    -- WHAT happened
    action          TEXT        NOT NULL,   -- 'user.login', 'member.invite', 'data.export', 'plan.change'
    resource_type   TEXT,                  -- 'user', 'order', 'subscription', 'api_key'
    resource_id     TEXT,                  -- the specific resource acted upon
    result          TEXT        NOT NULL   -- 'success', 'failure', 'denied'
                    CHECK (result IN ('success','failure','denied')),
    failure_reason  TEXT,                  -- populated when result = 'failure' or 'denied'

    -- WHERE and WHEN
    tenant_id       UUID        REFERENCES tenants(id),
    request_id      TEXT,                  -- correlation ID from the HTTP request
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- WHAT changed
    before_state    JSONB,                 -- resource state before the action
    after_state     JSONB,                 -- resource state after the action
    metadata        JSONB NOT NULL DEFAULT '{}'  -- additional context

    -- Do not add: updated_at, deleted_at — this table is append-only
);

-- Indexes for the most common audit queries
CREATE INDEX idx_audit_tenant_created   ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_actor_created    ON audit_log(actor_id, created_at DESC);
CREATE INDEX idx_audit_resource         ON audit_log(resource_type, resource_id, created_at DESC);
CREATE INDEX idx_audit_action_tenant    ON audit_log(action, tenant_id, created_at DESC);
```

**Prevent modifications (PostgreSQL):**
```sql
-- Deny UPDATE and DELETE on the audit_log table for the application role
REVOKE UPDATE, DELETE, TRUNCATE ON audit_log FROM app_role;
-- Only INSERT is permitted
GRANT INSERT, SELECT ON audit_log TO app_role;
```

Checklist:
- [ ] `audit_log` has no `updated_at` or `deleted_at` columns — it is truly append-only
- [ ] Application DB role has `INSERT` and `SELECT` only — no `UPDATE`, `DELETE`, or `TRUNCATE`
- [ ] `actor_type` distinguishes human users from system jobs, API keys, and webhooks
- [ ] `impersonated_by` column tracks admin impersonation — not hidden in metadata
- [ ] `before_state` and `after_state` stored as JSONB for diffable change history

---

## Step 2 — Audit Logger Implementation

**Centralized audit logger — called from service layer, not handlers:**
```go
type AuditLogger interface {
    Log(ctx context.Context, entry AuditEntry) error
}

type AuditEntry struct {
    ActorID       string
    ActorType     string // "user" | "system" | "api_key"
    ImpersonatedBy string
    Action        string // "member.invite" — namespace.verb
    ResourceType  string
    ResourceID    string
    TenantID      string
    Result        string // "success" | "failure" | "denied"
    FailureReason string
    BeforeState   interface{}
    AfterState    interface{}
    Metadata      map[string]interface{}
}

type pgAuditLogger struct{ db *pgxpool.Pool }

func (l *pgAuditLogger) Log(ctx context.Context, e AuditEntry) error {
    // Extract request context for IP, request ID
    reqCtx := requestContextFromCtx(ctx)

    beforeJSON, _ := json.Marshal(e.BeforeState)
    afterJSON, _  := json.Marshal(e.AfterState)
    metaJSON, _   := json.Marshal(e.Metadata)

    _, err := l.db.Exec(ctx, `
        INSERT INTO audit_log (
            actor_id, actor_type, actor_ip, impersonated_by,
            action, resource_type, resource_id,
            tenant_id, request_id, result, failure_reason,
            before_state, after_state, metadata
        ) VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14)`,
        nullableString(e.ActorID), e.ActorType, reqCtx.IP, nullableString(e.ImpersonatedBy),
        e.Action, e.ResourceType, e.ResourceID,
        e.TenantID, reqCtx.RequestID, e.Result, nullableString(e.FailureReason),
        beforeJSON, afterJSON, metaJSON,
    )
    return err
}
```

**Audit decorator pattern — wrap service methods:**
```go
func (s *auditedTeamService) InviteMember(ctx context.Context, tenantID, email, role string) error {
    actor := ActorFromContext(ctx)

    err := s.inner.InviteMember(ctx, tenantID, email, role)

    result := "success"
    failureReason := ""
    if err != nil {
        result = "failure"
        failureReason = err.Error()
    }

    s.audit.Log(ctx, AuditEntry{
        ActorID:      actor.UserID,
        ActorType:    "user",
        TenantID:     tenantID,
        Action:       "member.invite",
        ResourceType: "user",
        ResourceID:   email,
        Result:       result,
        FailureReason: failureReason,
        Metadata:     map[string]any{"role": role},
    })

    return err
}
```

Checklist:
- [ ] Audit logging happens in the service layer — not in HTTP handlers (called regardless of request source)
- [ ] Both success and failure outcomes are logged — denied/failed operations are equally important
- [ ] Failed operations include a `failure_reason` — essential for security investigation
- [ ] Audit log writes do not fail silently — log errors to structured logging but do not block the operation

---

## Step 3 — Admin Impersonation Tracking

Admin impersonation (support staff acting as a tenant user) must be explicitly tracked.

```go
type ImpersonationSession struct {
    ID          string
    AdminID     string    // the support staff member
    TenantID    string
    ImpersonatedUserID string
    StartedAt   time.Time
    EndedAt     *time.Time
    Reason      string    // required: why is this impersonation needed
}

// Every action during an impersonation session includes impersonated_by
func actorFromImpersonationCtx(ctx context.Context) (actor, impersonatedBy string) {
    session := ImpersonationSessionFromCtx(ctx)
    if session == nil {
        return userFromCtx(ctx).ID, ""
    }
    // Actor is the impersonated user, but we track who is actually driving
    return session.ImpersonatedUserID, session.AdminID
}
```

**Audit events for impersonation itself:**
```go
s.audit.Log(ctx, AuditEntry{
    ActorID:      adminUser.ID,
    ActorType:    "user",
    Action:       "impersonation.started",
    TenantID:     targetTenant.ID,
    ResourceType: "user",
    ResourceID:   targetUser.ID,
    Result:       "success",
    Metadata: map[string]any{
        "reason": req.Reason,
        "session_id": session.ID,
    },
})
```

Checklist:
- [ ] Impersonation session explicitly created and ended — not an implicit "log in as" mechanism
- [ ] Every action during impersonation has `impersonated_by` populated — visible to tenant admins
- [ ] Impersonation requires a documented `reason` — reduces abuse; supports compliance review
- [ ] Tenant admins can see audit log entries tagged with impersonation — no hidden admin actions

---

## Step 4 — Sensitive Operation Coverage

**Minimum required audit coverage:**

| Operation Category | Actions to Audit |
|---|---|
| Authentication | login success, login failure, MFA challenge, password reset, session revoke |
| Member management | invite, role change, remove member, join team |
| Data access | bulk export, report download, data download |
| API keys | create, revoke, rotation |
| Billing | plan upgrade, plan downgrade, payment method update, subscription cancel |
| Configuration | SSO config change, security settings, integration enable/disable |
| Admin actions | any action taken via admin console (impersonation, plan override, manual status change) |
| Data mutations | create, update, delete of critical business entities |

Checklist:
- [ ] Authentication events (success and failure) in the audit log
- [ ] Data export events logged — who exported what and when (GDPR erasure requests are easier to handle)
- [ ] API key operations audited — creation, use, and revocation
- [ ] All admin console operations logged — support staff actions are not exempt

---

## Step 5 — Audit Log API and Retention

**Tenant admin audit log API:**
```go
// GET /api/audit-log?action=member.invite&from=2026-01-01&to=2026-03-29
// Tenants can only see their own audit log
func (h *AuditHandler) List(w http.ResponseWriter, r *http.Request) {
    tenant := TenantFromContext(r.Context())
    filter := parseAuditFilter(r.URL.Query())

    entries, err := h.audit.List(r.Context(), tenant.ID, filter)
    // ...
}
```

**Retention policy:**

| Log Category | Retention | Rationale |
|---|---|---|
| Authentication events | 2 years | SOC 2 requirement; breach investigation |
| Data access logs | 3 years | GDPR/LGPD; data access audit |
| Billing events | 7 years | Tax / financial record requirement |
| Admin actions | 3 years | Compliance and accountability |
| Routine operations | 1 year | Operational debugging |

**Archival — move old logs to cold storage:**
```sql
-- Monthly archival job: copy old rows to audit_log_archive, then delete from hot table
INSERT INTO audit_log_archive SELECT * FROM audit_log WHERE created_at < NOW() - INTERVAL '1 year';
DELETE FROM audit_log WHERE created_at < NOW() - INTERVAL '1 year';
```

Checklist:
- [ ] Audit log API scoped to authenticated tenant — tenants cannot query other tenants' logs
- [ ] Retention periods defined by event category — not a single blanket retention
- [ ] Archival job moves data to cold storage rather than deleting — preserved for compliance
- [ ] Audit log export function available — CSV or JSONL export for compliance submissions

---

## Output Report

### Critical
- No audit log — no ability to answer "who deleted this record?" during a security incident
- Audit table has `DELETE` permission for app role — audit records can be wiped by an attacker who compromises the app
- Admin impersonation not logged — support staff can take actions in tenant accounts with no accountability

### High
- Only successful operations logged — failed/denied attempts not recorded; attack patterns invisible
- Authentication failures not audited — brute force and credential stuffing invisible to security team
- Audit log co-mingled in main application tables — harder to protect; harder to archive separately

### Medium
- `before_state` and `after_state` not captured — can determine that a change happened, but not what changed
- Audit log API not available to tenant admins — tenants cannot self-audit; support burden increases
- No retention policy — audit log grows indefinitely; or worse, deleted uniformly to save space

### Low
- Audit log write failure silently swallowed — operations proceed but accountability is lost
- Impersonation reason not required — support staff impersonate without documented justification
- Audit log not indexed on `resource_type, resource_id` — finding the full history of a record is slow

### Passed
- Append-only audit table; app role has INSERT + SELECT only; no UPDATE/DELETE/TRUNCATE granted
- Both success and failure outcomes logged; `failure_reason` populated on denied/failed
- Admin impersonation tracked with `impersonated_by` + `impersonation_session_id`; `reason` required
- All categories covered: auth, member management, data export, API keys, billing, admin actions
- Retention periods defined by category; archival job moves to cold storage; export function available

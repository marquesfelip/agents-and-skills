---
name: customer-offboarding
description: 'Safe customer offboarding procedures for SaaS products. Use when: customer offboarding, tenant offboarding, cancellation flow, account deletion, tenant deletion, offboarding checklist, cancellation workflow, data export on cancel, offboarding safety, cancel subscription, account closure, member offboarding, seat removal, data retention on cancel, GDPR erasure request, offboarding automation, cancel and delete, offboarding audit, SaaS churn, exit flow, voluntary cancellation, account termination.'
argument-hint: 'Describe the offboarding scenario (voluntary cancellation, admin-forced deletion, GDPR erasure request), your data retention requirements (30-day window, legal hold, immediate purge), and which downstream systems must be notified (billing, storage, email, CRM).'
---

# Customer Offboarding

## When to Use

Invoke this skill when you need to:
- Implement a cancellation flow that is safe, reversible, and auditable
- Cascade offboarding side-effects: cancel billing, revoke sessions, freeze access, preserve data
- Offer a compliant data export before deletion (GDPR/LGPD right to data portability)
- Execute the hard-delete pipeline after the retention window elapses
- Notify downstream systems (Stripe, CRM, storage, email provider) without leaving orphaned records
- Handle member offboarding (removing a user from a tenant) distinct from full tenant offboarding

---

## Offboarding Scenarios

| Scenario | Initiator | Reversible? | Retention window |
|---|---|---|---|
| Voluntary cancellation | Tenant owner | Yes (reactivate during window) | Configurable (default 30 days) |
| Trial expiry without conversion | System | Yes (add payment to reactivate) | Configurable |
| Admin-forced closure | Platform admin | Admin discretion | Configurable |
| GDPR erasure request | Data subject / tenant | No (legal obligation) | Zero — purge ASAP |
| Billing failure (dunning exhausted) | Billing system | Yes (pay to restore) | Grace period |
| Member removal from tenant | Tenant admin | No (re-invite to restore) | N/A — user account persists |

---

## Step 1 — Voluntary Cancellation (Tenant-Level)

```go
type CancelTenantInput struct {
    TenantID      uuid.UUID
    RequestedBy   uuid.UUID  // user who initiated
    Reason        string     // survey response
    FeedbackNotes string
    RetentionDays int        // 0 = immediate purge (GDPR); default = 30
}

func (s *offboardingService) CancelTenant(ctx context.Context, input CancelTenantInput) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // 1. Cancel billing subscription immediately (no further charges)
        if err := s.billing.CancelNow(ctx, input.TenantID); err != nil {
            // Non-fatal: log and continue — billing can be reconciled later
            slog.Error("billing cancel failed", "tenant_id", input.TenantID, "err", err)
        }

        // 2. Revoke all active sessions (users cannot access the app)
        if err := s.sessionStore.RevokeAllForTenant(ctx, input.TenantID); err != nil {
            return fmt.Errorf("revoke sessions: %w", err)
        }

        // 3. Revoke all API keys
        if err := s.apiKeyRepo.RevokeAllForTenant(ctx, tx, input.TenantID); err != nil {
            return fmt.Errorf("revoke api keys: %w", err)
        }

        // 4. Record cancellation reason for analytics / win-back
        if err := s.cancellationRepo.Record(ctx, tx, CancellationRecord{
            TenantID:      input.TenantID,
            Reason:        input.Reason,
            Notes:         input.FeedbackNotes,
            RequestedBy:   input.RequestedBy,
            RequestedAt:   time.Now().UTC(),
        }); err != nil {
            return fmt.Errorf("record cancellation: %w", err)
        }

        // 5. Schedule deletion
        retentionDays := input.RetentionDays
        if retentionDays == 0 {
            retentionDays = DefaultRetentionDays
        }
        return s.tenantSvc.ScheduleDeletion(ctx, input.TenantID, retentionDays, Actor{
            ID:   input.RequestedBy,
            Type: "user",
        })
    })
}
```

---

## Step 2 — Cancellation Record + Churn Analytics

```sql
CREATE TABLE cancellation_records (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       UUID      NOT NULL REFERENCES tenants(id),
    requested_by    UUID      REFERENCES users(id),
    reason          TEXT,     -- 'too_expensive' | 'missing_feature' | 'switching' | 'not_using' | 'other'
    notes           TEXT,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    reactivated_at  TIMESTAMPTZ  -- set if tenant reactivates — measures win-back
);

CREATE INDEX idx_cancellations_tenant ON cancellation_records(tenant_id);
CREATE INDEX idx_cancellations_reason ON cancellation_records(reason, requested_at);
```

---

## Step 3 — Hard Delete Pipeline

Run after the retention window — orchestrate cascading deletes in FK-safe order:

```go
type DeletionStep struct {
    Name    string
    Execute func(ctx context.Context, tx DB, tenantID uuid.UUID) error
}

// deletionPipeline: ordered — child tables before parent tables
var deletionPipeline = []DeletionStep{
    {"revoke_sessions",       func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_files",           func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_invitations",     func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_api_keys",        func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_entitlements",    func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_memberships",     func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_onboarding",      func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_billing_data",    func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    {"purge_domain_data",     func(ctx context.Context, tx DB, id uuid.UUID) error { /* ... */ }},
    // Tenant row transitions to 'deleted' — row is kept, data is purged
    {"mark_deleted",          func(ctx context.Context, tx DB, id uuid.UUID) error { /* Transition to 'deleted' */ }},
}

func (s *offboardingService) ExecuteHardDelete(ctx context.Context, tenantID uuid.UUID) error {
    for _, step := range deletionPipeline {
        if err := s.db.Transaction(ctx, func(tx DB) error {
            return step.Execute(ctx, tx, tenantID)
        }); err != nil {
            // Log and halt — do NOT skip steps; partial deletes leave inconsistent state
            return fmt.Errorf("deletion step %q failed: %w", step.Name, err)
        }
        slog.Info("deletion step complete", "tenant_id", tenantID, "step", step.Name)
    }

    // Post-delete: notify external systems outside the DB transaction
    s.crm.MarkChurned(ctx, tenantID)
    s.stripe.DeleteCustomer(ctx, tenantID) // if GDPR requires full purge from Stripe too
    return nil
}
```

---

## Step 4 — Member Offboarding (Removing a User from a Tenant)

Distinct from tenant deletion — the user account persists; only the membership is removed.

```go
func (s *offboardingService) RemoveMember(ctx context.Context, tenantID, userID, removedBy uuid.UUID) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // Guard: cannot remove last owner (see organization-membership-security)
        if err := s.memberSvc.GuardLastOwner(ctx, tx, tenantID, userID); err != nil {
            return err
        }

        // Remove membership row
        if err := s.memberRepo.Delete(ctx, tx, tenantID, userID); err != nil {
            return fmt.Errorf("delete membership: %w", err)
        }

        // Revoke tenant-scoped sessions (user can still log into other tenants)
        if err := s.sessionStore.RevokeForTenantUser(ctx, tenantID, userID); err != nil {
            return fmt.Errorf("revoke sessions: %w", err)
        }

        // Revoke tenant-scoped API keys
        if err := s.apiKeyRepo.RevokeForTenantUser(ctx, tx, tenantID, userID); err != nil {
            return fmt.Errorf("revoke api keys: %w", err)
        }

        // Reassign or unassign content owned by this user
        if err := s.contentRepo.ReassignOwnership(ctx, tx, tenantID, userID, nil); err != nil {
            return fmt.Errorf("reassign content: %w", err)
        }

        // Audit log entry
        return s.auditLog.Record(ctx, tx, AuditEvent{
            TenantID:  tenantID,
            ActorID:   removedBy,
            Action:    "member.removed",
            TargetID:  userID,
        })
    })
}
```

---

## Step 5 — GDPR Erasure Request

```go
func (s *offboardingService) ProcessErasureRequest(ctx context.Context, tenantID uuid.UUID, requestedBy uuid.UUID) error {
    // GDPR: must purge ASAP — no retention window
    if err := s.CancelTenant(ctx, CancelTenantInput{
        TenantID:      tenantID,
        RequestedBy:   requestedBy,
        Reason:        "gdpr_erasure",
        RetentionDays: 0, // immediate
    }); err != nil {
        return err
    }

    // Queue immediate hard delete (not the retention-window scheduler)
    s.jobs.Enqueue(ctx, HardDeleteJob{TenantID: tenantID, Priority: High})

    // Record erasure request for compliance audit
    return s.gdprRepo.RecordErasureRequest(ctx, tenantID, requestedBy)
}
```

---

## Offboarding Checklist

| Action | Where | Notes |
|---|---|---|
| Cancel billing subscription | Stripe / payment provider | Non-fatal — reconcile if it fails |
| Revoke all sessions | Session store | In-transaction |
| Revoke API keys | DB | In-transaction — status = 'revoked' |
| Revoke pending invitations | DB | In-transaction |
| Record cancellation reason | DB | For analytics / win-back |
| Schedule deletion | DB | Retention window start |
| Send confirmation email | Email provider | Post-commit, async |
| Notify CRM | CRM API | Post-commit, async |
| Offer data export link | Email / in-app | Before access is revoked |

---

## Quality Checks

- [ ] Cancellation is split from deletion — cancel → `pending_deletion` → (retention window) → `deleted`
- [ ] Hard delete steps run in FK-safe order — child rows before parent
- [ ] `ExecuteHardDelete` aborts on any step failure — no partial deletes
- [ ] GDPR erasure requests bypass the retention window — immediate purge path
- [ ] Member removal revokes tenant-scoped sessions and API keys atomically
- [ ] Cancellation reason recorded for churn analytics — reason is a closed enum
- [ ] Data export is offered BEFORE access is revoked (in cancellation confirmation email)
- [ ] Stripe customer NOT deleted unless GDPR erasure requires it — needed for invoice history

## After Completion

- Use **`data-export-compliance`** to implement the data export offered during offboarding
- Use **`soft-delete-strategy`** for reversible deletion of individual records within an active tenant
- Use **`reactivation-flow`** for the path back to active within the retention window
- Use **`tenant-lifecycle-management`** for the `pending_deletion → deleted` transition
- Use **`audit-trail-design`** to ensure the deletion pipeline itself is auditable

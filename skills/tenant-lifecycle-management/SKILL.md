---
name: tenant-lifecycle-management
description: 'Tenant creation, suspension, and deletion lifecycle for SaaS products. Use when: tenant lifecycle, tenant creation, tenant provisioning, tenant suspension, tenant deletion, tenant state machine, tenant status, tenant deactivation, tenant archival, tenant hard delete, tenant soft delete, tenant data retention, tenant offboarding, tenant reactivation, tenant status transitions, SaaS tenant management, multi-tenant lifecycle, tenant status enum, tenant bootstrap, tenant teardown.'
argument-hint: 'Describe your tenancy model (shared DB with tenant_id, schema-per-tenant), the lifecycle event you are implementing (create, suspend, delete, reactivate), and any compliance constraints (data retention period, GDPR erasure, legal hold).'
---

# Tenant Lifecycle Management

## When to Use

Invoke this skill when you need to:
- Define all tenant status states and the valid transitions between them
- Implement tenant creation with atomic provisioning and rollback
- Suspend a tenant (billing failure, abuse, manual admin action) reversibly
- Hard-delete a tenant with cascading data cleanup and compliance retention
- Audit tenant state transitions with a tamper-proof history log
- Ensure no tenant data leaks across state boundaries (e.g., suspended tenant can still read their own data)

---

## Tenant State Machine

| State | Meaning | Data access | Billing |
|---|---|---|---|
| `provisioning` | Creation in progress — not yet usable | None | None |
| `trialing` | Trial period active | Full or feature-limited | No charge |
| `active` | Paid and healthy | Full | Charged |
| `suspended` | Temporarily blocked (billing, abuse, admin) | Read-only or none | Paused/frozen |
| `pending_deletion` | Deletion requested; retention window active | None (data preserved) | Canceled |
| `deleted` | Hard-deleted; data purged per retention policy | None | None |

**Valid transitions:**
```
provisioning   → trialing        (provisioning completed successfully)
provisioning   → deleted         (provisioning failed; cleanup required)
trialing       → active          (trial converted to paid)
trialing       → suspended       (trial expired without conversion)
trialing       → pending_deletion (tenant requested cancel during trial)
active         → suspended       (billing failure, abuse detected, admin action)
active         → pending_deletion (cancellation requested)
suspended      → active          (suspension lifted; billing restored)
suspended      → pending_deletion (tenant canceled during suspension)
pending_deletion → deleted       (retention window elapsed; purge complete)
pending_deletion → active        (reactivation during retention window)
```

---

## Step 1 — Schema

```sql
CREATE TYPE tenant_status AS ENUM (
    'provisioning', 'trialing', 'active', 'suspended', 'pending_deletion', 'deleted'
);

CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug                TEXT NOT NULL UNIQUE,   -- URL-safe identifier
    display_name        TEXT NOT NULL,
    status              tenant_status NOT NULL DEFAULT 'provisioning',
    trial_ends_at       TIMESTAMPTZ,
    suspended_at        TIMESTAMPTZ,
    suspension_reason   TEXT,                   -- 'billing_failure' | 'abuse' | 'admin'
    deletion_requested_at TIMESTAMPTZ,
    deletion_scheduled_at TIMESTAMPTZ,          -- deletion_requested_at + retention window
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_status      ON tenants(status);
CREATE INDEX idx_tenants_deletion_at ON tenants(deletion_scheduled_at) WHERE status = 'pending_deletion';

-- Immutable audit trail: every status transition is recorded
CREATE TABLE tenant_status_history (
    id           BIGSERIAL PRIMARY KEY,
    tenant_id    UUID        NOT NULL REFERENCES tenants(id),
    from_status  tenant_status,
    to_status    tenant_status NOT NULL,
    reason       TEXT,
    actor_id     UUID,                          -- NULL = system; set = admin/user who triggered
    actor_type   TEXT,                          -- 'system' | 'admin' | 'user'
    metadata     JSONB,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_status_history_tenant ON tenant_status_history(tenant_id, occurred_at DESC);
```

---

## Step 2 — Tenant Service Interface

```go
type TenantStatus string

const (
    TenantStatusProvisioning   TenantStatus = "provisioning"
    TenantStatusTrialing       TenantStatus = "trialing"
    TenantStatusActive         TenantStatus = "active"
    TenantStatusSuspended      TenantStatus = "suspended"
    TenantStatusPendingDeletion TenantStatus = "pending_deletion"
    TenantStatusDeleted        TenantStatus = "deleted"
)

type Tenant struct {
    ID                   uuid.UUID
    Slug                 string
    DisplayName          string
    Status               TenantStatus
    TrialEndsAt          *time.Time
    SuspendedAt          *time.Time
    SuspensionReason     string
    DeletionRequestedAt  *time.Time
    DeletionScheduledAt  *time.Time
    CreatedAt            time.Time
}

type TenantService interface {
    Create(ctx context.Context, input CreateTenantInput) (*Tenant, error)
    Transition(ctx context.Context, tenantID uuid.UUID, to TenantStatus, opts TransitionOptions) error
    ScheduleDeletion(ctx context.Context, tenantID uuid.UUID, retentionDays int, actor Actor) error
    PurgeDeleted(ctx context.Context, tenantID uuid.UUID) error
    GetByID(ctx context.Context, id uuid.UUID) (*Tenant, error)
}
```

---

## Step 3 — Tenant Creation (Atomic Provisioning)

```go
type CreateTenantInput struct {
    Slug        string
    DisplayName string
    OwnerEmail  string
    OwnerName   string
    PlanID      string
    TrialDays   int
}

func (s *tenantService) Create(ctx context.Context, input CreateTenantInput) (*Tenant, error) {
    var tenant *Tenant

    err := s.db.Transaction(ctx, func(tx DB) error {
        // 1. Insert tenant in 'provisioning' state
        t, err := s.repo.Insert(ctx, tx, Tenant{
            Slug:        input.Slug,
            DisplayName: input.DisplayName,
            Status:      TenantStatusProvisioning,
        })
        if err != nil {
            return fmt.Errorf("insert tenant: %w", err)
        }

        // 2. Create owner membership
        if err := s.memberRepo.CreateOwner(ctx, tx, t.ID, input.OwnerEmail, input.OwnerName); err != nil {
            return fmt.Errorf("create owner: %w", err)
        }

        // 3. Seed default plan entitlements
        if err := s.entitlementRepo.SeedPlan(ctx, tx, t.ID, input.PlanID); err != nil {
            return fmt.Errorf("seed entitlements: %w", err)
        }

        // 4. Transition to trialing (or active if no trial)
        targetStatus := TenantStatusTrialing
        var trialEndsAt *time.Time
        if input.TrialDays > 0 {
            t := time.Now().UTC().AddDate(0, 0, input.TrialDays)
            trialEndsAt = &t
        } else {
            targetStatus = TenantStatusActive
        }

        if err := s.applyTransition(ctx, tx, t.ID, TenantStatusProvisioning, targetStatus, TransitionOptions{
            TrialEndsAt: trialEndsAt,
            Actor:       SystemActor,
            Reason:      "provisioning_complete",
        }); err != nil {
            return err
        }

        tenant = t
        return nil
    })
    if err != nil {
        return nil, err
    }

    // 5. Post-commit: send welcome email, create Stripe customer (outside TX to avoid long lock)
    s.events.Publish(ctx, TenantCreatedEvent{TenantID: tenant.ID})
    return tenant, nil
}
```

---

## Step 4 — State Transition Guard

```go
// validTransitions defines allowed (from → to) pairs
var validTransitions = map[TenantStatus][]TenantStatus{
    TenantStatusProvisioning:   {TenantStatusTrialing, TenantStatusDeleted},
    TenantStatusTrialing:       {TenantStatusActive, TenantStatusSuspended, TenantStatusPendingDeletion},
    TenantStatusActive:         {TenantStatusSuspended, TenantStatusPendingDeletion},
    TenantStatusSuspended:      {TenantStatusActive, TenantStatusPendingDeletion},
    TenantStatusPendingDeletion: {TenantStatusDeleted, TenantStatusActive},
    TenantStatusDeleted:        {}, // terminal — no transitions out
}

func (s *tenantService) Transition(ctx context.Context, tenantID uuid.UUID, to TenantStatus, opts TransitionOptions) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        tenant, err := s.repo.LockByID(ctx, tx, tenantID) // SELECT ... FOR UPDATE
        if err != nil {
            return err
        }

        allowed := validTransitions[tenant.Status]
        for _, a := range allowed {
            if a == to {
                return s.applyTransition(ctx, tx, tenantID, tenant.Status, to, opts)
            }
        }
        return fmt.Errorf("%w: %s → %s", ErrInvalidTransition, tenant.Status, to)
    })
}

func (s *tenantService) applyTransition(ctx context.Context, tx DB, tenantID uuid.UUID,
    from, to TenantStatus, opts TransitionOptions) error {
    // Update tenant row
    if err := s.repo.UpdateStatus(ctx, tx, tenantID, to, opts); err != nil {
        return err
    }
    // Record immutable history entry
    return s.historyRepo.Insert(ctx, tx, TenantStatusHistory{
        TenantID:   tenantID,
        FromStatus: from,
        ToStatus:   to,
        Reason:     opts.Reason,
        ActorID:    opts.Actor.ID,
        ActorType:  opts.Actor.Type,
        Metadata:   opts.Metadata,
    })
}
```

---

## Step 5 — Deletion Pipeline

```go
const DefaultRetentionDays = 30 // configurable per compliance requirement

func (s *tenantService) ScheduleDeletion(ctx context.Context, tenantID uuid.UUID, retentionDays int, actor Actor) error {
    opts := TransitionOptions{
        Actor:               actor,
        Reason:              "deletion_requested",
        DeletionRequestedAt: ptr(time.Now().UTC()),
        DeletionScheduledAt: ptr(time.Now().UTC().AddDate(0, 0, retentionDays)),
    }
    return s.Transition(ctx, tenantID, TenantStatusPendingDeletion, opts)
}

// PurgeDeleted: called by scheduled job when deletion_scheduled_at has passed
func (s *tenantService) PurgeDeleted(ctx context.Context, tenantID uuid.UUID) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // Order matters: child rows first, tenant row last
        if err := s.memberRepo.DeleteAll(ctx, tx, tenantID); err != nil {
            return fmt.Errorf("purge members: %w", err)
        }
        if err := s.entitlementRepo.DeleteAll(ctx, tx, tenantID); err != nil {
            return fmt.Errorf("purge entitlements: %w", err)
        }
        if err := s.fileRepo.DeleteAll(ctx, tx, tenantID); err != nil {
            return fmt.Errorf("purge files: %w", err)
        }
        // Transition to 'deleted' (keeps the tenant row + history for audit)
        return s.applyTransition(ctx, tx, tenantID, TenantStatusPendingDeletion, TenantStatusDeleted, TransitionOptions{
            Actor:  SystemActor,
            Reason: "retention_window_elapsed",
        })
    })
}
```

---

## Step 6 — Scheduled Jobs

```go
// Run daily: expire trials → suspend
func (s *tenantService) ExpireTrials(ctx context.Context) error {
    expired, err := s.repo.ListByStatusAndTrialEndBefore(ctx, TenantStatusTrialing, time.Now().UTC())
    if err != nil {
        return err
    }
    for _, t := range expired {
        if err := s.Transition(ctx, t.ID, TenantStatusSuspended, TransitionOptions{
            Actor:  SystemActor,
            Reason: "trial_expired",
        }); err != nil {
            slog.Error("failed to expire trial", "tenant_id", t.ID, "err", err)
        }
    }
    return nil
}

// Run daily: purge tenants past their deletion_scheduled_at
func (s *tenantService) PurgeScheduledDeletions(ctx context.Context) error {
    due, err := s.repo.ListDueForPurge(ctx, time.Now().UTC())
    if err != nil {
        return err
    }
    for _, t := range due {
        if err := s.PurgeDeleted(ctx, t.ID); err != nil {
            slog.Error("failed to purge tenant", "tenant_id", t.ID, "err", err)
        }
    }
    return nil
}
```

---

## Decision Table: Suspension Trigger → Action

| Trigger | Target state | Access after | Reversible? |
|---|---|---|---|
| Billing failure (dunning exhausted) | `suspended` | Read-only for data export | Yes — pay to restore |
| Abuse detected | `suspended` | None | Admin review required |
| Admin manual suspend | `suspended` | Admin-configurable | Yes — admin lifts |
| Trial expired (no payment) | `suspended` | None or export-only | Yes — add payment |
| Cancellation requested | `pending_deletion` | None (data preserved) | Yes — within retention window |

---

## Quality Checks

- [ ] All tenant status transitions are guarded by `validTransitions` map — invalid transitions return a typed error
- [ ] Every transition writes a row to `tenant_status_history` in the same DB transaction
- [ ] `SELECT ... FOR UPDATE` used when reading tenant before transition (prevents concurrent state corruption)
- [ ] Post-commit side effects (emails, external APIs) are dispatched AFTER the DB transaction commits
- [ ] Scheduled deletion respects the configured `retention_days` — default ≥ 30
- [ ] `PurgeDeleted` deletes child rows in FK-safe order; tenant row kept with `status = 'deleted'` for audit
- [ ] Suspended tenants cannot authenticate or call mutating APIs (middleware checks `tenant.Status`)
- [ ] `provisioning` state is only visible internally — public APIs return 404 for provisioning tenants

## After Completion

- Use **`tenant-provisioning`** for the full new tenant setup ceremony (Stripe customer, default data seeding)
- Use **`tenant-suspension-handling`** for the logic that determines suspension based on billing or abuse signals
- Use **`reactivation-flow`** for the reverse path from `suspended` or `pending_deletion` to `active`
- Use **`customer-offboarding`** for the full cancellation and data cleanup workflow
- Use **`data-export-compliance`** to implement the data export that should be offered during `pending_deletion`

---
name: tenant-provisioning
description: >
  Automated tenant creation and initialization for SaaS products.
  Use when: tenant provisioning, tenant creation, tenant setup, tenant initialization,
  new tenant workflow, tenant onboarding, provision SaaS tenant, tenant bootstrap,
  idempotent provisioning, tenant provisioning pipeline, provisioning saga, provisioning rollback,
  provisioning failure, partial provisioning, provisioning retry, provisioning status,
  default data seeding, owner user creation, tenant welcome email, async provisioning,
  synchronous provisioning, provisioning timeout, provisioning hook, post-provisioning.
argument-hint: >
  Describe what provisioning creates (tenant record, owner user, default workspace, integrations),
  whether provisioning is sync (< 2s) or async (requires polling/webhook), and what should happen
  on partial failure. Include any external resources provisioned (S3 bucket, subdomain, etc.)
---

# Tenant Provisioning Specialist

## When to Use

Invoke this skill when you need to:
- Design the end-to-end workflow for creating a new tenant
- Make provisioning idempotent so retries are safe
- Handle partial provisioning failures and rollback
- Provision external resources (storage, DNS, schema) as part of tenant creation
- Track provisioning status for async flows
- Seed default data (workspace, roles, sample data) on tenant creation

---

## Step 1 — Define the Provisioning Workflow

**Standard provisioning steps:**
```
1. Validate input     — email unique, slug valid, plan exists
2. Create tenant      — INSERT into tenants table; status = 'provisioning'
3. Create owner user  — INSERT into users + tenant_memberships (role='owner')
4. Provision resources — external resources (schema, S3 prefix, DNS record)
5. Seed default data  — default roles, categories, sample data
6. Bill/subscribe     — create Stripe customer + trial subscription
7. Send welcome email — with login link and getting-started resources
8. Mark complete      — tenant.status = 'active'; provisioning_completed_at = NOW()
```

**Provisioning status tracking:**
```sql
ALTER TABLE tenants
    ADD COLUMN provisioning_status TEXT NOT NULL DEFAULT 'pending'
        CHECK (provisioning_status IN ('pending','in_progress','completed','failed')),
    ADD COLUMN provisioning_started_at  TIMESTAMPTZ,
    ADD COLUMN provisioning_completed_at TIMESTAMPTZ,
    ADD COLUMN provisioning_failed_at   TIMESTAMPTZ,
    ADD COLUMN provisioning_error       TEXT;
```

Checklist:
- [ ] Provisioning steps defined in explicit, ordered sequence
- [ ] `provisioning_status` tracked on the tenant record — not inferred from existence of related records
- [ ] Tenant is visible in the system only after `provisioning_status = 'completed'`
- [ ] Failing tenants remain in `failed` state for diagnostic inspection — not silently deleted

---

## Step 2 — Idempotent Provisioning

The provisioning process must be safely re-enterable — network failures and retries are inevitable.

```go
type ProvisioningService struct {
    db        *pgxpool.Pool
    userRepo  UserRepository
    billing   BillingService
    storage   StorageService
    mailer    Mailer
    audit     AuditLogger
}

type ProvisionInput struct {
    TenantID    string  // if empty, generate a new UUID
    Slug        string
    OwnerEmail  string
    OwnerName   string
    PlanID      string
}

func (s *ProvisioningService) Provision(ctx context.Context, input ProvisionInput) (*Tenant, error) {
    // Step 1: Upsert tenant record (idempotent)
    tenant, err := s.upsertTenant(ctx, input)
    if err != nil {
        return nil, fmt.Errorf("create tenant: %w", err)
    }

    // Already fully provisioned — return early (safe to call multiple times)
    if tenant.ProvisioningStatus == "completed" {
        return tenant, nil
    }

    if err := s.markInProgress(ctx, tenant.ID); err != nil {
        return nil, err
    }

    // Execute steps; each step is idempotent
    steps := []struct {
        name string
        fn   func(context.Context, *Tenant, ProvisionInput) error
    }{
        {"create_owner",    s.createOwnerUser},
        {"create_schema",   s.createTenantSchema},    // only for schema-per-tenant
        {"seed_defaults",   s.seedDefaultData},
        {"create_billing",  s.createBillingCustomer},
        {"send_welcome",    s.sendWelcomeEmail},
    }

    for _, step := range steps {
        if err := step.fn(ctx, tenant, input); err != nil {
            s.markFailed(ctx, tenant.ID, step.name+": "+err.Error())
            return nil, fmt.Errorf("provisioning step %s: %w", step.name, err)
        }
    }

    return s.markCompleted(ctx, tenant.ID)
}
```

**Idempotent step — owner user creation:**
```go
func (s *ProvisioningService) createOwnerUser(ctx context.Context, tenant *Tenant, input ProvisionInput) error {
    // Check if owner already exists — handles retry after partial failure
    existing, err := s.userRepo.FindByEmail(ctx, input.OwnerEmail)
    if err != nil && !errors.Is(err, ErrNotFound) {
        return err
    }

    var userID string
    if existing != nil {
        userID = existing.ID
    } else {
        user, err := s.userRepo.Create(ctx, &User{Email: input.OwnerEmail, Name: input.OwnerName})
        if err != nil {
            return err
        }
        userID = user.ID
    }

    // Upsert membership — idempotent
    _, err = s.db.Exec(ctx, `
        INSERT INTO tenant_memberships (tenant_id, user_id, role)
        VALUES ($1, $2, 'owner')
        ON CONFLICT (tenant_id, user_id) DO UPDATE SET role = 'owner'`,
        tenant.ID, userID,
    )
    return err
}
```

Checklist:
- [ ] Each provisioning step uses `INSERT ... ON CONFLICT DO NOTHING/UPDATE` — safe to retry
- [ ] `provisioning_status = 'completed'` checked at the start — no duplicate work on retry
- [ ] Failed provisioning records the failed step name — retry can resume from the failed step
- [ ] Provisioning is transactionally safe: tenant only becomes `active` when all steps succeed

---

## Step 3 — Async Provisioning for Slow Resources

When provisioning involves slow operations (DNS propagation, large schema migrations), use async provisioning.

```go
// Handler: enqueue provisioning job; return 202 Accepted
func (h *TenantHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateTenantRequest
    json.NewDecoder(r.Body).Decode(&req)

    tenantID := uuid.NewString()

    // Create stub tenant record synchronously
    tenant, err := h.tenantRepo.CreateStub(r.Context(), tenantID, req.Slug, req.PlanID)
    if err != nil {
        http.Error(w, "failed to reserve tenant", http.StatusInternalServerError)
        return
    }

    // Enqueue the heavy provisioning work
    h.queue.Enqueue(r.Context(), ProvisionTenantJob{
        TenantID:   tenantID,
        OwnerEmail: req.OwnerEmail,
        OwnerName:  req.OwnerName,
    })

    // Return immediately with the tenant ID for status polling
    w.WriteHeader(http.StatusAccepted)
    json.NewEncoder(w).Encode(map[string]string{
        "tenant_id":   tenantID,
        "status":      "provisioning",
        "status_url":  fmt.Sprintf("/api/tenants/%s/provisioning-status", tenantID),
    })
}

// GET /api/tenants/:id/provisioning-status
func (h *TenantHandler) ProvisioningStatus(w http.ResponseWriter, r *http.Request) {
    tenantID := chi.URLParam(r, "id")
    tenant, _  := h.tenantRepo.Get(r.Context(), tenantID)

    switch tenant.ProvisioningStatus {
    case "completed":
        w.WriteHeader(http.StatusOK)
    case "failed":
        w.WriteHeader(http.StatusInternalServerError)
    default:
        w.WriteHeader(http.StatusAccepted) // still in progress
    }
    json.NewEncoder(w).Encode(tenant)
}
```

Checklist:
- [ ] Async provisioning returns `202 Accepted` with a status URL — not `200` or `201`
- [ ] Provisioning status API is publicly accessible to the creating user — no auth required for the status URL (it's already scoped to the tenant ID)
- [ ] Provisioning timeout defined — tenant marked `failed` if not completed within N minutes
- [ ] Background job is idempotent — re-queued jobs do not double-provision

---

## Step 4 — Rollback on Failure

Full rollback is hard for distributed resources; compensating transactions are more practical.

**Rollback strategy by resource type:**

| Resource | Rollback Action |
|---|---|
| Tenant DB record | Set `provisioning_status = 'failed'`; mark for human review |
| User record | Delete if created during this provision and no other tenants |
| PostgreSQL schema | `DROP SCHEMA tenant_xyz CASCADE` |
| S3 prefix | Queue deletion job (async; does not block) |
| Billing customer | Archive in Stripe; do not delete (preserves audit) |
| DNS record | Remove via provider API |

```go
func (s *ProvisioningService) markFailed(ctx context.Context, tenantID, reason string) {
    s.db.Exec(ctx, `
        UPDATE tenants
        SET provisioning_status = 'failed',
            provisioning_failed_at = NOW(),
            provisioning_error = $2
        WHERE id = $1`,
        tenantID, reason,
    )

    // Queue cleanup for external resources
    s.queue.Enqueue(ctx, CleanupFailedTenantJob{TenantID: tenantID})

    s.audit.Log(ctx, AuditEntry{
        Action:    "tenant.provisioning_failed",
        TenantID:  tenantID,
        Metadata:  map[string]any{"reason": reason},
    })
}
```

Checklist:
- [ ] Rollback defined for every external resource provisioned
- [ ] Cleanup is async — slow rollback (DNS, S3) does not block the failure response
- [ ] Failed tenant records preserved for inspection — not immediately deleted
- [ ] Ops alerting on provisioning failures — not silently swallowed

---

## Output Report

### Critical
- Provisioning is not idempotent — retrying after a network failure creates duplicate tenants or double charges the billing customer
- Tenant reachable by other users while provisioning is incomplete — tenant appears active before setup is done

### High
- No provisioning status tracking — impossible to detect stuck or failed provisionings
- Failed provisioning silently deleted — no record for support investigation or retry
- External resources (schema, S3) not cleaned up on failure — orphaned resources accumulate

### Medium
- Async provisioning returns `201 Created` instead of `202 Accepted` — client assumes provisioning is complete
- Provisioning timeout not defined — stuck provisioning jobs run indefinitely with no alert
- Welcome email sent synchronously — email provider failure blocks the entire provisioning flow

### Low
- Default data seeding not idempotent — running twice creates duplicate default roles or categories
- Provisioning error message too generic — "provisioning failed" without indicating which step
- No ops alert on provisioning failure — failures discovered only when customers report them

### Passed
- All steps use `INSERT ... ON CONFLICT DO NOTHING/UPDATE` — idempotent on retry
- `provisioning_status` field tracks state throughout; tenant not accessible until `completed`
- Failed provisionings preserved with error details; cleanup queued asynchronously for external resources
- Async flow returns `202 Accepted` with a status URL; timeout enforced on the job
- Ops alerts fire on provisioning failures; audit log records the failed step

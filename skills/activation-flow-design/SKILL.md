---
name: activation-flow-design
description: 'User activation and setup workflows for SaaS products. Use when: activation flow, user activation, product activation, activation event, activated user, activation milestone, aha moment, time to value, activation tracking, activation metric, activation funnel, activation rate, activation criteria, first value moment, activation gate, user setup flow, setup completion, activated tenant, activation hook, activation email, activation analytics, PQL product qualified lead, activation KPI, SaaS activation.'
argument-hint: 'Describe your product''s activation criteria (e.g., first report created, first integration connected, first team member invited), the entities being activated (user, tenant, or both), and how activation relates to billing (trial conversion signal, expansion trigger, etc.).'
---

# Activation Flow Design

## When to Use

Invoke this skill when you need to:
- Define what "activated" means for your product — the specific actions a user/tenant must take
- Track activation milestones in the database to drive analytics, sales, and email sequences
- Distinguish between sign-up, activation, and conversion — three different events
- Implement the activation event emission from domain services (not from a tracking script)
- Build activation-based triggers: upgrade prompts, sales alerts, onboarding nudges
- Measure activation rate and time-to-activation as product health metrics

---

## The Activation Hierarchy

```
Sign-up       → User/tenant exists in the system (account created)
      ↓
Onboarding    → User completes setup steps (see onboarding-state-machine)
      ↓
Activation    → User experiences the core value of the product for the first time
      ↓
Conversion    → User becomes a paying customer
      ↓
Expansion     → User upgrades, adds seats, or uses more features
```

**Activation** is not onboarding completion. It is the moment when a user realizes the product's core value — the "aha moment." Define it as a concrete, observable action.

---

## Step 1 — Defining Activation Criteria

Define activation criteria as a set of measurable domain events. Examples by product type:

| Product type | Activation criteria example |
|---|---|
| Project management | First task created AND assigned to another user |
| Analytics | First report with data created and viewed |
| CRM | First contact imported AND first deal created |
| Communication | First channel created AND first message sent |
| Developer tool | First API call with a production API key |
| File storage | First file shared with an external user |

**Rule:** pick at most 1–3 criteria. Activation with 5+ criteria is rarely reached. Measure drop-off per criterion to find the real "aha moment."

```go
package activation

type Criterion string

// Define your product's activation criteria here — update to match your domain
const (
    CriterionFirstActionCreated  Criterion = "first_action_created"   // core value action
    CriterionTeamMemberInvited   Criterion = "team_member_invited"     // optional: collaboration signal
    CriterionIntegrationConnected Criterion = "integration_connected"  // optional: stickiness signal
)

// RequiredForActivation: these ALL must be met to mark a user/tenant activated
var RequiredForActivation = []Criterion{
    CriterionFirstActionCreated, // replace with your actual aha moment
}
```

---

## Step 2 — Schema

```sql
-- Per-tenant activation record
CREATE TABLE tenant_activation (
    tenant_id       UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    activated_at    TIMESTAMPTZ,          -- NULL until all required criteria met
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Individual criterion completions
CREATE TABLE activation_criteria (
    id           BIGSERIAL PRIMARY KEY,
    tenant_id    UUID      NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id      UUID      REFERENCES users(id) ON DELETE SET NULL,  -- who triggered it
    criterion    TEXT      NOT NULL,
    met_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata     JSONB,
    UNIQUE (tenant_id, criterion)  -- each criterion met once per tenant
);

CREATE INDEX idx_activation_criteria_tenant ON activation_criteria(tenant_id);

-- Optional: per-user activation (for multi-user products)
CREATE TABLE user_activation (
    user_id      UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    activated_at TIMESTAMPTZ,
    tenant_id    UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE
);
```

---

## Step 3 — ActivationService

```go
type ActivationService interface {
    RecordCriterion(ctx context.Context, tenantID, userID uuid.UUID, criterion Criterion, meta map[string]any) error
    IsActivated(ctx context.Context, tenantID uuid.UUID) (bool, error)
    GetStatus(ctx context.Context, tenantID uuid.UUID) (*ActivationStatus, error)
}

type ActivationStatus struct {
    TenantID    uuid.UUID
    ActivatedAt *time.Time
    IsActivated bool
    MetCriteria []Criterion
    Missing     []Criterion
}

func (s *activationService) RecordCriterion(ctx context.Context, tenantID, userID uuid.UUID, criterion Criterion, meta map[string]any) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // INSERT ... ON CONFLICT DO NOTHING — idempotent
        inserted, err := s.repo.InsertCriterionIfAbsent(ctx, tx, tenantID, userID, criterion, meta)
        if err != nil {
            return fmt.Errorf("record criterion: %w", err)
        }
        if !inserted {
            return nil // already recorded — idempotent
        }

        // Check if all required criteria are now met
        met, err := s.repo.ListMetCriteria(ctx, tx, tenantID)
        if err != nil {
            return err
        }

        allMet := containsAll(met, RequiredForActivation)
        if allMet {
            now := time.Now().UTC()
            if err := s.repo.MarkActivated(ctx, tx, tenantID, now); err != nil {
                return err
            }
            // Fire once — downstream: sales alert, conversion email, analytics
            s.events.Publish(ctx, TenantActivatedEvent{
                TenantID:    tenantID,
                ActivatedAt: now,
                TriggeredBy: userID,
                Criterion:   criterion,
            })
        }

        // Per-criterion event for analytics funnel
        s.events.Publish(ctx, ActivationCriterionMetEvent{
            TenantID:  tenantID,
            UserID:    userID,
            Criterion: criterion,
        })

        return nil
    })
}

func containsAll(met []Criterion, required []Criterion) bool {
    metSet := make(map[Criterion]bool, len(met))
    for _, c := range met {
        metSet[c] = true
    }
    for _, r := range required {
        if !metSet[r] {
            return false
        }
    }
    return true
}
```

---

## Step 4 — Hooking Criteria to Domain Events

Record activation criteria from the relevant domain services — not from a tracking script or client event:

```go
// In ActionService.Create (your product's core value action):
func (s *actionService) Create(ctx context.Context, tenantID, userID uuid.UUID, input ActionInput) (*Action, error) {
    action, err := s.repo.Create(ctx, tenantID, userID, input)
    if err != nil {
        return nil, err
    }

    // Record activation criterion from the domain action itself
    _ = s.activation.RecordCriterion(ctx, tenantID, userID, activation.CriterionFirstActionCreated, map[string]any{
        "action_id": action.ID,
    })
    return action, nil
}

// In InviteService — optional collaboration criterion
func (s *inviteService) Accept(ctx context.Context, token string, acceptingUserID uuid.UUID) error {
    invite, err := s.acceptInvite(ctx, token, acceptingUserID)
    if err != nil {
        return err
    }
    _ = s.activation.RecordCriterion(ctx, invite.TenantID, acceptingUserID, activation.CriterionTeamMemberInvited, nil)
    return nil
}
```

---

## Step 5 — Activation Downstream Triggers

Subscribe to `TenantActivatedEvent` to drive downstream workflows:

```go
func (h *ActivationEventHandler) Handle(ctx context.Context, event TenantActivatedEvent) error {
    // 1. Sales alert (CRM notification or Slack to sales team)
    if err := h.crm.NotifySalesActivation(ctx, event.TenantID); err != nil {
        slog.Error("crm sales notify failed", "tenant_id", event.TenantID, "err", err)
    }

    // 2. Trigger conversion email sequence
    if err := h.mailer.SendActivationEmail(ctx, event.TenantID); err != nil {
        slog.Error("activation email failed", "tenant_id", event.TenantID, "err", err)
    }

    // 3. Identify in analytics (Segment, Mixpanel, etc.)
    h.analytics.Track(ctx, event.TenantID, "Tenant Activated", map[string]any{
        "activated_at": event.ActivatedAt,
        "triggering_criterion": event.Criterion,
    })

    return nil
}
```

---

## Activation Metrics

Track these in your analytics/BI layer:

| Metric | Definition |
|---|---|
| Activation rate | `activated_tenants / all_tenants_created` (same cohort period) |
| Time to activation | `activated_at - tenants.created_at` |
| Activation by criterion | Which criterion is the last to be met (the bottleneck) |
| Trial-to-activation rate | `activated_trialing_tenants / total_trialing_tenants` |
| Activation-to-conversion rate | `converted_tenants / activated_tenants` |

---

## Quality Checks

- [ ] Activation criteria defined as constants in a single package — never magic strings
- [ ] `RecordCriterion` is idempotent — `INSERT ... ON CONFLICT DO NOTHING`
- [ ] `TenantActivatedEvent` fires exactly once: guarded by `activated_at IS NULL` check before update
- [ ] Activation criteria are recorded from domain service calls — not from client-side tracking events
- [ ] `ActivationCriterionMetEvent` emitted for every criterion — even non-required — for funnel analysis
- [ ] `IsActivated` query is fast: indexed `activated_at` column on `tenant_activation`
- [ ] Activation events are processed asynchronously — never block the user-facing request

## After Completion

- Use **`onboarding-state-machine`** for the setup steps that precede activation
- Use **`trial-management`** to understand activation as a trial conversion signal
- Use **`subscription-lifecycle`** for converting activated tenants to paying customers
- Use **`event-driven-architecture`** for reliable downstream event handling after activation

---
name: onboarding-state-machine
description: 'Onboarding progression states and step tracking for SaaS products. Use when: onboarding state machine, onboarding steps, onboarding checklist, onboarding progress, onboarding completion, onboarding flow, setup wizard, onboarding funnel, onboarding drop-off, onboarding gates, onboarding required steps, onboarding optional steps, onboarding analytics, time to value, onboarding milestone, setup progress, tenant onboarding, user onboarding, product tour, first-run experience, onboarding blocking step, activation event onboarding, aha moment tracking.'
argument-hint: 'Describe the onboarding steps your product requires (e.g., profile setup, invite team, connect integration, first action), which steps are required vs optional, and whether you track onboarding at the tenant level, user level, or both.'
---

# Onboarding State Machine

## When to Use

Invoke this skill when you need to:
- Define all onboarding steps as explicit states with completion conditions
- Track per-tenant and/or per-user onboarding progress in the database
- Gate product features or trial advancement behind required onboarding steps
- Drive an in-app checklist or progress bar from server state (never client-only)
- Emit analytics events at each step completion to measure drop-off and time-to-value
- Resume onboarding progress across sessions and devices without losing state

---

## Onboarding Architecture Principles

| Principle | Rationale |
|---|---|
| State lives in the DB, not the client | Client-side state resets on refresh / new device |
| Steps are idempotent | Completing a step twice is safe — same outcome |
| Required vs optional distinction | Gate features only on required steps |
| Completion is event-driven | Step checked off by the action itself, not a manual flag |
| Analytics emit at completion | Use step completion events for funnel analysis |

---

## Step 1 — Schema

```sql
-- One row per tenant: tracks overall onboarding completion
CREATE TABLE tenant_onboarding (
    tenant_id        UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    completed_at     TIMESTAMPTZ,          -- NULL until all required steps done
    started_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    current_step     TEXT,                 -- hint for UI: last incomplete required step
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- One row per (tenant, step): the canonical step completion record
CREATE TABLE onboarding_steps (
    id           BIGSERIAL PRIMARY KEY,
    tenant_id    UUID      NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    step_key     TEXT      NOT NULL,   -- 'profile_complete' | 'team_invited' | etc.
    completed_at TIMESTAMPTZ,          -- NULL = not yet completed
    skipped_at   TIMESTAMPTZ,          -- for optional steps user explicitly skipped
    data         JSONB,                -- step-specific completion metadata
    UNIQUE (tenant_id, step_key)
);

CREATE INDEX idx_onboarding_steps_tenant ON onboarding_steps(tenant_id);

-- Optional: per-user onboarding (for user-level steps like profile, MFA)
CREATE TABLE user_onboarding_steps (
    id           BIGSERIAL PRIMARY KEY,
    user_id      UUID      NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    step_key     TEXT      NOT NULL,
    completed_at TIMESTAMPTZ,
    UNIQUE (user_id, step_key)
);
```

---

## Step 2 — Step Registry

Define all steps in code — never scatter step keys as magic strings across the codebase.

```go
package onboarding

type StepKey string

const (
    // Required steps — must all be completed before onboarding is "done"
    StepProfileComplete   StepKey = "profile_complete"
    StepBillingAdded      StepKey = "billing_added"
    StepFirstAction       StepKey = "first_action"       // your product's core value action

    // Optional steps — tracked but do not block completion
    StepTeamInvited       StepKey = "team_invited"
    StepIntegrationLinked StepKey = "integration_linked"
    StepNotificationsSet  StepKey = "notifications_set"
)

type StepDefinition struct {
    Key         StepKey
    Label       string
    Description string
    Required    bool
    Order       int    // display order in checklist
}

var Steps = []StepDefinition{
    {StepProfileComplete,   "Complete your profile",        "Add your name and avatar",              true,  1},
    {StepBillingAdded,      "Add billing information",      "Enter a payment method",                true,  2},
    {StepFirstAction,       "Take your first action",       "Use the core product feature once",     true,  3},
    {StepTeamInvited,       "Invite a team member",         "Collaboration unlocks more value",      false, 4},
    {StepIntegrationLinked, "Connect an integration",       "Link your first external tool",         false, 5},
    {StepNotificationsSet,  "Configure notifications",      "Stay informed about key events",        false, 6},
}

var RequiredSteps = func() map[StepKey]bool {
    m := make(map[StepKey]bool)
    for _, s := range Steps {
        if s.Required {
            m[s.Key] = true
        }
    }
    return m
}()
```

---

## Step 3 — OnboardingService

```go
type OnboardingService interface {
    CompleteStep(ctx context.Context, tenantID uuid.UUID, step StepKey, data map[string]any) error
    SkipStep(ctx context.Context, tenantID uuid.UUID, step StepKey) error
    GetProgress(ctx context.Context, tenantID uuid.UUID) (*OnboardingProgress, error)
}

type OnboardingProgress struct {
    TenantID    uuid.UUID
    Steps       []StepProgress
    CompletedAt *time.Time
    IsComplete  bool          // all required steps done
    CurrentStep *StepKey      // next required incomplete step (for UI focus)
}

type StepProgress struct {
    Key         StepKey
    Required    bool
    Completed   bool
    Skipped     bool
    CompletedAt *time.Time
}

func (s *onboardingService) CompleteStep(ctx context.Context, tenantID uuid.UUID, step StepKey, data map[string]any) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // Upsert step completion — idempotent
        if err := s.repo.UpsertStep(ctx, tx, tenantID, step, data); err != nil {
            return fmt.Errorf("upsert step: %w", err)
        }

        // Check if all required steps are now complete
        allDone, err := s.repo.AllRequiredComplete(ctx, tx, tenantID, RequiredSteps)
        if err != nil {
            return err
        }

        if allDone {
            if err := s.repo.MarkOnboardingComplete(ctx, tx, tenantID); err != nil {
                return err
            }
            // Publish event for downstream: enable features, send congratulations email
            s.events.Publish(ctx, OnboardingCompletedEvent{TenantID: tenantID})
        }

        // Always emit step-level analytics event
        s.events.Publish(ctx, OnboardingStepCompletedEvent{
            TenantID: tenantID,
            Step:     step,
            Required: RequiredSteps[step],
        })

        return nil
    })
}

func (s *onboardingService) GetProgress(ctx context.Context, tenantID uuid.UUID) (*OnboardingProgress, error) {
    steps, err := s.repo.ListSteps(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    progress := &OnboardingProgress{TenantID: tenantID, Steps: steps}

    // Find first incomplete required step for UI focus
    for _, def := range Steps { // ordered by def.Order
        if !def.Required {
            continue
        }
        found := false
        for _, sp := range steps {
            if sp.Key == def.Key && (sp.Completed || sp.Skipped) {
                found = true
                break
            }
        }
        if !found {
            progress.CurrentStep = &def.Key
            break
        }
    }

    allDone := progress.CurrentStep == nil
    progress.IsComplete = allDone
    return progress, nil
}
```

---

## Step 4 — Trigger Completion from Domain Events

Onboarding steps should complete automatically when the user performs the relevant action — not from a separate "complete step" API call.

```go
// In ProfileService.Update:
func (s *profileService) Update(ctx context.Context, tenantID uuid.UUID, input ProfileInput) error {
    if err := s.repo.Update(ctx, tenantID, input); err != nil {
        return err
    }
    // Trigger automatically — user does not click "complete"
    if input.Name != "" && input.AvatarURL != "" {
        _ = s.onboarding.CompleteStep(ctx, tenantID, onboarding.StepProfileComplete, map[string]any{
            "name": input.Name,
        })
    }
    return nil
}

// In InviteService.Create:
func (s *inviteService) Create(ctx context.Context, tenantID uuid.UUID, input InviteInput) error {
    invite, err := s.createInvite(ctx, tenantID, input)
    if err != nil {
        return err
    }
    _ = s.onboarding.CompleteStep(ctx, tenantID, onboarding.StepTeamInvited, map[string]any{
        "invite_id": invite.ID,
    })
    return nil
}
```

---

## Step 5 — Onboarding Gate Middleware

Gate features that require onboarding completion:

```go
// RequireOnboardingComplete blocks access to routes that need full onboarding
func RequireOnboardingComplete(onboardingSvc OnboardingService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := auth.TenantFromCtx(r.Context())
            progress, err := onboardingSvc.GetProgress(r.Context(), tenantID)
            if err != nil {
                http.Error(w, "onboarding check failed", http.StatusInternalServerError)
                return
            }
            if !progress.IsComplete {
                render.JSON(w, http.StatusForbidden, map[string]any{
                    "error":        "onboarding_incomplete",
                    "current_step": progress.CurrentStep,
                })
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## Decision Table: Step Completion Trigger

| Step | Triggered by | Data recorded |
|---|---|---|
| `profile_complete` | `ProfileService.Update` when name + avatar set | `{"name": "..."}` |
| `billing_added` | Stripe `payment_method.attached` webhook | `{"pm_last4": "..."}` |
| `first_action` | Core value action (product-specific) | Action metadata |
| `team_invited` | `InviteService.Create` success | `{"invite_id": "..."}` |
| `integration_linked` | `IntegrationService.Connect` success | `{"integration": "..."}` |
| `notifications_set` | `NotificationService.UpdatePreferences` | `{}` |

---

## Quality Checks

- [ ] All `StepKey` constants are defined in a single registry — no magic strings in feature code
- [ ] `CompleteStep` is idempotent — calling twice does not emit duplicate events or corrupt state
- [ ] Onboarding completion check (`AllRequiredComplete`) runs in the same TX as the step upsert
- [ ] `OnboardingCompletedEvent` published exactly once (check `completed_at IS NULL` before marking)
- [ ] `GetProgress` is cached (short TTL) or reads from a read replica — called on every page load
- [ ] `RequireOnboardingComplete` middleware applied only to routes that genuinely need it — not all routes
- [ ] Skipping is allowed only for optional steps — required steps cannot be skipped
- [ ] Analytics events include `required: bool` and `step_key` for funnel analysis

## After Completion

- Use **`activation-flow-design`** for the specific activation events that mark a user as "activated"
- Use **`trial-management`** to understand how onboarding completion interacts with trial progression
- Use **`feature-flag-architecture`** to gate features on onboarding completion using flags
- Use **`tenant-lifecycle-management`** for the tenant status that gates onboarding entirely

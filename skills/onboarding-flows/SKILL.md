---
name: onboarding-flows
description: >
  Structured customer onboarding workflows for SaaS products.
  Use when: onboarding flow, customer onboarding, user onboarding, SaaS onboarding,
  onboarding checklist, activation event, product activation, first-run experience,
  setup wizard, onboarding email sequence, trial onboarding, welcome email,
  onboarding completion, aha moment, time to value, onboarding funnel, onboarding analytics,
  onboarding drop-off, onboarding gate, onboarding validation, tenant setup, workspace setup,
  team onboarding, invite flow, first-time user experience, FTUE, product-led onboarding.
argument-hint: >
  Describe your SaaS product, the target user (individual or team), the main value proposition,
  and where users currently drop off in onboarding. Include whether onboarding is self-serve,
  sales-assisted, or has a minimum setup requirement before value is delivered.
---

# Onboarding Flow Specialist

## When to Use

Invoke this skill when you need to:
- Design a structured onboarding checklist with completion tracking
- Define the activation event ("aha moment") for your product
- Build onboarding email sequences timed to user actions
- Implement a setup wizard with progress persistence
- Analyze onboarding funnel drop-off
- Gate access to core features on initial required setup steps

---

## Step 1 — Define the Activation Event

The activation event is the single milestone that predicts retention. All onboarding design flows from it.

**Identifying your activation event:**
```
Framework: What is the EARLIEST indicator that a user has received your core value?

Examples:
- Slack: Team has sent 2,000 messages
- Dropbox: First file synced across 2+ devices
- GitHub: First push to a repository
- Notion: First page shared with a collaborator
- Loom: First video watched by someone other than the creator
- B2B CRM: First contact imported + first deal created
```

**Activation event in your SaaS — define it precisely:**
```go
type ActivationEvent struct {
    Name      string  // "first_connection_created" — specific, not generic
    TenantID  string
    UserID    string
    OccurredAt time.Time
    Properties map[string]interface{}
}

// Track when activation occurs
func (s *AnalyticsService) TrackActivation(ctx context.Context, evt ActivationEvent) {
    // 1. Persist for internal analytics
    s.repo.InsertActivation(ctx, evt)

    // 2. Send to analytics (Mixpanel, Amplitude, PostHog)
    s.posthog.Capture(evt.UserID, evt.Name, evt.Properties)

    // 3. Mark tenant as activated
    s.tenantRepo.MarkActivated(ctx, evt.TenantID, evt.OccurredAt)

    // 4. Trigger onboarding completion email
    s.emailQueue.Enqueue(ctx, OnboardingCompletionEmail{TenantID: evt.TenantID})
}
```

Checklist:
- [ ] Activation event defined as a specific, measurable user action — not "signed up" or "logged in"
- [ ] Activation event tracked in the database with a timestamp
- [ ] Activation event correlated to retention data — validate it actually predicts retention
- [ ] Activation rate monitored by cohort — % of signups that activate within 7 days

---

## Step 2 — Onboarding Checklist

A checklist guides users to activation without overwhelming them.

**Database schema:**
```sql
CREATE TABLE onboarding_checklist_items (
    key             TEXT    PRIMARY KEY,   -- 'connect_data_source', 'invite_team_member'
    display_name    TEXT    NOT NULL,
    description     TEXT,
    is_required     BOOLEAN NOT NULL DEFAULT FALSE,  -- gates full access if true
    display_order   INT     NOT NULL,
    reward          TEXT    -- 'unlocks feature X'
);

CREATE TABLE tenant_onboarding_progress (
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    item_key        TEXT        NOT NULL REFERENCES onboarding_checklist_items(key),
    completed_at    TIMESTAMPTZ,
    completed_by    UUID        REFERENCES users(id),
    PRIMARY KEY (tenant_id, item_key)
);
```

**Progress tracking in the UI/API:**
```go
type OnboardingStatus struct {
    Items          []OnboardingItem `json:"items"`
    CompletedCount int              `json:"completed_count"`
    TotalCount     int              `json:"total_count"`
    ProgressPct    float64          `json:"progress_pct"`
    IsActivated    bool             `json:"is_activated"`
}

// GET /api/onboarding
func (h *OnboardingHandler) Get(w http.ResponseWriter, r *http.Request) {
    tenant := TenantFromContext(r.Context())
    status, err := h.onboarding.GetStatus(r.Context(), tenant.ID)
    json.NewEncoder(w).Encode(status)
}

// POST /api/onboarding/:key/complete — idempotent step completion
func (h *OnboardingHandler) CompleteStep(w http.ResponseWriter, r *http.Request) {
    tenant := TenantFromContext(r.Context())
    user   := UserFromContext(r.Context())
    key    := chi.URLParam(r, "key")

    err := h.onboarding.CompleteStep(r.Context(), tenant.ID, user.ID, key)
    // ...
}
```

**Auto-complete steps from product actions:**
```go
// When user connects a data source → auto-complete that onboarding step
func (s *DataSourceService) Connect(ctx context.Context, tenantID string, ds *DataSource) error {
    if err := s.repo.Create(ctx, tenantID, ds); err != nil {
        return err
    }
    // Auto-complete the onboarding step — user doesn't have to manually check it off
    s.onboarding.AutoComplete(ctx, tenantID, "connect_data_source")
    return nil
}
```

Checklist:
- [ ] Checklist has ≤ 7 items — more than 7 feels overwhelming; cut aggressively
- [ ] All required steps can be completed in < 10 minutes — time-to-value is critical
- [ ] Steps auto-complete from product actions — no manual "mark as done" for functional steps
- [ ] Required steps clearly distinguished from optional — don't block access on nice-to-have setup

---

## Step 3 — Onboarding Email Sequences

Behavioral email sequences outperform time-based sequences.

**Trigger-based sequence (not time-based):**
```
[T=0: Signup]      → Welcome email: What you can do; get started CTA
[T=1h: No action]  → "Need help?" email with video walkthrough
[T=24h: No action] → Checklist reminder: "You're 3 steps away from X"
[T=3d: No action]  → Social proof: "How Company Y achieved Z with us"
[T=7d: No action]  → Check-in offer: "Book a 15-min call with our team"

[Activation event] → "You did it!" celebration email; tips for next level
[Day 3 of trial]   → Feature spotlight: "Have you tried X?"
[Trial Day 10]     → Upgrade prompt: "3 days left; here's what you'll keep"
[Trial Day 13]     → Urgent: "Trial ends tomorrow"
```

**Email scheduling with behavioral suppression:**
```go
type EmailSequenceStep struct {
    TriggerEvent    string         // "signup", "no_activation_after"
    DelayFromTrigger time.Duration // when to send relative to trigger
    SuppressIf      []string       // suppress if these events occurred: "activated", "upgraded"
    TemplateKey     string
}

// Schedule emails, suppressing if user already progressed
func (s *EmailScheduler) Schedule(ctx context.Context, tenantID string, triggerEvent string) {
    steps := s.sequenceFor(triggerEvent)
    for _, step := range steps {
        // Check suppression conditions already met
        if s.shouldSuppress(ctx, tenantID, step.SuppressIf) {
            continue
        }
        s.queue.ScheduleAt(ctx, EmailJob{
            TenantID:    tenantID,
            TemplateKey: step.TemplateKey,
            SendAt:      time.Now().Add(step.DelayFromTrigger),
            SuppressIf:  step.SuppressIf,
        })
    }
}
```

Checklist:
- [ ] Emails triggered by user behavior — not purely by elapsed time
- [ ] Suppression logic prevents sending "get started" emails after activation
- [ ] Unsubscribe respected for marketing emails — transactional emails (billing) always send
- [ ] Trial expiry emails timed to billing cycle, not calendar time — accurate countdown

---

## Step 4 — Onboarding Analytics and Funnel

```sql
-- Track step completion rates
SELECT
    item_key,
    COUNT(*) AS tenants_completed,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM tenants WHERE created_at > NOW() - INTERVAL '30 days') AS completion_pct
FROM tenant_onboarding_progress
WHERE completed_at IS NOT NULL
  AND completed_at > NOW() - INTERVAL '30 days'
GROUP BY item_key
ORDER BY display_order;
```

**Key onboarding metrics to track:**
- **Signup → Step 1 completion rate** — are users starting?
- **Step N → Step N+1 drop-off** — where do users stop?
- **Time to activation** (median) — is it getting faster with improvements?
- **Activation rate within 7 days** — % of signups who complete the activation event
- **Activated → retained at 30 days** — validates the activation event definition

Checklist:
- [ ] Funnel drop-off by step tracked in analytics — know which step has highest abandonment
- [ ] Activation rate by signup cohort tracked — detect regressions from product or flow changes
- [ ] Time to activation (T50, T90) tracked — optimization target for onboarding improvements
- [ ] A/B tests on onboarding flow tracked — measure impact of changes on activation rate

---

## Output Report

### Critical
- No defined activation event — onboarding success unmeasurable; improvements are guesses

### High
- All onboarding steps are required — users blocked from core functionality until completing every step, including optional setup
- Onboarding checklist has > 10 items — cognitive overload; completion rate collapses
- Email sequence continues sending after activation — annoying activated users with "get started" messages

### Medium
- Steps require manual completion ("mark as done") for actions the app can detect automatically
- No onboarding funnel analytics — drop-off point unknown; improvements are designed blind
- Time-based email sequence instead of behavioral — emails arrive at wrong time relative to user intent

### Low
- Required vs. optional steps not visually distinguished — users don't know which steps block access
- Onboarding progress not persisted — page refresh resets the wizard
- No "skip onboarding" option — power users who know the product cannot bypass guided setup

### Passed
- Activation event defined, measured, and validated against 30-day retention data
- Checklist has ≤ 7 items; required vs. optional clearly distinguished; takes < 10 minutes
- Steps auto-complete from product actions; manual completion only for informational steps
- Email sequence is behavioral; suppression logic prevents post-activation emails
- Funnel drop-off by step tracked; activation rate by cohort monitored; time to activation measured

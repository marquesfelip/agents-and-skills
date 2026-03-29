---
name: feature-flag-architecture
description: >
  Feature flag design and controlled rollout strategies for SaaS products.
  Use when: feature flags, feature toggles, controlled rollout, gradual rollout, canary release,
  A/B testing, kill switch, dark launch, feature gate, flag targeting, percentage rollout,
  tenant-based flag, user-based flag, environment flag, LaunchDarkly, Unleash, Flagsmith,
  feature flag lifecycle, flag cleanup, flag debt, flag evaluation, flag context, flag SDK,
  flag override, flag audit, boolean flag, multivariate flag, flag targeting rule,
  flag dependency, flag kill switch, flag rollback, production flag, flag management.
argument-hint: >
  Describe the feature you want to flag (new UI, backend algorithm, third-party integration),
  the rollout strategy (all users, % of tenants, specific beta customers), and whether you
  need A/B testing or just a kill switch. Specify your flag management tool if in use.
---

# Feature Flag Architecture Specialist

## When to Use

Invoke this skill when you need to:
- Design a feature flag system (in-house or evaluate a vendor)
- Implement gradual rollout to a percentage of tenants or users
- Create tenant-based or user-based targeting rules
- Define a flag lifecycle policy to prevent flag debt accumulation
- Design kill switches for high-risk deployments
- Set up flag evaluation context with tenant and user attributes

---

## Step 1 — Choose Flag Types and Scope

**Flag type taxonomy:**

| Type | Description | Example |
|---|---|---|
| **Release flag** | Hide incomplete feature; gradually roll out | `new-checkout-flow` |
| **Kill switch** | Disable a feature instantly in production | `payment-provider-v2` |
| **Ops flag** | Control operational behavior | `enable-cache`, `batch-size` |
| **Experiment flag** | A/B test variant assignment | `pricing-page-variant` |
| **Permission flag** | Grant feature access by plan/tenant | `enable-api-v3`, `enable-sso` |

**Flag scope:**

| Scope | Targeting By | Use When |
|---|---|---|
| Global | All users/tenants | Kill switch; ops config |
| Tenant | `tenant_id` or plan | B2B SaaS feature rollout |
| User | `user_id` | Consumer A/B test |
| Percentage | Hash of identifier | Gradual rollout to % of tenants |
| Environment | `env: production/staging` | Dev/staging preview |

Checklist:
- [ ] Each flag has a declared type (release, kill switch, ops, experiment, permission)
- [ ] Each flag has a defined scope (global, tenant, user, %)
- [ ] Ops/kill-switch flags separated from release flags — different default behaviors
- [ ] Flag type determines its default: release flags default OFF; kill switches default ON

---

## Step 2 — Flag Evaluation Context

The evaluation context carries the attributes needed for targeting rules.

**Flag context structure:**
```go
type FlagContext struct {
    TenantID   string            // mandatory in B2B SaaS
    UserID     string
    PlanID     string
    TenantSlug string
    UserEmail  string
    Env        string            // "production", "staging"
    Attributes map[string]string // extensible for custom targeting
}

// Build context from the HTTP request
func flagContextFromRequest(r *http.Request) FlagContext {
    tenant := TenantFromContext(r.Context())
    user   := UserFromContext(r.Context())
    return FlagContext{
        TenantID:   tenant.ID,
        TenantSlug: tenant.Slug,
        PlanID:     tenant.PlanID,
        UserID:     user.ID,
        UserEmail:  user.Email,
        Env:        os.Getenv("APP_ENV"),
    }
}
```

**Consistent percentage hash (tenant-stable rollout):**
```go
// A tenant always gets the same flag value for a given percentage rollout
func percentageEnabled(flagKey, tenantID string, percentage int) bool {
    h := fnv.New32a()
    fmt.Fprintf(h, "%s:%s", flagKey, tenantID)
    return int(h.Sum32()%100) < percentage
}
```

Why FNV over random: same tenant always hashes to the same bucket — the rollout is sticky without a database lookup.

Checklist:
- [ ] Flag context includes `tenant_id` for all B2B targeting — no flag evaluated without tenant context
- [ ] Percentage rollout uses a deterministic hash — not `rand.Intn(100)` which changes per evaluation
- [ ] Context built once per request in middleware; passed to all flag evaluations in that request
- [ ] Sensitive user attributes (email, name) not sent to third-party flag SDKs without consent review

---

## Step 3 — Flag Evaluation API

**Simple in-house flag service:**
```go
type FlagService interface {
    IsEnabled(ctx context.Context, flagKey string, fctx FlagContext) bool
    Variant(ctx context.Context, flagKey string, fctx FlagContext) string // for multivariate
}

// In-house implementation backed by database
type dbFlagService struct {
    db    *pgxpool.Pool
    cache *ristretto.Cache // local in-process cache
}

func (s *dbFlagService) IsEnabled(ctx context.Context, flagKey string, fctx FlagContext) bool {
    flag, err := s.getFlag(ctx, flagKey)
    if err != nil || flag == nil {
        return false // default: disabled if flag doesn't exist
    }
    if !flag.Enabled {
        return false // globally off
    }
    return s.evaluate(flag, fctx)
}

func (s *dbFlagService) evaluate(flag *Flag, fctx FlagContext) bool {
    for _, rule := range flag.Rules {
        if rule.matches(fctx) {
            return rule.Enabled
        }
    }
    return flag.DefaultEnabled
}
```

**Database schema for in-house flags:**
```sql
CREATE TABLE feature_flags (
    key             TEXT        PRIMARY KEY,
    type            TEXT        NOT NULL CHECK (type IN ('release','kill_switch','ops','experiment','permission')),
    enabled         BOOLEAN     NOT NULL DEFAULT FALSE,
    default_enabled BOOLEAN     NOT NULL DEFAULT FALSE,
    description     TEXT,
    owner           TEXT,        -- team or person responsible
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ  -- when this flag should be cleaned up
);

CREATE TABLE flag_targeting_rules (
    id          UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_key    TEXT    NOT NULL REFERENCES feature_flags(key) ON DELETE CASCADE,
    priority    INT     NOT NULL,   -- lower number = higher priority
    attribute   TEXT    NOT NULL,   -- 'tenant_id', 'plan_id', 'percentage'
    operator    TEXT    NOT NULL,   -- 'in', 'equals', 'percentage_hash'
    values      JSONB   NOT NULL,   -- ["ten_abc", "ten_xyz"] or [50] for percentage
    enabled     BOOLEAN NOT NULL,
    UNIQUE (flag_key, priority)
);
```

Checklist:
- [ ] Flag evaluation defaults to `false` (disabled) when the flag key does not exist
- [ ] Flag evaluation never throws — any error returns the safe default (off)
- [ ] Flag values cached in-process with short TTL (< 30s) — avoids DB round-trip per request
- [ ] Kill switches cached with very short TTL (< 5s) or use pub/sub for instant propagation

---

## Step 4 — Gradual Rollout Strategies

**Rollout progression — staged approach:**

```
Stage 1: Internal only      → flag targeting: user_id IN [internal team]
Stage 2: Beta tenants       → flag targeting: tenant_id IN [beta list]
Stage 3: 5% of tenants      → percentage_hash: 5
Stage 4: 25% of tenants     → percentage_hash: 25
Stage 5: 50% of tenants     → percentage_hash: 50
Stage 6: 100% (GA)          → flag globally enabled → archive flag
```

**Rollout monitoring — each stage requires metrics:**
```go
// Record flag evaluation outcomes for analysis
var (
    flagEvaluationTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "feature_flag_evaluations_total",
    }, []string{"flag", "result"}) // result: "enabled" | "disabled"

    flagErrorRate = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "feature_flag_error_rate",
    }, []string{"flag"})
)
```

**Go/No-go gates between stages:**
- Error rate in flag-enabled cohort vs. disabled cohort does not diverge
- Latency in enabled cohort not significantly higher
- No increase in support tickets from enabled tenants
- Key business metric not negatively impacted

Checklist:
- [ ] Each rollout stage has a defined dwell time before advancing (e.g., 48 hours)
- [ ] Error rate comparison between enabled/disabled cohorts monitored continuously
- [ ] Rollout can be halted instantly — percentage set to 0 or flag globally disabled
- [ ] Beta tenants explicitly consented — not silently enrolled

---

## Step 5 — Flag Lifecycle and Debt Prevention

Flag debt is a silent killer — flags that were never cleaned up after GA become permanent complexity.

**Lifecycle states:**
```
created → rollout → stable → deprecated → removed
```

**Mandatory flag metadata:**
```sql
-- Flags must have an owner and planned expiry
INSERT INTO feature_flags(key, type, owner, expires_at, description)
VALUES (
    'new-checkout-flow',
    'release',
    'payments-team',
    NOW() + INTERVAL '90 days',  -- must review in 90 days
    'New checkout UX - roll out to all tenants by Q2'
);
```

**Automated flag debt detection:**
```go
// find flags past their expiry date
func (s *FlagService) FindExpiredFlags(ctx context.Context) ([]*Flag, error) {
    return s.db.QueryFlags(ctx,
        `SELECT key, owner, expires_at FROM feature_flags
         WHERE expires_at < NOW() AND type != 'kill_switch'
         ORDER BY expires_at`)
}
// Alert flag owners weekly about expired flags
```

**Flag removal process:**
1. Verify flag is at 100% or globally enabled
2. Remove all `IsEnabled(flagKey)` calls from code
3. Deploy without the checks
4. Delete the flag record from the database
5. Remove from tests

Checklist:
- [ ] Every flag has an `expires_at` date set at creation time
- [ ] Every flag has an `owner` (team or person) responsible for cleanup
- [ ] Automated process alerts owners when flags are past expiry
- [ ] Flag removal is a tracked engineering task — not optional after GA
- [ ] Tests do not reference removed flag keys — CI catches stale flag references

---

## Output Report

### Critical
- Flag evaluation throws exceptions — a missing flag causes 5xx errors instead of safe default behavior
- Kill switch has high cache TTL — cannot disable a broken feature in under 5 minutes in production

### High
- Percentage rollout uses `rand.Intn(100)` — same tenant gets different values per request; experience is inconsistent
- Flag context does not include `tenant_id` — B2B targeting rules cannot distinguish between tenants
- No monitoring of enabled vs. disabled cohort error rates — broken feature rolled out undetected

### Medium
- No flag expiry policy — flags accumulate indefinitely; codebase complexity grows with each release
- Flag defaults not explicitly defined — behavior on missing flag key is undefined (may default to enabled)
- Beta tenants enrolled without consent — support complaints when experimental features break

### Low
- All flags use the same cache TTL — kill switches take minutes to propagate; release flags reload every 5s unnecessarily
- No flag owner field — nobody responsible for cleanup; flags become permanent
- Flag evaluation not instrumented — no data on which flags are being evaluated and how often

### Passed
- Flag types defined (release/kill switch/ops/experiment/permission); each has appropriate default (off/on)
- Deterministic percentage hash used for stable rollout — same tenant always in same cohort
- Flag evaluation never throws; errors return the safe default
- Kill switch TTL < 5s; kill switch instantly disables feature in production
- Every flag has `owner` and `expires_at`; automated alerts notify owners of expired flags
- Error rate in enabled vs. disabled cohort compared at each rollout stage before advancing

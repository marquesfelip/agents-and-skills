---
name: abuse-prevention-system
description: 'Automated abuse prevention mechanisms for SaaS products. Use when: abuse prevention system, automated abuse prevention, abuse detection system, abuse response, API abuse prevention, account abuse, abusive tenant, abuse rate limiting, abuse quarantine, abuse ban, abuse scoring, abuse signal, abuse workflow, abuse automation, abuse pipeline, multi-signal abuse, abuse escalation, abuse remediation, bot abuse prevention, credential stuffing prevention, content abuse, trial abuse, referral abuse, seat abuse, invite spam, automated abuse response, abuse risk score, abuse review queue, abuse enforcement, SIEM SaaS.'
argument-hint: 'Describe the abuse types you are most at risk for (trial farming, API scraping, bot accounts, invite spam, data exfiltration, billing fraud), whether abuse response should be automated (block immediately) or require human review, and the scale of tenants and users in the system.'
---

# Abuse Prevention System

## When to Use

Invoke this skill when you need to:
- Build a multi-signal abuse scoring pipeline that aggregates behavior signals into a risk score
- Define automated responses: warning, throttle, quarantine, suspend, or ban — based on score thresholds
- Implement trial abuse detection: the same person creating multiple free accounts to avoid paying
- Detect and block bot accounts at signup (credential stuffing, fake registrations)
- Route high-risk accounts to a manual review queue with full context for the trust & safety team
- Emit abuse events to an audit trail for compliance and legal hold
- Combine rate limiting, anomaly detection, and identity signals into a unified abuse risk model

---

## Abuse Signal Taxonomy

```
Signal category        Examples                                   Weight
─────────────────────  ─────────────────────────────────────────  ──────
Identity signals       Disposable email, known bad domain,        High
                       VPN/Tor exit node IP, same IP as banned account

Behavioral signals     API rate >> plan limit, rapid bulk export, High
                       mass invite sending, impossible velocity

Account lifecycle      Trial → cancel → new trial (same IP/email), Medium
                       Multiple accounts same payment method

Content signals        Spam content, prohibited keywords in data  Medium

Billing signals        Chargeback filed, fraudulent payment,      High
                       failed card with many retries

Access signals         Impossible travel, new device + password   Medium
                       reset in same session, multiple failed MFA
```

---

## Step 1 — Abuse Risk Score Model

```go
// AbuseRiskScore aggregates signals into a 0–100 score
// Score thresholds map to automated and manual actions

type AbuseSignal struct {
    Name   string
    Score  int    // contribution to overall score (0–30 per signal)
    Reason string // human-readable explanation for review queue
}

type AbuseEvaluator struct {
    emailChecker    DisposableEmailChecker
    ipChecker       IPReputationChecker
    anomalyDetector UsageAnomalyDetector
    billingChecker  BillingAbuseChecker
}

func (e *AbuseEvaluator) Evaluate(ctx context.Context, tenantID string) (AbuseScore, error) {
    var signals []AbuseSignal
    total := 0

    // Signal 1: disposable or known-bad email domain
    if e.emailChecker.IsDisposable(ctx, tenantID) {
        sig := AbuseSignal{Name: "disposable_email", Score: 20, Reason: "Email domain is a known disposable address provider"}
        signals = append(signals, sig)
        total += sig.Score
    }

    // Signal 2: IP on abuse list (Tor, VPN, known attacker IP)
    ipRep, _ := e.ipChecker.Check(ctx, tenantID)
    if ipRep.IsTor || ipRep.IsVPN {
        sig := AbuseSignal{Name: "anonymous_ip", Score: 15, Reason: fmt.Sprintf("Access from %s node", ipRep.Type)}
        signals = append(signals, sig)
        total += sig.Score
    }
    if ipRep.IsKnownAbuser {
        sig := AbuseSignal{Name: "abuser_ip", Score: 25, Reason: "IP found in abuse blocklist"}
        signals = append(signals, sig)
        total += sig.Score
    }

    // Signal 3: usage rate anomaly
    if anomaly, _ := e.anomalyDetector.IsAnomalous(ctx, tenantID); anomaly {
        sig := AbuseSignal{Name: "usage_anomaly", Score: 20, Reason: "Usage rate significantly exceeds plan cohort average"}
        signals = append(signals, sig)
        total += sig.Score
    }

    // Signal 4: chargeback or payment fraud history
    if e.billingChecker.HasChargeback(ctx, tenantID) {
        sig := AbuseSignal{Name: "chargeback", Score: 35, Reason: "Chargeback filed on account"}
        signals = append(signals, sig)
        total += sig.Score
    }

    // Signal 5: multiple trial accounts (same IP or email fingerprint)
    if e.isTrialFarmer(ctx, tenantID) {
        sig := AbuseSignal{Name: "trial_farming", Score: 30, Reason: "IP or email fingerprint matches multiple trial accounts"}
        signals = append(signals, sig)
        total += sig.Score
    }

    score := min(total, 100) // cap at 100
    return AbuseScore{
        TenantID: tenantID,
        Score:    score,
        Signals:  signals,
        EvaluatedAt: time.Now().UTC(),
    }, nil
}
```

---

## Step 2 — Automated Response Pipeline

```go
// AbuseResponseRouter maps score thresholds to automated actions
type AbuseScore struct {
    TenantID    string
    Score       int
    Signals     []AbuseSignal
    EvaluatedAt time.Time
}

type AbuseResponder struct {
    suspender   TenantSuspender
    throttler   RateLimiter
    reviewer    ReviewQueueService
    notifier    AbuseNotifier
    auditLog    AuditLogger
}

func (r *AbuseResponder) Respond(ctx context.Context, score AbuseScore) {
    r.auditLog.Record(ctx, AuditEvent{
        Type:     "abuse_score_evaluated",
        TenantID: score.TenantID,
        Data:     score,
    })

    switch {
    case score.Score < 30:
        // Low risk — log only, monitor
        slog.Info("abuse score: low risk", "tenant_id", score.TenantID, "score", score.Score)

    case score.Score < 60:
        // Medium risk — add to review queue + throttle
        slog.Warn("abuse score: medium risk — throttling tenant",
            "tenant_id", score.TenantID, "score", score.Score)
        r.throttler.ThrottleTenant(ctx, score.TenantID, 24*time.Hour)
        r.reviewer.AddToQueue(ctx, ReviewItem{
            TenantID: score.TenantID,
            Priority: "medium",
            Signals:  score.Signals,
            Score:    score.Score,
        })
        r.notifier.NotifyTrustAndSafety(ctx, score)

    case score.Score < 85:
        // High risk — suspend + urgent review
        slog.Error("abuse score: high risk — suspending tenant",
            "tenant_id", score.TenantID, "score", score.Score)
        r.suspender.Suspend(ctx, score.TenantID, "automated_abuse_detection")
        r.reviewer.AddToQueue(ctx, ReviewItem{
            TenantID: score.TenantID,
            Priority: "urgent",
            Signals:  score.Signals,
            Score:    score.Score,
        })
        r.notifier.PageOnCall(ctx, score) // high risk requires human eyes fast

    default:
        // Critical — immediate ban + legal hold flag
        slog.Error("abuse score: critical — banning tenant immediately",
            "tenant_id", score.TenantID, "score", score.Score)
        r.suspender.Ban(ctx, score.TenantID, "automated_abuse_critical")
        r.reviewer.AddToQueue(ctx, ReviewItem{
            TenantID: score.TenantID,
            Priority: "critical",
            Signals:  score.Signals,
            Score:    score.Score,
            LegalHold: true,
        })
        r.notifier.PageOnCall(ctx, score)
    }
}
```

---

## Step 3 — Trial Abuse Detection

```go
// Detect accounts that repeatedly create trials to avoid paying
func (e *AbuseEvaluator) isTrialFarmer(ctx context.Context, tenantID string) bool {
    // Fingerprint: hash of normalized email domain + signup IP
    fingerprint, _ := e.identityRepo.GetFingerprint(ctx, tenantID)

    // Count active or past trial accounts with same fingerprint in last 90 days
    count, _ := e.tenantRepo.CountByFingerprint(ctx, fingerprint, 90*24*time.Hour)
    return count >= 3 // 3+ accounts with same fingerprint = trial farming
}
```

```sql
-- Fingerprint matching query
SELECT COUNT(*) FROM tenants t
JOIN tenant_fingerprints f ON f.tenant_id = t.id
WHERE f.fingerprint = $1
  AND t.created_at >= now() - '90 days'::interval
  AND t.id != $2  -- exclude self
```

---

## Step 4 — Review Queue Schema

```sql
CREATE TYPE review_priority AS ENUM ('low', 'medium', 'urgent', 'critical');
CREATE TYPE review_status AS ENUM ('open', 'investigating', 'cleared', 'actioned');

CREATE TABLE abuse_review_queue (
    id             UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID          NOT NULL,
    priority       review_priority NOT NULL,
    score          INT           NOT NULL,
    signals        JSONB         NOT NULL,  -- array of AbuseSignal
    automated_action TEXT,                 -- 'throttled' | 'suspended' | 'banned' | null
    status         review_status NOT NULL DEFAULT 'open',
    legal_hold     BOOL          NOT NULL DEFAULT false,
    detected_at    TIMESTAMPTZ   NOT NULL DEFAULT now(),
    assigned_to    UUID,                   -- trust & safety operator
    resolved_at    TIMESTAMPTZ,
    resolution     TEXT
);

CREATE INDEX idx_review_queue_open ON abuse_review_queue(priority, detected_at)
    WHERE status = 'open';
```

---

## Step 5 — Evaluation Schedule

```go
// Evaluate abuse risk on triggering events — not only periodically
// Trigger points:
//   1. On signup: check identity signals immediately
//   2. On anomaly detection: re-evaluate affected tenant
//   3. On billing event (chargeback): re-evaluate immediately
//   4. On schedule: nightly sweep of all active tenants using batch Z-score

func (j *NightlyAbuseEvaluationJob) Run(ctx context.Context) error {
    tenants, _ := j.tenantRepo.GetActiveTenantIDs(ctx)
    for _, tenantID := range tenants {
        score, err := j.evaluator.Evaluate(ctx, tenantID)
        if err != nil {
            slog.Error("abuse evaluation failed", "tenant_id", tenantID, "err", err)
            continue
        }
        if score.Score >= 30 {
            j.responder.Respond(ctx, score)
        }
        // Throttle to avoid overwhelming downstream services
        time.Sleep(10 * time.Millisecond)
    }
    return nil
}
```

---

## Abuse Response Threshold Reference

| Score range | Label | Automated action | Human action |
|---|---|---|---|
| 0–29 | Low risk | None | Visible in weekly report |
| 30–59 | Medium risk | Throttle for 24h | Added to review queue (medium) |
| 60–84 | High risk | Suspend account | Added to queue (urgent) + Slack page |
| 85–100 | Critical | Ban immediately | Queue (critical) + on-call page + legal hold |

---

## Quality Checks

- [ ] Score cap at 100 — individual signals cannot cause score > 100 through accumulation
- [ ] New tenants (< 7 days old) exempt from anomaly signal — no historical baseline yet
- [ ] Every automated action emits an audit log entry with signals and score at time of action
- [ ] `false_positive` path exists in review queue — clearing a suspended tenant lifts the restriction
- [ ] Nightly evaluation throttled (10ms between tenants) — does not spike DB or downstream services
- [ ] Legal hold tenants cannot be deleted — GDPR erasure blocked pending legal review
- [ ] Signal weights are configurable — stored in config table, not hardcoded

## After Completion

- Use **`usage-anomaly-detection`** for the usage signal that feeds the abuse score
- Use **`tenant-suspension-handling`** for the suspension mechanism invoked at high score
- Use **`audit-trail-design`** for immutable audit records of all automated abuse actions
- Use **`rate-limiting-strategies`** for the throttling mechanism applied at medium risk score

---
name: account-takeover-protection
description: 'Detection and prevention of account takeover attacks for SaaS applications. Use when: account takeover protection, ATO prevention, credential stuffing defense, account takeover detection, session hijacking prevention, suspicious login detection, impossible travel detection, new device detection, account compromise, unauthorized access detection, login anomaly detection, account security monitoring, ATO signals, stolen credential protection, account enumeration prevention, password reset hijacking, MFA bypass prevention, account takeover response.'
argument-hint: 'Describe your authentication mechanism (password, OAuth, passwordless), current security controls (rate limiting, MFA, session management), and the specific ATO vector you want to address (credential stuffing, session hijacking, phishing, SIM swap).'
---

# Account Takeover Protection

## When to Use

Invoke this skill when you need to:
- Detect and block credential stuffing attacks using leaked username/password pairs
- Identify suspicious login patterns: new device, impossible travel, unusual time/location
- Design layered ATO defenses: rate limits + MFA + behavioral detection + alerting
- Harden password reset and email change flows against takeover-via-workflow attacks
- Build an account security event log for investigation and user notification
- Respond to an active ATO incident or implement post-incident hardening

---

## Threat Model: ATO Attack Vectors

| Vector | Mechanism | Primary Defense |
|---|---|---|
| Credential stuffing | Replays leaked credential pairs at scale | HaveIBeenPwned check + rate limit + MFA |
| Brute force | Exhaustive password guessing | Per-account + per-IP lockout |
| Phishing | Victim submits credentials to fake site | MFA (TOTP/passkey bind to origin) |
| Session hijacking | Steals active session cookie or token | HttpOnly/Secure cookie, session binding |
| Password reset abuse | Triggers reset on victim's account | Rate limit + notification + token expiry |
| Email change abuse | Changes email while controlling session | Re-authentication + old-email notification |
| MFA bypass | SIM swap, OTP phishing, recovery code theft | Hardware key option, MFA recovery hardening |
| Impossible travel | Login from two distant geos within minutes | Geo-velocity anomaly detection |
| New device | First-seen device fingerprint | Device challenge / email confirmation |

---

## Step 1 — Credential Stuffing Defense

```go
// Check password against HaveIBeenPwned (k-Anonymity API — no full hash sent)
func CheckPwnedPassword(ctx context.Context, password string) (bool, error) {
    hash := sha1.Sum([]byte(password))
    hashHex := strings.ToUpper(hex.EncodeToString(hash[:]))
    prefix, suffix := hashHex[:5], hashHex[5:]

    resp, err := http.Get("https://api.pwnedpasswords.com/range/" + prefix)
    if err != nil {
        return false, nil // fail open — don't block login on HIBP unavailability
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    for _, line := range strings.Split(string(body), "\n") {
        parts := strings.SplitN(strings.TrimSpace(line), ":", 2)
        if len(parts) == 2 && strings.EqualFold(parts[0], suffix) {
            count, _ := strconv.Atoi(parts[1])
            if count > 0 {
                return true, nil // password found in breach database
            }
        }
    }
    return false, nil
}

// Use at: registration, password change, AND login (to catch pre-breach accounts)
func (s *AuthService) Login(ctx context.Context, email, password string) (*Session, error) {
    user, err := s.userRepo.GetByEmail(ctx, email)
    if err != nil {
        return nil, ErrInvalidCredentials // never reveal whether email exists
    }

    if !bcrypt.CompareHashAndPassword(user.HashedPassword, []byte(password)) {
        s.recordFailedAttempt(ctx, user.ID, ipFromCtx(ctx))
        return nil, ErrInvalidCredentials
    }

    // Async check — never block login on HIBP latency
    go func() {
        if pwned, _ := CheckPwnedPassword(context.Background(), password); pwned {
            s.flagCompromisedPassword(context.Background(), user.ID)
            s.notifyCompromisedPassword(context.Background(), user)
        }
    }()

    return s.createSession(ctx, user)
}
```

---

## Step 2 — Login Anomaly Detection

Track signals per login attempt and compute a risk score:

```go
type LoginSignal struct {
    UserID        string
    IPAddress     string
    UserAgent     string
    DeviceID      string   // derived from fingerprint
    GeoCountry    string
    GeoCity       string
    Timestamp     time.Time
}

type RiskScore int
const (
    RiskLow    RiskScore = 0
    RiskMedium RiskScore = 50
    RiskHigh   RiskScore = 100
)

func (s *AnomalyDetector) Evaluate(ctx context.Context, signal LoginSignal) (RiskScore, []string) {
    var score RiskScore
    var reasons []string

    // 1. New device never seen for this account
    if !s.deviceRepo.IsSeen(ctx, signal.UserID, signal.DeviceID) {
        score += 30
        reasons = append(reasons, "new_device")
    }

    // 2. New country never seen for this account
    if !s.geoRepo.IsSeenCountry(ctx, signal.UserID, signal.GeoCountry) {
        score += 25
        reasons = append(reasons, "new_country:"+signal.GeoCountry)
    }

    // 3. Impossible travel: last login was far away within too short a time
    if last := s.loginRepo.LastSuccessful(ctx, signal.UserID); last != nil {
        dist := geoDistance(last.GeoCity, signal.GeoCity)
        elapsed := signal.Timestamp.Sub(last.Timestamp)
        if dist > 500 && elapsed < 2*time.Hour { // 500km in under 2 hours
            score += 50
            reasons = append(reasons, "impossible_travel")
        }
    }

    // 4. High-frequency attempt (brute force indicator)
    if s.rateLimiter.RecentFailures(ctx, signal.UserID, 15*time.Minute) > 3 {
        score += 40
        reasons = append(reasons, "recent_failures")
    }

    // 5. Known malicious IP (Tor exit, abuse DB)
    if s.ipReputation.IsMalicious(ctx, signal.IPAddress) {
        score += 60
        reasons = append(reasons, "malicious_ip")
    }

    return score, reasons
}
```

**Risk score action table:**

| Score | Action |
|---|---|
| 0–30 | Allow login normally |
| 31–60 | Send suspicious login email notification; log event |
| 61–80 | Require email OTP challenge before session creation |
| 81–100 | Block + require MFA re-enrollment; alert security team |

---

## Step 3 — Security-Sensitive Flow Hardening

### Password Reset
```go
// Hardening checklist for reset tokens:
// 1. Token: 32 bytes cryptographically random — not user ID based
resetToken := make([]byte, 32)
if _, err := rand.Read(resetToken); err != nil {
    return err
}
tokenHash := sha256.Sum256(resetToken) // store hash, send plaintext

// 2. Expiry: 15–60 minutes (not 24 hours)
expiry := time.Now().Add(30 * time.Minute)

// 3. Single-use: invalidate on use
// 4. Rate limit: max 3 reset requests per account per hour
// 5. Notification: always email the account about reset initiation
// 6. Never reveal whether email exists (return same message either way)
// 7. Invalidate ALL active sessions on password change
func (s *AuthService) CompletePasswordReset(ctx context.Context, token, newPassword string) error {
    // ... validate token, update password ...
    return s.sessionRepo.InvalidateAllForUser(ctx, user.ID) // critical: boot all sessions
}
```

### Email Address Change
```go
// Re-authentication required before email change
// Send confirmation to BOTH old and new email addresses
// New email does not take effect until confirmed via new-email link
// Old email notification includes a "cancel this change" link that invalidates the flow
// Invalidate all sessions except current after email confirmed
```

### MFA Recovery Code Security
```go
// Recovery codes: generate 10 × 8-char alphanumeric codes
// Store: bcrypt hash each code individually
// Usage: single-use, mark as consumed on first use
// On exhaustion: force new MFA enrollment
// Never expose existing codes after initial display
```

---

## Step 4 — Account Security Event Log

```sql
CREATE TABLE account_security_events (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id),
    user_id     UUID        NOT NULL REFERENCES users(id),
    event_type  TEXT        NOT NULL,   -- see constants below
    ip_address  INET,
    user_agent  TEXT,
    geo_country TEXT,
    geo_city    TEXT,
    device_id   TEXT,
    risk_score  INT,
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sec_events_user_created ON account_security_events(user_id, created_at DESC);
CREATE INDEX idx_sec_events_type_created ON account_security_events(event_type, created_at DESC);
```

```go
const (
    EventLoginSuccess          = "login_success"
    EventLoginFailure          = "login_failed"
    EventLoginBlocked          = "login_blocked"
    EventLoginSuspicious       = "login_suspicious"
    EventPasswordChanged       = "password_changed"
    EventPasswordResetRequested= "password_reset_requested"
    EventPasswordResetCompleted= "password_reset_completed"
    EventEmailChangeRequested  = "email_change_requested"
    EventEmailChangeCompleted  = "email_change_completed"
    EventMFAEnabled            = "mfa_enabled"
    EventMFADisabled           = "mfa_disabled"
    EventMFAChallengeFailed    = "mfa_challenge_failed"
    EventSessionRevoked        = "session_revoked"
    EventCompromisedPassword   = "compromised_password_detected"
    EventNewDevice             = "new_device_login"
    EventImpossibleTravel      = "impossible_travel_detected"
)
```

---

## Step 5 — User Notification Strategy

| Event | Notify user? | Channel | Notes |
|---|---|---|---|
| Suspicious login | ✅ Yes | Email | Include IP, location, device; provide "Secure my account" link |
| New device login | ✅ Yes | Email | "Was this you?" with 1-click session revoke |
| Password changed | ✅ Yes | Email | If user did NOT initiate: include emergency reset link |
| Password reset initiated | ✅ Yes | Email | Alert on initiation — victim can abort |
| Email change | ✅ Yes | Both old + new | Old email gets cancel link |
| MFA disabled | ✅ Yes | Email | High-risk: always notify |
| API key created | ✅ Yes | Email + in-app | Show masked key, alert |
| Login failure (single) | ❌ No | — | Noise; log only |
| Login failure (threshold) | ✅ Yes | Email | "Multiple failed attempts" after 5 failures |

---

## Quality Checks

- [ ] HIBP password check runs at registration AND login — compromised accounts are flagged proactively
- [ ] Failed login increments both per-account AND per-IP counters independently
- [ ] Risk score evaluation runs on every successful credential verification
- [ ] Password reset tokens: 32-byte random, SHA-256 stored, 30-min expiry, single-use, revokes all sessions
- [ ] Email change requires re-auth, notifies old+new address, and is revocable from old email
- [ ] All security-relevant events written to `account_security_events` with IP, device, geo metadata
- [ ] "New device" logins send email with "Was this you?" + session revoke link
- [ ] MFA disable is always notified via email regardless of how it was triggered
- [ ] Error messages never reveal whether an email address exists in the system

## After Completion

Recommended next steps:

- Use **`brute-force-protection`** for detailed rate-limit configuration
- Use **`authentication-patterns`** for session binding, token rotation, and secure cookie attributes
- Use **`jwt-security`** for token validation, rotation, and revocation
- Use **`audit-trail-design`** to make the security event log tamper-evident

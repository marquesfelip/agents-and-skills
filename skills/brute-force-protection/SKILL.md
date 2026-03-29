---
name: brute-force-protection
description: 'Login protection strategies including lockouts, throttling, MFA enforcement, and detection signals. Use when: brute force protection, login protection, account lockout, credential brute force, password brute force, login throttling, progressive delay, CAPTCHA login, login rate limit, MFA enforcement, brute force detection, online attack prevention, login security, account lockout policy.'
argument-hint: 'Login endpoint, authentication flow, or password reset to review — or describe the credential protection strategy being designed'
---

# Brute Force Protection Specialist

## When to Use
- Reviewing or designing protection for login, password reset, or PIN entry endpoints
- Auditing rate limiting, lockout, and challenge mechanisms on authentication endpoints
- Hardening against online brute force, credential stuffing, and account enumeration
- Implementing MFA enforcement to eliminate brute force as a viable attack path
- Designing progressive challenge responses (delay, CAPTCHA, lockout)

---

## Step 1 — Understand the Attack Variants

| Attack type | How it works | Key defense |
|---|---|---|
| Online brute force | Attacker tries many passwords against one account from one IP | Per-account + per-IP rate limit |
| Distributed brute force | Same as above but from many IPs (botnet) | Per-account lockout regardless of IP |
| Credential stuffing | Real username:password pairs from breached DBs | HaveIBeenPwned check + MFA |
| Password spraying | One or few passwords tried across many accounts | Per-IP rate limit + behavioral detection |
| Reverse brute force | Many usernames tried with one common password | Per-IP + per-endpoint global throttle |
| PIN / OTP brute force | Short code guessed in few attempts | Short expiry + max 3 attempts |

---

## Step 2 — Rate Limiting Strategy (per-account AND per-IP)

Both dimensions are required — one without the other is bypassable:

| Counter key | Limit | Window | Why |
|---|---|---|---|
| `login_fail:{account_id}` | 5 failures | 15 minutes | Stop targeted brute force per account |
| `login_fail:{ip}` | 20 failures | 1 minute | Stop spray attacks and botnets |
| `login_attempt:{ip}` | 100 total | 1 hour | Detect slow spray (fewer failures) |
| `login_global:{endpoint}` | Custom | 1 minute | Detect sudden global flood |
| `reset_request:{account_id}` | 3 requests | 1 hour | Prevent reset link flooding |
| `otp_attempt:{session_id}` | 3 attempts | OTP lifetime | Short-code brute force prevention |

**Counter behavior:**
- Increment on **failure only** (do not count successful logins toward lockout).
- Reset counter on **successful** login (clear threat signal).
- Use Redis with TTL — do not store in DB for per-request performance.

---

## Step 3 — Account Lockout Policy

After the failure threshold is reached, choose an appropriate response:

| Mode | Mechanism | Trade-off |
|---|---|---|
| Hard lockout | Account locked for N minutes | Clear; attackers know it worked; DoS risk |
| Soft lockout (progressive delay) | Each failure adds delay (1s, 2s, 4s, 8s…) | Low friction; no hard cutoff; effective |
| CAPTCHA challenge | Require CAPTCHA after N failures | Blocks bots; friction for real users |
| Notify + require MFA | Alert user; require 2nd factor | Best for legitimate users under attack |
| Temporary lockout + notify | Lock for 15 min; email user | Balance of security and usability |

**Recommended combination:**
1. 1–3 failures: allow (silent)
2. 4–5 failures: add 2–5 second delay; show generic error
3. 6+ failures: require CAPTCHA
4. 10+ failures: temporary lockout (15 min) + email user notification
5. 20+ failures: extended lockout + require password reset to unlock

**Avoid permanent lockouts triggered by attacker** — an attacker can lock out all accounts by intentionally failing logins (DoS via lockout). Use temporary lockouts with automatic reset.

---

## Step 4 — Progressive Delay (Exponential Backoff)

Progressive delay is more effective than hard lockout for stopping bots while preserving UX:

```python
def calculate_delay(failures: int) -> float:
    """Returns delay in seconds based on consecutive failure count."""
    if failures <= 3:
        return 0        # no delay for first few attempts
    elif failures <= 5:
        return 1.0      # 1 second
    elif failures <= 8:
        return min(2 ** (failures - 3), 30)   # exponential: 2, 4, 8, 16, 30s cap
    else:
        return 30.0 + random.uniform(0, 5)    # max + jitter to prevent timing detection

# Apply server-side before returning response — not in headers
time.sleep(calculate_delay(current_failures[account_id]))
```

**Jitter**: add small random delay (±0–5s) to prevent timing-based enumeration of lockout state.

---

## Step 5 — Account Enumeration Prevention

Brute force and credential stuffing rely on knowing which accounts exist. Prevent enumeration:

### Login Response Normalization
```
# BAD — reveals whether username exists
"Invalid password for this account"   # account exists
"Account not found"                   # account doesn't exist

# GOOD — identical response regardless
"Invalid credentials"                  # never reveal which field is wrong
```

### Timing Normalization
Password hash comparison takes time (bcrypt ~300ms). If the account doesn't exist, return immediately — the timing difference reveals whether the account exists:

```python
def login(username: str, password: str) -> bool:
    user = db.get_user(username)

    if user is None:
        # Always run the hash check anyway (against a dummy hash) for constant timing
        bcrypt.checkpw(password.encode(), DUMMY_HASH)
        return False

    return bcrypt.checkpw(password.encode(), user.password_hash)
```

### Password Reset Enumeration
```
# BAD
"A reset link has been sent to user@example.com"   # confirms email exists
"No account found for that email"                   # reveals non-existence

# GOOD
"If an account exists with that email, a reset link has been sent"   # always same
```

---

## Step 6 — MFA as Brute Force Elimination

With strong MFA, even correct credentials from brute force are useless:

| MFA method | Effectiveness against brute force |
|---|---|
| TOTP (authenticator app) | High — 30-second window, 6-digit code, 3-attempt limit |
| SMS OTP | Medium — vulnerable to SIM swap; still stops most automated attacks |
| Hardware key (FIDO2) | Very high — phishing-resistant; no code to brute force |
| Email OTP | Medium — only as strong as email security |

**MFA enforcement for brute force scenarios:**
- Require MFA step-up when anomalous login signals are detected (new IP, new device, unusual hour).
- OTP codes: expire after 2–5 minutes; max 3 attempts before code invalidation and re-send required.
- Backup codes: single-use; hashed at rest; limited number (8–12 codes).

---

## Step 7 — Password Strength Enforcement

Weak passwords are the root cause that makes brute force viable:

| Rule | Recommendation |
|---|---|
| Minimum length | 12 characters |
| Complexity | Length > complexity rules (NIST 800-63B) |
| Blocklist | Block top-10k common passwords |
| Breach database | Reject passwords found in HaveIBeenPwned |
| Personal info | Reject password containing username, email, or birthday |
| Password meters | Real-time feedback (zxcvbn library) |

---

## Step 8 — Detection and Alerting

Beyond prevention, detect attacks in progress:

| Alert | Trigger | Response |
|---|---|---|
| Account under attack | > 10 failures on one account in 1h | Lock account; email user |
| Spray attack | > 100 failed logins across accounts from one IP in 5 min | Block IP; alert team |
| Credential stuffing pattern | > 30% of login attempts using unique passwords | Engage bot protection; enable global CAPTCHA |
| Unusual success after failures | Account succeed after 8+ failures | MFA step-up required; flag for review |
| Global login failure spike | > 3× baseline failure rate | Enable stricter global rate limits |

---

## Step 9 — Output Report

```
## Brute Force Protection Review: <service/endpoint>

### Critical
- POST /login — no rate limiting of any kind
  Credential stuffing and brute force attacks proceed without resistance
  Fix: 5 failures / 15 min per account + 20 / min per IP; progressive delay starting at failure 4

- Account enumeration via timing: user lookup returns immediately when account missing;
  bcrypt check runs only when account found → 300ms timing difference reveals existence
  Fix: always run bcrypt.checkpw() with dummy hash when account not found

### High
- POST /password-reset — success response reveals whether email is registered
  "No account found" response confirms non-existence of email
  Fix: always return "If an account exists with that email, a reset link has been sent"

- OTP codes valid for 10 minutes with no attempt limit
  6-digit OTP brute-forceable in < 10 minutes with 1,000,000 combinations
  Fix: expire OTPs after 2 minutes; invalidate after 3 failed attempts

### Medium
- No CAPTCHA triggered after repeated failures — bot-friendly login
  Fix: trigger invisible CAPTCHA after 3 failures; visual challenge after 6

- Account lockout is permanent (requires admin unlock) → DoS via lockout attack
  Fix: use 15-minute automatic temporary lockout; email user with unlock option

### Low
- No brute force detection alerting configured
  Ongoing attacks not visible to security team until damage is done
  Fix: alert on > 10 failures per account in 1h; alert on > 100 cross-account failures from one IP

### Passed
- bcrypt cost=12 making each check ~300ms — limits effective brute force rate ✓
- HaveIBeenPwned check at registration ✓
- MFA available and encouraged for all users ✓
```

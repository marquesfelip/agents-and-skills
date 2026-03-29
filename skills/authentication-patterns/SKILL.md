---
name: authentication-patterns
description: 'Secure authentication flows including sessions, tokens, OAuth, passwordless, MFA, and protection against common auth vulnerabilities. Use when: authentication design, secure login, session vs token auth, OAuth authentication, passwordless auth, magic link, TOTP, MFA implementation, brute force protection, credential stuffing, auth vulnerability, login security, multi-factor authentication, authentication review.'
argument-hint: 'Describe the authentication flow or provide the auth code/file to review'
---

# Authentication Patterns Specialist

## When to Use
- Designing or reviewing a login, registration, or authentication flow
- Choosing between session-based, token-based, OAuth, or passwordless authentication
- Adding MFA (TOTP, SMS, hardware key) to an existing system
- Auditing an auth implementation for common vulnerabilities
- Protecting against brute force, credential stuffing, and account enumeration

---

## Step 1 — Understand the Auth Context

Before advising, identify:
- Application type: web app, SPA, mobile, API-only, B2B/B2C
- Existing auth mechanism (sessions, JWTs, API keys, OAuth SSO)
- Identity provider strategy: own IdP, federated (Google, Azure AD, Auth0), or hybrid
- Sensitivity level of the resource being protected

---

## Step 2 — Select the Right Authentication Pattern

| Pattern | Best for | Avoid when |
|---|---|---|
| Session + Cookie | Server-rendered web apps, same-origin | Mobile apps, cross-domain APIs |
| JWT (Bearer token) | SPAs, mobile apps, stateless APIs | Long-lived sessions needing instant revocation |
| OAuth2 + OIDC | Third-party login (SSO), delegated access | First-party-only, no external IdP needed |
| Passwordless (magic link) | Low-friction B2C | High-security contexts needing strong factors |
| Passkeys (WebAuthn) | Phishing-resistant, high assurance | Systems needing broad legacy support |
| API keys | Machine-to-machine, service accounts | User-facing authentication |

---

## Step 3 — Secure Implementation Checklist

### Credential Handling
- [ ] Passwords hashed with Argon2id, bcrypt (cost ≥12), or scrypt — never plain, MD5, or SHA-1/256 alone
- [ ] Password comparison is constant-time (prevent timing attacks)
- [ ] No passwords logged at any log level
- [ ] Password reset tokens are random, single-use, and expire in ≤1 hour

### Login Endpoint
- [ ] Rate limited per account **and** per IP (separate counters)
- [ ] Account lockout or progressive delay after N failures (5–10 attempts)
- [ ] Generic error messages ("Invalid credentials") — no username enumeration
- [ ] CAPTCHA or proof-of-work after repeated failures
- [ ] Login events logged with timestamp, IP, user-agent (but not credentials)

### Session Management (if applicable)
- [ ] Session ID generated with cryptographically secure RNG (≥128 bits)
- [ ] Session rotated on privilege escalation and login (prevent fixation)
- [ ] Secure + HttpOnly + SameSite=Strict cookies
- [ ] Session invalidated on logout server-side (not just cookie deletion)
- See `secure-session-management` skill for full session lifecycle

### Token-Based Auth (if applicable)
- [ ] Tokens signed with strong algorithm (RS256, ES256 — never `none` or HS256 with weak secret)
- [ ] Short expiry (access: 15 min; refresh: days with rotation)
- [ ] Refresh tokens are single-use and rotated on each use
- See `jwt-security` skill for full JWT hardening

---

## Step 4 — MFA Implementation

| MFA Factor | Strength | Notes |
|---|---|---|
| TOTP (authenticator app) | High | Prefer over SMS; use `totp` or RFC 6238 compliant library |
| SMS OTP | Medium | Vulnerable to SIM swap; acceptable for low-medium risk |
| Hardware key (FIDO2/WebAuthn) | Very high | Phishing-resistant; gold standard |
| Email OTP | Medium | Only if email account is well-secured |
| Backup codes | Supplemental | Single-use, hashed at rest; required for account recovery |

MFA enforcement rules:
- Enforce MFA for admin and privileged accounts unconditionally.
- Offer MFA opt-in for all users; require for sensitive actions (payment, export, role change).
- Step-up authentication: re-prompt MFA for sensitive operations even within an active session.
- MFA bypass (rate limiting, lockout) must be as strict as primary auth.

---

## Step 5 — Common Auth Vulnerability Checklist

| Vulnerability | Test | Fix |
|---|---|---|
| Username enumeration | Try login with valid vs invalid username; compare response time and message | Return identical responses and timing for both |
| Brute force | Attempt rapid repeated logins | Rate limit + lockout + IP throttle |
| Credential stuffing | Check if breached passwords are rejected | Integrate HaveIBeenPwned API on registration/login |
| Session fixation | Observe session ID before and after login | Rotate session ID on successful authentication |
| Open redirect on login | Append `?redirect=https://evil.com` to login URL | Validate redirect target against allowlist |
| Password reset poisoning | Inject `Host` header to hijack reset link | Generate reset URL from server-side config, not request headers |
| OAuth state CSRF | Omit or replay `state` parameter | Validate `state` parameter on callback |

---

## Step 6 — Output Report

Produce a structured review:

```
## Authentication Review: <system/flow name>

### Pattern Used
Session-based with bcrypt passwords

### Critical
- Password reset link generated from Host header → vulnerable to poisoning
  Fix: use application base URL from server config

### High
- No rate limiting on /login endpoint
  Fix: 5 attempts/15min per account, 20/min per IP

### Medium
- Session not rotated on login → session fixation risk
  Fix: regenerate session ID immediately after successful auth

### Passed
- bcrypt cost=12 ✓
- Constant-time comparison ✓
- HttpOnly + Secure + SameSite=Strict cookies ✓
```

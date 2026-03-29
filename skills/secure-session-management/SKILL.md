---
name: secure-session-management
description: 'Secure session lifecycle, cookie security, expiration strategy, rotation, and session invalidation practices. Use when: session management, session security, session fixation, cookie security, HttpOnly cookie, SameSite cookie, session expiry, session timeout, session invalidation, session rotation, session hijacking, CSRF protection, logout security, secure cookie attributes.'
argument-hint: 'Session management code, middleware, or framework config to review — or describe the session strategy being designed'
---

# Secure Session Management Specialist

## When to Use
- Reviewing or designing session handling in a web application
- Auditing cookie configuration for security attributes
- Implementing session rotation, expiry, and revocation policies
- Protecting against session hijacking, fixation, and CSRF
- Designing logout and multi-device invalidation flows

---

## Step 1 — Understand the Session Context

Gather before advising:
- Session storage: in-memory, server-side store (Redis, DB), or stateless JWT
- Cookie delivery vs Authorization header
- Single-device or multi-device session support
- SSO/federated identity involved?
- Sensitivity level of the application

---

## Step 2 — Session ID Properties

A secure session identifier must be:

| Property | Requirement |
|---|---|
| Entropy | ≥128 bits of cryptographically random data |
| Generator | CSPRNG — never `Math.random()`, sequential counters, or UUIDs v1/v3 |
| Length | ≥32 hex characters (or 22+ base64url characters) |
| Format | Opaque — no encoded user info, timestamps, or predictable patterns |
| Uniqueness | Verified unique at generation time |

```
# GOOD (example pseudo-code)
session_id = crypto.randomBytes(32).toString('hex')   # 256-bit, hex-encoded

# BAD
session_id = user_id + "_" + Date.now()   # predictable, enumerable
```

---

## Step 3 — Cookie Security Attributes

Every session cookie must have all of the following:

| Attribute | Value | Purpose |
|---|---|---|
| `HttpOnly` | true | Prevents JavaScript access (XSS mitigation) |
| `Secure` | true | Cookies transmitted only over HTTPS |
| `SameSite` | `Strict` or `Lax` | CSRF protection |
| `Path` | `/` or specific path | Scope cookie to application |
| `Domain` | Explicit, minimal scope | Prevent cookie leaking to subdomains unless needed |
| `Max-Age` / `Expires` | Explicit expiry | Enforces session lifetime |

`SameSite` decision guide:
- `Strict`: safest; cookie not sent on cross-site navigations (may break OAuth callbacks)
- `Lax`: sent on top-level navigations (GET); adequate for most apps; required for OAuth flows
- `None + Secure`: only for cross-site embeds/iframes (e.g., payment widgets); high risk

---

## Step 4 — Session Lifecycle & Rotation

### On Login
- [ ] Generate a **new** session ID immediately after successful authentication (prevent session fixation).
- [ ] Invalidate any pre-authentication session that existed.
- [ ] Store minimal session data server-side; never store sensitive data in the session that lives client-side.

### During Active Session
- [ ] Implement absolute timeout: session expires after a fixed wall-clock duration (e.g., 8 hours) regardless of activity.
- [ ] Implement idle timeout: session expires after inactivity (e.g., 30 minutes).
- [ ] Rotate session ID on **privilege escalation** (e.g., user gains elevated role, MFA step-up).
- [ ] Bind session to IP and/or user-agent (optional; consider mobile/corporate NAT tradeoffs).

### On Logout
- [ ] Invalidate the session server-side — deleting only the client cookie is not sufficient.
- [ ] Clear the cookie by overwriting it with an expired, empty value.
- [ ] If multi-device: provide "log out all devices" option that invalidates all sessions for the user.
- [ ] Redirect to login page after logout — never to a cached authenticated page.

---

## Step 5 — Session Store Security

| Store type | Requirements |
|---|---|
| Redis / in-memory | Use authentication, TLS, and key prefix namespacing; set TTL on all session keys |
| Relational DB | Index by session ID; enforce TTL via scheduled cleanup or `expires_at` column with filtering |
| JWT (stateless) | Short expiry (≤15 min for access); revocation list for logout; see `jwt-security` skill |

**Never** store sessions in:
- Local filesystem in multi-instance deployments (sticky sessions are not a substitute for secure storage)
- Plain-text cookies
- localStorage / sessionStorage (XSS accessible)

---

## Step 6 — CSRF Protection

For session-cookie-based apps, implement CSRF protection:

| Method | When to use |
|---|---|
| Synchronizer Token Pattern | Form-based apps; embed CSRF token in form, validate server-side |
| Double Submit Cookie | SPAs reading cookie and sending as header |
| `SameSite=Strict` | Strong protection for same-origin apps (not sufficient alone if Lax is required) |
| Custom request header (e.g., `X-Requested-With`) | Simple SPA protection; works for non-simple requests |

Never rely on `Referer` or `Origin` headers alone — they can be absent or spoofed in some contexts.

---

## Step 7 — Common Vulnerabilities Checklist

| Vulnerability | Check | Fix |
|---|---|---|
| Session fixation | Is the session ID rotated after login? | `session.regenerate()` on auth success |
| Session hijacking | Is the cookie `HttpOnly` and `Secure`? | Add missing attributes |
| CSRF | Are state-changing requests validated against a CSRF token? | Add synchronizer token or SameSite |
| Session not invalidated on logout | Does server-side store delete the session on logout? | Delete from store, not just client |
| Predictable session ID | Is CSPRNG used? | Replace with `crypto.randomBytes()` or equivalent |
| Overly long absolute timeout | Does the session never expire? | Add both absolute and idle timeout |
| Session data in URL | Is the session ID ever in a URL? | Always use cookies only |

---

## Step 8 — Output Report

```
## Session Management Review: <application/framework>

### Critical
- Session ID not rotated on login → session fixation vulnerability
  Fix: call session.regenerate() immediately after successful authentication

### High
- Session cookie missing HttpOnly flag → accessible via JavaScript (XSS risk)
  Fix: set HttpOnly=true on session cookie configuration

- No server-side session invalidation on logout → logout is only cosmetic
  Fix: delete session record from Redis/DB on logout, don't just clear the cookie

### Medium
- No idle timeout configured → sessions remain valid indefinitely while active
  Fix: track last-activity timestamp; expire sessions after 30 minutes of inactivity

### Low
- SameSite not set (defaults to browser default) → CSRF risk in older browsers
  Fix: explicitly set SameSite=Lax

### Passed
- 256-bit CSPRNG session IDs ✓
- Secure flag set ✓
- Absolute 8-hour timeout enforced ✓
```

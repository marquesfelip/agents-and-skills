---
name: csrf-protection
description: 'CSRF mitigation including tokens, SameSite cookies, and request validation. Use when: CSRF protection, cross-site request forgery, CSRF token, synchronizer token pattern, double submit cookie, SameSite cookie, state-changing request validation, anti-CSRF, origin validation, referer validation, form security, CSRF defense, POST protection.'
argument-hint: 'Form handling, API endpoints, or middleware to review — or describe the state-changing request flow being designed'
---

# CSRF Protection Specialist

## When to Use
- Reviewing or designing CSRF protection for web forms and API endpoints
- Auditing state-changing endpoints (POST, PUT, PATCH, DELETE) for CSRF mitigation
- Choosing a CSRF defense strategy for a SPA, server-rendered app, or hybrid
- Reviewing cookie configuration in relation to CSRF exposure
- Ensuring logout, payment, and account-modification endpoints are CSRF-safe

---

## Step 1 — Understand What Needs Protection

CSRF affects **state-changing requests** authenticated via cookies (or any credentials that browsers send automatically).

| At risk | Not at risk |
|---|---|
| Session-cookie authenticated endpoints | API keys sent in Authorization header (not auto-sent) |
| POST/PUT/PATCH/DELETE endpoints | Pure GET endpoints (if they don't cause side effects) |
| Any endpoint that mutates data, charges money, changes passwords, or modifies settings | Non-mutating GETs |

> **Note:** If your API uses only `Authorization: Bearer <token>` headers and no cookies, you are generally not vulnerable to CSRF. Cookies are the key risk vector.

---

## Step 2 — Choose the Right Defense

| Strategy | Best for | How it works |
|---|---|---|
| **Synchronizer Token Pattern** | Server-rendered apps (forms) | Server embeds a random CSRF token in forms; validates on submit |
| **Double Submit Cookie** | SPAs with separate API | Cookie with random value; JS reads and sends as header; server validates both match |
| **SameSite=Strict Cookie** | Same-origin apps with no OAuth callbacks | Browser won't send cookie on cross-origin requests |
| **SameSite=Lax Cookie** | Apps needing OAuth/top-level navigation | Cookie sent on same-origin and safe top-level GET only |
| **Custom Request Header** | SPAs; simple; no token management | Server requires `X-Requested-With` or custom header; cross-origin JS can't set custom headers |
| **Origin/Referer validation** | Defense-in-depth layer | Check Origin or Referer header on sensitive endpoints |

> Use **multiple layers**: SameSite cookie + CSRF token (or custom header) provides defense-in-depth.

---

## Step 3 — Synchronizer Token Pattern Implementation

For server-rendered apps (forms):

```
1. On session creation: generate CSRF token (CSPRNG, ≥128 bits)
2. Store token in server-side session
3. Embed token in every form as a hidden field:
   <input type="hidden" name="_csrf" value="{{ csrf_token }}">
4. On form submit:
   a. Read token from request body
   b. Compare (constant-time) with token stored in session
   c. Reject with 403 if mismatch or missing
5. Rotate CSRF token after use (or per-session; per-request increases complexity)
```

**Token properties:**
- Generated with CSPRNG — never sequential, timestamp-based, or derived from user data
- At least 128 bits (32 hex chars)
- Bound to the user's session — a token from session A must not be accepted for session B
- Compared with constant-time function (prevent timing attacks)

---

## Step 4 — Double Submit Cookie Pattern (SPA)

For SPAs calling a separate API:

```
1. Server sets a CSRF cookie (non-HttpOnly, readable by JS):
   Set-Cookie: csrf-token=<random_value>; SameSite=Strict; Secure; Path=/

2. SPA JS reads cookie and includes value in every state-changing request header:
   X-CSRF-Token: <value from cookie>

3. Server validates: cookie value == header value
   If mismatch: reject with 403

4. Never store the CSRF cookie value in localStorage — read from cookie only
```

> The cookie must **not** be HttpOnly — the JS needs to read it. This is intentional.

---

## Step 5 — SameSite Cookie Configuration

| `SameSite` value | CSRF protection | Caveats |
|---|---|---|
| `Strict` | Strongest — cookie not sent on any cross-site request | Breaks OAuth redirects (coming back from IdP) |
| `Lax` | Good — cookie sent only on top-level navigation GETs | POST from cross-origin won't send cookie |
| `None + Secure` | None — cookie sent cross-origin | Requires explicit `Secure`; use only for cross-site embeds |

**Recommended configuration:**
- Session cookie: `SameSite=Lax; HttpOnly; Secure` — baseline for most apps
- Upgrade to `Strict` if OAuth callbacks are not used or handled by a separate cookie
- CSRF token cookie (for double-submit): `SameSite=Strict; Secure` (non-HttpOnly)

**SameSite alone is not sufficient** for full CSRF protection — it's a defense-in-depth layer:
- Doesn't protect if the attacker can set cookies on the user's domain (cookie injection)
- Older browsers (IE 11, some Safari) don't support SameSite

---

## Step 6 — Origin and Referer Validation

Use as an additional layer, not as the primary control:

```
On state-changing requests:
1. Check the Origin header (preferred — always present on cross-origin requests)
2. If Origin absent, check Referer
3. If both absent and the request is sensitive, reject with 403

Validation:
  allowed_origins = {"https://app.example.com", "https://api.example.com"}
  if request.headers["Origin"] not in allowed_origins:
      return 403
```

**Pitfalls:**
- `Referer` can be suppressed by browser privacy settings or `Referrer-Policy` headers — do not rely on it alone.
- `Origin` is not present on same-origin requests in some browsers — handle the absence case explicitly.
- Never allow `null` Origin (sent from sandboxed iframes used by attackers).

---

## Step 7 — Endpoints That Always Need CSRF Protection

| Endpoint type | Why |
|---|---|
| Password change | Account takeover |
| Email change | Account takeover |
| MFA enrollment/removal | Account takeover |
| Payment / purchase | Financial loss |
| Data export/deletion | Data loss or exfiltration |
| Admin actions | Platform compromise |
| Logout | Session hanging |
| OAuth authorization | OAuth CSRF → account linking attacks |

---

## Step 8 — Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| CSRF token in URL param | Token logged in access logs | Use hidden form field or header |
| Non-constant-time comparison | Token oracle via timing | Use `hmac.Equal()` or `crypto/subtle` |
| Token not bound to session | Token from one user accepted for another | Store token in session; validate against session |
| Skipping CSRF on JSON API (assuming JSON = safe) | `Content-Type: text/plain` with JSON body bypasses MIME check | Always validate CSRF on state-changing endpoints |
| CSRF protection on GET endpoints only | Missing coverage for actual mutations | Protect all non-idempotent methods |
| SameSite=None without Secure | Cookie sent in plaintext | Always pair `SameSite=None` with `Secure` |

---

## Step 9 — Output Report

```
## CSRF Protection Review: <application/service>

### Critical
- POST /account/change-password — no CSRF token validation
  Attacker can host a page that auto-submits a form to change any logged-in user's password
  Fix: add synchronizer token pattern; validate token server-side on submit

- POST /api/payments — JSON endpoint with no CSRF protection
  Content-Type header check bypassed via Flash or old browser techniques
  Fix: add X-CSRF-Token header requirement; validate server-side

### High
- Session cookie missing SameSite attribute — CSRF exposure in modern browsers
  Fix: set SameSite=Lax minimum; Strict where OAuth flows allow it

### Medium
- CSRF token compared with == operator → timing oracle possible
  Fix: replace with constant-time comparison (crypto/subtle.ConstantTimeCompare)

### Low
- Origin header not validated on logout endpoint
  Fix: add Origin allowlist check as defense-in-depth layer

### Passed
- All forms include hidden _csrf field ✓
- Token bound to session ID and validated server-side ✓
- CSRF token rotated on each login ✓
```

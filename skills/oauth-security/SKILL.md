---
name: oauth-security
description: 'Secure OAuth2/OIDC integrations, token exchange safety, scopes handling, redirect validation, and provider trust boundaries. Use when: OAuth2 security, OIDC integration, OAuth authorization code flow, PKCE, OAuth redirect validation, state parameter CSRF, scope minimization, OAuth token exchange, refresh token security, OAuth provider trust, OAuth attack, implicit flow, client credentials, OAuth misconfiguration.'
argument-hint: 'OAuth integration code, callback handler, or provider config to review — or describe the OAuth flow being implemented'
---

# OAuth Security Specialist

## When to Use
- Implementing or reviewing OAuth2 / OIDC integrations (login with Google, GitHub, Azure AD, Auth0, etc.)
- Auditing authorization code flow, PKCE, or client credentials implementations
- Validating redirect URI handling, state parameter usage, and scope requests
- Hardening token exchange, storage, and refresh flows
- Reviewing trust boundaries between your app and OAuth providers

---

## Step 1 — Understand the Integration Context

Before advising, identify:
- OAuth2 flow in use: Authorization Code, Authorization Code + PKCE, Client Credentials, Device Code
- Client type: web server app, SPA, mobile app, CLI, M2M service
- Identity provider: public (Google, GitHub) or enterprise (Azure AD, Okta, internal)
- Whether OIDC (identity) or pure OAuth2 (authorization/delegation) is the goal
- Token types: access tokens, ID tokens, refresh tokens, and where each is used

---

## Step 2 — Choose the Right Flow

| Flow | Use when | Security level |
|---|---|---|
| Authorization Code + PKCE | SPAs, mobile apps, public clients | High — no client secret needed |
| Authorization Code (server-side) | Server-rendered apps with client secret stored server-side | High |
| Client Credentials | M2M / service-to-service; no user involved | High — keep secret server-side |
| Device Code | CLI tools, TVs, limited-input devices | Medium |
| Implicit Flow | **Deprecated — do not use** | Low — tokens in URL fragment |
| Resource Owner Password | **Deprecated — do not use** | Low — requires user password |

---

## Step 3 — Authorization Request Security

### State Parameter (CSRF Protection)
- Generate a cryptographically random `state` value (≥128 bits) before redirecting to the provider.
- Store the `state` in the user's session (server-side or secure cookie).
- On callback, **verify the `state` exactly matches** what was stored — reject mismatches unconditionally.
- Each authorization request must use a unique `state` — never reuse.

### PKCE (for Public Clients)
- Generate a random `code_verifier` (43–128 chars, URL-safe random), store it server-side or in memory.
- Send `code_challenge = BASE64URL(SHA256(code_verifier))` with `code_challenge_method=S256`.
- On token exchange, send the original `code_verifier` — the server validates the hash.
- Never use `code_challenge_method=plain` in production.

### Nonce (for OIDC)
- Generate a random `nonce`, include it in the authorization request.
- Validate the `nonce` claim in the returned ID token.
- Prevents ID token replay attacks.

---

## Step 4 — Redirect URI Validation

Redirect URI mismatches are the most commonly exploited OAuth vulnerability:

| Rule | Implementation |
|---|---|
| Register exact redirect URIs | Register `https://app.example.com/auth/callback` — not wildcard patterns |
| No wildcard subdomains | Never register `*.example.com` as a redirect |
| Validate on both sides | Your app must also verify the redirect URI matches what you sent |
| HTTPS only | Never register `http://` redirect URIs in production |
| No open redirect | If redirect URI is configurable, validate against a strict allowlist |

```
# BAD — accepts arbitrary redirect parameter
/auth?redirect=https://evil.com/steal-token

# GOOD — validate redirect target against allowlist
if redirect_uri not in ALLOWED_REDIRECT_URIS:
    return 400 Bad Request
```

---

## Step 5 — Token Exchange & Storage

### Authorization Code Exchange
- Exchange the code for tokens **server-side** (never client-side for confidential clients).
- Use the code exactly once — codes expire fast (≤10 minutes); reject reuse.
- Include `client_id` + `client_secret` (or PKCE verifier for public clients) in the exchange.
- Validate the `iss`, `aud`, and `exp` of returned ID tokens (see `jwt-security` skill).

### Access Token Handling
- Treat provider access tokens as opaque unless you're the resource server.
- Never log access tokens or include them in URLs (query strings, logs, analytics).
- Pass access tokens via `Authorization: Bearer <token>` header only.
- Store access tokens in memory (SPA) or secure server-side session (not localStorage).

### Refresh Token Handling
- Store refresh tokens server-side — never expose to the browser directly.
- Rotate refresh tokens on each use (most providers support this).
- Revoke refresh tokens on logout via the provider's revocation endpoint.

---

## Step 6 — Scope Minimization

- Request only the scopes your application actually needs — never request maximum scopes for convenience.
- If a scope is needed rarely (e.g., calendar access), request it incrementally at the time it's needed.
- Review and document which scopes are requested and why.
- Validate that the granted scopes include what you expected — providers may grant fewer scopes.

| Anti-pattern | Fix |
|---|---|
| Requesting `*.read` or `admin` scope for basic functionality | Request granular scopes (e.g., `email`, `profile`) |
| Never reviewing granted scopes vs requested scopes | Add scope validation after token exchange |
| Requesting offline access universally | Only request `offline_access` / refresh tokens when needed |

---

## Step 7 — Provider Trust Boundaries

- Never trust claims in ID tokens or access tokens without signature validation.
- Validate `iss` matches the expected provider's issuer URL exactly.
- Validate `aud` matches your registered client ID.
- Verify the signing key via the provider's JWKS endpoint — cache with TTL, not forever.
- If accepting tokens from multiple providers, maintain separate validation rules per provider.

**Sub-claim uniqueness across providers:**
- `sub` (subject) is unique per provider, not globally. Prefix with issuer: `google|123456`, not just `123456`.
- Never merge accounts across providers using only email — link accounts through an explicit, user-consented flow.

---

## Step 8 — Common OAuth Vulnerabilities

| Vulnerability | Check | Fix |
|---|---|---|
| CSRF on callback | Is `state` validated on callback? | Generate, store, and validate `state` |
| Authorization code interception | Is PKCE used for public clients? | Add PKCE with S256 |
| Open redirect on login | Can an attacker control `redirect_uri`? | Strict allowlist validation |
| Token leakage in URL | Are tokens passed as query params? | Use POST body or Authorization header |
| Implicit flow usage | Is `response_type=token` used? | Migrate to Authorization Code + PKCE |
| Missing `aud` validation | Are ID tokens validated for audience? | Add audience claim validation |
| Mix-up attack | Multiple providers accepted without binding | Bind the `state` to the intended provider |

---

## Step 9 — Output Report

```
## OAuth Security Review: <integration/provider>

### Critical
- No `state` parameter validation on callback → CSRF vulnerability
  Fix: generate random state, store in session, validate on callback

### High
- Implicit flow in use (response_type=token) — tokens exposed in URL/browser history
  Fix: migrate to Authorization Code + PKCE

- Access tokens stored in localStorage → XSS accessible
  Fix: store access tokens in memory; use HttpOnly cookie for refresh tokens

### Medium
- Requesting `user:read:all` scope when only email is needed
  Fix: downscope to `email` + `profile`

### Low
- JWKS cached indefinitely — no TTL set
  Fix: cache JWKS with 24-hour TTL; allow refresh on unknown `kid`

### Passed
- Redirect URI registered exactly (no wildcards) ✓
- `iss` and `aud` validated on ID tokens ✓
- Refresh tokens revoked on logout ✓
```

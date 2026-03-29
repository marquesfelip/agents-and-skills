---
name: jwt-security
description: 'Secure JWT usage, signing algorithms, expiration handling, rotation, validation, and common JWT attack prevention. Use when: JWT security, JSON web token, JWT validation, JWT signing algorithm, RS256 vs HS256, alg none attack, JWT expiration, refresh token rotation, JWT revocation, token claims validation, JWT attack, bearer token security, JWT best practices, token hijacking.'
argument-hint: 'JWT issuing/validation code, middleware, or auth config to review — or describe the token strategy being designed'
---

# JWT Security Specialist

## When to Use
- Reviewing JWT issuance or validation code for security flaws
- Choosing a signing algorithm and key management strategy
- Implementing access + refresh token rotation
- Adding JWT revocation to a stateless system
- Preventing common JWT attacks (`alg: none`, algorithm confusion, weak secrets)

---

## Step 1 — Understand the Token Context

Before advising, identify:
- Who issues tokens (your service, external IdP, federated)
- What is being protected (user sessions, service-to-service, third-party delegated access)
- Token lifetime requirements (short-lived access vs long-lived refresh)
- Whether revocation is needed (logout, account suspension)
- Key management: symmetric (shared secret) vs asymmetric (key pair)

---

## Step 2 — Algorithm Selection

| Algorithm | Type | Use when | Never use when |
|---|---|---|---|
| RS256 | Asymmetric (RSA) | Multiple validators, public key needed | Constrained environments with small key support |
| ES256 | Asymmetric (ECDSA) | Performance-sensitive; smaller keys | Legacy systems without EC support |
| HS256 | Symmetric (HMAC) | Single issuer + single validator with shared secret | Multiple validators or externally shared tokens |
| `none` | None | **Never** | **Always** — reject tokens with `alg: none` |
| RS384/ES384/RS512/ES512 | Stronger variants | High-assurance contexts | |

**Key rules:**
- Explicitly validate the `alg` header against your expected algorithm — never trust the token's `alg` claim.
- Reject any token with `alg: none` unconditionally.
- For HS256: use a secret ≥256 bits generated with CSPRNG; never use a password, environment variable name, or words.
- For RS256/ES256: key size ≥2048-bit RSA or P-256 ECDSA minimum.

---

## Step 3 — Required Claims Validation

**Every JWT validation must check ALL of the following:**

| Claim | Validation |
|---|---|
| `alg` | Matches expected algorithm (server-configured, not token-derived) |
| `exp` (expires at) | Must be present and in the future |
| `nbf` (not before) | If present, must not be in the future |
| `iat` (issued at) | Sanity check: not far in the past or future |
| `iss` (issuer) | Must match your expected issuer exactly |
| `aud` (audience) | Must match your service identifier |
| `sub` (subject) | Must be a valid, existing user ID in your system |
| Custom claims | Validate domain-specific claims (roles, tenant, scopes) |

A validation that skips any of these claims is incomplete.

---

## Step 4 — Access + Refresh Token Strategy

```
Access Token:  short-lived (5–15 minutes), stateless, used for API calls
Refresh Token: long-lived (1–30 days), stored securely, used only to get new access tokens
```

**Refresh token rotation (recommended):**
1. Issue a new access token AND a new refresh token on every refresh operation.
2. Invalidate the old refresh token immediately.
3. If the old refresh token is used again (reuse detection): **invalidate the entire token family** — it indicates theft.

```
# Token family invalidation on reuse detection:
# 1. Mark entire session/family as compromised
# 2. Revoke all refresh tokens for the user
# 3. Force re-authentication
# 4. Alert user of potential account compromise
```

**Token storage:**
| Client type | Access token | Refresh token |
|---|---|---|
| Web SPA | Memory (JS variable) | HttpOnly Secure cookie |
| Mobile app | Secure keychain/keystore | Secure keychain/keystore |
| Server-to-server | Environment var / secrets manager | Not needed (short-lived access only) |

**Never** store tokens in `localStorage` or `sessionStorage` — XSS accessible.

---

## Step 5 — Revocation Strategies

JWTs are stateless by design — revocation requires one of:

| Strategy | How | Trade-off |
|---|---|---|
| Short expiry | Keep access tokens ≤15 min | No revocation needed; stale access for up to TTL |
| Revocation list (denylist) | Store revoked JTI (`jti` claim) in Redis; check on each request | Adds latency; storage grows with revocations |
| Refresh token invalidation | Revoke refresh tokens; access tokens expire naturally | Revocation takes effect after current access token expires |
| Opaque token + introspection | Replace JWTs with opaque tokens; validate via introspection endpoint | Fully revocable but adds latency |

For logout and account suspension: combine short access token TTL (≤15 min) + refresh token revocation.

---

## Step 6 — Common JWT Attack Prevention

| Attack | Description | Prevention |
|---|---|---|
| `alg: none` | Attacker strips signature by setting `alg: none` | Explicitly reject `none`; validate `alg` server-side |
| Algorithm confusion (RS → HS) | Attacker uses public key as HS256 secret | Pin expected algorithm per endpoint; never infer from token |
| Weak secret | HS256 secret brute-forced | Use ≥256-bit CSPRNG secret; never use a dictionary word |
| JWT bomb | Oversized JWT causes parser DoS | Set max token length limit (e.g., 8KB) before parsing |
| `kid` injection | Attacker injects `kid` pointing to attacker-controlled key | Validate `kid` against an allowlist of known key IDs |
| Claim confusion | Library uses wrong claim names | Use a well-maintained JWT library; never parse manually |
| Stolen token replay | Token stolen via XSS or network | Store in HttpOnly cookie; enforce HTTPS; use DPoP binding for high-value APIs |

---

## Step 7 — Key Management

- Rotate signing keys on a regular schedule (e.g., every 90 days) and on suspected compromise.
- Publish public keys via JWKS endpoint (`/.well-known/jwks.json`) for asymmetric schemes.
- Support key overlap during rotation: sign with new key, accept old key during transition window.
- Store private keys in a secret manager or HSM — never in source code or version control.
- Use separate key pairs per environment (dev, staging, prod).

---

## Step 8 — Output Report

```
## JWT Security Review: <service/file>

### Critical
- Algorithm derived from token header — `alg: none` attack possible
  Fix: hardcode expected algorithm in validator config; reject tokens with `alg: none`

### High
- Access tokens have 24-hour expiry — compromise window is too large
  Fix: reduce to 15 minutes; implement refresh token rotation

- HS256 secret is 12 characters from config file
  Fix: generate 256-bit CSPRNG secret; store in secrets manager

### Medium
- `aud` claim not validated — tokens from other services accepted
  Fix: add audience validation matching this service's identifier

### Low
- No `jti` claim set — replay attacks harder to detect
  Fix: add unique `jti` to each token; store in revocation check if needed

### Passed
- `exp` validated on every request ✓
- `iss` validated against known issuer ✓
- Refresh tokens are single-use with family invalidation ✓
```

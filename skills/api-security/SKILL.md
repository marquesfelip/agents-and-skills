---
name: api-security
description: 'Secure API design, input validation, authentication enforcement, authorization checks, secure headers, error handling, and API attack surface reduction. Use when: api security review, secure api design, input validation, api authentication, api authorization, secure headers, CORS policy, rate limiting, api attack surface, OWASP API security, REST security, GraphQL security, API hardening, injection prevention.'
argument-hint: 'API file, endpoint, router, or controller to review — or describe the API being designed'
---

# API Security Specialist

## When to Use
- Reviewing an existing REST, GraphQL, or gRPC API for security vulnerabilities
- Designing a new API endpoint with security-first principles
- Auditing input validation, authentication enforcement, or authorization checks
- Configuring secure headers, CORS, rate limiting, and error responses
- Reducing the API attack surface before a release or security audit

---

## Step 1 — Understand the API Surface

Read the target files. Identify:
- Transport protocol (REST/HTTP, GraphQL, gRPC, WebSocket)
- Authentication mechanism (JWT, session, API key, OAuth2)
- Authorization model (RBAC, ABAC, ownership checks)
- Where inputs enter the system (query params, body, headers, path params)
- External integrations (third-party APIs, databases, queues)

---

## Step 2 — Audit Against OWASP API Security Top 10

Check each endpoint against:

| OWASP ID | Risk | What to Check |
|---|---|---|
| API1 | Broken Object Level Authorization | Does every resource access verify the caller owns/can access it? |
| API2 | Broken Authentication | Is every non-public route authenticated? Are tokens validated correctly? |
| API3 | Broken Object Property Level Auth | Does the response expose only fields the caller is allowed to see? |
| API4 | Unrestricted Resource Consumption | Is rate limiting, pagination, and payload size enforced? |
| API5 | Broken Function Level Authorization | Are admin/privileged endpoints protected separately? |
| API6 | Unrestricted Access to Sensitive Business Flows | Are sensitive flows (password reset, checkout) protected against abuse? |
| API7 | Server-Side Request Forgery | Is any URL or IP provided by the user used in server-side requests? |
| API8 | Security Misconfiguration | Are debug routes, CORS, headers, and TLS configured correctly? |
| API9 | Improper Inventory Management | Are old API versions, test endpoints, and docs exposed? |
| API10 | Unsafe Consumption of APIs | Are third-party API responses validated before use? |

---

## Step 3 — Input Validation Checklist

Every input crossing a trust boundary must be validated:

| Input type | Required checks |
|---|---|
| String fields | Max length, allowed characters, no null bytes |
| Numeric fields | Min/max bounds, integer overflow prevention |
| Enum fields | Strict allowlist validation, reject unknown values |
| IDs / UUIDs | Format validation + ownership authorization |
| File uploads | Size limit, MIME type, filename sanitization (see `secure-file-upload`) |
| URLs | Allowlist of allowed schemes/hosts; never pass raw to server-side fetchers |
| Dates | Format parsing + range sanity checks |
| Free text | Sanitize for output context (HTML encoding, SQL parameterization) |

> **Rule**: Never trust client-supplied data. Validate at the boundary — not deeper in the stack.

---

## Step 4 — Authentication & Authorization Review

### Authentication
- Every non-public route must require a valid, verified credential.
- Token validation must check: signature, expiry, issuer, audience.
- Reject requests with missing, malformed, or expired tokens with `401 Unauthorized`.
- Never expose internal user IDs or role details in error messages.

### Authorization
- Check object-level ownership on every resource access (not just at the route level).
- Privileged endpoints (admin, internal, bulk ops) must be gated by role/permission checks.
- Deny by default — return `403 Forbidden` for unauthorized actions.
- Never rely on obscurity (hidden routes, opaque IDs) as a security control.

---

## Step 5 — Secure Headers & Transport

Required HTTP response headers:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=(), microphone=()
```

CORS configuration:
- Never use `Access-Control-Allow-Origin: *` for authenticated APIs.
- Explicitly allowlist trusted origins.
- Do not reflect arbitrary `Origin` headers back.
- Credentials (`withCredentials`) require an explicit non-wildcard `Origin`.

TLS:
- Enforce TLS 1.2 minimum; prefer TLS 1.3.
- Disable weak cipher suites.
- Redirect all HTTP to HTTPS.

---

## Step 6 — Error Handling

| Scenario | Correct response | Never include |
|---|---|---|
| Authentication failure | `401` + generic message | Token details, username existence hints |
| Authorization failure | `403` + generic message | Permission model details, resource structure |
| Not found | `404` or `403` (for sensitive resources) | Whether the resource exists |
| Validation error | `400` + field-level detail | Stack traces, internal paths |
| Server error | `500` + correlation ID | Exception messages, DB errors, stack traces |

Log all errors server-side with full context. Return only safe, minimal information to the client.

---

## Step 7 — Attack Surface Reduction

- Disable or remove unused HTTP methods on each route.
- Remove debug endpoints, admin UIs, and API explorers from production.
- Version your API and decommission old versions on a schedule.
- Implement rate limiting per user/IP on all endpoints; use stricter limits on auth endpoints.
- Apply request size limits (body, upload, query string).
- Use API gateways or WAFs for additional perimeter defense.

---

## Step 8 — Output Report

Produce a structured report with:

```
## API Security Review: <API/File Name>

### Critical
- [API1] GET /users/:id — no ownership check; any authenticated user can read any user's data
  Fix: add `if record.UserID != caller.ID { return 403 }` before returning

### High
- [API8] CORS allows all origins (`*`) on authenticated routes
  Fix: restrict to `["https://app.example.com"]`

### Medium
- [API4] No rate limiting on POST /login
  Fix: apply 5 req/min per IP with exponential backoff on failure

### Low
- [Headers] X-Content-Type-Options missing
  Fix: add middleware to set `nosniff` on all responses

### Passed
- Input validation: all body fields validated with explicit schema
- TLS enforced with redirect
```

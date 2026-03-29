---
name: data-leak-prevention
description: 'Prevention of sensitive data exposure through APIs, logs, storage, and responses. Use when: data leak prevention, sensitive data exposure, API data leak, response over-exposure, PII in response, secret in response, data masking API, field filtering, response scrubbing, sensitive field exposure, OWASP A02, data breach prevention, information disclosure, excessive data exposure.'
argument-hint: 'API response, serializer, DTO, or data access layer to review — or describe the data exposure concern being addressed'
---

# Data Leak Prevention Specialist

## When to Use
- Reviewing API responses for fields that expose more data than the caller needs
- Auditing serializers, DTOs, and ORM models for sensitive field leakage
- Detecting sensitive data in logs, error messages, or debug outputs
- Designing field-level access control and response filtering
- Preventing accidental exposure of credentials, keys, PII, or internal metadata

---

## Step 1 — Classify Sensitive Data

Before reviewing, identify what counts as sensitive in the system:

| Sensitivity tier | Examples |
|---|---|
| Critical | Passwords (hashed or plain), private keys, API secrets, session tokens, MFA seeds |
| High | Full SSNs, credit card numbers, bank account numbers, health records (PHI) |
| High | Government IDs, passport numbers, biometric data |
| Medium | Full email addresses, phone numbers, physical addresses, IP addresses |
| Medium | Date of birth, gender, race, religion, political views (GDPR special categories) |
| Low | First name, last name, publicly visible usernames, aggregated/statistical data |
| Internal | Internal IDs, system paths, stack traces, SQL queries, server versions |

---

## Step 2 — API Response Over-Exposure Audit

The most common data leak vector: returning more fields than the caller needs.

### Pattern: ORM Model Directly Serialized

```python
# BAD — entire DB model returned; includes password_hash, internal_id, etc.
@app.get('/users/{user_id}')
def get_user(user_id: int):
    return db.query(User).filter(User.id == user_id).first()  # leaks password_hash!

# GOOD — explicit response schema (Pydantic / serializer)
class UserPublicResponse(BaseModel):
    id: int
    display_name: str
    avatar_url: str
    # password_hash, email, internal_flags: NEVER included

@app.get('/users/{user_id}', response_model=UserPublicResponse)
def get_user(user_id: int):
    return db.query(User).filter(User.id == user_id).first()
```

### Required Review Points

- [ ] Every API endpoint has an explicit response schema (DTO/serializer) — no raw model serialization
- [ ] Response schema reviewed against principle of minimum necessary disclosure
- [ ] Sensitive fields (`password`, `secret`, `token`, `key`, `pin`, `ssn`, `card_number`) explicitly excluded
- [ ] Internal/system fields (`internal_id`, `_raw`, `debug_info`, stack traces) not present in responses
- [ ] Admin responses (with extra fields) are on separate endpoints, not just a superset

### Caller-Context Field Filtering

Different callers should receive different field sets:

| Caller | Allowed fields |
|---|---|
| Authenticated user (own profile) | All personal fields for their own record |
| Authenticated user (other user) | Public fields only |
| Admin | All fields except credentials |
| Public (unauthenticated) | Minimal public fields |
| Internal service | Service-specific, scoped subset |

---

## Step 3 — Error Message Data Leakage

Error responses are a major unintentional leak channel:

| Leaked data | Example bad response | Safe response |
|---|---|---|
| Stack trace | `NullPointerException at com.app.db.UserRepo:43` | `An error occurred. Reference: ERR-4821` |
| SQL query | `ERROR: relation "users" does not exist — query: SELECT * FROM users WHERE...` | Generic DB error |
| Internal path | `File not found: /home/app/config/secrets.yml` | `Resource not found` |
| Server version | `Server: Apache/2.4.41 (Ubuntu)` | Remove `Server` header |
| Framework | `X-Powered-By: Express 4.18.2` | Remove `X-Powered-By` header |
| Dependency versions | Debug page listing all dependencies | Disable debug pages in production |
| Username existence | `No account found for that email` | `If an account exists, a reset was sent` |

**Always:**
- Return opaque error IDs (`ERR-{uuid}`) to clients; log full detail server-side.
- Strip `Server`, `X-Powered-By`, `X-AspNet-Version`, and similar informational headers.
- Disable debug/verbose error mode in all production environments.

---

## Step 4 — Sensitive Data in Logs

Review all logging call sites for sensitive field inclusion:

```python
# BAD — credentials and PII in logs
logger.info(f"User login: email={email}, password={password}, token={auth_token}")
logger.debug(f"Request body: {request.body}")   # may contain card numbers, passwords

# GOOD — log only safe identifiers
logger.info(f"User login: user_id={user_id}, ip={client_ip}, success={success}")
```

Log field audit — **never log**:
- Passwords (plain or hashed)
- Session tokens, JWTs, API keys, OAuth tokens
- Credit card numbers (full or partial beyond last 4)
- SSNs, passport numbers, full government IDs
- Full request bodies on auth/payment endpoints
- Private encryption keys or secrets
- MFA codes or backup codes

---

## Step 5 — Storage-Level Exposure

### Database

- [ ] Passwords stored as hashes — never plaintext or reversibly encrypted
- [ ] Card numbers: store only last 4 digits + tokenized reference (Stripe token, vault token)
- [ ] SSNs: store encrypted at field level; decrypt only in trusted service
- [ ] Health data: encrypted at rest; separate schema with strict access control
- [ ] Sensitive columns not included in wildcard `SELECT *` queries in code; use explicit column lists
- [ ] DB users granted minimum required columns — no access to `password_hash` from reporting role

### Backups

- [ ] Backups encrypted with same or stronger protection as production
- [ ] Backup storage access restricted — not same credentials as application
- [ ] Backup content tested to verify sensitive fields are encrypted (not just at container level)

### File Storage

- [ ] Files containing sensitive data stored in private buckets — no public-read ACL
- [ ] No sensitive data in filenames or path components
- [ ] Temporary files containing sensitive data cleaned up immediately after use
- [ ] Object storage access logs enabled to detect unauthorized access

---

## Step 6 — Transit Exposure

- [ ] All sensitive data transferred over TLS only — no HTTP fallback for data endpoints
- [ ] Sensitive fields not passed as URL query parameters (logged by proxies, CDN, browser history)
- [ ] Sensitive data not included in HTTP referrer headers (set `Referrer-Policy: no-referrer` on sensitive pages)
- [ ] API responses with sensitive fields set `Cache-Control: no-store`
- [ ] Webhooks delivering sensitive data use HTTPS endpoints only; verify recipient TLS certificate

---

## Step 7 — Third-Party and Integration Exposure

- [ ] Sensitive data minimized in payloads sent to third parties (analytics, logging, CRM)
- [ ] PII scrubbed or hashed before sending to analytics platforms (GA, Segment, Mixpanel)
- [ ] Error reporting tools (Sentry, Datadog) configured to scrub sensitive fields from payloads
- [ ] Third-party API responses with sensitive data not stored or logged unnecessarily

---

## Step 8 — Output Report

```
## Data Leak Prevention Review: <service/API>

### Critical
- GET /api/users/:id returns full ORM model including password_hash field
  Any authenticated user can retrieve another user's hashed password
  Fix: add UserPublicResponse DTO; exclude password_hash, internal_flags, reset_token

- POST /auth/login logs full request body including plaintext password field
  Passwords appear in application logs and any log aggregation system
  Fix: never log request bodies on auth endpoints; log user_id + result only

### High
- HTTP 500 responses include full stack trace in JSON body
  Reveals: file paths, class names, dependency versions, DB schema hints
  Fix: return {"error": "internal_error", "ref": "<error_id>"}; log full detail server-side

- GET /api/orders returns full card_number field from payment records
  PCI-DSS violation; card numbers should never be returned after initial capture
  Fix: return last 4 digits only; use payment provider token for reference

### Medium
- Credit card last 4 + expiry returned in order list endpoint unnecessarily
  Not needed for order listing; reduces exposure scope
  Fix: omit from list endpoint; include only on detail endpoint when required

- Sentry error reporting sends full request body including form fields
  PII (email, name, phone) appears in error tracking system
  Fix: configure Sentry beforeSend to scrub sensitive fields

### Low
- Server header returns "nginx/1.21.0" — version fingerprinting possible
  Fix: set server_tokens off in nginx config

### Passed
- Explicit Pydantic response models on all endpoints ✓
- Backup files encrypted with AES-256-GCM ✓
- No sensitive fields in URL query parameters ✓
```

---
name: secure-logging
description: 'Safe logging practices preventing credential or sensitive data exposure in log outputs. Use when: secure logging, logging security, sensitive data in logs, credentials in logs, PII in logs, log injection, log scrubbing, log redaction, OWASP logging, structured logging security, secrets in logs, password in logs, token in logs, log sanitization, audit logging, log security review.'
argument-hint: 'Logger call sites, log configuration, or the feature/service whose logging needs security review'
---

# Secure Logging Specialist

## When to Use
- Reviewing application logging code for sensitive data exposure
- Designing a logging strategy that is safe for production and compliant
- Auditing log shipping configuration (ELK, Datadog, Splunk, Loki) for data exposure
- Implementing structured logging with field-level scrubbing
- Preparing logging for GDPR / LGPD compliance — PII must not flow into log systems without legal basis

---

## Step 1 — Define Sensitive Field Taxonomy for Logging

Before reviewing code, establish what must **never** appear in logs:

| Category | Examples | Verdict |
|---|---|---|
| Credentials | Passwords, PINs, secrets, private keys, certificates | Never log — not even hashed |
| Auth tokens | JWTs, session tokens, OAuth tokens, API keys, bearer tokens | Never log |
| PII (Direct) | Full name, email, phone, national ID (CPF/SSN), passport number | Never log; log opaque ID (user_id) instead |
| Payment data | Card numbers, CVV, IBAN, account numbers | Never log |
| MFA/OTP | TOTP codes, backup codes, SMS OTP | Never log |
| Cryptographic material | Encryption keys, HMAC secrets, signing keys | Never log |
| Request bodies (auth/payment) | POST body for login, payment, password reset | Never log full body |
| PII (Quasi) | IP addresses, device fingerprints, user agents | Log only with justified purpose; pseudonymize for analytics |
| Internal system detail (verbose) | Stack traces, SQL query text, internal paths | Log server-side errors internally; never send to client |

---

## Step 2 — Audit Existing Log Call Sites

### Common Patterns to Find and Fix

#### Pattern 1: Logging full request/response

```python
# BAD — logs full HTTP body including passwords, tokens, card numbers
logger.debug(f"Request received: {request.body}")
logger.debug(f"Response sent: {response.json()}")

# GOOD — log only safe metadata
logger.info("Request received", extra={
    "method": request.method,
    "path": request.path,
    "user_id": current_user.id,
    "content_length": request.content_length,
    "request_id": request.id,
})
```

#### Pattern 2: Logging function arguments directly

```python
# BAD — password appears in log if login() args are logged
logger.debug(f"Calling login with args: email={email}, password={password}")

# GOOD — log only what's safe
logger.info("Login attempt", extra={"email_hash": hash_email(email)})
```

#### Pattern 3: Exception logging leaks request state

```python
# BAD — exception serializes all local variables including form_data containing password
try:
    process_payment(card_number, cvv, amount)
except Exception:
    logger.exception("Payment failed", extra={"locals": locals()})  # card_number exposed!

# GOOD — log safe context explicitly
try:
    process_payment(card_number, cvv, amount)
except Exception:
    logger.error("Payment failed", extra={
        "user_id": user_id,
        "amount": amount,
        "currency": currency,
        # card_number and cvv: NEVER included
    })
```

#### Pattern 4: Log injection

An attacker can inject fake log lines if user input is logged unsanitized:

```python
# Attacker input: "admin\nINFO 2024-01-01 admin logged in successfully"
# BAD — malicious input creates fake log entries
logger.info(f"User search query: {user_input}")

# GOOD — structured logging prevents injection; encode control characters
logger.info("User search performed", extra={"query": sanitize_log_field(user_input)})

def sanitize_log_field(value: str) -> str:
    # Remove newlines, carriage returns, and null bytes from user-controlled strings
    return value.replace('\n', ' ').replace('\r', ' ').replace('\x00', '')
```

---

## Step 3 — Implement Structured Logging with Field Scrubbing

Structured logging (JSON logs) enables automatic scrubbing at the logger level:

```python
import logging
import json

SENSITIVE_FIELDS = frozenset([
    'password', 'passwd', 'secret', 'token', 'api_key', 'apikey',
    'authorization', 'auth', 'credential', 'private_key',
    'card_number', 'cvv', 'ssn', 'cpf', 'pin',
    'access_token', 'refresh_token', 'id_token',
])

class SensitiveDataFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        if hasattr(record, '__dict__'):
            self._scrub(record.__dict__)
        return True

    def _scrub(self, data: dict) -> None:
        for key in list(data.keys()):
            if any(sensitive in key.lower() for sensitive in SENSITIVE_FIELDS):
                data[key] = '[REDACTED]'
            elif isinstance(data[key], dict):
                self._scrub(data[key])

# Apply filter globally
logger = logging.getLogger()
logger.addFilter(SensitiveDataFilter())
```

**Go example:**

```go
// Zap logger with field scrubbing
func scrubFields(fields []zap.Field) []zap.Field {
    sensitiveKeys := map[string]bool{
        "password": true, "token": true, "secret": true,
        "api_key": true, "authorization": true,
    }
    for i, f := range fields {
        if sensitiveKeys[strings.ToLower(f.Key)] {
            fields[i] = zap.String(f.Key, "[REDACTED]")
        }
    }
    return fields
}
```

---

## Step 4 — Log Level Discipline

| Level | Purpose | What to include | What NOT to include |
|---|---|---|---|
| ERROR | Operational failures requiring attention | Error message, error code, user_id, request_id, timestamp | Stack trace (server-side only), no PII, no credentials |
| WARN | Degraded behavior, unexpected conditions | Same as ERROR scope | Same restrictions |
| INFO | Normal significant events | Event name, entity ID, user_id, duration | No PII fields, no full objects |
| DEBUG | Detailed flow tracing (disabled in prod) | Method calls, decision branches | NO credentials, tokens, or PII even in debug |

**Critical rule:** DEBUG level must be **disabled in production**. Never log credentials even at DEBUG.

```python
# Configuration guard
if os.environ.get('ENVIRONMENT') == 'production':
    logging.basicConfig(level=logging.INFO)
else:
    logging.basicConfig(level=logging.DEBUG)
```

---

## Step 5 — Third-Party Log Shipping Security

When shipping logs to external systems (Datadog, Splunk, ELK, Loki):

### Pre-Shipping Scrubbing

Configure the log shipper (Fluentd, Logstash, Vector, Fluent Bit) to scrub sensitive fields before they leave the network:

```yaml
# Vector — redact sensitive fields before forwarding
[transforms.scrub_sensitive]
type = "remap"
inputs = ["app_logs"]
source = '''
  if exists(.password) { .password = "[REDACTED]" }
  if exists(.token) { .token = "[REDACTED]" }
  if exists(.authorization) { .authorization = "[REDACTED]" }
'''
```

```yaml
# Logstash — mutate to remove sensitive fields
filter {
  mutate {
    remove_field => ["password", "token", "secret", "authorization"]
  }
}
```

### Checklist — Log Shipping

- [ ] Log aggregation system access is role-restricted — not all engineers can query all logs
- [ ] Log retention policy defined and enforced (typically 30–90 days for application logs)
- [ ] Logs encrypted in transit (TLS) and at rest in the aggregation system
- [ ] PII in logs triggers GDPR/LGPD data processing agreement with log vendor
- [ ] Production logs are in a separate index/project from dev/staging logs

---

## Step 6 — Error Tracking Tool Configuration

Error tracking tools (Sentry, Bugsnag, Rollbar) automatically capture request context — must be scrubbed:

```python
# Sentry — scrub sensitive fields before sending
import sentry_sdk

def before_send(event, hint):
    # Scrub request body
    if 'request' in event and 'data' in event['request']:
        data = event['request']['data']
        for field in ['password', 'token', 'card_number', 'cvv']:
            if field in data:
                data[field] = '[Filtered]'
    # Scrub headers
    if 'request' in event and 'headers' in event['request']:
        headers = event['request']['headers']
        for header in ['Authorization', 'Cookie', 'X-API-Key']:
            if header in headers:
                headers[header] = '[Filtered]'
    return event

sentry_sdk.init(dsn="...", before_send=before_send)
```

---

## Step 7 — Log Integrity and Tamper Detection

For audit and compliance purposes, logs must demonstrate integrity:

- [ ] Logs written to append-only storage — no update or delete permissions for application role
- [ ] Write-once storage or WORM-compliant log destination (S3 Object Lock, immutable log buckets)
- [ ] Centralized log server receives logs over authenticated TLS — not writable by compromised application
- [ ] Log integrity hashing: each log entry hashed; rolling hash chain for tamper detection
- [ ] Clock synchronization (NTP) across all log-producing services for correlation accuracy

---

## Step 8 — Compliance and Retention

| Log type | Recommended retention | Regulatory driver |
|---|---|---|
| Authentication events (login, logout, MFA) | 1–2 years | SOC 2, ISO 27001 |
| Authorization failures (403s) | 1 year | SOC 2 |
| Data access logs (who accessed what PII) | 3–5 years | LGPD, GDPR, HIPAA |
| Administrative actions | 3–5 years | SOC 2, compliance |
| Application errors | 30–90 days | Operational |
| Debug logs | 7–14 days | Operational (never in prod) |

- [ ] Retention periods defined, documented, and enforced via automated deletion/archival
- [ ] PII in logs covered by privacy policy and data processing agreement with log vendor

---

## Output Report

```
## Secure Logging Review: <service>

### Critical
- Login endpoint logs full request body: `{"email": "...", "password": "..."}`
  Passwords appear in plaintext in Datadog and accessible to any engineer with log access
  Fix: never log request body on auth endpoints; log only user_id + login result

- JWT access tokens logged in middleware: `logger.debug(f"Token: {auth_header}")`
  Any engineer with log access can replay sessions using logged tokens
  Fix: log only token_hash (SHA-256 prefix) or `[REDACTED]` — never full tokens

### High
- Sentry error reporting captures full request headers including Authorization
  Bearer tokens forwarded to third-party error tracking system
  Fix: configure Sentry beforeSend to filter Authorization and Cookie headers

- Stack traces returned in HTTP 500 responses expose file paths and class names
  Fix: return opaque error reference; log full trace server-side only

### Medium
- logger.exception() calls include locals() — some locals contain form field data
  Fix: explicitly list safe fields in log extra; never include locals() or args dict

- DEBUG level enabled in staging; staging shares log index with internal team
  Fix: set INFO level in staging; separate log index per environment

### Low
- User search queries logged without control character sanitization
  Log injection possible via crafted newlines in search input
  Fix: apply sanitize_log_field() before logging user-provided strings

### Passed
- Structured JSON logging configured globally ✓
- SensitiveDataFilter applied to root logger ✓
- Log shipping over authenticated TLS to Datadog ✓
```

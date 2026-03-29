---
name: owasp-top-10-defense
description: 'Enforcing protections aligned with OWASP Top 10 vulnerabilities and secure coding practices. Use when: OWASP Top 10, secure coding review, vulnerability checklist, A01 broken access control, A02 cryptographic failures, A03 injection, A04 insecure design, A05 security misconfiguration, A06 vulnerable components, A07 auth failures, A08 integrity failures, A09 logging failures, A10 SSRF, security audit, web application security, full security review.'
argument-hint: 'Application, service, file, or PR to audit against OWASP Top 10 — or describe the system being designed'
---

# OWASP Top 10 Defense Specialist

## When to Use
- Performing a full security audit of a web application or API
- Reviewing a PR or feature for any of the OWASP Top 10 vulnerability classes
- Validating secure coding practices before a release or penetration test
- Creating a security checklist for a new application
- Reporting security findings with OWASP-standard severity classification

---

## Step 1 — Scope the Audit

Identify what is in scope:
- Application type: web, API, SPA, mobile backend, microservice
- Entry points: HTTP endpoints, file uploads, WebSockets, message queues, scheduled jobs
- Data sensitivity: PII, financial data, health records, authentication credentials
- External integrations: third-party APIs, databases, queues, cloud services

---

## Step 2 — Audit Each OWASP Category

Work through each category systematically. For each, identify findings and assign severity.

---

### A01 — Broken Access Control

> Most common OWASP finding. Every access decision deserves scrutiny.

- [ ] Object-level authorization: does each resource access verify caller ownership?
- [ ] Function-level authorization: are admin/privileged endpoints separately gated?
- [ ] Horizontal privilege escalation: can user A access user B's data by changing an ID?
- [ ] Vertical privilege escalation: can a low-privilege user call admin functions?
- [ ] Tenant isolation: does every query include a `tenant_id` guard?
- [ ] Directory traversal: are server paths derived from user input?
- [ ] CORS: are origins explicitly allowlisted rather than reflected?

→ See `authorization-models` skill for in-depth RBAC/ABAC review.

---

### A02 — Cryptographic Failures

- [ ] Sensitive data transmitted without TLS (plaintext HTTP)?
- [ ] Weak or deprecated algorithms: MD5, SHA-1, DES, RC4, AES-ECB?
- [ ] Passwords hashed with non-purpose-built algorithms (SHA-256, MD5)?
- [ ] Hardcoded keys, weak keys, or keys co-located with encrypted data?
- [ ] Sensitive data stored in logs, URLs, or browser history?
- [ ] Certificates expired, self-signed, or with weak key sizes?

→ See `encryption-best-practices` and `password-storage-security` skills.

---

### A03 — Injection

- [ ] SQL injection: are all queries parameterized or using a safe ORM?
- [ ] Command injection: is `exec()` / `shell_exec()` called with user input?
- [ ] LDAP/XPath/NoSQL injection: are queries built with string concatenation?
- [ ] Template injection: is user input rendered inside a server-side template?
- [ ] XML/XXE injection: is external entity processing disabled in XML parsers?

→ See `sql-injection-prevention` and `rce-prevention` skills.

---

### A04 — Insecure Design

- [ ] Are security requirements documented before implementation?
- [ ] Are threat models available for sensitive flows (auth, payment, data export)?
- [ ] Are rate limits and abuse controls designed into sensitive flows?
- [ ] Is multi-factor authentication available for privileged actions?
- [ ] Are security invariants (e.g., "user can never read another user's data") testable?

---

### A05 — Security Misconfiguration

- [ ] Default credentials removed on all services and infrastructure?
- [ ] Debug mode / verbose error messages disabled in production?
- [ ] Unnecessary HTTP methods disabled (e.g., `TRACE`, `OPTIONS` where unused)?
- [ ] Dependency versions pinned; unused dependencies removed?
- [ ] Secure HTTP headers present (CSP, HSTS, X-Frame-Options, etc.)?
- [ ] Cloud storage buckets private by default?
- [ ] Admin panels and management UIs not publicly accessible?

---

### A06 — Vulnerable and Outdated Components

- [ ] Dependencies scanned for known CVEs (npm audit, `go vuln`, Dependabot, Snyk)?
- [ ] No abandoned / unmaintained packages in use?
- [ ] Base Docker images updated regularly; no packages with critical CVEs?
- [ ] Dependency updates automated (Dependabot, Renovate)?

---

### A07 — Identification and Authentication Failures

- [ ] Brute force protection on login and password reset endpoints?
- [ ] Passwords hashed with Argon2id or bcrypt (cost ≥12)?
- [ ] Session IDs generated with CSPRNG; rotated on login?
- [ ] MFA available for admin and sensitive accounts?
- [ ] Default credentials changed; no test accounts in production?

→ See `authentication-patterns` and `brute-force-protection` skills.

---

### A08 — Software and Data Integrity Failures

- [ ] CI/CD pipelines protected from unauthorized modification?
- [ ] Dependency integrity verified (lock files, checksums, signed packages)?
- [ ] Deserialization of untrusted data avoided or strictly validated?
- [ ] Unsigned updates or auto-update mechanisms without integrity checks?

→ See `deserialization-security` skill.

---

### A09 — Security Logging and Monitoring Failures

- [ ] Authentication events (success, failure, lockout) logged with timestamp + IP?
- [ ] Authorization failures logged?
- [ ] Logs stored centrally; tamper-evident?
- [ ] Sensitive data (passwords, tokens, PII) never logged?
- [ ] Alerts configured for critical events (mass data access, repeated auth failures)?
- [ ] Log retention policy defined and enforced?

---

### A10 — Server-Side Request Forgery (SSRF)

- [ ] Server-side requests made with user-supplied URLs?
- [ ] Allowlist of permitted target hosts/schemes enforced?
- [ ] Internal network addresses (169.254.x.x, 10.x, 172.16.x, 192.168.x, localhost) blocked?
- [ ] Cloud metadata endpoints (169.254.169.254) blocked at application layer?
- [ ] DNS rebinding protection in place?

→ See `ssrf-protection` skill.

---

## Step 3 — Severity Classification

| Severity | Criteria | Example |
|---|---|---|
| Critical | Directly exploitable; data breach or full system compromise possible | SQLi on unauthenticated endpoint |
| High | Significant impact; exploitable with low effort | Missing ownership check, no rate limit on auth |
| Medium | Non-trivial exploitation; partial impact | Verbose error messages, missing security header |
| Low | Defense-in-depth; minor exposure | Cookie missing SameSite, old TLS version accepted |
| Informational | Best practice gap; no direct risk | Dependency with no CVE but unmaintained |

---

## Step 4 — Output Report

```
## OWASP Top 10 Security Audit: <Application/Service>

### Critical
- [A03 Injection] GET /search?q= concatenated into SQL query without parameterization
  Fix: use prepared statements / parameterized queries

### High
- [A01 Access Control] GET /api/invoices/:id — no ownership check
  Fix: verify invoice.user_id == session.user_id on every fetch

- [A07 Auth Failures] No rate limiting on POST /login
  Fix: 5 attempts/15min per account; progressive delay

### Medium
- [A05 Misconfiguration] Debug stack traces returned in HTTP 500 responses
  Fix: configure generic error handler for production

### Low
- [A09 Logging] Auth failures not logged
  Fix: log IP, timestamp, and username (not password) on failed login

### Informational
- [A06 Components] 3 indirect dependencies with no security advisories but no activity in 3+ years
  Fix: evaluate replacement or fork

### Passed
- A02: AES-256-GCM encryption at rest; TLS 1.3 enforced ✓
- A08: Lock files committed; Dependabot enabled ✓
- A10: No user-supplied URLs used in server-side requests ✓
```

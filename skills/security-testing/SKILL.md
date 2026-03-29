---
name: security-testing
description: >
  Automated security validation and vulnerability testing strategies for applications.
  Use when: security testing, automated security testing, DAST, SAST, SCA, vulnerability scanning,
  OWASP ZAP, Burp Suite, Semgrep, Snyk, Trivy, dependency scanning, secret scanning, OWASP Top 10 testing,
  penetration testing, pen test, fuzzing, API security test, authentication test, authorization test,
  injection test, XSS test, CSRF test, SSRF test, security regression, security CI gate,
  shift left security, DevSecOps, security pipeline, security scan, supply chain security,
  container scanning, license compliance, security audit, security QA.
argument-hint: >
  Describe the system and security concern (e.g., "REST API with JWT authentication and PostgreSQL backend"),
  and specify the testing type (SAST/DAST/pen test), framework, or vulnerability category to focus on.
---

# Security Testing Specialist

## When to Use

Invoke this skill when you need to:
- Design or review an automated security testing pipeline (DevSecOps)
- Select and configure SAST, DAST, and SCA tools for a project
- Write security-focused test cases for authentication, authorization, and input handling
- Integrate security gates into CI/CD that block on critical findings
- Validate that OWASP Top 10 risks are covered by the test suite
- Prepare for a penetration test or security audit

---

## Step 1 — Security Testing Taxonomy

| Layer | Type | What It Finds | When to Run |
|---|---|---|---|
| **SAST** | Static Analysis | Code-level vulnerabilities (injection, hardcoded secrets, unsafe deserialization) | Every commit |
| **SCA** | Dependency Scan | Known CVEs in third-party libraries | Every commit |
| **Secret Scan** | Secret Detection | Credentials, API keys, tokens in source code or history | Every commit |
| **DAST** | Dynamic Analysis | Runtime vulnerabilities (XSS, SQLi, auth bypass) | On every build against a live environment |
| **IAST** | Interactive | Runtime instrumentation during functional tests | During integration / E2E test run |
| **Container Scan** | Image Scan | Vulnerable base images, OS packages | Every Docker build |
| **Fuzzing** | Input Fuzzing | Unexpected crash paths, edge-case injection | Nightly / scheduled |
| **Pen Test** | Manual | Complex logic flaws, chained vulnerabilities | Quarterly / pre-release |

Checklist:
- [ ] Each layer covered by at least one tool
- [ ] SAST and SCA run on every commit with fast feedback (< 5 min)
- [ ] DAST runs against a deployed test environment — never directly against production
- [ ] Container images scanned before being pushed to registry

---

## Step 2 — SAST — Static Analysis

Recommended tools:

| Tool | Languages | Strengths |
|---|---|---|
| **Semgrep** | Multi-language | Custom rules, fast, OSS |
| **CodeQL** | Multi-language | Deep data-flow analysis, GitHub native |
| **SonarQube** | Multi-language | Broad coverage, quality gates |
| **Gosec** | Go | Go-specific security rules |
| **Bandit** | Python | Python security linting |
| **ESLint Security** | JavaScript | JS/TS security patterns |
| **Brakeman** | Ruby on Rails | Rails-specific vulnerabilities |

Checklist:
- [ ] SAST tool integrated into CI — fails pipeline on `high` / `critical` severity findings
- [ ] Custom rules added for project-specific patterns (e.g., internal unsafe APIs)
- [ ] False positive suppressions documented and reviewed periodically
- [ ] SAST runs on PR diff — not only on full codebase scan

---

## Step 3 — SCA and Secret Scanning

**Dependency Scanning:**
- **Snyk** — multi-language, license compliance, fix PRs
- **Trivy** — OSS, container + filesystem, fast
- **OWASP Dependency-Check** — Java/Maven/Gradle focus
- **Dependabot / Renovate** — automated dependency update PRs

**Secret Scanning:**
- **git-secrets** — prevents secrets entering git history
- **TruffleHog** — scans git history for real secrets
- **Gitleaks** — fast regex + entropy-based detection
- **GitHub Secret Scanning** — GitHub-native, enterprise-wide

Checklist:
- [ ] Dependency scanner runs on every CI build
- [ ] Critical CVEs block the build; high CVEs create issues for triage
- [ ] Secret scanner runs as a pre-commit hook and in CI
- [ ] `git log` scanned for historical secret leaks on every new contributor onboarding
- [ ] License compliance enforced — GPL/AGPL licenses flagged for legal review
- [ ] `SBOM` (Software Bill of Materials) generated and stored per release

---

## Step 4 — DAST — Dynamic Analysis

DAST tools attack the running application:

**Tools:**
- **OWASP ZAP** — open source, scriptable, API scanning
- **Burp Suite** (Enterprise) — comprehensive, CI integration
- **Nuclei** — template-based, fast, extensible
- **Nikto** — web server configuration checks

**Automated DAST flow:**
1. Deploy application to test environment
2. Run DAST scan against test environment
3. Parse findings by severity
4. Fail CI on critical/high findings; create issues for medium/low

Checklist:
- [ ] DAST runs against a fully deployed test environment with realistic data
- [ ] Authentication configured so DAST can reach authenticated endpoints
- [ ] API spec (OpenAPI/Swagger) provided to DAST tool for comprehensive endpoint coverage
- [ ] DAST not run against production — use staging or dedicated security test env
- [ ] Active scan modes reviewed — some active checks can corrupt test data

---

## Step 5 — Security-Focused Functional Tests

Write explicit test cases for security behaviors:

**Authentication:**
- [ ] Login with invalid credentials returns 401, not 200 with error in body
- [ ] Brute force → lockout after N attempts
- [ ] Expired token returns 401
- [ ] Token from User A cannot authenticate requests for User B

**Authorization:**
- [ ] Authenticated User A cannot access User B's resource (IDOR)
- [ ] Non-admin cannot call admin-only endpoints (returns 403)
- [ ] Privilege escalation via parameter tampering returns 403

**Input Validation:**
- [ ] SQL injection payloads in all input fields → 400, no SQL error in response
- [ ] XSS payloads in text inputs are sanitized in the response
- [ ] Path traversal (`../etc/passwd`) in file parameters → 400
- [ ] Oversized payloads return 413 / 400 — not crash or timeout

**Session / Token:**
- [ ] Logout invalidates the session server-side
- [ ] JWT `alg: none` rejected
- [ ] JWT signed with wrong key rejected

**CSRF / SSRF:**
- [ ] State-changing requests without CSRF token rejected
- [ ] URL parameter that triggers outbound request does not reach internal IPs

---

## Step 6 — Container and Infrastructure Scanning

Checklist:
- [ ] Base images use minimal, maintained images (distroless, Alpine)
- [ ] Container image scanned with **Trivy** or **Grype** before push to registry
- [ ] Critical / high CVEs in OS packages and app dependencies block the push
- [ ] IaC files (Terraform, CloudFormation, Helm) scanned with **Checkov**, **tfsec**, or **KICS**
- [ ] Kubernetes manifests scanned with **Polaris** or **Kubescape** for security misconfigurations
- [ ] No secrets in container environment variables or image layers

---

## Step 7 — CI Security Pipeline

```yaml
security-pipeline:
  stages:
    - name: Secret Scan        # pre-commit + CI
    - name: SAST               # every commit, < 5 min
    - name: SCA / Dependency   # every commit, < 5 min
    - name: Container Scan     # every Docker build
    - name: DAST               # every deploy to test env
    - name: Security Smoke     # critical security functional tests
```

Checklist:
- [ ] Security stages run in parallel where possible to minimize CI time
- [ ] `critical` findings always block merge / deploy
- [ ] `high` findings create tickets and are tracked to resolution within SLA
- [ ] `medium` / `low` findings logged and reviewed in weekly security triage
- [ ] Security findings trended over time — regression tracked in dashboard

---

## Output Report

### Critical
- Known critical CVE in a direct dependency in production
- Hardcoded credentials found in source code or Docker image
- Authentication bypass possible through parameter tampering or JWT misconfiguration
- DAST reveals SQL injection or RCE in a production-facing endpoint

### High
- No DAST coverage — runtime vulnerabilities not detected before production
- IDOR vulnerability confirmed — User A can read/modify User B's resources
- Secret scanner not configured — credentials could be committed undetected

### Medium
- SAST findings suppressed without documented justification
- DAST runs but cannot reach authenticated endpoints — coverage is incomplete
- No security functional tests for authorization boundaries

### Low
- Container images not scanned — OS CVEs accumulate undetected
- License compliance not enforced — GPL dependencies may introduce legal risk
- Security findings not trended — regression invisible over time

### Passed
- SAST, SCA, DAST, and container scanning all integrated in CI
- Critical CVEs block pipeline automatically
- Security-focused functional tests cover OWASP Top 10 attack surfaces
- IDOR, authentication bypass, and privilege escalation tests all pass
- Secret scanner integrated as pre-commit hook and in CI

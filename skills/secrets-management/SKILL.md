---
name: secrets-management
description: 'Secret storage, rotation, environment isolation, vault usage, and prevention of secret exposure in code or logs. Use when: secrets management, secret storage, environment variables security, vault integration, secret rotation, hardcoded secrets, secret scanning, API key management, credential exposure, .env security, secret in code, secret in logs, secret in git, secrets leak prevention, HashiCorp Vault, AWS Secrets Manager, secret lifecycle.'
argument-hint: 'Code file, config, or CI/CD pipeline with secrets handling to review — or describe the secrets architecture being designed'
---

# Secrets Management Specialist

## When to Use
- Reviewing code, configs, or environments for secret exposure
- Designing a secrets management architecture (vault, env vars, CI/CD secrets)
- Implementing secret rotation without downtime
- Auditing for hardcoded credentials, secrets in logs, or secrets committed to git
- Choosing between secrets management tools for a given platform

---

## Step 1 — Understand the Secrets Landscape

Before advising, identify:
- Types of secrets in play: API keys, DB credentials, TLS certs, signing keys, OAuth client secrets, encryption keys
- Deployment environment: local dev, CI/CD, cloud (AWS/GCP/Azure), Kubernetes, on-prem
- Current secrets storage: hardcoded, env vars, `.env` files, vault, secrets manager
- Rotation requirements: manual, automated, zero-downtime

---

## Step 2 — Secret Classification

| Tier | Examples | Rotation frequency | Storage requirement |
|---|---|---|---|
| Critical | DB passwords, signing keys, OAuth secrets, root credentials | 30–90 days or less | Vault / secrets manager only |
| High | API keys for paid services, email credentials, webhook secrets | 90 days | Vault / secrets manager |
| Medium | Internal service API keys, read-only credentials | 180 days | Vault / env injection |
| Low | Non-sensitive config values | Annual or on-demand | Env vars or config files (not secrets at all) |

---

## Step 3 — Secret Storage Rules

### Never Store Secrets In:
| Location | Why |
|---|---|
| Source code (hardcoded) | Committed to version control; visible to all contributors |
| Version control (`.env`, config files) | Even if .gitignored locally, one accident exposes everything |
| Client-side code (JS bundles, mobile apps) | Easily extracted from binaries or browser |
| Logs | Log aggregators, analytics, and log files are rarely secured as credentials |
| URLs (as query params) | Logged by proxies, browsers, and analytics |
| Database as plaintext | DB breach exposes all secrets |
| Container images | Images are distributable; secrets embedded become public |

### Valid Secret Storage Options:

| Storage | Best for | Notes |
|---|---|---|
| HashiCorp Vault | Any platform; self-hosted; fine-grained control | Gold standard; supports dynamic secrets |
| AWS Secrets Manager | AWS workloads | Native IAM integration; automatic rotation |
| GCP Secret Manager | GCP workloads | Native IAM; versioning support |
| Azure Key Vault | Azure workloads | Native RBAC; supports HSM backing |
| Kubernetes Secrets + External Secrets Operator | K8s workloads | Sync from vault to K8s; avoid raw K8s secrets (base64 only) |
| CI/CD platform secrets (GitHub Actions, GitLab CI) | Build-time secrets | Masked in logs; scoped to environments |
| Environment variables (injected at runtime) | Simple deployments | Safe only if injected by a secure orchestrator; never from `.env` files in prod |

---

## Step 4 — Environment Isolation

- Each environment (dev, staging, prod) must have **separate secrets** — never share prod credentials with dev.
- Dev environments should use synthetic/mock credentials or dev-specific secrets with limited access.
- Secrets must be scoped: a service process only receives the secrets it needs (least privilege).
- CI/CD pipelines must not have access to production secrets unless deploying to production.

```
# Environment isolation matrix:
Dev   → dev-db-password, dev-api-key     (separate, limited-scope)
Stage → stage-db-password, stage-api-key (separate from prod)
Prod  → prod-db-password, prod-api-key   (most restricted access)
```

---

## Step 5 — Secret Rotation

### Rotation Strategy

| Secret type | Rotation approach |
|---|---|
| DB passwords | Dual-active rotation: create new credential, update app config, verify, revoke old |
| API keys | Issue new key, update consumers, invalidate old key |
| TLS certificates | Automate with Let's Encrypt / ACME or cert-manager in K8s |
| JWT signing keys | Overlap period: sign with new key, accept both old and new, drop old after TTL |
| OAuth client secrets | Coordinate with IdP; update all consumers atomically |

### Zero-Downtime Rotation Pattern:
1. Generate new secret in vault.
2. Update all consumers to read from vault (not cached).
3. Rotate: vault updates the secret; consumers reload on next request.
4. Monitor for auth failures during transition window.
5. Revoke old secret after confirming zero usage.

---

## Step 6 — Preventing Secret Exposure

### In Code
- Run secret scanning in pre-commit hooks and CI (e.g., `gitleaks`, `truffleHog`, `detect-secrets`).
- Configure `.gitignore` to exclude `.env`, `*.pem`, `*.key`, `config/secrets.*`.
- Use secret detection rules in PR review automation.

### In Logs
- Audit log statements for credential parameters.
- Use a log sanitizer/redactor for known secret patterns (API keys, tokens, passwords).
- Never log request bodies or auth headers at INFO level.
- Configure log aggregators to mask secret-pattern fields.

### In Version Control
If a secret is committed:
1. **Immediately revoke the exposed secret** — assume it is compromised.
2. Remove from git history (`git filter-repo` or BFG Repo Cleaner) — force-push after.
3. Issue a new secret.
4. Notify affected parties if the repo is public.

---

## Step 7 — Vault Integration Patterns

### Direct Vault API
```
# Application retrieves secret at startup:
secret = vault.get("secret/prod/db/password")
db.connect(password=secret)
```

### Sidecar / Agent Injection (K8s)
- Vault Agent or External Secrets Operator writes secrets to a mounted tmpfs volume.
- App reads from filesystem — no vault SDK dependency.
- Secrets rotated without pod restart when using file watches.

### Dynamic Secrets
- Vault generates a short-lived DB credential per service instance.
- Credential valid only for the pod lifespan — breach of one instance doesn't expose global credentials.
- Supported for: PostgreSQL, MySQL, MongoDB, AWS IAM, etc.

---

## Step 8 — Output Report

```
## Secrets Management Review: <project/service>

### Critical
- DB password hardcoded in application.properties committed to git
  Fix: immediately rotate password; remove from history; store in secrets manager

- API key present in frontend JavaScript bundle
  Fix: move API calls server-side; never expose API keys to client

### High
- Dev and prod environments share the same DB password
  Fix: separate credentials per environment with different access scopes

- Secrets not scanned in CI pipeline
  Fix: add gitleaks or truffleHog to PR checks and main branch pipeline

### Medium
- API keys stored in .env file without gitignore coverage for CI runners
  Fix: use CI/CD platform secret injection; remove .env from CI

- No rotation policy or schedule defined for critical credentials
  Fix: document and schedule rotation every 90 days; automate where possible

### Passed
- Vault used for production DB credentials ✓
- Environment variables injected at runtime by orchestrator ✓
- Log redaction in place for Authorization headers ✓
```

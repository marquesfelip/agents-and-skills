---
name: secrets-exposure-detection
description: 'Automated detection of exposed credentials and secrets across codebases, logs, and storage. Use when: secrets exposure detection, credential leak detection, secret scanning, hardcoded secret, secret in code, secret in git, secret in logs, secret in environment dump, exposed API key, leaked credential, secret in source code, gitleaks, trufflehog, semgrep secrets, detect secret, CI secret scan, pre-commit hook secret, secret rotation after leak, credential rotation, exposed token detection, log scrubbing, secret audit.'
argument-hint: 'Describe where you suspect or have confirmed secret exposure (git history, CI logs, application logs, environment file committed, error responses) and which secret types are in scope (API keys, DB credentials, JWT secrets, OAuth tokens, cloud provider keys).'
---

# Secrets Exposure Detection

## When to Use

Invoke this skill when you need to:
- Scan a repository for secrets accidentally committed to git history
- Set up automated pre-commit hooks, CI gates, and scheduled scans
- Respond to a confirmed secret exposure: assess blast radius, rotate affected credentials
- Scrub sensitive values from application logs and error responses
- Audit environment files, Docker images, and CI/CD pipeline configurations for leaked secrets
- Build a secret detection pipeline that covers all exposure surfaces

---

## Threat Model: Where Secrets Leak

| Surface | How it happens | Detection tool |
|---|---|---|
| Git history | `git add .env`, accidental commit of credentials | Gitleaks, TruffleHog, git-secrets |
| Source code | Hardcoded API key, connection string in code | Semgrep, Gitleaks, GitHub Secret Scanning |
| CI/CD logs | `echo $SECRET`, verbose build output prints env vars | Log scrubbing rules, CI log redaction |
| Docker images | Secret in `RUN` layer, `.env` copied into image | Trivy, Grype, Docker `--secret` instead of ARG |
| Error responses | Stack trace includes DB URL with credentials | Structured error handler — never expose raw errors |
| Application logs | Logging full request headers including Authorization | Log scrubbing middleware |
| IaC / config files | Terraform `.tfstate`, Helm values with embedded secrets | Checkov, tfsec, Semgrep |
| Third-party exports | Postman collections exported with auth headers | Manual review + rotation policy |
| Dependency config | `.npmrc`, `.pypirc`, `.m2/settings.xml` with tokens | Gitleaks rules for package manager configs |

---

## Step 1 — Repository Secret Scanning

### Pre-commit Hook (blocks commit before it reaches remote)

```bash
# Install gitleaks
brew install gitleaks   # macOS
# or: go install github.com/gitleaks/gitleaks/v8@latest

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
```

```bash
# Install pre-commit framework
pip install pre-commit
pre-commit install

# Test against staged files
pre-commit run --all-files
```

### CI Pipeline Gate (blocks merge on secret detection)

```yaml
# GitHub Actions: .github/workflows/secret-scan.yml
name: Secret Scanning
on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # full history — scan all commits in PR

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # required for org scans

  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog OSS
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified
```

### Scan Full Git History (incident response)

```bash
# Scan entire git history — use when a leak is suspected or confirmed
gitleaks detect --source=. --log-opts="--all" --report-path=secrets-report.json

# TruffleHog: verified secrets only (reduces false positives)
trufflehog git file://. --since-commit HEAD~100 --only-verified --json > findings.json

# Review findings
cat secrets-report.json | jq '.[] | {file, line, ruleID, secret: .Secret[0:6]+"***"}'
```

---

## Step 2 — Custom Secret Patterns

Define patterns for your own API key formats (see `api-key-management` for key prefix design):

```toml
# .gitleaks.toml — extend default rules with your app's key formats
[extend]
useDefault = true

[[rules]]
id          = "myapp-api-key"
description = "MyApp API Key"
regex       = '''sk_live_[A-Za-z0-9]{40,}'''
keywords    = ["sk_live_"]
tags        = ["api-key", "myapp"]

[[rules]]
id          = "myapp-webhook-secret"
description = "MyApp Webhook Secret"
regex       = '''whsec_[A-Za-z0-9+/=]{40,}'''
keywords    = ["whsec_"]

[[rules]]
id          = "myapp-db-url"
description = "Database connection string with credentials"
regex       = '''(postgres|mysql|mongodb):\/\/[^:]+:[^@]+@'''
tags        = ["database", "credentials"]

# Exclude known safe patterns (test fixtures, documentation examples)
[allowlist]
paths = [
    '''\.md$''',
    '''testdata/''',
    '''docs/examples/''',
]
regexes = [
    '''sk_live_EXAMPLE''',     # documentation placeholder
    '''sk_test_''',            # test keys are expected in test files
]
```

---

## Step 3 — Log Scrubbing

Sensitive values must never appear in application logs.

```go
// Middleware: scrub Authorization header before logging
func LogScrubMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Clone request for logging — never log Authorization, Cookie, X-API-Key
        safeHeaders := make(http.Header)
        for k, v := range r.Header {
            switch strings.ToLower(k) {
            case "authorization", "cookie", "x-api-key", "x-webhook-signature":
                safeHeaders[k] = []string{"[REDACTED]"}
            default:
                safeHeaders[k] = v
            }
        }
        // Log safeHeaders, not r.Header
        next.ServeHTTP(w, r)
    })
}

// Structured logger: never log sensitive fields
type SafeLogger struct {
    inner *slog.Logger
}

var sensitiveKeys = map[string]bool{
    "password": true, "token": true, "secret": true, "api_key": true,
    "authorization": true, "private_key": true, "access_token": true,
    "refresh_token": true, "client_secret": true, "connection_string": true,
}

func (l *SafeLogger) Log(ctx context.Context, level slog.Level, msg string, args ...any) {
    safe := make([]any, 0, len(args))
    for i := 0; i < len(args)-1; i += 2 {
        key, ok := args[i].(string)
        if ok && sensitiveKeys[strings.ToLower(key)] {
            safe = append(safe, key, "[REDACTED]")
        } else {
            safe = append(safe, args[i], args[i+1])
        }
    }
    l.inner.Log(ctx, level, msg, safe...)
}
```

---

## Step 4 — Docker Image Scanning

```bash
# Scan built image for secrets embedded in layers
trivy image --scanners secret myapp:latest

# Use Docker BuildKit secrets — never use ARG for secrets
# WRONG:
# ARG DB_PASSWORD
# RUN ./setup.sh --password="$DB_PASSWORD"

# CORRECT: BuildKit secret mount (not stored in image layer)
# RUN --mount=type=secret,id=db_password ./setup.sh --password="$(cat /run/secrets/db_password)"
# docker build --secret id=db_password,src=./secrets/db_password .

# Verify no .env files are in the image
docker run --rm myapp:latest find / -name ".env" -o -name "*.key" 2>/dev/null
```

---

## Step 5 — Incident Response: Confirmed Exposure

When a secret is confirmed exposed, follow this sequence:

```
1. ROTATE (within minutes — before anything else)
   ├─ Rotate the exposed credential FIRST in the target system
   ├─ For DB password: rotate in DB + update all connection strings + restart services
   ├─ For API key: revoke in provider console immediately
   └─ For JWT secret: rotate + force-invalidate all active sessions

2. ASSESS blast radius
   ├─ When was the secret first committed / deployed?
   ├─ Was the repository public at any time during that period?
   ├─ Which systems does this credential access?
   └─ Are there logs showing the credential was used by an unexpected party?

3. REMOVE from git history (AFTER rotation — don't delay rotation for cleanup)
   git filter-repo --path .env --invert-paths   # remove file from all history
   # or: BFG Repo Cleaner for simple string replacement
   java -jar bfg.jar --replace-text passwords.txt repo.git

   Force-push all branches + force-delete tags:
   git push origin --force --all
   git push origin --force --tags

4. NOTIFY affected parties if credential provided access to customer data

5. PREVENT recurrence via Step 1 (pre-commit hooks + CI gate)
```

---

## Step 6 — Environment and CI/CD Audit

```bash
# Audit GitHub Actions secrets — list all configured secrets (names only, values never exposed by API)
gh secret list --repo owner/repo

# Verify no secrets printed in CI logs (search recent workflow runs)
# Look for patterns in log output:
grep -i "sk_live_\|whsec_\|password\|secret\|token" .github/workflows/*.yml

# Checkov: scan Terraform for hardcoded secrets
pip install checkov
checkov -d ./infra --check CKV_SECRET_*

# Semgrep: scan source for hardcoded credentials
semgrep --config "p/secrets" --config "p/jwt" .
```

---

## Secret Detection Coverage Matrix

| Surface | Tool | When to run |
|---|---|---|
| Git commits (pre-write) | Gitleaks pre-commit hook | Every `git commit` |
| Git PRs | Gitleaks CI + TruffleHog CI | Every PR opened/updated |
| Full git history | TruffleHog `--since-commit` | Weekly scheduled scan |
| Application logs | SafeLogger / log scrubbing | Runtime (continuous) |
| Docker images | Trivy `--scanners secret` | Every image build in CI |
| Terraform / IaC | Checkov, tfsec | Every PR touching infra |
| Source code static | Semgrep `p/secrets` | Every PR |
| GitHub native | GitHub Secret Scanning | Automatic (enable in repo settings) |

---

## Quality Checks

- [ ] Pre-commit hook enforced for all engineers — documented in CONTRIBUTING.md
- [ ] CI gate blocks merge on any Gitleaks or TruffleHog finding
- [ ] Custom rules added for your own API key/secret prefixes (e.g., `sk_live_`, `whsec_`)
- [ ] Application logs scrub Authorization, Cookie, and all known secret field names
- [ ] Error responses never include stack traces, connection strings, or environment dumps
- [ ] Docker `ARG` never used for secrets — BuildKit `--mount=type=secret` used instead
- [ ] `.env` files in `.gitignore` AND `.dockerignore` — verified in both
- [ ] Incident rotation checklist documented and reachable within 5 minutes of detection
- [ ] GitHub Secret Scanning (push protection) enabled in repository settings
- [ ] Weekly scheduled scan of full git history runs and alerts on new findings

## After Completion

Recommended next steps:

- Use **`secrets-management`** for HashiCorp Vault or AWS Secrets Manager integration
- Use **`api-key-management`** to ensure your API key format is scannable by Gitleaks rules
- Use **`secure-logging`** for deep guidance on log redaction and safe structured logging
- Use **`dependency-security-scanning`** to extend the CI security gate to CVEs in dependencies

---
name: dependency-security-scanning
description: >
  Automated dependency vulnerability scanning practices in CI/CD pipelines.
  Use when: dependency security scanning, SCA, software composition analysis, vulnerability scanning,
  CVE detection, Snyk, Dependabot, Renovate, OWASP Dependency-Check, Trivy dependency, npm audit,
  pip-audit, govulncheck, cargo audit, dependency CVE, transitive dependency vulnerability,
  vulnerable library, outdated dependency, license compliance, dependency update automation,
  supply chain security, dependency risk, known vulnerability, NVD, OSV database.
argument-hint: >
  Describe the technology stack and current dependency management approach (e.g., "Go + Node.js
  monorepo with no automated scanning — need to add SCA to CI and automate update PRs"),
  and specify the CI platform and any license or severity thresholds.
---

# Dependency Security Scanning Specialist

## When to Use

Invoke this skill when you need to:
- Add automated dependency vulnerability scanning to a CI pipeline
- Choose and configure an SCA (Software Composition Analysis) tool
- Define severity thresholds that block builds vs. create tickets
- Set up automated dependency update PRs (Dependabot, Renovate)
- Enforce license compliance policies across dependencies
- Establish a triage and remediation process for CVE findings

---

## Step 1 — Map the Dependency Surface

Before choosing tools, enumerate all dependency sources:

| Layer | Package Manager | Lock File |
|---|---|---|
| Application (backend) | `go mod`, `pip`, `npm`, `maven`, `cargo` | `go.sum`, `requirements.txt`, `package-lock.json`, `pom.xml`, `Cargo.lock` |
| Container base image | Docker FROM | `Dockerfile` |
| OS packages (in image) | `apt`, `apk`, `yum` | Image layer |
| CI/CD tooling | GitHub Actions, scripts | `.github/workflows/*.yml` |
| Infrastructure | Terraform providers, Helm charts | `*.lock.hcl`, `Chart.lock` |

Checklist:
- [ ] All package managers in the repository identified
- [ ] Lock files committed to version control (reproducible builds)
- [ ] Transitive dependencies included in scan scope — not just direct dependencies
- [ ] Container base images included in scan scope

---

## Step 2 — Choose SCA Tools by Ecosystem

**Multi-ecosystem (recommended primary tool):**

| Tool | Ecosystems | Strengths | License |
|---|---|---|---|
| **Trivy** | Go, Node, Python, Java, Ruby, Rust, container, IaC | Fast, OSS, container + code + infra in one tool | Apache 2.0 |
| **Snyk** | All major ecosystems | Developer-friendly, fix PRs, SaaS | Commercial (free tier) |
| **OWASP Dependency-Check** | Java, .NET, Node, Python, Ruby | OSS, NVD-backed, mature | Apache 2.0 |
| **Grype** | Container images + filesystems | Fast, Syft-based, OSS | Apache 2.0 |

**Ecosystem-native tools (complement primary):**

| Tool | Ecosystem | Command |
|---|---|---|
| `govulncheck` | Go | `govulncheck ./...` |
| `npm audit` | Node.js | `npm audit --audit-level=high` |
| `pip-audit` | Python | `pip-audit` |
| `cargo audit` | Rust | `cargo audit` |
| `bundler-audit` | Ruby | `bundle audit check --update` |
| `dotnet list package --vulnerable` | .NET | Built-in |

Checklist:
- [ ] At least one multi-ecosystem scanner configured (Trivy recommended)
- [ ] Ecosystem-native tool added for each primary language stack
- [ ] Both tools run in CI on every commit

---

## Step 3 — Configure Trivy in CI

```yaml
# GitHub Actions
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    scan-ref: '.'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'           # Fail the build on critical/high findings
    ignore-unfixed: true     # Skip CVEs with no available fix (reduces noise)

- name: Upload Trivy results to GitHub Security tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'

# Also scan the Docker image:
- name: Scan Docker image
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'image'
    image-ref: 'myapp:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

Checklist:
- [ ] `exit-code: '1'` configured so critical/high findings fail the build
- [ ] `ignore-unfixed: true` set to reduce noise from CVEs with no fix available
- [ ] SARIF output uploaded to GitHub Security tab (or equivalent) for tracking
- [ ] Both filesystem (dependencies) and image (OS packages) scanned
- [ ] `.trivyignore` file used to acknowledge accepted risks with documented justification

---

## Step 4 — Define Severity Thresholds and Response Policy

| Severity | Action | SLA |
|---|---|---|
| **Critical** | Block build immediately | Fix within 24 hours |
| **High** | Block build; create tracking issue | Fix within 7 days |
| **Medium** | Create issue; do not block | Fix within 30 days |
| **Low** | Log only; review in weekly triage | Fix opportunistically |
| **Unfixable** | Acknowledge in `.trivyignore` with justification | Review every 90 days |

Checklist:
- [ ] Severity thresholds documented and communicated to the team
- [ ] Build gates enforce the policy automatically — Critical and High block merge
- [ ] Issues auto-created for High+ findings that don't block (medium, low)
- [ ] `.trivyignore` / `.snyk` files reviewed quarterly — accepted risks do not become permanent

---

## Step 5 — Automate Dependency Updates

**Dependabot (GitHub-native):**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
    labels: ["dependencies", "security"]
    groups:
      minor-patch:
        update-types: ["minor", "patch"]

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Renovate (more configurable):**
```json
{
  "extends": ["config:base"],
  "vulnerabilityAlerts": { "enabled": true, "labels": ["security"] },
  "automerge": true,
  "automergeType": "pr",
  "packageRules": [
    { "matchUpdateTypes": ["patch"], "automerge": true },
    { "matchUpdateTypes": ["major"], "automerge": false }
  ]
}
```

Checklist:
- [ ] Automated update PRs configured for all package managers
- [ ] Patch updates auto-merged if CI passes — reduces maintenance burden
- [ ] Major updates require manual review
- [ ] Security-only update PRs prioritized and labeled
- [ ] PR limit set to prevent update PR flood

---

## Step 6 — License Compliance

Checklist:
- [ ] Allowed and prohibited license list defined (e.g., MIT/Apache 2.0 allowed; GPL/AGPL requires review)
- [ ] License scanner configured: Trivy (`--scanners license`), FOSSA, or `license-checker` (Node)
- [ ] CI blocks on prohibited licenses in direct dependencies
- [ ] Transitive license violations logged and reviewed quarterly
- [ ] License compliance check results included in SBOM (see `sbom-generation` skill)

---

## Step 7 — Triage and Remediation Workflow

Weekly security triage:
1. Review new CVE findings from CI reports
2. Classify: exploitable in our context? Is a fix available?
3. Assign: owner and SLA based on severity policy
4. Track: issue created in project tracker
5. Verify: confirm fix applied and CVE no longer appears in scan

Checklist:
- [ ] Weekly triage meeting or async review process established
- [ ] CVE findings aggregated in a central dashboard (GitHub Security, Snyk dashboard)
- [ ] Remediation verified by re-running the scanner — not just by updating the version
- [ ] Near-miss CVEs (fixed but almost reached production) reviewed in post-mortem

---

## Output Report

### Critical
- Known critical CVE in a direct dependency currently deployed to production
- No automated scanning — vulnerabilities accumulate undetected
- Prohibitied license (GPL) in a direct dependency not detected by CI

### High
- High-severity CVE in a transitive dependency with a fix available but not applied
- No image scanning — OS-level CVEs in Docker base image undetected
- Dependabot/Renovate not configured — dependency updates require manual intervention

### Medium
- `.trivyignore` entries not reviewed — old accepted risks no longer documented or justified
- License compliance not enforced — legal risk accumulates undetected
- Medium-severity findings not tracked as issues — fall through the cracks

### Low
- Scan results not uploaded to GitHub Security tab — findings not visible to the team
- No update PR grouping — too many individual PRs create review fatigue
- Severity thresholds not documented — inconsistent triage decisions

### Passed
- Trivy (or equivalent) runs on every commit scanning both filesystem and container image
- Critical and High findings block the build automatically
- Dependabot/Renovate configured with patch auto-merge and security update priority
- License compliance enforced in CI
- Weekly triage process in place with documented SLAs

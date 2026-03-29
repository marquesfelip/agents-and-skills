---
name: pipeline-design
description: >
  CI/CD pipeline structure, stages, and automation best practices.
  Use when: CI/CD pipeline design, pipeline structure, pipeline stages, automation best practices,
  GitHub Actions pipeline, GitLab CI pipeline, Jenkins pipeline, Azure Pipelines, CircleCI,
  pipeline parallelism, pipeline caching, fail fast strategy, pipeline gates, deployment pipeline,
  build pipeline, test pipeline, pipeline optimization, CI pipeline review, CD pipeline review,
  deployment automation, pipeline as code, trunk-based development, branch strategy pipeline,
  pipeline security, secrets in pipeline, pipeline observability, pipeline performance.
argument-hint: >
  Describe the application type, tech stack, and deployment target (e.g., "Go microservice deployed
  to Kubernetes on AWS, using GitHub Actions"), and specify any current pipeline pain points or
  stages already in place.
---

# CI/CD Pipeline Design Specialist

## When to Use

Invoke this skill when you need to:
- Design a CI/CD pipeline from scratch for a new project
- Review and improve an existing pipeline for speed, reliability, or security
- Define pipeline stages, gates, and job sequencing
- Optimize pipeline performance with caching, parallelism, and fail-fast strategies
- Integrate security, quality, and compliance gates into the pipeline
- Choose and configure a CI/CD platform (GitHub Actions, GitLab CI, Jenkins, etc.)

---

## Step 1 — Define Pipeline Goals and Constraints

Before designing any pipeline, establish:

| Goal | Question to Answer |
|---|---|
| **Speed** | What is the target CI feedback time? (e.g., < 5 min on every PR) |
| **Safety** | What must be true before any code reaches production? |
| **Reliability** | What is the acceptable flake rate? |
| **Traceability** | Can every production artifact be traced back to a commit? |
| **Security** | Are secrets, signing, and supply chain requirements defined? |

Checklist:
- [ ] Maximum acceptable CI feedback time agreed (target: < 10 min for PR checks)
- [ ] Mandatory gates defined before deploy (tests pass, security scan, coverage threshold)
- [ ] Deployment targets identified (dev, staging, production) with promotion rules
- [ ] Branch strategy defined (trunk-based, GitFlow, GitHub Flow)

---

## Step 2 — Define Pipeline Stages

A well-structured pipeline follows this sequence:

```
[Commit] → Validate → Build → Test → Security → Package → Deploy (dev) → Verify → Deploy (prod)
```

| Stage | Runs On | Jobs | Fail Fast? |
|---|---|---|---|
| **Validate** | Every commit | Lint, format check, type check | Yes |
| **Build** | Every commit | Compile, generate artifacts | Yes |
| **Test** | Every commit | Unit tests (parallel), integration tests | Yes |
| **Security** | Every commit | SAST, secret scan, SCA | Yes (critical findings) |
| **Package** | Main branch + tags | Docker build, package publish | Yes |
| **Deploy Dev** | Main branch | Deploy to dev environment | Yes |
| **Smoke Test** | After deploy | Health check, critical path E2E | Yes |
| **Deploy Staging** | On release tag | Deploy to staging | Yes |
| **Performance** | On release tag | Load test, benchmark | Configurable |
| **Deploy Prod** | Manual gate | Deploy to production | Yes |

Checklist:
- [ ] Stages ordered: cheap/fast first, expensive/slow later (fail fast)
- [ ] Each stage has a single, clear responsibility
- [ ] Parallel jobs used within a stage where jobs are independent
- [ ] Manual approval gate configured before production deploy
- [ ] Rollback trigger defined for each deploy stage

---

## Step 3 — Fail-Fast and Parallelism Strategy

```yaml
# GitHub Actions example: parallel test jobs
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]    # Split tests across 4 runners
    steps:
      - run: go test ./... -run "Shard${{ matrix.shard }}"

  integration-tests:
    needs: unit-tests           # Integration only runs if unit tests pass
    runs-on: ubuntu-latest
```

**Parallelism patterns:**
- **Test sharding**: split test suite across N runners (halves runtime for each doubling)
- **Job matrix**: run the same job with different parameters (OS, language version)
- **Independent jobs**: lint, unit tests, and secret scan can run simultaneously
- **Dependent jobs**: integration tests only after unit tests; deploy only after all tests

Checklist:
- [ ] Independent jobs run in parallel (lint + test + secret scan simultaneously)
- [ ] Expensive jobs (`needs:`) depend on cheaper jobs to avoid wasted runner minutes
- [ ] Test suite split into shards for suites > 5 minutes
- [ ] `fail-fast: true` set on matrix jobs — cancel other shards on first failure

---

## Step 4 — Caching Strategy

Caching is the highest-leverage performance optimization in most pipelines:

```yaml
# GitHub Actions: dependency cache
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go/pkg/mod
      ~/.cache/go/build
    key: go-${{ hashFiles('go.sum') }}
    restore-keys: go-

# Docker layer cache (BuildKit)
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**What to cache:**
| Artifact | Cache Key |
|---|---|
| Language dependencies | Hash of lock file (`go.sum`, `package-lock.json`, `requirements.txt`) |
| Build output | Hash of source files |
| Docker layers | Registry or GitHub Actions Cache |
| Test results | Commit SHA (for re-run deduplication) |

Checklist:
- [ ] Dependency cache keyed on the lock file hash — invalidates only when deps change
- [ ] Docker layer cache enabled via BuildKit
- [ ] Cache restore fallback keys defined (partial cache hit is better than no cache)
- [ ] Cache size monitored — eviction causes full cache miss

---

## Step 5 — Secrets and Security in the Pipeline

Checklist:
- [ ] Secrets stored in the CI platform's secret store — never hardcoded in pipeline YAML
- [ ] Secrets scoped to the minimum environment needed (deploy key only in deploy job)
- [ ] `OIDC` (OpenID Connect) used instead of long-lived cloud credentials where supported (GitHub Actions → AWS/GCP)
- [ ] Pipeline YAML reviewed for secret exposure in `echo`, `env`, or log output
- [ ] Third-party actions pinned to a specific commit SHA — not a mutable tag (`uses: actions/checkout@v4` → `uses: actions/checkout@<SHA>`)
- [ ] `pull_request_target` trigger used with caution — it has access to secrets and can be abused
- [ ] Permissions declared explicitly per job (`permissions: contents: read`)

---

## Step 6 — Deployment Gates and Promotion

```yaml
# GitHub Actions: manual approval before production
deploy-prod:
  needs: [deploy-staging, performance-tests]
  environment:
    name: production
    url: https://app.example.com
  # "environment: production" requires manual approval if configured in repo settings
```

**Gate types:**
| Gate | Mechanism | When |
|---|---|---|
| Automated quality gate | Job dependency + status check | Always |
| Coverage gate | Coverage report threshold check | PR merge |
| Security gate | SAST/SCA finding severity threshold | PR merge |
| Performance gate | Load test threshold comparison | Release |
| Manual approval | Environment protection rule | Production deploy |
| Change window | Scheduled trigger / time check | Regulated environments |

Checklist:
- [ ] Branch protection rules enforce required status checks on the main branch
- [ ] Production environment requires manual approval from a designated reviewer
- [ ] Deployment records artifacts and commit SHA — every deploy is traceable
- [ ] Rollback job defined and tested — not improvised during an incident

---

## Step 7 — Pipeline Observability and Maintenance

Checklist:
- [ ] Pipeline duration tracked over time — alert on regressions > 20%
- [ ] Flaky test detection: failed-then-passed jobs logged and quarantined
- [ ] Pipeline YAML stored in version control and reviewed on change (treat as production code)
- [ ] Notifications configured: Slack/email on `main` branch failure; not on every PR failure
- [ ] Runner costs monitored — unused parallel jobs burn budget
- [ ] Self-hosted runners hardened: ephemeral, no persistent sensitive state, network-isolated

---

## Output Report

### Critical
- Secrets hardcoded in pipeline YAML or visible in job logs
- Third-party actions not pinned to commit SHA — supply chain attack surface
- No manual gate before production deployment — any passing PR can auto-deploy to prod
- `pull_request_target` trigger used without understanding its secret access implications

### High
- No caching configured — pipeline rebuilds all dependencies on every run (slow, costly)
- All pipeline jobs run sequentially — independent jobs not parallelized
- No branch protection enforcing required status checks — broken code can be merged

### Medium
- Stages not ordered fail-fast — expensive jobs run before cheap jobs fail
- No rollback job defined — recovery requires manual intervention during incidents
- Pipeline flakiness not tracked — intermittent failures erode team confidence

### Low
- Pipeline duration not monitored — gradual regressions go unnoticed
- Notifications sent on every PR failure — alert fatigue
- Docker layers not cached — image builds repeat from scratch on every run

### Passed
- Stages ordered fail-fast: lint → unit tests → integration → security → package → deploy
- Independent jobs run in parallel; dependent jobs use `needs:`
- Secrets scoped to minimum, stored in CI secret store, and accessed via OIDC where possible
- Third-party actions pinned to commit SHA
- Manual approval gate protects production; rollback job is defined and tested

---
name: preview-environments
description: >
  Ephemeral per-PR environments for pull request validation and review.
  Use when: preview environment, PR environment, ephemeral environment, review app, deploy preview,
  Heroku review apps, Vercel preview, Netlify deploy preview, PR deployment, per-PR environment,
  environment cleanup, namespace per PR, Kubernetes preview, feature branch environment,
  dynamic environment, branch deployment, preview URL, temporary environment, PR review app,
  isolated environment per PR, environment teardown, preview environment cost, PR environment secrets,
  PR smoke test, pull request deployment, environment per branch, auto-deploy PR.
argument-hint: >
  Describe the application stack and deployment target (e.g., "Next.js frontend + Go API on Kubernetes",
  "static frontend on Vercel + serverless API"), plus desired behavior: auto-create on PR open,
  auto-destroy on merge/close, smoke tests, and any cost/resource constraints.
---

# Preview Environments Specialist

## When to Use

Invoke this skill when you need to:
- Create ephemeral environments automatically when a PR is opened or updated
- Enable reviewers to test a feature branch in isolation without merging
- Run automated smoke or E2E tests against a branch-specific deployment
- Tear down environments automatically on PR close or merge
- Control costs and resource usage from preview environments
- Securely inject secrets into ephemeral environments

---

## Step 1 — Choose a Preview Environment Strategy

| Strategy | Best For | Complexity | Cost Control |
|---|---|---|---|
| **Platform-native** (Vercel, Netlify, Railway) | Frontend / full-stack PaaS | Low | Managed by platform |
| **Kubernetes namespace per PR** | Containerized microservices | High | Resource quotas per namespace |
| **Docker Compose on a shared VM** | Monolith or small teams | Medium | Shared host, limited isolation |
| **Ephemeral cloud environments** (Env0, Spacelift, Pulumi Deployments) | IaC-managed infra | High | TTL-based teardown |
| **Branch deployments on existing infra** | Teams with spare capacity | Low | Careful cleanup required |

**Decision guide:**
- Frontend-only or Jamstack → use Vercel/Netlify — zero configuration for preview URLs
- Containerized backend → use Kubernetes namespace-per-PR
- Mixed (frontend + API + DB) → namespace-per-PR or a PaaS that supports full-stack previews

Checklist:
- [ ] Strategy chosen based on stack, team size, and budget constraints
- [ ] Cost per preview environment estimated before implementation
- [ ] Maximum number of simultaneous preview environments defined (budget cap)
- [ ] Review: does the team actually need full-stack previews or just frontend?

---

## Step 2 — Naming Convention and URL Routing

Consistent naming prevents collisions and makes environments discoverable.

**Naming patterns:**
```
pr-<number>                       → pr-142
pr-<number>-<branch-slug>         → pr-142-feat-bulk-import
<owner>-<repo>-pr-<number>        → myorg-api-pr-142 (multi-repo)
```

**URL patterns:**
```
https://pr-142.preview.myapp.com          # Subdomain per PR
https://preview.myapp.com/pr-142          # Path-based (simpler cert management)
https://pr-142--myapp.netlify.app         # Netlify format
https://myapp-pr-142.vercel.app           # Vercel format
```

**Kubernetes namespace:**
```
preview-pr-142
```

Checklist:
- [ ] Naming is deterministic from the PR number — no manual input required
- [ ] URL/hostname communicated back to the PR as a GitHub status check or comment
- [ ] Wildcard TLS certificate provisioned for `*.preview.myapp.com` — avoids per-PR cert issuance
- [ ] DNS wildcard record configured for the preview subdomain

---

## Step 3 — Automate Environment Creation

**GitHub Actions: create environment on PR open/update**

```yaml
# .github/workflows/preview-deploy.yml
name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened]

env:
  PR_NUMBER: ${{ github.event.pull_request.number }}
  PREVIEW_NAMESPACE: preview-pr-${{ github.event.pull_request.number }}
  PREVIEW_URL: https://pr-${{ github.event.pull_request.number }}.preview.myapp.com

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    environment:
      name: preview-pr-${{ github.event.pull_request.number }}
      url: ${{ env.PREVIEW_URL }}

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t ghcr.io/myorg/myapp:pr-${{ env.PR_NUMBER }} .
          docker push ghcr.io/myorg/myapp:pr-${{ env.PR_NUMBER }}

      - name: Create Kubernetes namespace
        run: |
          kubectl create namespace ${{ env.PREVIEW_NAMESPACE }} --dry-run=client -o yaml | kubectl apply -f -
          kubectl label namespace ${{ env.PREVIEW_NAMESPACE }} preview=true pr=${{ env.PR_NUMBER }}

      - name: Deploy to preview namespace
        run: |
          helm upgrade --install myapp-preview ./helm/myapp \
            --namespace ${{ env.PREVIEW_NAMESPACE }} \
            --set image.tag=pr-${{ env.PR_NUMBER }} \
            --set ingress.host=pr-${{ env.PR_NUMBER }}.preview.myapp.com \
            --set resources.limits.cpu=500m \
            --set resources.limits.memory=512Mi \
            --wait --timeout=5m

      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Preview Environment Ready\n\n🔗 **URL:** ${{ env.PREVIEW_URL }}\n\n> Auto-generated for PR #${{ env.PR_NUMBER }}. Destroyed on merge or close.`
            })
```

Checklist:
- [ ] Deployment triggered on `pull_request` event types: `opened`, `synchronize`, `reopened`
- [ ] GitHub Deployment created with the correct environment name and URL — shows in the PR timeline
- [ ] Build uses the PR branch's code — not `main`
- [ ] Deployment status posted as a GitHub status check — blocks merge if deployment fails

---

## Step 4 — Secrets and Configuration in Preview Environments

Preview environments need configuration but must not expose production secrets.

**Tier the secrets:**

| Tier | Value | Approach |
|---|---|---|
| **Mock/stub** | Integration not needed in preview | Replace with mock server (WireMock, MSW) |
| **Preview-specific** | Dedicated external account | Store in a `preview` GitHub Environment |
| **Shared non-sensitive** | Same config as staging | Environment variable from CI |
| **Never in preview** | Production data, payment keys | Policy: blocked at workflow level |

```yaml
# Use GitHub Environments to scope secrets
environment:
  name: preview-pr-${{ github.event.pull_request.number }}

# Secrets available only to this environment:
env:
  DATABASE_URL: ${{ secrets.PREVIEW_DATABASE_URL }}   # Shared preview DB
  STRIPE_KEY: ${{ secrets.PREVIEW_STRIPE_TEST_KEY }}  # Stripe test mode only
```

**Database strategy for previews:**
| Option | Isolation | Cost | Complexity |
|---|---|---|---|
| Shared preview DB (schema per PR) | Medium | Low | Medium |
| Ephemeral DB per PR (Docker/RDS snapshot) | High | Medium | High |
| Read-only copy of anonymized staging | Low | Low | Low |
| In-memory or SQLite | High | None | Low (if testable) |

Checklist:
- [ ] No production secrets in preview environments — enforced by GitHub Environment protection rules
- [ ] Preview database is seeded with synthetic or anonymized data — never a prod copy
- [ ] External integrations (payment, email, SMS) use test/sandbox modes in previews
- [ ] Secrets scoped to the preview GitHub Environment — not available in unrelated workflows

---

## Step 5 — Automated Testing Against Preview

```yaml
  run-smoke-tests:
    needs: deploy-preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Wait for preview to be ready
        run: |
          for i in {1..30}; do
            curl -sf ${{ env.PREVIEW_URL }}/health && break
            echo "Waiting... ($i/30)"
            sleep 10
          done

      - name: Run smoke tests
        run: |
          BASE_URL=${{ env.PREVIEW_URL }} npx playwright test --project=smoke

      - name: Run API contract tests
        run: |
          API_URL=${{ env.PREVIEW_URL }}/api go test ./tests/contract/...
```

**Test scope for preview environments:**

| Test Type | Run in Preview? | Notes |
|---|---|---|
| Unit tests | No | Run in the build job |
| Smoke tests | Yes | Quick sanity — critical paths only |
| E2E (full suite) | Optional | Long-running; run on schedule or nightly |
| Contract tests | Yes | Validate API changes don't break consumers |
| Performance tests | No | Run against dedicated perf environment |
| Security DAST | Optional | Can run on preview; results in PR comment |

Checklist:
- [ ] Smoke tests run automatically after deployment — results posted to the PR
- [ ] Test suite scoped to fast-running tests — long suites block PR review unnecessarily
- [ ] Flaky tests quarantined — preview failures from flakiness must not block merge
- [ ] Preview URL parameterized in tests — not hardcoded

---

## Step 6 — Cost Controls and Resource Limits

Preview environments accumulate cost if not managed.

**Resource limits (Kubernetes):**
```yaml
# values-preview.yaml — applied to all preview Helm installs
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Namespace resource quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: preview-quota
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "10"
```

**TTL-based auto-teardown (in addition to PR close trigger):**
```yaml
# Scheduled cleanup: destroy previews older than 5 days
name: Cleanup Stale Preview Environments
on:
  schedule:
    - cron: '0 2 * * *'   # Daily at 2 AM UTC
jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete namespaces older than 5 days
        run: |
          kubectl get namespaces -l preview=true -o json | \
            jq -r '.items[] | select(.metadata.creationTimestamp | fromdateiso8601 < (now - 432000)) | .metadata.name' | \
            xargs -r kubectl delete namespace
```

Checklist:
- [ ] CPU and memory limits set on all preview deployments — no unbounded resource use
- [ ] Maximum number of concurrent preview environments enforced (namespace count limit)
- [ ] TTL-based cleanup job runs daily — destroys previews older than N days
- [ ] Preview environments do not include caches or queues unless explicitly needed — keep them minimal

---

## Step 7 — Teardown on PR Close or Merge

```yaml
# .github/workflows/preview-cleanup.yml
name: Destroy Preview Environment

on:
  pull_request:
    types: [closed]

jobs:
  destroy-preview:
    runs-on: ubuntu-latest
    steps:
      - name: Delete Kubernetes namespace
        run: |
          kubectl delete namespace preview-pr-${{ github.event.pull_request.number }} --ignore-not-found

      - name: Delete Docker image from registry
        run: |
          # Delete the PR-specific image tag to save registry storage
          skopeo delete docker://ghcr.io/myorg/myapp:pr-${{ github.event.pull_request.number }}

      - name: Remove GitHub Deployment environment
        uses: actions/github-script@v7
        with:
          script: |
            const envName = `preview-pr-${{ github.event.pull_request.number }}`;
            const envs = await github.rest.repos.getAllEnvironments({ owner: context.repo.owner, repo: context.repo.repo });
            const env = envs.data.environments.find(e => e.name === envName);
            if (env) {
              await github.rest.repos.deleteAnEnvironment({ owner: context.repo.owner, repo: context.repo.repo, environment_name: envName });
            }
```

Checklist:
- [ ] Teardown triggered on `pull_request.closed` — fires for both merged and abandoned PRs
- [ ] Kubernetes namespace deleted — all workloads, services, and ingresses removed
- [ ] PR-specific Docker image tag deleted from the registry — prevents registry bloat
- [ ] GitHub Deployment environment cleaned up — PR timeline stays clean
- [ ] Preview database schema/data dropped if a dedicated schema or DB was created per PR

---

## Output Report

### Critical
- Production secrets accessible in preview environments — branch code from forks can exfiltrate them
- No teardown on PR close — environments accumulate indefinitely, costs spiral

### High
- No resource limits on preview workloads — a single PR can consume all cluster capacity
- Preview environments using a production database — real data exposed to PR-branch code
- Preview URL not posted to the PR — reviewers cannot find or use the environment

### Medium
- No TTL-based cleanup — stale previews from abandoned PRs survive indefinitely
- Smoke tests not run against the preview — environment is deployed but never validated automatically
- External integrations using production-mode credentials — emails sent to real users, payments processed

### Low
- Preview images not cleaned from the registry after teardown — registry storage grows unbounded
- GitHub Deployment environments not removed on teardown — PR timeline accumulates ghost environments
- No maximum concurrent environment cap — cost can spike during high PR volume periods

### Passed
- Preview environments created automatically on PR open and updated on every push
- Secrets scoped to preview GitHub Environment; no production secrets accessible
- Resource limits and namespace quotas enforced on all preview deployments
- Smoke tests run automatically and results posted to the PR as a status check
- Teardown triggered on PR close/merge; namespaces, images, and GitHub Environments removed
- TTL-based cleanup job runs daily as a safety net against orphaned environments

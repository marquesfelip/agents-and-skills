---
name: release-management
description: >
  Controlled software release process design and execution.
  Use when: release management, release process, release checklist, release train, release gate,
  release candidate, release branch, hotfix release, rollback release, release notes, release approval,
  go no-go decision, feature freeze, deployment freeze, release coordinator, release schedule,
  release pipeline, release workflow, software release, production release, release readiness,
  release calendar, emergency release, patch release, major release, minor release,
  change management release, release sign-off, release communication, post-release monitoring.
argument-hint: >
  Describe the type of release (regular/hotfix/emergency) and the current pain points (e.g.,
  "releases break production frequently", "no rollback plan", "manual steps missing"), plus the
  deployment target (cloud, on-prem, multi-region).
---

# Release Management Specialist

## When to Use

Invoke this skill when you need to:
- Design or improve a controlled release process
- Define go/no-go criteria and release approval gates
- Create or audit a release checklist
- Plan rollback strategies before a release
- Run a hotfix or emergency release safely
- Define release communication templates and stakeholder notification

---

## Step 1 — Define Release Types and Ownership

| Release Type | Trigger | Branch Strategy | Risk Level |
|---|---|---|---|
| **Regular release** | Sprint end / scheduled cadence | `release/v1.2.0` branched from `main` | Medium |
| **Hotfix release** | Critical production bug | `hotfix/v1.1.1` branched from production tag | High |
| **Emergency release** | Security vulnerability or data issue | `hotfix/CVE-xxxx` fast-tracked | Critical |
| **Minor release** | Accumulated features | `release/v1.3.0` | Medium |
| **Major release** | Breaking changes / large migration | `release/v2.0.0` | High |

**Ownership matrix:**
| Role | Responsibility |
|---|---|
| Release coordinator | Drives the process, owns the checklist, calls go/no-go |
| Tech lead | Signs off on code quality and test results |
| QA lead | Signs off on test coverage and regression results |
| Security lead | Signs off on security scan results |
| Ops / SRE | Owns deployment execution and rollback readiness |

Checklist:
- [ ] Release type classified before release process begins
- [ ] Release coordinator assigned — single person accountable
- [ ] Release calendar published at the start of each sprint/quarter
- [ ] All participants understand their roles in the go/no-go decision

---

## Step 2 — Feature Freeze and Release Branch

```
Branching workflow:
                   feature/A ──┐
main ─────────────────────────┼──────── (fast-forward after release)
                               │
                        release/v1.2.0  ← only bug fixes from here
                               │
                           v1.2.0-rc.1
                           v1.2.0-rc.2
                           v1.2.0       ← production tag
```

**Feature freeze checklist:**
- [ ] Feature freeze date announced at least 5 business days before the release date
- [ ] `release/v1.x.y` branch created from `main` at freeze
- [ ] Only critical bug fixes cherry-picked into the release branch post-freeze
- [ ] No new features merged into the release branch — PRs target `main` instead
- [ ] Version bumped on the release branch (`v1.2.0-rc.1`)

---

## Step 3 — Release Candidate Validation

```
RC validation sequence:
1. Release branch created → RC tag pushed → CI/CD builds RC artifact
2. RC deployed to staging environment
3. Automated test suite (regression + integration + smoke) runs against RC
4. QA manual exploratory testing on staging
5. Security scan of RC artifact (SAST, DAST, SCA)
6. Performance baseline comparison (p95 latency, error rate)
7. Go/no-go meeting → decision to promote or create RC.2
```

**RC acceptance criteria (block release if any fail):**
| Gate | Pass Condition |
|---|---|
| Unit tests | 100% pass |
| Integration tests | 100% pass |
| Regression suite | 0 new failures vs baseline |
| Security scan | 0 Critical CVEs unresolved |
| Performance baseline | p95 latency within ±10% of last release |
| Smoke test on staging | All critical user paths functional |
| Dependency audit | No vulnerable dependencies above High severity |

Checklist:
- [ ] RC artifact built from the release branch — never from an untagged commit
- [ ] RC tested in an environment that mirrors production (data, config, integrations)
- [ ] All RC validation gates defined as pass/fail — no subjective criteria
- [ ] Failed gates automatically block the release in CI — go/no-go cannot be bypassed

---

## Step 4 — Go/No-Go Decision

The go/no-go meeting is a synchronous checkpoint before production deployment.

**Go criteria (all must be green):**
- [ ] All RC validation gates passed
- [ ] No P0/P1 bugs open against this release
- [ ] Rollback plan documented and reviewed by Ops/SRE
- [ ] Release notes drafted and reviewed
- [ ] Downstream teams (APIs consumers, integrations) notified and ready
- [ ] Deployment window confirmed (avoid Friday afternoons, peak traffic hours)
- [ ] On-call engineer identified and available during and after deployment

**No-go conditions (any triggers a no-go):**
- Unresolved Critical security finding
- Regression in a critical user flow
- Rollback plan not tested or missing
- Key stakeholder unavailable for sign-off
- Infrastructure instability in the target environment

**Decision record:**
Document the go/no-go decision in a release log with: date, participants, decision (Go/No-Go), rationale, and any open risks accepted.

---

## Step 5 — Deployment Execution

```yaml
# Recommended deployment order for blue-green or rolling releases:
1. Run database migrations (backward-compatible only — see zero-downtime-migrations skill)
2. Deploy to canary (5-10% of traffic)
3. Monitor for 10-30 minutes (error rate, latency, logs)
4. If metrics stable → promote to full rollout
5. Run smoke tests against production
6. Mark release as complete — update status page
```

**Deployment checklist:**
- [ ] Database migrations run before code deployment — no forward-only migrations that break the old version
- [ ] Feature flags used to decouple deployment from feature activation
- [ ] Canary or blue-green strategy used — never all-at-once for high-risk releases
- [ ] Automated smoke test runs immediately after deployment
- [ ] Monitoring dashboards open during deployment — error rate, latency, saturation
- [ ] Rollback procedure rehearsed — not assumed to work

---

## Step 6 — Release Notes and Communication

**Release notes template:**
```markdown
## v1.2.0 — 2026-03-15

### What's New
- feat: bulk import endpoint (#123)
- feat: email notification on order completion (#145)

### Bug Fixes
- fix: resolve null pointer in payment service (#167)
- fix: address timezone issue in report exports (#170)

### Breaking Changes
- The `/v1/orders` endpoint is deprecated. Migrate to `/v2/orders`. See migration guide: [link].

### Upgrade Instructions
1. Run migration script: `./scripts/migrate.sh`
2. Update client configuration: set `API_VERSION=v2`

### Known Issues
- None

### Security Fixes
- Upgraded `dependency-xyz` to address CVE-2026-12345
```

**Communication plan:**
| Audience | Channel | Timing |
|---|---|---|
| Internal engineering | Slack #releases | 1 hour before deployment |
| External API consumers | Email + developer portal | 48 hours before for breaking changes |
| End users | In-app notification / status page | 30 min before if UX-impacting downtime |
| Customer success | Email | 24 hours before major features land |

Checklist:
- [ ] Release notes written before deployment — not after
- [ ] Breaking changes called out with a migration guide
- [ ] Status page updated before and after deployment
- [ ] Post-release summary sent to stakeholders within 24 hours

---

## Step 7 — Post-Release Monitoring and Rollback

**Post-release monitoring window:** Monitor for a minimum of 1 hour (24 hours for major releases).

**Rollback decision tree:**
```
Error rate > baseline × 2?     → Immediate rollback
p99 latency > 3× baseline?     → Immediate rollback
Critical bug reported by user? → Assess; rollback or hotfix
Database corruption detected?  → Immediate rollback + incident
Canary stable, full rollout?   → Continue monitoring
```

**Rollback approaches:**
| Approach | When to Use | Speed |
|---|---|---|
| Blue-green switch back | Blue-green deployment in use | < 1 min |
| Previous Docker image deploy | Rolling deployment | 2-5 min |
| Kubernetes rollout undo | k8s deployment | < 2 min |
| Database rollback migration | If migration is reversible | Variable |
| Feature flag off | Feature-flagged change | < 30 sec |

Checklist:
- [ ] Rollback procedure documented *before* the release, not improvised during an incident
- [ ] Previous stable artifact tagged and available — not overwritten
- [ ] Database rollback migration exists if schema changed
- [ ] Rollback authorized by the release coordinator — single decision-maker
- [ ] Post-mortem scheduled if rollback is executed

---

## Step 8 — Hotfix Release Process

```
1. Identify and confirm the production bug (P0/P1 severity)
2. Create hotfix branch from the production tag: git checkout -b hotfix/v1.1.1 v1.1.0
3. Implement minimal fix — scope limited to the bug only
4. Run automated tests + focused regression on affected area
5. Abbreviated go/no-go (2-3 approvers minimum)
6. Deploy with accelerated monitoring window (30 min)
7. Cherry-pick the fix to main
8. Tag v1.1.1 on main
```

Checklist:
- [ ] Hotfix scoped to the minimum change needed — no opportunistic improvements
- [ ] Hotfix cherry-picked to `main` — not left only on the hotfix branch
- [ ] Abbreviated go/no-go documented — no unilateral deployments even in emergencies
- [ ] Communication sent immediately after hotfix — customers aware of the issue and resolution

---

## Output Report

### Critical
- No rollback plan in place — production incidents become prolonged outages
- Database migrations not backward-compatible — old version cannot run after migration applied
- Hotfixes deployed without any review — regressions introduced under pressure

### High
- No go/no-go gate defined — releases go to production without explicit sign-off
- Release notes not written — users and downstream teams surprised by breaking changes
- Feature freeze not enforced — last-minute changes increase release risk

### Medium
- Canary or progressive rollout not used — all-at-once deployment with no early warning
- Post-release monitoring window too short — regressions detected by customers, not ops
- Hotfix not cherry-picked to main — fix lost in the next regular release

### Low
- Release calendar not published ahead of time — stakeholders surprised by release windows
- Status page not updated during deployment — customers unfamiliar with ongoing maintenance
- Missing post-mortem culture — rollback incidents not analyzed, patterns repeat

### Passed
- Release types and ownership matrix defined and documented
- Feature freeze enforced; release branch strategy matches the team's risk tolerance
- All RC gates automated; go/no-go decision documented with participants and rationale
- Deployment uses canary or blue-green; rollback rehearsed before the release
- Release notes and communication sent to all relevant audiences before deployment

---
name: automated-versioning
description: >
  Semantic versioning automation practices in CI/CD pipelines.
  Use when: automated versioning, semantic versioning, semver, version automation, conventional commits,
  semantic-release, release-please, standard-version, git tag versioning, CHANGELOG generation,
  version bump automation, version strategy, pre-release version, release candidate, alpha beta version,
  monorepo versioning, package versioning, Docker image tagging, version from commit message,
  breaking change detection, minor patch major bump, automated release notes.
argument-hint: >
  Describe the repository structure and release cadence (e.g., "Node.js monorepo with 3 packages,
  releasing weekly, using conventional commits"), and specify the CI platform and whether changelogs
  and release notes should be auto-generated.
---

# Automated Versioning Specialist

## When to Use

Invoke this skill when you need to:
- Automate version bumps based on commit message semantics
- Set up conventional commits to drive automated versioning
- Configure `semantic-release`, `release-please`, or equivalent tools
- Generate CHANGELOGs and release notes automatically
- Define a version tagging strategy for Docker images and release artifacts
- Handle versioning for monorepos with independent package versions

---

## Step 1 — Choose a Versioning Strategy

| Strategy | How Version Is Determined | Tool |
|---|---|---|
| **Conventional Commits + semantic-release** | Commit messages drive version bump | `semantic-release` |
| **Conventional Commits + release-please** | PR-based release — release PR auto-created | `release-please` (Google) |
| **Manual tag + CI** | Engineer tags a commit; CI builds + publishes | Git tags + CI trigger |
| **CalVer** | Date-based version (2026.03.1) | Custom scripts |
| **Build number** | Monotonic integer from CI run | CI-native variable |

**Recommendation:** Conventional Commits + `release-please` for most teams — creates a visible release PR, gives a review checkpoint before bumping, and works well with monorepos.

Checklist:
- [ ] Versioning strategy agreed by the team before implementation
- [ ] Semantic Versioning (SemVer: `MAJOR.MINOR.PATCH`) understood and followed
- [ ] Release cadence defined (on-demand, weekly, per-sprint)
- [ ] Pre-release label convention defined (`v1.2.3-rc.1`, `v1.2.3-alpha.1`)

---

## Step 2 — Conventional Commits Standard

Conventional Commits drives automated version bumps via structured commit messages:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
BREAKING CHANGE: <description>   ← triggers MAJOR bump
```

**Type → version bump mapping:**
| Commit Type | Example | Version Bump |
|---|---|---|
| `fix:` | `fix: resolve null pointer in payment service` | PATCH |
| `feat:` | `feat: add bulk import endpoint` | MINOR |
| `feat!:` or `BREAKING CHANGE:` | `feat!: remove v1 API endpoints` | MAJOR |
| `chore:`, `docs:`, `refactor:`, `test:` | — | No bump |
| `perf:` | `perf: optimize query with index` | PATCH |

**Enforce with commitlint:**
```yaml
# .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [2, "always", ["feat", "fix", "docs", "style", "refactor", "perf", "test", "chore", "revert", "ci"]]
  }
}
```

```yaml
# GitHub Actions: validate commit messages on PR
- name: Validate commit messages
  uses: wagoid/commitlint-github-action@v5
```

Checklist:
- [ ] Conventional Commits format documented in `CONTRIBUTING.md`
- [ ] `commitlint` enforced on every PR — non-conforming commits blocked
- [ ] Husky pre-commit hook configured for local enforcement
- [ ] Team trained on `BREAKING CHANGE:` footer — misuse triggers unintended major bumps

---

## Step 3 — Configure release-please

`release-please` (Google) creates a release PR that accumulates changes until merged:

```yaml
# .github/workflows/release.yml
name: Release Please

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: go     # or: node, python, java, simple
          # For monorepos:
          # config-file: release-please-config.json
          # manifest-file: .release-please-manifest.json

      # Only runs when release-please merges a release PR:
      - name: Build and publish
        if: ${{ steps.release.outputs.release_created }}
        run: |
          make build
          make docker-publish TAG=${{ steps.release.outputs.tag_name }}
```

**Monorepo config (`release-please-config.json`):**
```json
{
  "packages": {
    "services/orders": { "release-type": "go", "package-name": "orders-service" },
    "services/payments": { "release-type": "go", "package-name": "payments-service" },
    "frontend": { "release-type": "node" }
  }
}
```

Checklist:
- [ ] `release-please-action` configured with the correct `release-type` for the ecosystem
- [ ] Release PR reviewed before merging — last checkpoint before version is bumped
- [ ] Post-release jobs triggered via `steps.release.outputs.release_created` output
- [ ] `CHANGELOG.md` auto-generated and committed by release-please

---

## Step 4 — Configure semantic-release (Alternative)

`semantic-release` bumps version and publishes in one automated step (no release PR):

```yaml
# .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    ["@semantic-release/npm", { "npmPublish": false }],
    "@semantic-release/github",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

```yaml
# GitHub Actions
- name: Release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
  run: npx semantic-release
```

Checklist:
- [ ] `[skip ci]` tag included in the version bump commit — prevents infinite loop
- [ ] `@semantic-release/changelog` plugin generates `CHANGELOG.md`
- [ ] `@semantic-release/github` creates GitHub Release with release notes
- [ ] Pre-release channels configured for `alpha` and `beta` branches if needed

---

## Step 5 — Docker Image Tagging Strategy

```yaml
# GitHub Actions: tag Docker images with semantic version
- name: Docker meta
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/myorg/myapp
    tags: |
      type=semver,pattern={{version}}        # v1.2.3
      type=semver,pattern={{major}}.{{minor}} # v1.2
      type=semver,pattern={{major}}           # v1
      type=sha                               # sha-abc1234
      type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

**Tag conventions:**
| Tag | Meaning | Mutable? |
|---|---|---|
| `v1.2.3` | Exact release | No |
| `v1.2` | Latest patch of 1.2.x | Yes |
| `v1` | Latest minor of 1.x.x | Yes |
| `latest` | Latest stable release | Yes |
| `sha-abc1234` | Specific commit | No |
| `main` | Latest build of main branch | Yes |

Checklist:
- [ ] Immutable tags (`v1.2.3`, `sha-...`) used in deployment manifests — not `latest`
- [ ] `docker/metadata-action` used to generate consistent tags from Git refs
- [ ] `latest` tag only updated from the main/default branch

---

## Step 6 — Version Artifacts and Provenance

Checklist:
- [ ] Version embedded in the binary at build time (Go: `-ldflags "-X main.version=$VERSION"`)
- [ ] Version exposed via a `/health` or `/version` endpoint — observable in production
- [ ] Git commit SHA embedded alongside the version — correlate deployed code to source
- [ ] CHANGELOG generated and attached to GitHub Release
- [ ] Release notes include: new features, bug fixes, breaking changes, and upgrade instructions

---

## Output Report

### Critical
- Version bumped manually with no automation — inconsistent versioning across releases
- BREAKING changes merged without a MAJOR version bump — consumers receive breaking changes silently
- `latest` Docker tag used in production — deployed version is untrackable

### High
- No commitlint enforcement — conventional commits convention not followed consistently
- `semantic-release` loop: version bump commit triggers another release pipeline run
- Monorepo packages versioned globally instead of independently — unrelated bumps couple releases

### Medium
- Release notes not generated — engineers must manually write changelogs
- Pre-release channel (`rc`, `alpha`) not defined — release candidates land directly on `latest`
- Version not embedded in the binary — cannot confirm running version without checking the registry

### Low
- Mutable version tags (`v1.2`) not updated after a patch release — consumers on the wrong version
- `[skip ci]` not added to version bump commit — extra CI run wasted
- CHANGELOG accumulated but never structured — hard to scan for breaking changes

### Passed
- Conventional commits enforced via commitlint on every PR
- release-please or semantic-release generates version, CHANGELOG, and GitHub Release automatically
- Docker images tagged with immutable version tags and SHA; mutable convenience tags updated
- Version and commit SHA embedded in the binary and exposed via a runtime endpoint
- Monorepo packages versioned independently with release-please manifest

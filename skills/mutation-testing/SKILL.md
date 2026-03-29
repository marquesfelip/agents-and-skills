---
name: mutation-testing
description: >
  Mutation-based testing to verify test suite effectiveness and detect weak assertions.
  Use when: mutation testing, mutation score, test effectiveness, PIT mutation testing, Pitest,
  Stryker, Mutmut, mutation operator, surviving mutant, killed mutant, test suite quality,
  weak test assertion, test without assertion, false confidence coverage, code coverage vs mutation score,
  equivalent mutant, mutation testing CI, mutation report, test strength, test verification,
  assertion quality, mutation analysis, mutation score threshold, test suite gap.
argument-hint: >
  Describe the module or service to analyze (e.g., "domain layer of the OrderService in Java"),
  and specify the mutation testing tool, current coverage, and mutation score target.
---

# Mutation Testing Specialist

## When to Use

Invoke this skill when you need to:
- Verify that existing tests are actually catching bugs, not just executing code
- Identify test assertions that are weak or missing
- Understand why high code coverage still allows bugs through
- Set up automated mutation testing in CI with a quality gate
- Prioritize which modules need stronger tests based on mutation score
- Educate a team on the difference between code coverage and test effectiveness

---

## Step 1 — Understand Mutation Testing Concepts

**Core Concepts:**

| Concept | Definition |
|---|---|
| **Mutant** | A copy of the source code with one deliberate bug injected (mutation operator) |
| **Killed Mutant** | A test failed when run against the mutant — the test detected the bug |
| **Surviving Mutant** | All tests passed against the mutant — the bug was not detected |
| **Mutation Score** | `killed / (killed + survived) × 100%` — higher is better |
| **Equivalent Mutant** | Mutation that does not change behavior — cannot be killed; must be excluded |

**Common Mutation Operators:**
- Arithmetic: `+` → `-`, `*` → `/`
- Logical: `&&` → `||`, `!x` → `x`
- Relational: `>` → `>=`, `==` → `!=`
- Boundary: `> 0` → `>= 0`
- Return value: `return true` → `return false`
- Null: `return value` → `return null`
- Conditional removal: delete an `if` block entirely

Checklist:
- [ ] Team understands that high coverage ≠ high mutation score
- [ ] Surviving mutants are understood as untested behaviors — potential real bugs
- [ ] Equivalent mutants acknowledged — exclude them from the score if necessary

---

## Step 2 — Choose a Mutation Testing Tool

| Tool | Language | Notes |
|---|---|---|
| **Pitest (PIT)** | Java / JVM | Fastest JVM mutation tool; Maven/Gradle plugin |
| **Stryker Mutator** | JavaScript, TypeScript, C#, Scala | Multi-language, excellent report UI |
| **Mutmut** | Python | Simple, pip-installable |
| **Infection** | PHP | Symfony/Laravel ecosystem |
| **mutagen** | Go | Go mutation testing |
| **cargo-mutants** | Rust | Rust ecosystem |
| **ArchUnit + PIT** | Java | Combined structural + mutation |

Checklist:
- [ ] Tool selected for the project's primary language
- [ ] Tool integrates with existing test framework (JUnit, Jest, pytest)
- [ ] Tool generates HTML report with surviving mutant details
- [ ] Tool supports incremental mode (analyze only changed code) for CI performance

---

## Step 3 — Configure the Tool

**Pitest (Java) example:**
```xml
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <configuration>
    <targetClasses>
      <param>com.example.domain.*</param>
      <param>com.example.usecases.*</param>
    </targetClasses>
    <targetTests>
      <param>com.example.*Test</param>
    </targetTests>
    <mutationThreshold>80</mutationThreshold>
    <coverageThreshold>80</coverageThreshold>
    <outputFormats>
      <param>HTML</param>
      <param>XML</param>
    </outputFormats>
  </configuration>
</plugin>
```

**Stryker (JavaScript) example:**
```json
{
  "mutate": ["src/domain/**/*.ts", "src/usecases/**/*.ts"],
  "thresholds": { "high": 80, "low": 60, "break": 60 },
  "reporters": ["html", "json", "progress"]
}
```

Checklist:
- [ ] Target classes scoped to domain and use case layers — **exclude** generated code, DTOs, and infrastructure adapters
- [ ] Test classes correctly associated with the mutation target
- [ ] Mutation threshold set (recommended: ≥ 80% for domain layer)
- [ ] HTML report output enabled for manual review
- [ ] Incremental / `--since-last-commit` mode enabled for CI performance

---

## Step 4 — Analyze Surviving Mutants

After running mutation tests, analyze survivors by category:

**Pattern: Missing Assertion**
```
// Test that does not assert the return value
void test_calculate_discount() {
    service.calculateDiscount(order);  // no assertion
}
```
Fix: Add assertion on the return value.

**Pattern: Weak Assertion**
```
// Asserts something happened but not what value was returned
assertTrue(result != null);  // survives any value mutation
```
Fix: Assert the exact value: `assertEquals(10.0, result.getDiscount())`.

**Pattern: Missing Edge Case**
```
// Boundary mutation on "> 0" survives as ">= 0"
// No test with value = 0
```
Fix: Add test case for the boundary value.

**Pattern: Missing Negative Path**
```
// Relational operator `<` mutated to `<=` survives
// No test that exercises the else branch
```
Fix: Add test for the alternative branch.

Checklist:
- [ ] Every surviving mutant reviewed — categorize as: missing assertion, weak assertion, missing test case, or equivalent mutant
- [ ] Equivalent mutants documented and excluded from score calculation
- [ ] Surviving mutants prioritized by risk (domain logic > utility code)
- [ ] Tests added or improved to kill surviving mutants

---

## Step 5 — Integrate into CI

**Full run (nightly or pre-release):**
```yaml
mutation-testing:
  schedule: nightly
  script:
    - mvn pitest:mutationCoverage
  artifacts:
    paths:
      - target/pit-reports/
  rules:
    - if: mutation_score < 75  # fail build
```

**Incremental run (on every PR — faster):**
```yaml
mutation-testing-incremental:
  script:
    - stryker run --since-last-commit
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

Checklist:
- [ ] Full mutation run scheduled nightly on main — complete picture
- [ ] Incremental run on PRs — only mutates changed code (fast feedback)
- [ ] Mutation score threshold set and build fails if breached
- [ ] Mutation report uploaded as CI artifact and linked in PR
- [ ] Score trend tracked over time — regression is visible before it accumulates

---

## Step 6 — Prioritize by Risk

Don't apply mutation testing uniformly — focus investment where it matters:

| Layer | Priority | Target Score |
|---|---|---|
| Domain entities and value objects | **Highest** | ≥ 90% |
| Use cases and application services | **High** | ≥ 85% |
| Input validators and business rules | **High** | ≥ 85% |
| Utility / helper functions | **Medium** | ≥ 75% |
| Infrastructure adapters (repos, clients) | **Low** | Covered by integration tests |
| Generated code, DTOs, config | **Exclude** | — |

Checklist:
- [ ] Mutation testing scoped to high-value code layers only
- [ ] Infrastructure and generated code excluded from mutation scope
- [ ] Highest-risk modules analyzed first when onboarding to mutation testing

---

## Output Report

### Critical
- Mutation score < 50% on domain or use case layer — test suite provides false safety
- Tests exist with no assertions — coverage is inflated and meaningless
- Critical business logic (pricing, permissions, state transitions) has surviving mutants on boundary conditions

### High
- Mutation score between 50–65% on domain layer — significant gaps in assertion quality
- Entire branches of business logic not killed by any test — hidden untested paths
- No mutation testing in CI — quality regressions are not automatically detected

### Medium
- Surviving mutants concentrated in one module — indicates low-quality tests for that area
- Equivalent mutants not excluded — score artificially deflated, causing unnecessary false failures
- Mutation run targets all layers including generated code — slow and noisy results

### Low
- Full mutation run not separated from incremental — CI is slow on every commit
- Mutation report not published as PR artifact — developers cannot review survivors inline
- Mutation score not trended — gradual regression not visible

### Passed
- Domain and use case layers achieve ≥ 80% mutation score
- All surviving mutants triaged: fixed or documented as equivalent mutants
- Incremental mutation run integrated into PR pipeline with < 5 min runtime
- Full nightly run tracks mutation score trend across commits
- No tests exist without meaningful assertions

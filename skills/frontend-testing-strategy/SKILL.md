---
name: frontend-testing-strategy
description: >
  Component testing, UI testing, and frontend behavior validation strategies.
  Use when: frontend testing strategy, component testing, UI testing, frontend behavior validation,
  unit test React, unit test Vue, unit test Angular, Jest, Vitest, Testing Library, React Testing Library,
  Playwright, Cypress, Storybook testing, snapshot testing, visual regression, accessibility testing,
  a11y test, end-to-end frontend, user interaction test, DOM testing, frontend test pyramid,
  frontend test coverage, frontend TDD, frontend QA, Puppeteer, user event simulation.
argument-hint: >
  Describe the frontend stack and what needs testing (e.g., "React + TypeScript SPA with Redux and REST API calls"),
  and specify the framework, layer, or behavior in focus.
---

# Frontend Testing Strategy Specialist

## When to Use

Invoke this skill when you need to:
- Design or review a frontend test suite from scratch
- Choose the right test type for UI components, hooks, or page flows
- Apply the frontend test pyramid correctly
- Configure testing frameworks (Jest, Vitest, Playwright, Cypress)
- Validate component behavior, accessibility, and visual correctness
- Improve test isolation, resilience, and maintainability

---

## Step 1 — Map the Frontend Test Pyramid

Frontend test layers (ordered cheapest → most expensive):

| Layer | What It Tests | Tools |
|---|---|---|
| **Unit** | Pure functions, hooks, reducers, selectors, formatters | Jest, Vitest |
| **Component** | Isolated UI components with simulated user interaction | Testing Library, Storybook |
| **Integration** | Multiple components working together, API mocking | Testing Library + MSW |
| **E2E** | Critical user flows in a real browser | Playwright, Cypress |
| **Visual** | Visual regression against baseline screenshots | Percy, Chromatic, Playwright |

Checklist:
- [ ] Most tests live at unit + component layer
- [ ] E2E covers only critical happy paths and highest-risk flows
- [ ] Visual tests run against Storybook stories or key pages

---

## Step 2 — Unit Test Pure Logic

Unit tests target framework-independent logic:

- Utility functions, formatters, validators, transformers
- Redux reducers, Zustand actions, Pinia stores
- Custom hooks (tested with `renderHook`)
- Route logic, guards, permission checks

Checklist:
- [ ] No DOM rendering in unit tests — pure function input/output only
- [ ] Hooks tested with `renderHook` from Testing Library
- [ ] State management logic tested in isolation without components
- [ ] All edge cases and boundary values covered with parameterized tests
- [ ] No network calls, no real timers (`jest.useFakeTimers()`)

---

## Step 3 — Component Tests

Component tests render a component in isolation and simulate user interactions:

- Use `@testing-library/react` (or Vue/Angular equivalent) — never access internals
- Query by accessible role, label, text — **not** by class, ID, or implementation details
- Simulate real user behavior: `userEvent.click()`, `userEvent.type()` — not `fireEvent`
- Assert on what the user sees or hears, not internal state

Checklist:
- [ ] `getByRole`, `getByLabelText`, `getByText` used — never `getByTestId` as default
- [ ] `userEvent` used over `fireEvent` for interaction simulation
- [ ] API calls mocked with MSW (Mock Service Worker) — not manual fetch mocks
- [ ] Loading states, error states, and empty states tested per component
- [ ] No assertions on component internal state or props directly
- [ ] Async assertions use `waitFor` / `findBy` — no arbitrary `setTimeout`

---

## Step 4 — Integration Tests

Integration tests verify multiple components wired together with mocked APIs:

- Render a full page or feature slice
- Mock the network layer with MSW — not individual service functions
- Test user flows that cross component boundaries (form → submit → success screen)

Checklist:
- [ ] MSW handlers defined in `src/mocks/handlers.ts` and shared across tests
- [ ] Each test resets MSW handlers to prevent state leakage
- [ ] Router context provided when testing navigation (MemoryRouter)
- [ ] Auth/session context provided as a wrapper when required
- [ ] Test covers complete user action → UI response cycles

---

## Step 5 — End-to-End Tests

E2E tests drive a real browser against a real or stubbed backend:

- **Playwright** (preferred): cross-browser, reliable, built-in assertions
- **Cypress**: rich DX, best for single-browser CI environments

Checklist:
- [ ] Cover only: critical user paths, checkout/payment, auth flows, high-value forms
- [ ] Each test seeds its own state (login, seed DB, or use test API routes)
- [ ] No E2E test relies on another test's side effects
- [ ] Selectors use `data-testid` only for elements with no accessible role
- [ ] Test retries configured for flaky network timing (Playwright: `retries: 2`)
- [ ] E2E suite runs in < 10 minutes

---

## Step 6 — Accessibility and Visual Tests

**Accessibility:**
- `jest-axe` for automated a11y checks in component tests
- Playwright `page.accessibility.snapshot()` for page-level checks
- Manual checks with screen reader for critical flows

**Visual regression:**
- Storybook + Chromatic (preferred) or Playwright visual snapshot
- Baseline images committed and reviewed on PR

Checklist:
- [ ] `axe` run on every component test for WCAG 2.1 AA violations
- [ ] Color contrast, focus management, ARIA labels validated
- [ ] Visual baselines updated intentionally, not silently
- [ ] No visual test is the sole test for a component's behavior

---

## Step 7 — CI Integration

```yaml
# Recommended CI order
- name: Unit + Component Tests   # Fast, runs on every commit
- name: Integration Tests        # Medium, runs on every commit
- name: Visual Tests             # Runs on PR against Storybook
- name: E2E Tests                # Runs on merge to main only
```

Checklist:
- [ ] Unit and component tests run in parallel workers
- [ ] E2E tests run in headed mode locally, headless in CI
- [ ] Test output includes screenshots and traces on failure (Playwright)
- [ ] Coverage report uploaded to CI artifact storage

---

## Output Report

### Critical
- No tests exist for components or user flows
- Tests assert on implementation details (class names, internal state) — brittle and misleading
- E2E tests cover edge cases that belong at the component layer (inverted pyramid)

### High
- `act()` warnings in test output — async state updates not properly awaited
- MSW not used — `fetch` mocked manually per test causing drift from real API shape
- Tests depend on execution order or shared global state

### Medium
- `getByTestId` overused instead of accessible queries
- `fireEvent` used instead of `userEvent` — doesn't simulate real behavior
- No accessibility testing integrated into the suite

### Low
- Snapshot tests used for large component trees — fragile and unhelpful
- Test file naming inconsistent
- No visual regression baseline for critical UI

### Passed
- Test pyramid is balanced: most tests at unit/component, few at E2E
- All queries use accessible roles and labels
- MSW centralized and shared across test suites
- CI runs cheap tests first and fails fast
- a11y checks run automatically on component render

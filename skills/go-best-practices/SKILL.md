---
name: go-best-practices
description: 'Review, refactor, or write Go code following SOLID, Clean Code, Clean Architecture, and Effective Go principles. Use when: Go best practices, SOLID in Go, clean code golang, clean architecture Go, idiomatic Go, refactor Go code, code review golang, effective go, go design patterns, go SOLID principles. Invokes go-best-practices subagent for deep analysis. Produces concrete refactored code or a prioritized list of violations with fixes.'
argument-hint: 'File, package, or function to review — or describe what you want to write'
---

# Go Best Practices Specialist

## When to Use
- Reviewing existing Go code for SOLID violations, anti-patterns, or non-idiomatic style
- Writing new Go code and wanting it to be clean, testable, and maintainable from the start
- Refactoring a package to align with Clean Architecture (ports & adapters, dependency inversion)
- Applying Effective Go idioms (error handling, naming, package design, interfaces)
- Checking for common Go design smells: god structs, leaky abstractions, over-engineering

---

## Required Reading
Load these references before producing output:
- [SOLID in Go](./references/solid-in-go.md)
- [Clean Architecture in Go](./references/clean-architecture-go.md)
- [Effective Go Cheatsheet](./references/effective-go-cheatsheet.md)

---

## Step 1 — Understand the Context

1. Read all files mentioned by the user. If none are specified, ask for the target file or package.
2. Determine the **mode**:

| User intent | Mode |
|---|---|
| "Review this code" / "check my code" | Review mode — audit existing code |
| "Write a …" / "implement …" | Write mode — produce clean code from scratch |
| "Refactor this" | Refactor mode — transform existing code in place |
| "Explain why this is bad" | Explain mode — annotate violations with rationale |

3. Identify the **domain layer** the code lives in (guides which principles apply most):

| Layer | Examples | Focus |
|---|---|---|
| Entity / Domain | structs, value objects, business rules | SOLID, DDD, immutability |
| Use Case / Application | orchestration, service logic | SRP, interface segregation |
| Interface / Adapter | HTTP handlers, CLI, XML parsers | Dependency inversion, clean boundaries |
| Infrastructure | DB drivers, file I/O, external APIs | Port/Adapter pattern, error wrapping |

---

## Step 2 — Run the go-best-practices Subagent

Invoke the `go-best-practices` subagent with the target files and the detected mode.

Ask it to:
- Identify **SOLID violations** with line references and severity (High / Medium / Low)
- Identify **Effective Go** style issues (naming, error strings, interface size, receiver types)
- Identify **Clean Architecture boundary violations** (e.g., business logic calling a driver directly)
- Suggest concrete fixes, not just descriptions

---

## Step 3 — Produce the Output

### For Review mode
Return a **prioritized violation list**:

```
## Violations Found

### High
- [SRP] `ProcessarPasta` does XML parsing AND progress reporting AND mutex management → split into separate functions
  Fix: extract `reportarProgresso()` and pass a `sync.Mutex`-protected collector struct

### Medium
- [ISP] `Repositorio` interface has 7 methods; callers only use 2 → split into focused interfaces

### Low
- [Naming] `resultado` slice declared at package level when it can be local → move inside function
```

### For Write mode
Produce idiomatic Go code directly, with inline comments explaining each principle applied.

### For Refactor mode
Show a **before/after diff** for each violation, keeping changes minimal and focused.

### For Explain mode
Annotate code with `// ❌ Violation: <principle> — <reason>` and `// ✅ Fix: <approach>`.

---

## Step 4 — Apply Effective Go Final Pass

Before finishing, scan for these quick wins regardless of mode:

| Check | Rule |
|---|---|
| Error strings | Must not be capitalized or end with punctuation |
| Interface names | Single-method interfaces: `<Verb>er` (e.g., `Reader`, `Processor`) |
| Receiver names | Short, consistent, never `this` or `self` |
| Return early | Prefer guard clauses over nested if/else |
| Exported comments | Every exported symbol must have a doc comment |
| `init()` usage | Avoid unless absolutely necessary |
| `panic` | Only at program startup for unrecoverable config; never in library code |

---

## Step 5 — Validate Before Delivering

Run (or instruct the user to run):
```bash
go vet ./...
staticcheck ./...   # if available
```

If `staticcheck` is not installed:
```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
```

---

## Quality Checks
- [ ] Every violation has a concrete fix suggestion, not just a description
- [ ] Changes respect existing naming conventions (Brazilian Portuguese vars in this project)
- [ ] No unnecessary abstractions introduced (YAGNI — don't over-engineer)
- [ ] Interfaces defined at the **consumer** side, not the implementer side
- [ ] All proposed code changes compile (mentally verified or run `go build`)
- [ ] Refactors are minimal — only change what was flagged

---

## After Completion

Tell the user:
1. **How many violations** were found per severity tier
2. **Which principles** were most relevant for their code
3. **Suggested next steps** — e.g., "add tests for the new interface boundary" (pair with `go-unit-tests` skill) or "check for race conditions" (pair with `avoid-race-conditions` skill)

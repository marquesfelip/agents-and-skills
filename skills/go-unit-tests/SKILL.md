---
name: go-unit-tests
description: 'Write, review, and improve unit tests for Go code. Use when: writing Go tests, unit test in golang, add tests to Go, test coverage Go, table-driven tests, subtests, mock in Go, testify, testing package, test strategy Go, go test, missing tests, write test suite. Invokes go-test-strategy subagent for risk analysis. Produces complete test files with table-driven cases, subtests, and mocking.'
argument-hint: 'File or package to test (e.g., entity/entity.go, or describe the function)'
---

# Go Unit Tests Specialist

## When to Use
- Writing tests for new or existing Go functions
- Reviewing test quality or coverage gaps
- Adding tests to untested packages (XML parsers, file walkers, concurrency code)
- Choosing between mocking strategies (interface fakes, `testify/mock`, stub files)
- Setting up a test suite from scratch for a Go module

## Step 1 — Understand What Needs Testing

1. Read the target file(s) to understand inputs, outputs, and side effects.
2. Identify the **risk tier** for each function:

| Tier | Criteria | Test depth |
|---|---|---|
| High | Parses external data (XML, JSON, file I/O), concurrency, DB writes | Table-driven + error cases + edge cases |
| Medium | Business logic, data transformations | Table-driven, happy path + 2–3 error cases |
| Low | Simple getters, logging, pure formatting | Smoke test or skip |

3. Note any external dependencies (filesystem, DB, network) — these need fakes or temp dirs.

## Step 2 — Choose the Right Test Pattern

| Situation | Pattern |
|---|---|
| Multiple input/output variants | Table-driven test (`[]struct{ name, input, want }`) |
| Grouped sub-scenarios | `t.Run(tc.name, ...)` inside table loop |
| External file I/O (e.g., XML parsing) | `os.CreateTemp` / `t.TempDir()` + embed fixture files |
| Interface dependency | Define minimal fake struct implementing the interface |
| Complex mock with call assertions | `testify/mock` (`github.com/stretchr/testify/mock`) |
| Parallel-safe unit tests | `t.Parallel()` at top of each `t.Run` body |

> **Prefer** stdlib `testing` + table-driven over heavy mock frameworks. Only reach for `testify` when assertion ergonomics matter significantly.

## Step 3 — Structure the Test File

Follow these conventions (consistent with Go idioms and this project's Brazilian Portuguese naming):

```go
package <same_package>_test   // external test package; use same package only to access unexported symbols

import (
    "testing"
    // other imports
)

func Test<FunctionName>(t *testing.T) {
    t.Parallel()

    casos := []struct {              // "casos" = test cases (PT-BR)
        nome    string              // "nome" = name
        entrada <InputType>         // "entrada" = input
        quer    <OutputType>        // "quer" = want
        erro    bool                // "erro" = error expected
    }{
        {nome: "caso feliz", entrada: ..., quer: ..., erro: false},
        {nome: "entrada vazia", entrada: ..., quer: ..., erro: true},
    }

    for _, tc := range casos {
        tc := tc
        t.Run(tc.nome, func(t *testing.T) {
            t.Parallel()
            got, err := FunctionUnderTest(tc.entrada)
            if tc.erro && err == nil {
                t.Fatal("esperava erro, não recebeu")
            }
            if !tc.erro && err != nil {
                t.Fatalf("erro inesperado: %v", err)
            }
            if got != tc.quer {
                t.Errorf("got %v, quer %v", got, tc.quer)
            }
        })
    }
}
```

## Step 4 — Handle File I/O and XML Fixtures

For functions that read files (like `readAxTableXML`, `readDescriptorXML`):

1. Place fixture XML files under `testdata/` in the same package directory.
   `testdata/` is ignored by `go build` but included by `go test`.
2. Use `t.TempDir()` for writable temp directories — cleaned up automatically.
3. For XML tests, create minimal valid XML that exercises the specific fields:

```go
const axTableXMLMinimo = `<?xml version="1.0" encoding="utf-8"?>
<AxTable>
  <Name>MinhaTabela</Name>
  <Fields>
    <AxTableField>
      <Name>MeuCampo</Name>
      <ExtendedDataType>AccountNum</ExtendedDataType>
    </AxTableField>
  </Fields>
</AxTable>`

func escreveFixture(t *testing.T, conteudo string) string {
    t.Helper()
    dir := t.TempDir()
    caminho := filepath.Join(dir, "tabela.xml")
    if err := os.WriteFile(caminho, []byte(conteudo), 0o600); err != nil {
        t.Fatal(err)
    }
    return caminho
}
```

## Step 5 — Concurrency Tests

For functions using `errgroup` or `sync.Mutex` (like `processAxTableFolder`):

- Use `t.Parallel()` to surface data races.
- Run tests with `-race` flag: `go test -race ./...`
- For goroutine logic, create a directory with multiple fixture files and verify all results are collected without duplication.

## Step 6 — Write the Tests

1. Create `<file>_test.go` adjacent to the source file.
2. Implement tests following patterns from Steps 2–5.
3. Cover:
   - [ ] Happy path (valid input → correct output)
   - [ ] Empty / zero-value input
   - [ ] Malformed input (bad XML, missing file, wrong path)
   - [ ] Boundary: zero files, one file, many files (for directory walkers)
   - [ ] Error propagation: errors are returned/logged, not swallowed silently

## Step 7 — Validate

Run locally and check:

```bash
# Run all tests with race detector
go test -race ./...

# With coverage report
go test -race -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

**Quality checklist:**
- [ ] Each test has a descriptive `nome` / `t.Run` label
- [ ] No `time.Sleep` in tests (use channels or `sync.WaitGroup` instead)
- [ ] Fixture files are in `testdata/`, not hardcoded absolute paths
- [ ] Test file uses `_test` package suffix (or same package only when necessary)
- [ ] `-race` passes with no warnings
- [ ] Tests do not touch real `temp/PackageLocalDirectory/` — use `t.TempDir()`

## After Completion

Tell the user:
- Which functions are now covered and at what depth
- Any remaining untested risk-tier-high functions
- Suggest running `go test -race -count=1 ./...` to confirm
- Propose creating a `go-test-strategy` subagent session for full suite planning if the package has many untested functions

# Effective Go Cheatsheet

Reference: https://go.dev/doc/effective_go

---

## Naming

| What | Rule | Example |
|---|---|---|
| Packages | Short, lowercase, single word | `xml`, `parser`, `usecase` |
| Exported types | PascalCase with doc comment | `type AxTable struct` |
| Unexported vars | camelCase | `maxWorkers`, `resultado` |
| Interfaces (1 method) | Verb + `er` suffix | `Reader`, `Processor`, `Saver` |
| Receivers | 1–2 letter abbreviation, consistent | `func (t *Tabela)` not `func (this *Tabela)` |
| Error vars | `Err` prefix | `var ErrNaoEncontrado = errors.New(...)` |
| Error types | `Error` suffix | `type ParseError struct` |

**Never**: `this`, `self`, `me` as receivers. Never stutter (`parser.ParserConfig` → `parser.Config`).

---

## Error Handling

```go
// ✅ Idiomatic — wrap with context
if err != nil {
    return fmt.Errorf("parsearTabela %q: %w", nome, err)
}

// ❌ Avoid — lost context
if err != nil {
    return err
}

// ❌ Avoid — capitalized error string or trailing punctuation
errors.New("Failed to parse file.")   // wrong
errors.New("failed to parse file")    // ✅ correct
```

**Sentinel errors**: use `errors.Is` / `errors.As` to check; never `==` on wrapped errors.

**When to panic**: only in `init()` or `main()` for truly unrecoverable startup failures. Never in library code.

---

## Interfaces

```go
// ✅ Small, consumer-defined
type Leitor interface {
    Ler(path string) ([]byte, error)
}

// ❌ Too large — violates ISP
type TudoDoSistema interface {
    Ler(...) ...
    Escrever(...) ...
    Deletar(...) ...
    Listar(...) ...
}
```

**Empty interface (`any`)**: avoid as function parameter unless truly generic. Prefer concrete types or constrained generics (Go 1.18+).

**Accept interfaces, return structs** — callers can then store the concrete type if needed.

---

## Goroutines & Concurrency

```go
// ✅ Use errgroup for bounded concurrency with error propagation
g, ctx := errgroup.WithContext(context.Background())
sem := make(chan struct{}, maxWorkers)

for _, item := range items {
    item := item // capture loop variable (pre-Go 1.22)
    sem <- struct{}{}
    g.Go(func() error {
        defer func() { <-sem }()
        return processar(ctx, item)
    })
}
if err := g.Wait(); err != nil { ... }

// ✅ Protect shared state
var mu sync.Mutex
mu.Lock()
resultado = append(resultado, item)
mu.Unlock()

// ✅ Prefer channels for ownership transfer, mutex for shared state
```

**Go 1.22+**: loop variable capture bug is fixed — `item := item` copy is no longer needed.

---

## Package Design

| Rule | Detail |
|---|---|
| Avoid `util`, `common`, `helpers` | Name by what the package *does*, not what it *is* |
| Keep `internal/` for private packages | Prevents accidental external use |
| `init()` only for registration patterns | e.g., `database/sql` driver registration |
| Circular imports = design smell | Introduce a shared `domain` or `types` package |
| `main` package is the wiring layer | Contains no business logic |

---

## Code Smells → Fixes

| Smell | Fix |
|---|---|
| `if err != nil { return err }` chains with no context | Wrap: `fmt.Errorf("operacao: %w", err)` |
| Naked `return` in long functions | Name return values only for short functions; avoid in long ones |
| Deeply nested `if` | Guard clause — return/continue early |
| `var x = T{}` then `x.Field = val` | Use struct literal: `x := T{Field: val}` |
| Ignoring errors: `val, _ := f()` | Handle or explicitly document why it's safe to ignore |
| Large `main()` | Extract `run() error` function; `main()` just calls it and handles exit |

```go
// ✅ Idiomatic main pattern
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "erro: %v\n", err)
        os.Exit(1)
    }
}

func run() error {
    // all real logic here
    return nil
}
```

---

## Clean Code Principles (Go-flavored)

| Principle | Go application |
|---|---|
| Functions do one thing | Max ~20–30 lines; extract if more |
| No side effects in pure functions | Separate query from command |
| No magic numbers | Named constants (`const maxWorkers = 8`) |
| Boy Scout Rule | Leave code cleaner than you found it — fix one thing per PR |
| YAGNI | Don't add interfaces/layers until you have 2+ implementations |
| DRY | But don't over-abstract; duplication is cheaper than wrong abstraction |

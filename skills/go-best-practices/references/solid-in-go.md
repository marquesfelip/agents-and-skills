# SOLID Principles in Go

## S — Single Responsibility Principle (SRP)
**A type or function should have one reason to change.**

❌ Violation — function does parsing, filtering, and I/O:
```go
func processFile(path string) {
    data, _ := os.ReadFile(path)
    // parse + filter + write DB all in one function
}
```

✅ Fix — split into focused units:
```go
func lerArquivo(path string) ([]byte, error)
func parsearTabela(data []byte) (Tabela, error)
func salvarTabela(db DB, t Tabela) error
```

**Go-specific signals of SRP violation:**
- Struct with 5+ unrelated fields
- Function doing I/O *and* business logic
- `main.go` containing domain logic

---

## O — Open/Closed Principle (OCP)
**Open for extension, closed for modification.**

In Go: use interfaces and function parameters instead of switch/type-assert chains.

❌ Violation:
```go
func renderizar(formato string, dados Dados) string {
    switch formato {
    case "json": ...
    case "xml": ...
    }
}
```

✅ Fix:
```go
type Renderer interface {
    Render(dados Dados) string
}
// Add JSON, XML, CSV by implementing Renderer — no change to existing code
```

---

## L — Liskov Substitution Principle (LSP)
**Subtypes must be substitutable for their base type.**

In Go: types implementing an interface must fully honor its contract (no panics, no silent no-ops).

❌ Violation — implementing interface but silently ignoring behavior:
```go
func (n NoOpRepo) Salvar(t Tabela) error { return nil } // never actually saves
```

✅ Fix — make the behavior explicit in the type name or document clearly; use a `testutil.FakeRepo` in tests only.

**Go-specific LSP checks:**
- Interface implementers must not return errors more broadly than the interface contract documents
- Pointer vs value receivers must be consistent per type

---

## I — Interface Segregation Principle (ISP)
**Clients should not depend on methods they don't use.**

Go naturally favors small interfaces. The standard library defines `io.Reader` (1 method) not `io.EverythingReader`.

❌ Violation — fat interface:
```go
type Repositorio interface {
    BuscarTabela(id int) (Tabela, error)
    SalvarTabela(t Tabela) error
    DeletarTabela(id int) error
    ListarCampos(tabelaID int) ([]Campo, error)
    SalvarCampo(c Campo) error
    // ...7 more methods
}
```

✅ Fix — split by consumer:
```go
type TabelaLeitor interface {
    BuscarTabela(id int) (Tabela, error)
}
type TabelaEscritor interface {
    SalvarTabela(t Tabela) error
}
```

**Rule of thumb:** If a consumer only calls 2 of your 7 interface methods, the interface needs splitting.

---

## D — Dependency Inversion Principle (DIP)
**Depend on abstractions, not concretions. Define interfaces at the consumer side.**

❌ Violation — importing a concrete DB driver in a use-case:
```go
import "github.com/lib/pq" // concretion in business logic layer

type ServicoTabela struct {
    db *sql.DB // depends on infrastructure
}
```

✅ Fix — define a port (interface) in the use-case layer:
```go
// use-case layer owns this interface
type TabelaRepo interface {
    Salvar(t Tabela) error
    Buscar(nome string) (Tabela, error)
}

type ServicoTabela struct {
    repo TabelaRepo // injected; decoupled from driver
}
```

The concrete `pq` implementation lives in `infrastructure/` and is wired at `main.go`.

**Go idiom:** Interfaces belong in the package that *uses* them, not the package that *implements* them.

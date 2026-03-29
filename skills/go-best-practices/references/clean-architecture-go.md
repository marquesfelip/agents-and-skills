# Clean Architecture in Go

## Core Concept

Dependencies flow **inward only**. The domain never knows about HTTP, databases, or file formats.

```
┌─────────────────────────────────────┐
│  Infrastructure (DB, HTTP, XML I/O) │
│  ┌───────────────────────────────┐  │
│  │  Interface / Adapter (CLI,    │  │
│  │  handlers, parsers)           │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Use Case / Application │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │  Domain / Entity  │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Rule**: an import from an outer ring into an inner ring is always a violation.

---

## Typical Go Package Layout

```
cmd/
  main.go                  # wire everything; only place concrete types meet
internal/
  domain/                  # entities, value objects, domain errors — no external imports
  usecase/                 # orchestration; depends only on domain + port interfaces
  port/                    # interface definitions (ports) owned by use-case layer
  adapter/
    xmlparser/             # reads XML, converts to domain types
    database/              # implements port.TabelaRepo with a real driver
    http/                  # HTTP handlers (if applicable)
```

---

## Ports & Adapters (Hexagonal Architecture)

**Ports** = interfaces defined by the application core (use-case layer).
**Adapters** = concrete implementations in the outer ring.

```go
// port/repo.go — owned by use-case layer
package port

type TabelaRepo interface {
    Salvar(t domain.Tabela) error
    Buscar(nome string) (domain.Tabela, error)
}

// adapter/database/postgres_repo.go — outer ring
package database

type PostgresTabelaRepo struct { db *sql.DB }

func (r *PostgresTabelaRepo) Salvar(t domain.Tabela) error { ... }
func (r *PostgresTabelaRepo) Buscar(nome string) (domain.Tabela, error) { ... }
```

Wire in `main.go`:
```go
repo := database.NewPostgresTabelaRepo(db)
svc  := usecase.NewServicoTabela(repo)  // receives port.TabelaRepo
```

---

## Common Violations

| Violation | Symptom | Fix |
|---|---|---|
| Domain importing infrastructure | `domain/tabela.go` imports `database/sql` | Move DB logic to adapter layer |
| Use-case importing HTTP | `usecase/` imports `net/http` or Echo | Use request/response structs in use-case, not HTTP types |
| Adapter containing business logic | Adapter decides pricing, validation rules | Move logic to use-case or domain |
| Constructors in `main.go` spread everywhere | Hard to test or swap | Use a `wire.go` or `app.go` assembler |
| Circular imports | Package A imports B imports A | Introduce a shared `domain` or `port` package |

---

## Dependency Injection in Go

Go has no DI container by default. Use **constructor injection**:

```go
// use-case receives its dependencies via constructor
func NovoServicoTabela(repo port.TabelaRepo, log port.Logger) *ServicoTabela {
    return &ServicoTabela{repo: repo, log: log}
}
```

For large apps, consider `google/wire` for compile-time DI graph generation.

---

## This Project's Architecture Context (d365-fo-db-diagram)

| Current component | Layer | Status |
|---|---|---|
| `entity/entity.go` | Domain (structs) | ✅ No external deps |
| `main.go` (XML reader) | Adapter (XML parser) + Infrastructure (file I/O) | ⚠️ Mixed with orchestration |
| DB writer | Adapter (Infrastructure) | 🔲 Planned |
| Diagram renderer | Interface adapter | 🔲 Planned |

**Recommended next step**: extract `readAxTableXML` and `processAxTableFolder` into an `adapter/xmlparser` package so `main.go` becomes a thin orchestrator.

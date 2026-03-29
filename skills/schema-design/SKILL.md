---
name: schema-design
description: >
  Relational schema modeling, normalization, and domain alignment.
  Use when: schema design, relational schema modeling, database design, normalization, 1NF 2NF 3NF BCNF,
  ER diagram, entity relationship, table design, column design, data types, primary key design,
  foreign key design, surrogate key, natural key, composite key, domain modeling database,
  denormalization, schema review, schema audit, multi-tenant schema, polymorphic association,
  soft delete, temporal table, history table, schema convention, naming convention database.
argument-hint: >
  Describe the domain to model (e.g., "e-commerce with orders, products, customers, and payments"),
  and specify any constraints such as multi-tenancy, soft deletes, audit trail requirements, or
  the target database engine (PostgreSQL, MySQL, SQL Server).
---

# Schema Design Specialist

## When to Use

Invoke this skill when you need to:
- Design or review a relational database schema from domain requirements
- Apply normalization rules and identify denormalization trade-offs
- Choose appropriate primary key strategies (surrogate vs. natural vs. composite)
- Align the schema to the domain model without introducing coupling
- Design for specific cross-cutting concerns: multi-tenancy, soft deletes, audit history, temporal data
- Enforce naming conventions and structural consistency

---

## Step 1 — Map Entities from the Domain

Extract entities, attributes, and relationships from the domain:

1. List all **nouns** from the domain description → candidate entities (tables)
2. List **verbs** between entities → relationships (foreign keys, join tables)
3. Identify **attributes** per entity → candidate columns
4. Classify relationships: one-to-one, one-to-many, many-to-many

Checklist:
- [ ] Every entity maps to exactly one table — no entity split across multiple tables without justification
- [ ] Many-to-many relationships resolved via an explicit join/association table
- [ ] Join table has its own primary key and may carry relationship attributes (e.g., `quantity` on an `order_item`)
- [ ] Hierarchical relationships (self-referential) represented with a `parent_id` column or closure table

---

## Step 2 — Apply Normalization

Normalize to **3NF (Third Normal Form)** as the default target:

| Normal Form | Rule | Violation Example |
|---|---|---|
| **1NF** | Atomic values; no repeating groups | `phone1, phone2, phone3` columns |
| **2NF** | No partial dependency on composite PK | `product_name` depending only on `product_id` in an order-product table |
| **3NF** | No transitive dependency | `zip_code → city → state` stored on the same row |
| **BCNF** | Every determinant is a candidate key | Applied when 3NF still leaves anomalies |

**Justified Denormalization:**
- Materialized summary columns (e.g., `order.total_amount`) — update with trigger or application logic
- Redundant columns to avoid expensive joins on hot read paths — document explicitly
- JSON/JSONB columns for flexible, schema-less attributes — use sparingly

Checklist:
- [ ] No repeating column groups (arrays-as-columns) — use a child table instead
- [ ] Every non-key column is fully dependent on the entire primary key
- [ ] No transitive dependencies — extract to lookup/reference tables
- [ ] Any denormalization is explicitly documented with the reason and the update strategy

---

## Step 3 — Primary Key Strategy

| Strategy | When to Use | Trade-offs |
|---|---|---|
| **Surrogate (UUID v7 / ULID)** | Default for most entities | Globally unique, safe for distributed inserts; sortable variants preserve B-tree efficiency |
| **Surrogate (BIGSERIAL / IDENTITY)** | Single-DB, high-volume inserts | Compact, fast index; not portable across DB instances |
| **Natural key** | Immutable, universally unique real-world identifier (ISO country code, ISIN) | No extra column; fragile if "immutable" attribute changes |
| **Composite key** | Join tables or when the combination is the identity | No surrogate needed; queries require all parts |

Checklist:
- [ ] Every table has a single-column primary key (surrogate preferred)
- [ ] UUID v7 or ULID used instead of UUID v4 when insert order matters for index performance
- [ ] Natural keys used only when the value is truly immutable and universally unique
- [ ] Join tables may use composite PK of the two FK columns plus a surrogate if needed

---

## Step 4 — Foreign Keys and Referential Integrity

Checklist:
- [ ] Every relationship enforced by a `FOREIGN KEY` constraint — not just application logic
- [ ] `ON DELETE` behavior explicitly set per relationship:
  - `CASCADE` — child records follow parent deletion (use carefully)
  - `RESTRICT` / `NO ACTION` — block parent deletion if children exist (safest default)
  - `SET NULL` — child FK set to NULL when parent deleted (use when relationship is optional)
- [ ] FK columns indexed — prevents full table scan on parent-side queries
- [ ] Circular FK references avoided or resolved with deferred constraints
- [ ] Soft-delete FK targets: decide if soft-deleted parents still satisfy FK constraint

---

## Step 5 — Data Types and Constraints

Checklist:
- [ ] Use the **narrowest correct type**: `SMALLINT` over `INT` where range is known, `DATE` over `TIMESTAMP` for date-only values
- [ ] Monetary values stored as `NUMERIC(precision, 2)` — never `FLOAT` or `DOUBLE`
- [ ] Timestamps stored as `TIMESTAMPTZ` (UTC) — never `TIMESTAMP WITHOUT TIME ZONE` unless the DB is UTC-only
- [ ] Booleans stored as `BOOLEAN` — not `TINYINT(1)` or `CHAR(1)`
- [ ] Enum-like values: use a `CHECK` constraint or a reference/lookup table — not magic strings
- [ ] `NOT NULL` applied by default; `NULL` allowed only when absence of value is semantically meaningful
- [ ] `CHECK` constraints used for domain rules expressible at column level (e.g., `amount > 0`)
- [ ] `UNIQUE` constraints applied to natural candidate keys (e.g., `email`, `slug`, `external_id`)

---

## Step 6 — Cross-Cutting Schema Patterns

**Soft Deletes:**
```sql
deleted_at TIMESTAMPTZ NULL  -- NULL = active; non-NULL = deleted
```
- [ ] Views or filtered indexes used to hide deleted records from application queries
- [ ] `UNIQUE` constraints scope-aware — unique among active records only

**Audit Columns:**
```sql
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
created_by  UUID REFERENCES users(id),
updated_by  UUID REFERENCES users(id)
```
- [ ] `updated_at` maintained by trigger or ORM hook — not manual

**Multi-Tenancy:**
```sql
tenant_id UUID NOT NULL REFERENCES tenants(id)
```
- [ ] `tenant_id` present on every tenant-scoped table
- [ ] Composite unique constraints include `tenant_id`
- [ ] Row-Level Security (RLS) or application-layer scoping enforced

**Temporal / History Tables:**
- [ ] `valid_from` / `valid_to` columns for bitemporal modeling
- [ ] History table stores previous versions on update/delete via trigger

---

## Step 7 — Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Table | `snake_case`, plural | `order_items` |
| Column | `snake_case`, singular | `created_at` |
| Primary key | `id` | `id` |
| Foreign key | `<referenced_table_singular>_id` | `customer_id` |
| Index | `idx_<table>_<columns>` | `idx_orders_customer_id` |
| Unique constraint | `uq_<table>_<columns>` | `uq_users_email` |
| Check constraint | `chk_<table>_<rule>` | `chk_products_price_positive` |

Checklist:
- [ ] Naming convention documented and consistently applied
- [ ] No reserved words used as column/table names
- [ ] Boolean columns prefixed `is_` or `has_` for readability

---

## Output Report

### Critical
- No primary key on a table — updates and deletes are unsafe
- Monetary values stored as `FLOAT` — rounding errors in financial calculations
- Referential integrity not enforced by FK constraints — orphaned records accumulate
- Many-to-many relationship stored as a delimited string column — unparseable by SQL

### High
- Repeating column groups (`phone1`, `phone2`, `phone3`) — 1NF violation
- Transitive dependencies not extracted — update anomalies possible
- FK columns not indexed — parent-side queries perform full table scans
- No `tenant_id` scoping on multi-tenant tables — cross-tenant data leak risk

### Medium
- UUID v4 used as PK on high-insert tables — index fragmentation over time
- `NULL` allowed on columns that should always have a value
- `TIMESTAMP WITHOUT TIME ZONE` used — timezone handling bugs in multi-region systems
- Naming conventions inconsistent across tables

### Low
- Audit columns (`created_at`, `updated_at`) missing from some tables
- `CHECK` constraints not used for expressible domain rules
- No soft delete strategy documented — hard deletes break audit trail

### Passed
- Schema normalized to 3NF with justified and documented denormalization
- All FKs declared with explicit `ON DELETE` behavior and backing indexes
- Monetary values use `NUMERIC`, timestamps use `TIMESTAMPTZ`
- Naming conventions applied consistently across all tables
- Multi-tenancy, soft delete, and audit requirements addressed

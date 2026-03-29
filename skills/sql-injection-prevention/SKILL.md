---
name: sql-injection-prevention
description: 'Safe query construction, parameterized queries, ORM safety, and database injection defenses. Use when: SQL injection, SQLi prevention, parameterized queries, prepared statements, ORM security, query builder safety, NoSQL injection, LDAP injection, database injection, raw SQL review, dynamic query, stored procedure security, database security, injection defense.'
argument-hint: 'Database query code, repository layer, or ORM usage to review — or describe the data access pattern being implemented'
---

# SQL Injection Prevention Specialist

## When to Use
- Reviewing data access code (raw SQL, query builders, ORMs) for injection vulnerabilities
- Auditing dynamic query construction where user input influences queries
- Designing safe data access patterns for a new service or repository layer
- Reviewing stored procedures, views, or database-level logic
- Extending injection defense to NoSQL, LDAP, or search engine queries

---

## Step 1 — Understand the Data Access Layer

Before reviewing, identify:
- Database engine: PostgreSQL, MySQL, MSSQL, SQLite, MongoDB, Cassandra, LDAP, Elasticsearch
- Query construction method: raw SQL strings, parameterized/prepared statements, query builder, ORM, stored procedures
- Sources of user-controlled input flowing into queries: HTTP params, body fields, headers, file content, third-party data
- Whether the DB user has minimal privileges or over-privileged access

---

## Step 2 — Identify Dangerous Patterns

### Immediate Red Flags (any of these = Critical finding)

```
# String concatenation with user input — all languages
query = "SELECT * FROM users WHERE name = '" + username + "'"
query = f"SELECT * FROM orders WHERE id = {order_id}"
query = "DELETE FROM records WHERE user_id = " + req.params.id
db.raw("SELECT * FROM products WHERE category = '" + category + "'")
```

```
# Format/template strings with input
fmt.Sprintf("SELECT * FROM table WHERE col = '%s'", input)
String.format("SELECT * FROM users WHERE email = '%s'", email)
```

```
# Dynamic ORDER BY / column names (ORM bypass)
db.order(params[:sort_field])        # Rails
query.order_by(request.query.sort)   # general
```

### Injection-Risky Patterns by Category

| Category | Dangerous pattern | Safe alternative |
|---|---|---|
| Raw SQL | String concat or format | Parameterized query |
| ORM raw query | `.raw()`, `.execute()` with input | ORM-safe query methods |
| Dynamic column/table name | User input as identifier | Validate against allowlist |
| Dynamic ORDER BY | User input as sort field | Map to enum of allowed columns |
| LIKE clause | `LIKE '%' + input + '%'` | Parameterized with `?` / `$1` |
| IN clause | String-joined values | Bind each value as a separate param |
| Stored procedure | String-building inside proc | Use parameterized stored procedures |

---

## Step 3 — Safe Query Patterns

### Parameterized Queries (by language)

**Go (database/sql)**
```go
// GOOD
row := db.QueryRow("SELECT id FROM users WHERE email = $1 AND active = $2", email, true)

// BAD
row := db.QueryRow("SELECT id FROM users WHERE email = '" + email + "'")
```

**Python (psycopg2 / DB-API)**
```python
# GOOD
cursor.execute("SELECT * FROM orders WHERE user_id = %s AND status = %s", (user_id, status))

# BAD
cursor.execute(f"SELECT * FROM orders WHERE user_id = {user_id}")
```

**JavaScript (pg / mysql2)**
```js
// GOOD
const res = await pool.query('SELECT * FROM users WHERE id = $1', [userId])

// BAD
const res = await pool.query(`SELECT * FROM users WHERE id = ${userId}`)
```

**Java (JDBC)**
```java
// GOOD
PreparedStatement ps = conn.prepareStatement("SELECT * FROM accounts WHERE id = ?");
ps.setInt(1, accountId);

// BAD
Statement st = conn.createStatement();
st.executeQuery("SELECT * FROM accounts WHERE id = " + accountId);
```

---

### ORM Safety Rules

| ORM | Safe | Unsafe |
|---|---|---|
| Hibernate/JPA | `createQuery("FROM User WHERE email = :e").setParameter("e", email)` | `createQuery("FROM User WHERE email = '" + email + "'")` |
| ActiveRecord | `User.where(email: params[:email])` | `User.where("email = '#{params[:email]}'")` |
| SQLAlchemy | `session.query(User).filter(User.email == email)` | `session.execute(f"SELECT * FROM users WHERE email = '{email}'"`) |
| GORM | `db.Where("email = ?", email).Find(&users)` | `db.Where("email = '" + email + "'").Find(&users)` |
| Prisma | `prisma.user.findMany({ where: { email } })` | `prisma.$queryRaw(Prisma.sql\`SELECT * FROM users WHERE email = '${email}'\`)` without template literal |
| TypeORM | `.where("user.email = :email", { email })` | `.where("user.email = '" + email + "'")` |

---

### Dynamic Identifiers (column names, table names, ORDER BY)

Column and table names **cannot** be parameterized — use an allowlist:

```go
// GOOD — allowlist for dynamic ORDER BY
allowed := map[string]bool{"name": true, "created_at": true, "amount": true}
if !allowed[sortField] {
    return errors.New("invalid sort field")
}
query := fmt.Sprintf("SELECT * FROM orders ORDER BY %s %s", sortField, direction)

// BAD
query := "SELECT * FROM orders ORDER BY " + userInput
```

---

## Step 4 — NoSQL & Non-Relational Injection

Injection is not limited to SQL:

### MongoDB
```js
// BAD — user controls the operator
db.users.find({ username: req.body.username, password: req.body.password })
// Attacker sends: { "password": { "$gt": "" } }

// GOOD — enforce types; sanitize operators
if (typeof req.body.username !== 'string') return 400
db.users.find({ username: req.body.username, password: req.body.password })
// Or better: use bcrypt comparison, never store plaintext passwords
```

### LDAP
```
// BAD — LDAP metacharacters not escaped
filter = "(&(uid=" + username + ")(userPassword=" + password + "))"
// Attacker: username = *)(&(uid=*)

// GOOD — escape metacharacters
filter = "(&(uid=" + ldapEscape(username) + ")(userPassword=" + ldapEscape(password) + "))"
```

### Elasticsearch / search engines
- Validate field names against an allowlist.
- Disable `_source` filtering by user input.
- Use the query DSL with typed fields, not query strings with user input.

---

## Step 5 — Database Privilege Hardening

Defense-in-depth: even if injection occurs, limit blast radius.

| Principle | Implementation |
|---|---|
| Least privilege | App DB user: `SELECT, INSERT, UPDATE, DELETE` on specific tables only. Never `DROP`, `CREATE`, `ALTER`, `GRANT` |
| Read-only users | Reporting/analytics services use a read-only DB user |
| Schema separation | App tables in one schema; system tables inaccessible to app user |
| No `xp_cmdshell` | Disabled in MSSQL; no shell execution from DB |
| Row-level security | Use DB-native row-level security where multi-tenancy requires it |
| Stored procedure isolation | App calls only named stored procs; cannot issue arbitrary SQL |

---

## Step 6 — Detection & Testing

Test for SQLi during development:
1. Input the following into every string field and observe behavior: `'`, `''`, `"`, `--`, `; DROP TABLE`, `1 OR 1=1`
2. Use automated tools: `sqlmap` (authorized testing only), OWASP ZAP, Burp Suite.
3. Enable DB query logging in dev/staging and review for unparameterized queries.
4. Use static analysis: `gosec`, `semgrep` with SQL injection rules, CodeQL.

---

## Step 7 — Output Report

```
## SQL Injection Review: <service/file>

### Critical
- /api/products?search= constructs: `"SELECT * FROM products WHERE name LIKE '%" + search + "%'"`
  No parameterization — direct SQLi vulnerability
  Fix: use parameterized LIKE: `WHERE name LIKE $1` with value `"%" + search + "%"`

- Dynamic ORDER BY: `ORDER BY ` + req.query.sort — no allowlist validation
  Fix: validate sort against {"name", "price", "created_at"}; default to "created_at"

### High
- GORM raw query in reports.go:38: `db.Raw("SELECT * FROM orders WHERE user_id = " + userID)`
  Fix: `db.Raw("SELECT * FROM orders WHERE user_id = ?", userID)`

### Medium
- App DB user has GRANT privilege — excessive permissions
  Fix: restrict to SELECT, INSERT, UPDATE, DELETE on application tables only

### Passed
- All login queries use parameterized statements ✓
- ORM used for all CRUD operations in user service ✓
- MongoDB queries use typed field comparisons ✓
```

---
name: authorization-models
description: 'RBAC, ABAC, permission boundaries, tenant-aware authorization, and least-privilege enforcement. Use when: authorization design, RBAC implementation, role-based access control, ABAC, attribute-based access control, permission model, access control review, multi-tenant authorization, tenant isolation, least privilege, permission boundaries, policy enforcement, authorization vulnerability, missing authorization check, privilege escalation.'
argument-hint: 'Describe the authorization model or provide the access-control code/file to review'
---

# Authorization Models Specialist

## When to Use
- Designing a permission system for a new application or service
- Reviewing existing access control code for missing or incorrect checks
- Modeling multi-tenant authorization (tenant isolation, cross-tenant access)
- Choosing between RBAC, ABAC, or a hybrid model
- Enforcing least-privilege across services, APIs, and database access

---

## Step 1 — Understand the Authorization Domain

Before modeling, gather:
- Who are the principals? (users, service accounts, API clients, tenants)
- What are the resources? (records, files, endpoints, features, tenant data)
- What actions exist? (read, write, delete, approve, export, admin)
- Is this single-tenant or multi-tenant?
- Does the system need dynamic policies (context-aware) or static roles?

---

## Step 2 — Choose the Right Model

| Model | Strengths | Use when |
|---|---|---|
| RBAC (Role-Based) | Simple, auditable, easy to reason about | Clear roles with stable permissions; most SaaS apps |
| ABAC (Attribute-Based) | Fine-grained, context-aware, flexible | Complex rules involving resource attributes (owner, department, location) |
| ReBAC (Relationship-Based) | Natural for social/graph data | Permissions follow ownership/relationship paths (e.g., Google Zanzibar) |
| ACL (Access Control List) | Per-resource granularity | File systems, shared documents |
| Hybrid RBAC+ABAC | Roles define coarse gates; attributes refine | Most real-world systems beyond simple CRUD |

---

## Step 3 — RBAC Design Principles

```
User → [has] Role(s) → [grants] Permission(s) → [on] Resource(s)
```

Rules:
- Roles should be **job-scoped**, not user-specific (avoid "admin-for-john" roles).
- Permissions should be **verb-noun pairs**: `invoice:read`, `user:delete`, `report:export`.
- Prefer many small roles over few large roles — compose them for flexibility.
- Never grant roles based on user-supplied input without validation.
- Implement **role hierarchy** explicitly if needed; don't rely on implicit inheritance.

| Anti-pattern | Problem | Fix |
|---|---|---|
| God `admin` role with all permissions | Violates least privilege | Break into `billing-admin`, `user-admin`, `content-admin` |
| Checking role name strings in code | Fragile, typo-prone | Use permission constants or enums |
| Role stored in JWT without server validation | JWT can be forged or stale | Always verify role/permission server-side from authoritative store |

---

## Step 4 — ABAC Design Principles

Policy structure: `Subject + Action + Resource + Environment → Allow/Deny`

Example policy rule:
```
Allow invoice:approve IF
  subject.role == "manager" AND
  resource.department == subject.department AND
  resource.amount < subject.approval_limit AND
  environment.time WITHIN business_hours
```

Rules:
- Define policies in a central policy engine (OPA, Casbin, Cedar) — not scattered in code.
- Keep attribute lookups consistent: resource attributes from the DB, not from input.
- Audit all policy decisions with subject, resource, action, and outcome.
- Test policies with both positive and negative cases before deployment.

---

## Step 5 — Multi-Tenant Authorization

Every resource access in a multi-tenant system must enforce **tenant isolation**:

```
# BAD — no tenant check
SELECT * FROM invoices WHERE id = ?

# GOOD — tenant-scoped
SELECT * FROM invoices WHERE id = ? AND tenant_id = ?
```

Checklist:
- [ ] Every DB query that touches tenant data includes a `tenant_id` filter.
- [ ] Tenant ID comes from the **authenticated session/token**, never from user input.
- [ ] Cross-tenant operations (admin, support) require an explicit elevated scope with audit logging.
- [ ] Bulk operations (exports, reports) scope data to the authenticated tenant.
- [ ] Service-to-service calls include tenant context propagated via verified headers.

| Isolation level | When to use |
|---|---|
| Row-level (shared schema + tenant_id column) | Most SaaS; simpler but requires careful query discipline |
| Schema-level (one schema per tenant) | Medium isolation; useful for compliance requirements |
| Database-level (one DB per tenant) | Strongest isolation; high operational cost |

---

## Step 6 — Least Privilege Enforcement

| Layer | Principle | Example |
|---|---|---|
| API endpoints | Each route grants minimum required permissions | Read-only route uses a read-only DB user |
| Service accounts | Scoped to specific resources and actions | Queue consumer reads from one topic only |
| Database | Per-query role with only needed table/column access | Reporting role has SELECT on aggregated views only |
| Infrastructure | IAM policies deny by default, allow explicitly | Lambda has only GetObject on specific S3 prefix |
| Third-party tokens | Request minimum OAuth scopes | Ask for `email` not `profile` if only email is needed |

---

## Step 7 — Authorization Vulnerability Review

| Vulnerability | Check | Fix |
|---|---|---|
| Missing object-level check | Can user A access user B's record by changing the ID? | Re-verify ownership on every resource fetch |
| Privilege escalation | Can a `viewer` call an `admin` endpoint? | Enforce role/permission on every function, not just routing |
| Tenant data leak | Can tenant A read tenant B's data? | Add `tenant_id` to every data access query |
| Role from client input | Is role/permission derived from the request? | Always derive from authenticated identity, not input |
| Stale permissions | Can revoked roles still access resources? | Validate permissions at request time, not at token issuance time |

---

## Step 8 — Output Report

```
## Authorization Review: <system/module>

### Model In Use
RBAC with implicit tenant scoping

### Critical
- GET /api/documents/:id — no tenant_id check; tenant A can access tenant B's documents
  Fix: add `WHERE id = ? AND tenant_id = session.tenantID`

### High
- Role stored in JWT and trusted without server lookup
  Fix: fetch role from DB on each request or use short-lived tokens with forced refresh

### Medium
- Single "admin" role controls billing, user management, and content moderation
  Fix: split into `billing:admin`, `users:admin`, `content:admin`

### Passed
- Permission checks on all write endpoints ✓
- Deny-by-default on unknown roles ✓
```

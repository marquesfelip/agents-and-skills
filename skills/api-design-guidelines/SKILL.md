---
name: api-design-guidelines
description: 'REST and API contract design, versioning, consistency, and backward compatibility. Use when: API design, REST API design, API contract, API versioning, backward compatibility, API consistency, HTTP methods, status codes, API naming conventions, pagination design, error response format, API evolution, breaking change, OpenAPI, Swagger, API review.'
argument-hint: 'API endpoint(s) or OpenAPI spec to review — or describe the resource and operations being designed'
---

# API Design Guidelines Specialist

## When to Use
- Designing a new REST API from scratch
- Reviewing an API for consistency, correctness, and consumer-friendliness
- Auditing an existing API for breaking changes before a release
- Establishing API design conventions for a team
- Evaluating versioning strategy for an evolving API

---

## Step 1 — Resource Modeling

### Resource Design Rules

```
# Resources are nouns, not verbs
✓ /orders              (correct: noun)
✗ /getOrders           (wrong: verb in path)
✗ /order/list          (wrong: action in path)

# Collections are plural
✓ /orders              ✓ /users              ✓ /invoices
✗ /order               ✗ /user               ✗ /invoice

# Sub-resources express ownership relationships
✓ /orders/{id}/items          (items belong to an order)
✓ /users/{id}/addresses       (addresses belong to a user)
✗ /getOrderItems?orderId=123  (query param used where path is cleaner)

# Actions that don't map to CRUD: use sub-resource nouns where possible
✓ POST /orders/{id}/cancellation    (creates a cancellation)
✓ POST /orders/{id}/confirmation    (creates a confirmation)
✓ POST /sessions                    (creates a session = login)
✗ POST /orders/{id}/cancel          (verb in path)
```

### URL Structure Rules

- [ ] All lowercase, hyphen-separated (`/shipping-addresses`, not `/shippingAddresses`)
- [ ] No trailing slashes
- [ ] IDs are path parameters, not query parameters: `/orders/{id}` not `/orders?id=...`
- [ ] Max 3 levels of nesting to avoid over-coupled resources: `/a/{id}/b/{id}/c` is the limit

---

## Step 2 — HTTP Method Semantics

| Method | Semantics | Safe? | Idempotent? | Body? |
|---|---|---|---|---|
| GET | Retrieve resource(s) | Yes | Yes | No |
| POST | Create resource / trigger action | No | No | Yes |
| PUT | Replace entire resource | No | Yes | Yes |
| PATCH | Partial update | No | No | Yes |
| DELETE | Remove resource | No | Yes | No |
| HEAD | GET without body (existence/metadata check) | Yes | Yes | No |

### Common Method Mistakes

```
# GET must not have side effects
✗ GET /orders/{id}/confirm      (should be POST — has side effect)
✓ POST /orders/{id}/confirmation

# PUT replaces the whole resource — must include all fields
✗ PUT /users/{id} { "email": "new@x.com" }   (partial update — use PATCH)
✓ PATCH /users/{id} { "email": "new@x.com" }

# DELETE is idempotent — second call on same resource returns 404, not 500
✓ DELETE /orders/{id} → 204 first call
✓ DELETE /orders/{id} → 404 second call (already deleted — correct)
✗ DELETE /orders/{id} → 500 second call (should not error — must be idempotent)
```

---

## Step 3 — HTTP Status Codes

### Reference Table

| Code | Meaning | When to use |
|---|---|---|
| 200 OK | Success with body | GET, PUT, PATCH success |
| 201 Created | Resource created | POST that creates a resource |
| 202 Accepted | Async operation started | Background job triggered |
| 204 No Content | Success without body | DELETE, PATCH with no return value |
| 400 Bad Request | Invalid input | Validation errors (shape/format) |
| 401 Unauthorized | Not authenticated | No or invalid credentials |
| 403 Forbidden | Not authorized | Authenticated but lacks permission |
| 404 Not Found | Resource doesn't exist | ID not found (never for auth failures) |
| 409 Conflict | State conflict | Duplicate creation, invalid state transition |
| 422 Unprocessable Entity | Semantic validation error | Valid JSON but business rule violated |
| 429 Too Many Requests | Rate limited | Include `Retry-After` header |
| 500 Internal Server Error | Unexpected failure | Only for truly unexpected errors |
| 503 Service Unavailable | Temporarily down | Include `Retry-After` header |

### Common Code Mistakes

```
✗ 200 with { "success": false }    — use proper status codes, not envelope error flags
✗ 404 for "wrong password"         — use 401; 404 exposes that the account does not exist
✗ 500 for validation errors        — use 400 or 422
✗ 200 for async operations         — use 202 Accepted + Location header to poll status
```

---

## Step 4 — Request and Response Schema Design

### Consistent Error Response Format

All error responses must follow the same schema:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Product SKU-123 has only 2 units in stock",
    "details": [
      {
        "field": "items[0].quantity",
        "code": "MAX_EXCEEDED",
        "message": "Requested 5, only 2 available"
      }
    ],
    "request_id": "req_abc123"
  }
}
```

```json
// Validation error (400/422) — always include field-level details
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "Not a valid email address" },
      { "field": "quantity", "code": "MIN_VALUE", "message": "Must be at least 1" }
    ],
    "request_id": "req_xyz789"
  }
}
```

### Response Envelope Rules

- [ ] Success responses return the resource directly (or a wrapper with `data` + `meta`) — no `success: true` flag
- [ ] Error responses always follow the same error schema — never ad hoc
- [ ] Lists return `{ "data": [...], "pagination": { ... } }` — never a bare array (future-proof)
- [ ] Timestamps always in ISO 8601 UTC with timezone: `"2024-03-15T14:32:01.342Z"`
- [ ] IDs always strings (never integers in JSON — JavaScript loses precision for large int64)
- [ ] Amounts: use string-encoded decimal or integer cents — never float for money

---

## Step 5 — Pagination Design

```json
// Cursor-based pagination (preferred for real-time data or large datasets)
GET /orders?cursor=eyJpZCI6IjEyMyJ9&limit=20

{
  "data": [...],
  "pagination": {
    "limit": 20,
    "has_more": true,
    "next_cursor": "eyJpZCI6IjE0MyJ9",
    "prev_cursor": "eyJpZCI6IjEwMyJ9"
  }
}

// Offset-based pagination (acceptable for small stable datasets)
GET /orders?page=3&per_page=20

{
  "data": [...],
  "pagination": {
    "page": 3,
    "per_page": 20,
    "total_pages": 15,
    "total_count": 298
  }
}
```

| Consideration | Cursor-based | Offset-based |
|---|---|---|
| Consistent under inserts/deletes | Yes | No (page drift) |
| Random access to page N | No | Yes |
| Performance on large tables | Excellent (index seek) | Poor (OFFSET scan) |
| Use case | Feeds, timelines, large datasets | Admin grids, small stable lists |

---

## Step 6 — API Versioning Strategy

### When to Version

Version when a change is **breaking**. A change is breaking if an existing client will fail without modification.

| Breaking change | Non-breaking change |
|---|---|
| Removing a field from response | Adding a new optional field to response |
| Renaming a field | Adding a new optional query parameter |
| Changing field type (`string` → `int`) | Adding a new endpoint |
| Changing required → optional (request) | Adding a new enum value (with `unknown` handling) |
| Changing status code for existing behavior | Changing error message text |
| Removing an endpoint | Adding new optional request fields |

### Versioning Strategies

```
# URL path versioning (most common, highly explicit)
/v1/orders
/v2/orders

# Header versioning (cleaner URLs; harder to test in browser)
Accept: application/vnd.myapi.v2+json
API-Version: 2024-03-15

# Query parameter versioning (avoid — conflates versioning with filtering)
/orders?api_version=2
```

**Recommended:** URL path versioning (`/v1/`, `/v2/`) for public APIs — explicit, cacheable, easy to route.

### Version Lifecycle Rules

- [ ] Non-breaking changes deployed to current version without bumping
- [ ] Breaking changes → new version; old version maintained for minimum N months (document SLA)
- [ ] Deprecation communicated via `Deprecation` and `Sunset` headers:
  ```
  Deprecation: Sat, 01 Jan 2025 00:00:00 GMT
  Sunset: Mon, 01 Jul 2025 00:00:00 GMT
  Link: <https://docs.api.com/migration-v2>; rel="successor-version"
  ```
- [ ] Old versions monitored for traffic before decommissioning

---

## Step 7 — Backward Compatibility Checklist

Before releasing any API change:

- [ ] No fields removed from any response object
- [ ] No fields renamed in request or response
- [ ] No field types changed (even "widening" like `int` → `string` is breaking for some clients)
- [ ] No required fields added to request bodies (only optional fields may be added)
- [ ] No enum values removed (adding is acceptable if consumers handle unknown gracefully)
- [ ] No status codes changed for existing operations
- [ ] No URL paths changed (redirect old → new if unavoidable)
- [ ] Consumer-driven contract tests pass against the change

---

## Output Report

```
## API Design Review: <service/API>

### Critical
- DELETE /orders/{id} returns 500 when called a second time (not idempotent)
  Causes retry storms — clients retry on 500; each retry hits "already deleted" path
  Fix: return 404 for already-deleted resources; never 500 for expected states

- POST /users/deactivate violates REST resource modeling — verb in path
  Inconsistent with resource-oriented design; confusing for API consumers
  Fix: POST /users/{id}/deactivation (creates a deactivation resource)

### High
- Error responses have inconsistent schema: auth errors return {"msg": "..."}, validation errors return {"errors": [...]}
  Clients must implement multiple error parsers
  Fix: standardize all errors to {"error": {"code": ..., "message": ..., "details": [...]}}

- List endpoint GET /orders returns bare JSON array
  Cannot add pagination or metadata later without breaking change
  Fix: wrap in {"data": [...], "pagination": {...}}

### Medium
- Money amounts returned as float (1299.9900000000002 — IEEE 754 precision loss)
  Financial rounding errors in client applications
  Fix: return as string-encoded decimal ("1299.99") or integer cents (129999)

- No versioning strategy — breaking changes deployed over existing endpoint
  Existing clients break on each release
  Fix: introduce /v1/ prefix; establish breaking-change policy with deprecation notices

### Low
- Timestamp fields returned in epoch seconds (1710512521)
  Harder to debug; no timezone information
  Fix: return ISO 8601 UTC strings ("2024-03-15T14:32:01Z")

### Passed
- Resources named as plural nouns ✓
- HTTP methods correctly assigned (no GETs with side effects) ✓
- 401 vs 403 correctly differentiated ✓
```

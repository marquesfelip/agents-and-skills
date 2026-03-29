---
name: tenant-file-isolation
description: 'Design and enforce tenant-level file namespace isolation in multi-tenant SaaS object storage — prefix strategy, IAM scoping, cross-tenant prevention, and DB-layer ownership enforcement. Use when: tenant file isolation, tenant storage isolation, multi-tenant S3, tenant namespace, tenant bucket, tenant prefix, object storage multi-tenant, tenant file separation, cross-tenant file access, tenant owned file, per-tenant storage, storage tenant isolation, S3 tenant prefix, tenant data isolation storage, file ownership tenant.'
argument-hint: 'Describe the tenancy model (shared bucket with prefix / per-tenant bucket) and how tenants are identified — or paste existing storage/file code to review'
---

# Tenant File Isolation Specialist

## When to Use
- Designing file storage for a multi-tenant SaaS product for the first time
- Auditing an existing storage layout for cross-tenant access risks
- Implementing IAM or bucket policy scoping to enforce tenant prefix boundaries
- Ensuring DB-level ownership validation prevents object enumeration across tenants
- Migrating from a flat storage layout to a tenant-namespaced one

---

## Step 1 — Select the Isolation Model

| Model | Description | When to Choose |
|---|---|---|
| **Shared bucket, key prefix** | `tenants/{tenant_id}/...` | Default for most SaaS — simple, cost-efficient |
| **Per-tenant bucket** | `bucket: tenant-{tenant_id}` | Enterprise / compliance tiers with strict isolation needs |
| **Hybrid** | Shared bucket for standard; dedicated bucket for enterprise | Multi-tier SaaS with different compliance profiles |

**For shared bucket model (recommended default):**
- Tenant isolation is enforced at the **application layer** (DB ownership check) + optional IAM prefix conditions
- Cost: one bucket, simpler operations
- Risk surface: misconfigured query returning wrong tenant's `object_key` → cross-tenant read

**For per-tenant bucket:**
- Strong isolation: IAM denies cross-bucket access at the infrastructure level
- Cost: bucket proliferation (S3 limit: 1000 per account, increasable)
- Risk surface: bucket name enumeration if naming is predictable

---

## Step 2 — Design the Key Prefix Schema

### Shared Bucket — Recommended prefix layout

```
{bucket}/
  tenants/
    {tenant_id}/           ← UUID, not slug (harder to enumerate)
      uploads/
        {object_key}       ← server-generated UUID
      media/
        {object_key}
      exports/
        {object_key}
      tmp/
        {object_key}       ← short-lived, lifecycle rule deletes after 24h
```

**Rules:**
- [ ] `tenant_id` in the prefix is always the internal UUID — never the display name or slug
- [ ] Object key under the prefix is also a UUID — not derived from filename
- [ ] `tmp/` prefix has a dedicated lifecycle rule (delete after 24h) for failed/abandoned uploads
- [ ] Prefix never constructed from user-supplied input (path traversal prevention)

### Per-Tenant Bucket — Recommended naming

```
{product}-tenant-{tenant_id}          ← safe, opaque
NOT: {product}-{company_name}         ← guessable, enumerable
```

---

## Step 3 — IAM Prefix Scoping (Shared Bucket)

For microservices or isolated workers that must only touch one tenant's data, scope IAM to the prefix:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:HeadObject"],
    "Resource": "arn:aws:s3:::my-saas-prod/tenants/${aws:PrincipalTag/TenantId}/*"
  }]
}
```

This uses IAM policy variables to dynamically scope access to the tenant's prefix using the caller's `TenantId` tag — enforced at AWS level, not application level.

For a single application backend (most common), instead enforce at DB layer (Step 4) and leave the IAM role with prefix-level access to the entire `tenants/*` subtree.

---

## Step 4 — DB-Layer Ownership Enforcement (Critical)

**This is the primary isolation boundary.** Every file operation must validate ownership through the database before any storage action.

```sql
-- storage_objects table (always present)
CREATE TABLE storage_objects (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID NOT NULL REFERENCES tenants(id),
  object_key  TEXT NOT NULL UNIQUE,
  status      TEXT NOT NULL DEFAULT 'pending',
  ...
);
CREATE INDEX ON storage_objects (tenant_id, id);
```

**Download authorization pattern:**
```go
// CORRECT — validates ownership before signing
func (h *FileHandler) Download(ctx context.Context, req DownloadRequest) (string, error) {
    obj, err := h.repo.FindByIDAndTenant(ctx, req.FileID, req.TenantID)
    if err != nil || obj == nil {
        return "", ErrNotFound  // 404 — never 403 (oracle)
    }
    return h.storage.PresignGet(ctx, obj.ObjectKey, 10*time.Minute)
}

// WRONG — trusts client-supplied key directly
func (h *FileHandler) DownloadUnsafe(ctx context.Context, key string) (string, error) {
    return h.storage.PresignGet(ctx, key, 10*time.Minute)  // no ownership check!
}
```

- [ ] `FindByIDAndTenant` is the ONLY way to retrieve a file — always filters on `tenant_id`
- [ ] `object_key` never accepted from the client in any request parameter
- [ ] List operations scoped: `WHERE tenant_id = $1` always present in file queries
- [ ] No query uses `SELECT * FROM storage_objects WHERE object_key = $1` (bypasses tenant check)

---

## Step 5 — Upload Isolation

When generating a presigned upload URL:

```go
func (h *FileHandler) RequestUploadURL(ctx context.Context, req UploadRequest) (UploadResponse, error) {
    tenantID := auth.TenantIDFromContext(ctx)  // from authenticated session

    // Object key includes tenant prefix — server constructed, never from client
    objectKey := fmt.Sprintf("tenants/%s/uploads/%s", tenantID, uuid.NewString())

    // Persist placeholder before returning URL (prevent orphaned uploads)
    obj := &StorageObject{
        TenantID:  tenantID,
        ObjectKey: objectKey,
        Status:    "pending",
        Filename:  sanitize(req.Filename),  // display name only, never used as key
    }
    if err := h.repo.Create(ctx, obj); err != nil { return UploadResponse{}, err }

    url, err := h.storage.PresignPut(ctx, objectKey, req.ContentType, 15*time.Minute)
    return UploadResponse{FileID: obj.ID, UploadURL: url}, err
}
```

- [ ] Tenant ID extracted from authenticated context — never from request body or URL param
- [ ] Object key always constructed server-side with tenant prefix
- [ ] DB record created before returning the URL (prevents ghost uploads)
- [ ] `req.Filename` stored as display metadata only — not used to construct the storage key

---

## Step 6 — Cross-Tenant Access Audit

Run this review against existing file access code:

| Check | Correct Pattern | Anti-Pattern |
|---|---|---|
| File lookup | `WHERE id = $1 AND tenant_id = $2` | `WHERE id = $1` (no tenant filter) |
| File list | `WHERE tenant_id = $1` always present | `SELECT * FROM storage_objects` |
| Key construction | `"tenants/" + tenantID + "/" + uuid` | `"uploads/" + filename` |
| Key in response | Never exposed | `object_key` in API response JSON |
| Key in request | Never accepted | Client sends `?key=tenants/other/...` |

---

## Quality Checks

- [ ] All object keys include tenant UUID as a prefix segment
- [ ] Object key never derived from user-supplied filename or path
- [ ] Every file read/download validates `tenant_id` ownership in DB before presigning
- [ ] List endpoints always include `WHERE tenant_id = ?` — no global queries
- [ ] `object_key` is an internal field — not exposed in any API response
- [ ] DB index on `(tenant_id, id)` for performant scoped lookups
- [ ] Abandoned `pending` records cleaned up by a scheduled job
- [ ] Cross-tenant access tested: attempt to download another tenant's `file_id` returns 404

---

## After Completion

Produce:
1. Object key prefix schema diagram
2. `storage_objects` table DDL with isolation-enforcing indexes
3. Upload URL generation handler (with tenant context injection)
4. Download handler with `FindByIDAndTenant` ownership check
5. Cross-tenant access test cases (request isolation)

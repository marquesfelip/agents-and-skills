---
name: private-object-access-control
description: 'Enforce private object access control on S3-compatible storage — bucket policies, IAM, presigned-only access, and API proxy patterns. Use when: private object access, private bucket, object access control, S3 bucket policy, IAM policy S3, block public access, object ACL, private file access, access control storage, storage authorization, object-level access, S3 access control, restrict object access, secure file access, bucket ACL, public access block, storage permission.'
argument-hint: 'Describe what objects need to be protected and who should access them — or paste existing bucket policy or IAM config to review'
---

# Private Object Access Control Specialist

## When to Use
- Ensuring a storage bucket (S3, R2, Wasabi, GCS) is never publicly accessible
- Designing application-layer access control for stored objects
- Auditing existing bucket policies, ACLs, or IAM roles for overly permissive access
- Deciding between bucket policy enforcement vs. application-side authorization
- Reviewing how application credentials are scoped (least-privilege IAM)

---

## Step 1 — Define the Access Model

Objects can be accessed via several mechanisms. Choose **only the appropriate ones**:

| Access Type | When to Use | Risk Level |
|---|---|---|
| Public bucket URL | Truly public assets (logos, marketing images) | Low (intentional) |
| Presigned URL | User-specific, time-limited access | Low when scoped correctly |
| API proxy (server streams bytes) | High sensitivity; need audit trail | Lowest (fully controlled) |
| CloudFront / CDN + Signed Cookies | High-traffic media delivery | Low when properly signed |
| Direct SDK with IAM (server-to-server) | Backend service fetching a file | Low when least-privilege |

**Rule for SaaS:** Default to **presigned URLs with short TTL** for all user-facing access. Use the API proxy pattern only for documents requiring strict audit or DLP scanning on read.

---

## Step 2 — Block All Public Access (Day 1 Configuration)

Apply this for every bucket regardless of intent — even "public" assets should be served via CDN, not raw bucket URLs.

### AWS S3
```json
// Block Public Access settings — enable all four:
{
  "BlockPublicAcls": true,
  "IgnorePublicAcls": true,
  "BlockPublicPolicy": true,
  "RestrictPublicBuckets": true
}
```
Apply via AWS Console, CLI, or Terraform (`aws_s3_bucket_public_access_block`).

### Cloudflare R2
R2 buckets are **private by default**. Do not enable "R2.dev subdomain" in production — it makes the bucket publicly accessible.

### Wasabi / MinIO
Block public ACLs at the bucket level via the provider console or policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringNotEquals": { "aws:PrincipalArn": "arn:aws:iam::ACCOUNT:role/my-app-role" }
    }
  }]
}
```

- [ ] Block public access settings enabled on all buckets (prod, staging, dev)
- [ ] No ACLs set to `public-read` on any object
- [ ] Bucket policy explicitly denies any `s3:GetObject` except your application principal

---

## Step 3 — Least-Privilege IAM for Application Credentials

Application credentials should be scoped to **exactly the operations they perform**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UploadObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:HeadObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-saas-prod/tenants/*"
    },
    {
      "Sid": "ListBucketForConfirmation",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::my-saas-prod",
      "Condition": {
        "StringLike": { "s3:prefix": ["tenants/*"] }
      }
    }
  ]
}
```

**Rules:**
- [ ] `s3:DeleteBucket` never granted to the application role
- [ ] `s3:PutBucketPolicy` never granted to the application role
- [ ] No wildcard actions (`"Action": "s3:*"`) on production buckets
- [ ] Separate IAM roles/credentials for prod, staging, and dev environments
- [ ] Credentials rotated at least every 90 days (use IAM roles with STS for EC2/ECS — no long-lived keys)

---

## Step 4 — Application-Layer Access Authorization

**The bucket being private is necessary but not sufficient** — you must also enforce authorization at the application layer so tenants cannot access other tenants' objects.

Pattern:
```
GET /files/{file_id}/download
  → Authenticate request (session / JWT)
  → Look up storage_objects WHERE id = file_id AND tenant_id = current_tenant_id
  → If not found: 404 (not 403 — don't disclose the object exists)
  → Generate presigned GET URL for the validated object_key
  → Return 302 redirect or { url } response
```

**Why look up by `file_id` not `object_key`:**
- `object_key` is internal — never exposed in the API
- `file_id` (UUID) is opaque but validates ownership via DB join
- Prevents path traversal (e.g., `../tenants/other/file.pdf`)

- [ ] Every download endpoint validates object ownership against DB — not just that the key exists in storage
- [ ] Response is `404` for unauthorized objects — not `403` (avoids oracle)
- [ ] `object_key` never appears in API responses (only `file_id`)
- [ ] Never trust a client-supplied `object_key` to generate a presigned URL directly

---

## Step 5 — API Proxy Pattern (High-Sensitivity Files)

For highly sensitive files (contracts, medical records, financial reports) that require an audit trail:

```
GET /files/{file_id}/content
  → Authenticate + authorize (same as Step 4)
  → Log access event: { user_id, file_id, tenant_id, ip, timestamp }
  → Download object from storage (server-side SDK)
  → Stream bytes to client with correct Content-Type / Content-Disposition
```

```go
// Go — stream from S3 without buffering entire file in memory
out, err := s3Client.GetObject(ctx, &s3.GetObjectInput{Bucket: &bucket, Key: &key})
if err != nil { ... }
defer out.Body.Close()
w.Header().Set("Content-Type", aws.ToString(out.ContentType))
w.Header().Set("Content-Disposition", `attachment; filename="report.pdf"`)
io.Copy(w, out.Body)  // stream directly — no io.ReadAll
```

- [ ] Use `io.Copy` (Go) / `shutil.copyfileobj` (Python) — never load entire file into memory
- [ ] Set response timeout longer than typical API calls (large files take time)
- [ ] Log every access in an audit table (who downloaded what, when, from where)
- [ ] Rate-limit the proxy endpoint to prevent bulk harvesting

---

## Step 6 — Access Control Review Checklist

Run this audit against any existing storage configuration:

- [ ] **Public access block**: all four settings enabled on every bucket?
- [ ] **Bucket policy**: explicit deny for anonymous access?
- [ ] **ACLs**: no object has `public-read` ACL?
- [ ] **IAM**: application role uses least-privilege, no wildcard actions?
- [ ] **Credentials**: no long-lived access keys for server workloads (use IAM roles / instance profiles)?
- [ ] **API layer**: downloads go through authorization check before presigning?
- [ ] **Object key**: never derived from or exposed as user input?
- [ ] **Audit**: sensitive file access is logged?

---

## Quality Checks

- [ ] Bucket is private by default — confirmed via provider console or IaC
- [ ] Application IAM policy scoped to required operations only, on required prefix only
- [ ] Download endpoint validates `tenant_id` ownership before generating presigned URL
- [ ] `object_key` never exposed in API — only `file_id`
- [ ] API proxy used for high-sensitivity documents with audit logging
- [ ] Access control tested: attempt cross-tenant access returns 404

---

## After Completion

Produce:
1. Bucket policy JSON (deny-all-public + allow-app-role)
2. Least-privilege IAM policy JSON for the application role
3. Download endpoint implementation with ownership validation
4. Audit log schema for proxy-mode access

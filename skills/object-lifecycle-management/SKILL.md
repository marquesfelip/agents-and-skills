---
name: object-lifecycle-management
description: 'Design and implement storage lifecycle rules, retention automation, and object expiration policies for S3-compatible object storage in SaaS. Use when: object lifecycle, storage lifecycle, S3 lifecycle rule, object expiration, storage retention, data retention S3, lifecycle policy, object TTL, storage cleanup, expired objects, delete old files, storage lifecycle automation, S3 expiration rule, incomplete multipart cleanup, transition storage class, Glacier transition, object retention policy, cheap storage tier, storage archival.'
argument-hint: 'Describe the file types, retention requirements, and compliance constraints — or paste existing lifecycle config to review'
---

# Object Lifecycle Management Specialist

## When to Use
- Defining how long different file types should be retained in object storage
- Configuring S3 lifecycle rules to auto-expire or transition objects
- Implementing application-layer soft-delete + async storage cleanup
- Ensuring incomplete multipart uploads are cleaned up automatically
- Designing GDPR/LGPD-compliant data deletion for stored files

---

## Step 1 — Map Retention Requirements by Object Type

Before writing lifecycle rules, define the data classification:

| Object Category | Examples | Retention Target | Action at Expiry |
|---|---|---|---|
| **Temp / abandoned uploads** | `tmp/`, `pending` uploads >1h | 24 hours | Hard-delete from storage |
| **Processed media variants** | Thumbnails, HLS segments | Same as source | Delete when source deleted |
| **User uploads (active)** | Documents, images tied to active account | While account active | Soft-delete → async purge |
| **User uploads (deleted)** | User or tenant deleted the file | Configurable grace period (30–90 days) | Hard-delete after grace |
| **Exports / reports** | Generated CSV, PDF exports | 7–30 days | Auto-expire |
| **Audit attachments** | Compliance docs, signed contracts | 5–7 years | Archive (Glacier / cold storage) |
| **Backups** | Database backup files stored in S3 | Per backup policy | Transition + expire |

---

## Step 2 — Configure S3 Lifecycle Rules (IaC)

Apply lifecycle rules via Terraform (never ad-hoc console changes).

### Terraform example
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  # Rule 1 — Clean up abandoned/tmp uploads after 24h
  rule {
    id     = "expire-tmp-objects"
    status = "Enabled"
    filter { prefix = "tenants/" }
    # More specific: only tmp/ subpath
    filter { prefix = "tenants/*/tmp/" }
    expiration { days = 1 }
  }

  # Rule 2 — Clean up incomplete multipart uploads (critical cost control)
  rule {
    id     = "abort-incomplete-multipart"
    status = "Enabled"
    filter { prefix = "" }  # applies to entire bucket
    abort_incomplete_multipart_upload { days_after_initiation = 1 }
  }

  # Rule 3 — Transition old exports to Intelligent-Tiering after 30 days
  rule {
    id     = "exports-transition"
    status = "Enabled"
    filter { prefix = "tenants/*/exports/" }
    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }
    expiration { days = 90 }
  }

  # Rule 4 — Archive audit attachments to Glacier after 90 days
  rule {
    id     = "archive-audit-docs"
    status = "Enabled"
    filter { prefix = "tenants/*/audit/" }
    transition {
      days          = 90
      storage_class = "GLACIER_IR"   # Instant Retrieval — hours not ms
    }
  }

  # Rule 5 — Delete soft-deleted objects after 30-day grace period
  # (Application sets object tag: deleted=true + deleted_at epoch)
  rule {
    id     = "purge-soft-deleted"
    status = "Enabled"
    filter {
      tag {
        key   = "lifecycle"
        value = "soft-deleted"
      }
    }
    expiration { days = 30 }
  }
}
```

**Key rules every bucket must have:**
- [ ] `abort_incomplete_multipart_upload` — 1 day (prevents abandoned multipart cost accumulation)
- [ ] Expiry for `tmp/` prefix — 1 day
- [ ] Soft-delete tag-based expiry (see Step 3)

---

## Step 3 — Application Soft-Delete + Async Purge

Object storage delete is immediate and irreversible. Use a soft-delete pattern:

### On user-initiated file deletion
```go
func (r *FileRepo) SoftDelete(ctx context.Context, fileID, tenantID uuid.UUID) error {
    now := time.Now().UTC()
    // 1. Mark DB record as deleted
    err := r.db.ExecContext(ctx,
        `UPDATE storage_objects SET deleted_at = $1 WHERE id = $2 AND tenant_id = $3`,
        now, fileID, tenantID,
    )
    if err != nil { return err }

    // 2. Tag the S3 object for lifecycle-driven deletion
    _, err = r.s3.PutObjectTagging(ctx, &s3.PutObjectTaggingInput{
        Bucket: &r.bucket,
        Key:    &objectKey,
        Tagging: &types.Tagging{TagSet: []types.Tag{
            {Key: aws.String("lifecycle"), Value: aws.String("soft-deleted")},
            {Key: aws.String("deleted_at"), Value: aws.String(strconv.FormatInt(now.Unix(), 10))},
        }},
    })
    return err
}
```

### Grace period options
| Grace Period | Use Case |
|---|---|
| 0 days (immediate) | Scratch/temp files, test environments |
| 7 days | Standard user deletes (allows recovery) |
| 30 days | Enterprise/compliance default |
| 90+ days | Regulated data (healthcare, finance) |

- [ ] Deletion from the application never calls `DeleteObject` directly — always soft-delete + tag
- [ ] S3 lifecycle rule on `lifecycle=soft-deleted` tag does the actual delete after grace period
- [ ] Grace period is configurable per tenant plan or data category

---

## Step 4 — Tenant Account Deletion (GDPR/LGPD Right to Erasure)

When a tenant cancels and requests data deletion:

```
1. Set account status = 'pending_deletion' (allow grace period, e.g., 30 days)
2. At grace period end:
   a. DB: soft-delete all storage_objects for tenant
   b. Storage: apply 'lifecycle=soft-deleted' tag to all objects under tenants/{tenant_id}/
   c. DB: anonymize/delete tenant PII records
   d. Enqueue async confirmation job
3. S3 lifecycle rule purges tagged objects on schedule (within 30 days)
4. Log deletion completion to compliance audit table
```

**For immediate regulatory deletion (right to erasure with no grace period):**
```go
// Bulk-tag all tenant objects for immediate S3 deletion
// Use S3 Batch Operations for tenants with >1000 objects
```

- [ ] Right-to-erasure flow deletes or anonymizes all tenant objects
- [ ] Compliance audit record created confirming deletion date
- [ ] S3 Batch Operations used for bulk-delete of large tenants (>1000 objects)

---

## Step 5 — Storage Class Transitions

| S3 Storage Class | Cost Profile | Retrieval Latency | Use For |
|---|---|---|---|
| **S3 Standard** | High storage, zero retrieval | Milliseconds | Active user files |
| **S3 Intelligent-Tiering** | Automatic tiering | Milliseconds (frequent) | Variable-access files |
| **S3 Standard-IA** | Lower storage, retrieval fee | Milliseconds | Accessed <1/month |
| **S3 Glacier Instant Retrieval** | Very low storage, retrieval fee | Milliseconds | Archives accessed rarely |
| **S3 Glacier Flexible Retrieval** | Lowest storage | Minutes–hours | Long-term archives |
| **S3 Glacier Deep Archive** | Cheapest | 12 hours | Compliance archives, 7+ years |

**Decision:**
- Active user files: Standard
- Exports/reports >30 days old: Intelligent-Tiering or Standard-IA
- Audit/compliance docs >90 days: Glacier Instant Retrieval
- Long-term legal hold >3 years: Glacier Deep Archive

---

## Step 6 — Lifecycle Observability

- [ ] CloudWatch metric: `NumberOfObjects` per prefix — alert on unexpected growth
- [ ] CloudWatch metric: `BucketSizeBytes` — alert on cost-anomaly threshold
- [ ] Monthly cost report: by prefix, by storage class (use S3 Storage Lens or Cost Explorer)
- [ ] Log lifecycle rule deletions to audit table via S3 Event Notification → SQS → consumer
- [ ] Test lifecycle rules in staging with minimum-day overrides before applying to prod

---

## Quality Checks

- [ ] `abort_incomplete_multipart_upload` (1 day) applied to entire bucket
- [ ] `tmp/` prefix expires in 24 hours
- [ ] Soft-delete pattern in use — application never calls `DeleteObject` directly
- [ ] Soft-delete tag (`lifecycle=soft-deleted`) triggers S3 lifecycle expiry after grace period
- [ ] Storage class transitions defined for exports and archive categories
- [ ] Tenant deletion flow covers all objects under tenant prefix
- [ ] GDPR/LGPD right-to-erasure path exists and is audited
- [ ] All lifecycle rules defined in IaC (Terraform/Pulumi) — no console-only config

---

## After Completion

Produce:
1. Terraform lifecycle configuration block for the production bucket
2. Soft-delete implementation (`SoftDelete` function + object tagging)
3. Tenant deletion flow pseudocode with grace period and audit logging
4. Storage class transition decision table for the product's data categories
5. Retention policy summary table (by object type, retention period, action)

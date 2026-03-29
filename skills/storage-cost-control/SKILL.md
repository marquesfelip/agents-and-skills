---
name: storage-cost-control
description: 'Monitor, analyze, and optimize S3-compatible object storage costs for SaaS products — egress, request costs, storage class optimization, quota enforcement, and cost anomaly alerting. Use when: storage cost, S3 cost, storage optimization, egress cost, object storage bill, storage cost control, storage cost monitoring, S3 pricing, R2 cost, Wasabi cost, cost anomaly storage, storage quota, storage budget, storage billing, reduce S3 cost, storage class optimization, cost per tenant, storage spend, request cost, data transfer cost, storage overage.'
argument-hint: 'Describe current storage usage (GB, object count, egress pattern) and cost concerns — or paste current infrastructure config to review'
---

# Storage Cost Control Specialist

## When to Use
- Diagnosing an unexpectedly high storage bill (S3, R2, Wasabi)
- Designing per-tenant storage quotas for a SaaS product
- Choosing the right storage class or provider to optimize cost
- Setting up cost monitoring, alerting, and anomaly detection
- Reviewing upload/download patterns that drive egress costs

---

## Step 1 — Understand S3 Cost Dimensions

Storage costs come from four distinct dimensions:

| Dimension | What Drives It | Optimization Levers |
|---|---|---|
| **Storage (GB/month)** | Total bytes stored across all classes | Lifecycle transitions, delete unused objects, compress |
| **Requests** | PUT, GET, LIST, DELETE, HEAD calls | Batch operations, reduce LIST frequency, cache metadata |
| **Egress (data transfer)** | Bytes transferred out of the region | CDN, same-region access, switch to R2/Wasabi |
| **Retrieval fees** | Bytes retrieved from IA/Glacier tiers | Only transition cold data; ensure access patterns match tier |

**Egress is the highest risk.** AWS S3 charges $0.09/GB for egress. R2 charges $0.00. For egress-heavy SaaS, provider choice alone can cut costs 80%+.

---

## Step 2 — Provider Cost Comparison

| Provider | Storage/GB/mo | Egress/GB | PUT (per 1000) | GET (per 1000) |
|---|---|---|---|---|
| **AWS S3 Standard** | $0.023 | $0.09 | $0.005 | $0.0004 |
| **AWS S3 IA** | $0.0125 | $0.09 + $0.01 retrieve | $0.01 | $0.001 |
| **Cloudflare R2** | $0.015 | **$0.00** | $4.50 | $0.36 |
| **Wasabi** | $0.0068 | **$0.00** | $0 | $0 |
| **Backblaze B2** | $0.006 | $0.01 | $0 | $0 |
| **GCS Standard** | $0.020 | $0.08 | $0.005 | $0.0004 |

**Decision guide:**

| Scenario | Best Provider |
|---|---|
| High egress, low storage | **R2** (zero egress) |
| High storage, low egress | **Wasabi** or **Backblaze B2** |
| Heavy Lambda/event integration | **S3** (ecosystem) |
| GCP-native workload | **GCS** |
| Privacy-first, GDPR | **R2** (EU region) or **Wasabi** (EU DC) |

---

## Step 3 — Track Storage Usage Per Tenant

Expose storage usage as a billable metric from day one:

```sql
-- storage_usage_by_tenant view
CREATE MATERIALIZED VIEW tenant_storage_usage AS
SELECT
  tenant_id,
  COUNT(*) FILTER (WHERE status != 'deleted' AND scan_status = 'clean') AS object_count,
  COALESCE(SUM(size_bytes) FILTER (WHERE status != 'deleted' AND scan_status = 'clean'), 0) AS used_bytes
FROM storage_objects
GROUP BY tenant_id;

CREATE UNIQUE INDEX ON tenant_storage_usage (tenant_id);
-- Refresh on a schedule (every 15 min or triggered post-upload)
```

**Per-tenant quota enforcement:**
```go
func (s *StorageService) CheckQuota(ctx context.Context, tenantID uuid.UUID, uploadSizeBytes int64) error {
    usage, err := s.repo.GetTenantUsage(ctx, tenantID)
    if err != nil { return err }

    plan, err := s.planRepo.GetByTenant(ctx, tenantID)
    if err != nil { return err }

    if usage.UsedBytes + uploadSizeBytes > plan.StorageQuotaBytes {
        return ErrStorageQuotaExceeded  // HTTP 422 with upgrade prompt
    }
    return nil
}
```

- [ ] Quota checked before issuing a presigned upload URL (not after upload completes)
- [ ] `size_bytes` updated in `storage_objects` after upload confirmation (`HeadObject`)
- [ ] Materialized view refreshed frequently enough for accurate quota display

---

## Step 4 — Eliminate the Largest Cost Leaks

### Leak 1 — Abandoned incomplete multipart uploads
Abandoned multipart uploads accumulate bytes silently. A single 1GB upload attempt left incomplete costs $0.023/month forever.

```hcl
# Terraform — abort incomplete multipart after 1 day
rule {
  id     = "abort-incomplete-multipart"
  status = "Enabled"
  abort_incomplete_multipart_upload { days_after_initiation = 1 }
}
```

- [ ] Lifecycle rule `abort_incomplete_multipart_upload` enabled on every bucket

### Leak 2 — Orphaned objects (no DB record)
Objects whose DB records were deleted (bad migration, bug) continue accumulating storage cost.

Run a monthly reconciliation job:
```go
// List all objects in storage; cross-reference with storage_objects table
// Flag orphaned keys (in storage, not in DB) for review
// Optionally auto-delete after human review
```

### Leak 3 — Redundant object versions
If bucket versioning is enabled, every overwrite creates a new version. Enable versioning only if you need it; add a lifecycle rule to expire non-current versions.

```hcl
rule {
  id     = "expire-old-versions"
  status = "Enabled"
  noncurrent_version_expiration { noncurrent_days = 30 }
}
```

### Leak 4 — Serving media directly from S3 (no CDN)
Each download generates an S3 GET request + egress charge. CDN eliminates repeat egress:

- CloudFront: $0.0085/GB after cache HIT (85–95% cache ratio typical → 90% egress reduction)
- R2 + Cloudflare CDN: zero egress altogether

- [ ] Processed media variants served via CDN — not direct S3 presigned URLs
- [ ] CDN cache headers set: `Cache-Control: public, max-age=31536000, immutable` for processed variants

---

## Step 5 — Storage Class Optimization Review

Run S3 Storage Lens or AWS Cost Explorer to identify:

| Finding | Action |
|---|---|
| Objects >30 days old in Standard, not accessed in 14+ days | Transition to Intelligent-Tiering |
| Objects in Standard-IA retrieved frequently (>1/month) | Move back to Standard (retrieval surcharges exceed savings) |
| Glacier objects retrieved for processing (not human access) | Switch to Glacier Instant Retrieval |
| Many small objects transferred to cold tier | Small objects (<128KB) cheaper to keep in Standard than IA/Glacier |

**Minimum size for S3 IA transition:** Objects <128KB are charged as 128KB in IA — not worth transitioning.

---

## Step 6 — Cost Alerting and Anomaly Detection

### AWS Budget Alert
```hcl
resource "aws_budgets_budget" "storage" {
  name         = "s3-monthly-storage"
  budget_type  = "COST"
  limit_amount = "500"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator = "GREATER_THAN"
    threshold           = 80   # alert at 80% of budget
    threshold_type      = "PERCENTAGE"
    notification_type   = "ACTUAL"
    subscriber_email_addresses = ["ops@yourcompany.com"]
  }
}
```

### Application-Level Metrics to Track
- `storage.used_bytes_total` (gauge, by tenant) — detect runaway growth
- `storage.objects_total` (gauge, by prefix) — detect orphan accumulation
- `storage.upload_bytes_per_hour` — detect upload spikes (potential abuse)
- `storage.egress_bytes_per_hour` — detect download exfiltration (if proxied)

### Alerts to Configure
- [ ] Total bucket size grows >X% week-over-week (possible runaway)
- [ ] Single tenant exceeds Y% of total storage (fairness + quota signal)
- [ ] Incomplete multipart count >0 for >24 hours (lifecycle rule not working)
- [ ] Monthly cost forecast exceeds budget threshold

---

## Step 7 — Plan-Based Storage Tiers

Define storage quotas per subscription plan and enforce them:

| Plan | Storage Quota | Max File Size | Retention |
|---|---|---|---|
| Free | 1 GB | 25 MB | 90 days |
| Starter | 10 GB | 100 MB | 1 year |
| Pro | 100 GB | 1 GB | 3 years |
| Enterprise | Custom | Custom | Custom |

- [ ] Quotas enforced at upload time (not retroactively)
- [ ] Usage dashboard visible to tenants in real time
- [ ] Overage notification sent at 80% and 95% of quota
- [ ] Soft overage grace period configurable (e.g., 7 days before hard block)

---

## Quality Checks

- [ ] `abort_incomplete_multipart_upload` lifecycle rule active on all buckets
- [ ] Per-tenant storage quota tracked in DB and enforced at upload time
- [ ] CDN serving processed media variants — not direct S3 egress
- [ ] Cost alert configured at 80% of monthly storage budget
- [ ] Storage class transitions defined for files older than 30/90 days
- [ ] Monthly orphan reconciliation job scheduled
- [ ] Storage usage surfaced in tenant dashboard
- [ ] Plan-based quotas defined with overage notification thresholds

---

## After Completion

Produce:
1. Terraform lifecycle + budget alert configuration
2. `tenant_storage_usage` materialized view DDL
3. Quota enforcement middleware/service function
4. Month-over-month cost breakdown analysis checklist
5. Storage tier table (plan → quota, max file size, retention)

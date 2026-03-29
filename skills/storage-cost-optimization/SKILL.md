---
name: storage-cost-optimization
description: 'Storage lifecycle and cost optimization strategies for SaaS products. Use when: storage cost optimization, S3 lifecycle, object storage cost, storage tiering, intelligent tiering, S3 Glacier, storage cost reduction, object expiration, lifecycle rule, storage cost control, storage cost monitoring, S3 cost, storage class transition, cold storage, warm storage, hot storage, storage retention policy, blob storage cost, GCS lifecycle, Azure Blob lifecycle, storage cost per tenant, orphan object cleanup, multipart upload cleanup, unused object deletion, storage budget, storage spend alert, S3 cost anomaly.'
argument-hint: 'Describe the object storage provider (S3, GCS, R2, Azure Blob), the types of objects stored (user uploads, exports, backups, logs, media), the typical object access pattern (hot / warm / cold / archive), and whether there is a per-tenant storage quota enforced.'
---

# Storage Cost Optimization

## When to Use

Invoke this skill when you need to:
- Implement S3/GCS lifecycle rules to automatically transition objects to cheaper storage classes
- Delete expired objects (exports, temp files, stale backups) on a schedule without manual intervention
- Detect and clean up orphaned objects: objects in storage with no corresponding DB record
- Alert when total storage spend or per-tenant storage usage exceeds a threshold
- Identify the largest cost buckets in object storage (user uploads, logs, exports, backups)
- Design a storage retention policy that satisfies compliance requirements at minimum cost

---

## Storage Tier Cost Reference (AWS S3 approximate)

```
Tier                  Cost/GB/mo    Retrieval     Use when
────────────────────  ────────────  ────────────  ─────────────────────────────────
S3 Standard           $0.023        Instant       Actively used (< 30 days old)
S3 Standard-IA        $0.0125       Instant       Accessed < monthly (30–90 days)
S3 Glacier Instant    $0.004        Instant       Rarely accessed (90–365 days)
S3 Glacier Flexible   $0.0036       3–5 hours     Archives, compliance (> 1 year)
S3 Glacier Deep       $0.00099      12 hours      Long-term archives (> 5 years)
S3 Intelligent-Tiering $0.023+fee  Adapts        Unknown access patterns

Rule: Transition inactive objects within 30 days. Archive after 90 days. Delete at compliance boundary.
```

---

## Step 1 — Lifecycle Rule Design by Object Category

```go
// Each object category has different access patterns and retention requirements

type StorageLifecyclePolicy struct {
    Prefix          string        // S3 key prefix to apply policy to
    Description     string
    TransitionIA    *int          // days until Standard-IA transition (nil = skip)
    TransitionGlacier *int        // days until Glacier Instant transition
    ExpireAfter     *int          // days until hard delete (nil = never delete)
}

var DefaultPolicies = []StorageLifecyclePolicy{
    {
        Prefix:          "uploads/",    // user-uploaded files
        Description:     "User uploads — retain 1 year unless user deletes",
        TransitionIA:    intPtr(30),    // to Standard-IA after 30 days
        TransitionGlacier: intPtr(90), // to Glacier Instant after 90 days
        ExpireAfter:     intPtr(365),   // delete after 1 year (configurable per plan)
    },
    {
        Prefix:          "exports/",    // CSV/PDF exports — short lived
        Description:     "Data exports — expire after 7 days",
        TransitionIA:    nil,           // no transition — delete directly
        ExpireAfter:     intPtr(7),     // hard delete after 7 days
    },
    {
        Prefix:          "backups/",    // DB backups
        Description:     "Database backups — 90-day retention",
        TransitionIA:    intPtr(7),
        TransitionGlacier: intPtr(30),
        ExpireAfter:     intPtr(90),
    },
    {
        Prefix:          "logs/",       // application log archives
        Description:     "Log archives — 1 year retention",
        TransitionIA:    intPtr(30),
        TransitionGlacier: intPtr(90),
        ExpireAfter:     intPtr(365),
    },
    {
        Prefix:          "tmp/",        // temporary processing files
        Description:     "Temp files — expire after 24 hours",
        ExpireAfter:     intPtr(1),     // delete after 1 day
    },
}
```

---

## Step 2 — Apply Lifecycle Rules via AWS SDK

```go
func ApplyLifecycleRules(ctx context.Context, s3Client *s3.Client, bucket string, policies []StorageLifecyclePolicy) error {
    rules := make([]s3types.LifecycleRule, 0, len(policies))

    for i, p := range policies {
        rule := s3types.LifecycleRule{
            ID:     aws.String(fmt.Sprintf("rule-%d-%s", i, strings.Trim(p.Prefix, "/"))),
            Status: s3types.ExpirationStatusEnabled,
            Filter: &s3types.LifecycleRuleFilterMemberPrefix{
                Value: p.Prefix,
            },
            Transitions: []s3types.Transition{},
        }

        if p.TransitionIA != nil {
            rule.Transitions = append(rule.Transitions, s3types.Transition{
                Days:         aws.Int32(int32(*p.TransitionIA)),
                StorageClass: s3types.TransitionStorageClassStandardIa,
            })
        }
        if p.TransitionGlacier != nil {
            rule.Transitions = append(rule.Transitions, s3types.Transition{
                Days:         aws.Int32(int32(*p.TransitionGlacier)),
                StorageClass: s3types.TransitionStorageClassGlacierIr,
            })
        }
        if p.ExpireAfter != nil {
            rule.Expiration = &s3types.LifecycleExpiration{
                Days: aws.Int32(int32(*p.ExpireAfter)),
            }
        }

        // Always clean up incomplete multipart uploads — they accumulate silently
        rule.AbortIncompleteMultipartUpload = &s3types.AbortIncompleteMultipartUpload{
            DaysAfterInitiation: aws.Int32(7),
        }

        rules = append(rules, rule)
    }

    _, err := s3Client.PutBucketLifecycleConfiguration(ctx, &s3.PutBucketLifecycleConfigurationInput{
        Bucket: aws.String(bucket),
        LifecycleConfiguration: &s3types.BucketLifecycleConfiguration{
            Rules: rules,
        },
    })
    return err
}
```

---

## Step 3 — Orphan Object Cleanup

```go
// Orphan: an object in S3 with no corresponding record in the DB
// Common cause: upload succeeded but DB write failed (or record deleted without deleting object)

func (c *OrphanCleaner) Run(ctx context.Context) error {
    slog.Info("orphan cleanup started")
    var cleaned, kept int

    paginator := s3.NewListObjectsV2Paginator(c.s3, &s3.ListObjectsV2Input{
        Bucket: aws.String(c.bucket),
        Prefix: aws.String("uploads/"),
    })

    for paginator.HasMorePages() {
        page, err := paginator.NextPage(ctx)
        if err != nil {
            return fmt.Errorf("list objects: %w", err)
        }

        for _, obj := range page.Contents {
            key := aws.ToString(obj.Key)
            exists, err := c.fileRepo.ExistsByStorageKey(ctx, key)
            if err != nil {
                slog.Error("DB lookup failed", "key", key, "err", err)
                continue
            }
            if exists {
                kept++
                continue
            }

            // Object has no DB record — safe to delete
            if c.dryRun {
                slog.Info("[dry-run] would delete orphan", "key", key, "size", obj.Size)
            } else {
                c.s3.DeleteObject(ctx, &s3.DeleteObjectInput{
                    Bucket: aws.String(c.bucket),
                    Key:    aws.String(key),
                })
                slog.Info("orphan deleted", "key", key, "size_bytes", obj.Size)
            }
            cleaned++
        }
    }

    slog.Info("orphan cleanup complete", "cleaned", cleaned, "kept", kept, "dry_run", c.dryRun)
    return nil
}
```

---

## Step 4 — Per-Tenant Storage Quota and Cost Tracking

```sql
-- Track storage usage per tenant in DB (updated on upload/delete)
CREATE TABLE tenant_storage_usage (
    tenant_id       UUID        PRIMARY KEY REFERENCES tenants(id),
    used_bytes      BIGINT      NOT NULL DEFAULT 0,
    file_count      INT         NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Increment on upload
UPDATE tenant_storage_usage
SET used_bytes = used_bytes + $1,
    file_count = file_count + 1,
    last_updated_at = now()
WHERE tenant_id = $2;

-- Decrement on delete
UPDATE tenant_storage_usage
SET used_bytes = GREATEST(used_bytes - $1, 0),
    file_count = GREATEST(file_count - 1, 0),
    last_updated_at = now()
WHERE tenant_id = $2;
```

---

## Step 5 — Storage Cost Alert

```go
const StorageCostPerGBCents = 2  // $0.023/GB/mo ≈ 2 cents per GB

func (m *StorageCostMonitor) CheckTenantUsage(ctx context.Context) {
    usages, _ := m.repo.GetAllTenantUsage(ctx)
    for _, u := range usages {
        gb := float64(u.UsedBytes) / (1024 * 1024 * 1024)
        estimatedMonthlyCents := int64(gb * StorageCostPerGBCents)

        if estimatedMonthlyCents > m.TenantStorageBudgetCents {
            slog.Warn("tenant storage over budget",
                "tenant_id", u.TenantID,
                "used_gb", gb,
                "estimated_monthly_cents", estimatedMonthlyCents,
                "budget_cents", m.TenantStorageBudgetCents,
            )
        }
    }
}
```

---

## Lifecycle Policy Design Table

| Object type | Hot (Standard) | Transition to IA | Archive | Delete |
|---|---|---|---|---|
| Active user uploads | Until accessed | 30 days | 90 days (Glacier) | 365 days or user delete |
| Data exports | None | — | — | 7 days |
| DB backups | 7 days | 7 days | 30 days | 90 days |
| Processing temp files | None | — | — | 1 day |
| Log archives | 30 days | 30 days | 90 days | 365 days |
| Compliance records | Until required | 30 days | 90 days (Glacier Deep) | Per compliance requirement |

---

## Quality Checks

- [ ] Lifecycle rules applied to all prefixes — not just `uploads/` — including `tmp/`, `exports/`, `backups/`
- [ ] `AbortIncompleteMultipartUpload` set to 7 days on all buckets — prevents silent accumulation
- [ ] Orphan cleaner runs in dry-run mode first — results reviewed before live deletion
- [ ] `tenant_storage_usage` table updated atomically on every upload and delete
- [ ] Storage cost alert fires when any tenant's estimated monthly cost exceeds plan limit
- [ ] Lifecycle rules managed as code (Terraform or CDK) — not manually via console
- [ ] Compliance record retention verified against legal requirements before setting expiry

## After Completion

- Use **`object-lifecycle-management`** for detailed S3 lifecycle rule patterns and Terraform templates
- Use **`compute-cost-guardrails`** for the compute-side cost companion
- Use **`cost-aware-architecture`** for overall FinOps strategy and cost attribution
- Use **`private-object-access-control`** to ensure lifecycle deletions respect access control boundaries

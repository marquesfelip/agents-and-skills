---
name: data-export-compliance
description: 'Compliant data export workflows for SaaS products — GDPR, LGPD, and right to portability. Use when: data export compliance, GDPR data export, LGPD data export, right to portability, data portability, user data export, tenant data export, export all my data, GDPR article 20, data subject request, DSR data export, compliant export, data export pipeline, export job, async data export, export notification, export download link, export formats, export scope, export encryption, export retention.'
argument-hint: 'Describe the export scope (tenant-level or user-level), applicable regulations (GDPR, LGPD, CCPA), export format requirements (JSON, CSV, ZIP), and how the export is delivered (email link, in-app download, S3 presigned URL).'
---

# Data Export Compliance

## When to Use

Invoke this skill when you need to:
- Implement GDPR/LGPD Article 20 right-to-portability export for tenant or user data
- Build an async export pipeline that does not time out for large datasets
- Deliver exports securely via time-limited download links (not email attachments)
- Scope the export to exactly the data the requester is entitled to — no more, no less
- Audit all export requests for compliance record-keeping
- Handle export requests triggered during offboarding (see `customer-offboarding`)

---

## Export Workflow Overview

```
Request received → validate authorization → create export job (pending)
      ↓
Background worker → gather data per domain → write to ZIP → upload to storage
      ↓
Notify requester → generate time-limited presigned download URL
      ↓
Requester downloads → URL expires after TTL → file deleted after retention period
```

---

## Step 1 — Schema

```sql
CREATE TYPE export_status AS ENUM ('pending', 'processing', 'ready', 'failed', 'expired');

CREATE TABLE data_export_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID      NOT NULL REFERENCES tenants(id),
    requested_by    UUID      REFERENCES users(id),  -- NULL = system-triggered
    scope           TEXT      NOT NULL,              -- 'tenant' | 'user'
    target_user_id  UUID      REFERENCES users(id),  -- set when scope = 'user'
    status          export_status NOT NULL DEFAULT 'pending',
    regulation      TEXT,                            -- 'gdpr' | 'lgpd' | 'ccpa' | 'voluntary'
    format          TEXT      NOT NULL DEFAULT 'zip', -- 'zip' | 'json' | 'csv'
    storage_key     TEXT,                            -- populated when ready
    download_count  INT NOT NULL DEFAULT 0,
    expires_at      TIMESTAMPTZ,                     -- download link expiry
    error_message   TEXT,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    CONSTRAINT chk_user_scope CHECK (scope != 'user' OR target_user_id IS NOT NULL)
);

CREATE INDEX idx_export_requests_tenant  ON data_export_requests(tenant_id);
CREATE INDEX idx_export_requests_status  ON data_export_requests(status) WHERE status IN ('pending','processing');
```

---

## Step 2 — Authorization and Rate Limiting

Only authorized actors may request exports:

```go
func (s *exportService) RequestExport(ctx context.Context, input ExportRequest) (*DataExportRequest, error) {
    actor := auth.ActorFromCtx(ctx)

    // Authorization: tenant owner/admin can export tenant-scope; user can export own data
    switch input.Scope {
    case ScopeTenant:
        if !actor.HasRole(RoleAdmin) && !actor.HasRole(RoleOwner) {
            return nil, ErrForbidden
        }
    case ScopeUser:
        if actor.UserID != input.TargetUserID && !actor.HasRole(RoleAdmin) {
            return nil, ErrForbidden
        }
    }

    // Rate limit: one pending/processing export per tenant per scope per day
    existing, err := s.repo.FindPendingExport(ctx, input.TenantID, input.Scope)
    if err != nil {
        return nil, err
    }
    if existing != nil {
        return existing, nil // idempotent — return existing job
    }

    job := &DataExportRequest{
        ID:           uuid.New(),
        TenantID:     input.TenantID,
        RequestedBy:  actor.UserID,
        Scope:        input.Scope,
        TargetUserID: input.TargetUserID,
        Regulation:   input.Regulation,
        Format:       input.Format,
    }
    if err := s.repo.Create(ctx, job); err != nil {
        return nil, err
    }

    s.jobs.Enqueue(ctx, BuildExportJob{ExportID: job.ID})
    return job, nil
}
```

---

## Step 3 — Export Builder (Background Worker)

```go
type DomainExporter interface {
    Name() string
    Export(ctx context.Context, tenantID uuid.UUID, w io.Writer) error
}

// Register all domain exporters here
var domainExporters = []DomainExporter{
    &ProfileExporter{},
    &MembershipExporter{},
    &BillingExporter{},
    &AuditLogExporter{},
    &ContentExporter{},
    // Add new exporters as new domain areas are added
}

func (w *exportWorker) Build(ctx context.Context, exportID uuid.UUID) error {
    job, err := w.repo.FindByID(ctx, exportID)
    if err != nil {
        return err
    }

    // Mark processing
    if err := w.repo.UpdateStatus(ctx, exportID, ExportStatusProcessing, ""); err != nil {
        return err
    }

    // Build ZIP in memory (stream to temp file for large exports)
    var buf bytes.Buffer
    zw := zip.NewWriter(&buf)

    for _, exporter := range domainExporters {
        fw, err := zw.Create(exporter.Name() + ".json")
        if err != nil {
            return w.fail(ctx, exportID, fmt.Errorf("zip create %s: %w", exporter.Name(), err))
        }
        if err := exporter.Export(ctx, job.TenantID, fw); err != nil {
            // Non-fatal per domain — record warning but continue
            slog.Warn("exporter failed", "name", exporter.Name(), "err", err)
        }
    }

    // Add manifest
    manifest := buildManifest(job, domainExporters)
    mw, _ := zw.Create("manifest.json")
    _ = json.NewEncoder(mw).Encode(manifest)
    zw.Close()

    // Upload to storage — key includes export ID (never predictable)
    storageKey := fmt.Sprintf("exports/%s/%s.zip", job.TenantID, exportID)
    if err := w.storage.PutObject(ctx, storageKey, bytes.NewReader(buf.Bytes())); err != nil {
        return w.fail(ctx, exportID, fmt.Errorf("upload: %w", err))
    }

    expiresAt := time.Now().UTC().Add(72 * time.Hour) // 72h download window

    if err := w.repo.MarkReady(ctx, exportID, storageKey, expiresAt); err != nil {
        return err
    }

    // Notify requester
    w.jobs.Enqueue(ctx, SendExportNotificationJob{ExportID: exportID})
    return nil
}
```

---

## Step 4 — Secure Download Link

```go
func (s *exportService) GetDownloadURL(ctx context.Context, exportID uuid.UUID) (string, error) {
    actor := auth.ActorFromCtx(ctx)
    job, err := s.repo.FindByID(ctx, exportID)
    if err != nil {
        return "", err
    }

    // Authorization: only requester or admin can download
    if job.RequestedBy != actor.UserID && !actor.HasRole(RoleAdmin) {
        return "", ErrForbidden
    }
    if job.Status != ExportStatusReady {
        return "", ErrExportNotReady
    }
    if time.Now().UTC().After(*job.ExpiresAt) {
        return "", ErrExportExpired
    }

    // Generate a time-limited presigned URL (15 min max — force re-auth for each download)
    url, err := s.storage.PresignedGetURL(ctx, job.StorageKey, 15*time.Minute)
    if err != nil {
        return "", err
    }

    // Increment download count for audit
    _ = s.repo.IncrementDownloadCount(ctx, exportID)
    return url, nil
}
```

---

## Step 5 — Domain Exporter Implementation Pattern

```go
type ProfileExporter struct{ repo ProfileRepository }

func (e *ProfileExporter) Name() string { return "profile" }

func (e *ProfileExporter) Export(ctx context.Context, tenantID uuid.UUID, w io.Writer) error {
    profiles, err := e.repo.ListForTenant(ctx, tenantID)
    if err != nil {
        return err
    }

    // Only export fields the data subject is entitled to — no internal/admin fields
    type exportedProfile struct {
        UserID    string `json:"user_id"`
        Email     string `json:"email"`
        Name      string `json:"name"`
        CreatedAt string `json:"created_at"`
    }

    exported := make([]exportedProfile, len(profiles))
    for i, p := range profiles {
        exported[i] = exportedProfile{
            UserID:    p.UserID.String(),
            Email:     p.Email,
            Name:      p.Name,
            CreatedAt: p.CreatedAt.Format(time.RFC3339),
        }
    }
    return json.NewEncoder(w).Encode(exported)
}
```

---

## Export Scope Reference

| Data category | Include in tenant export | Include in user export | Notes |
|---|---|---|---|
| Profile / identity | Yes | Yes | |
| Memberships | Yes | Yes (own only) | |
| Billing history | Yes | No | Stripe invoices can be shared via Stripe portal |
| API keys | No | No | Never export secrets — export existence only |
| Audit log | Yes | Yes (own actions only) | |
| Domain content | Yes | Yes (own content only) | |
| Internal admin notes | No | No | Not subject data |
| System metadata | No | No | Machine IDs, internal flags |

---

## Compliance Checklist

| Requirement | GDPR Article | Implementation |
|---|---|---|
| Right to portability | Art. 20 | This skill |
| Machine-readable format | Art. 20(1) | JSON / CSV in ZIP |
| Response within 30 days | Art. 12(3) | Export job SLA monitoring |
| No charge for first export | Art. 12(5) | Free per regulations; rate-limit subsequent |
| Scope limited to personal data | Art. 4 | DomainExporter field filtering |
| Secure delivery | Art. 5(1)(f) | Presigned URL, not email attachment |

---

## Quality Checks

- [ ] Export requests are idempotent — second request for same scope returns existing job
- [ ] Presigned download URL TTL ≤ 15 minutes — requester must re-authenticate per download
- [ ] Storage key is not guessable — contains export UUID (opaque, random)
- [ ] Export file deleted from storage after `expires_at` (scheduled cleanup job)
- [ ] Admin fields (internal notes, system flags, raw IDs) excluded from exported `DomainExporter`
- [ ] Export request audit log: who requested, when, scope, download count
- [ ] GDPR export requests acknowledged within 72 hours — SLA alert if `status = pending` > 24h
- [ ] Billing/Stripe data: direct requester to Stripe Customer Portal rather than exporting raw data

## After Completion

- Use **`customer-offboarding`** to trigger export as part of the cancellation flow
- Use **`signed-url-management`** for the presigned URL generation and expiry patterns
- Use **`pii-handling`** for field-level personal data classification across exporters
- Use **`audit-trail-design`** to ensure export requests are themselves auditable
- Use **`object-lifecycle-management`** to auto-delete export files after their expiry window

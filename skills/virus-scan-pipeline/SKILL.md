---
name: virus-scan-pipeline
description: 'Design and implement malware scanning workflows for file uploads in SaaS — ClamAV, cloud antivirus APIs, async scan queues, quarantine state, and post-scan routing. Use when: virus scan, malware scan, antivirus upload, ClamAV, file scan pipeline, upload scanning, quarantine file, infected file, malware detection upload, virus check S3, scan before serve, file safety check, upload antivirus, object storage scan, virus scan queue, malware pipeline.'
argument-hint: 'Describe the upload pipeline and expected file types — or paste existing upload handling code to review'
---

# Virus Scan Pipeline Specialist

## When to Use
- Adding malware scanning to a file upload workflow for the first time
- Choosing between ClamAV (self-hosted), cloud AV APIs, or S3-native scanning
- Designing the quarantine state and what happens to infected vs. clean files
- Ensuring files cannot be downloaded before scanning completes
- Reviewing an existing scan pipeline for race conditions or bypass risks

---

## Step 1 — Choose the Scanning Approach

| Approach | Latency | Cost | Operational Burden | Best For |
|---|---|---|---|---|
| **ClamAV** (self-hosted, sidecar) | Low (ms–s) | Infrastructure only | Medium | High-volume SaaS, on-prem compliance |
| **ClamAV via TCP daemon** | Low (ms–s) | Infrastructure only | Medium | Centralized scanning service |
| **AWS GuardDuty Malware Protection** | Seconds | Per-GB scanned | Low | AWS-native stacks |
| **Cloudflare Gateway AV** | Low | Bundled with R2+Gateway | Low | R2-native stacks |
| **VirusTotal API** | Seconds–minutes | Per-request (quota) | Low | High-confidence second opinion |
| **Third-party API (Metadefender, etc.)** | Seconds | Per-scan | Low | Compliance-driven requirements |

**For most SaaS products:** ClamAV via a dedicated scanning service (async, queue-driven) is the right balance of cost, latency, and control.

---

## Step 2 — Design the Scan State Machine

Every uploaded file must pass through these states before becoming accessible:

```
pending → scanning → clean → (available to user)
                   → quarantined → (blocked, never served)
                   → scan_failed → (retry or manual review)
```

```sql
-- Add to storage_objects table
ALTER TABLE storage_objects ADD COLUMN scan_status TEXT NOT NULL DEFAULT 'pending';
-- Values: pending | scanning | clean | quarantined | scan_failed
ALTER TABLE storage_objects ADD COLUMN scanned_at TIMESTAMPTZ;
ALTER TABLE storage_objects ADD COLUMN scan_engine TEXT;    -- "clamav-1.3", "guardduty", etc.

CREATE INDEX ON storage_objects (scan_status) WHERE scan_status NOT IN ('clean');
```

**Rule:** Files must only be served (presigned URLs generated) when `scan_status = 'clean'` AND `status = 'ready'`. Both conditions required.

---

## Step 3 — Async Scan Queue Pattern

Never block the upload confirmation response waiting for a scan. Use a queue:

```
1. Client uploads file (presigned PUT)
2. Client: POST /uploads/confirm → { object_key }
3. Server: HeadObject → verify upload exists
4. Server: UPDATE storage_objects SET status='ready', scan_status='pending'
5. Server: Enqueue scan job → { object_id, object_key, bucket }
6. Server: 200 OK → { file_id, status: "processing" }

(async, in scan worker):
7. Worker: Dequeue job
8. Worker: Download object from storage → stream to ClamAV
9. ClamAV: Return FOUND / OK / ERROR
10. Worker: UPDATE storage_objects SET scan_status='clean'|'quarantined'|'scan_failed'
11. Worker (if clean): Publish event → downstream (media processing, indexing, etc.)
12. Worker (if quarantined): Delete object from storage; notify tenant if needed
```

---

## Step 4 — ClamAV Integration

### Architecture
```
┌─────────────┐     TCP (3310)    ┌─────────────┐
│  Scan Worker │ ────────────────► │   clamd     │
│  (Go/Python) │ ◄──────────────── │  (sidecar)  │
└─────────────┘   FOUND/OK/ERROR  └─────────────┘
```

### Go — ClamAV TCP client (clamd protocol)
```go
import "github.com/dutchcoders/go-clamd"

func scanObject(ctx context.Context, reader io.Reader) (clean bool, err error) {
    c := clamd.NewClamd("tcp://clamd:3310")
    response, err := c.ScanStream(reader, make(chan bool))
    if err != nil {
        return false, fmt.Errorf("scan error: %w", err)
    }
    for s := range response {
        if s.Status == clamd.RES_FOUND {
            return false, nil   // malware detected
        }
        if s.Status == clamd.RES_ERROR {
            return false, fmt.Errorf("clamav error: %s", s.Description)
        }
    }
    return true, nil   // clean
}
```

### ClamAV configuration best practices
- [ ] `MaxFileSize`: set to match your upload file size limit + 10% (ClamAV default: 25MB — raise for document/video SaaS)
- [ ] `MaxScanSize`: same or larger than `MaxFileSize`
- [ ] `MaxRecursion`: limit zip bomb depth (default: 16 — acceptable)
- [ ] Keep `clamd` database updated: `freshclam` on a daily cron
- [ ] Run `clamd` as a non-root user with read-only filesystem mount

---

## Step 5 — Handling Scan Outcomes

### Clean (scan_status = 'clean')
- Mark file available
- Trigger downstream processing queue (media pipeline, indexing)
- Do nothing else — normal flow resumes

### Quarantined (scan_status = 'quarantined')
- [ ] Delete the object from storage immediately: `DeleteObject(objectKey)`
- [ ] Soft-delete the `storage_objects` record (set `deleted_at`, keep for audit)
- [ ] Do NOT notify the user with specifics — generic: "File could not be processed"
- [ ] Log the detection: `{ file_id, tenant_id, filename, scan_engine, virus_name, detected_at }` to an audit table
- [ ] Alert the security team if detection rate exceeds threshold (potential attack)

### Scan Failed (scan_status = 'scan_failed')
- [ ] Retry up to 3 times with exponential backoff
- [ ] After max retries: move to `scan_failed`, alert ops team
- [ ] Do NOT serve the file — treat as unverified
- [ ] Offer tenant "retry scan" action in the UI (optional)

---

## Step 6 — Race Condition Prevention

**Problem:** A download request may arrive before the scan completes.

**Solution:** All download endpoints check `scan_status`:

```go
func (h *FileHandler) Download(ctx context.Context, req DownloadRequest) (string, error) {
    obj, err := h.repo.FindByIDAndTenant(ctx, req.FileID, req.TenantID)
    if err != nil || obj == nil {
        return "", ErrNotFound
    }
    if obj.ScanStatus != "clean" {
        switch obj.ScanStatus {
        case "pending", "scanning":
            return "", ErrFileProcessing   // 202 Accepted
        case "quarantined":
            return "", ErrFileUnavailable  // 422 — "cannot be served"
        case "scan_failed":
            return "", ErrFileProcessing   // 202 — retry later
        }
    }
    return h.storage.PresignGet(ctx, obj.ObjectKey, 10*time.Minute)
}
```

- [ ] Status `pending` / `scanning` returns HTTP 202 (not 404, not 403)
- [ ] Status `quarantined` returns HTTP 422 with generic error message
- [ ] Status `scan_failed` returns HTTP 202 — not an error visible to the user
- [ ] API response for file lists includes `processing_status` field — UI can poll or use websockets

---

## Quality Checks

- [ ] Scan state machine defined: pending → scanning → clean | quarantined | scan_failed
- [ ] Files never served before `scan_status = 'clean'`
- [ ] Scan is async (queue-based) — upload confirmation does not block on scan
- [ ] ClamAV virus DB updated daily
- [ ] Quarantined files deleted from storage + audit-logged
- [ ] Download endpoint blocks on non-clean status with correct HTTP codes
- [ ] Retry logic for `scan_failed` with max retries and ops alerting
- [ ] Scan worker idempotent — safe to retry the same job

---

## After Completion

Produce:
1. DB migration adding `scan_status` column and index
2. Scan worker implementation (queue consumer + ClamAV integration)
3. Download endpoint gating logic
4. Quarantine handler (delete + audit log)
5. Scan pipeline sequence diagram

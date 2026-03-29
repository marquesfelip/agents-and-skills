---
name: object-storage-architecture
description: 'Design scalable object storage integration patterns for SaaS products. Use when: object storage architecture, S3 architecture, blob storage design, storage integration design, scalable file storage, multi-region object storage, storage layer design, cloud storage architecture, SaaS file storage, object storage strategy, R2 architecture, GCS architecture, Azure Blob architecture, storage backend design, file storage SaaS.'
argument-hint: 'Describe the feature or system that needs object storage — or provide existing storage code/config to review'
---

# Object Storage Architecture Specialist

## When to Use
- Designing a new file storage layer for a SaaS product from scratch
- Choosing between S3, R2, Wasabi, GCS, or Azure Blob (cost/latency/compliance tradeoffs)
- Reviewing an existing storage integration for scalability or correctness issues
- Planning multi-region or multi-bucket storage topology
- Defining how uploads, downloads, and metadata flow through the system

---

## Step 1 — Understand the Storage Requirements

Before recommending patterns, gather:

| Dimension | Questions to Answer |
|---|---|
| **Access pattern** | Read-heavy? Write-heavy? Large files? Many small files? |
| **Tenancy model** | Single-tenant? Multi-tenant SaaS? Shared bucket or per-tenant? |
| **Privacy requirements** | Public objects? Presigned-only? Fully private with auth proxy? |
| **Compliance** | Data residency? GDPR/LGPD? Encryption at rest required? |
| **Scale** | Expected GB/TB? Number of objects? Concurrent upload rate? |
| **Downstream processing** | Does upload trigger media processing, virus scan, indexing? |

---

## Step 2 — Select the Storage Provider

| Provider | Best For | Key Tradeoff |
|---|---|---|
| **AWS S3** | Ecosystem breadth, Lambda triggers, large org | Cost (egress), complexity |
| **Cloudflare R2** | Zero-egress cost, CDN-native, privacy | Less ecosystem integration |
| **Wasabi** | Predictable low-cost storage | No free egress tier; no compute triggers |
| **GCS** | GCP-native workloads, ML pipelines | Egress costs similar to S3 |
| **Azure Blob** | Azure-locked stacks, enterprise licensing | Complexity for non-Azure stacks |
| **MinIO (self-hosted)** | On-premise / air-gap requirement | Operational burden |

**Rule:** Prefer R2 or Wasabi for cost-sensitive SaaS products with pre-signed URL delivery. Prefer S3 when you need event notifications (S3 → SNS → Lambda / SQS) to trigger processing pipelines.

---

## Step 3 — Design the Bucket Topology

### Single shared bucket (simplest)
```
bucket: my-saas-prod
  ├── tenants/{tenant_id}/uploads/{object_key}
  ├── tenants/{tenant_id}/media/{object_key}
  └── system/...
```
- Use object key prefixes as tenant namespace isolation
- Apply bucket policy or IAM to deny cross-tenant prefix access
- Suitable for most SaaS products with <1M tenants

### Per-tenant bucket (strongest isolation)
```
bucket: tenant-{tenant_id}
  ├── uploads/
  └── media/
```
- Full billing separation, no cross-tenant risk
- S3 soft limit: 1000 buckets per account (requestable increase)
- Suitable only when tenants are enterprise-grade or have compliance requirements

### Environment separation (always required)
```
my-saas-prod     # production
my-saas-staging  # staging (same provider, isolated)
```
- Never share buckets between environments
- Use separate IAM credentials per environment

---

## Step 4 — Design the Upload Flow

**Always use direct-to-storage upload** — never proxy file bytes through your API server.

```
Client → [GET /upload-url] → API Server
API Server → [GeneratePresignedPUT] → Storage
API Server → Client: { upload_url, object_key, fields }

Client → [PUT bytes directly] → Storage (presigned URL)
Client → [POST /uploads/confirm] → API Server
API Server → [HEAD object] → Storage (verify upload completed)
API Server → [trigger downstream] → Queue / Event
```

**Why:** Proxying through the server burns bandwidth, blocks threads, and creates an ingress bottleneck.

---

## Step 5 — Define Metadata Strategy

Object storage stores bytes, not rich metadata. Maintain a **metadata record in your relational database** for every stored object:

```sql
CREATE TABLE storage_objects (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id    UUID NOT NULL REFERENCES tenants(id),
  bucket       TEXT NOT NULL,
  object_key   TEXT NOT NULL,
  filename     TEXT NOT NULL,          -- original display name
  content_type TEXT NOT NULL,
  size_bytes   BIGINT,
  status       TEXT NOT NULL DEFAULT 'pending',  -- pending | ready | quarantined | deleted
  uploaded_at  TIMESTAMPTZ,
  deleted_at   TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON storage_objects (tenant_id, status);
```

- `status = pending` until the upload is confirmed server-side
- `status = quarantined` if virus scan fails
- `status = deleted` for soft-deletes (actual object deleted async)
- Never expose `object_key` directly in API responses — expose `id` and generate signed URLs on demand

---

## Step 6 — Assess Cross-Cutting Concerns

For each concern, verify a pattern is in place:

| Concern | Required Skill / Pattern |
|---|---|
| Signed URL generation | `signed-url-management` |
| Private access enforcement | `private-object-access-control` |
| Tenant namespace isolation | `tenant-file-isolation` |
| Malware scanning | `virus-scan-pipeline` |
| Media transcoding / thumbnails | `media-processing-pipeline` |
| Lifecycle and retention | `object-lifecycle-management` |
| Cost monitoring | `storage-cost-control` |

---

## Quality Checks

- [ ] File bytes never transit through the API server
- [ ] Bucket topology designed (shared prefix vs. per-tenant bucket)
- [ ] Separate buckets per environment (prod / staging / dev)
- [ ] Metadata persisted in relational DB for every object
- [ ] Object keys are server-generated UUIDs — never derived from user input
- [ ] Downstream triggers (queue, event) wired for post-upload processing
- [ ] Encryption at rest enabled on all buckets
- [ ] All cross-cutting concerns (scan, lifecycle, access) have assigned patterns

---

## After Completion

Produce:
1. A bucket topology diagram (text or Mermaid)
2. An upload flow sequence diagram
3. The `storage_objects` DB schema DDL
4. A prioritized list of cross-cutting concerns with recommended companion skills

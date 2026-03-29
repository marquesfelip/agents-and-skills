---
name: s3-compatible-storage-patterns
description: 'Implement correct S3-compatible API usage patterns across S3, R2, Wasabi, MinIO, and GCS interop. Use when: S3 compatible storage, S3 API patterns, AWS SDK S3, R2 S3 compatible, Wasabi S3, MinIO S3, S3 client configuration, S3 multipart upload, S3 SDK usage, S3 API integration, S3 presigned URL SDK, boto3 S3, aws-sdk-go S3, S3 CORS configuration, S3 bucket policy, S3 compatible client.'
argument-hint: 'Language/SDK being used (Go, Python, Node.js, etc.) and storage provider (S3, R2, Wasabi, MinIO) — or paste existing storage client code to review'
---

# S3-Compatible Storage Patterns Specialist

## When to Use
- Wiring up an S3-compatible storage client for the first time (any provider)
- Reviewing existing S3 SDK usage for misconfigurations or anti-patterns
- Migrating between S3-compatible providers (e.g., S3 → R2 or S3 → Wasabi)
- Configuring CORS, bucket policies, or multipart upload for an SDK
- Debugging SDK authentication, region, or endpoint issues

---

## Step 1 — Identify the SDK and Provider

| SDK | Language | Compatible Providers |
|---|---|---|
| `aws-sdk-go-v2` | Go | S3, R2, Wasabi, MinIO |
| `boto3` / `aioboto3` | Python | S3, R2, Wasabi, MinIO |
| `@aws-sdk/client-s3` v3 | Node.js / TS | S3, R2, Wasabi, MinIO |
| `aws-sdk` for Java | Java | S3, R2, Wasabi, MinIO |
| Presign via SDK | Any | S3, R2, Wasabi (R2 needs custom endpoint) |

**For non-AWS providers**, always require `endpoint_url` / `endpoint` override — the SDK defaults to `s3.amazonaws.com` and will silently fail.

---

## Step 2 — Client Configuration Checklist

### Go (`aws-sdk-go-v2`)
```go
cfg, _ := config.LoadDefaultConfig(ctx,
    config.WithRegion(os.Getenv("STORAGE_REGION")),           // e.g. "auto" for R2
    config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(
        os.Getenv("STORAGE_ACCESS_KEY_ID"),
        os.Getenv("STORAGE_SECRET_ACCESS_KEY"),
        "",
    )),
    config.WithEndpointResolverWithOptions(
        aws.EndpointResolverWithOptionsFunc(func(service, region string, opts ...interface{}) (aws.Endpoint, error) {
            return aws.Endpoint{URL: os.Getenv("STORAGE_ENDPOINT_URL"), SigningRegion: region}, nil
        }),
    ),
)
client := s3.NewFromConfig(cfg, func(o *s3.Options) {
    o.UsePathStyle = true  // Required for MinIO and Wasabi
})
```

### Python (`boto3`)
```python
import boto3
client = boto3.client(
    "s3",
    region_name=os.environ["STORAGE_REGION"],
    endpoint_url=os.environ.get("STORAGE_ENDPOINT_URL"),   # None = default AWS
    aws_access_key_id=os.environ["STORAGE_ACCESS_KEY_ID"],
    aws_secret_access_key=os.environ["STORAGE_SECRET_ACCESS_KEY"],
)
```

### Node.js (`@aws-sdk/client-s3` v3)
```typescript
import { S3Client } from "@aws-sdk/client-s3";
const s3 = new S3Client({
  region: process.env.STORAGE_REGION,
  endpoint: process.env.STORAGE_ENDPOINT_URL,   // omit for AWS
  credentials: {
    accessKeyId: process.env.STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.STORAGE_SECRET_ACCESS_KEY!,
  },
  forcePathStyle: true,  // Required for MinIO and Wasabi
});
```

**Checklist:**
- [ ] Credentials loaded from environment variables — never hardcoded
- [ ] `endpoint_url` / `endpoint` configurable per environment
- [ ] `use_path_style` / `forcePathStyle` set to `true` for MinIO and Wasabi
- [ ] Region set to `"auto"` for R2, actual AWS region for S3
- [ ] Single client instance created at startup and reused (not per-request)

---

## Step 3 — Multipart Upload Pattern

Use multipart upload for files **>5 MB**. Never buffer the entire file in memory.

```
Minimum part size: 5 MB (except the last part)
Maximum parts:     10,000
Maximum object:    5 TB
```

| File Size | Upload Method |
|---|---|
| < 5 MB | Single PutObject |
| 5 MB – 5 GB | Multipart upload (SDK managed, e.g., `transfer.Upload` in Go) |
| > 5 GB | Multipart upload (mandatory) |

**Go — use the Transfer Manager:**
```go
uploader := manager.NewUploader(client, func(u *manager.Uploader) {
    u.PartSize = 10 * 1024 * 1024  // 10 MB parts
    u.Concurrency = 3
})
_, err := uploader.Upload(ctx, &s3.PutObjectInput{
    Bucket:      aws.String(bucket),
    Key:         aws.String(key),
    Body:        reader,
    ContentType: aws.String(contentType),
})
```

- [ ] Use SDK-managed multipart upload — do not implement the 5-step multipart protocol manually
- [ ] Stream from source (disk/network) — do not `io.ReadAll` into memory
- [ ] Pass `ContentType` on every upload — do not rely on auto-detection

---

## Step 4 — Object Key Design

| Pattern | Example | Notes |
|---|---|---|
| UUID-based | `uploads/a3f8b2c1-....pdf` | Safest — no guessability |
| `{tenant}/{uuid}` | `t_abc123/uploads/uuid.pdf` | Prefix-based isolation |
| `{date}/{uuid}` | `2026/03/uuid.pdf` | Partition by time (useful for lifecycle rules) |

**Rules:**
- [ ] Object key is always server-generated — never derived from the client-supplied filename
- [ ] Use URL-safe characters only (`[a-z0-9_\-/.]`)
- [ ] No leading slashes (breaks some SDKs)
- [ ] Max key length: 1024 bytes

---

## Step 5 — CORS Configuration

Required for browser direct-to-storage uploads (presigned PUT).

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://app.yourproduct.com"],
      "AllowedMethods": ["PUT", "GET", "HEAD"],
      "AllowedHeaders": ["Content-Type", "Content-Length", "x-amz-*"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

- [ ] `AllowedOrigins` is an explicit allowlist — never `"*"` on a private bucket
- [ ] Only `PUT` and `GET` allowed — never `DELETE` from the browser
- [ ] `ETag` exposed so the client can confirm multipart assembly
- [ ] Apply via IaC (Terraform / Pulumi) — not via console ad-hoc

---

## Step 6 — Provider-Specific Gotchas

| Provider | Known issues |
|---|---|
| **R2** | Region must be `"auto"`; endpoint format: `https://<account_id>.r2.cloudflarestorage.com` |
| **Wasabi** | `path_style` required; endpoint per region (`s3.us-east-1.wasabisys.com`); no requester-pays |
| **MinIO** | `path_style` required; TLS cert may be self-signed in dev |
| **GCS (S3 interop)** | Enable via "Interoperability" in GCS console; HMAC keys required; endpoint: `https://storage.googleapis.com` |

---

## Quality Checks

- [ ] Client is a singleton — one instance per application lifecycle
- [ ] Credentials come from environment — not hardcoded or committed
- [ ] `forcePathStyle` set correctly per provider
- [ ] Multipart upload used for files >5 MB
- [ ] Object keys are UUID-based and server-generated
- [ ] CORS config applied via IaC with explicit origin allowlist
- [ ] Provider-specific endpoint and region configured correctly
- [ ] Client configuration is environment-specific (prod/staging/dev each have own bucket + creds)

---

## After Completion

Produce:
1. A ready-to-use storage client module for the chosen language/provider
2. A CORS configuration JSON/YAML snippet
3. An environment variable reference table (`STORAGE_ENDPOINT_URL`, `STORAGE_ACCESS_KEY_ID`, etc.)
4. A multipart upload helper function if files >5 MB are expected

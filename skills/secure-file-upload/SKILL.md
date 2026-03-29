---
name: secure-file-upload
description: 'Secure file upload validation, storage isolation, MIME validation, malware scanning, and access control. Use when: file upload security, secure file upload, file type validation, MIME type check, upload size limit, path traversal, filename sanitization, malware scanning, ClamAV, upload storage isolation, S3 upload security, multipart upload, file upload vulnerability, arbitrary file upload, file upload review.'
argument-hint: 'File upload handler, middleware, or storage config to review — or describe the file upload feature being designed'
---

# Secure File Upload Specialist

## When to Use
- Reviewing or designing a file upload endpoint
- Auditing filename sanitization, MIME type validation, or content inspection
- Designing storage isolation (S3, filesystem, cloud blob) for uploaded files
- Implementing malware scanning or content policy enforcement
- Preventing path traversal, arbitrary file execution, and storage abuse

---

## Step 1 — Understand the Upload Context

Before advising, identify:
- File types expected: documents, images, videos, arbitrary user files
- Storage target: local filesystem, S3/GCS/Azure Blob, database BLOB
- Who uploads: authenticated users, anonymous, third-party integrations
- Who downloads: same user, other users, public, internal only
- Size expectations: profile photos (KB), documents (MB), videos (GB)

---

## Step 2 — Input Validation Checklist

All checks must happen **server-side** — client-side validation is for UX only.

### File Size
- [ ] Enforce maximum file size at the HTTP layer (before parsing the body), not after reading it all into memory.
- [ ] Set limits per file type (profile photo: 5MB; document: 25MB; video: 2GB).
- [ ] Apply total upload quota per user to prevent storage abuse.

### File Type Validation (three-layer approach)
| Layer | Method | Trust level |
|---|---|---|
| Extension check | Whitelist `.jpg`, `.pdf`, `.docx` | Low — trivially spoofed |
| MIME type check | Inspect `Content-Type` header | Low — client-controlled |
| Magic bytes / file signature | Read first N bytes; compare against known signatures | High — content-based |

```
# File signature examples (magic bytes):
JPEG: FF D8 FF
PNG:  89 50 4E 47 0D 0A 1A 0A
PDF:  25 50 44 46 2D
ZIP:  50 4B 03 04
```

Use a library for magic byte detection: `python-magic`, `file-type` (Node.js), Apache Tika, Go `net/http.DetectContentType`.

- [ ] Reject any file whose magic bytes don't match an allowed type
- [ ] Reject files with double extensions (`malware.php.jpg`)
- [ ] Reject archives containing disallowed types (zip bomb / polyglot file protection)

### Filename Sanitization
- [ ] Generate a new, server-controlled filename — **never use the client-supplied filename** for storage.
- [ ] Use a UUID or random token as the stored filename.
- [ ] If preserving the original name for display, strip all path separators (`/`, `\`, `..`), null bytes, and shell metacharacters.
- [ ] Enforce lowercase, URL-safe characters only if displaying the filename in a URL.

```
# GOOD — server-generated filename
stored_name = uuid4() + ".pdf"

# BAD — using client-supplied filename directly
stored_name = request.files['file'].filename   # path traversal risk
```

---

## Step 3 — Storage Isolation

### Never Store Uploaded Files:
- In the web root / document root (prevents direct execution by the web server)
- In a path guessable from the user-supplied filename
- Mixed with application code or configuration files

### Recommended Storage Strategy:
| Option | Notes |
|---|---|
| Cloud object storage (S3, GCS, Azure Blob) | Preferred; files are not executable; separate from web server |
| Private bucket + signed URLs | Files not publicly accessible; access via time-limited signed URL |
| Dedicated volume, outside web root | For self-hosted; ensure web server config does not serve this path |
| Separate subdomain for user content | `cdn-user-content.example.com` — separate cookie scope, separate CSP |

**Permissions for stored files:**
- Default to private; explicitly grant read access only when needed.
- Never grant write or delete access to the upload service's storage credentials beyond its needs.
- Separate buckets per file category (profile photos vs documents vs exports).

---

## Step 4 — Malware Scanning

Scan every uploaded file before making it accessible:

| Tool | Integration |
|---|---|
| ClamAV | Open source; antivirus daemon; good for documents |
| VirusTotal API | Cloud-based; aggregates multiple AV engines; rate limits apply |
| AWS Macie | S3 PII/sensitive data detection (not general malware) |
| Cloud AV services | AWS GuardDuty, GCP Security Command Center add-ons |
| YARA rules | Custom malware signatures for domain-specific threats |

**Scan flow:**
1. Accept upload to quarantine storage area (not accessible).
2. Scan with AV engine.
3. If clean: move to production storage.
4. If infected: delete; log; alert security team; notify user.
5. Set a time limit — files in quarantine too long without scan result should be rejected.

For image-only uploads: also validate image dimensions and re-encode via an image library (strips embedded malicious content).

---

## Step 5 — Access Control

- [ ] Every file download request must verify the requesting user has permission to access that file.
- [ ] Never derive access permissions from the filename or URL alone.
- [ ] Store file ownership and access metadata in the database, not in the filename.
- [ ] For multi-tenant systems: include `tenant_id` on every file record and validate on access.
- [ ] Serve files via authenticated endpoints — not by exposing storage bucket URLs directly (unless using signed URLs with TTL).

```
# BAD — file accessible by anyone who knows the URL
GET https://s3.amazonaws.com/uploads/invoice-12345.pdf

# GOOD — authenticated access with ownership check
GET /api/files/f3a9b2c1
    → verify session
    → look up file record by ID
    → check file.user_id == session.user_id (or tenant match)
    → generate signed URL with TTL
    → redirect or stream
```

---

## Step 6 — Execution Prevention

Prevent uploaded files from being executed by the server or browser:

- [ ] Set `Content-Type: application/octet-stream` (or exact type) on download responses to prevent browser execution.
- [ ] Set `Content-Disposition: attachment; filename="..."` to force download instead of inline rendering.
- [ ] Never serve uploaded files from the same origin as the application — use a CDN subdomain or separate domain.
- [ ] Configure storage to deny execution (S3: disable static website hosting on upload bucket; nginx: no PHP/CGI for upload directory).
- [ ] For image galleries: serve via an image proxy that re-encodes the image (strips EXIF and embedded scripts).

---

## Step 7 — Common Vulnerabilities

| Vulnerability | Check | Fix |
|---|---|---|
| Path traversal | Is the client filename used in the storage path? | Use server-generated UUIDs; sanitize display name separately |
| Arbitrary file execution | Can `.php`, `.jsp`, `.sh` files be uploaded and executed? | Strict whitelist + separate storage domain |
| Zip bomb | Are archives unpacked without size limits? | Limit uncompressed size before extraction |
| Polyglot file | Is content trusted based on extension alone? | Magic byte validation + re-encode images |
| SSRF via SVG | Are SVGs rendered server-side with external entity fetching? | Sanitize SVG; disable external entity loading |
| Missing access control | Can any user download any file by ID? | Ownership check on every download |
| Storage DoS | No per-user quota? | Enforce file count and total size quotas per user |

---

## Step 8 — Output Report

```
## File Upload Security Review: <endpoint/service>

### Critical
- Client-supplied filename used directly in storage path
  `/uploads/` + request.filename → path traversal possible
  Fix: generate UUID filename server-side; never use client filename in path

- Uploaded files stored in web root and served directly → PHP/script execution possible
  Fix: move upload directory outside web root; serve via authenticated download endpoint

### High
- No magic byte validation — file type checked only by Content-Type header (client-controlled)
  Fix: add magic byte detection library; reject on mismatch

- No ownership check on GET /files/:id → any authenticated user can access any file
  Fix: verify file.user_id == session.user_id before serving

### Medium
- No malware scanning before files become accessible
  Fix: add ClamAV scan in quarantine flow before moving to production storage

- No per-user storage quota enforced
  Fix: track total bytes per user; reject uploads exceeding limit

### Low
- Content-Disposition header not set on file download
  Fix: add `Content-Disposition: attachment` to force download

### Passed
- File size enforced at HTTP middleware layer (before body parsing) ✓
- Files stored in private S3 bucket with signed URL access ✓
- Server-generated UUIDs used as storage filenames ✓
```

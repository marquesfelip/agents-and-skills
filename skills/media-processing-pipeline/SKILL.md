---
name: media-processing-pipeline
description: 'Design async media processing pipelines for SaaS — image resizing, video transcoding, thumbnail generation, FFmpeg workers, output routing, and status tracking. Use when: media processing pipeline, async media processing, image resize, thumbnail generation, video transcoding, FFmpeg pipeline, image pipeline, media worker, video pipeline, image optimization, media conversion, upload processing, media queue, async image, async video, file transformation, media variants, image CDN, video HLS, media job.'
argument-hint: 'Describe the media types (images / videos / documents) and required output variants — or paste existing processing code to review'
---

# Media Processing Pipeline Specialist

## When to Use
- Designing an async media processing pipeline triggered by file uploads
- Implementing image resizing/optimization (thumbnails, responsive variants)
- Implementing video transcoding (HLS, MP4 normalization, thumbnail extraction)
- Wiring up FFmpeg, Sharp, Pillow, or a managed service (AWS MediaConvert, Cloudflare Images)
- Reviewing an existing pipeline for race conditions, cost overruns, or re-processing gaps

---

## Step 1 — Classify Media Types and Required Variants

| Media Type | Common Variants | Processing Tool |
|---|---|---|
| **Images** (JPG, PNG, WebP, HEIC) | Thumbnail, medium, full; WebP conversion | Sharp (Node), Pillow (Python), `imaging` (Go), `libvips` |
| **Videos** (MP4, MOV, AVI, MKV) | HLS playlist, MP4 web-safe, thumbnail still | FFmpeg, AWS Elemental MediaConvert, Mux |
| **PDFs** | Page-1 thumbnail, text extraction | `pdfium`, `poppler`, AWS Textract |
| **Audio** | MP3 normalization, waveform image | FFmpeg |
| **Office docs** | PDF rendition, thumbnail | LibreOffice headless, AWS Textract |

Map each file type to its required variants before designing the pipeline.

---

## Step 2 — Pipeline Architecture

**Always async — never process media synchronously in the upload request handler.**

```
Upload confirmed (status='ready', scan_status='clean')
  │
  ▼
Queue: media-processing-jobs
  │
  ▼
Worker reads job: { file_id, object_key, media_type, variants: ["thumb", "medium"] }
  │
  ├─► Download source from storage (stream, no full buffer for large files)
  │
  ├─► Process variant 1 (e.g., 150x150 thumb)
  │     └─► Upload result to storage (tenants/{tid}/media/{file_id}/thumb.webp)
  │
  ├─► Process variant 2 (e.g., 800px medium)
  │     └─► Upload result to storage (tenants/{tid}/media/{file_id}/medium.webp)
  │
  └─► Update DB: media_assets table with variant URLs + processing_status='complete'
```

### media_assets DB table
```sql
CREATE TABLE media_assets (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL REFERENCES tenants(id),
  source_file_id  UUID NOT NULL REFERENCES storage_objects(id),
  variant         TEXT NOT NULL,             -- "thumb", "medium", "hls", "original"
  object_key      TEXT NOT NULL,
  width_px        INT,
  height_px       INT,
  duration_sec    INT,                       -- for video/audio
  size_bytes      BIGINT,
  content_type    TEXT NOT NULL,
  processing_status TEXT NOT NULL DEFAULT 'pending',  -- pending | complete | failed
  processed_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON media_assets (tenant_id, source_file_id);
```

---

## Step 3 — Image Processing Patterns

### Node.js — Sharp (recommended for high throughput)
```typescript
import sharp from "sharp";
import { Readable } from "stream";

async function resizeImage(
  sourceStream: Readable,
  widthPx: number,
): Promise<Buffer> {
  return sharp(sourceStream)
    .resize({ width: widthPx, withoutEnlargement: true })
    .webp({ quality: 82 })
    .toBuffer();
}
```

### Python — Pillow
```python
from PIL import Image
import io

def resize_image(source_bytes: bytes, width: int) -> bytes:
    img = Image.open(io.BytesIO(source_bytes))
    img.thumbnail((width, 99999), Image.LANCZOS)
    img = img.convert("RGB")
    out = io.BytesIO()
    img.save(out, format="WEBP", quality=82)
    return out.getvalue()
```

**Image processing rules:**
- [ ] Always convert to WebP for output variants (30–50% smaller than JPEG)
- [ ] `withoutEnlargement: true` — never upscale smaller images
- [ ] Strip EXIF metadata (privacy — photos contain GPS, device info)
- [ ] Process from stream — do not load multi-MB originals into memory all at once when feasible
- [ ] Set `quality: 80–85` for WebP — above 85 produces minimal visual gain with large file size

### Standard Variants
| Variant | Typical Dimensions | Use Case |
|---|---|---|
| `thumb` | 150×150 (crop center) | Grids, lists, avatars |
| `medium` | 800px wide | Article/content preview |
| `large` | 1920px wide | Lightbox, full view |
| `original` | As uploaded | Download / export |

---

## Step 4 — Video Processing Patterns

### FFmpeg — HLS output (adaptive streaming)
```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -crf 23 -preset fast \
  -c:a aac -b:a 128k \
  -vf scale=1280:-2 \
  -hls_time 6 -hls_list_size 0 \
  -hls_segment_filename "segment_%03d.ts" \
  output.m3u8
```

### FFmpeg — Thumbnail extraction (frame at 5 seconds)
```bash
ffmpeg -i input.mp4 -ss 00:00:05 -vframes 1 -vf scale=640:-2 thumb.jpg
```

**Video processing rules:**
- [ ] Never store raw `.mov` / `.avi` files as the delivery format — always transcode to H.264/AAC MP4 or HLS
- [ ] HLS segments and playlist stored under the same tenant prefix as the source
- [ ] Extract a thumbnail frame at ~5% into the video duration (not second 0 — often black)
- [ ] Use `libx264 + aac` baseline for maximum compatibility
- [ ] Set a maximum resolution cap (e.g., 1080p) — never transcode to higher than input
- [ ] Run FFmpeg in a separate process with a timeout to prevent runaway transcodes

---

## Step 5 — Managed Services (Alternative to Self-Hosted Workers)

| Service | Media Type | Tradeoff |
|---|---|---|
| **AWS Elemental MediaConvert** | Video | Pay-per-minute; fully managed |
| **Mux** | Video | Best-in-class HLS + analytics; SaaS cost |
| **Cloudflare Images** | Images | Per-image transform; CDN-native |
| **Imgix / Cloudinary** | Images | SaaS; URL-based transforms; cost at scale |
| **AWS Lambda + Sharp** | Images | Serverless; pay-per-request |

**When to use managed:** You want zero infrastructure for media processing and can afford the per-unit SaaS cost.
**When to self-host:** High volume makes SaaS per-unit cost prohibitive, or you need full control over codec output.

---

## Step 6 — Job Reliability and Idempotency

- [ ] Job payload includes `file_id` and `variant` — worker uses these as idempotency key
- [ ] Worker checks `media_assets` for existing `(source_file_id, variant)` before re-processing
- [ ] Retry on transient errors (network, OOM) with exponential backoff — max 3 retries
- [ ] After max retries: set `processing_status = 'failed'`, alert ops, keep source object intact
- [ ] Dead Letter Queue captures permanently failed jobs for manual reprocessing
- [ ] Worker timeout set per media type (image: 30s, video: 10–30min depending on duration)

---

## Step 7 — Delivering Processed Media

Processed variants should be served via CDN (CloudFront, Cloudflare) — not raw presigned S3 URLs:

```
CDN Distribution → Origin: private S3 bucket
CDN uses Origin Access Control (OAC) — bucket not public
Client requests: https://cdn.yourapp.com/media/{tenant_id}/{variant}/{file_id}.webp
CDN: cache HIT → return from edge cache
CDN: cache MISS → fetch from S3 via OAC → cache → return
```

- [ ] CDN in front of media delivery — not direct presigned URLs for processed variants
- [ ] CDN cache-control: `public, max-age=31536000, immutable` for processed variants (content-addressed)
- [ ] Original upload still served via presigned URL (not CDN — user-specific, must be authorized)
- [ ] CDN does not expose bucket name or internal object keys

---

## Quality Checks

- [ ] Processing is async — upload confirmation does not wait for media variants
- [ ] `media_assets` table tracks each variant's status and storage location
- [ ] EXIF metadata stripped from image output variants
- [ ] Video transcoded to web-safe format (H.264/AAC) — original codec not assumed
- [ ] HLS generated for video > 2 min (adaptive streaming)
- [ ] Jobs are idempotent — safe to re-enqueue without duplicating variants
- [ ] DLQ or equivalent for permanently failed jobs
- [ ] Processed variants delivered via CDN, not direct presigned URLs
- [ ] CDN cache headers set for immutable variant content

---

## After Completion

Produce:
1. `media_assets` table DDL
2. Worker queue consumer implementation with image/video branching
3. Image resizing function (Sharp/Pillow) producing WebP variants
4. FFmpeg command template for HLS + thumbnail generation
5. Pipeline sequence diagram (upload → queue → worker → storage → CDN)

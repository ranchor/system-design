# Handling Large Blobs (Uploads & Downloads)

> **TL;DR** — Never stream large files through your app servers. Issue **presigned URLs** so clients upload/download **directly to object storage** (S3/GCS), keep **metadata in a DB**, split big files into
> **chunks (multipart / resumable)** with per-part checksums, finish with a **complete** call, and process afterwards via **events** (S3 → queue → workers). Serve downloads through a **CDN** with
> range requests. Add **virus scanning, size/type limits, lifecycle rules, and dedup** as needed. See [Blob storage](blob_storage.md), [CDN](cdn.md), [Checksums](checksums.md), [Dropbox](../problems/dropbox.md), [Google Photos](../problems/google_photos.md).

## Why not through the API?
App servers become bandwidth/CPU/connection bottlenecks, hit request-size/timeouts limits at LB/gateway, and a failed 5 GB upload restarts from zero. Object stores are built for this.

## Upload flow (presigned, direct-to-store)
```
1. Client ─▶ API: POST /uploads {name, size, type, checksum}  → API authz, creates upload record (PENDING)
2. API ─▶ Client: presigned URL(s) (time-limited, scoped to one key, size/content-type conditions)
3. Client ─▶ Object store: PUT parts directly (parallel, retriable)
4. Client ─▶ API (or store event): complete → store assembles; record → READY
5. Store event ─▶ queue ─▶ workers: scan, thumbnail, transcode, index ─▶ metadata/status updates
```
- **Presigned URL**: signed with server credentials; expiry (minutes), method, key, content-length/type constraints. Never expose long-lived credentials.
- **Metadata DB** holds owner, key, size, checksum, status, version; the blob store holds bytes ([Data modeling](data_modeling.md)).

## Chunked / resumable uploads
- **Multipart upload** (S3: parts 5 MiB-5 GiB, up to 10,000 parts; check current limits) lets clients upload parts **in parallel**, **retry individual parts**, and **resume** after failure by listing uploaded parts.
- Client computes **per-chunk checksum** (and optionally whole-file) → verify integrity ([Checksums](checksums.md)); server validates ETag/SHA on complete.
- **tus / Google resumable upload protocol**: standard resumable HTTP uploads with offset queries.
- Choose chunk size by network (e.g. 8-64 MB): bigger = fewer calls, smaller = cheaper retry.
- **Abort and clean up** incomplete multipart uploads with lifecycle rules (they cost money).
- **Deduplication**: content-hash chunks (content-defined chunking) so identical data isn't stored twice; hash lookup before upload (instant "upload") ([Dropbox](../problems/dropbox.md)).

## Download flow
- **Presigned GET** or **CDN signed URL/cookie** for private content; cache public content at the edge ([CDN](cdn.md)).
- **Range requests** (`Range: bytes=`) for resume, seeking video, parallel download.
- **Streaming/adaptive bitrate** (HLS/DASH) for video; transcode into renditions + segments ([YouTube](../problems/youtube.md)).
- **Redirect** (302) from API to presigned URL for auth + offload.
- **Compression and formats**: serve WebP/AVIF/thumbnails, not originals ([File formats](file_formats.md)).

## Processing and lifecycle
- **Event-driven processing**: object-created event → queue → idempotent workers (thumbnailing, virus scan, OCR); status `PROCESSING → READY` ([Async](asynchronism.md), [Long-running tasks](long_running_tasks.md)).
- **Storage tiers**: hot → infrequent → archive via lifecycle policies; delete/expire; versioning for recovery.
- **Security**: validate type by content (not extension), size limits, malware scan, quarantine until clean, encryption at rest (SSE/KMS), per-object authorization, avoid public buckets, CORS configured for direct upload.
- **Consistency**: S3 provides strong read-after-write consistency for objects; still order your DB writes: mark metadata READY only after the object exists and is verified.
- **Cost**: egress and request charges; use CDN, compression, lifecycle, and avoid tiny-object explosions (batch small files).

## Pitfalls
- Proxying uploads through the API; presigned URLs with long expiry or no size limit; trusting client-reported size/type; orphaned multipart uploads; DB says READY but object missing (or vice versa); no checksum verification; serving private files via public URLs; not handling resume.

## Interview follow-ups
- *"Design uploading a 10 GB video."* Presigned multipart, parallel chunk upload with checksums, resume, event-driven transcode, CDN delivery.
- *"How do you stop someone uploading arbitrary huge files?"* Presigned policy conditions (content-length-range), quotas, API validation, post-upload scanning.
- *"How do you make downloads fast globally?"* CDN with signed URLs and range support.
- *"How do you dedupe?"* Chunk hashing + content-addressed storage; reference counting for deletes.

# References
* [Hello Interview: Handling large blobs](https://www.hellointerview.com/learn/system-design/patterns/large-blobs) · [AWS S3: multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html), [Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html), [tus protocol](https://tus.io/)

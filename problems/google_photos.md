## Problem Statement
Design **Google Photos**-style photo storage: users automatically back up photos/videos from phones, browse a fast, infinite timeline across devices, organize into albums, search by content (people, places,
objects), and share with others, all with durable, cost-efficient storage at massive scale.

## Clarification Questions to Interviewer
- Scale: users (1B), photos per user, daily uploads (billions), average size (3-5 MB photos, 100 MB+ videos)?
- Original quality kept vs compressed "storage saver"? Video support?
- Features: auto backup, albums, sharing (links/shared albums), search (ML), editing, deletion/trash?
- Latency: timeline scroll smoothness, thumbnails first? Offline mode and sync conflicts?
- Privacy: encryption, face recognition opt-in, regional storage?

## Requirements
### Functional Requirements
- **Upload/backup** photos & videos (resumable, background, dedup); view **original** and **thumbnails**.
- **Timeline** (by capture time, infinite scroll), **albums**, **sharing** (users/links), **trash** with restore.
- **Search** by metadata (date, location) and ML labels (faces, objects, text); **sync** across devices.
#### Below the line (out of scope)
- Photo editing, print ordering, memories/auto-collages ML details, billing.

### Non-Functional Requirements
- **Durability**: never lose a photo (11 nines) → replicated/erasure-coded blobs across AZs.
- **Availability** for viewing; upload may be async/eventual.
- **Low latency** timeline: first thumbnails <200 ms; tolerate slow processing for ML indexing.
- **Cost-efficiency**: storage dominates; tier cold data; dedupe.
- **Privacy/security**: per-user ACLs, encryption at rest, signed URLs.
#### Below the line (out of scope)
- Real-time collaborative editing.

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 1B users × avg 2 uploads/day = 2B uploads/day ≈ 23k/s avg, ×3 peak ≈ 70k/s. Avg 4 MB → ~8 PB/day ingest (≈ 740 Gbps avg). Thumbnails (3 sizes ≈ 100 KB total) add ~0.2 PB/day.
- 10-year storage at 8 PB/day ≈ 29 EB raw before dedup/compression; erasure coding (1.5×) and tiering are essential ([Blob storage](../basics/blob_storage.md)).
- Metadata: 2B/day × ~1 KB = 2 TB/day → ~7 PB over 10 years: sharded NoSQL/NewSQL ([Data partitioning](../basics/data_partitioning.md)).
- Reads: timeline views are thumbnail-heavy; **CDN + caching** absorb most.

## High-level API design
```
POST /v1/uploads {fileName, size, mime, checksum, takenAt, deviceId} → {uploadId, presignedParts[]}   (resumable, chunked)
POST /v1/uploads/{id}/complete                                       → {mediaId}
GET  /v1/timeline?cursor=&pageSize=&from=&to=                        → [{mediaId, thumbUrl, takenAt, …}]
GET  /v1/media/{id}  ·  GET /v1/media/{id}/thumb?size=               (signed CDN URLs)
POST /v1/albums  ·  POST /v1/albums/{id}/items  ·  POST /v1/share {mediaIds|albumId, recipients|link}
GET  /v1/search?q=beach+2023                                          ·  DELETE /v1/media/{id} (to trash)
```

## Data Model
- **Blob store** (object storage): originals at `media/{userId}/{mediaId}/orig`, derived `thumb_256/1024`, transcoded video renditions ([Large blobs](../basics/large_blobs.md)).
- **Media metadata** (wide-column/NoSQL, partition `userId`, cluster by `takenAt desc`): `mediaId, blobKey, contentHash, size, mime, takenAt, uploadedAt, location, deviceId, status, labels[], faces[]`; good for timeline range scans ([Cassandra](../basics/cassandra.md), [DynamoDB](../basics/dynamodb.md)).
- **Albums**: `albums(id, owner, title, cover)`, `album_items(albumId, mediaId, order)`; **shares**: `shares(resource, grantee, role)` with ACL checks.
- **Dedup index**: `contentHash → blobKey` (per user, or global with care for privacy).
- **Search index**: inverted index over labels/text/location + vector index for visual similarity ([Elasticsearch](../basics/elasticsearch.md)).

## High Level System Design
```
Phone ─presigned multipart─▶ Object store (originals)
   └─complete──▶ Upload svc ─▶ metadata DB (status=PROCESSING) ─▶ event (Kafka/queue)
                         ┌─────────────────────────────────────────────────────────┘
        Workers: virus scan · extract EXIF · thumbnails · video transcode · perceptual hash · ML (labels, faces, OCR) · index
                         └─▶ metadata DB (READY, labels) ─▶ search index ─▶ push/sync notification to devices
Viewer ─▶ API (timeline from metadata DB, cursor) ─▶ signed CDN URLs ─▶ CDN ─▶ object store
```
- **Upload path**: client hashes content, asks `exists?(hash)`: if present, skip bytes (**instant upload**). Otherwise chunked/resumable presigned upload straight to object storage; server never proxies bytes ([Large blobs](../basics/large_blobs.md)).
- **Processing pipeline** event-driven and **idempotent**: thumbnails and previews first (so the photo shows quickly), ML enrichment later; failures retry then DLQ ([Reliable messaging](../basics/reliable_messaging.md), [Long-running tasks](../basics/long_running_tasks.md)).
- **Timeline**: partition by user, ordered by `takenAt`, **keyset pagination** (cursor = last `(takenAt, mediaId)`); returns thumbnail URLs only; client lazily loads and caches, prefetches next page ([Scaling reads](../basics/scaling_reads.md)).
- **Sync**: devices pull changes via a per-user **change feed/sequence number** (`since=seq`); push notification triggers pull ([Real-time updates](../basics/realtime_updates.md)).
- **Delivery**: CDN-cached thumbnails (immutable keys), signed URLs, adaptive video (HLS) ([CDN](../basics/cdn.md), [YouTube](youtube.md)).

## Deep Dive
### Storage cost and tiering
Hot (recent/viewed) on standard tier; cold originals auto-move to infrequent/archive by access pattern; keep thumbnails hot. Erasure coding; optional **storage-saver compression** (re-encode to smaller JPEG/AVIF/HEVC). Deduplicate exact duplicates via content hash and near-duplicates via perceptual hash (pHash) for clustering/suggestions.

### Search and ML
Asynchronous inference: object/scene labels, face embeddings → clusters ("people"), OCR, geocoding → indexed into inverted + vector indexes; per-user private indexes; queries combine filters (date, place) with label matches and ranking. Face grouping is opt-in and region-restricted for privacy.

### Consistency and conflicts
Metadata writes per user are single-partition; use idempotent `mediaId` (derived from `(userId, contentHash)` or client-generated UUID) so retried uploads don't duplicate ([Idempotency](../basics/idempotency.md)). Album edits from multiple devices: last-writer-wins on scalar fields, set-union (CRDT-like) for item membership.

### Sharing and ACLs
Share links with unguessable tokens + expiry; shared albums via ACL rows; authorization at the API and encoded in signed URL claims; revoke by token invalidation and short URL TTLs.

### Deletion & trash
Soft delete (status=TRASHED, 30-60 days) then hard delete via lifecycle job; **reference counting** for deduped blobs; ensure ML indexes and shares are purged; legal/GDPR deletion workflow ([Event sourcing/CDC](../basics/event_sourcing_cqrs_cdc.md) for propagating deletions).

### Reliability and scale
Multi-AZ blob/metadata replication; upload sessions resumable for days; per-user rate limits; backpressure on the processing queues with priority (thumbnails > ML) ([Backpressure](../basics/backpressure_load_shedding.md)); regional homing for residency ([Multi-region](../basics/multi_region.md)).

### Security
Encryption at rest (per-object keys, KMS), TLS, malware/CSAM scanning in pipeline, signed URLs, audit logs.

### Pitfalls
- Proxying uploads; blocking "complete" on ML; offset pagination on huge timelines; storing originals on the hot tier forever; orphaned blobs after failed uploads; global dedup leaking whether another user has a file (use per-user dedup or proof-of-possession).

## References
* [Hello Interview: Design Dropbox / Google Drive](https://www.hellointerview.com/learn/system-design/problem-breakdowns/dropbox) · [Dropbox](dropbox.md) · [Instagram](instagram.md)
* [Google Photos engineering talks/papers (Colossus, Bigtable, Spanner)](https://research.google/pubs/) · [S3 multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)

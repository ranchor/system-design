## Problem Statement
Design an **object storage service** like Amazon S3: a flat, highly durable, highly available store for immutable blobs (bytes + metadata) addressed by `bucket/key`, scaling to trillions of objects.

## Clarification Questions to Interviewer
- Object size range (bytes to 5 TB)? Mostly large? Many small?
- Durability (11 nines) and availability (99.99%) targets? Multi-AZ by default?
- Consistency: read-after-write for new objects and overwrites/lists?
- Features: versioning, multipart upload, lifecycle, presigned URLs, ACL/IAM, cross-region replication, storage classes?
- Operations per second per prefix? Egress and cost constraints?

## Requirements
### Functional Requirements
- **Buckets**: create/delete/list. **Objects**: `PUT`, `GET` (range), `DELETE`, `HEAD`, `LIST` by prefix.
- **Multipart upload** for large objects; **versioning**; **presigned URLs**; per-object metadata/tags.
- Access control (bucket policy/IAM).
#### Below the line (out of scope)
- Query-in-place (S3 Select), event notifications detail, object lock/legal hold, storage analytics.

### Non-Functional Requirements
- **Durability** ≥ 99.999999999% via replication/erasure coding across AZs plus continuous integrity checks.
- **Availability** 99.99% for reads; scales horizontally; throughput scales with prefix partitioning.
- **Strong consistency** for read-after-write and list-after-write (S3 offers this today).
- **Cost efficiency**: erasure coding, tiering.
#### Below the line (out of scope)
- POSIX semantics; in-place partial updates (objects are immutable; overwrite = new version).

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 1 trillion objects, avg 1 MB → 1 EB logical; with 1.5× erasure overhead ≈ 1.5 EB raw; at 20 TB/drive ≈ 75M drives total across the fleet (illustrates fleet scale and constant failure).
- Metadata: 1T objects × ~500 B ≈ 500 TB of metadata: must be a sharded, replicated store.
- Peak 10M req/s → front-end tier of thousands of stateless servers.
- Annual disk failure rate 1-3% → thousands of disks fail daily → automated repair is core.

## High-level API design
```
PUT    /{bucket}/{key}              (Content-MD5/checksum, metadata headers) → ETag, versionId
GET    /{bucket}/{key}?versionId=…  (Range) → bytes
DELETE /{bucket}/{key}              → delete marker (versioned) / removal
GET    /{bucket}?prefix=&delimiter=&continuation-token=   → list
POST   /{bucket}/{key}?uploads → uploadId;  PUT …?partNumber=n&uploadId=…;  POST …?uploadId=… (complete)
```
Auth via signed requests (SigV4) and presigned URLs ([Large blobs](../basics/large_blobs.md), [API design](../basics/api_design.md)).

## Data Model
- **Metadata store** (sharded KV/LSM): key `(bucket, key, versionId)` → `{size, checksum, storage class, user metadata, ACL, chunk map [(chunkId, node/AZ placement)], created, delete-marker}`. Partitioned by hash/range of `(bucket, key-prefix)`; supports ordered **prefix listing** ([Data partitioning](../basics/data_partitioning.md), [WAL & LSM](../basics/wal_lsm.md)).
- **Data store**: immutable **chunks/extents** on storage nodes, content checksummed; objects are lists of chunks.
- **Bucket/account/IAM metadata** in a strongly consistent store (small).

## High Level System Design
```
Client ─▶ LB/DNS ─▶ Front-end (auth, routing, throttling) ─┬─▶ Metadata service (sharded, replicated; consensus per shard)
                                                           └─▶ Data service ─▶ Storage nodes (chunk placement across AZs)
Background: replication/repair · scrubber (checksum) · garbage collector · lifecycle · placement/rebalancer
```
- **PUT**: front end authenticates → streams bytes to storage nodes: split into chunks, **erasure-code** (e.g. 10+4 across AZs) or replicate 3× → once durably written to a quorum, **commit metadata** (atomic) → return `200` with ETag. Metadata commit is the visibility point (gives read-after-write).
- **GET**: lookup metadata → fetch enough chunks (any k of n for EC; parallel, hedged requests to tail-latency) → verify checksum → stream; support `Range`.
- **LIST**: ordered scan over the metadata shard by key prefix; paginated with continuation tokens; "directories" are just prefix + delimiter conventions.
- **Delete**: with versioning, write a **delete marker**; actual bytes reclaimed by **garbage collection** after retention (reference counting/mark-sweep over chunk references).

## Deep Dive
### Durability
- **Placement**: spread chunks across AZs/racks/disks (failure domains). **Erasure coding** gives ~1.4-1.5× overhead vs 3× for replication with comparable durability; replicate small objects and hot metadata.
- **Continuous scrubbing**: re-read and verify checksums; repair from remaining chunks; track repair backlog as an SLO ([Checksums](../basics/checksums.md), [Merkle trees](../basics/merkle_tree.md)).
- **Write path integrity**: end-to-end checksum from client (`Content-MD5`/CRC32C/SHA-256), verified at the front end and storage node.
- **Fast repair**: prioritize objects with least redundancy; rack-aware repair traffic throttling.

### Consistency and metadata scaling
- Metadata sharded by key range with per-shard **consensus (Paxos/Raft)** → strong consistency without a global bottleneck ([Consensus](../basics/consensus.md)).
- **Hot prefix**: partitions auto-split on load; encourage key randomization; per-prefix rate limits give request-rate scaling ([Hot keys](../basics/hot_keys.md)).
- **Listing**: ordered index; avoid scanning everything with delimiters and pagination; versioned listing is separate.

### Multipart upload
Initiate → upload parts in parallel (each stored as independent chunk sets, ETag per part) → **complete** stitches the part list into one metadata record (no data copy). Abort/lifecycle cleans orphaned parts.

### Versioning, lifecycle, storage classes
Versioning = keep multiple metadata entries per key; lifecycle rules transition (standard → infrequent → archive) and expire; archive tiers use slower media with restore delays.

### Replication
Cross-region replication via an async log of metadata changes + copy workers; versioned to avoid conflicts; ordering per key ([Multi-region](../basics/multi_region.md)).

### Security
SigV4/presigned URLs with expiry; bucket policies + IAM; server-side encryption (per-object data keys wrapped by KMS, envelope encryption); block public access defaults; audit logs.

### Performance and cost
Request parallelism by prefix and multipart; byte-range GETs; hedged reads; CDN in front for public content ([CDN](../basics/cdn.md)); small-object overhead mitigated by packing small objects into larger extents internally.

### Pitfalls
- Treating it like a file system (rename = copy+delete); overwriting hot keys concurrently (last writer wins); tiny-object cost explosions; listing huge prefixes; forgetting incomplete multipart cleanup.

## References
* [AWS: Amazon S3 strong read-after-write consistency](https://aws.amazon.com/s3/consistency/), [S3 user guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
* [Hello Interview: Design a dropbox / blob store](https://www.hellointerview.com/learn/system-design/problem-breakdowns/dropbox) · [Blob storage](../basics/blob_storage.md) · [GFS/HDFS](file_storage_gfs_hdfs.md) · [Facebook f4 / Haystack papers](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf)

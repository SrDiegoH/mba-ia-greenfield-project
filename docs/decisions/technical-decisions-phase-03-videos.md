---
scope_type: phase
related_phases: [3]
status: decided
date: 2026-06-27
scope_description: "Video upload pipeline: object storage, async processing queue, video worker (FFmpeg), streaming with range requests, unique URL identifiers, and video lifecycle management."
---

# Technical Decisions — Phase 03: Upload e Processamento de Vídeos

_Subprojects in scope:_

- `nestjs-project/` — backend delivering all video endpoints (initiate upload, confirm upload, stream, download, list, update, delete), the async processing queue (BullMQ + Redis), the video worker (FFmpeg), and the object storage integration (MinIO/S3).

> **Note on Object Storage:** MinIO is not an open decision — the architecture diagram (`docs/diagrams/software-arch.mermaid`) already specifies S3-compatible storage with MinIO locally. The decisions here cover _how_ to use it (presigned URLs, bucket/key layout, streaming strategy), not _which_ storage.

---

## TD-01: Queue Technology for Video Processing

**Scope:** Backend + Infrastructure

**Capability:** Processamento assíncrono de vídeos após upload

**Context:** Phase 03 requires a background processing queue so that video encoding (FFmpeg) does not block the API. The project plan leaves this as a TBD ("Message Queue (TBD)"). The chosen technology determines the new infrastructure service, the NestJS integration library, and the retry/failure model.

**Options:**

### Option A: BullMQ + Redis
- NestJS 11 provides `@nestjs/bullmq` as an official first-class integration. BullMQ is the actively-maintained successor to Bull v4 (same team, Taskforce.sh). Jobs are stored in Redis, which is lightweight and container-friendly.
- **Pros:** Official NestJS integration with DI-native `@Processor`/`WorkerHost` pattern. TypeScript-native API. Per-job `attempts` + `backoff` configuration covers FR-008 (3 retries, fixed interval) without manual retry code. `lockDuration` option prevents job orphaning on long-running FFmpeg processes. Single Redis dependency serves both the queue and potential future uses (caching, rate limiting).
- **Cons:** Redis is a new infrastructure service (not in Phase 01/02 stack). In-memory by default — Redis restart loses pending jobs (acceptable for dev; production would use Redis persistence). BullMQ requires more setup than simpler alternatives.

### Option B: RabbitMQ (AMQP)
- Enterprise-grade message broker with exchange routing, fanout, dead-letter queues. NestJS integration via `@nestjs/microservices` or `@golevelup/nestjs-rabbitmq`.
- **Pros:** Mature, feature-rich, durable messaging. Excellent for complex topologies (fanout, multiple consumers).
- **Cons:** Heavier infrastructure than Redis. Retry and dead-letter setup is more verbose. Overkill for a single-queue, single-consumer use case. `@golevelup/nestjs-rabbitmq` is a third-party library, not official NestJS.

### Option C: pg-boss (PostgreSQL-backed queue)
- Queue implementation on top of PostgreSQL — eliminates Redis entirely.
- **Pros:** No new infrastructure. Jobs are durable by default (PostgreSQL persistence). Single database for app state + queue.
- **Cons:** Performance degrades under medium-to-high load vs. Redis. Advanced features (priority, rate limiting) are more limited. Adds queue complexity to the PostgreSQL container that serves business data.

### Option D: Bull v4
- Predecessor to BullMQ by the same team.
- **Cons:** Deprecated — the BullMQ team no longer maintains Bull v4. Must not be used in new projects.

**Recommendation:** **Option A (BullMQ + Redis)** — Official NestJS integration, TypeScript-native, native retry/backoff support, and lightweight Redis infrastructure. The single-queue use case is exactly what BullMQ is optimized for.

**Decision:** A (BullMQ + Redis)

**Configuration defaults chosen:**
- Max attempts: 3 (`attempts: 3`)
- Retry interval: 5 seconds fixed (`backoff: { type: 'fixed', delay: 5000 }`)
- Worker concurrency: 2 (2 videos in parallel per worker instance)
- Job lock timeout: 300 000 ms (5 minutes) — BullMQ default is 30 s, which is insufficient for FFmpeg processing of large files

---

## TD-02: Upload Strategy (Files up to 10 GB)

**Scope:** Backend + Client protocol

**Capability:** Upload de vídeos de até 10 GB sem travar a API

**Context:** Uploading files up to 10 GB through a NestJS API would consume all process memory and block the event loop. The architecture must route file bytes directly to object storage, bypassing the API.

**Options:**

### Option A: Pre-signed PUT URL (direct client → MinIO)
- The API generates a time-limited signed URL for a PUT operation. The client sends the file directly to MinIO using the signed URL. The API never receives the file bytes.
- **Pros:** Zero bytes transit through Node.js — API stays responsive for all other requests. MinIO supports single-part PUT for objects up to 5 TiB (no S3 5 GB single-part limit). Simplest possible protocol: `POST /videos` → API returns `upload_url` → client PUTs → client POSTs `upload-complete`. URL validity of 15 minutes is sufficient for a 10 GB upload on a reasonable connection.
- **Cons:** Client must perform two API calls (initiate + confirm). File integrity must be verified at confirm-time (client signals completion; worker verifies by downloading).

### Option B: S3 Multipart Upload
- Client initiates multipart upload, uploads in parts (5 MB–5 GB each), then calls `CompleteMultipartUpload`. Each part gets its own presigned URL.
- **Pros:** Required for AWS S3 real files > 5 GB (S3 limits single-part PUT to 5 GB). More resilient to network interruptions (resume at part level).
- **Cons:** Significantly more complex client protocol (initiate/upload-parts/complete). MinIO does not have the 5 GB single-part limit, making this unnecessary in the local environment. Added complexity without benefit for the 10 GB target on MinIO.

### Option C: API as upload proxy (streaming)
- Client sends the file to the API, which streams it to MinIO.
- **Cons:** All bytes transit through Node.js. Even with streaming (no buffering), the Node.js process handles the I/O and occupies the TCP connection for the duration of the upload. Violates the requirement directly.

### Option D: TUS protocol (resumable uploads)
- Standard resumable upload protocol. Client can resume failed uploads.
- **Cons:** Requires dedicated TUS server or library. Complex client implementation. Out of scope for this phase.

**Recommendation:** **Option A (Pre-signed PUT URL)** — Simplest approach that fully satisfies the requirement. MinIO removes the 5 GB constraint, making multipart unnecessary.

**Decision:** A (Pre-signed PUT URL, 15-minute validity)

**AWS SDK configuration for MinIO:**
```
endpoint: http://minio:9000   (Docker Compose service name — never localhost)
forcePathStyle: true          (MinIO requires path-style, not virtual-hosted)
region: us-east-1             (any value; MinIO ignores it but SDK requires it)
```

---

## TD-03: Streaming Strategy (Range Requests)

**Scope:** Backend

**Capability:** Reprodução via streaming sem download completo

**Context:** Video streaming requires support for HTTP Range requests (partial content, RFC 7233) so that players can seek to any position without downloading the full file. The strategy defines whether the API proxies the stream or redirects to storage.

**Options:**

### Option A: API proxy with range requests (pass-through)
- The API receives the client's `Range` header, passes it to MinIO via `GetObjectCommand({ Range: rangeHeader })`, and pipes the resulting stream back to the client with appropriate response headers (`206 Partial Content`, `Content-Range`, `Accept-Ranges`).
- **Pros:** MinIO/storage credentials never leave the server. Allows per-endpoint access control (streaming public, download authenticated) without bucket policies. Simple Node.js implementation (pipe stream). Works in Docker local environment where MinIO is not reachable from the browser.
- **Cons:** All streaming bytes transit through Node.js (slight overhead vs. direct storage access). In production at scale, a CDN would be more efficient.

### Option B: Redirect to pre-signed GET URL
- API generates a short-lived signed GET URL for the video file and redirects the client (`302 Found`). The client fetches directly from MinIO.
- **Pros:** Zero bytes through Node.js in production. Better for CDN integration.
- **Cons:** Exposes MinIO's internal Docker network URL (`http://minio:9000`) to the client browser — not reachable outside Docker. Requires MinIO to be publicly accessible, which it is not in the local dev setup. Deferred to production CDN setup.

### Option C: Serve files from the filesystem
- Worker writes processed files to a shared volume; API serves from disk.
- **Cons:** Not scalable. Files lost on container restart. Requires shared volume between API and worker containers.

**Recommendation:** **Option A (API proxy with range requests)** — Correct behavior in the local Docker environment. Storage credentials stay server-side. Access control is trivially enforced at the controller layer.

**Decision:** A (API proxy, range requests passthrough)

---

## TD-04: Video Worker Architecture

**Scope:** Backend + Infrastructure

**Capability:** Processamento assíncrono: extração de metadados e geração de thumbnail

**Context:** FFmpeg is CPU-intensive. Running it in the same Node.js process as the API would starve the event loop during processing. The architecture must isolate the worker.

**Options:**

### Option A: NestJS standalone application, same project, separate Docker container
- A second entry point (`src/worker.ts`) that calls `NestFactory.createApplicationContext(VideoProcessingModule)` (no HTTP server). Built from the same `nestjs-project/` directory but run via a separate `Dockerfile.worker`. Registered as a `video-worker` service in `compose.yaml`.
- **Pros:** Shares the same `package.json`, TypeORM entities, config factories, and code conventions. No code duplication. CPU isolation — FFmpeg runs in a separate container and does not affect API response times. Independently scalable (add more worker replicas in Compose). Standard NestJS BullMQ worker pattern. FFmpeg installed only in `Dockerfile.worker` (keeps API image lean).
- **Cons:** Two Docker build targets from the same Dockerfile base. Shared project means worker changes require a full project rebuild.

### Option B: Worker running in the API process
- BullMQ worker registered in the same NestJS application context as the HTTP API.
- **Cons:** FFmpeg is CPU-intensive; blocking the event loop during processing degrades API latency. Fails SC-001 (API must remain responsive during processing). No independent scaling.

### Option C: Separate NestJS project (own package.json)
- A completely separate `nestjs-worker/` project.
- **Cons:** Duplicates dependencies, configs, entities, and CI pipeline. Overkill when a separate entry point achieves the same isolation.

### Option D: Plain Node.js script (no NestJS)
- A bare Node.js script consuming from Redis directly.
- **Cons:** Loses NestJS DI, TypeORM, and all project conventions. Duplicate code for DB access and storage.

**Recommendation:** **Option A (NestJS standalone, separate container)** — Best balance of code reuse and process isolation.

**Decision:** A (NestJS standalone application in `src/worker.ts`, `Dockerfile.worker`, `video-worker` Compose service)

**FFmpeg toolchain:** `fluent-ffmpeg` (high-level API) + `@ffprobe-installer/ffprobe` (bundled ffprobe binary). System `ffmpeg` installed in `Dockerfile.worker` via `apt-get install -y ffmpeg`.

---

## TD-05: Unique Video URL Identifier

**Scope:** Backend

**Capability:** URL única por vídeo, sem conflito com outros

**Context:** Each video must have an identifier in its URL that is globally unique even when titles are identical. The identifier must be immutable (should not change when the title changes).

**Options:**

### Option A: UUID v4 as the primary key (the `id` field itself)
- The video's URL identifier is the same UUID generated as its primary key by `@PrimaryGeneratedColumn('uuid')`. URL pattern: `GET /videos/{uuid}`.
- **Pros:** Collision probability is negligible (2^122 combinations — SC-005). Consistent with project conventions (User, Channel, all tokens use UUID). No extra field or generation logic. Immutable by definition (primary key does not change). No DB query needed to check for slug conflicts.
- **Cons:** UUIDs are longer and less readable than slugs (36 chars vs. short slugs). Not guessable — good for security, but no "friendly URL" for SEO.

### Option B: Nanoid or CUID2 (short unique identifier)
- A shorter, URL-friendly ID generated at insert time. Requires uniqueness check or collision strategy.
- **Pros:** Shorter URL (21 chars for nanoid). More shareable.
- **Cons:** Requires custom generation logic and uniqueness enforcement. No significant functional benefit over UUID for this phase.

### Option C: Slug derived from title
- Sanitize the title into a URL-safe slug. Add a random suffix to ensure uniqueness.
- **Pros:** Human-readable URL.
- **Cons:** Slug must be decoupled from the title (or the URL changes on title edit). Requires sanitization, uniqueness check, and collision suffix logic — same complexity as Channel.nickname but without the display-name benefit.

**Recommendation:** **Option A (UUID v4 as PK)** — Consistent with project conventions, zero additional code, immutable, negligible collision probability.

**Decision:** A (UUID v4 — `@PrimaryGeneratedColumn('uuid')`)

---

## TD-06: Video Status Lifecycle

**Scope:** Backend + Worker

**Capability:** Ciclo de status do vídeo e tratamento de falha no processamento

**Context:** The video moves through lifecycle states from creation to playback availability. The state machine must be clear about transitions, failure handling, and idempotency.

**Options:**

### Option A: Four-state machine (draft → processing → ready | error)
- `draft`: video pre-registered, file not yet uploaded or not confirmed.
- `processing`: upload confirmed, job enqueued, worker running FFmpeg.
- `ready`: processing succeeded; video is playable.
- `error`: processing failed after all retries; `error_cause` field holds the last failure message (truncated to 2048 chars).
- **Pros:** Minimal states that directly map to client-visible behavior (streaming only available in `ready`). Idempotency enforced by worker: if status is not `processing` at execution time, job is discarded without modification.
- **Cons:** No separate `published`/`unlisted` states (out of scope — `ready` equals published in Phase 03).

### Option B: Six-state machine (draft → uploading → uploaded → processing → ready | error)
- Separates "file being uploaded" from "upload confirmed".
- **Cons:** The client controls the upload (PUT to MinIO), not the API. The API has no visibility into upload progress — `uploading` is not a meaningful API state. Unnecessary complexity.

**Recommendation:** **Option A (four states)** — Minimal, maps exactly to observable API behavior.

**Decision:** A (four-state: `draft` → `processing` → `ready` | `error`)

**Retry policy (FR-008):** 3 attempts, 5 s fixed backoff. After the final attempt fails, worker updates `status = error` and sets `error_cause = message.slice(0, 2048)`. BullMQ `lockDuration = 300 000 ms` prevents jobs from becoming orphaned during long FFmpeg runs.

---

## Decisions Summary

| ID | Decision | Recommendation | Choice |
|----|----------|---------------|--------|
| TD-01 | Queue Technology | BullMQ + Redis | A (BullMQ + Redis) |
| TD-02 | Upload Strategy | Pre-signed PUT URL | A (Pre-signed PUT URL, 15 min) |
| TD-03 | Streaming Strategy | API proxy with range requests | A (API proxy, range passthrough) |
| TD-04 | Worker Architecture | NestJS standalone, separate container | A (NestJS standalone, Dockerfile.worker) |
| TD-05 | Unique URL Identifier | UUID v4 as PK | A (UUID v4 — PrimaryGeneratedColumn) |
| TD-06 | Video Status Lifecycle | Four-state machine | A (draft → processing → ready \| error) |

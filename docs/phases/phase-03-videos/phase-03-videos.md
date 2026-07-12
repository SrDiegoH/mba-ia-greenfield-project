---
kind: phase
name: phase-03-videos
sources_mtime:
  docs/project-plan.md: "2026-06-27T00:00:00-03:00"
  docs/decisions/technical-decisions-phase-03-videos.md: "2026-06-27T00:00:00-03:00"
  docs/phases/phase-03-videos/context.md: "2026-06-27T00:00:00-03:00"
  docs/phases/phase-03-videos/library-refs.md: "2026-06-27T00:00:00-03:00"
---

# Phase 03 — Upload e Processamento de Vídeos

## Objective

Deliver the complete video lifecycle on StreamTube: direct upload to object storage (no bytes through the API), asynchronous processing with FFmpeg (metadata extraction + thumbnail generation), public streaming with range request support, authenticated download, and full CRUD management for video owners.

---

## Technical Specifications

### Data Model

#### Entity: `Video`

Table: `videos`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, NOT NULL, DEFAULT uuid_generate_v4() | Unique video identifier; also the URL identifier (TD-05) |
| `title` | `varchar(255)` | NOT NULL | Video title provided by creator |
| `status` | `video_status_enum` | NOT NULL, DEFAULT `'draft'` | Lifecycle state (TD-06) |
| `channel_id` | `uuid` | NOT NULL, FK → `channels.id` ON DELETE RESTRICT | Owner channel |
| `storage_key` | `varchar` | NOT NULL | S3/MinIO object key for the video file; pattern `videos/{id}/original.{ext}` |
| `thumbnail_key` | `varchar` | nullable | S3/MinIO object key for the thumbnail; set after processing; pattern `videos/{id}/thumbnail.jpg` |
| `duration_seconds` | `integer` | nullable | Duration in whole seconds extracted by ffprobe (`Math.floor`) |
| `processing_metadata` | `jsonb` | nullable | `{ width, height, codec, bitrate_kbps }` extracted by ffprobe |
| `error_cause` | `varchar(2048)` | nullable | Last processing failure message; set only on `status = error` |
| `created_at` | `timestamp` | NOT NULL, DEFAULT now() | Auto-populated by TypeORM |
| `updated_at` | `timestamp` | NOT NULL, DEFAULT now() | Auto-populated by TypeORM |

**Indexes:**
- `idx_videos_channel_id` on `(channel_id)` — FK lookup optimization
- `idx_videos_channel_id_created_at` on `(channel_id, created_at DESC)` — paginated list query
- `idx_videos_status` on `(status)` — status filter queries

**`video_status_enum` values:** `draft`, `processing`, `ready`, `error`

**State Machine (TD-06):**
```
[initial]  →  draft  →  processing  →  ready
                                   ↘  error
```
- `draft`: record created; file not yet confirmed as uploaded.
- `processing`: `POST /videos/:id/upload-complete` called; BullMQ job enqueued.
- `ready`: FFmpeg succeeded; `thumbnail_key`, `duration_seconds`, `processing_metadata` populated.
- `error`: All retry attempts exhausted; `error_cause` populated.

**Idempotency rule (worker):** If `status !== processing` when worker picks up the job, the job is discarded without modification (covers re-delivery and delete-during-processing scenarios).

---

### API Contracts

#### Authentication Matrix

| Endpoint | Access Level | Rule |
|----------|--------------|------|
| `POST /videos` | AUTHENTICATED | Any user with a channel |
| `POST /videos/:id/upload-complete` | OWNER | Channel owner of the video |
| `GET /videos/:id` | PUBLIC | Any user |
| `GET /videos/:id/stream` | PUBLIC | Any user; video must be `ready` |
| `GET /videos/:id/download` | AUTHENTICATED | Any authenticated user; video must be `ready` |
| `GET /channels/:channelId/videos` | OWNER | Channel owner only |
| `PATCH /videos/:id` | OWNER | Channel owner of the video |
| `DELETE /videos/:id` | OWNER | Channel owner of the video |

OWNER = authenticated user whose channel's `user_id` matches the JWT `sub`.

---

#### Contract 1: `POST /videos` — Initiate Upload

**Authentication:** Bearer JWT required

**Request body:**
```json
{
  "title": "string (1–255 chars, required)",
  "file_size": "integer (1–10737418240 bytes, required)",
  "mime_type": "string (must match /^video\\// regex, required)"
}
```

**Response 201:**
```json
{
  "id": "uuid",
  "status": "draft",
  "upload_url": "https://minio:9000/streamtube/videos/{id}/original.{ext}?X-Amz-...",
  "storage_key": "videos/{id}/original.{ext}"
}
```

**`storage_key` format:** `videos/{uuid}/original.{ext}` where `ext` is derived from the declared `mime_type` (e.g., `video/mp4` → `mp4`).
**`upload_url` validity:** 15 minutes (900 s).

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 400 | `VALIDATION_ERROR` | Invalid body (missing fields, wrong types, mime_type not video/*) |
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 403 | `CHANNEL_REQUIRED` | Authenticated user has no channel |

---

#### Contract 2: `POST /videos/:id/upload-complete` — Confirm Upload

**Authentication:** Bearer JWT required (OWNER)

**Response 200:**
```json
{
  "id": "uuid",
  "status": "processing"
}
```

**Behavior:** Enqueues BullMQ job with `{ videoId, storageKey }` and `{ jobId: videoId, attempts: 3, backoff: { type: 'fixed', delay: 5000 } }`, then updates `status = processing`.

**Idempotency:** If status is already `processing`, returns `400 INVALID_VIDEO_STATUS`.

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 400 | `INVALID_VIDEO_STATUS` | Video is not in `draft` status |
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 403 | `VIDEO_OWNERSHIP` | Requester does not own the video's channel |
| 404 | `VIDEO_NOT_FOUND` | Video ID does not exist |

---

#### Contract 3: `GET /videos/:id` — Get Video Details

**Authentication:** None (public)

**Response 200:**
```json
{
  "id": "uuid",
  "title": "string",
  "status": "draft | processing | ready | error",
  "channel_id": "uuid",
  "storage_key": "string",
  "thumbnail_key": "string | null",
  "duration_seconds": "integer | null",
  "processing_metadata": "object | null",
  "error_cause": "string | null",
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 404 | `VIDEO_NOT_FOUND` | Video ID does not exist |

---

#### Contract 4: `GET /videos/:id/stream` — Stream Video

**Authentication:** None (public)

**Range request:** Client sends `Range: bytes=<start>-<end?>`. API passes range header to MinIO via `GetObjectCommand({ Range })`.

**Response 206 (range request):**
- Headers: `Content-Type`, `Content-Length`, `Content-Range: bytes <start>-<end>/<total>`, `Accept-Ranges: bytes`
- Body: partial video bytes piped from MinIO

**Response 200 (no range header):**
- Headers: `Content-Type`, `Content-Length`, `Accept-Ranges: bytes`
- Body: full video stream piped from MinIO

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 404 | `VIDEO_NOT_FOUND` | Video does not exist |
| 409 | `VIDEO_NOT_READY` | Video status is not `ready` |
| 416 | `VIDEO_RANGE_NOT_SATISFIABLE` | Requested range is beyond file size (S3 SDK 416 response) |

---

#### Contract 5: `GET /videos/:id/download` — Download Video

**Authentication:** Bearer JWT required (any authenticated user, not only owner)

**Response 200:**
- Headers: `Content-Type`, `Content-Length`, `Content-Disposition: attachment; filename="{encoded_filename}"`
- Body: full video file piped from MinIO (no `Range` header; full object)

**Filename derivation:** Extracted from `storage_key` (last path segment: `original.ext`).

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 404 | `VIDEO_NOT_FOUND` | Video does not exist |
| 409 | `VIDEO_NOT_READY` | Video status is not `ready` |

---

#### Contract 6: `GET /channels/:channelId/videos` — List Channel Videos

**Authentication:** Bearer JWT required (OWNER)

**Query parameters:**
| Param | Type | Default | Max | Description |
|-------|------|---------|-----|-------------|
| `page` | integer | 1 | — | Page number (1-based) |
| `limit` | integer | 20 | 100 | Items per page |

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "string",
      "status": "string",
      "created_at": "ISO8601"
    }
  ],
  "total": "integer",
  "page": "integer",
  "limit": "integer",
  "has_next": "boolean"
}
```

**Ordering:** `created_at DESC` (most recent first, enforced by `idx_videos_channel_id_created_at`).

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 403 | `VIDEO_OWNERSHIP` | Requester does not own the channel |
| 404 | `CHANNEL_NOT_FOUND` | Channel ID does not exist |

---

#### Contract 7: `PATCH /videos/:id` — Update Video Title

**Authentication:** Bearer JWT required (OWNER)

**Request body:**
```json
{ "title": "string (1–255 chars, required)" }
```

**Response 200:** Full updated video object (same shape as Contract 3).

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 400 | `VALIDATION_ERROR` | Invalid body |
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 403 | `VIDEO_OWNERSHIP` | Requester does not own the video |
| 404 | `VIDEO_NOT_FOUND` | Video does not exist |

---

#### Contract 8: `DELETE /videos/:id` — Delete Video

**Authentication:** Bearer JWT required (OWNER)

**Response 204:** No body.

**Behavior:**
1. Verify ownership (403 if not owner, 404 if not found)
2. Cancel BullMQ job (best-effort: `queue.getJob(videoId)` → `job.remove()`; no-op if job already consumed)
3. Delete `storage_key` from MinIO (tolerates `NoSuchKey`)
4. Delete `thumbnail_key` from MinIO if set (tolerates `NoSuchKey`)
5. Delete database record

**Errors:**
| Status | Error Code | Condition |
|--------|-----------|-----------|
| 401 | `UNAUTHORIZED` | No or invalid JWT |
| 403 | `VIDEO_OWNERSHIP` | Requester does not own the video |
| 404 | `VIDEO_NOT_FOUND` | Video does not exist |

---

### Error Catalog

| Error Code | HTTP Status | Domain Exception Class | Description |
|-----------|-------------|----------------------|-------------|
| `VIDEO_NOT_FOUND` | 404 | `VideoNotFoundException` | Video record does not exist |
| `VIDEO_NOT_READY` | 409 | `VideoNotReadyException` | Operation requires `status = ready` |
| `VIDEO_OWNERSHIP` | 403 | `VideoOwnershipException` | Authenticated user does not own the video's channel |
| `CHANNEL_REQUIRED` | 403 | `ChannelRequiredException` | Operation requires the user to have a channel |
| `CHANNEL_NOT_FOUND` | 404 | `ChannelNotFoundException` | Channel record does not exist |
| `INVALID_VIDEO_STATUS` | 400 | `InvalidVideoStatusException` | Status transition is not allowed |
| `VIDEO_RANGE_NOT_SATISFIABLE` | 416 | `VideoRangeNotSatisfiableException` | Range header refers to bytes beyond file size |

---

### Events / Messages (BullMQ Queue)

**Queue name:** `video-processing` (constant `VIDEO_QUEUE_NAME` in `videos.constants.ts`)

**Job payload schema (`VideoProcessingJob`):**
```typescript
interface VideoProcessingJob {
  videoId: string;   // UUID of the Video record
  storageKey: string; // e.g. "videos/{uuid}/original.mp4"
}
```

**Job options (set at enqueue time):**
```typescript
{
  jobId: videoId,         // deduplicates concurrent confirm-upload calls
  attempts: 3,
  backoff: { type: 'fixed', delay: 5000 },
}
```

**Worker options:**
```typescript
@Processor(VIDEO_QUEUE_NAME, { lockDuration: 300_000 })
```
`lockDuration: 300_000` ms (5 minutes) — prevents job orphaning for large video FFmpeg runs. BullMQ auto-renews the lock while the job is active.

**Worker concurrency:** 2 videos processed in parallel per worker instance.

**Processing pipeline (worker):**
1. Load `Video` from DB; discard if `status !== processing` (idempotency).
2. Download video from MinIO to `/tmp/{videoId}/original`.
3. Run `ffprobe` → extract `{ duration, width, height, codec, bitrate_kbps }`.
4. Run `fluent-ffmpeg` → capture frame at `00:00:01` as JPEG (`1280x?` size) to `/tmp/{videoId}/thumbnail.jpg`.
5. Upload thumbnail buffer to MinIO at `videos/{videoId}/thumbnail.jpg`.
6. Update DB: `status = ready`, `thumbnail_key`, `duration_seconds = Math.floor(duration)`, `processing_metadata`.
7. Cleanup: `fs.rmSync('/tmp/{videoId}', { recursive: true, force: true })` (in `finally`).

**Failure path (any step after step 1):**
- Rethrow error — BullMQ handles retry according to job options.
- On final attempt (`job.attemptsMade >= attempts - 1`): update DB `status = error`, `error_cause = message.slice(0, 2048)`.
- Cleanup always runs in `finally`.

**Cancellation (from DELETE):**
```typescript
const job = await this.queue.getJob(videoId);
if (job) await job.remove();
```
Best-effort: if the job is already being processed by the worker, the worker discards it when it finds the DB record is gone.

---

## Step Implementations

### SI-03.1 — Infrastructure Setup: Docker Services, Config Namespaces, and Dependencies

**Description:** Install new production dependencies, create `storage.config.ts` and `queue.config.ts` using the `registerAs` pattern from Phase 01, extend the Joi validation schema, and add MinIO, Redis, and `video-worker` services to `compose.yaml`.

**Technical actions:**

- Install production dependencies in `nestjs-project/`: `@nestjs/bullmq@^11.0.0`, `bullmq@^5.0.0`, `ioredis@^5.0.0`, `@aws-sdk/client-s3@^3.750.0`, `@aws-sdk/s3-request-presigner@^3.750.0`, `fluent-ffmpeg@^2.1.3`, `@ffprobe-installer/ffprobe@^1.4.1`. DevDependency: `@types/fluent-ffmpeg@^2.1.27`
- Create `src/config/storage.config.ts` — `registerAs('storage', ...)` reading `STORAGE_ENDPOINT`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`, `STORAGE_BUCKET`
- Create `src/config/queue.config.ts` — `registerAs('queue', ...)` reading `REDIS_HOST` (default `'redis'`), `REDIS_PORT` (default `6379`), `REDIS_PASSWORD` (optional)
- Extend `src/config/env.validation.ts` Joi schema with all new environment variables; update `.env.example`
- Add to `compose.yaml`: MinIO service (image `minio/minio`, ports `9000:9000`/`9001:9001`, health check, persistent volume), Redis service (image `redis:7-alpine`, port `6379:6379`, health check), `video-worker` service (build: `Dockerfile.worker`, `depends_on` with health conditions for `redis`, `db`, `minio`)
- Create `Dockerfile.worker` — Node.js 22 slim base, `apt-get install -y ffmpeg`, copy project, `npm run build`, CMD `node dist/worker.js`
- Add storage + Redis env vars to `nestjs-api` service in `compose.yaml` `depends_on` for `redis` and `minio`

**Dependencies:** None

**Acceptance criteria:**
- `docker compose up -d` starts all services (API, db, mailpit, redis, minio, video-worker) without errors
- `docker compose exec nestjs-api npm start:dev` still serves `GET /` with 200

---

### SI-03.2 — Video Entity, Migration, and Module Scaffold

**Description:** Create the `Video` entity, generate and execute the migration, create `VideoStatus` enum and queue constant, and scaffold `VideosModule` and `VideoProcessingModule`.

**Technical actions:**

- Create `src/videos/videos.constants.ts` — export `VideoStatus` enum (`DRAFT`, `PROCESSING`, `READY`, `ERROR`) and `VIDEO_QUEUE_NAME = 'video-processing'`; export `VideoProcessingJob` interface `{ videoId: string; storageKey: string }`
- Create `src/videos/entities/video.entity.ts` — `@Entity('videos')`, all columns per Data Model, `@ManyToOne(() => Channel)` with `@JoinColumn({ name: 'channel_id' })`
- Generate migration: `npm run migration:generate -- src/database/migrations/CreateVideos`; verify ENUM, table, FK `ON DELETE RESTRICT`, and all three indexes; run `npm run migration:run`
- Create `src/storage/storage.module.ts` (global) and `src/storage/storage.service.ts` — `S3Client` with `forcePathStyle: true`; `onModuleInit()` creates bucket if not exists; methods: `generatePresignedPutUrl`, `getObject`, `putObject`, `deleteObject`
- Scaffold `src/videos/videos.module.ts` — imports `TypeOrmModule.forFeature([Video])`, `BullModule.registerQueue({ name: VIDEO_QUEUE_NAME })`, `StorageModule`; declares `VideosController`, `VideosService`
- Scaffold `src/video-processing/video-processing.module.ts` — imports `BullModule.registerQueue({ name: VIDEO_QUEUE_NAME })`, `TypeOrmModule.forFeature([Video])`, `StorageModule`; declares `VideoProcessingProcessor`
- Register in `AppModule`: add `storage.config` and `queue.config` to `ConfigModule.load`, `BullModule.forRootAsync` with Redis connection, `StorageModule`, `VideosModule`. **`VideoProcessingModule` not imported in AppModule** — only in `worker.ts`
- Create `src/worker.ts` — `NestFactory.createApplicationContext(VideoProcessingModule)`, no HTTP server, `enableShutdownHooks()`; create type declaration `src/types/ffprobe-installer.d.ts` for `@ffprobe-installer/ffprobe`

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/entities/video.entity.integration-spec.ts` | Integration | NOT NULL on `storage_key`, invalid enum rejected, FK constraint on `channel_id`, status defaults to `draft` |

**Dependencies:** SI-03.1

**Acceptance criteria:**
- `npm run migration:run` creates `video_status_enum` and `videos` table with all columns, FK, and indexes
- `npx tsc --noEmit` exits with code 0

---

### SI-03.3 — Video Upload Initiation (`POST /videos`)

**Description:** Implement the `POST /videos` endpoint that pre-registers the video as `draft` and returns a pre-signed PUT URL for direct upload to MinIO.

**Technical actions:**

- Create `src/videos/dto/initiate-upload.dto.ts` — `title: string` (`@IsString`, `@MinLength(1)`, `@MaxLength(255)`), `file_size: number` (`@IsInt`, `@Min(1)`, `@Max(10737418240)`), `mime_type: string` (`@IsString`, `@Matches(/^video\//)`); all with `@ApiProperty()`
- Implement `VideosService.initiateUpload(userId, dto)`: find channel by `userId`; throw `ChannelRequiredException` if none; generate `storageKey = videos/{uuid}/original.{ext}`; call `StorageService.generatePresignedPutUrl`; save `Video` with `status=DRAFT`; return `{ id, status, upload_url, storage_key }`
- Implement `POST /videos` in `VideosController` — `@HttpCode(201)`, `@ApiBearerAuth`, full `@ApiResponse` set

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/videos.service.spec.ts` | Unit | Successful creation returns `upload_url`; `ChannelRequiredException` when no channel; `storage_key` follows pattern `videos/{uuid}/original.{ext}`; two videos with same title get distinct IDs |
| `src/videos/videos.controller.spec.ts` | Unit | `POST /videos` returns 201 with correct payload; `userId` extracted from JWT |
| `src/videos/videos.service.integration-spec.ts` | Integration | Two videos with same title get distinct UUIDs (US5); `storage_key` is NOT NULL; initial status is `draft`; `channel_id` references existing channel |

**Dependencies:** SI-03.2

**Acceptance criteria:**
- `POST /videos` authenticated with valid `title`, `file_size`, `mime_type` → 201 `{ id, status: "draft", upload_url, storage_key }`
- `mime_type: "image/png"` → 400 VALIDATION_ERROR
- Unauthenticated request → 401 UNAUTHORIZED

---

### SI-03.4 — Upload Confirmation and Queue Enqueue (`POST /videos/:id/upload-complete`)

**Description:** Implement the endpoint that transitions the video to `processing` and enqueues the BullMQ job.

**Technical actions:**

- Implement `VideosService.confirmUpload(videoId, userId)`: find video (404 if not found); verify ownership (403 if not owner); throw `InvalidVideoStatusException` if `status !== DRAFT`; enqueue BullMQ job with `{ jobId: videoId, attempts: 3, backoff: { type: 'fixed', delay: 5000 } }`; update `status = PROCESSING`; return `{ id, status: 'processing' }`
- Implement `POST /videos/:id/upload-complete` in `VideosController` — `@HttpCode(200)`, `@ApiBearerAuth`, full `@ApiResponse` set

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/videos.service.spec.ts` (section `confirmUpload`) | Unit | Status transitions `draft→processing`; `InvalidVideoStatusException` for non-draft; `VideoOwnershipException` for wrong owner; `InternalServerErrorException` when queue.add throws |
| `src/videos/videos.service.integration-spec.ts` (section `confirmUpload`) | Integration | `draft→processing` persists to DB; `InvalidVideoStatusException` for already-processing; `VideoOwnershipException` for other user's channel |

**Dependencies:** SI-03.3

**Acceptance criteria:**
- `POST /videos/:id/upload-complete` for draft video → 200 `{ id, status: "processing" }`
- Second call for same video → 400 INVALID_VIDEO_STATUS
- Call by non-owner → 403 VIDEO_OWNERSHIP

---

### SI-03.5 — Video Processing Worker (FFmpeg)

**Description:** Implement the BullMQ processor that downloads the video from MinIO, extracts metadata with ffprobe, generates a thumbnail with FFmpeg, and updates the video record.

**Technical actions:**

- Create `src/video-processing/video-processing.processor.ts` extending `WorkerHost` with `@Processor(VIDEO_QUEUE_NAME, { lockDuration: 300_000 })`
- Implement `override async process(job: Job<VideoProcessingJob>)`: idempotency check (`status !== processing` → discard); download to `/tmp/{videoId}/original`; `ffprobe` → extract metadata; `fluent-ffmpeg` → thumbnail at `00:00:01`, `1280x?`; upload thumbnail via `StorageService.putObject`; DB update `status=READY` with all fields; failure path: rethrow, on final attempt set `status=ERROR` with `error_cause.slice(0, 2048)`; `finally`: `fs.rmSync` cleanup

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/video-processing/video-processing.processor.spec.ts` | Unit | Full success path → `status=READY` with metadata; `status !== processing` → discard (idempotency); ffprobe error → rethrow; `attemptsMade >= 3` → `status=ERROR` with truncated `errorCause`; fluent-ffmpeg failure → error without partial `status=ready` |

**Dependencies:** SI-03.2

**Acceptance criteria:**
- Worker container starts and connects to Redis queue
- Job enqueued by SI-03.4 is consumed; video transitions to `ready` with `thumbnail_key`, `duration_seconds`, `processing_metadata` populated
- Corrupted file → retries 3x → `status=error` with `error_cause`

---

### SI-03.6 — Public Video Detail and Streaming (`GET /videos/:id`, `GET /videos/:id/stream`)

**Description:** Implement the public endpoint returning video metadata and the streaming endpoint with range request support.

**Technical actions:**

- Implement `VideosService.findById(id)` — find video by UUID; throw `VideoNotFoundException` if not found; return entity
- Implement `VideosService.streamVideo(id, rangeHeader?)` — find video (404); throw `VideoNotReadyException` if `status !== READY`; call `StorageService.getObject(storageKey, rangeHeader)`; handle AWS SDK 416 response as `VideoRangeNotSatisfiableException`; return `{ stream, statusCode, contentType, contentLength, contentRange? }`
- Implement `GET /videos/:id` (`@Public()`) and `GET /videos/:id/stream` (`@Public()`) in `VideosController`; pipe stream to `@Res() res: Response` with correct headers (`Accept-Ranges: bytes`, `Content-Range` when 206)

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/videos.service.spec.ts` | Unit | `findById` returns entity; `streamVideo` throws when not ready; range header forwarded to storage service |
| `test/videos.e2e-spec.ts` | E2E | `GET /videos/:id` returns 200 with video metadata; `GET /videos/:id` non-existent → 404; `GET /videos/:id/stream` for `ready` video returns 206 with `Content-Range`; `GET /videos/:id/stream` for non-ready → 409 |

**Dependencies:** SI-03.5

**Acceptance criteria:**
- `GET /videos/:id` public → 200 with all metadata fields
- `GET /videos/:id/stream` with `Range: bytes=0-1023` → 206 with `Content-Range` header
- `GET /videos/:id/stream` without range → 200 full stream
- Video in `processing` → `GET /videos/:id/stream` → 409 VIDEO_NOT_READY

---

### SI-03.7 — Authenticated Download (`GET /videos/:id/download`)

**Description:** Implement the authenticated download endpoint that proxies the full video file with `Content-Disposition: attachment`.

**Technical actions:**

- Implement `VideosService.downloadVideo(id)` — find video (404); throw `VideoNotReadyException` if not ready; call `StorageService.getObject(storageKey)` without range; derive filename from `storageKey` (last path segment); return `{ stream, contentType, contentLength, filename }`
- Implement `GET /videos/:id/download` in `VideosController` — JWT protected (`@ApiBearerAuth`, no `@Public()`); set `Content-Disposition: attachment; filename="{encodeURIComponent(filename)}"`; pipe stream to response

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/videos.service.spec.ts` | Unit | `downloadVideo` returns stream; throws when not ready |
| `test/videos.e2e-spec.ts` | E2E | Authenticated `GET /videos/:id/download` → 200 with `Content-Disposition: attachment`; unauthenticated → 401; non-ready video → 409 |

**Dependencies:** SI-03.6

**Acceptance criteria:**
- `GET /videos/:id/download` authenticated, `ready` video → 200 with `Content-Disposition: attachment; filename="original.mp4"`
- Unauthenticated → 401 UNAUTHORIZED
- Non-ready video → 409 VIDEO_NOT_READY

---

### SI-03.8 — Video Management: List, Update, Delete

**Description:** Implement the authenticated CRUD endpoints for video management: paginated list by channel, title update, and deletion with storage cleanup.

**Technical actions:**

- Create `src/videos/dto/list-videos-query.dto.ts` — `page: number` (default `1`, min `1`), `limit: number` (default `20`, min `1`, max `100`); use `@Type(() => Number)` for query param coercion
- Create `src/videos/dto/update-video.dto.ts` — `title: string` (`@IsString`, `@MinLength(1)`, `@MaxLength(255)`)
- Implement `VideosService.listChannelVideos(channelId, userId, query)` — find channel (404 if not found); verify ownership (403); query with `findAndCount`, `where: { channel_id: channelId }`, `order: { created_at: 'DESC' }`, `skip/take`; return `{ data, total, page, limit, has_next }`
- Implement `VideosService.updateTitle(videoId, userId, dto)` — find video (404); verify ownership (403); update title; return updated entity
- Implement `VideosService.deleteVideo(videoId, userId)` — find video (404); verify ownership (403); cancel BullMQ job (best-effort); delete storage files (tolerates NoSuchKey); delete DB record
- Implement `GET /channels/:channelId/videos`, `PATCH /videos/:id`, `DELETE /videos/:id` in `VideosController`

**Tests:**

| File | Layer | Verifies |
|------|-------|----------|
| `src/videos/videos.service.spec.ts` | Unit | `listChannelVideos` paginates correctly; `updateTitle` persists new title; `deleteVideo` cancels job and deletes files |
| `src/videos/videos.service.integration-spec.ts` | Integration | Paginated list returns correct order and metadata; `updateTitle` persists; `deleteVideo` removes record |
| `test/videos.e2e-spec.ts` | E2E | Full CRUD: list returns paginated videos; PATCH updates title without changing ID; DELETE removes record and returns 204; non-owner operations return 403 |

**Dependencies:** SI-03.7

**Acceptance criteria:**
- `GET /channels/:channelId/videos` → 200 `{ data: [...], total, page, limit, has_next }`
- `PATCH /videos/:id` → 200 with updated title, same `id`
- `DELETE /videos/:id` → 204 with no body
- All OWNER checks: 403 for wrong user, 404 for missing resource

---

## Authorization Matrix

| Endpoint | Public | Authenticated | Owner |
|----------|--------|---------------|-------|
| `POST /videos` | | ✅ (needs channel) | |
| `POST /videos/:id/upload-complete` | | | ✅ |
| `GET /videos/:id` | ✅ | | |
| `GET /videos/:id/stream` | ✅ | | |
| `GET /videos/:id/download` | | ✅ | |
| `GET /channels/:channelId/videos` | | | ✅ |
| `PATCH /videos/:id` | | | ✅ |
| `DELETE /videos/:id` | | | ✅ |

---

## Dependency Map

```
SI-03.1  (infra, deps, config)
    └── SI-03.2  (entity, migration, modules)
            ├── SI-03.3  (POST /videos — initiate upload)
            │       └── SI-03.4  (POST /videos/:id/upload-complete)
            │               └── SI-03.5  (worker — FFmpeg processing)
            │                       └── SI-03.6  (GET /videos/:id + stream)
            │                               └── SI-03.7  (GET /videos/:id/download)
            │                                       └── SI-03.8  (list, update, delete)
            └── SI-03.5  (worker — independent of SI-03.3 at code level)
```

---

## Deliverables

| Deliverable | SI | Status |
|-------------|-----|--------|
| `compose.yaml` updated with MinIO, Redis, video-worker | SI-03.1 | ✅ implemented |
| `Dockerfile.worker` with FFmpeg | SI-03.1 | ✅ implemented |
| `src/config/storage.config.ts` | SI-03.1 | ✅ implemented |
| `src/config/queue.config.ts` | SI-03.1 | ✅ implemented |
| `src/videos/entities/video.entity.ts` | SI-03.2 | ✅ implemented |
| `src/database/migrations/1783990000000-CreateVideos.ts` | SI-03.2 | ✅ implemented |
| `src/storage/storage.module.ts` + `storage.service.ts` | SI-03.2 | ✅ implemented |
| `src/videos/videos.module.ts`, `videos.service.ts`, `videos.controller.ts` | SI-03.3–SI-03.8 | ✅ implemented |
| `src/video-processing/video-processing.module.ts` + `processor.ts` | SI-03.5 | ✅ implemented |
| `src/worker.ts` | SI-03.2 | ✅ implemented |
| Unit tests (`*.spec.ts`) for service, controller, processor | SI-03.3–SI-03.8 | ✅ implemented |
| Integration tests (`*.integration-spec.ts`) for entity and service | SI-03.2–SI-03.8 | ✅ implemented |
| E2E tests (`test/videos.e2e-spec.ts`) | SI-03.6–SI-03.8 | ✅ implemented |
| `docs/decisions/technical-decisions-phase-03-videos.md` | Research | ✅ created |
| `docs/phases/phase-03-videos/context.md` | Plan Context | ✅ created |
| `docs/phases/phase-03-videos/validation.md` (status: clean) | Plan Validate | ✅ created |
| `docs/phases/phase-03-videos/library-refs.md` | Plan Resolve | ✅ created |
| `docs/phases/phase-03-videos/phase-03-videos.md` (this file) | Plan Build | ✅ created |
| `CLAUDE.md` updated with video section | Post-implementation | ✅ updated |
| `nestjs-project/CLAUDE.md` updated with video endpoints and worker | Post-implementation | ✅ updated |

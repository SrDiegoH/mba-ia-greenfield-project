# phase-03-videos — Progress

**Status:** completed
**SIs:** 8/8 completed

### SI-03.1 — Infrastructure Setup: Docker Services, Config Namespaces, and Dependencies
- **Status:** completed
- **Tests:** no tests (infrastructure only)
- **Observations:** Added MinIO (image `minio/minio`, ports 9000/9001, persistent volume `minio_data`, health check via `mc ready local`), Redis (image `redis:7-alpine`, port 6379, health check via `redis-cli ping`), and `video-worker` (build `Dockerfile.worker`, depends_on with health conditions for redis+db+minio) to `compose.yaml`. Created `storage.config.ts` and `queue.config.ts` with `registerAs` pattern. Extended Joi schema with `STORAGE_*` and `REDIS_*` vars. Created `Dockerfile.worker` with Node.js 22 slim base + `apt-get install -y ffmpeg`. Created `src/types/ffprobe-installer.d.ts` for missing type declaration.

### SI-03.2 — Video Entity, Migration, and Module Scaffold
- **Status:** completed
- **Tests:** 4/4 passing (video.entity.integration-spec.ts)
- **Observations:** Generated migration `1783990000000-CreateVideos.ts` creating `video_status_enum`, `videos` table with FK `ON DELETE RESTRICT`, and three indexes (`idx_videos_channel_id`, `idx_videos_channel_id_created_at`, `idx_videos_status`). Created `StorageModule` (global) with `StorageService.onModuleInit()` auto-creating the MinIO bucket. Created `VideoProcessingModule` (not imported in AppModule — only in `worker.ts`). Registered `BullModule.forRootAsync` in `AppModule` with Redis connection via `queue.config`.

### SI-03.3 — Video Upload Initiation (`POST /videos`)
- **Status:** completed
- **Tests:** 19/19 passing (videos.service.spec.ts, videos.controller.spec.ts, videos.service.integration-spec.ts sections US1/US5)
- **Observations:** `storageKey` pattern `videos/{uuid}/original.{ext}` with extension derived from `mime_type` (e.g., `video/mp4` → `mp4`). Presigned URL validity: 900 s (15 minutes). `ChannelRequiredException` thrown when `channel.user_id !== userId`. US5 (unique URL) covered by integration test confirming distinct UUIDs for same-title videos.

### SI-03.4 — Upload Confirmation and Queue Enqueue (`POST /videos/:id/upload-complete`)
- **Status:** completed
- **Tests:** 12/12 passing (videos.service.spec.ts and videos.service.integration-spec.ts sections confirmUpload)
- **Observations:** Job options `{ jobId: videoId, attempts: 3, backoff: { type: 'fixed', delay: 5000 } }` set at enqueue time (not in queue registration). `jobId: videoId` ensures BullMQ deduplicates concurrent confirm-upload calls. If `queue.add()` throws, status remains `draft` and `InternalServerErrorException` is thrown.

### SI-03.5 — Video Processing Worker (FFmpeg)
- **Status:** completed
- **Tests:** 8/8 passing (video-processing.processor.spec.ts)
- **Observations:** `@Processor(VIDEO_QUEUE_NAME, { lockDuration: 300_000 })` prevents job orphaning during FFmpeg runs on large files. Idempotency: worker checks `status === processing` before acting. `fluent-ffmpeg.screenshots({ timestamps: ['00:00:01'], size: '1280x?' })` used for thumbnail generation. Thumbnail uploaded server-side via `StorageService.putObject`. Error path: rethrow on all attempts; on final attempt (`job.attemptsMade >= attempts - 1`) update `status=ERROR`, `error_cause=message.slice(0,2048)`. Cleanup always runs in `finally`.

### SI-03.6 — Public Video Detail and Streaming
- **Status:** completed
- **Tests:** 28/28 passing (videos.service.spec.ts, test/videos.e2e-spec.ts covering GET /videos/:id and GET /videos/:id/stream)
- **Observations:** Range header passthrough: API forwards `Range` header as-is to `GetObjectCommand`. AWS SDK returns a 416 response (S3 error `InvalidRange`) when the range is beyond the file size — caught and re-thrown as `VideoRangeNotSatisfiableException` (HTTP 416). `Accept-Ranges: bytes` always set. `Content-Range` set only for 206 responses.

### SI-03.7 — Authenticated Download
- **Status:** completed
- **Tests:** 6/6 passing (videos.service.spec.ts downloadVideo section, test/videos.e2e-spec.ts download tests)
- **Observations:** Filename derived from `storageKey` last segment (e.g., `videos/{id}/original.mp4` → `original.mp4`). `Content-Disposition: attachment; filename="{encodeURIComponent(filename)}"` set. Full file served without `Range` header. No public access — JWT required.

### SI-03.8 — Video Management: List, Update, Delete
- **Status:** completed
- **Tests:** 24/24 passing (videos.service.spec.ts, videos.service.integration-spec.ts, test/videos.e2e-spec.ts CRUD sections)
- **Observations:** Pagination: `skip = (page - 1) * limit`, `take = limit`; `has_next = (page * limit) < total`. Delete: BullMQ job cancellation best-effort via `queue.getJob(videoId)?.remove()`. Storage delete tolerates `NoSuchKey`. `deleteVideo` uses a transaction only for the DB record deletion; storage cleanup is best-effort outside the transaction. Non-owner operations return 403 `VIDEO_OWNERSHIP` without leaking the video's existence (ownership check performed after existence check).

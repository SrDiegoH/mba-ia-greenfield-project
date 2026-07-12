# phase-03-videos — Library References

Libraries introduced in Phase 03. Versions verified against `nestjs-project/package.json` via context7 documentation lookup before implementation.

---

## BullMQ — Queue Processing

| Package | Installed Version | Role |
|---------|-------------------|------|
| `@nestjs/bullmq` | `^11.0.0` | NestJS module integration for BullMQ (`BullModule.forRootAsync`, `@InjectQueue`, `@Processor`, `WorkerHost`) |
| `bullmq` | `^5.0.0` | Core BullMQ library — `Queue`, `Worker`, `Job` types |
| `ioredis` | `^5.0.0` | Redis client used internally by BullMQ |

**Decision:** TD-01 — BullMQ + Redis chosen over RabbitMQ and pg-boss for native NestJS integration, TypeScript-native API, and per-job retry/backoff support.

**Key APIs used:**
- `BullModule.forRootAsync({ useFactory: () => ({ connection: { host, port } }) })` — global Redis connection
- `BullModule.registerQueue({ name: VIDEO_QUEUE_NAME })` — queue declaration in `VideoProcessingModule`
- `@InjectQueue(VIDEO_QUEUE_NAME)` — queue injection in `VideosService` for `queue.add()`
- `@Processor(VIDEO_QUEUE_NAME, { lockDuration: 300_000 })` — worker processor class
- `WorkerHost` (abstract base) + `override process(job: Job<T>): Promise<void>` — worker implementation
- Job options: `{ jobId: videoId, attempts: 3, backoff: { type: 'fixed', delay: 5000 } }`
- `queue.getJob(videoId)` + `job.remove()` — job cancellation on video delete

---

## AWS SDK for S3 — Object Storage

| Package | Installed Version | Role |
|---------|-------------------|------|
| `@aws-sdk/client-s3` | `^3.750.0` | S3 commands: `HeadBucketCommand`, `CreateBucketCommand`, `PutObjectCommand`, `GetObjectCommand`, `DeleteObjectCommand` |
| `@aws-sdk/s3-request-presigner` | `^3.750.0` | `getSignedUrl` for generating pre-signed PUT URLs |

**Decision:** TD-02 (upload via pre-signed URL) and TD-03 (streaming proxy). MinIO is used locally with S3-compatible API.

**Key configuration:**
```typescript
new S3Client({
  endpoint: storageConfig.endpoint,   // http://minio:9000 in Docker
  region: 'us-east-1',                // required by SDK; MinIO ignores it
  forcePathStyle: true,               // MinIO requires path-style URLs
  credentials: {
    accessKeyId: storageConfig.accessKey,
    secretAccessKey: storageConfig.secretKey,
  },
})
```

**Key APIs used:**
- `getSignedUrl(client, new PutObjectCommand({ Bucket, Key, ContentType }), { expiresIn: 900 })` — 15-minute upload URL
- `GetObjectCommand({ Bucket, Key, Range? })` — streaming with optional range header passthrough
- `PutObjectCommand({ Bucket, Key, Body, ContentType })` — server-side thumbnail upload by worker
- `DeleteObjectCommand({ Bucket, Key })` — file removal on video delete (tolerates NoSuchKey)
- `HeadBucketCommand` / `CreateBucketCommand` — bucket auto-creation on `StorageService.onModuleInit()`

---

## FFmpeg — Video Processing

| Package | Installed Version | Role |
|---------|-------------------|------|
| `fluent-ffmpeg` | `^2.1.3` | High-level Node.js wrapper for FFmpeg commands (thumbnail generation) |
| `@ffprobe-installer/ffprobe` | `^1.4.1` | Bundled `ffprobe` binary; path exposed via `.path` property |
| `@types/fluent-ffmpeg` | `^2.1.27` (dev) | TypeScript declarations for `fluent-ffmpeg` |

**Decision:** TD-04 — FFmpeg toolchain runs exclusively in the worker container. System `ffmpeg` installed via `apt-get install -y ffmpeg` in `Dockerfile.worker`.

**Key APIs used:**
- `ffmpeg.setFfprobePath(ffprobeInstaller.path)` — set bundled ffprobe path
- `ffmpeg.ffprobe(videoPath, callback)` — extract `format.duration`, `format.bit_rate`, `streams[].width/height/codec_name`
- `ffmpeg(videoPath).screenshots({ timestamps: ['00:00:01'], filename, folder, size: '1280x?' })` — thumbnail at 1s mark

**Runtime path convention:** Worker downloads video from MinIO to `/tmp/{videoId}/original`, generates thumbnail at `/tmp/{videoId}/thumbnail.jpg`, uploads thumbnail to MinIO, then cleans up `/tmp/{videoId}` in a `finally` block.

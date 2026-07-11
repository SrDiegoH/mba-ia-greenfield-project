---
kind: phase
name: phase-03-videos
sources_mtime:
  docs/project-plan.md: "2026-06-27T00:00:00-03:00"
  docs/decisions/technical-decisions-phase-03-videos.md: "2026-06-27T00:00:00-03:00"
  docs/phases/phase-02-auth/phase-02-auth.md: "2026-05-12T13:36:17-03:00"
---

# phase-03-videos — Context

## Scope

**Phase name:** Fase 03 — Upload e Processamento de Vídeos

**Capabilities**

- Object storage para arquivos de vídeo e thumbnails (MinIO/S3)
- Fila de processamento em segundo plano (BullMQ + Redis)
- Worker de processamento de vídeo (FFmpeg + ffprobe, container separado)
- Upload de vídeos de até 10 GB sem travar a API (URL pré-assinada, bytes direto ao storage)
- Pré-cadastro automático do vídeo como rascunho (`draft`) ao iniciar o upload
- Processamento automático após upload: extração de duração e metadados técnicos
- Geração automática de thumbnail a partir de frame do vídeo
- URL única por vídeo sem conflito (UUID v4 como PK)
- Reprodução via streaming com range requests (sem download completo)
- Download autenticado do vídeo completo
- Ciclo de status do vídeo: `draft` → `processing` → `ready` | `error`
- Listagem paginada de vídeos do canal (autenticada, somente owner)
- Atualização de título de vídeo (somente owner)
- Exclusão de vídeo com remoção de arquivos do storage e cancelamento de job pendente

**Out of scope:** Interface de vídeo no frontend (next-frontend), controle de visibilidade (vídeos privados/não listados), campo de descrição, sistema de comentários, likes, inscrições — todos em fases posteriores.

**Deliverables:** upload de até 10 GB funcional, processamento automático do vídeo, streaming funcionando e URLs únicas geradas. Infrastructure (MinIO, Redis, video-worker) subindo via `docker compose up`.

**Affected subprojects:** `nestjs-project/`

**Deferred subprojects:** `next-frontend/` — interface de vídeo não faz parte desta fase.

**Sequencing notes:** Depends on Fase 02 — Cadastro, Login e Gerenciamento de Conta. Each video belongs to a Channel (created in Phase 02); the JWT guard is inherited from Phase 02.

**Neighbors (for boundary detection only):** Fase 02 — Auth (prior), Fase 04 — Gerenciamento de Canal e Vídeos (next).

## Decisions Index

| Ref | Source | Scope | Topic | Status | Decision | Libraries |
|-----|--------|-------|-------|--------|----------|-----------|
| phase-03-videos/TD-01 | technical-decisions-phase-03-videos.md | Backend + Infra | Queue Technology | decided | A (BullMQ + Redis) | @nestjs/bullmq@^11.0.0, bullmq@^5.0.0, ioredis@^5.0.0 |
| phase-03-videos/TD-02 | technical-decisions-phase-03-videos.md | Backend | Upload Strategy | decided | A (Pre-signed PUT URL, 15 min) | @aws-sdk/client-s3@^3.750.0, @aws-sdk/s3-request-presigner@^3.750.0 |
| phase-03-videos/TD-03 | technical-decisions-phase-03-videos.md | Backend | Streaming Strategy | decided | A (API proxy with range requests) | — (uses same AWS SDK) |
| phase-03-videos/TD-04 | technical-decisions-phase-03-videos.md | Backend + Infra | Worker Architecture | decided | A (NestJS standalone, Dockerfile.worker) | fluent-ffmpeg@^2.1.3, @ffprobe-installer/ffprobe@^1.4.1 |
| phase-03-videos/TD-05 | technical-decisions-phase-03-videos.md | Backend | Unique URL Identifier | decided | A (UUID v4 as PK) | — (TypeORM built-in) |
| phase-03-videos/TD-06 | technical-decisions-phase-03-videos.md | Backend + Worker | Video Status Lifecycle | decided | A (4-state: draft → processing → ready \| error) | — |

_Source files:_

- `docs/decisions/technical-decisions-phase-03-videos.md`

## Capability Coverage

| Capability | Covered by |
|------------|------------|
| Object storage (MinIO/S3) | phase-03-videos/TD-02 (upload strategy), phase-03-videos/TD-03 (streaming) |
| Fila de processamento (BullMQ + Redis) | phase-03-videos/TD-01 |
| Worker de vídeo (FFmpeg, container separado) | phase-03-videos/TD-04 |
| Upload sem bytes pela API (até 10 GB) | phase-03-videos/TD-02 |
| Pré-cadastro como rascunho | phase-03-videos/TD-06 |
| Processamento automático (metadados + thumbnail) | phase-03-videos/TD-04 |
| URL única por vídeo | phase-03-videos/TD-05 |
| Streaming com range requests | phase-03-videos/TD-03 |
| Download autenticado | phase-03-videos/TD-03 (same proxy mechanism, different access rule) |
| Ciclo de status (draft → processing → ready \| error) | phase-03-videos/TD-06 |
| Listagem, edição e exclusão de vídeos | _Inherited from Phase 02 auth patterns (JWT guard, ownership checks); no separate TD._ |

## Decisions Detail

### phase-03-videos/TD-01

**Recommendation:** BullMQ + Redis — Official NestJS integration, TypeScript-native, per-job retry/backoff (covers FR-008 natively), lightweight Redis in Docker. Lock timeout of 300 000 ms prevents job orphaning during long FFmpeg runs.

**Libraries:** `@nestjs/bullmq@^11.0.0`, `bullmq@^5.0.0`, `ioredis@^5.0.0`

### phase-03-videos/TD-02

**Recommendation:** Pre-signed PUT URL — Zero bytes through Node.js API; MinIO supports single-part PUT up to 5 TiB (no 5 GB limitation unlike AWS S3). 15-minute URL validity. Storage key convention: `videos/{uuid}/original.{ext}`.

**Libraries:** `@aws-sdk/client-s3@^3.750.0`, `@aws-sdk/s3-request-presigner@^3.750.0`

### phase-03-videos/TD-03

**Recommendation:** API proxy with range requests — Credentials stay server-side; access control enforced at controller layer; works within Docker network (MinIO not publicly accessible from browser). Streaming public (`@Public()`), download authenticated (JWT required).

**Libraries:** — (reuses `@aws-sdk/client-s3` from TD-02)

### phase-03-videos/TD-04

**Recommendation:** NestJS standalone application in `src/worker.ts`, built via `Dockerfile.worker`, registered as `video-worker` in `compose.yaml`. Shares all project code (entities, config, services). FFmpeg installed only in the worker image. Worker concurrency: 2 videos in parallel.

**Libraries:** `fluent-ffmpeg@^2.1.3`, `@ffprobe-installer/ffprobe@^1.4.1`

### phase-03-videos/TD-05

**Recommendation:** UUID v4 as the primary key — zero additional code, immutable, consistent with project conventions (User, Channel, tokens all use UUID). URL pattern: `GET /videos/{uuid}`.

**Libraries:** — (TypeORM `@PrimaryGeneratedColumn('uuid')`)

### phase-03-videos/TD-06

**Recommendation:** Four-state machine (`draft`, `processing`, `ready`, `error`). Idempotency enforced by worker (checks `status === processing` before acting). Retry: 3 attempts, 5 s fixed backoff. `error_cause` truncated to 2048 chars on final failure.

**Libraries:** —

## Inherited Decisions Detail

### phase-02-auth/TD-02

**Recommendation:** Custom guards with `@nestjs/jwt` — JWT guard registered globally as `APP_GUARD`. `@Public()` decorator opts out per endpoint. Phase 03 inherits this pattern: streaming and video detail are `@Public()`; all write operations and download require a valid JWT.

**Libraries:** `@nestjs/jwt@^11.0.0`

### phase-02-auth/TD-07

**Recommendation:** Custom Domain Exception Filter — All Phase 03 domain exceptions (`VideoNotFoundException`, `VideoNotReadyException`, `VideoOwnershipException`, `ChannelRequiredException`, `VideoRangeNotSatisfiableException`) follow the same `{ statusCode, error, message }` envelope. No new filter needed.

**Libraries:** —

### phase-01-configuracao-base/TD-01

**Recommendation:** `@nestjs/config` with `registerAs` — Phase 03 adds `storage.config.ts` and `queue.config.ts` following the same namespaced factory pattern.

**Libraries:** `@nestjs/config@^4.x`

## Inherited Conventions

- Config: `registerAs('storage', ...)` in `src/config/storage.config.ts`, `registerAs('queue', ...)` in `src/config/queue.config.ts`, injected via `ConfigType<typeof xxxConfig>`. _(from phase 01)_
- Env validation: new variables (`STORAGE_*`, `REDIS_*`) added to Joi schema in `src/config/env.validation.ts`. _(from phase 01)_
- Domain exceptions: new `VideoNotFoundException`, `VideoNotReadyException`, `VideoOwnershipException`, `ChannelRequiredException`, `VideoRangeNotSatisfiableException` extend `DomainException`. _(from phase 02)_
- TypeORM: `@PrimaryGeneratedColumn('uuid')`, `@CreateDateColumn()`, `@UpdateDateColumn()`, migration required for any schema change. _(from phase 01)_
- Docker Compose: service names as hostnames (`minio`, `redis`, `db`) — never `localhost`. _(global rule)_

## Testing Requirements

Refer to the `testing-guide-nestjs-project` Skill for layer requirements per artifact type. Phase 03 introduces:
- Unit tests (`*.spec.ts`): `VideosService`, `VideosController`, `VideoProcessingProcessor` with all collaborators mocked
- Integration tests (`*.integration-spec.ts`): `Video` entity constraints, `VideosService` with real DB, storage service with real MinIO
- E2E tests (`*.e2e-spec.ts`): full HTTP cycle for all video endpoints via Supertest
- Worker processing is covered at the E2E level (real BullMQ + Redis + MinIO); no integration spec for the processor itself


---

# Backend — End-to-End Plan (Studio.Anime)

Short summary: build a robust Node/TypeScript backend that exposes REST/JSON APIs (or server actions when using Next.js), stores metadata in Postgres via Prisma, uses Clerk for auth, stores assets in S3, runs a BullMQ/Redis worker to transcode video to HLS with FFmpeg, signs HLS URLs for private content, and provides monitoring + CI.

---

## Tech choices (recommended)

* Language: **TypeScript**
* Framework: **Next.js (App Router) API routes / server actions** OR **Fastify/Express (ts-node / build)** for standalone API. (I'll provide examples for Next.js + separate worker.)
* ORM: **Prisma** (Postgres)
* Auth: **Clerk** (JWT + server session verification)
* Storage: **AWS S3** (or S3-compatible MinIO for dev)
* Queue: **BullMQ** + Redis (or Bee-Queue)
* Transcoding: **FFmpeg** in worker container (or AWS MediaConvert for managed)
* CI: **GitHub Actions**
* Local dev infra: `docker-compose` (postgres, redis, minio, worker)

---

## Environment variables (.env.example)

```
# Postgres
DATABASE_URL=postgresql://user:pass@postgres:5432/studioanime?schema=public

# Clerk
CLERK_API_KEY=clerk_api_key
CLERK_FRONTEND_API=clerk_frontend_api
CLERK_JWT_KEY=clerk_jwt_key

# S3
S3_ENDPOINT=http://minio:9000
S3_BUCKET=studio-anime
S3_REGION=us-east-1
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_FORCE_PATH_STYLE=true

# Redis
REDIS_URL=redis://redis:6379

# Worker
FFMPEG_PATH=/usr/bin/ffmpeg
TEMP_DIR=/tmp

# Other
NODE_ENV=development
PORT=3000
JWT_SECRET=supersecret
SENTRY_DSN=
CDN_BASE_URL=https://cdn.example.com
```

---

## Prisma schema (full, ready-to-use)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(cuid())
  clerkId   String    @unique
  email     String
  role      String    @default("viewer")
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  favorites Favorite[]
  playbacks PlaybackPosition[]
}

model Title {
  id        String    @id @default(cuid())
  slug      String    @unique
  name      String
  type      String
  year      Int?
  synopsis  String?
  posterUrl String?
  bannerUrl String?
  genres    String[]  @default([])
  languages String[]  @default([])
  platforms String[]  @default([])
  ratingAvg Float?    
  seasons   Season[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Season {
  id        String    @id @default(cuid())
  title     Title     @relation(fields: [titleId], references: [id])
  titleId   String
  number    Int
  episodes  Episode[]
  createdAt DateTime  @default(now())
}

model Episode {
  id             String    @id @default(cuid())
  season         Season    @relation(fields: [seasonId], references: [id])
  seasonId       String
  titleId        String
  number         Int
  name           String
  synopsis       String?
  durationSec    Int?
  hlsMasterUrl   String?   // CDN URL or signed URL
  thumbnails     String[]  @default([])
  subtitles      Json?
  transcodeState String    @default("pending") // pending|queued|processing|ready|failed
  transcodeLog   String?
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
}

model Favorite {
  id        String   @id @default(cuid())
  user      User     @relation(fields: [userId], references: [id])
  userId    String
  title     Title    @relation(fields: [titleId], references: [id])
  titleId   String
  createdAt DateTime @default(now())
}

model PlaybackPosition {
  id         String   @id @default(cuid())
  user       User     @relation(fields: [userId], references: [id])
  userId     String
  episodeId  String
  positionSec Int
  updatedAt  DateTime @updatedAt
}

model Job {
  id        String   @id @default(cuid())
  type      String
  payload   Json
  status    String   @default("queued")
  logs      String?
  attempts  Int      @default(0)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

Run:

```bash
pnpm prisma generate
pnpm prisma migrate dev --name init
```

---

## API design (main endpoints)

Use `pages/api` or `app/api`:

**Public**

* `GET /api/titles?genre=&type=&page=&q=` → list
* `GET /api/titles/:slug` → details with seasons & episodes
* `GET /api/episodes/:id/hls` → returns signed HLS URL (if private) or direct URL

**Auth (user)**

* `POST /api/favorites` `{ titleId }` → toggle add/remove (Clerk guard)
* `GET /api/users/:id/watchlist`
* `POST /api/playback` `{ episodeId, positionSec }` → save resume position

**Admin (requireRole: admin/editor)**

* `POST /api/admin/titles` → create title
* `PUT /api/admin/titles/:id`
* `DELETE /api/admin/titles/:id`
* `POST /api/admin/presign` `{ key, contentType }` → returns presigned URL
* `POST /api/admin/episodes` `{ seasonId, number, name, sourceKey }` → creates episode, enqueues transcode job
* `GET /api/admin/jobs` → jobs list
* `POST /api/admin/jobs/:id/retry`

---

## Authentication integration (Clerk)

* Install `@clerk/nextjs` for Next.js or `@clerk/clerk-sdk-node` for server.
* Protect admin routes with middleware (see earlier). On server endpoints, verify Clerk token:

```ts
import { verifyToken } from "@clerk/clerk-sdk-node"; // pseudocode

export async function requireUser(req) {
  const authHeader = req.headers.get('authorization') || '';
  const token = authHeader.replace('Bearer ', '');
  const session = await verifyToken(token);
  if (!session) throw new Error('UNAUTH');
  return session;
}
```

* Use Clerk metadata `role` OR keep roles in local DB mapped by `clerkId`. Use `requireRole(clerkId, ['admin'])` helper.

---

## S3 presigned upload flow

1. Admin requests presign: `POST /api/admin/presign` `{ key: 'posters/slug/poster.jpg', contentType: 'image/jpeg' }`.
2. Server returns `{ url, key, expiresIn }`.
3. Client uploads directly to S3 via PUT or using multipart with presigned parts.
4. After upload, client calls `POST /api/admin/episodes` with `sourceKey` (S3 key), server enqueues transcode.

Server presign (AWS SDK v3):

```ts
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: process.env.S3_REGION, endpoint: process.env.S3_ENDPOINT, forcePathStyle: true, credentials: { accessKeyId: process.env.S3_ACCESS_KEY, secretAccessKey: process.env.S3_SECRET_KEY } });

export async function createPresign(key, contentType) {
  const cmd = new PutObjectCommand({ Bucket: process.env.S3_BUCKET, Key: key, ContentType: contentType, ACL: 'private' });
  const url = await getSignedUrl(s3, cmd, { expiresIn: 3600 });
  return { url, key };
}
```

---

## Transcoding workflow (FFmpeg + worker)

**Overview**:

* After upload, server enqueues a job to BullMQ: `{ episodeId, sourceKey }`.
* Worker downloads source (or streams), runs FFmpeg to produce HLS (master + variants), uploads HLS folder to S3 (`hls/{titleSlug}/season-{n}/ep-{n}/master.m3u8`), updates DB.

**FFmpeg template**:

```bash
ffmpeg -hide_banner -y -i input.mp4 \
 -map 0:v -map 0:a \
 -c:a aac -b:a 128k \
 -vf "scale=-2:1080" -c:v:libx264 -b:v:5000k -preset medium -g 48 -keyint_min 48 -sc_threshold 0 \
 -hls_time 6 -hls_playlist_type vod -hls_segment_filename "1080p/seg_%03d.ts" 1080p/prog_index.m3u8
# Repeat for 720p and 480p, then create a master m3u8 referencing variants (or use -var_stream_map)
```

**Recommended variant command (single command to produce multiple variants)** — same as earlier multi-variant command using `-var_stream_map` which outputs `v0`, `v1`, etc and a `master.m3u8`.

**Worker (simplified)**

```ts
import { spawn } from "child_process";
import fs from "fs";
import path from "path";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import prisma from './prisma';

async function processJob(job) {
  const { episodeId, sourceKey } = job.data;
  await prisma.episode.update({ where: { id: episodeId }, data: { transcodeState: 'processing' }});
  const tmpDir = `/tmp/${episodeId}`;
  fs.mkdirSync(tmpDir, { recursive: true });
  const inputPath = path.join(tmpDir, 'input.mp4');
  // download from S3 -> inputPath (use getObject and stream)
  // run ffmpeg spawn with args (as above) writing into tmpDir/hls/
  // after finish, upload each file from tmpDir/hls -> S3 under hls/.../
  // update DB with hlsMasterUrl = `https://cdn.example.com/hls/.../master.m3u8`
  // update transcodeState = 'ready'
}
```

**Notes**:

* Worker must stream upload segments concurrently to S3 to avoid storing large files locally.
* For production, consider using managed transcoding services for scale (AWS MediaConvert, Elastic Transcoder).

---

## Signed HLS URLs & streaming

* If content private, generate signed CDN URL (CloudFront signed URL) that expires (e.g., 1 hour) and return to client when they request `/api/episodes/:id/hls`.
* For public dev, return `s3://` or direct CDN URL.

Example endpoint:

```ts
GET /api/episodes/:id/hls
- Validate user/role if necessary.
- If episode.hlsMasterUrl is private path, generate signed URL via CloudFront (private key + policy) or S3 pre-signed GET.
- Return { url: signedUrl }
```

Player should fetch that URL to initialize HLS.js (or native HLS on iOS).

---

## Worker scaling & error handling

* Use BullMQ with concurrency control and job retries (3 attempts, exponential backoff).
* Save job logs to `Job` model and `Episode.transcodeLog` for debugging.
* If transcode fails, set `transcodeState = 'failed'` and surface reason in admin UI.

---

## Docker Compose (local dev)

`docker-compose.yml` (key services)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: studioanime
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7
    ports: ["6379:6379"]

  minio:
    image: minio/minio
    command: server /data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000"]
    volumes: ["minio:/data"]

  web:
    build: .
    command: pnpm dev
    environment:
      DATABASE_URL: ...
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
    ports: ["3000:3000"]
    depends_on: ["postgres","redis","minio"]

  worker:
    build: ./worker
    command: pnpm start
    environment: ...
    depends_on: ["redis","minio","postgres"]

volumes:
  pgdata:
  minio:
```

Start local:

```bash
docker compose up --build
pnpm prisma migrate dev
pnpm seed
```

---

## Security & Best practices

* **Never** commit secrets. Use `.env` and `.env.example`.
* Validate uploads: MIME type, extension, max file size. Reject unrecognized formats.
* Rate-limit endpoints (e.g., presign, playback) to prevent abuse.
* Authenticate and authorize every admin endpoint.
* Use HTTPS and HSTS in production.
* Scan uploads (virus scan) if feasible.
* CSP headers for player pages to avoid mixed content.

---

## Tests

* Unit tests (Vitest/Jest) for utilities and API handlers (mock S3 & DB).
* Integration tests with a test Postgres instance (docker) for Prisma models.
* E2E (Playwright): admin flow + upload + job simulation + playback.
* Example commands:

```bash
pnpm test        # unit
pnpm test:playwright  # run E2E
```

CI pipeline (GitHub Actions):

* install, lint, unit tests, build, run Playwright in headless, run `prisma migrate deploy`.

---

## Observability

* Add Sentry to API & worker (server-side DSN) for crashes.
* Structured logs (json) with timestamps and job ids.
* Metrics: queue length, worker successes/failures (prometheus exporter or simple metrics pushed to Datadog).

---

## Deployment checklist

1. Provision Postgres (Neon/Supabase/RDS).
2. Provision S3 bucket + CloudFront (for HLS CDN). Configure CORS for player origins.
3. Provision Redis (Upstash/ElastiCache).
4. Deploy backend (Render/ECS/Cloud Run), worker on separate service, web on Vercel if using Next.js.
5. Configure environment variables securely.
6. Run `prisma migrate deploy`.
7. Seed minimal dataset and test admin transcode with small mp4.
8. Configure Sentry, backups, and monitoring.

---

## Useful commands & scripts

* Install deps:

  ```bash
  pnpm install
  ```
* Dev (web):

  ```bash
  pnpm dev
  ```
* Start worker:

  ```bash
  pnpm --filter worker dev
  ```
* Migrate & seed:

  ```bash
  pnpm prisma:migrate
  pnpm prisma:generate
  pnpm seed
  ```
* Run tests:

  ```bash
  pnpm test
  pnpm test:e2e
  ```

---

## Acceptance criteria (backend)

* All API endpoints respond as spec’d and respect RBAC.
* Presign upload + direct S3 upload flow works (tested with MinIO).
* Creating episode enqueues transcode job; worker processes job (or dev fallback) and updates `hlsMasterUrl`.
* Signed HLS URL endpoint returns working URL usable by HLS.js.
* Playback resume and favorites persist per user.
* CI runs tests and prevents merging broken code.
* Secrets and keys not committed; `.env.example` present.

---


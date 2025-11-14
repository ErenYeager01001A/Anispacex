
# 1. Goals & responsibilities

Admin dashboard must let admins and editors manage the entire platform:

* CRUD Titles / Seasons / Episodes
* Upload posters, banners, thumbnails, and video sources
* Start/monitor transcode jobs (HLS generation)
* Review & manage users, roles, and activity logs
* Moderate comments & ratings
* View analytics: uploads, plays, top titles
* Manual tasks: import from TMDB, bulk CSV import/export
* Secure: Clerk-based RBAC, audit logs, signed HLS URLs

---

# 2. High-level architecture

* **Frontend Admin**: Next.js (app router) within same repo under `/apps/admin` or `/app/admin` (protected routes). TailwindCSS + TypeScript.
* **APIs**: Next.js server actions / API routes or a separate Node + Express service (`/apps/api`) implementing admin endpoints. Use Clerk middleware for auth.
* **Worker**: Background worker (Node.js + BullMQ / Redis) to perform FFmpeg transcodes and upload HLS outputs to S3. Runs in a separate container/service.
* **Storage**: S3-compatible storage (AWS S3). Use presigned URLs for direct uploads of large files.
* **DB**: PostgreSQL via Prisma.
* **Queue**: Redis + BullMQ for job queue.
* **CI/CD**: GitHub Actions; deploy admin frontend to Vercel (or same NextJS deployment). Worker & API to Render/AWS ECS.

---

# 3. Admin UI: pages & components

Pages (Next.js routes under `/admin`):

1. **Dashboard (/admin)**

   * KPIs: total users, recent uploads, active transcodes, server health.
   * Quick actions: Create Title, Import TMDB, Upload video.

2. **Titles List (/admin/titles)**

   * Table with search, sort, pagination, bulk actions (publish/unpublish, delete).
   * Row actions: Edit, Manage Seasons, Delete, View on site.

3. **Create / Edit Title (/admin/titles/new, /admin/titles/:id/edit)**

   * Form fields: Title, slug, type, year, synopsis, genres (multi-select), languages, platforms, poster/banner upload, tags, rating override.
   * Auto-generate slug; preview og image; validation.

4. **Manage Seasons & Episodes (/admin/titles/:id/seasons)**

   * Add Season, Add Episode modal: episode number, name, synopsis, duration, upload video (direct to S3 via presigned), upload subtitles (.vtt), optional remote HLS link.
   * Episode list with status: `uploaded`, `queued`, `transcoding`, `ready`, `failed`. Show logs.

5. **Uploader Modal**

   * Drag & drop, chunked uploads for large files, show presigned progress. After upload finishes, create episode record and push transcode job.

6. **Transcode Jobs (/admin/jobs)**

   * Show queue: job id, type, status, logs, start time, duration. Retry / cancel options.

7. **Users (/admin/users)**

   * List Clerk-linked users, roles, actions: change role, disable, impersonate (dev only), view favorites.

8. **Comments & Moderation (/admin/moderation)**

   * List flagged comments, approve/delete.

9. **Imports & Data Tools (/admin/import)**

   * TMDB import UI, CSV import/export. Map fields before import.

10. **Logs & Analytics (/admin/analytics)**

    * Play counts, top titles, storage usage. Charts (recharts) and CSV export.

UI components:

* Data table, modals, form inputs, file uploader with progress, badge/chips, job log viewer, activity feed.

---

# 4. RBAC & Clerk integration (exact approach)

**Roles**: `admin`, `editor`, `moderator`, `viewer`.
Store role in Clerk's user metadata or in local `User` table mapped to `clerkId`.

## Server-side middleware (Next.js) — sample

```ts
// middleware.ts (Next.js edge or server middleware)
import { auth } from "@clerk/nextjs";
import { NextResponse } from "next/server";

export async function middleware(req: Request) {
  const { userId, sessionId, getToken } = auth();
  const url = new URL(req.url);
  if (url.pathname.startsWith('/admin')) {
    if (!userId) return NextResponse.redirect('/login');
    // Optionally validate role from your DB
    const token = await getToken({ template: "backend" });
    // Call your API to check role or decode custom claims
    // If not admin/editor -> redirect /403
  }
  return NextResponse.next();
}
```

## Server-side role check helper (API route)

```ts
// utils/auth.ts
import { clerkClient } from "@clerk/clerk-sdk-node";

export async function requireRole(clerkId: string | null, roles: string[]) {
  if (!clerkId) throw new Error("UNAUTH");
  const user = await clerkClient.users.getUser(clerkId);
  const role = user.privateMetadata?.role || user.publicMetadata?.role;
  if (!roles.includes(role)) throw new Error("FORBIDDEN");
  return { user, role };
}
```

When creating admin user for first time, set their Clerk metadata `role = "admin"`.

---

# 5. Database additions & Prisma models

Add `Job`, `Asset`, `Comment`, and extend `Episode` to track transcode_state.

```prisma
model User {
  id        String   @id @default(cuid())
  clerkId   String   @unique
  email     String
  role      String   @default("viewer")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  favorites Favorite[]
}

model Title {
  id        String   @id @default(cuid())
  slug      String   @unique
  name      String
  type      String
  year      Int?
  synopsis  String?
  posterUrl String?
  bannerUrl String?
  genres    String[]
  seasons   Season[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Season {
  id        String   @id @default(cuid())
  title     Title    @relation(fields: [titleId], references: [id])
  titleId   String
  number    Int
  episodes  Episode[]
}

model Episode {
  id           String   @id @default(cuid())
  season       Season   @relation(fields: [seasonId], references: [id])
  seasonId     String
  number       Int
  name         String
  synopsis     String?
  durationSec  Int?
  hlsMasterUrl String?
  thumbnails   String[]  // array of URLs
  subtitles    Json?
  transcodeState String  @default("pending") // pending|queued|processing|ready|failed
  transcodeLog  String?
  createdAt    DateTime @default(now())
}

model Asset {
  id        String   @id @default(cuid())
  path      String
  type      String
  meta      Json?
  createdAt DateTime @default(now())
}

model Job {
  id        String   @id @default(cuid())
  type      String
  payload   Json
  status    String   @default("queued") // queued, active, completed, failed
  logs      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

Run `prisma migrate dev` then seed.

---

# 6. File upload & presigned S3 flow (detailed)

**Steps**:

1. Admin client requests a presigned URL for a given target path and content type: `POST /api/admin/presign` with `{ path, contentType, size }`.
2. Server signs URL with S3 SDK (`getSignedUrl` for `PutObjectCommand`) and returns `{ url, key, expiresIn }`.
3. Admin client uploads directly to S3 with `PUT` to `url`. Use chunked upload for >100MB (multipart).
4. After upload completes, admin calls `POST /api/admin/episodes` with metadata including `sourceKey` (S3 path).
5. Server enqueues job: `BullMQ.add('transcode', { titleId, seasonId, episodeId, sourceKey })`.

**Server presign example (AWS SDK v3)**:

```ts
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: process.env.AWS_REGION });

export async function createPresign(key: string, contentType: string) {
  const cmd = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    ContentType: contentType,
    ACL: 'private',
  });
  const url = await getSignedUrl(s3, cmd, { expiresIn: 3600 });
  return url;
}
```

---

# 7. Transcode worker & FFmpeg commands (exact templates)

**Worker tech**: Node.js script, BullMQ consumer connected to Redis. When job dequeued, worker downloads source from S3 (or streams), runs FFmpeg to generate HLS variants, uploads results to S3, then updates DB `Episode.hlsMasterUrl` and job logs.

**FFmpeg master command** (produces 3 variants: 1080p, 720p, 480p):

```bash
ffmpeg -hide_banner -y -i input.mp4 \
 -map 0:v -map 0:a -map 0:a -map 0:a \
 -s:v:0 1920x1080 -c:v:0 libx264 -b:v:0 5000k -profile:v:0 high -preset veryfast -g 48 -keyint_min 48 \
 -s:v:1 1280x720  -c:v:1 libx264 -b:v:1 3000k -profile:v:1 main -preset veryfast -g 48 -keyint_min 48 \
 -s:v:2 854x480   -c:v:2 libx264 -b:v:2 1200k -profile:v:2 baseline -preset veryfast -g 48 -keyint_min 48 \
 -c:a aac -b:a 128k \
 -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" \
 -master_pl_name master.m3u8 \
 -f hls \
 -hls_time 6 \
 -hls_playlist_type vod \
 -hls_segment_filename "v%v/seg_%03d.ts" \
 v%v/prog_index.m3u8
```

**Worker snippet (Node + child_process)**:

```ts
import { spawn } from "child_process";
import fs from "fs";
import { S3Client, GetObjectCommand, PutObjectCommand } from "@aws-sdk/client-s3";

async function transcode(jobPayload) {
  // download source to /tmp/input.mp4
  // run ffmpeg command (spawn)
  // stream or upload generated files to S3 (walk folder)
  // update DB
}
```

**Important**: ensure worker has sufficient CPU & disk; recommend transcoding on dedicated instance or use a serverless transcoder like AWS Elastic Transcoder / MediaConvert in production.

---

# 8. Job queue (BullMQ) example

```ts
// queue.ts
import { Queue } from 'bullmq';
export const transcodeQueue = new Queue('transcode', { connection: { host: process.env.REDIS_HOST } });

// worker.ts
import { Worker } from 'bullmq';
new Worker('transcode', async job => {
  // call transcode(job.data)
}, { connection: { host: process.env.REDIS_HOST }});
```

Add retries, concurrency limits, exponential backoff.

---

# 9. Admin APIs (summary)

* `GET /api/admin/titles` — list, filters (search, genre, status)
* `GET /api/admin/titles/:id` — details
* `POST /api/admin/titles` — create
* `PUT /api/admin/titles/:id` — edit
* `DELETE /api/admin/titles/:id` — delete
* `POST /api/admin/presign` — request upload URL
* `POST /api/admin/episodes` — create episode (with `sourceKey`)
* `GET /api/admin/jobs` — list jobs
* `POST /api/admin/jobs/:id/retry` — retry
* `GET /api/admin/users` — list users
* `PUT /api/admin/users/:id/role` — change role
* `POST /api/admin/import/tmdb` — import metadata
* `POST /api/admin/moderation/comments/:id/action` — approve/delete

All admin endpoints must call `requireRole(clerkId, ['admin','editor'])`.

---

# 10. Notifications & realtime updates

* Use WebSockets (Redis pub/sub or Pusher) or server-sent events to stream job progress & logs to admin UI.
* Example: worker publishes progress to Redis channel `transcode:progress:{jobId}`; UI subscribes to receive percent/log lines.

---

# 11. Audit logs & activity

* Every admin action (create/edit/delete/title/episode/upload/role change) writes an `ActivityLog` entry with `userId`, `action`, `meta`, `ip`, `timestamp`. Store in DB `ActivityLog` table and show in admin UI with filters.

---

# 12. Moderation & safety

* For comments: provide flags, content-length limits, profanity filter (3rd party lib or simple wordlist), and an approval queue for guest submissions.
* Rate-limit upload endpoints and file sizes.
* Scan uploaded files with antivirus (optional), or validate MIME types.

---

# 13. Tests & QA

Automated tests:

* Unit tests: forms, API handlers (mock S3 & DB), job enqueuing. (Vitest/Jest)
* Integration: Prisma & DB tests using test DB.
* E2E: Playwright tests to simulate admin flows:

  * login as admin (Clerk test user), create title, upload small mp4 via presigned URL, enqueue job, simulate worker run (use dev HLS), confirm episode shows as `ready` and playable.
  * change a user role and confirm protected route access.

Example Playwright test outline:

```ts
test('admin can create title and upload episode', async ({ page }) => {
  await page.goto('/admin');
  await adminLogin(page); // helper for test user
  await page.click('text=Create Title');
  await page.fill('#title', 'Test Anime');
  // upload via presign flow - use stub
  await page.click('text=Save');
  // check item appears in list
});
```

---

# 14. Monitoring, logging & alerts

* Integrate Sentry for worker & API errors.
* Logs: structured JSON logs to stdout (ELK stack or Papertrail).
* Alerts: set up health checks & alerts for worker failures and queue backlogs (CloudWatch, Datadog, or simple cron).

---

# 15. Deployment & infra notes

* **Dev**: docker-compose with services: `postgres`, `redis`, `minio`, `admin-next`, `api`, `worker`.
* **Production**:

  * Frontend (Next.js) → Vercel or Cloud Run (SSR on edge if needed).
  * API & Worker → Render / ECS Fargate / Heroku (worker as separate dyno).
  * Postgres → Managed DB (Neon, Supabase, RDS).
  * S3 → AWS S3 + CloudFront for HLS + signed URLs.
  * Redis → Managed Redis (Upstash/ElastiCache).

Provide GitHub Actions to deploy frontend to Vercel and backend to Render/ECS.

---

# 16. Acceptance criteria (Admin)

* Admin UI accessible only to Clerk users with `admin` or `editor` role.
* Admin can create Title → add Season → add Episode with source upload → see transcode job queued.
* Worker processes job (or dev fallback) and updates `Episode.hlsMasterUrl`.
* Episode becomes playable in frontend via HLS master URL (or preview mp4).
* Activity logs exist for every admin action.
* Tests (unit + 2 Playwright tests for create+play) pass in CI.
* `.env.example` includes keys for Clerk, S3, Redis, POSTGRES.

---

# 17. Helpful code snippets (quick copy)

**Enqueue job endpoint**:

```ts
// /api/admin/episodes (simplified)
import { transcodeQueue } from '../../lib/queue';
import prisma from '../../lib/prisma';

export default async function handler(req, res) {
  const { titleId, seasonId, episodeNumber, sourceKey } = req.body;
  // create episode
  const ep = await prisma.episode.create({ data: { seasonId, number: episodeNumber, transcodeState: 'queued' }});
  await transcodeQueue.add('transcode', { episodeId: ep.id, sourceKey });
  res.json({ ok: true, episode: ep });
}
```

**Update episode after transcode**:

```ts
await prisma.episode.update({
  where: { id: episodeId },
  data: { transcodeState: 'ready', hlsMasterUrl: uploadedMasterUrl, transcodeLog: logs }
});
```

---

# 18. Checklist for you (what to implement first)

1. Add RBAC: Clerk metadata & requireRole helper.
2. Prisma schema + migrations for `Episode.transcodeState` and `Job`.
3. Presign endpoint + client uploader UI.
4. Enqueue transcode job after upload.
5. Local worker that uses simple FFmpeg fallback (or just copy prebuilt HLS) for dev.
6. Admin pages: Titles list → Create Title → Manage Episodes.
7. WebSocket/Redis pubsub for job progress UI.
8. Tests & CI.

---



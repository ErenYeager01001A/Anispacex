# Full repository — complete files & folders structure (end-to-end)


```
studio-anime/                      # repo root
├─ .github/
│  ├─ workflows/
│  │  ├─ ci.yml                    # CI: lint, unit tests, build
│  │  ├─ e2e.yml                   # Playwright E2E workflow
│  │  └─ deploy.yml                # Deploy pipeline (Vercel + Render/ECS)
│  └─ PULL_REQUEST_TEMPLATE.md
├─ .vscode/
│  └─ settings.json
├─ .dockerignore
├─ .gitignore
├─ README.md                       # repo overview + quickstart
├─ CODE_OF_CONDUCT.md
├─ CONTRIBUTING.md
├─ LICENSE
├─ .env.example                     # environment variable template
├─ package.json                     # workspace root (pnpm/workspaces or yarn)
├─ pnpm-workspace.yaml              # workspace config (if pnpm)
├─ tsconfig.base.json
├─ scripts/
│  ├─ dev.sh                        # helper to run dev (docker-compose + installs)
│  ├─ build-all.sh
│  ├─ seed.sh                       # runs seed script
│  └─ migrate.sh                    # run prisma migrations
├─ infra/
│  ├─ terraform/
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  └─ outputs.tf
│  ├─ cloudformation/
│  │  └─ cloudfront-s3.yml
│  └─ k8s/
│     ├─ api-deployment.yaml
│     └─ worker-deployment.yaml
├─ docker-compose.yml
├─ .github/                           # (already above)
│
├─ apps/
│  ├─ frontend/                       # Next.js app (public web)
│  │  ├─ package.json
│  │  ├─ next.config.js
│  │  ├─ tsconfig.json
│  │  ├─ postcss.config.js
│  │  ├─ tailwind.config.js
│  │  ├─ public/
│  │  │  ├─ posters/
│  │  │  │  ├─ onepunch-banner.jpg
│  │  │  │  ├─ violet.jpg
│  │  │  │  └─ umamusume-poster.jpg
│  │  │  ├─ banners/
│  │  │  ├─ player/
│  │  │  │  └─ poster-default.jpg
│  │  │  └─ seed/
│  │  │     └─ media/hls/...         # dev HLS fallback files
│  │  └─ src/
│  │     ├─ app/
│  │     │  ├─ layout.tsx
│  │     │  ├─ page.tsx              # Home
│  │     │  ├─ browse/
│  │     │  │  └─ page.tsx
│  │     │  ├─ titles/
│  │     │  │  └─ [slug]/page.tsx
│  │     │  └─ admin/
│  │     │     └─ preview/page.tsx   # Admin preview (client-side)
│  │     ├─ components/
│  │     │  ├─ Navbar.tsx
│  │     │  ├─ BottomNav.tsx
│  │     │  ├─ HeroCarousel.tsx
│  │     │  ├─ PosterCard.tsx
│  │     │  ├─ CircularScroller.tsx
│  │     │  ├─ GenreTile.tsx
│  │     │  ├─ FeaturedCard.tsx
│  │     │  ├─ EpisodeStrip.tsx
│  │     │  └─ HlsPlayer.tsx
│  │     ├─ lib/
│  │     │  ├─ api.ts                # front <-> backend client helpers
│  │     │  └─ assetsMap.ts
│  │     └─ styles/
│  │        └─ globals.css
│  │
│  ├─ admin/                          # Optional separate admin app (React/Next)
│  │  ├─ package.json
│  │  ├─ src/
│  │  │  ├─ app/
│  │  │  │  ├─ layout.tsx
│  │  │  │  └─ page.tsx
│  │  │  ├─ components/
│  │  │  │  ├─ TitlesTable.tsx
│  │  │  │  ├─ TitleEditor.tsx
│  │  │  │  ├─ EpisodeUploader.tsx   # presign + multipart upload + progress
│  │  │  │  └─ JobQueueMonitor.tsx
│  │  │  └─ lib/
│  │  │     └─ adminApi.ts
│  │  └─ public/
│  │     └─ admin-assets/
│  │
│  └─ api/                            # Backend service (Next.js app or standalone Express/Fastify)
│     ├─ package.json
│     ├─ tsconfig.json
│     ├─ src/
│     │  ├─ server.ts                 # entry for standalone server (if not Next.js)
│     │  ├─ app/                      # if Next.js App Router - api routes live here
│     │  ├─ routes/
│     │  │  ├─ public/
│     │  │  │  ├─ titles.ts
│     │  │  │  └─ episodes.ts
│     │  │  └─ admin/
│     │  │     ├─ presign.ts
│     │  │     ├─ titles.ts
│     │  │     ├─ episodes.ts
│     │  │     ├─ jobs.ts
│     │  │     └─ users.ts
│     │  ├─ lib/
│     │  │  ├─ prisma.ts              # prisma client
│     │  │  ├─ s3.ts                  # S3 presign / upload helpers
│     │  │  ├─ auth.ts                # Clerk helpers + requireRole
│     │  │  ├─ queue.ts               # BullMQ queue instance
│     │  │  ├─ transcode.ts           # helper for worker integration (used by worker)
│     │  │  └─ logging.ts
│     │  └─ workers/                  # lightweight in-server worker endpoint(s)
│     │     └─ index.ts
│     └─ prisma/
│        ├─ schema.prisma
│        ├─ migrations/
│        └─ seed.ts
│
├─ packages/
│  ├─ ui/                             # shared UI components & styles
│  │  ├─ package.json
│  │  └─ src/
│  │     └─ components/ shared...
│  └─ types/                          # shared TypeScript types
│     └─ index.ts
│
├─ worker/                            # Dedicated worker service (transcode + jobs)
│  ├─ package.json
│  ├─ Dockerfile
│  ├─ src/
│  │  ├─ index.ts                     # BullMQ worker bootstrap
│  │  ├─ transcodeJob.ts              # contains ffmpeg spawn & upload pipeline
│  │  ├─ ffmpeg-commands.ts           # templates for ffmpeg commands
│  │  ├─ s3Uploader.ts
│  │  └─ health.ts
│  └─ scripts/
│     └─ run-local-worker.sh
│
├─ infra-scripts/
│  ├─ setup-minio.sh                  # local minio setup helper
│  └─ setup-postgres.sh
│
├─ devops/
│  ├─ docker/
│  │  └─ Dockerfile.web
│  └─ k8s/
│     └─ manifests...
│
├─ tests/
│  ├─ unit/
│  │  └─ api.utils.test.ts
│  ├─ integration/
│  │  └─ prisma.test.ts
│  └─ e2e/
│     ├─ playwright.config.ts
│     └─ admin-upload.spec.ts
│
├─ docs/
│  ├─ architecture.md
│  ├─ deployment.md
│  ├─ runbook.md
│  ├─ legal/
│  │  ├─ privacy-policy.md
│  │  └─ terms-of-service.md
│  └─ onboarding.md
│
├─ ci-scripts/
│  ├─ lint.sh
│  ├─ test.sh
│  └─ build.sh
│
└─ misc/
   ├─ assets-csv/                     # optional CSV for uploader mapping
   └─ sample-uploads/                 # small mp4s, sample hls for seeding
```

---

## Short descriptions / what lives where (quick reference)

* **`apps/frontend`** — public-facing Next.js app (UI code, client components, HLS player). Put all user-facing pages here.
* **`apps/admin`** — admin dashboard (if separate app). Contains upload UI, job monitor, moderation tool.
* **`apps/api`** — backend API (Next.js server routes or standalone Express). Contains routes for public catalog, admin presign, episode creation, hls signing, favorites, playback.
* **`worker`** — independent service that consumes BullMQ queue, runs FFmpeg, uploads HLS artifacts, updates DB.
* **`packages/ui`** — shared UI components if you prefer monorepo sharing.
* **`prisma/`** — DB schema, migrations, seeds.
* **`infra/`** — terraform / cloudformation / k8s manifests to provision infra.
* **`tests/`** — unit, integration, E2E tests (Playwright).
* **`docs/`** — architecture, deployment, runbooks, legal docs.

---

## Example key files you should implement first

1. `apps/api/src/lib/prisma.ts` — instantiate Prisma client
2. `apps/api/src/routes/admin/presign.ts` — presign endpoint for S3 uploads
3. `worker/src/transcodeJob.ts` — job handler that runs FFmpeg and uploads
4. `apps/api/src/routes/admin/episodes.ts` — create episode & enqueue job
5. `apps/frontend/src/app/titles/[slug]/page.tsx` — Title detail + HlsPlayer using signed HLS URL
6. `apps/admin/src/components/EpisodeUploader.tsx` — presign + upload + progress
7. `prisma/seed.ts` — 20 sample titles using provided images + small HLS preview

---

## Helpful commands (repo root)

```bash
# start local dev infra
docker compose up --build

# run prisma migrations
pnpm --filter api prisma migrate dev

# seed DB
pnpm --filter api node prisma/seed.js

# run frontend
pnpm --filter frontend dev

# run api (if standalone)
pnpm --filter api dev

# run worker locally
pnpm --filter worker dev
```

(If using pnpm workspaces; adapt to npm/yarn if needed.)

---

## Acceptance checklist (repo-level)

* [ ] `README.md` has quickstart for local dev (docker-compose + seed + run)
* [ ] `.env.example` lists all keys (DB, Clerk, S3, Redis, FFMPEG path)
* [ ] `prisma/schema.prisma` & `prisma/seed.ts` present
* [ ] `apps/frontend` uses images from `public/` and HLS fallback files
* [ ] `apps/admin` has uploader using `POST /api/admin/presign`
* [ ] `apps/api` has admin endpoints with Clerk `requireRole`
* [ ] `worker` consumes jobs and updates DB on success/failure
* [ ] GitHub Actions CI runs lint, unit tests, basic e2e

---


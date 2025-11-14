# Anispacex

Is that really... the limit to your power?" and "Human beings are strong because we have the ability to change ourselves🌌

---

# Studio.Anime – Full-Stack Anime Streaming Platform

> **“Human beings are strong because we have the ability to change ourselves.”** 🌌
>
> **Studio.Anime** is a complete end-to-end full-stack Anime Streaming Platform built with a modern, scalable architecture including **Next.js**, **Node/Express or App Router API**, **PostgreSQL**, **Prisma**, **Clerk authentication**, **S3 media storage**, **HLS video streaming**, **FFmpeg transcoding pipeline**, **BullMQ workers**, and a full-featured **Admin Dashboard**.
>
> This repository provides a production-ready monorepo structure including **frontend UI**, **backend API**, **admin dashboard**, **worker service**, **infrastructure**, **tests**, and **CI/CD pipelines**.

---

# 🚀 Features

## 🎨 Frontend (Next.js + TailwindCSS)

* Modern anime UI based on your custom designs and images.
* Mobile-first responsive layouts.
* Poster grid, hero carousel, circular studio scroller.
* Beautiful title detail page with badges, meta info, episodes list.
* HLS video player with fallback support using **hls.js**.
* User authentication and favorites via Clerk.
* Bottom navigation for mobile.
* Browse with categories, filters, search.

---

## 🛠️ Backend (Node.js / Next.js Server Actions / Express)

* REST/JSON API for titles, episodes, search, favorites, playback.
* Secure Clerk-based authentication with RBAC.
* Prisma ORM with PostgreSQL.
* HLS streaming endpoint with signed URLs.
* S3 integration for video & assets.
* Caching & performance optimization.

---

## 🧰 Admin Dashboard

* Create & edit anime titles.
* Upload posters, banners, thumbnails.
* Episode creation with direct S3 upload.
* Video transcoding queue management.
* Log viewer & status monitoring.
* User management (roles, moderation).
* TMDB import (optional).

---

## 🎞️ Transcoding Worker (FFmpeg + BullMQ)

* Consumes queued jobs for video transcoding.
* Generates multi-bitrate HLS variants.
* Extracts thumbnails & preview clips.
* Uploads segmented HLS structure to S3.
* Updates DB with master.m3u8 URL.

---

## 🗄️ Database (PostgreSQL + Prisma)

Tables include:

* `User`
* `Title`
* `Season`
* `Episode`
* `Favorite`
* `PlaybackPosition`
* `Job`
* `Asset`
* `ActivityLog`

Migration and seed files included.

---

## ☁️ Infrastructure

* Docker Compose (dev): Postgres, Redis, MinIO, Worker, Frontend, API.
* Production-ready IaC via Terraform / CloudFormation (optional).
* Deployment with Vercel (frontend), Render/ECS (API + Worker).
* S3 + CloudFront for media delivery.
* Redis (queue + caching).

---

## 🧪 Testing

* Unit tests (Vitest / Jest).
* Integration tests (DB + API).
* Playwright E2E tests for admin + streaming flows.
* GitHub Actions CI pipelines.

---

# 📁 Repository Structure

```
studio-anime/
├─ apps/
│  ├─ frontend/        # Next.js UI
│  ├─ admin/           # Admin Dashboard
│  └─ api/             # Backend API
├─ worker/             # FFmpeg transcoder + BullMQ
├─ prisma/             # Schema, migrations, seeds
├─ infra/              # Terraform / CloudFormation / k8s
├─ docs/               # Architecture, backend, admin, frontend, practical notes
├─ tests/              # Unit, integration, e2e
└─ docker-compose.yml
```

All files described in detailed structure documentation (`structure.md`).

---

# 🔐 Authentication

* Uses **Clerk** for user sessions.
* RBAC with metadata roles: `viewer`, `editor`, `admin`.
* Middleware-level route protection.
* Secure API endpoint access.

---

# 🎬 Video Pipeline Overview

1. Admin uploads MP4 via presigned URL.
2. API stores metadata and queues job.
3. Worker downloads source.
4. Worker runs FFmpeg to generate multiple HLS qualities.
5. Uploads `.ts` segments and master playlist.
6. CDN cache distributes final video.
7. Player loads signed HLS master URL.

---

# 📦 Local Development Setup

### 1. Clone repo

```
git clone https://github.com/your-org/studio-anime.git
cd studio-anime
```

### 2. Copy env template

```
cp .env.example .env
```

Fill in S3, Postgres, Clerk, Redis, and worker variables.

### 3. Start local infra

```
docker compose up --build
```

### 4. Run migrations & seed

```
pnpm --filter api prisma migrate dev
pnpm --filter api node prisma/seed.js
```

### 5. Start frontend

```
pnpm --filter frontend dev
```

### 6. Start backend API

```
pnpm --filter api dev
```

### 7. Start worker

```
pnpm --filter worker dev
```

---

# 🌍 Production Deployment

### Frontend

* Vercel (recommended)
* Automatic builds from `/apps/frontend`

### Backend API

* AWS ECS Fargate / Render / Fly.io
* Connect to managed PostgreSQL

### Worker

* ECS / Render Worker / Cloud Run Job

### CDN + Storage

* AWS S3 + CloudFront
* Signed URLs for secure streaming

---

# 🛡️ Security Checklist

* HTTPS everywhere
* Clerk JWT verification
* Private S3 bucket
* Signed URL streaming
* Role-based admin access
* Rate limits on upload + auth APIs
* Prisma input validation
* Activity logs for moderation
* No secrets committed (use `.env`)

---

# 🧾 Docs Included

* `admin-dashboard.md`
* `backend.md`
* `fronted.md`
* `practical.md`
* `structure.md`

These describe architecture, UI flows, backend APIs, worker pipeline, and practical implementation details.

---

# 🧭 Roadmap

* [ ] Multi-language subtitles
* [ ] Multi-region CDN setup
* [ ] Advanced search (Meilisearch)
* [ ] Recommendation engine (vector search)
* [ ] Ad-supported mode / monetization
* [ ] Mobile apps (React Native + Expo)

---

# ✨ Credits

Built for **Studio.Anime / Anispacex** by passionate developers and AI-powered workflows. All images, UI assets, and designs used as per user-provided resources.

If you like this project, ⭐ **star it on GitHub** & follow for updates.

---

# 🔥 Final Quote

> **“Is that really... the limit to your power?”** 👊

**No. We're just getting started.** 💫

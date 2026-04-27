# TaskFlow — Multi-tenant B2B SaaS

**Project:** TaskFlow — Employee task, project & meeting management with WhatsApp-based reminders
**Role:** Software Developer Intern · Sole engineer
**Company:** Digital Inclined (Noida, hybrid)
**Duration:** January 2026 – Present
**Status:** In pilot with 3 companies (pre-GA, unmonetized pilot phase)

> **Note:** TaskFlow is an active commercial product currently in pilot with 3 companies. The source code is proprietary and cannot be made public. This repository is a detailed engineering case study of the system I am building as the sole engineer — architecture, key technical decisions, and scale.

---

### 🎥 Demo

A live walkthrough of the application (web app + WhatsApp reminder flow + org-admin console) is available on request. Email **pvk1294@gmail.com** with the subject **"TaskFlow demo access"** and I will share a recorded walkthrough or a live session.

---

### 📝 The Problem

Small Indian SMBs (3–50 employees) run on WhatsApp, not email. Existing project-management tools — Asana, ClickUp, Trello — assume an email-first culture, a Slack-first chat layer, and a per-seat pricing model that doesn't survive a 5-person team's budget review. TaskFlow targets that gap: the same task / meeting / project surface area, with reminders and notifications delivered where these teams actually live (WhatsApp), under a multi-tenant architecture so the agency operating it can onboard new client companies with a single organization-creation form.

---

### 🏗️ Architecture

````
┌──────────────────────────────────┐
│ Next.js 14 (App Router) Client   │
│  Web app · Socket.io client      │
└─────────┬────────────────────────┘
          │ HTTPS · JWT in HTTP-only cookie
          ▼
┌──────────────────────────────────┐         ┌──────────────────┐
│ Express 5 API (Node 20)          │◀───────▶│ PostgreSQL 15    │
│  82+ endpoints · 15 modules      │  Prisma │ 23 models        │
│  middleware: auth · RBAC · subs  │   ORM   │ multi-tenant via │
└──┬─────────┬────────┬────────────┘         │ organizationId   │
   │         │        │                      └──────────────────┘
   │         │        │
   │         │        └─► Socket.io ──► per-user rooms (chat + live notifications)
   │         │
   │         └─► node-cron (every 1m, 10-min window for meeting reminders)
   │
   └─► BullMQ queue ──► Redis ──► Worker (concurrency 5)
       enqueue-on-create,        │
       fire at T-24h             ├─► WhatsApp (BotBiz)
                                 ├─► MessageLog (audit row per send)
                                 └─► retry / dead-letter

External integrations:
  WhatsApp (BotBiz) · Google Calendar · Cloudinary · AWS S3 · Resend (email)
````

Hosted on AWS EC2 with a CI/CD pipeline. PostgreSQL and Redis containerised via Docker Compose.

---

### 🔑 Key Engineering Decisions

> **Why a delayed-job queue, not a database-polling cron, for task reminders?**
>
> The naive design polls the `Task` table every minute looking for tasks due in 24 hours — that's an O(n) scan every minute, getting worse as task count grows, and the WhatsApp API latency couples back to whatever process is scanning. Instead, I enqueue a delayed BullMQ job at task-creation time, and a worker (concurrency 5) fires at T-24h. **O(1) dispatch, survives backend restarts (Redis-persisted), retries and dead-letter built in, and the request that creates a task isn't blocked on WhatsApp delivery.** Meeting reminders still use `node-cron` because the reminder window is small (10 minutes) and the per-scan row count is tiny — pragmatic trade-off, not dogma.

> **Multi-tenancy: shared database with `organizationId`, not schema-per-tenant.**
>
> At 3-pilot scale, schema-per-tenant means N migrations to run, N connection pools to size, and a much harder analytics surface. A shared DB with an `organizationId` column on every tenant-scoped table — filtered at the middleware layer — keeps operations simple and lets one set of indexes work for all tenants. Postgres row-level security is the planned next step before the tenant count crosses ~50 and the "trust the middleware" story gets uncomfortable.

> **Two-tier RBAC: `PlatformRole` and `OrgRole`, checked separately.**
>
> `PlatformRole` (`SUPER_ADMIN`, `WHITE_LABEL_PARTNER`) is coarse — it gates access to the entire platform surface. `OrgRole` (`ORG_OWNER`, `MANAGER`, `DOER`) is fine-grained and scoped per organization. Splitting them lets a white-label partner manage many client orgs without inheriting `ORG_OWNER` powers in any one of them, and keeps each org-level check to a single role lookup against the `organizationId` already on the JWT.

> **Refresh-token rotation with HTTP-only cookies, not pure-JWT sessions.**
>
> Pure-JWT sessions can't be revoked before token expiry — for a B2B product with paid subscriptions and admin-controlled access, that isn't acceptable. Access tokens live 1 day; refresh tokens live 7 days, are stored in HTTP-only cookies (not localStorage), and rotate on every use against a `RefreshToken` table. Logout, password change, and subscription suspension all invalidate tokens immediately. TOTP 2FA via `otplib` with QR setup and backup codes is enforced for `ORG_OWNER`s.

---

### 📊 Scale

| Metric | Value |
| --- | --- |
| Pilot companies | 3 (active) |
| Prisma models | 23 |
| API endpoints | 82+ across 15 route modules |
| Backend code | ~9.3k LOC |
| Frontend code | ~23.7k LOC |
| Async workers | BullMQ (task reminders) + node-cron (meeting reminders) |
| Realtime | Socket.io with per-user rooms |
| Auth | JWT + refresh-token rotation + TOTP 2FA |

---

### 💻 Technology Stack

| Category | Tools |
| --- | --- |
| **Backend** | Node 20 · Express 5 · Prisma 6 · PostgreSQL 15 |
| **Async** | BullMQ · Redis (ioredis) · node-cron |
| **Realtime** | Socket.io |
| **Auth** | jsonwebtoken · bcryptjs · otplib (TOTP 2FA) |
| **Notifications** | BotBiz WhatsApp API · Resend (email) |
| **Files** | Cloudinary · AWS S3 · Multer |
| **Integrations** | Google Calendar API · PDFKit · qrcode |
| **Security** | helmet · cors · cookie-parser · subscription-state middleware |
| **Frontend** | Next.js 14 (App Router) · React 18 · Tailwind 3 · Framer Motion · Recharts · Socket.io client |
| **Deploy** | Docker Compose · AWS EC2 · CI/CD pipeline |

---

### 📫 Discuss

If you'd like a walkthrough of the architecture, a live demo, or to discuss any of the engineering decisions above:

- **Email** — pvk1294@gmail.com
- **LinkedIn** — [linkedin.com/in/pvk1294](https://linkedin.com/in/pvk1294)
- **GitHub** — [github.com/pvk1294](https://github.com/pvk1294)

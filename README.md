# Dempo Learn LMS

A learning management system for a single Indian college (~2,000 students, ~200 staff). Teachers run courses, assignments, quizzes, attendance and a weighted gradebook. Students submit work and track their grades. Admins control who gets in.

> **Read this first.** This repo was inherited without an entry point, and its other docs contradict each other and the code. Where they disagree, **trust the code and git history**. This README only states what we checked in the source. Anything we have not verified is marked **TODO — unverified**.

---

## What it is

- **Users and roles:** `student`, `teacher`, plus an `isAdmin` flag (`lib/db/src/schema/users.ts`) and a coordinator concept (`lib/db/src/schema/coordinatorAssignments.ts`).
- **Access is invite-only.** On first sign-in, anyone not listed in `ADMIN_EMAILS` and not on the invite list gets `403 not_invited` (`artifacts/api-server/src/middlewares/auth.ts`). Admins invite people from the **Invitations** page (`artifacts/dempo/src/pages/admin-invites.tsx`).
- **Features:** courses, assignments (text, file, link, video, audio), quizzes, rubrics, weighted gradebook, attendance, discussions, announcements, journal, cohorts and groups, early alerts, transcript, course plan, student planner, analytics with CSV export. See `lib/db/src/schema/` and `artifacts/api-server/src/routes/`.
- **AI grading and email are built but OFF by default** (`AI_GRADING_ENABLED=false`, `MSG91_EMAIL_ENABLED=false` in `artifacts/api-server/.env.example`). Plagiarism checking is local word-similarity, with no external service (`artifacts/api-server/src/lib/similarity.ts`).

## Tech stack

| Part | Tech | Where |
|---|---|---|
| Monorepo | pnpm workspaces, TypeScript | `pnpm-workspace.yaml` |
| Web app | React 19, Vite, Tailwind CSS v4, shadcn/ui | `artifacts/dempo/` |
| API | Express 5 (Node) | `artifacts/api-server/` |
| Database | PostgreSQL + Drizzle ORM | `lib/db/` (schema = source of truth) |
| API contract | OpenAPI → generated client + Zod | `lib/api-spec/openapi.yaml` |
| Auth | Clerk | `artifacts/api-server/src/middlewares/auth.ts` |
| File storage | Any S3-compatible bucket | `artifacts/api-server/src/lib/objectStorage.ts` |
| Theme / design tokens | Tailwind v4 CSS | `artifacts/dempo/src/index.css` |

## Repo map

```
artifacts/dempo/          Web app (React)           ← UI, pages in src/pages/
artifacts/api-server/     API (Express) + tests     ← routes in src/routes/
artifacts/mockup-sandbox/ UI prototyping scratchpad  (not part of the product)
lib/db/                   Database schema (Drizzle)
lib/api-spec/             OpenAPI spec + codegen
api/index.mjs             Vercel serverless entry
.github/workflows/ci.yml  CI: typecheck → test → build
```

## How to run it locally

**You need:**
- Node.js 24 (`.node-version`)
- pnpm via corepack
- Git (on Windows, **Git for Windows** is required; see below)
- A PostgreSQL database
- A Clerk account (free dev instance)

**Never commit `.env` or paste keys into chats or issues.**

1. **Install dependencies**
   ```bash
   corepack enable
   corepack pnpm install
   ```
2. **Create a `.env` at the repo root**, not in `artifacts/api-server/`. The web app reads the root `.env` (`artifacts/dempo/vite.config.ts`). Start from `artifacts/api-server/.env.example` and **also add** `VITE_CLERK_PUBLISHABLE_KEY` (same value as `CLERK_PUBLISHABLE_KEY`). The example file is missing it.
   - Set `ADMIN_EMAILS` to the email you'll sign in with. Otherwise you will be locked out (`403 not_invited`).
3. **Create the database tables**
   ```bash
   corepack pnpm --filter @workspace/db run push
   ```
4. **Start the app**
   - **Windows:** `powershell -ExecutionPolicy Bypass -File .\dev.ps1` opens the API on `http://localhost:8080` and the web app on `http://localhost:5173`.
   - **macOS/Linux:** export the `.env` variables into your shell, set `PORT=8080`, then run `pnpm --filter @workspace/api-server run dev`. In a second terminal, run `pnpm --filter @workspace/dempo run dev`.
5. **Check it:** `http://localhost:8080/api/healthz` should return OK. Then open `http://localhost:5173` and sign in with your admin email.

**Run the tests** (needs a throwaway Postgres in `DATABASE_URL`):
```bash
corepack pnpm run typecheck
corepack pnpm --filter @workspace/api-server run test
```

TODO — unverified: the steps above are reconstructed from config files and not yet tested end-to-end. Our failure log from Task 3 will update this section.

## Where it deploys

- **Current target: Vercel** (`vercel.json`; `ci.yml` notes "Vercel deploys from main"), with the database on **Supabase Mumbai** (`MIGRATE-SUPABASE.md`, commit `5b913b9`).
- `render.yaml` / `DEPLOY.md` (Render) and the DigitalOcean plan in `ARCHITECTURE.md` / `PLAN.md` are **outdated**. Replit files (`.replit`, `replit.md`) are leftovers.
- **Do not touch the live deployment.** Deploy your own copy if you need one.
- TODO — unverified: the live URL, and whether the database and file storage really are in India.

## Known broken or risky

| Issue | Evidence |
|---|---|
| `pnpm install` fails on Windows without Git Bash: the `preinstall` script uses `sh -c` | `package.json`; workaround in `dev.ps1` |
| API `dev` script fails on Windows: it uses `export` | `artifacts/api-server/package.json` |
| API crashes on start without `PORT` | `artifacts/api-server/src/index.ts` |
| `.env.example` is in the wrong folder and missing `VITE_CLERK_PUBLISHABLE_KEY` | `artifacts/api-server/.env.example` vs `artifacts/dempo/vite.config.ts` |
| File uploads do nothing until `STORAGE_*` variables point at a real bucket | `DEPLOY.md`, `artifacts/api-server/src/lib/objectStorage.ts` |
| Docs contradict the code: role picker, AI model, hosting, data residency | See the contradictions table in our Task 2 notes |
| Replit leftovers: plugins, `*.replit.app` share links | `artifacts/dempo/vite.config.ts`, `artifacts/dempo/index.html` |
| Newest and least-proven feature: course-plan autosave (last commit fixes data loss) | `git log`, commit `02b6bc7` |
| No exam timer or auto-save, no CSV roster import, no monitoring, no legal/DPDP docs | `ARCHITECTURE.md` §15 |

## Where to look first

1. This README → `.github/workflows/ci.yml` (what is *supposed* to pass)
2. `lib/db/src/schema/` (what data exists)
3. `artifacts/api-server/src/middlewares/auth.ts` + `src/lib/authz.ts` (who can do what)
4. `artifacts/dempo/src/pages/` (every screen)
5. `ARCHITECTURE.md` for the *intended* design. Treat it as a plan, not a description.

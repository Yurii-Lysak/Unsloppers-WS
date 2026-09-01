# Deployment (Agent Reference)

> **Purpose:** Current state of the deployed environment — where each piece runs, how it auto-deploys, and the env vars each platform needs. Supersedes AD-12's original docker-compose plan; see `_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md`'s AD-12 for why.
>
> **Story:** 1.17 (`_bmad-output/planning-artifacts/epics.md`).

---

## 1. Topology

Three separate managed platforms, one production environment, no staging tier:

| Piece | Repo | Platform | Auto-deploy trigger |
|---|---|---|---|
| Frontend (React/Vite SPA) | `Unsloppers-FE` | **Vercel** | push to `main` |
| Backend (NestJS API) | `Unsloppers-BE` | **Render** | push to `main` |
| Database (Postgres 18) | — | **Neon** (managed) | n/a |

Both Vercel and Render install a webhook on the connected GitHub repo automatically when the project/service is created — there is no separate toggle for "deploy on push." A push to any other branch/a PR produces a Preview deployment on Vercel; Render has no built-in preview-per-PR equivalent on the free tier, it only tracks the one configured branch.

This replaced the architecture spine's original AD-12 plan (a single docker-compose stack — backend, frontend, Postgres — on one host). Deploying a persistent NestJS/Express server (`app.listen(PORT)` in `services/backend/src/main.ts`) onto Vercel would have required rewriting it into a serverless handler; Render runs it as-is.

---

## 2. Frontend — Vercel

- Framework preset: Vite (auto-detected). Build command `npm run build` (`tsc -b && vite build`), output directory `dist`.
- `services/frontend/vercel.json` sets `installCommand: npm install --include=dev` — Vercel's default `npm install` under `NODE_ENV=production` skips `devDependencies`, which breaks the build (Tailwind's Vite plugin and other build-time tooling live there). Don't override the install command in the dashboard; let this file win.
- Root Directory: `.` — `Unsloppers-FE` is its own repo, not a monorepo subfolder.
- Production Branch: `main` (Vercel default on import).

Environment variables (Project Settings → Environment Variables):

| Var | Value |
|---|---|
| `VITE_API_BASE_URL` | the Render backend URL, e.g. `https://unsloppers-be.onrender.com` |
| `VITE_API_TIMEOUT` | `30000` |

`VITE_*` vars are inlined at **build time** by Vite, not read at runtime — changing one in the dashboard requires a redeploy (Deployments → ⋯ → Redeploy) to take effect, a restart alone does nothing.

---

## 3. Backend — Render

- Runtime: Node. Branch: `main`. Root Directory: repo root.
- Build Command: `npm install --include=dev && npm run build`. The `--include=dev` is required for the same reason as the frontend: Render sets `NODE_ENV=production` during the build, which makes plain `npm install` skip `devDependencies` — and `@nestjs/cli` (needed for `nest build`) lives there. Without it the build fails with `sh: 1: nest: not found`.
- Start Command: `npm run start:prod` (`node dist/src/main`).
- Settings → Build & Deploy → Auto-Deploy: `Yes`, tied to `main` — Render's equivalent of Vercel's default git integration.
- Don't set a `PORT` env var — Render injects its own, and `main.ts` reads it via `ConfigService.getOrThrow('PORT')` already.

Environment variables (Environment tab) — see `services/backend/.env.example` and `src/config/env.validation.ts` for the full schema:

| Var | Value |
|---|---|
| `NODE_ENV` | `production` |
| `DATABASE_URL` | Neon **direct** (non-pooled) connection string, `sslmode=require` |
| `JWT_SECRET` | random, 32+ chars — never the `.env.example` placeholder |
| `JWT_TTL_SECONDS` | `3600` |
| `CORS_ORIGIN` | exact Vercel frontend origin, e.g. `https://unsloppers-fe.vercel.app` — scheme + host, no trailing slash |
| `HR_DEPARTMENT_VALUE` | `HR` |
| `BOOTCAMP_INITIAL_PASSWORD` / `TIMETRACKER_*` | only if actually needed in prod |

`CORS_ORIGIN` must match the FE origin **exactly** — `enableCors()` in `src/bootstrap.ts` echoes this string verbatim as `Access-Control-Allow-Origin`, paired with `credentials: true`; a mismatch (wrong scheme, trailing slash, wrong subdomain) breaks every cross-origin request silently.

### Migrations and seeding

`postbuild` in `package.json` runs `prisma migrate deploy` only — **not** `db:seed`. It used to run both on every build (`prisma migrate deploy && npm run db:seed`), which would re-run the seed script against the real database on every push once auto-deploy was live (fixed in `Unsloppers-BE#20`). Seed manually, once, with `npm run db:seed` run locally against the target `DATABASE_URL` — it upserts by identity, so it's safe to re-run deliberately, just not automatically.

### Neon connection string

Use the **direct** (non-pooled) connection string, not the `-pooler` one. Render runs one persistent Node process, and `@prisma/adapter-pg` manages its own connection pool — pairing that with Neon's PgBouncer transaction-mode pooling can break prepared statements. Grab it from Neon Console → project → Connection Details, with the pooling toggle off.

**Known limitation:** local dev's `services/backend/.env` currently points `DATABASE_URL` directly at this same Neon database — there is no separate non-production database yet. This is fine for now (the seeded population is already pseudonymized bootcamp test data, not real PII — NFR-5 is not violated), but it means local development and the deployed environment share state: a local `prisma migrate dev` or ad hoc local write touches the same rows Render serves. Revisit with a second Neon branch/database for local dev if this becomes a problem.

---

## 4. Cross-site auth — why the cookie needed a code change

FE (`*.vercel.app`) and BE (`*.onrender.com`) are different registrable domains, i.e. genuinely **cross-site**, not just cross-port like local dev (`localhost:4200` → `localhost:3001`, which is same-site regardless of port). This has two consequences, both already handled in code — noted here so nobody "fixes" them back:

- **Session cookie `SameSite`** (`services/backend/src/modules/auth/auth-cookie.ts`): `SameSite=Strict` (or even `Lax`) is silently dropped by the browser on cross-site requests. Symptom was exactly this: `POST /auth/login` succeeds and sets the cookie, but the very next call (`GET /permissions/me`) 401s because the cookie never gets attached. Fixed in `Unsloppers-BE#22` — `sameSite` is `'none'` in production (paired with the `secure: true` that `SameSite=None` requires), and stays `'strict'` outside production since local FE/BE share the `localhost` site.
- **CORS** (`services/backend/src/bootstrap.ts`): `enableCors({ origin: CORS_ORIGIN, credentials: true })` must echo the exact FE origin, not a wildcard, for the browser to accept cross-origin cookies at all — already correct, just easy to break by typo'ing `CORS_ORIGIN`.

**Residual caveat, not yet hit:** `SameSite=None; Secure` makes the cookie eligible to be sent cross-site, but some browsers (Safari ITP today, Chrome eventually) additionally block **third-party cookies** outright regardless of `SameSite`, since this cookie genuinely belongs to a third-party domain from the FE page's point of view. If auth ever breaks in Safari or a hardened Chrome profile while working fine elsewhere, that's why — the real fix at that point is putting FE and BE on the same registrable domain (e.g. `app.yourdomain.com` + `api.yourdomain.com`, or a Vercel rewrite proxying `/api/*` to Render), not another cookie flag.

---

## 5. Verifying a deploy

1. Push to `main` on `Unsloppers-FE` and/or `Unsloppers-BE` — check the platform dashboard for the triggered deployment.
2. Backend: `Unsloppers-BE`'s Render build log should show `prisma migrate deploy` running in `postbuild`; confirm no unexpected migration failures.
3. Frontend: after a Vercel deploy, load the app and confirm `VITE_API_BASE_URL` is baked in correctly (Network tab — API calls should hit the Render URL, not `localhost`).
4. Log in through the FE login page, then confirm an authenticated call (e.g. `/permissions/me`) succeeds rather than 401ing — this is the fastest end-to-end check that `CORS_ORIGIN`, the cookie `SameSite` setting, and the Vercel↔Render pairing are all correct together.

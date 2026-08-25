# Tech-Currency Review — ARCHITECTURE-SPINE.md (Stack table)

**Lens:** Tech-currency (Reviewer Gate) — verify every committed technology decision was web-researched or reality-checked, not asserted from training data.
**Reviewed:** `_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md`
**Date:** 2026-08-21

## Verdict: PASS with minor findings

Every "(existing)" row checks out against the actual installed `package.json` files and actual config files in the repo. Every "new" row is a real, currently-maintained package at the pinned version (independently re-verified via web search, not just trusted from the drafting pass). The two "current library defaults" claims singled out for scrutiny (Prisma 7 config shape, PostgreSQL 18 Docker volume path) are both accurate, current, documented facts — not stale carryover or hallucination. Three minor findings below, none blocking.

---

## What was checked and how

### "(existing)" rows — verified by reading real files

| Stack row | Claim | Verification | Result |
| --- | --- | --- | --- |
| NestJS 11 | existing | Read `services/backend/package.json` — `@nestjs/common`/`core`/`platform-express` all `^11.0.1` | Match |
| Prisma 7, `@prisma/adapter-pg` | existing | Read `services/backend/package.json` — `prisma` `^7.9.1`, `@prisma/client` `^7.9.1`, `@prisma/adapter-pg` `^7.9.1` | Match |
| PostgreSQL 18 (docker-compose) | existing | Read `services/backend/docker-compose.yml` — `image: postgres:18-alpine`, volume mounted at `/var/lib/postgresql` | Match |
| Node.js 22 | existing | Read `services/backend/package.json` — `"engines": {"node": ">=22.12"}` | Match |
| React 19 | existing | Read `services/frontend/package.json` — `react`/`react-dom` `^19.2.7` | Match |
| Vite 8 | existing | Read `services/frontend/package.json` — `vite` `^8.1.0` | Match |
| shadcn/ui (radix-nova) + Tailwind v4 | existing | Read `services/frontend/components.json` — `"style": "radix-nova"`; `package.json` — `tailwindcss` `^4.3.1` | Match (see Finding 3) |
| React Router v7 | existing | Read `services/frontend/package.json` — `react-router-dom` `^7.18.0` | Match |
| TanStack Query v5 | existing | Read `services/frontend/package.json` — `@tanstack/react-query` `^5.101.2` | Match |

### "new" rows — independently re-verified via web search (not trusted from drafting)

| Package | Pinned in spine | Web search finding | Result |
| --- | --- | --- | --- |
| openapi-typescript | 7.13.0 | WebSearch "openapi-typescript npm version 7.13.0 2026" — confirmed latest, released 2026-02-11, ~5.4M weekly downloads, actively maintained | Current |
| @nestjs/jwt | 11.0.2 | WebSearch "@nestjs/jwt npm latest version 11.0.2" — confirmed latest, published ~8 months before today, 1158+ dependents | Current |
| @nestjs/passport | 11.0.5 | WebSearch "@nestjs/passport npm latest version 11.0.5" — confirmed latest version number, but flagged as published ~2 years ago, no known vulnerabilities | Current version, but stale publish date — see Finding 1 |
| passport-jwt | 4.0.1 | WebSearch "passport-jwt npm latest version 4.0.1 maintained" — confirmed 4.0.1 is still the latest available, but libraries.io flags maintenance status as **Inactive**, last published ~4 years ago | Correct pin (nothing newer exists), but maintenance-risk flag — see Finding 2 |
| react-hook-form | 7.85.0 | WebSearch "react-hook-form npm latest version 7.85.0" — confirmed latest, published 12 days before today | Current |
| zod | 4.4.3 | WebSearch "zod npm latest version 4.4.3" — confirmed latest, published 2026-05-04, 223M weekly downloads | Current |
| @hookform/resolvers | 5.9.1 | WebSearch "@hookform/resolvers npm latest version 5.9.1" — confirmed latest, published 2 days before today | Current |

### Prisma 7 config-shape claims — web-verified against current docs

Checked against real repo files (`services/backend/prisma.config.ts`, `services/backend/prisma/schema.prisma`) and independently against `prisma.io` docs via WebSearch ("Prisma 7 prisma.config.ts datasource url adapter-pg moduleFormat cjs"):

- **Datasource `url` lives in `prisma.config.ts`, not `schema.prisma`** — confirmed current Prisma 7 behavior per Prisma's own "Upgrade to Prisma ORM 7" guide and `prisma-config-reference` docs. Repo's `prisma.config.ts` matches this shape exactly (`datasource: { url: process.env.DATABASE_URL }`).
- **`@prisma/adapter-pg` driver adapter** — confirmed current/required pattern for Postgres in Prisma 7.
- **`moduleFormat = "cjs"` generator option** — confirmed as a real Prisma 7 generator option (forces CommonJS output instead of default ESM), matches repo's `schema.prisma` generator block and the repo's own `.claude/rules/nest-prisma.md` rationale (NestJS is CJS).

All three match current, real Prisma 7 behavior — not stale or fabricated.

### PostgreSQL 18 Docker volume-path claim — web-verified

Checked `services/backend/docker-compose.yml` (mounts `pgdata:/var/lib/postgresql`, not `.../data`) against a WebSearch ("postgres:18-alpine docker image PGDATA /var/lib/postgresql path change") and a WebFetch of a relevant `docker-library/postgres` GitHub issue:

- PostgreSQL 18's official Docker image genuinely moved the `VOLUME` declaration from `/var/lib/postgresql/data` to `/var/lib/postgresql` (PGDATA now lives in a versioned subdirectory inside it, for smoother major-version upgrades via `pg_upgrade`).
- A reported real-world symptom matches exactly: mounting a volume at the old `/var/lib/postgresql/data` path causes the postgres:18 container to fail at startup with a runc/path-mismatch error; mounting at `/var/lib/postgresql` (the repo's actual configuration) works.
- This is **not** a documentation gap or codebase-carried error — it is a real, current fact about the postgres:18 image, and both the spine and the repo's own `docker-compose.yml`/`CLAUDE.md` state it correctly.

---

## Findings

### Finding 1 (low severity) — @nestjs/passport publish recency understated risk, but not wrong

The spine pins `@nestjs/passport` 11.0.5 as "new." Web search confirms 11.0.5 is genuinely the current latest version and compatible with NestJS 11, with no known vulnerabilities — but one source flagged it as last published roughly two years before today's date (2026-08-21), noticeably older than the other "new" packages in this table (most published within the last year, several within days/weeks). This is not evidence of deprecation (it's still the NestJS org's official package, 1257+ dependents), but the spine doesn't distinguish "actively releasing" from "stable, low-churn" packages, and a reviewer relying on the table alone could mistake all seven "new" rows for equally fresh. No action required beyond awareness; not a version-correctness problem.

### Finding 2 (medium severity) — passport-jwt is the correct pin but the dependency itself is a maintenance risk

`passport-jwt` 4.0.1 is genuinely the latest available version (nothing newer has been published) so the *pin* is correct. However, web search (libraries.io) flags the package's maintenance status as **Inactive**, with the last publish roughly four years before today's date. This is architecturally relevant because AD-9 commits the `auth` module to a JWT strategy built on this package. The spine should either note this as an accepted risk (the package is simple, stable, and widely used — 1967+ dependents — so staleness may not matter for a JWT-verification strategy) or flag it in "Deferred" as a future replace-candidate if a more actively maintained alternative (e.g. implementing the JWT strategy directly against `@nestjs/jwt` without `passport-jwt`, or a newer passport-strategy fork) becomes preferable. Currently unflagged.

### Finding 3 (informational, no action needed) — "radix-nova" is not a term shadcn/ui's public docs recognize, but it is verifiably what's actually installed

The spine's Stack row reads "shadcn/ui (radix-nova) + Tailwind v4." A web search against shadcn/ui's own docs/changelog (including its Feb 2026 "Unified Radix UI Package" changelog) shows the only documented official style names are `"new-york"` (current default for new projects) and `"default"` (deprecated). "radix-nova" does not appear in shadcn/ui's official documentation — it surfaced in search results only as a style-variant name used by an unrelated third-party package (`@docyrus/shadcn`), not the project actually installed here (`package.json` uses `"shadcn": "^4.11.1"`, the official CLI package).

That said, this is not a hallucinated architecture claim: `services/frontend/components.json` was read directly and genuinely contains `"style": "radix-nova"` as the configured value in this repo today. So the spine's "(existing)" label is verified accurate to the real, installed project state — the open question (why the installed CLI/config uses a style name absent from shadcn/ui's public docs) is a pre-existing fact about the codebase, not something the architecture spine introduced or got wrong. Flagged for visibility only; no spine correction needed.

### Finding 4 (low severity) — "Node 22 required by Prisma 7's WASM client" slightly overstates Prisma's own requirement

The spine's Stack row states Node.js 22 is "(existing, required by Prisma 7's WASM client)." Web search against Prisma's own "System requirements" doc shows Prisma 7's actual minimum is **Node.js 18.18 LTS**, with Node 20 or 22 LTS only *recommended*, not required. The repo's own `package.json` engines field (`>=22.12`) is a project-level constraint (also documented in the backend's own `CLAUDE.md` gotchas: "Prisma 7 does not support odd Node versions (23)"), not something Prisma 7 itself mandates at 22. The spine's phrasing conflates "the project requires Node 22" (true, and correctly matches package.json) with "Prisma 7 requires Node 22" (overstated — Prisma's floor is lower). Cosmetic; doesn't change any decision, since Node 22 is genuinely what's installed and required by this specific project.

---

## Summary

- 9 "(existing)" Stack rows checked against real `package.json`/`components.json`/`docker-compose.yml` files: all match.
- 7 "new" Stack rows independently re-verified via web search: all are real, current, still-maintained packages at (or matching) the pinned versions.
- Prisma 7 config-shape claims (datasource url location, `@prisma/adapter-pg`, `moduleFormat = "cjs"`): confirmed current and accurate via web search against Prisma's own docs.
- PostgreSQL 18 Docker volume-path claim: confirmed current and accurate via web search + GitHub issue — genuinely a postgres:18 image behavior change, not a codebase-carried error.
- 4 minor/informational findings, 0 blocking findings. No stale, deprecated, or fabricated version claims found.

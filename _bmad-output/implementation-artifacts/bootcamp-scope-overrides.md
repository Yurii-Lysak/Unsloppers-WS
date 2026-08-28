# Bootcamp scope overrides (agent reference)

**Read this before interpreting seed/population requirements.** The frozen Story 1.16 spec and parts of `docs/project-requirements-v2.md` §4.17 still describe the *original* bootcamp plan (500+ identities via TimeTracker Accounting). **Shipped code follows this document instead** (2026-08-28 pivot).

## Seed population (Story 1.16 — shipped)

| Topic | Original plan (do not implement) | **Current truth (shipped)** |
|---|---|---|
| Identity source | `POST /api/accounting/report` + `GET /api/projects/talents` | Bundled JSON manifest |
| Manifest path (backend) | — | `services/backend/src/prisma/seed/data/bootcamp-identities.json` |
| Source CSV (workspace) | — | `docs/bootcamp-seed-accounts-source.csv` |
| Archive | — | `_bmad-output/archive/bootcamp-seed-identities.json` + `bootcamp-seed-accounts-source.csv` |
| Record count | 500–2000 floor/ceiling (`PopulationSizeError`) | **24** fixed accounts |
| VPN / TimeTracker keys for seed | Required | **Not required** |
| Empty population | Halt (Ask First below 500) | Halt only if manifest has **zero** identities after dedupe |
| `hash` field | From TimeTracker API | Deterministic `SHA256("bootcamp-seed-v1:" + lower(email))` when not in source CSV |

**Commands:** `npm run db:seed` from `services/backend` — no `.env` TimeTracker keys needed.

**Do not:** reintroduce the 500 minimum, block seed on Accounting response size, or tell developers they need VPN to seed locally.

## TimeTracker module (still in codebase)

`TimetrackerService` (`src/modules/timetracker/`) remains for **Epic 13** leave/project sync — **not** for Story 1.16 seeding. Optional env keys: `TIMETRACKER_ACCOUNTING_API_KEY`, `TIMETRACKER_TALENTS_API_KEY`.

OpenAPI contract: `docs/api-external-openapi.json` (vendored copy in `services/backend/docs/`).

## 500+ records elsewhere in the PRD

These **still apply** — they are performance/NFR targets, not seed manifest size:

- §1 / §7: org scale and All Employees list **performance at 500+ records** (Story 3.7)
- Epic 3 list/export perf tests may use **synthetic scale-up** or load fixtures — not the 24-account seed manifest

## PRD §4.17 nuance

§4.17 says population is imported from TimeTracker. **Bootcamp pivot:** the delivered list is **`docs/bootcamp-seed-accounts-source.csv`** (24 rows); platform seed imports that via the JSON manifest. TimeTracker sync for leaves/projects remains Epic 13.

## Related artifacts

- Pivot rationale: `_bmad-output/archive/bootcamp-seed-pivot-2026-08-28.md`
- Story spec (frozen intent + change log + shipped state): `spec-1-16-pseudonymized-seed-data-tool.md`

---
title: 'Pseudonymized Seed Data Tool'
type: 'feature'
created: '2026-08-28'
status: 'done'
review_loop_iteration: 0
baseline_commit: '0b5583e42553d57820cfc37b1cf01ed44d098bf9'
context:
  - '{project-root}/docs/api-external-openapi.json'
  - '{project-root}/docs/bootcamp-seed-accounts-source.csv'
  - '{project-root}/_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md'
  - '{project-root}/_bmad-output/archive/bootcamp-seed-identities.json'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** No environment (local dev, or the single auto-deployed environment) has a populated dataset, and no seed mechanism exists at all yet. The v1.5 requirements update (SPEC §4.17) also redefines the population itself: it is no longer synthetic — it's a real test-user list, delivered by the TimeTracker team, that must be imported rather than invented, while everything the platform owns natively (departments, roles, history) still has no source but generation.

**Approach:** Add a Prisma-standard `prisma/seed.ts` that (1) imports the identity population from the TimeTracker external API (`docs/api-external-openapi.json`) as the authoritative base — id/email/name/hash/countryCode, matched to project membership by email — and (2) layers synthetic values on top for every field/table TimeTracker doesn't carry. One idempotent command wired into both the local dev flow and the existing `postbuild` deploy hook, so "seed the DB" behaves identically everywhere without needing Story 1.17's deployment topology to exist first. Query the Accounting endpoint for the most recent complete calendar month at seed time — a single month, not a range — and treat its `employees[]` as the full population for that run.

## Boundaries & Constraints

**Always:**
- The only identity source is the TimeTracker external API (`POST /api/accounting/report` for id/email/name/hash/countryCode, `GET /api/projects/talents` for project membership by email) — never a hardcoded or hand-authored employee list.
- Every TimeTracker-sourced field is stored verbatim; every field TimeTracker doesn't carry (phone, address, and any platform-native table) is clearly synthetic-generated, never fabricated to look TimeTracker-sourced.
- The seed command is idempotent — rerunning against an already-seeded DB creates no duplicates and exits 0.
- Writes to the four temporal-history tables go only through `PrismaService`'s Client Extension `create` (Story 1.20's only legal write path) — never a raw/bulk insert.
- `TIMETRACKER_ACCOUNTING_API_KEY` / `TIMETRACKER_TALENTS_API_KEY` are added to `env.validation.ts` + `.env.example`, read only via `ConfigService.getOrThrow`.
- The same seed logic runs from both `npm run db:seed` (local) and `postbuild` (auto-deploy) — one implementation, two triggers.
- Fetch and fully validate both TimeTracker responses first — Accounting count meets the 500 floor, both calls succeeded, every record passes required-field validation — before writing anything to the database. No writes start until both preconditions hold, so a failure anywhere (either endpoint, or the floor check) never leaves partial rows.
- Within a single Accounting fetch, deduplicate identities by email before upsert — if two records share an email, keep the last one returned and log a warning; never write two rows for one email.
- On rerun, update an existing row's TimeTracker-sourced fields (name/countryCode/hash) to match the latest response rather than leaving them stale — "stored verbatim" means kept in sync with the latest fetch, not frozen at first-seed values.
- Match Talents project-membership records to identities by email; a membership record whose email has no matching Accounting-endpoint identity is skipped with a logged warning, not a run failure.
- Identities present in the DB from a previous seed run but absent from the latest TimeTracker response are left untouched — this seed path only adds/updates identities, it never deletes or deactivates them.

**Ask First:**
- If the delivered TimeTracker list is smaller than 500 identities: HALT and ask whether to pad with fully synthetic identity-anchor people, rather than silently inventing extra "real-looking" employees.
- If the delivered list exceeds 2,000 identities: HALT and confirm before writing — an unexpectedly large response likely signals a wrong filter or endpoint, not real data.
- Confirm which platform-native tables to seed now: today only `User`/`Employee`/4 history tables/`TimelineEvent` exist — no Department, roles, risk, or notes tables yet. Ask before deciding whether this story's synthetic layer is a no-op beyond history rows for now, versus stubbing data for tables that don't exist.
- Confirm with the TimeTracker team what `hash` represents (opaque identity token vs. credential-derived value) before persisting it verbatim in any environment, including auto-deploy.

**Never:**
- Never call the TimeTracker API from anywhere except this seed path — no runtime/request-time dependency on it (that integration is Epic 13's scope).
- Never import or retain any identity data beyond what the TimeTracker endpoints return.
- Never bypass `temporal-history.extension.ts` for history-table writes.
- Never commit `.env` or hardcode the API keys.
- Never delete or deactivate an identity because it dropped out of a later TimeTracker response — see Always.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Fresh DB, TimeTracker reachable | Empty DB, valid API keys | Users/Employees created for every returned identity; history rows seeded via extension | N/A |
| Already-seeded DB, unchanged data | Seed run a second time, TimeTracker data unchanged | No duplicate rows; command exits 0 | Match existing rows by unique `email` before insert |
| Already-seeded DB, upstream data changed | Seed run again; an existing email's name/countryCode/hash changed upstream | Existing row updated to match the latest fetch | Update-in-place by `email`, not a new insert |
| TimeTracker unreachable (either endpoint) | Accounting or Talents down, or key rejected (401/403) | Seed fails loudly, exits non-zero | Nothing has been written yet (fetch-before-write, see Always); error logged with endpoint + status code |
| Delivered list smaller than 500 | e.g. 300 identities returned | Command halts before writing, per Ask-First | Thrown error naming the shortfall, not a silent pad |
| Delivered list larger than 2,000 | e.g. 5,000 identities returned | Command halts before writing, per Ask-First | Thrown error naming the count, asking for confirmation |
| Empty population | Accounting returns `employees: []` | Same as "smaller than 500" — halts before writing, per Ask-First | Thrown error naming the shortfall (zero) |
| Malformed TimeTracker response | A record is missing a field the OpenAPI contract marks required | Seed fails loudly, exits non-zero | Treated identically to TimeTracker-unreachable — no writes have happened yet |
| Talents membership with no matching identity | A project member's email isn't in the Accounting response | That membership record is skipped; seeding continues | Logged warning naming the orphaned email, not a run failure |

</frozen-after-approval>

## Shipped state — agent reference (2026-08-28 pivot)

**Agents: read `_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md` first.** The frozen Intent/Boundaries/I/O matrix above describes the *original* TimeTracker Accounting design. **Shipped code differs** as follows.

### Current behavior

1. **Identity source:** bundled manifest `services/backend/src/prisma/seed/data/bootcamp-identities.json` (24 accounts), derived from `docs/bootcamp-seed-accounts-source.csv`. Archive mirror: `_bmad-output/archive/bootcamp-seed-identities.json`.
2. **No TimeTracker at seed time** — no VPN, no API keys. `TimetrackerService` exists for Epic 13 only.
3. **Population guard:** halt only on **empty** manifest after dedupe (`EmptySeedPopulationError`). The 500/2000 floor/ceiling is **removed**.
4. **`hash`:** SHA256 of `bootcamp-seed-v1:` + lowercased email (uppercase hex) — source CSV has no hash column.
5. **Synthetic layer:** unchanged — four history dimensions via `seed.synthetic.ts` + temporal-history extension.
6. **Idempotent upsert by email** — unchanged.

### Amended acceptance (supersedes frozen AC where they conflict)

- Given an empty database, when `npm run db:seed` runs, then **24** `User`/`Employee` pairs exist matching the manifest.
- Given a missing or empty manifest, when seed runs, then it exits non-zero before any write.
- Given a database already seeded, when seed runs again, then no duplicate rows and exit 0.
- ~~TimeTracker unreachable blocks seed~~ — **N/A** (manifest-only).
- ~~500 floor / Talents orphan warnings~~ — **N/A** (TT not called from seed).

### Amended verification

- `npm run db:seed` → exit 0, log `Seed complete: 24 identities upserted`
- `npm run test` → seed unit tests pass (manifest fixture, not TT mocks)

## Code Map

- `services/backend/src/prisma/seed/data/bootcamp-identities.json` -- **runtime seed manifest** (24 identities); keep in sync with archive + source CSV.
- `docs/bootcamp-seed-accounts-source.csv` -- delivered account list (workspace); semicolon-separated source of truth for manifest regeneration.
- `_bmad-output/archive/bootcamp-seed-identities.json` -- archive mirror of runtime manifest.
- `_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md` -- **agent reference** when frozen spec / PRD §4.17 conflict with shipped seed.
- `docs/api-external-openapi.json` -- TimeTracker external API contract (Epic 13; **not** seed source post-pivot).
- `docs/project-requirements-v2.md:506-535` (§4.17, §5.1) -- source of the seeded-population requirement; project-assignment sync is a *separate* concern (Epic 13), not this story's.
- `_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md:264` -- Design Notes row already names `prisma/seed.ts` and confirms the seeded population *is* the non-prod data (no separate real-population leak risk).
- `services/backend/prisma/schema.prisma:12-42` -- `User` (email/name) and `Employee` (1:1, cascade) models; seed writes here. **Neither model has columns for `hash`/`countryCode` today — a migration adding them to `User` (alongside the existing `email`/`name`) is required before seed.ts can store these verbatim, per Acceptance Criteria.**
- `services/backend/prisma/schema.prisma:67-125` -- four temporal-history tables; extension-mediated `create` only.
- `services/backend/prisma.config.ts` -- Prisma 7's CLI config (datasource url, migrations path). This is where the seed command must be registered (`migrations.seed`) — **not** `package.json`'s legacy `"prisma": {"seed": ...}` key, which Prisma 7 does not read from this location.
- `services/backend/src/prisma/extensions/temporal-history.extension.ts:1-58` -- only legal write path for history rows; every other op throws.
- `services/backend/src/modules/contracts/external-identity-mapping.contract.ts` (C5) -- DI token for `(system, externalId) -> employeeId` mapping; the new TimeTracker importer should populate this instead of matching on email at read time.
- `services/backend/src/modules/contracts/contracts.module.ts:10-11,37-39,49` -- current stub binding for `ExternalIdentityMapping`; **deferred (review 2026-08-28, decision 1B):** C5 population with TimeTracker `id` is Epic 13 / follow-up scope — this story seeds identity on `User` by email only.
- `services/backend/.dependency-cruiser.cjs:17-32,48-61` -- new code must live under its own `src/modules/<name>/`, depending only on `contracts`/`registry`.
- `services/backend/src/config/env.validation.ts:1-10` -- add the two new required env vars here.
- `services/backend/.env.example` -- document the two new keys.
- `services/backend/package.json` -- `scripts.postbuild` currently `prisma migrate deploy`; no `db:seed`/`prisma.seed` exists yet.
- `services/backend/README.md` -- Quick Start section to extend with the seed step.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/config/env.validation.ts` -- add `TIMETRACKER_ACCOUNTING_API_KEY` / `TIMETRACKER_TALENTS_API_KEY` -- validated the same way as `DATABASE_URL` (amended to `.optional()` — see Spec Change Log)
- [x] `services/backend/.env.example` -- document the two new keys -- keeps local bootstrap self-describing (amended to commented-out — see Spec Change Log)
- [x] `services/backend/prisma/schema.prisma` -- add `hash` / `countryCode` (`String`) columns to `User` + migration -- required to store these TimeTracker-sourced fields verbatim (Acceptance Criteria); must land before seed.ts writes them
- [x] `services/backend/src/modules/timetracker/` (new module) -- HTTP client calling `POST /api/accounting/report` and `GET /api/projects/talents` with the correct per-endpoint API key header, plus typed response interfaces for `Employee`/`WorkingDay`/`ProjectTalentDto` and the enum lookups (per `docs/api-external-openapi.json`) rather than consuming raw `any` -- first external-HTTP integration in the codebase; needs its own module home
- [x] `services/backend/prisma/seed.ts` (new) -- bootstrap via `NestFactory.createApplicationContext`, pull identities through the new TimeTracker module, upsert `User`/`Employee` idempotently by email, write initial history rows through the real `PrismaService` extension -- single entrypoint reused by both triggers
- [x] `services/backend/prisma.config.ts` -- register the seed command under `migrations.seed` (`"nest build && node dist/prisma/seed.js"`, not literal `ts-node` — see Design Notes) -- Prisma 7 reads seed config from here, not from a `package.json` `"prisma"` key
- [x] `services/backend/package.json` -- add `"db:seed"`, append `&& npm run db:seed` to `postbuild` -- `db:seed` runs the compiled seed directly rather than via `prisma db seed` (see Spec Change Log); `postbuild` covers the auto-deploy path without needing Story 1.17's topology
- [x] `services/backend/README.md` -- document new env vars and the seed step
- [x] `services/backend/prisma/seed.spec.ts` (or module-local test) -- cover every row of the I/O & Edge-Case Matrix: idempotent rerun (unchanged + drifted data), either-endpoint-unreachable, short-list and over-2,000 HALTs, empty population, malformed response, in-fetch email dedup, and orphaned Talents membership

**Acceptance Criteria:**
- Given an empty database and valid TimeTracker credentials, when `npm run db:seed` runs, then a `User`/`Employee` pair exists for every identity the Accounting endpoint returns, with email/name/countryCode/hash stored verbatim.
- Given a database already seeded, when the seed command runs again, then no duplicate rows are created and the command exits 0.
- Given either TimeTracker endpoint is unreachable or rejects the key, when the seed command runs, then it exits non-zero and no rows are written (checked before any write begins).
- Given a database already seeded, when TimeTracker later returns a changed name/countryCode/hash for an existing email, then the existing row is updated to match, not left stale.
- Given a Talents membership record whose email has no matching Accounting-endpoint identity, when the seed command runs, then that membership is skipped with a logged warning and the run still succeeds.
- Given `npm run build` runs, when `postbuild` executes, then migrations and seeding both apply automatically, matching what `npm run db:migrate` triggers locally.

### Review Findings

**2026-08-28 — second review pass (four layers: blind-hunter, edge-case-hunter, verification-gap, acceptance-auditor)**

1. **`decision-needed`** — resolved 2026-08-28:
   - [x] [Review][Decision] Populate `ExternalIdentityMapping` (C5) — **1B:** defer to Epic 13 / follow-up substrate story; amend Code Map (see defer below).
   - [x] [Review][Decision] Expose `hash` on public Users API — **2C:** persist in DB but exclude from DTO/entity API serialization (added to patch list below).

2. **`patch`** — all applied 2026-08-28:
   - [x] [Review][Patch] Exclude `hash` from Users API responses (persist in DB only; remove from `UserEntity` Swagger/serialization) [`user.entity.ts:14`]
   - [x] [Review][Patch] Validate Accounting response envelope before accessing `employees[]` [`seed.service.ts:54-55`, `seed.helpers.ts:41`]
   - [x] [Review][Patch] Validate Talents response required top-level `statuses`/`types` arrays [`seed.service.ts:56`, `seed.helpers.ts:70`]
   - [x] [Review][Patch] Guard against 2xx JSON body parsing to null/non-object [`timetracker.service.ts:110`]
   - [x] [Review][Patch] Add SeedService test: Talents endpoint unreachable → no writes [`seed.service.spec.ts`]
   - [x] [Review][Patch] Add SeedService test: malformed Talents response → no writes [`seed.service.spec.ts`]
   - [x] [Review][Patch] Assert `fetchAccountingReport` receives correct `{ month, year }` from `mostRecentCompleteMonth` [`seed.service.spec.ts:46`]
   - [x] [Review][Patch] Assert all four history dimensions seeded on fresh DB and skipped on rerun [`seed.service.spec.ts:156-188`]
   - [x] [Review][Patch] Add test: identities absent from latest TimeTracker response are left untouched [`seed.service.spec.ts`]
   - [x] [Review][Patch] Fix outdated comment: `prisma/seed.ts` says `db:seed` runs via `prisma db seed` [`prisma/seed.ts:3`]
   - [x] [Review][Patch] Reconcile README Quick Start (fill in keys) vs optional-keys guidance [`README.md:27`]
   - [x] [Review][Patch] Include TimeTracker error response body in `TimetrackerApiError` for non-2xx [`timetracker.service.ts:99`]

3. **`defer`** (checked, pre-existing or accepted by design):
   - [x] [Review][Defer] `ExternalIdentityMapping` (C5) not populated by seed — deferred to Epic 13 / follow-up; Code Map amended (review decision 1B) [`contracts.module.ts:36`]
   - [x] [Review][Defer] `TimelineEventWriterStub` no-ops — history rows populate but `timeline_events` stays empty until Epic 7 [`contracts.module.ts:35`] — deferred, pre-existing Wave-0 stub
   - [x] [Review][Defer] Compiled seed entrypoint / `postbuild` chain not exercised in CI — same root cause as Story 1.19 CI gap (no test job) [`package.json:17,20`] — deferred, pre-existing
   - [x] [Review][Defer] `postbuild` unconditionally seeds with no environment gate — Design Notes defer gating to Story 1.17 [`package.json:20`] — deferred, accepted for single-environment scope

## Spec Change Log

- **2026-08-28, bootcamp pivot (post-merge):** test task scope reduced from 500+ TimeTracker Accounting population to a **fixed 24-account manifest**. Seed now reads `src/prisma/seed/data/bootcamp-identities.json` (archive mirror: `_bmad-output/archive/bootcamp-seed-identities.json`); removed 500/2000 `PopulationSizeError` floor/ceiling — only empty manifest halts. TimeTracker keys no longer required for seed (Epic 13 sync only). `TimetrackerService` retained in codebase.
- **2026-08-28, found during step-03 verification (not step-04 review):**** the implementer made both TimeTracker API keys `.required()` in `env.validation.ts`, following this repo's own `nest-config.md` convention ("`.required()` when no sane default exists") literally. Verified this breaks the documented `cp .env.example .env` Quick Start and the pre-existing `temporal-history.extension.spec.ts` suite — both boot via the same global `envValidationSchema`, so *any* app bootstrap (not just seeding) now hard-fails without TimeTracker credentials, since `.env.example` shipped the keys empty. Amended: keys are now `.optional()` in Joi; `TimetrackerService` already enforces presence at point-of-use via `ConfigService.getOrThrow` (per this spec's own "Always" bullet), which is the correct place for a seed-only credential to fail loudly. `.env.example` amended to comment the two keys out entirely (an optional-but-present-empty string still fails `Joi.string()` validation). **KEEP:** the `getOrThrow`-at-point-of-use pattern in `timetracker.service.ts` was already correct and needed no change.
- **2026-08-28, found during step-03 verification:** confirmed live (not hypothetical) that Prisma 7's `prisma db seed` CLI always exits 0 and prints "The seed command has been executed" regardless of the underlying seed command's real exit code — reproduced directly against the real TimeTracker dev host (unreachable from this network), where the seed script itself correctly threw and exited 1, but `prisma db seed`/`npm run db:seed` (as originally wired) reported exit 0. This would have silently defeated the AC "unreachable TimeTracker → exits non-zero" for every real `postbuild` run. Amended: `package.json`'s `db:seed` now runs `nest build && node dist/prisma/seed.js` directly, bypassing the wrapper; `postbuild`'s exit code was re-verified end-to-end (`npm run build` → exit 1 on unreachable TimeTracker, confirmed via real Postgres with zero rows written). `prisma.config.ts`'s `migrations.seed` keeps the old (still-swallowing) path solely for `prisma migrate dev`'s interactive local convenience, documented as an accepted, human-supervised gap.
- **2026-08-28, step-04 review (second pass):** four review layers re-ran after first patch round. Decisions: (1B) defer `ExternalIdentityMapping` (C5) population to Epic 13; (2C) persist `hash` in DB but exclude from Users API responses. Twelve patch findings applied: envelope validation for Accounting/Talents responses, null/non-object JSON guard, error response bodies on non-2xx, README/seed.ts comment fixes, and expanded `seed.service.spec.ts` coverage (Talents failure/malformed, month/year wiring, all four history dimensions, absent-identity boundary). Three items deferred (TimelineEvent stub, CI/postbuild smoke, postbuild env gating). Branch: `feature/1-16-seed-data-review-patches` in `services/backend`.

- **2026-08-28, step-04 review (patch round, `review_loop_iteration` not incremented — patches don't loop back):** three review layers (Blind Hunter, Edge Case Hunter, Verification Gap) ran against the diff. Six real, verified findings were patched: (1) `TimetrackerService`'s `.json()` parse wasn't error-wrapped — a malformed 2xx body threw a raw `SyntaxError` instead of `TimetrackerApiError`; added a `parseJson` helper. (2) No fetch timeout — a stalled TimeTracker connection could hang the seed (and `postbuild`) indefinitely; added `AbortSignal.timeout(15000)`. (3)+(4) Email matching was case-sensitive in both `dedupeEmployeesByEmail` and the orphaned-Talents-membership check, risking false duplicates/false orphans on case-variant emails; added `normalizeEmailKey` for both comparisons while keeping the original-cased email for the actual DB write (residual, accepted gap: cross-*run* case drift for the same person isn't caught, since the `User.email` unique constraint is case-sensitive at the DB level and dedup only operates within a single fetch — judged acceptably unlikely given TimeTracker is a single stable source). (5) `buildSyntheticProfile`'s four dimensions were derived from the same hash via small bit-shifts, correlating them per employee; salted each dimension's hash independently. (6) Comments across several files cited `docs/api-external-openapi.json` as if it lived inside this repo — it only existed at the monorepo workspace root, unreachable from a standalone clone of this submodule; vendored a copy into `services/backend/docs/api-external-openapi.json`. Two README clarifications were bundled in (documented the pre-existing `contracts/`/`registry/` dirs already missing from the Structure tree; explained what happens if TimeTracker keys are skipped). **Deferred** (pre-existing, not caused by this story): CI runs no tests at all (already tracked from Story 1.19); `npm run start:prod` is broken (`dist/main.js` doesn't exist, real entrypoint is `dist/src/main.js`) — logged to `deferred-work.md`. **Rejected** as false positives or explicitly out of scope: no-transaction write loop (self-healing via idempotent upserts), a concurrent-seed-run race (no realistic trigger in intended usage), no retry/backoff (spec explicitly wants fail-fast, not retry), `seed.service.ts` importing from `timetracker` module (no automated boundary violated, `timetracker` is quasi-infra by design), `TimetrackerModule` in the full `AppModule` (zero-cost constructor), "offboarded employees aren't removed by seed" (this is literally the frozen spec's own **Never** bullet — reviewers lacked spec visibility), hand-written migration SQL (verified correct against real Postgres), and the baked-in dev TimeTracker URL (single-environment scope, already covered by Design Notes). **KEEP:** the fetch-before-write ordering, the `getOrThrow`-at-point-of-use pattern, and the direct-compiled-seed invocation (bypassing `prisma db seed`) all held up under three independent adversarial reviews with no findings against them — do not "simplify" any of these three in a future pass.

## Design Notes

- **Reuse real DI, don't hand-roll a second Prisma client.** `prisma/seed.ts` should call `NestFactory.createApplicationContext(AppModule)` so it gets the extended `PrismaService` (temporal-history guarantees intact) and the new TimeTracker client via normal DI, rather than bypassing the extension.
- **Two API keys, not one.** AD-12 names a single `TIMETRACKER_API_KEY`; the OpenAPI contract defines two independent, non-interchangeable keys (`AccountingApiKey`, `TalentsApiKey` — wrong one gets a 403). This spec follows the OpenAPI contract; reconciling AD-12's naming is a follow-up, not a blocker here.
- **500+ list performance ≠ seed size (post-pivot).** The PRD §7 NFR targets All Employees perf at 500+ records (Story 3.7). The seed manifest is **24 accounts** for bootcamp dev/demo — do not pad seed to 500 or re-block on TimeTracker Accounting count.
- **Migrate-then-seed ordering is already guaranteed.** `postbuild` chains `prisma migrate deploy && npm run db:seed` with shell `&&` — seed only ever runs after migrate exits 0, so a schema-changing deploy can't race the seed against a stale shape. No extra coordination needed.
- **`prisma/seed.ts` is intentionally outside `.dependency-cruiser.cjs`'s module-boundary rule.** That rule only constrains `src/modules/**`; the seed script lives under `prisma/` and imports `PrismaService` directly (no TimeTracker dependency post-pivot).
- **TimeTracker keys are optional at bootstrap, Epic-13-only.** `TIMETRACKER_*` keys are `.optional()` in `env.validation.ts`; `TimetrackerService` uses `getOrThrow` only when Epic 13 sync code invokes it — not during seed.
- **The single-environment assumption is temporary.** "No separate real-population leak risk" (Code Map, ARCHITECTURE-SPINE) holds only while local dev and the one auto-deployed environment are the only targets `postbuild` ever runs against. Revisit gating the seed step behind an explicit environment check when Story 1.17 introduces real deployment topology — not required now, since there is nothing yet to gate against.

## Verification

**Commands:**
- `npm run db:seed` (services/backend, local docker-compose Postgres) -- expected: exits 0, **24** identities upserted (see manifest)
- `npm run db:seed` run twice -- expected: second run exits 0, no new rows
- `node dist/prisma/seed.js` run directly -- expected: loads manifest from `dist/prisma/seed/data/` or `dist/src/prisma/seed/data/` fallback paths
- `npm run test` -- expected: seed manifest tests pass alongside the existing suite
- `npm run depcruise` -- expected: `timetracker` module passes the module-boundary rule

## Suggested Review Order

> **Post-pivot note:** Steps referencing TimeTracker fetch, 500/2000 guards, and TT-specific tests describe the **original** design. For current code, start with **Shipped state** above and `bootcamp-scope-overrides.md`.

**Orchestration — entry point**

- Thin CLI entrypoint: bootstraps DI, hands off to `SeedService`, exit-codes on failure.
  [`seed.ts:14`](../../services/backend/prisma/seed.ts#L14)

- The whole run in one place: fetch → validate → dedupe → halt-check → write, in that order.
  [`seed.service.ts:40`](../../services/backend/src/prisma/seed/seed.service.ts#L40)

**Fetch-before-write safety (the story's core guarantee)**

- Nothing is written until both responses are fully validated — the guard the AC depends on.
  [`seed.service.ts:54`](../../services/backend/src/prisma/seed/seed.service.ts#L54)

- Required-field validation per OpenAPI contract; malformed data fails loud before any write.
  [`seed.helpers.ts:41`](../../services/backend/src/prisma/seed/seed.helpers.ts#L41)

- 500/2000 Ask-First floor and ceiling — halts rather than silently padding or truncating.
  [`seed.helpers.ts:148`](../../services/backend/src/prisma/seed/seed.helpers.ts#L148)

- Case-insensitive email dedup/matching (review-round fix) — preserves original casing for the DB write.
  [`seed.helpers.ts:118`](../../services/backend/src/prisma/seed/seed.helpers.ts#L118)

**TimeTracker HTTP client**

- Shared fetch + timeout + error-mapping: unreachable, timeout, and non-2xx all become one error type.
  [`timetracker.service.ts:85`](../../services/backend/src/modules/timetracker/timetracker.service.ts#L85)

- Malformed-JSON-body parsing now maps to the same error type (review-round fix).
  [`timetracker.service.ts:106`](../../services/backend/src/modules/timetracker/timetracker.service.ts#L106)

- Names the endpoint (and status, when known) so failure logs are actionable without reconstruction.
  [`timetracker.errors.ts:1`](../../services/backend/src/modules/timetracker/timetracker.errors.ts#L1)

**Idempotent write phase**

- Upsert-by-email keeps TimeTracker fields in sync on rerun; never a new insert for a known email.
  [`seed.service.ts:110`](../../services/backend/src/prisma/seed/seed.service.ts#L110)

- Guards against re-creating history rows on rerun — the extension's `create` always closes-and-inserts.
  [`seed.service.ts:153`](../../services/backend/src/prisma/seed/seed.service.ts#L153)

- Deterministic per-employee synthetic profile, independently salted per dimension (review-round fix).
  [`seed.synthetic.ts:58`](../../services/backend/src/prisma/seed/seed.synthetic.ts#L58)

**Deployment wiring — the "works everywhere" mechanism**

- Real seed command bypasses `prisma db seed`'s exit-code swallowing (step-03 finding — see Spec Change Log).
  [`package.json:17`](../../services/backend/package.json#L17)

- `postbuild` chains migrate → seed with `&&`, so a failed seed fails the whole build/deploy.
  [`package.json:20`](../../services/backend/package.json#L20)

- `migrations.seed` kept only for `prisma migrate dev`'s local interactive convenience — documented divergence from `db:seed`.
  [`prisma.config.ts:40`](../../services/backend/prisma.config.ts#L40)

- Keys are `.optional()`, not `.required()` — a step-03 fix so app bootstrap doesn't need TimeTracker credentials.
  [`env.validation.ts:22`](../../services/backend/src/config/env.validation.ts#L22)

- `hash`/`countryCode` columns added to `User` — the two remaining TimeTracker-sourced identity fields.
  [`schema.prisma:18`](../../services/backend/prisma/schema.prisma#L18)

- `TimetrackerModule` registered so `createApplicationContext` can resolve it via normal DI.
  [`app.module.ts:25`](../../services/backend/src/app.module.ts#L25)

**Peripherals**

- Vendored contract copy (review-round fix) so a standalone submodule clone can resolve the comment references.
  [`api-external-openapi.json`](../../services/backend/docs/api-external-openapi.json)

- Hand-written migration SQL (no live shadow DB at authoring time) — verified applied cleanly against real Postgres.
  [`migration.sql:1`](../../services/backend/prisma/migrations/20260828090000_add_user_hash_country_code/migration.sql#L1)

- Setup instructions and skip-TimeTracker guidance for first-time contributors.
  [`README.md:37`](../../services/backend/README.md#L37)

- Full I/O-matrix coverage, including the two review-round regression tests for case-insensitive matching.
  [`seed.service.spec.ts:1`](../../services/backend/src/prisma/seed/__tests__/seed.service.spec.ts#L1)

---
title: 'Cache Access Resolution Safely and Revoke Immediately on Project-Assignment End'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 1
baseline_commit: '66ed5faa008abd9f408e4db315555b969674bdad'
story_key: '1-13-cache-access-resolution-safely-and-revoke-immediately-on-pro'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-3-assign-people-partner-relationships.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/decisions.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `AccessResolverService` (C1) recomputes every `(viewerId, subjectId)` pair from live graph walks on every call — correct for NFR-8 today, but too slow for All Employees at scale (NFR-2). There is no generation-gated cache (D1/AD-4) and no central invalidation when relationship data changes; a future ad-hoc cache would risk serving stale `ProjectLine` grants after an assignment ends.

**Approach:** Add a durable monotonic `AccessGraphGeneration` counter plus an in-process, generation-tagged resolution cache inside `access`. Wire synchronous `bump()` on every relationship-graph write that exists today (`Employee.managerId`/`peoplePartnerId`, all `ProjectAssignment` mutations including `confirmed`/`confirmedAt` flips, `DepartmentHistory` appends that affect the PP HR-line walk, `FullAccessGrant` grant/revoke). `resolveAudience` consults the cache when enabled; correctness comes from the generation stamp **plus** revalidation of clock-driven predicates, with optional TTL as a performance knob only. Formal 500+-record / 2s proof stays Story 3.7 — this story delivers the mechanism 3.7 exercises.

## Boundaries & Constraints

**Always:**
- **D1/AD-4 shape only:** in-process cache keyed by `` `${viewerId}:${subjectId}` `` (pair-scoped — the correct shape for C1; D1's "per-subject" wording is the storage family, not a single-subject key). Each entry stores `{ generation, audience, computedAt }`. On `resolveAudience`, read current generation once; return cached audience only when `entry.generation === currentGeneration` **and** clock-driven predicates still hold (see next bullet). Optional `ACCESS_RESOLUTION_CACHE_TTL_MS` may evict entries early — never the sole invalidation mechanism.
- **Time-sensitive revalidation (normative):** `ProjectLine` active/fresh checks and PP HR-line `DepartmentHistory` reads depend on injectable `Clock`, not only graph writes. On every cache hit, re-run the time-sensitive predicates from spec-1-2 (`isProjectAssignmentActive`, 4h freshness on `confirmedAt`) and spec-1-3 (open `DepartmentHistory` HR match on the PP walk) against current `Clock`; if the result would differ from the cached audience, discard the entry and recompute. A generation match alone is insufficient when time passes without a bump.
- **Generation storage:** singleton `AccessGraphGeneration` row (`id = 'default'`, `generation BigInt`, seeded to `0` in migration). `RelationshipGraphGenerationService` holds an in-memory `currentGeneration` mirror (loaded at module init, updated synchronously in `bump()`); hot-path `getGeneration()` returns the mirror without a per-request DB read. `bump()` atomically increments in DB (`UPDATE … RETURNING`), updates the mirror, and clears the in-process cache map synchronously in the same call — no async race where a stale entry survives a bump.
- **Bump triggers (normative):** any create/update/delete touching `Employee.managerId`, `Employee.peoplePartnerId`, any `ProjectAssignment` column (including `confirmed`/`confirmedAt` on an existing row per AD-8), any `DepartmentHistory` `create` (Story 1.20 append-only writes — affects PP HR-line resolution per spec-1-3), or any `FullAccessGrant` row. Implement via a Prisma Client Extension on those models delegating to `RelationshipGraphGenerationService.bump()` after successful writes — including `createMany`/`updateMany`/`deleteMany`/`createManyAndReturn`/`updateManyAndReturn` on covered models (Story 1.20 precedent: bulk ops must not bypass hooks). For `Employee`, bump only when the mutation touches `managerId` or `peoplePartnerId` (other profile-field edits must not flush the cache). Seed scripts, tests, and future C9/C11 writers route through the extended client so they cannot miss invalidation.
- **Project-assignment lifecycle writes:** extend `ProjectAssignmentService` with `update(id, patch)` (at minimum `endDate`, `confirmed`, `confirmedAt`) — all paths call through Prisma so the extension bumps. `create` already exists; no bare `prisma.projectAssignment.update` in feature code.
- **Config:** `ACCESS_RESOLUTION_CACHE_ENABLED` (boolean, default `false` — AD-4/epic-1: no cache by default; enable explicitly for perf work and Story 3.7) and `ACCESS_RESOLUTION_CACHE_TTL_MS` (optional positive int, default unset = no TTL eviction). Add to `.env.example` + `env.validation.ts`.
- **Self fast-path unchanged:** `viewerId === subjectId` still returns `Self` without touching cache (trivial, always fresh).
- **Revocation semantics unchanged:** ending an assignment (`endDate` in the past, inclusive boundary per spec-1-2) still denies `ProjectLine` on the next `resolveAudience` — cache must not extend access beyond the graph state the resolver would compute live.

**Ask First:** none identified.

**Never:**
- Client-side or session-stable permission caching; Redis/multi-instance shared cache (single-container deploy, D10).
- TTL-only invalidation without generation bump.
- Local Prisma reads bypassing bump hooks for graph fields (violates AD-4).
- `Department` entity / department-management graph bumps (AD-14/C12 not landed) — document extension hook point only; `DepartmentHistory` appends **are** in scope today.
- C9 `OrgRelationshipWriter`, departure cascade (C11), or timetracker sync writer changes beyond calling the same bump path.
- NFR-2 load fixture, 500+-row benchmark, or CI perf regression gate (Story 3.7).
- Frontend changes.

## I/O & Edge-Case Matrix

Canonical source for cache behavior; aligns with spec-1-2 time predicates and spec-1-3 PP HR-line rules.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Cache hit | Cache enabled; same gen; repeat `resolveAudience(P,B)`; clock predicates unchanged | Second call returns identical audience without re-querying graph tables | N/A |
| Gen bump invalidates | Warm cache for `(P,B)`; `ProjectAssignment.endDate` set to past | Next `resolveAudience(P,B)` misses cache; `ProjectLine` denied | N/A |
| Confirmed flip bumps | Warm cache; `confirmed: true→false` on existing row (no delete) | Next resolve misses cache; `ProjectLine` denied | N/A |
| Freshness expires (no write) | Warm cache with `ProjectLine`; `confirmedAt` crosses 4h boundary; gen unchanged | Cache hit fails time-sensitive revalidation; recompute denies `ProjectLine` | N/A |
| Start date arrives (no write) | Warm cache denied `ProjectLine` while `startDate` in future; clock advances past `startDate` | Revalidation forces recompute; `ProjectLine` granted if other predicates pass | N/A |
| DepartmentHistory bumps | Warm cache for PP HR-line viewer; new `DepartmentHistory` row closes prior open dept on PP chain | Next resolve misses cache (gen bumped); PP grant reflects new HR membership | N/A |
| Manager change bumps | Warm cache; `Employee.managerId` updated for subject | Next resolve misses cache; reflects new reporting line | N/A |
| PP change bumps | Warm cache; `peoplePartnerId` updated | Next resolve misses cache; reflects new PP grant | N/A |
| Full-access grant bumps | Warm cache for viewer with new `FullAccessGrant` | Next resolve misses cache; `FullAccess` granted | N/A |
| Non-graph Employee edit | Warm cache; unrelated `Employee` field updated (e.g. display name) | Generation unchanged; cache hit still valid if clock predicates pass | N/A |
| Bulk Prisma write | `updateMany` on `ProjectAssignment` changes `confirmed` | Extension bumps once; cache cleared | N/A |
| Cache disabled | `ACCESS_RESOLUTION_CACHE_ENABLED=false` | Every call computes live; bump still increments gen (harmless) | N/A |
| TTL eviction | TTL set very low; gen unchanged | Entry may evict; next call recomputes same audience | N/A |
| Assignment end inclusive | `endDate === Clock.now()` | Still active per spec-1-2; cache may store `ProjectLine` until end boundary passes | N/A |
| Concurrent bump | Two writes bump in parallel | Generation strictly increases; no stale entry served after either bump completes | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — add `AccessGraphGeneration` singleton model; `FullAccessGrant` already exists from Story 1.12
- `services/backend/src/modules/access/relationship-graph-generation.service.ts` (new) — `getGeneration()`, `bump()`, in-process generation mirror + `Map` cache
- `services/backend/src/prisma/relationship-graph.extension.ts` (new) — Prisma `$extends` on `Employee`/`ProjectAssignment`/`DepartmentHistory`/`FullAccessGrant` writes (including bulk ops) → field-scoped `bump()` on `Employee`
- `services/backend/src/prisma/prisma.service.ts` — apply extension to client (mirror temporal-history extension pattern from Story 1.20; compose both extensions on one client)
- `services/backend/src/modules/access/access-resolver.service.ts:189-236` — wrap `resolveAudience` with generation read + cache get/set + clock revalidation; update class doc ("Never cached" → generation-gated)
- `services/backend/src/modules/access/project-assignment.service.ts:52-68` — add `update(id, patch)` for `endDate`/`confirmed`/`confirmedAt`; ensure feature code uses service not raw Prisma
- `services/backend/src/modules/access/access.module.ts` — register `RelationshipGraphGenerationService`; export if tests need direct bump
- `services/backend/src/config/env.validation.ts` — new cache env keys
- `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` — cache hit/miss, clock revalidation (freshness expiry, startDate arrival), warm-cache revocation rows
- `services/backend/src/modules/access/__tests__/relationship-graph-generation.service.spec.ts` (new) — bump atomicity, mirror sync, cache clear on bump
- `services/backend/src/modules/access/__tests__/project-assignment.service.spec.ts` — `update` bumps generation (mock gen service)
- `services/backend/src/prisma/__tests__/relationship-graph.extension.spec.ts` (new) — bulk-op bump coverage; Employee non-graph update does not bump
- `services/backend/test/access-resolution-cache.e2e-spec.ts` (new) — AC1 path: seed assignment → enable cache via env → warm cache → end assignment → denied on next HTTP profile/list call

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — `AccessGraphGeneration` singleton seeded at `0` — durable generation counter
- [x] `services/backend/src/modules/access/relationship-graph-generation.service.ts` — generation read/bump + in-process mirror + cache API — D1 core
- [x] `services/backend/src/prisma/relationship-graph.extension.ts` + `prisma.service.ts` — auto-bump on graph writes (incl. bulk ops, field-scoped Employee) — AD-4 invalidation coverage
- [x] `services/backend/src/modules/access/access-resolver.service.ts` — generation-gated cache with clock revalidation in `resolveAudience` — consumer integration
- [x] `services/backend/src/modules/access/project-assignment.service.ts` — `update` for lifecycle/confirmation patches — AC1 write path
- [x] `services/backend/src/config/env.validation.ts` + `.env.example` — cache feature flags (`ENABLED` default `false`)
- [x] `services/backend/src/modules/access/access.module.ts` — wire new service
- [x] `services/backend/src/modules/access/__tests__/relationship-graph-generation.service.spec.ts` + extend `access-resolver.service.spec.ts` + `relationship-graph.extension.spec.ts` — matrix unit coverage
- [x] `services/backend/test/access-resolution-cache.e2e-spec.ts` — warm-cache revocation integration test (set `ACCESS_RESOLUTION_CACHE_ENABLED=true` in test env; skip with documented reason if Node 22 e2e blocker reproduces — unit tests remain merge gate)

**Acceptance Criteria:**
- Given PM P holds `ProjectLine` access to employee B solely via B's active confirmed assignment, the resolution cache is enabled, and the cache is warm for `(P,B)`, when B's assignment `endDate` is set to the past and P immediately requests B's profile (or any endpoint that calls C1 for that pair), then P is denied `ProjectLine`-level access with no stale cached grant served
- Given the generation-gated cache is enabled and relationship-graph writes bump the counter synchronously, when any write changes `managerId`, `peoplePartnerId`, `ProjectAssignment` fields (including `confirmed`), `DepartmentHistory` (PP HR-line), or `FullAccessGrant` state, then the next `resolveAudience` for affected pairs recomputes from live data
- Given a warm cache entry whose generation still matches but a clock-driven predicate from spec-1-2 or spec-1-3 would now yield a different audience (freshness expiry, `startDate` arrival, inclusive `endDate` boundary), when `resolveAudience` is called, then the entry is discarded and the audience is recomputed from live data without requiring a generation bump

## Design Notes

**Why Prisma extension vs. scattered `bump()` calls:** AD-4 requires invalidation on *any* graph write, including seed and tests that touch Prisma directly today. A write extension centralizes the rule; service-layer `bump()` on `ProjectAssignmentService` alone would miss `Employee` FK updates until C9 lands.

**Generation vs. TTL vs. Clock:** D1 separates correctness (generation + clock revalidation) from performance (TTL). Tests must prove revocation with generation and clock predicates — TTL tests are optional sanity, not the safety proof.

**D1 key shape:** C1 resolves a `(viewer, subject)` pair; the cache key must be pair-scoped. Storing only `subjectId` would serve the wrong audience across viewers.

**Intra-call `reportingLineCache` Map** inside `resolveProjectLine` (existing) is request-scoped dedup, not cross-request cache — leave it untouched.

**C12 hook point:** When `Department` entity writes land, add them to the same extension — same `bump()` path; do not invent a second invalidation mechanism.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run depcruise` — expected: clean
- `cd services/backend && npm test -- --testPathPatterns="access-resolver|relationship-graph|project-assignment"` — expected: all new/extended unit tests pass (includes clock revalidation and bulk-op bump cases)
- `cd services/backend && ACCESS_RESOLUTION_CACHE_ENABLED=true npm run test:e2e -- access-resolution-cache` — NOT RUN: Postgres unavailable locally (`P1001`); run on CI or with `npm run db:up` before merge

### Review Findings

- [x] [Review][Adversarial] Clock-driven `ProjectLine`/PP predicates can go stale without a generation bump — Fixed: time-sensitive revalidation on cache hit; `computedAt` on entries; matrix rows for freshness expiry and `startDate` arrival.
- [x] [Review][Adversarial] `DepartmentHistory` appends affect PP HR-line but were not bump triggers — Fixed: added to bump list, extension models, matrix row, and AC2.
- [x] [Review][Adversarial] `ACCESS_RESOLUTION_CACHE_ENABLED` default `true` contradicts AD-4/epic-1 "no cache by default" — Fixed: default `false`; e2e enables explicitly.
- [x] [Review][Adversarial] Bulk Prisma ops (`updateMany`, etc.) not covered — Fixed: normative bulk-op enumeration; extension spec test file.
- [x] [Review][Adversarial] Employee extension scope ambiguous (all writes vs graph fields) — Fixed: bump only on `managerId`/`peoplePartnerId` mutation; matrix row for non-graph edits.
- [x] [Review][Adversarial] Per-request DB read for generation hurts NFR-2 hot path — Fixed: in-memory generation mirror synced on `bump()`.
- [x] [Review][Edge] Missing handling for freshness expiry without write — Fixed: revalidation rule + matrix row + unit tests.
- [x] [Review][Edge] Missing handling for future `startDate` becoming active — Fixed: matrix row + unit tests.
- [x] [Review][Edge] Missing handling for `DepartmentHistory` write — Fixed: bump trigger + matrix row.
- [x] [Review][Structure] Matrix lacked canonical-source note — Fixed: header line per spec-1-12 pattern.
- [x] [Review][Structure] Code map missing extension spec test, `FullAccessGrant` precedent, extension composition note — Fixed.
- [x] [Review][Prose] "Formal 500+-record / 2s proof" awkward; NFR-2 naming — Fixed: kept NFR-2 consistently in Intent; clarified Story 3.7 deferral in Approach.

- [x] [Review][Patch] PP HR-line predicates not re-run on cache hit — Fixed: `audienceFromRevalidation` re-calls `resolvePp` with snapshotted `peoplePartnerId` and live `DepartmentHistory` [`access-resolver.service.ts`]
- [x] [Review][Patch] Stale-cache window between write commit and `bump()` completion — Fixed: `cache.clear()` runs before DB upsert await [`relationship-graph-generation.service.ts`]
- [x] [Review][Patch] E2E harness `resetDatabase` does not clear generation-service cache — Fixed: `clearCache()` before `loadGeneration()` [`test/support/app-harness.ts`]
- [x] [Review][Patch] E2E does not prove cache was warm before revocation — Fixed: second GET asserts `ProjectLine` [`test/access-resolution-cache.e2e-spec.ts`]
- [x] [Review][Patch] Prisma extension bump wiring lacks integration test — Fixed: `runWithBump` integration tests in `relationship-graph.extension.spec.ts`
- [x] [Review][Patch] I/O matrix cache scenarios largely untested — Fixed: added cache tests for endDate boundary, confirmed flip bump, manager/PP bump, PP HR-line revalidation, FullAccess re-query, TTL [`access-resolver.service.spec.ts`]
- [x] [Review][Patch] `project-assignment.service.spec` `update` test does not assert generation bump — Fixed: contract test with bump registry spy [`project-assignment.service.spec.ts`]
- [x] [Review][Patch] Missing C12 `Department` entity hook-point comment in extension — Fixed: documented in extension header [`relationship-graph.extension.ts`]
- [x] [Review][Defer] Employee delete / FK `SetNull` rewiring bypasses generation bump — deleting a manager/PP employee changes others' `managerId`/`peoplePartnerId` at DB level without Prisma extension hooks; no feature delete path exercised today [`relationship-graph.extension.ts`, `schema.prisma`] — deferred, pre-existing cascade semantics
- [x] [Review][Defer] `employee.createMany` array payload may skip graph-field detection — `employeeDataTouchesGraph` does not scan array elements; no current callers [`relationship-graph.extension.ts:16-21`] — deferred, latent until bulk employee seed lands

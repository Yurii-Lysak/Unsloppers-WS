---
title: 'All Employees List Performance at 500+ Records Under 2 Seconds'
type: 'feature'
created: '2026-09-08'
status: 'deferred'
review_loop_iteration: 2
story_key: '3-7-all-employees-list-performance-at-500-records-under-2-second'
baseline_commit: 'a9d3d2a23c158ce286071bcaba4a7a9e811573c7'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-13-cache-access-resolution-safely-and-revoke-immediately-on-pro.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-6-colleague-mode-of-the-list.md'
  - '{project-root}/services/backend/defects/bugs/05-employees-query-full-table-load.md'
  - '{project-root}/_bmad-output/test-artifacts/test-design-qa.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Epic 3 delivered the All Employees list (3.1–3.6), and the PRD still states NFR-2 — response under 2 seconds at 500+ records with filters and permission resolution included. The current pipeline loads every employee into memory before filtering (`field-registry.service.ts:513–554`), resolves audience multiple times per row (`employees.service.ts:92–114`), and leaves Story 1.13's access cache disabled by default. **Bootcamp decision (2026-09-08):** the shipped **~24-account seed** is sufficient for bootcamp duration; list performance at that scale is acceptable. Formal NFR-2 proof (500+ fixture, k6, CI load gate) is **deferred** — not in bootcamp scope.

**Approach (bootcamp):** Do not build synthetic 500+ fixtures, k6 harnesses, or CI perf jobs. Track known scalability defects (`05-employees-query-full-table-load.md`, `deferred-work.md`) as post-bootcamp work. Revisit full NFR-2 validation when product scale requires it.

## Boundaries & Constraints

**Always (bootcamp):**
- List endpoint scope remains `GET /api/v1/employees` — same endpoint Stories 3.1–3.6 use.
- **Dataset:** bootcamp **24-account manifest** only (`bootcamp-scope-overrides.md` / `npm run db:seed`) — no synthetic 500+ scale-up, no `perf_` fixture script, no seed expansion.
- **Verification:** existing unit + `employees.e2e-spec.ts` suites on the bootcamp seed — no k6, no load testing, no CI perf job.
- Preserve functional behavior from 3.1–3.6 — no row-set filtering, no client-only masking, no whitelist widening.
- `ACCESS_RESOLUTION_CACHE_ENABLED` stays **`false` by default** in dev/prod for bootcamp (Story 1.13 / AD-4); no perf-driven prod cache decision required now.

**Resolved (2026-09-08 — no longer Ask First):**
- **Q-1 (concurrency):** When NFR-2 is measured in the future, use **1 VU** — sequential single-viewer requests, **p(95) < 2000 ms**. No concurrent-user load target; no multi-VU load testing for now.
- **Q-7 (dataset / environment):** Bootcamp uses the **standard seed only** (~24 accounts). Synthetic 500+ fixtures and dedicated perf environments **do not apply** for bootcamp. No shared demo/staging load target.
- **CI placement:** **Skip** — no k6 or perf regression job in backend CI for bootcamp.

**Never (bootcamp):** Expanding bootcamp seed to 500+; Redis or cross-instance cache; weakening permission masking to win perf; frontend-only optimizations as NFR-2 proof; changing saved-view or colleague semantics.

**Deferred — post-bootcamp (do not implement until story reopens):**
- Synthetic ≥500 perf fixture, k6 harness (P0-014), and CI benchmark gate.
- SQL pushdown for builtin filter/sort/pagination; single `resolveAudience` per row; S10/S11 batch enrichment — tracked in `05-employees-query-full-table-load.md` and `deferred-work.md`.
- Future harness defaults (when reopened): 1 VU; three AND filters (builtin + derived + custom) + builtin sort; warm-up before p(95); ephemeral Postgres (CI service container or local `db:up`); cache-on vs cache-off deep-equal spot check optional.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Bootcamp list at seed scale | 24-account seed; typical viewer; filters/sort per 3.1–3.6 | `GET /employees` returns correct shape and entitlements; responsive at bootcamp scale | Existing 400 validation unchanged |
| Custom-field filter | Filter on custom select field | Correct row set | 400 on invalid operator (unchanged) |
| No perf harness | Any bootcamp PR | No k6/perf CI job required | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/directory/field-registry.service.ts:468–565` — `queryEmployees`; full-table `loadEmployeeSnapshots` at **637–655**; in-memory filter/sort **526–550** (defect `05-employees-query-full-table-load.md` — **open, deferred**)
- `services/backend/src/modules/directory/employee-query.helpers.ts` — `applyFilters`, `sortSnapshots`, `years_with_company` / `computeTenureYears`
- `services/backend/src/modules/directory/employees.service.ts:64–124` — `listEmployees` orchestration; per-row `maskRowCells` **402–444**, `enrichIntegratedFields` **448+**, `resolveWritableFieldIds` **503+**
- `services/backend/src/modules/access/access-resolver.service.ts:202–237` — generation-gated cache consult (Story 1.13; default off)
- `services/backend/test/employees.e2e-spec.ts:156+` — list behavior at bootcamp scale
- `services/backend/defects/bugs/05-employees-query-full-table-load.md` — remains open until post-bootcamp optimization
- Deferred from 3.1/3.3/3.5/3.6: `deferred-work.md` entries at in-memory query, per-row writableFieldIds N+1, duplicate `resolveAudience`

## Tasks & Acceptance

**Execution (bootcamp — story deferred, no implementation required):**
- [x] ~~Synthetic 500+ perf fixture~~ — **cancelled (bootcamp deferral)**
- [x] ~~k6 harness + `perf:*` scripts~~ — **cancelled (bootcamp deferral)**
- [x] ~~CI perf regression job~~ — **cancelled (bootcamp deferral)**
- [ ] Optional housekeeping only if reopening later: ensure `deferred-work.md` and defect `05` reference this deferral — rationale: traceability

**Acceptance Criteria (bootcamp):**
- Given the 24-account bootcamp seed, when `GET /api/v1/employees` is exercised via existing `employees.e2e-spec.ts`, then list behavior matches Stories 3.1–3.6 (shape, filters, masking) with no bootcamp perf regression reported.
- Given bootcamp scope, when this story is marked **deferred**, then no 500+ fixture, k6 scripts, or CI load job are added to the backend repo.

**Acceptance Criteria (deferred — post-bootcamp reopen only):**
- Given ≥500 synthetic employees and 1 VU k6 with three AND filters + sort, then p(95) < 2000 ms including permission resolution and `totalCount`.
- Given a CI perf benchmark is configured, when p(95) ≥ 2000 ms at that fixture, then the pipeline fails before merge.

## Spec Change Log

**Waiver template (post-bootcamp only):** `YYYY-MM-DD | waiver | reason | p(95) observed | approver | expiry/revisit date`

- **2026-09-08, review loop 1:** [Review] Spec hardened for NFR-2 harness — superseded by bootcamp deferral below.
- **2026-09-08, PM decision:** **Q-1 resolved** — 1 VU only when measured; no concurrent load testing for now.
- **2026-09-08, PM decision:** **Q-7 + CI resolved** — 500+ synthetic fixture outdated for bootcamp; ~24-account seed sufficient; skip k6 and CI load job for bootcamp duration. Story status → **deferred**; SQL pushdown and formal NFR-2 proof move to post-bootcamp backlog.

## Design Notes

**Why defer:** Bootcamp pivot (Story 1.16) fixed seed at 24 accounts. List perf at that scale is acceptable; investing in 500+ fixtures and CI load gates does not pay off for bootcamp demos or delivery.

**Post-bootcamp reopen triggers:** Product targets orgs at 500+ employees again; observed list latency exceeds 2s at bootcamp scale; or PM explicitly reopens NFR-2 / P0-014.

**Known debt (unchanged):** Full-table load + in-memory filter/sort remains a scalability risk documented in defect `05` — acceptable for bootcamp, not for production scale.

## Verification

**Commands (bootcamp):**
- `cd services/backend && npm run db:up && npm run db:seed && npm test && npm run test:e2e -- employees.e2e-spec.ts` — expected: pass on 24-account seed

**Not required for bootcamp:**
- ~~`perf:seed` / `perf:k6`~~
- ~~CI perf job~~

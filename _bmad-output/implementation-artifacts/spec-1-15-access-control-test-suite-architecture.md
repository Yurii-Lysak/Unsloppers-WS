---
title: 'Access-Control Test Suite Architecture'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'af406d4061c56e0278c83997269f8fd6c42b01e3'
story_key: '1-15-access-control-test-suite-architecture'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-14-prevent-section-leaks-across-all-surfaces-for-every-denied-a.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.14 delivered the denial half of AD-13 (denial coverage gate `assertDeniedMatrixCoverage`, leak end-to-end tests, and flag-gated catalog) but the full matrix program remains incomplete: `assertMatrixCoverage()` has no runtime suite recording all 80 section×audience pairs; `AccessResolver` has no master parameterized spec; multi-audience union (AD-15) and shared-link re-clamp (AD-16) have zero tests; `graph-factory.ts` still models a combined `managerLine`; CI runs only `depcruise` (no test execution); and there is no documented, repeatable test strategy — so access-control correctness still depends on ad hoc per-story coverage.

**Approach:** Finish AD-13 for the profile matrix: master `AccessResolver` unit suite plus shared-link column coverage wired through a unified `recordMatrixCoverage` gate calling `assertMatrixCoverage`; add AD-15 overlap and AD-16 re-clamp e2e scenarios; split the test oracle to `reportingLine` and `projectLine`; publish `docs/access-control-test-strategy.md`; extend the existing Story 1.19 `ci.yml` with unit, lint, and access e2e jobs; and add one harness-pattern exemplar for the Epic 6 Resourcing acceptance criterion until resourcing routes exist.

## Boundaries & Constraints

**Always:**
- **Story 1.14 is frozen** — do not weaken `assertDeniedMatrixCoverage` or duplicate denial e2e; extend collectors and add positive/grant/overlap/re-clamp suites alongside.
- **Matrix source of truth:** `test/support/access-matrix.ts` mirrors `access-model.md`. Rename `managerLine` to `reportingLine` in `PROFILE_AUDIENCES`/`ACCESS_MATRIX` (display label stays "Manager line (reporting)"); keep `projectLineDeniedCells()` for AD-14 narrowed denials. Update `graph-factory.ts` to expose `reportingLine` and `projectLine` roles matching C1 — project assignment grants `projectLine`, reports-to grants `reportingLine`, never a combined union in the oracle. Also update `flagGatedCases()` audience labels (`ReportingLine`, `ProjectLine`) and any downstream imports (`matrix-flag-gated.e2e-spec.ts`, leak assertions).
- **Coverage partition (80 pairs vs Project line):** `assertMatrixCoverage` enforces exactly **80** pairs = 16 sections × 5 oracle audiences (`self`, `reportingLine`, `pp`, `colleague`, `sharedLink`). The access-model **Project line** column is **not** a sixth oracle audience — its grants and AD-14 narrowings are covered outside this gate via `projectLineDeniedCells()` (Story 1.14), the flag-gated catalog, overlap e2e, and graph-factory oracle tests. Do not add `projectLine` to `ACCESS_MATRIX` or the 80-pair gate.
- **Full coverage gate:** Add `recordMatrixCoverage(pair: MatrixPair)` (dedupe on `pairKey` like `missingMatrixCoverage`), `getRecordedMatrixPairs()`, and `resetMatrixCoverage()` to `matrix-coverage-collector.ts`. Suites that exercise a pair call `recordMatrixCoverage`. Denied matrix cells exercised in Story 1.14 must call **both** `recordDeniedCoverage` and `recordMatrixCoverage` so both gates stay green. Complements Story 1.14's denial gate; it does not replace it.
- **Master resolver suite:** New `access-resolver.service.spec.ts` runs a parameterized suite over `matrixCells()` filtered to C1-resolvable audiences (`self`, `reportingLine`, `pp`, `colleague`). Seed mocked Prisma and `ProjectAssignment` per audience using patterns from `functional-role-data-access-boundary.spec.ts`. Disable access-resolution cache in suite setup (`ACCESS_RESOLUTION_CACHE_ENABLED=false` or mock `RelationshipGraphGenerationService`). Each case asserts `resolveAudience(viewerId, subjectId).sections[section]` matches the cell: `none`→`'none'`, `read`→`'R'`, `readWrite`→`'RW'`. **Exclude** `perFieldVisibility` cells (S16 and any other field-level cells) from section-level assertions — those remain Story 1.14 flag-gated / section-provider coverage. Call `recordMatrixCoverage` per executed pair.
- **Shared-link column:** Extend `shared-links.e2e-spec.ts` to record every `sharedLink` pair from `matrixCells()`: for `level: 'none'` cells, denial behavior stays in the leak suite (dual-record both collectors); for `read`/`readWrite` cells, assert grant `'R'` max on consume (per AD-5) and record the pair. Cells with `sharedLinkDefault: 'off'` must pass explicit `sections` in the create payload so cfg-off paths are exercised.
- **AD-15 overlap:** New `access-matrix-overlap.e2e-spec.ts` — seed viewer who is both PP and ProjectLine PM over the same subject (reuse `matrix-actors.ts`). Assert least-restrictive union (AD-15): S4 grant is `RW` (PP over ProjectLine `R`), and S2 is present on profile (PP `RW` vs ProjectLine profile-absent). Document in test strategy.
- **AD-16 re-clamp:** Extend `shared-links.e2e-spec.ts` — creator is **project-line DM only** (project assignment, no `reports-to` ReportingLine chain). Create link with S1+S2 enabled; first consume includes S2. Delete project assignment (or end DM leg) so creator narrows to ProjectLine-only with no ReportingLine fallback; second consume drops S2 without manual revoke. Record applicable `sharedLink` pairs via `recordMatrixCoverage`.
- **Harness-pattern exemplar (Epic 6 Resourcing AC):** New `cross-feature-access.exemplar.e2e-spec.ts` — ProjectLine PM with no assignment to subject calls `GET /employees/:subjectId/profile` → 403/404. Uses the same collector/assertion patterns as matrix leak tests (not a distinct Epic 6 route yet). Strategy doc names the future resourcing path and how recordings migrate when Epic 6 ships.
- **Test strategy doc:** `services/backend/docs/access-control-test-strategy.md` — coverage partition (80-pair gate vs Project line), audience×relationship-path×section structure, file map, Jest orchestration, CI commands, extension recipe for Epics 4–6, pseudonymized seed entry points (`test/support/matrix-actors.ts`, `test/support/bootcamp-seed.ts`).
- **CI:** Extend existing `services/backend/.github/workflows/ci.yml` (Story 1.19 depcruise job — see `deferred-work.md`) — keep `depcruise`; add jobs for `npm test`, `npm run lint`, and `npm run test:e2e:serial -- --testPathPattern='access-matrix|matrix-flag|shared-links|cross-feature-access'` with a Postgres 18 service container, env parity (`DATABASE_URL`, `JWT_SECRET`, `JWT_TTL_SECONDS`), and `npm run db:deploy` before e2e. Node 22 per `.nvmrc`. E2e must use `--runInBand` (already via `test:e2e:serial`).

**Ask First:** none identified.

**Never:**
- Production access-logic changes except fixes exposed by new tests.
- Epic 5 Risks/S6 `SectionProvider` AD-13 unit tests — no S6 provider module exists.
- Epic 4 action-items or Epic 12 dashboard access suites — defer until those modules land; strategy doc only.
- Directory list-column / filter-param leak e2e — Epic 3 Story 3.5 scope.
- Frontend Playwright full matrix UI suite — API e2e satisfies bootcamp epic AC (same deferral as Story 1.14).
- S8 `feedbacks-section.provider.spec.ts` — still no S8 provider module.
- Campaign-sender Colleague widening (AD-5 exception) — Epic 10.
- `graph-factory.ts` Prisma `persist()` — remains TODO until needed.
- Adding `projectLine` as a sixth column in `ACCESS_MATRIX` or the 80-pair gate.

## Design Notes

**Jest orchestration:** Module-level collector state accumulates across e2e files in one `test:e2e:serial` invocation (`--runInBand`). Call `resetMatrixCoverage()` once in `jest-e2e.global-setup.ts` (alongside `resetDeniedCoverage()`). `assertMatrixCoverage(getRecordedMatrixPairs())` runs in `jest-e2e.global-teardown.ts` after all suites finish; coverage is persisted under `test/support/.matrix-coverage-run/` so teardown can read recordings from the parent process. `recordMatrixCoverage` dedupes by `pairKey` so unit + e2e double-recording does not inflate counts.

**80-pair accounting:** 64 pairs from the master resolver unit suite (4 C1 audiences × 16 sections) plus 16 `sharedLink` pairs from shared-links / leak dual-recording. Project line positive grants are **out of scope** for `assertMatrixCoverage`.

**Self S6 regression:** Covered by existing `access-matrix-leaks.e2e-spec.ts` self/S6 denial case — no new test; ensure it continues calling both denial and positive recorders after collector extension.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Covered by |
|----------|--------------|---------------------------|------------|
| Full matrix gap | New matrix pair added; no test records it | `assertMatrixCoverage` throws naming pair | Orchestrator e2e + unit suite |
| PP + ProjectLine union | Viewer is PP and ProjectLine PM | S4 `RW`; S2 present (union) | `access-matrix-overlap.e2e-spec.ts` |
| Shared-link re-clamp | DM-only creator; assignment ends between consumes | First consume has S2; second drops S2 | `shared-links.e2e-spec.ts` |
| Unrelated PM profile | ProjectLine PM, no assignment | `GET /profile` → 403/404 | `cross-feature-access.exemplar.e2e-spec.ts` |
| CI regression | PR drops a matrix registration | CI test or e2e job fails | `ci.yml` test jobs |
| ProjectLine-only oracle | Project assignment, no reports-to | `rolesFor` → `projectLine` only; `audienceFor` ≠ `colleague` | `graph-factory.spec.ts` |

</frozen-after-approval>

## Code Map

- `test/support/access-matrix.ts` — `matrixCells()`, gates; `managerLine`→`reportingLine`; `flagGatedCases()` labels
- `test/support/matrix-coverage-collector.ts` — positive `MatrixPair` recording with dedupe
- `test/support/graph-factory.ts` (+ spec) — `reportingLine` | `projectLine` split; projectLine-only cases
- `src/modules/access/access-resolver.service.ts:159-168` — `unionSectionMaps` / AD-15
- `src/modules/access/__tests__/access-resolver.service.spec.ts` — **new** master C1 suite (cache off)
- `test/jest-e2e.global-setup.ts` / `jest-e2e.global-teardown.ts` — reset + final `assertMatrixCoverage`
- `test/access-matrix-overlap.e2e-spec.ts` — **new** AD-15
- `src/modules/access/shared-link.service.ts:316-324` — AD-16 `computeClampedSectionIds`
- `test/shared-links.e2e-spec.ts` — AD-16 re-clamp; sharedLink column recordings
- `test/cross-feature-access.exemplar.e2e-spec.ts` — **new** harness exemplar
- `test/support/matrix-actors.ts` — overlap, re-clamp, exemplar seeding
- `.github/workflows/ci.yml` — extend Story 1.19 workflow (depcruise exists); add test, lint, e2e jobs
- `docs/access-control-test-strategy.md` — **new**

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/test/support/access-matrix.ts` — rename `managerLine`→`reportingLine` across types/matrix; update `flagGatedCases()` audience strings; fix downstream imports and `access-matrix.spec.ts`
- [x] `services/backend/test/support/graph-factory.ts` + `graph-factory.spec.ts` — split `GraphRole` into `reportingLine` | `projectLine`; precedence `self` > `pp` > `reportingLine` > `projectLine` > `colleague`; add projectLine-only viewer cases (`audienceFor` ≠ `colleague`)
- [x] `services/backend/test/support/matrix-coverage-collector.ts` — add `recordMatrixCoverage` (dedupe on `pairKey`), `getRecordedMatrixPairs`, `resetMatrixCoverage`
- [x] `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` — **new** parameterized suite over C1 `matrixCells()`; cache disabled; exclude `perFieldVisibility` from section assertions; `recordMatrixCoverage` per pair
- [x] `services/backend/test/jest-e2e.global-setup.ts` + `jest-e2e.global-teardown.ts` — `resetMatrixCoverage` at suite start; `assertMatrixCoverage(getRecordedMatrixPairs())` at suite end
- [x] `services/backend/test/access-matrix-overlap.e2e-spec.ts` — **new** AD-15: S4 `RW` + S2 present for PP∩ProjectLine PM
- [x] `services/backend/test/shared-links.e2e-spec.ts` — AD-16 re-clamp (DM-only creator, two consumes); record all non-`none` `sharedLink` pairs; dual-record denials with leak suite
- [x] `services/backend/test/cross-feature-access.exemplar.e2e-spec.ts` — **new** unrelated PM profile rejection using harness patterns
- [x] `services/backend/test/access-matrix-leaks.e2e-spec.ts` — add `recordMatrixCoverage` alongside existing `recordDeniedCoverage` for exercised pairs
- [x] `services/backend/test/matrix-flag-gated.e2e-spec.ts` — call `recordMatrixCoverage` where pairs overlap the 80-pair gate (if any) — N/A: flag-gated cases are Project-line catalog only
- [x] `services/backend/docs/access-control-test-strategy.md` — **new** per Boundaries (partition, orchestration, CI, Epic 6 migration)
- [x] `services/backend/.github/workflows/ci.yml` — add `npm test`, `npm run lint`, filtered `test:e2e:serial` with Postgres service + `DATABASE_URL`/`JWT_*` env + `db:deploy`

**Acceptance Criteria:**

- Given the automated access-control suite runs in CI, when a code change accidentally grants Self read access to S6, then the corresponding negative test fails and blocks the merge
- Given the access matrix defines 80 section×audience pairs (5 oracle audiences, excluding Project line), when any pair lacks a recorded test, then `assertMatrixCoverage` fails naming the uncovered pair
- Given a viewer holds both PP and ProjectLine access over the same subject, when access is resolved, then section grants reflect the least-restrictive union (AD-15), not a single winning role
- Given a shared link was created while the creator held ReportingLine access via project assignment only, when that assignment ends before the next consume, then consumed profile sections re-clamp without manual revocation (AD-16)
- Given the harness-pattern exemplar runs until Epic 6 ships, when a PM without access to a subject attempts `GET /employees/:id/profile`, then the test rejects the request with the same rigor as matrix leak tests and the strategy doc describes the Epic 6 migration path
- Given `docs/access-control-test-strategy.md` exists, when a developer adds a new access-gated feature, then the doc specifies which harness files to extend and which CI job must pass

## Verification

**Commands:**
- `cd services/backend && npm test` — all unit tests pass including new `access-resolver.service.spec.ts` and updated `graph-factory.spec.ts` / `access-matrix.spec.ts`
- `cd services/backend && npm run lint` — no errors
- `cd services/backend && npm run db:up && npm run db:deploy && npm run test:e2e:serial -- --testPathPattern='access-matrix|matrix-flag|shared-links|cross-feature-access'` — matrix coverage gates green (orchestrator included via `access-matrix` pattern)
- `cd services/backend && npm run depcruise` — still passes

**CI parity:** The workflow must run the same lint, unit, and filtered e2e commands as above (Postgres service container with `db:deploy`).

## Spec Change Log

- **2026-09-03 (review loop 1):** Clarified 80-pair vs Project line coverage partition; added Design Notes for Jest orchestration and dedupe; pinned AD-16 seed (DM-only, two consumes); excluded `perFieldVisibility` from master resolver assertions; extended CI env/lint; reframed cross-feature exemplar as harness pattern; updated `ci.yml` and `bootcamp-seed.ts` references; condensed I/O matrix; expanded tasks for flag-gated labels and graph-factory projectLine-only cases.

### Review Findings

- [x] [Review][Patch] Cross-feature exemplar asserts `GET /timeline` instead of `GET /profile` [`test/cross-feature-access.exemplar.e2e-spec.ts:67-72`]
- [x] [Review][Patch] AD-16 re-clamp scenario tests S9 removal, not spec-pinned S2 first-consume / second-absent [`test/shared-links.e2e-spec.ts:382-454`]
- [x] [Review][Patch] `sharedLink` `none` cells recorded in shared-links suite without asserting the denied section on consume [`test/shared-links.e2e-spec.ts:456-483`]
- [x] [Review][Patch] `recordFlagGatedCoverage` lost dedupe when collector moved to file persistence [`test/support/matrix-coverage-collector.ts:61-64`]
- [x] [Review][Patch] Redundant `resetDeniedCoverage()` in leaks `beforeAll` fights global-setup orchestration [`test/access-matrix-leaks.e2e-spec.ts:37-38`]
- [x] [Review][Patch] Add `test/support/.matrix-coverage-run/` to `.gitignore` [`.gitignore`]
- [x] [Review][Patch] `agentForAudience` silent `default` falls back to `reportingLineAgent` [`test/access-matrix-positive.e2e-spec.ts:29-30`]
- [x] [Review][Patch] Strategy doc still documents profile URL for exemplar while code hits timeline [`docs/access-control-test-strategy.md:57`]
- [x] [Review][Defer] Full `npm test` CI job surfaces pre-existing AppModule/functional-role DI failures unrelated to Story 1.15 diff — deferred, pre-existing

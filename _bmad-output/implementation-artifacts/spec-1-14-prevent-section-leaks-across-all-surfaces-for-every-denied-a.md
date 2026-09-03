---
title: 'Prevent Section Leaks Across All Surfaces for Every Denied Audience'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 2
baseline_commit: '333c0f4cb976821c9cc6ebaba590e851418ee1de'
story_key: '1-14-prevent-section-leaks-across-all-surfaces-for-every-denied-a'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-8-enforce-the-colleague-whitelist-everywhere.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-11-generate-a-shareable-profile-link.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 1.6–1.12 enforce access on individual surfaces, and Story 1.8 covers the Colleague whitelist ad hoc, but there is no matrix-driven negative suite proving every `—` cell and flag-gated case stays absent on every live emission path. `access-matrix.ts` and `assertMatrixCoverage()` exist yet no runtime test registers covered denial pairs — a new section could leak without failing CI.

**Approach:** Add a matrix leak harness (implements AD-13's negative-case convention, not the full AD-13 program) driven from `test/support/access-matrix.ts`: enumerate all `level: 'none'` cells plus Project-line narrowed denials (AD-14) and an explicit flag-gated catalog, assert literal absence (not `null`, not `unavailable`) on each applicable surface via backend e2e and per-`SectionProvider` unit tests, and call `assertDeniedMatrixCoverage()` so matrix edits to denial cells force new cases. Full-matrix positive coverage and overlap/re-clamp suites remain Story 1.15.

## Boundaries & Constraints

**Always:**
- **Denied semantics (normative):** For `accessLevel === 'none'`, the section id must be **absent** from profile `sections` — no key, no `null`, no `{ status: 'unavailable' }`. `unavailable` applies only when grant ≠ `none` but provider is missing/broken (spec-1-6); never conflate with denial.
- **Matrix source of truth:** `test/support/access-matrix.ts` mirrors `access-model.md`. Add:
  - `deniedMatrixCells(): MatrixCell[]` — `cell.level === 'none'`
  - `projectLineDeniedCells(): ProjectLineDeniedCell[]` — AD-14 narrowed cells absent for Project line only (S2, S3 profile absence; S5 non-CV document types) even though combined `managerLine` in the matrix shows `read`
  - `flagGatedCases(): FlagGatedCase[]` — explicit catalog entries, **not** inferred from every matrix `exception`/`qualifier`. Each entry: `{ section, audience, rule, absentFields?, seedRef? }` where `rule` is one of `field-absent`, `record-absent`, `write-denied`, `payload-narrowed`
  When `access-model.md` changes, update `access-matrix.ts` in the same commit.
- **Coverage gate (denied pairs only):** `assertDeniedMatrixCoverage(coveredPairs)` filters to `deniedMatrixCells()` ∪ `projectLineDeniedCells()` (as section/audience pairs). Each executed denial pair calls `recordDeniedCoverage(pair)` via a shared `test/support/matrix-coverage-collector.ts` module. A single orchestrator `afterAll` in `access-matrix-leaks.e2e-spec.ts` calls `assertDeniedMatrixCoverage(getRecordedDeniedPairs())` — flag-gated and provider unit suites register the same pairs they cover. **Do not** call full `assertMatrixCoverage()` here (that is Story 1.15's 80-pair scope).
- **Surfaces:** Canonical behavioral scenarios are in the I/O matrix below. Bootcamp scope:
  - **Profile API** — `GET /employees/:subjectId/profile`
  - **Parallel routes** — only where a dedicated route exists today; denied grant → **403**, granted grant with narrowing → **200** with narrowed payload. Inventory:

    | Section | Route | Denied audiences (bootcamp) | Notes |
    |---------|-------|----------------------------|-------|
    | S7 | `/management-notes` | Colleague, Self (flag-off notes) | Self has section grant; route tests flag-gated absence |
    | S9 | `/timeline` | Colleague | Self/Manager granted |
    | S10 | `/leaves` | — | Colleague **granted** (dates-only); do not map to 403 |
    | S16 | `/custom-fields/values/:id` | Colleague | Per-field visibility in flag-gated suite |

    Sections with no parallel route (S6, S15, S2, S3, etc.): profile-key absence is sufficient; do not invent routes.
  - **Shared link** — `GET /shared-links/:token/profile` for `sharedLink` denied cells; consume assertions on **200** bodies only
  - **Directory (deferred):** List columns and filter params are Epic 3 / Story 1.15. Bootcamp `GET /employees` returns `{id, displayName}` only (spec-1-8). Cover directory intent via a unit test that `SectionAccessGate.listGrantedSections(Colleague)` returns `['S1','S10','S11']` until Story 1.10 — no e2e column assertions in this story.
- **Flag-gated catalog (minimum explicit entries):**

  | Section | Audience | Rule | Absence |
  |---------|----------|------|---------|
  | S7 | Self | record-absent | Notes with both visibility flags off |
  | S7 | ProjectLine PM | record-absent + write-denied | Only `visibleForPm` notes; POST/PATCH → 403 |
  | S8 | Self | record-absent | Feedback not shared with employee |
  | S1 | Colleague, Self (D5) | field-absent | `mentor` field |
  | S10 | Colleague | field-absent | `type`, `approvalState` |
  | S11 | Colleague | payload-narrowed | `pm`, `dm`, `period` absent; project name only |
  | S16 | Colleague | field-absent | Management-only field absent; colleague-visible field may appear (seed two fields) |
  | S5 | ProjectLine | payload-narrowed | Non-CV/certificate document types absent |
  | S9 | ProjectLine | write-denied | POST `/timeline` → 403; ReportingLine POST → 200 |

  Reuse seed/fixtures from `management-notes.e2e-spec.ts`, `employee-profile.e2e-spec.ts`, `employee-profile-custom-fields.e2e-spec.ts` — do not rewrite business logic.
- **Per-provider AD-13 (denied cells):** Each registered `SectionProvider` (`S1`, `S7`, `S8`, `S9`, `S10`, `S11`, `S16`) gets `describe.each` over `deniedMatrixCells()` filtered to that section, asserting `ForbiddenException` or non-invocation when grant is `none`. **S6 Risks provider** is deferred to Story 1.15 — Self S6 profile absence is covered by e2e here, not provider-level AD-13.
- **Test actors:** Extend `test/support/bootcamp-seed.ts` (or add `matrix-actors.ts`) to return logged-in agents for Self, ReportingLine manager, **ProjectLine DM** (dedicated — not combined `managerLine`), ProjectLine PM, PP, Colleague, and shared-link recipient — reuse `seedBootcampWhitelistGraph` + shared-links seed pattern from `shared-links.e2e-spec.ts`.
- **Audience mapping:** In `access-matrix.ts` today, `managerLine` encodes **ReportingLine** only. ReportingLine and PP have zero `none` cells in that column — skip them in `deniedMatrixCells()`. **ProjectLine** denials are **not** in `deniedMatrixCells()`; use `projectLineDeniedCells()` and the ProjectLine DM/PM agents. C1 `ReportingLine`/`ProjectLine` split matters for S7, S9, and S5 flag-gated cases.

**Ask First:** none identified.

**Never:**
- New production access logic — this story adds tests and thin test helpers only; fix production only when a new test exposes a real leak.
- Export `.xlsx`, search, notifications, campaign-sender exception (Epic 10) — surfaces not implemented; defer assertions until those stories land.
- Frontend Playwright matrix suite — no profile UI e2e exists yet; **API consume assertions satisfy bootcamp epic AC** (epics.md Story 1.14 UI clause deferred). Optional thin UI check only if profile page + `data-section` hooks land in the same sprint.
- Multi-audience overlap union suite (AD-15) — Story 1.15.
- Shared-link creator access re-clamp on every view (AD-16) — Story 1.12 / 1.15.
- Directory list columns and filter-param leak assertions — Epic 3 Story 3.5 / Story 1.15.
- S6 `SectionProvider` AD-13 unit tests — Story 1.15 (profile e2e Self S6 absence is in scope here).
- `graph-factory.ts` `managerLine` → `ReportingLine`/`ProjectLine` rename (deferred-work.md) — Story 1.15.
- CI workflow / test-strategy documentation — Story 1.15.
- Resourcing and Action Items cross-epic coverage — Story 1.15.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Self denied profile | Employee E requests own profile | `sections` has no `S6` or `S15` keys | 401 unauthenticated |
| Colleague denied profile | Colleague V requests B's profile | `sections` keys ⊆ `{S1,S10,S11}` until Story 1.10 ships Colleague S16 per-field visibility | 404 unknown employee |
| Colleague granted leaves route | Colleague V → B's `/leaves` | 200; dates-only payload; no `type` | 401; 403 no Employee; 404 |
| Colleague denied parallel route | Colleague V → B's `/timeline` | — | 403 |
| ProjectLine denied profile | ProjectLine DM requests B's profile | No `S2` or `S3` keys | 404 unknown employee |
| ProjectLine S5 narrowing | ProjectLine DM requests B's profile S5 | CV and certificates only; other document types absent | N/A |
| Viewer without Employee | Authenticated user, no linked `Employee` → gated profile or parallel route | — | 403 before C1 |
| Shared link never sections | Link with S1+S9; recipient consumes (valid token) | `sections` keys ⊆ `{S1,S9}`; no S3/S7/S13/S14 | 401/403/404 per shared-link rules |
| Shared link invalid token | Expired, revoked, or malformed token → consume | — | 401/403/404; no section absence assertions on error body |
| Create forces never section | POST shared-link with `sections: ['S7']` | — | 400 |
| Flag-off S7 to Self | Note with both flags off; Self reads profile/route | Note absent from S7 payload | N/A |
| PM flag-gated S7 | ProjectLine PM; one note `visibleForPm` | PM sees only flagged note, read-only | 403 on POST/PATCH |
| S9 ProjectLine write denied | ProjectLine DM POST B's `/timeline` | — | 403; ReportingLine manager POST → 200 |
| Colleague S10 narrowing | Colleague profile S10 | `type` and `approvalState` null/absent | N/A |
| Colleague S11 narrowing | Colleague profile S11 | Project name only; `pm`, `dm`, `period` absent | N/A |
| S16 per-field Colleague | Colleague profile S16; seed colleague-visible + management-only fields | Colleague-visible field present; management-only field absent | N/A |
| Granted unavailable ≠ denied | ReportingLine viewer; S6 no Risks provider | `sections.S6.status === 'unavailable'` allowed | Excluded from denied-cell enumeration |
| Denied matrix coverage gap | New `none` cell or ProjectLine denial added; no test records pair | `assertDeniedMatrixCoverage` throws naming pair | Build fails |

</frozen-after-approval>

## Code Map

Symbol anchors only — implementation tasks live in **Tasks & Acceptance**.

- `access-matrix.ts` — `deniedMatrixCells`, `projectLineDeniedCells`, `flagGatedCases`, `assertDeniedMatrixCoverage`
- `matrix-coverage-collector.ts`, `matrix-leak-assertions.ts`, `matrix-actors.ts` — shared test harness
- `profile-assembler.service.ts`, `section-access-gate.service.ts`, `shared-link-matrix.ts` — production denial paths under test
- `colleague-whitelist.e2e-spec.ts`, `employee-profile.e2e-spec.ts`, `shared-links.e2e-spec.ts`, `management-notes.e2e-spec.ts` — patterns to extend, not duplicate wholesale
- `*-section.provider.spec.ts` (S1, S7, S8, S9, S10, S11, S16) — per-provider AD-13 denied-cell tests

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/test/support/access-matrix.ts` — add `deniedMatrixCells()`, `projectLineDeniedCells()`, `flagGatedCases()` (explicit `FlagGatedCase[]`), and `assertDeniedMatrixCoverage()`; keep `assertMatrixCoverage()` for Story 1.15
- [x] `services/backend/test/support/matrix-coverage-collector.ts` (new) — `recordDeniedCoverage(pair)`, `getRecordedDeniedPairs()`, `resetDeniedCoverage()`; imported by e2e + provider specs
- [x] `services/backend/test/support/matrix-leak-assertions.ts` (new) — `expectSectionAbsentFromProfile(body, sectionId)`, `expectParallelRouteDenied(agent, route)`, `expectParallelRouteGranted(agent, route)` helpers; parallel-route map from Boundaries inventory only
- [x] `services/backend/test/support/matrix-actors.ts` (new) — seed + login agents for Self, ReportingLine, ProjectLine DM, ProjectLine PM, PP, Colleague, shared-link recipient
- [x] `services/backend/test/access-matrix-leaks.e2e-spec.ts` (new) — `it.each(deniedMatrixCells())` profile assertions; `it.each(projectLineDeniedCells())` ProjectLine profile assertions; parallel-route 403 only for mapped denied sections; `afterAll(() => assertDeniedMatrixCoverage(getRecordedDeniedPairs()))`
- [x] `services/backend/test/matrix-flag-gated.e2e-spec.ts` (new) — `it.each(flagGatedCases())` including S7/S8/S1-mentor/S10/S11/S16/S5/S9-write cases; register covered pairs via collector
- [x] `services/backend/test/shared-links.e2e-spec.ts` — consume assertions on 200 only: S3/S7/S13/S14 absent; expired/revoked token rows per matrix; DM creator + S1+S9 link per epics AC
- [x] `services/backend/test/employee-profile.e2e-spec.ts` — Self own-profile: `S6`/`S15` keys absent
- [x] `services/backend/src/modules/access/__tests__/section-access-gate.service.spec.ts` — add `listGrantedSections(Colleague)` → `['S1','S10','S11']` (directory deferral)
- [x] `services/backend/src/modules/access/__tests__/projects-section.provider.spec.ts` (new) — AD-13 denied-cell tests for S11
- [x] `services/backend/src/modules/timeline/__tests__/timeline-section.provider.spec.ts` (new) — AD-13 denied-cell tests for S9
- [ ] `services/backend/src/modules/feedbacks/__tests__/feedbacks-section.provider.spec.ts` (new or extend) — AD-13 denied cells + S8 flag-gated Self record-absent unit case — **skipped:** no S8 `SectionProvider` module yet
- [x] Existing `*-section.provider.spec.ts` files (S1, S7, S10, S16) — add `describe.each` over denied cells; call `recordDeniedCoverage` per executed pair

**Acceptance Criteria:**

*Bootcamp: API-only — epics.md Story 1.14 UI clause ("inspects the rendered page") is satisfied by API consume assertions; Playwright deferred per Boundaries Never.*

- Given Employee E requests their own profile, when the response is inspected, then S6 and S15 are entirely absent — no keys, no empty placeholders
- Given a DM opens a shared link created with only S1 and S9 enabled, when the recipient calls `GET /shared-links/:token/profile` (valid token) and attempts create with `sections` containing S3, S7, or S13, then those sections are absent from the consume response and create returns 400 for never-sections
- Given any `(section, audience)` pair where `ACCESS_MATRIX[section][audience].level === 'none'`, when the applicable surface is exercised with a correctly seeded actor, then the section is absent from profile JSON or the parallel route returns 403 (parallel routes only where inventory defines a route)
- Given a ProjectLine DM requests a subject's profile, when S2 or S3 would be denied under AD-14, then those section keys are absent from the profile response
- Given the access matrix gains a new `none` cell or ProjectLine denial without a matching test recording the pair, when the leak suite runs, then `assertDeniedMatrixCoverage` fails naming the uncovered pair
- Given a ReportingLine manager views a subject with no S6 provider, when the profile is returned, then `sections.S6` may be `{ status: 'unavailable' }` and this case is excluded from the denied-cell enumeration

## Verification

**Commands:**
- `cd services/backend && npm test` — expected: all unit tests pass including new provider matrix specs and `listGrantedSections` directory deferral test
- `cd services/backend && npm run test:e2e -- --testPathPattern='access-matrix-leaks|matrix-flag-gated|shared-links|employee-profile|colleague-whitelist|management-notes'` — expected: denied-cell, ProjectLine, and flag-gated e2e green
- `cd services/backend && npm run lint` — expected: no errors

### Review Findings

- [x] [Review][Patch] ProjectLine S5 `payload-narrowed` records coverage without asserting document narrowing [`services/backend/test/access-matrix-leaks.e2e-spec.ts:588-598`]
- [x] [Review][Patch] `flagGatedCases()` catalog not enforced — hand-written e2e tests + length smoke only; no `assertFlagGatedCoverage` gate [`services/backend/test/matrix-flag-gated.e2e-spec.ts`, `services/backend/test/support/access-matrix.ts`]
- [x] [Review][Patch] Remove incorrect `flagGatedCases` entry S9/ReportingLine/`write-denied` (ReportingLine timeline POST is granted per spec) [`services/backend/test/support/access-matrix.ts:425-428`]
- [x] [Review][Patch] Provider denied-matrix specs omit `recordDeniedCoverage` per spec task [`services/backend/src/modules/management-notes/__tests__/management-notes-section.provider.spec.ts`, `services/backend/src/modules/timeline/__tests__/timeline-section.provider.spec.ts`]
- [x] [Review][Patch] Shared-link denied-cell e2e always creates links with `sections: ['S9']` — may not exercise absence for never-section denied cells [`services/backend/test/access-matrix-leaks.e2e-spec.ts:548-575`]
- [x] [Review][Patch] `access-matrix-leaks` create-rejection loop omits S14 (covers S3/S7/S13 only) [`services/backend/test/access-matrix-leaks.e2e-spec.ts:671-680`]
- [x] [Review][Patch] PM S7 `write-denied` flag case tests POST only, not PATCH [`services/backend/test/matrix-flag-gated.e2e-spec.ts:811-814`]
- [x] [Review][Patch] No test documenting `unavailable` ≠ denied (S6 exclusion from denied enumeration) [`services/backend/test/support/access-matrix.spec.ts`]
- [x] [Review][Patch] Timeline provider unit test mocks service `ForbiddenException` instead of asserting provider behavior on `S9: 'none'` [`services/backend/src/modules/timeline/__tests__/timeline-section.provider.spec.ts:32-41`]
- [x] [Review][Patch] Mojibake in Story 1.14 comment strings (`ΓÇö` instead of em dash) [`services/backend/test/support/access-matrix.ts`]
- [x] [Review][Defer] S8 Self `record-absent` flag-gated case — spec task explicitly skipped (no S8 `SectionProvider` yet) — deferred per spec open task
- [x] [Review][Defer] S16 parallel route `/custom-fields/values/:id` Colleague narrowing — already covered in `colleague-whitelist.e2e-spec.ts` — deferred, duplicate coverage

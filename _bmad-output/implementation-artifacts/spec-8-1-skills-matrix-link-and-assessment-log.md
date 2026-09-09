---
title: 'Skills Matrix Link and Assessment Log'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 1
story_key: '8-1-skills-matrix-link-and-assessment-log'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-8-context.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Profile section S12 (CDS) has no `SectionProvider`, so every viewer with `S12` access (Self, Reporting line, Project line, PP, cfg'd shared links) gets `unavailable` instead of the person's skills-matrix link and assessment history — Epic 8's registry/hub is entirely undelivered.

**Approach:** Add a `cds` module owning two new tables — a department+position-keyed skills-matrix dictionary and a flat assessment log — and register an `S12` `SectionProvider` that resolves the employee's current department+position to a matrix link and lists their assessment entries, newest first. Read-only: creating/editing assessments (8.3), IDP records (8.2), and directory filters (8.4) are later stories; the dictionary itself has no maintenance UI yet in any CDS story, so it is seed-managed.

## Boundaries & Constraints

**Always:**
- `@RegisterProvider('section', 'S12')` in new `cds` module; mirror the S6 gate (`ForbiddenException` on `'none'`) — `risks-section.provider.ts:11-38`. `ProfileAssembler` already lists `S12` in `ALL_SECTION_IDS` and `AccessResolver` already grants it (R self, RW reporting/project-line/PP, none colleague, cfg shared-link) — no changes needed there.
- Resolve current department+position from the employee's open `DepartmentHistory`/`PositionHistory` rows (`effectiveTo: null`), then look up the `Department` via the existing `DepartmentDirectory.getDepartmentByName` contract — never a private copy of the department table (AD-1/AD-14).
- New `SkillsMatrixEntry` model, unique on `[departmentId, position]`; the section provider re-resolves it on every call — never denormalized onto `Employee` — so a dictionary update or department rename changes every affected profile with zero per-employee writes.
- New `CDSAssessment` model FK'd directly to `employeeId` (flat, mirrors `RiskRecord`/`GradeHistory` — no separate "CDSRecord" wrapper table); listed newest-`date`-first. No numeric score/rating field anywhere (registry-only boundary, `SPEC.md`).
- Seed: new `seedSkillsMatrix` (mirrors `seedDepartments`'s "derive from data, never invent" discipline) creates one dictionary entry per distinct (current department, current position) pair actually present in the bootcamp population, plus a couple of demonstrative completed `CDSAssessment` rows so the acceptance criteria are manually verifiable; called from `seed.service.ts` after `seedDepartments`.
- No dictionary entry for a dept+position → `matrixLink: null`; section still renders (`data`, not `unavailable`) — "no mapping yet" is never conflated with a registration gap.
- Frontend: real `S12` title key + read-only card (mirror `RequestHistorySection` pattern) in `profile-sections.tsx`; swap `shared-link-sections.ts`'s generic `S12` fallback title for the same key.

**Ask First:** If the bootcamp seed's deterministic dept/position assignment can't produce at least one employee with a demonstrative completed assessment without hand-picking one, flag to the human rather than inventing extra seed fixtures.

**Never:** IDP fields/table (8.2); assessment create/edit/conclusion-edit endpoints (8.3); any dictionary maintenance UI/endpoint (unscoped by any of the 4 CDS stories); last-assessment/open-IDP directory filters (8.4); a `CDSRecord` wrapper entity.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| MATRIX_RESOLVED | ReportingLine viewer; dept+position mapped; subject has 1 completed assessment | `sections.S12.data` has `matrixLink` + one entry (date, assessor, resultLink, conclusion) | — |
| DICTIONARY_UPDATE_PROPAGATES | Dictionary entry's `fileUrl` updated; N employees share dept+position | Every affected profile's next view shows the new link; zero employee rows touched | — |
| PROJECT_LINE_UNNARROWED | ProjectLine viewer (not ReportingLine) | Same S12 payload as ReportingLine — S12 is not in AD-14's narrowed-cell set | — |
| COLLEAGUE_DENIED | Colleague viewer | `sections` has no `S12` key at all | — |
| NO_DICTIONARY_ENTRY | Authorized viewer; dept+position has no `SkillsMatrixEntry` row | `matrixLink: null`; assessments still render if present | — |
| EMPTY_LOG | Authorized viewer; employee with zero assessments | `assessments: []` (section present, not `unavailable`) | — |
| SHARED_LINK_CFG | Shared link with S12 enabled, creator had R/RW | Consume response includes S12 via same provider | 403 if creator can't create links; 400 if requested beyond creator's grant |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- add `SkillsMatrixEntry` (`departmentId`+`position` unique; `fileUrl` field -- exposed on the wire as `matrixLink`, per the I/O matrix and AC wording) and `CDSAssessment` (flat `employeeId` FK; `date`, `assessor`, `resultLink`, `conclusion` fields) models + `Department`/`Employee` back-relations; migration
- `services/backend/src/prisma/seed/seed.skills-matrix.ts` -- **new**, `seedSkillsMatrix()` mirroring `seed.departments.ts:36-119` including its upsert-by-key discipline (`findUnique` on `[departmentId, position]` then `update`/`create`, never delete stale entries -- reruns of `db:seed` must stay idempotent as the bootcamp population changes); also seeds demo `CDSAssessment` rows
- `services/backend/src/prisma/seed/seed.service.ts:94-100` -- call `seedSkillsMatrix` after `seedDepartments`
- `services/backend/src/modules/cds/cds.module.ts`, `cds.service.ts`, `cds-section.provider.ts`, `entities/cds-section.entity.ts` -- **new**; `buildSection()` mirrors `risks.service.ts:30-33`; assessment list query mirrors `risks.service.ts`'s `loadRecordsForSubject` multi-key `orderBy` (`date` desc, then `createdAt` desc, then `id` desc) so same-`date` entries sort deterministically; provider mirrors `risks-section.provider.ts:11-38`; the section entity must always serialize both `matrixLink` and `assessments` keys (never an object with zero own keys) -- `ProfileAssemblerService.isUnavailablePayload` (`profile-assembler.service.ts:202-212`) treats a zero-key object as a registration-gap "unavailable", which would misfire on a genuinely-empty-but-valid CDS payload (no dictionary entry + no assessments) if either key were dropped during serialization
- `services/backend/src/modules/access/department-directory.service.ts` -- reuse `getDepartmentByName` (no change)
- `services/backend/src/app.module.ts` -- register `CdsModule`
- `services/backend/test/employee-profile.e2e-spec.ts` -- extend for S12 positive (ReportingLine/ProjectLine/PP/Self)/negative (Colleague) cases + shared-link cfg
- `services/backend/src/modules/cds/__tests__/cds-section.provider.spec.ts`, `cds.service.spec.ts` -- **new**, matrix rows
- `services/frontend/src/types/employee-profile.ts` -- add `CdsSection`/`CdsAssessmentEntry` types (`SectionId` already includes `S12`)
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx:44-55,76-173` -- add `S12` title key + renderer
- `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/CdsSection.tsx` -- **new**, mirror `RequestHistorySection/RequestHistorySection.tsx`
- `services/frontend/src/pages/EmployeeProfilePage/shared-link-sections.ts:30` -- swap generic S12 title for real key
- `services/frontend/src/locales/en/translation.json` -- add `employeeProfile.sections.cds` + `employeeProfile.s12.*` keys

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration -- add `SkillsMatrixEntry`, `CDSAssessment` -- foundation tables for the section provider and seed
- [x] `services/backend/src/prisma/seed/seed.skills-matrix.ts` + `seed.service.ts` wiring -- seed dictionary + demo assessments -- makes ACs manually verifiable
- [x] `services/backend/src/modules/cds/*` + `app.module.ts` registration -- S12 section provider -- delivers the read side
- [x] `services/backend/src/modules/cds/__tests__/*` + e2e extensions -- cover I/O matrix rows
- [x] `services/frontend/src/types/employee-profile.ts` + `profile-sections.tsx` + `CdsSection/` + `shared-link-sections.ts` + `translation.json` -- S12 read UI

### Review Findings

- [x] [Review][Patch] Add unit tests for missing position history and department-directory miss paths [`cds.service.spec.ts`]
- [x] [Review][Patch] Add unit tests for NO_DICTIONARY_ENTRY with assessments and EMPTY_LOG matrix-mapped cases [`cds.service.spec.ts`]
- [x] [Review][Patch] Assert assessment `orderBy` clause in `CdsService` [`cds.service.spec.ts`]
- [x] [Review][Patch] Add Self and PP delegation coverage in `CdsSectionProvider` [`cds-section.provider.spec.ts`]
- [x] [Review][Patch] Strengthen ProjectLine/PP e2e to assert payload parity with ReportingLine [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Add e2e for NO_DICTIONARY_ENTRY (`matrixLink: null`, assessments present, not `unavailable`) [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Add e2e for EMPTY_LOG (mapped matrix, `assessments: []`, not `unavailable`) [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Prove dictionary updates propagate to multiple employees sharing dept+position [`employee-profile.e2e-spec.ts`]
- [x] [Review][Defer] S12-specific shared-link 400 beyond-grant e2e — deferred, pre-existing: `shared-link.service.spec.ts` already covers the grant-validation path generically with S6
- [x] [Review][Defer] Playwright coverage for `CdsSectionCard` UI branches — deferred, pre-existing: frontend e2e not in story 8.1 verification commands; API contract covered by backend e2e
- [x] [Review][Decision] Resourcing auto-generated manager links should include S12 by default — deferred to a later Resourcing/CDS story; frozen spec Never list and change-log item 5 keep this out of 8.1 scope

**Acceptance Criteria:**
- Given "Engineering"+"Software Engineer" is mapped to a matrix file and the employee has a completed assessment on file, when their manager opens the CDS section, then the manager sees the current matrix link and the assessment log entry with its date, assessor, result link, and conclusion
- Given several employees share that department+position and the dictionary entry is updated to a new file, when any of those profiles is viewed afterward, then all show the new file with no individual employee record edited — and a Colleague viewer sees no trace of S12 at all

## Spec Change Log

**Proposed — pending human renegotiation of the frozen block (bmad-review, iteration 1):**

1. **Missing history rows.** Boundaries only defines resolution when an open `DepartmentHistory`/`PositionHistory` row exists. No row/rule covers an employee with no open row in either table (or a `DepartmentDirectory.getDepartmentByName` miss — `DepartmentHistory.value` present but no matching `Department.name`). Proposal: treat both as equivalent to NO_DICTIONARY_ENTRY (`matrixLink: null`, section still `data`, assessments render if present) rather than `unavailable`, and add an explicit I/O matrix row.
2. **`assessor` field shape.** `epic-8-context.md` explicitly leaves this to the implementer ("free text or a person reference"); Boundaries never picks one. Proposal: free-text `String` — simplest, and Boundaries already rules out any wrapper/reference entity elsewhere in CDS's read model.
3. **`resultLink` required or nullable.** AC #1 implies every listed assessment carries a result link, but nothing states whether `CDSAssessment.resultLink` is required at creation — relevant now for seeding and later for Story 8.3's create endpoint. Proposal: required (non-null) `String`, consistent with the registry-only, no-partial-record framing.
4. **Self/PP have no dedicated I/O matrix row.** Boundaries states Self gets `R` and PP gets `RW` (same payload shape as ReportingLine), and Code Map's e2e bullet already commits to testing both, but the frozen I/O matrix never enumerates either scenario — Self in particular differs in `accessLevel` (`R`, not `RW`) from every listed row. Proposal: add SELF_READ_ONLY and PP_SAME_AS_REPORTING rows for completeness with the e2e coverage already promised.
5. **Cross-epic gap: resourcing auto-generated shared links.** `epic-8-context.md`'s UX Patterns section states "CDS is one of the sections included in that link's scope by default" for a resourcing candidate's auto-generated manager link. Checked in code: `shared-link-matrix.ts`'s `SHARED_LINK_DEFAULT_SECTIONS` is `['S1']` only (S12 is `cfg`, opt-in), and `resourcing.service.ts` reuses Story 6.2's submit-flow link rather than constructing its own section list — so this requirement is not satisfied anywhere in the current codebase, and neither Boundaries nor the Never list in this spec addresses it. Needs a decision: is wiring S12 into that default scope in-scope for 8.1, or explicitly deferred to a later Resourcing/CDS story (and if so, which)?

## Design Notes

`CDSAssessment` FKs directly to `employeeId` rather than through an intermediate "CDS record" entity — Prisma's `employee.cdsAssessments` relation already gives "many entries per employee" without a join table nobody else needs; `RiskRecord`/`GradeHistory` establish this flat-FK convention already. `SkillsMatrixEntry.position` is a plain string matching `PositionHistory.value` (no `Position` entity exists in this schema) — consistent with `DepartmentHistory.value` also being a plain string matched against `Department.name`. Unlike the department leg, there is no directory/lookup contract for position — matching is exact-string equality (Prisma default, case- and whitespace-sensitive) directly against `SkillsMatrixEntry.position`; `seedSkillsMatrix` must derive its `position` values byte-identically from `PositionHistory.value`, since any mismatch resolves silently to `matrixLink: null` (indistinguishable from a genuinely unmapped pair) rather than an error.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- employee-profile.e2e-spec.ts` -- expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` -- expected: pass

**Manual checks (if no CLI):**
- After `npm run db:seed`, a bootcamp manager opens a report's profile and sees the CDS section with a matrix link and at least one assessment entry; the same employee's Colleague viewer sees no CDS section at all
- Via `npm run db:studio`, edit one `SkillsMatrixEntry.fileUrl` shared by two or more seeded employees (same dept+position); reload both profiles and confirm both show the new link with no `Employee`/`CDSAssessment` row touched (AC #2 / DICTIONARY_UPDATE_PROPAGATES) — the second acceptance criterion, previously absent from the manual checklist

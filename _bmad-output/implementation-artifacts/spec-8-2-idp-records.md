---
title: 'IDP Records'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 0
story_key: '8-2-idp-records'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-8-context.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** S12 (CDS) is read-only since Story 8.1 — nobody can create an Individual Development Plan or mark one complete, so FR-33's development-plan tracking is entirely undelivered.

**Approach:** Add an `IDPRecord` model owned by the existing `cds` module and three endpoints on it — manager/PP create, manager/PP update, and a self-only "mark complete" action — reusing the S12 access already resolved by Epic 1 (Self `R`, Reporting/Project-line/PP `RW`). Extend the existing S12 section payload to list IDP records alongside the matrix link and assessment log.

## Boundaries & Constraints

**Always:**
- New `IDPRecord` model FK'd directly to `employeeId` (flat single-FK shape, mirrors `CDSAssessment` — not `RiskRecord`, which carries two FKs, `subjectEmployeeId` and `authorEmployeeId` — no wrapper entity); fields `description`, `deadline` (Date), `fileUrl`, nullable `completedAt`. All three input fields required at creation — no partial IDP records (registry-only framing, per spec-8-1's `resultLink` decision).
- Create/update gated like `risks.controller.ts:79-102`'s `assertCanCreate`: allow when resolved S12 is `RW`, else allow only if the viewer holds `maintain_cds_records` (`PERMISSION_KEYS.MAINTAIN_CDS_RECORDS`) **and** S12 is not `none`; otherwise 403.
- `completedAt` is never accepted by the create/update DTOs (`forbidNonWhitelisted: true` ValidationPipe, mirrors `mentorship.controller.ts`'s pattern) — it is set only by the complete endpoint.
- Complete endpoint is self-only: 403 unless `viewerEmployeeId === employeeId` (mirrors `action-items.service.ts:128-132`); guarded update (`where: { completedAt: null }`, mirrors `action-items.service.ts:140-146`) — 409 if already completed, making re-completion and races safe.
- Once `completedAt` is set, the update endpoint 409s for that record — a completed IDP cannot be edited or reopened (decisions.md Appendix — "Reopening a completed IDP (CAP-8)" — current default: disallow until told otherwise; D7 covers only which story/developer owns the self-complete checkbox, not this default).
- `CdsSectionEntity.idpRecords` always serializes, `[]` when empty — never omitted (same zero-key "unavailable" trap noted in spec-8-1's Code Map).

**Ask First:** If a completed IDP later needs a reopen path, flag to the human rather than adding one undocumented.

**Never:** an uncomplete/reopen endpoint; delete for IDP records (unscoped by this story); any numeric score/rating field (CDS registry-only boundary); assessment create/edit (Story 8.3) or directory filters (Story 8.4).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| IDP_CREATED | UM of Jane; description, deadline, fileUrl | New record appears on Jane's S12, `completedAt: null` (open) | — |
| CREATE_DENIED | Viewer has no S12 RW and lacks `maintain_cds_records` | — | 403 |
| EDIT_OPEN | Manager edits description/deadline/fileUrl of an open IDP | Fields updated, `completedAt` untouched | — |
| EDIT_COMPLETED_BLOCKED | Manager edits an IDP with `completedAt` set | — | 409 |
| SELF_COMPLETE | Jane (employee) checks complete on her open IDP | `completedAt` = today, `deadline` unchanged | — |
| SELF_COMPLETE_TWICE | Jane completes an already-completed IDP | — | 409 |
| MANAGER_CANNOT_COMPLETE | Manager/PP calls the complete endpoint for Jane | — | 403 |
| SUBJECT_NOT_FOUND | `employeeId` in the path does not exist | — | 404 |
| IDP_NOT_FOUND | `idpId` does not exist, or exists but belongs to a different `employeeId` | — | 404 |
| UPDATE_REJECTS_COMPLETED_AT | Update payload includes a `completedAt` field on an otherwise-open IDP | — | 400 |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- add `IDPRecord` (`employeeId` FK with `onDelete: Restrict`, matching `RiskRecord`/`CDSAssessment`; `description` Text, `deadline` `@db.Date`, `fileUrl` String, `completedAt DateTime?`, `createdAt`/`updatedAt`) + `idpRecords IDPRecord[]` back-relation on `Employee` next to `cdsAssessments` (line 134); migration
- `services/backend/src/modules/cds/dto/create-idp-record.dto.ts`, `update-idp-record.dto.ts` (`extends PartialType(CreateIdpRecordDto)`), `is-idp-calendar-date.validator.ts` -- **new**; mirror `action-items/dto/create-action-item.dto.ts`'s `description`/`dueDate`/`link` style (trim `Transform`, calendar-date validator, `IsUrl({ require_protocol: true, protocols: ['http', 'https'] })` per `create-campaign.dto.ts`'s stricter variant) but `fileUrl` required, never a `completedAt` field; the calendar-date validator allows past dates (an already-missed deadline is still a valid one); an empty-body `PATCH` on an open record is a no-op 200, not a 400
- `services/backend/src/modules/cds/entities/cds-section.entity.ts` -- add `CdsIdpRecordEntity` (`id`, `description`, `deadline`, `fileUrl`, `completedAt: string | null`) and `idpRecords: CdsIdpRecordEntity[]` on `CdsSectionEntity`
- `services/backend/src/modules/cds/cds.service.ts` -- extend `buildSection` to also load IDP records (`orderBy: createdAt desc`, mirrors `loadAssessmentsForSubject:67-76`); add `createIdpRecord`, `updateIdpRecord` and `completeIdpRecord`, each using a guarded `updateMany` scoped `where: { id: idpId, employeeId: subjectEmployeeId, completedAt: null }` (409 on `count === 0` -- covers "already completed" and "idpId belongs to a different employee" atomically, and closes the same update/complete race for `updateIdpRecord` that `completeIdpRecord` already closes)
- `services/backend/src/modules/cds/idp-records.controller.ts` -- **new**, `@Controller('employees/:employeeId/idp-records')`; assert the subject employee exists (mirror `risks.controller.ts:104-112`) before every route; create/update carry `@UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))`, mirroring `mentorship.controller.ts` -- the global pipe (`bootstrap.ts`) only sets `whitelist`/`transform`, so this decorator is what actually makes Boundaries' "`completedAt` never accepted" hold; `POST` create + `PATCH :idpId` update gated per Boundaries; `POST :idpId/complete` (`HttpCode(200)`) gated self-only + `sectionGate.requireSection(viewer, employeeId, 'S12')` (no `minLevel` — Self's `R` suffices)
- `services/backend/src/modules/cds/cds.module.ts` -- add `controllers: [IdpRecordsController]`
- `services/backend/src/modules/cds/__tests__/*`, `test/employee-profile.e2e-spec.ts` -- extend for I/O matrix rows
- `services/frontend/src/types/employee-profile.ts:100-112` -- add `IdpRecord` + `idpRecords: IdpRecord[]` on `CdsSection`; `CreateIdpRecordPayload`/`UpdateIdpRecordPayload`
- `services/frontend/src/api/services/cds.service.ts` -- **new**, mirror `mentorship.service.ts`
- `services/frontend/src/api/hooks/useCds.ts`, `hooks/data/useCdsData.ts` -- **new**, mirror `useMentorshipMutations.ts`/`useMentorshipData.ts` (mutation + toast + invalidate the profile section query)
- `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/CdsSection.tsx` -- accept `employeeId`, `accessLevel`, `audienceRole`; render `idpRecords` (mirror `ManagementNotesSection.tsx`'s per-item inline-form pattern for RW viewers, disabled once `completedAt` set, + create form mirroring the local `AddRiskRecordForm` function defined inside `RisksSection.tsx`, not a standalone file) and a Self-only completion `Checkbox` per open record, gated on `audienceRole === 'Self'` alone -- S12 never grants Self `RW` (Self is always `R`, per Boundaries), so `MentorshipSection.tsx`'s `canEditFlag === 'RW'` condition does not transfer directly and would hide the checkbox from the only audience it's for
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx:171-173` -- pass `employeeId`/`accessLevel`/`audienceRole` into `CdsSectionCard` (mirror the `S13` registration at :163-169)
- `services/frontend/src/locales/en/translation.json:616-624` -- add `employeeProfile.s12.idp.*` keys

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration -- add `IDPRecord` model + back-relation -- foundation table
- [x] `services/backend/src/modules/cds/dto/*` -- create/update DTOs + calendar-date validator -- input validation
- [x] `services/backend/src/modules/cds/entities/cds-section.entity.ts` + `cds.service.ts` -- extend section payload + CRUD/complete methods -- delivers the read+write logic
- [x] `services/backend/src/modules/cds/idp-records.controller.ts` + `cds.module.ts` -- new endpoints, access-gated -- delivers the write surface
- [x] `services/backend/src/modules/cds/__tests__/*` + e2e extensions -- cover I/O matrix rows
- [x] `services/frontend/src/types/employee-profile.ts` + `api/services/cds.service.ts` + `api/hooks/useCds.ts` + `hooks/data/useCdsData.ts` -- typed client + mutations
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/CdsSection.tsx` + `profile-sections.tsx` + `translation.json` -- IDP list, create/edit form, self-complete checkbox

### Review Findings

- [x] [Review][Patch] Completed IDP rows lack positive status badge [`CdsSection.tsx`] — added `statusComplete` badge
- [x] [Review][Patch] Frontend Zod deadline validation weaker than backend [`idp-record-form.schema.ts`] — calendar-date refine added
- [x] [Review][Patch] Frontend fileUrl accepts non-http(s) protocols [`idp-record-form.schema.ts`] — http/https refine added
- [x] [Review][Patch] Duplicate add/edit Zod schemas [`idp-record-form.schema.ts`] — consolidated shared schema
- [x] [Review][Patch] Edit form submits unchanged values [`CdsSection.tsx`] — Save disabled when form not dirty
- [x] [Review][Patch] Complete checkbox re-clickable during in-flight request [`CdsSection.tsx`] — hide checkbox while completing
- [x] [Review][Patch] Unhandled promise rejection on complete failure [`useCdsSection.ts`] — try/catch added
- [x] [Review][Patch] Calendar date display timezone shift [`CdsSection.tsx`] — local calendar parsing for YYYY-MM-DD
- [x] [Review][Patch] `maintain_cds_records` UI write gate missing [`CdsSection.tsx`, `usePermissionsData.ts`] — wired permission fallback
- [x] [Review][Patch] Missing backend tests for I/O matrix rows [`cds.service.spec.ts`, `idp-records.controller.spec.ts`, `employee-profile.e2e-spec.ts`] — EDIT_OPEN, SELF_COMPLETE_TWICE, SUBJECT_NOT_FOUND, empty PATCH, create completedAt rejection, maintain allow, orderBy assertion
- [x] [Review][Defer] Playwright coverage for IDP UI [`services/frontend/e2e/`] — deferred, pre-existing gap (no profile section Playwright tests yet)
- [x] [Review][Defer] `idp-record-input.ts` duplicates DTO validation [`idp-record-input.ts`] — deferred, intentional service-layer normalization shared with validator

**Acceptance Criteria:**
- Given I am the Unit Manager of employee Jane, when I create an IDP with description, deadline 2026-12-01, and an external file link, then it appears on Jane's CDS section with no completion date — shown as open
- Given Jane has that open IDP, when Jane opens self-service and checks "complete", then the completion date is recorded as today and shown alongside the deadline; Jane has no control to create, edit, or delete IDP records — only the checkbox — and a shared-link viewer with S12 enabled never sees the checkbox at all

## Design Notes

`fileUrl` is required (not optional) at creation, consistent with spec-8-1's `resultLink` decision — CDS never stores partial records. Blocking edits once `completedAt` is set (rather than only blocking un-completion) keeps the "no reopen" default coherent: a completed IDP is a closed historical fact, matching how `ActionItem.cancelledReason` is set once and never revised. The create/update permission fallback (S12 `RW` OR `maintain_cds_records` + S12 not `none`) mirrors `risks.controller.ts` exactly. Both give an org-wide "maintain" role (e.g. HR Admin) a path even without a direct reporting/project-line/PP relationship to the subject.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- employee-profile.e2e-spec.ts` -- expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` -- expected: pass

**Manual checks (if no CLI):**
- As a seeded manager, open a report's profile, create an IDP, confirm it shows open with no completion date
- Switch to that employee's own session, check the IDP complete; confirm the completion date appears and the manager can no longer edit that record (edit form is disabled/blocked)

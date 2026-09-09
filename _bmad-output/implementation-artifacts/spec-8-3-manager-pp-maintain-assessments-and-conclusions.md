---
title: 'Manager/PP Maintain Assessments and Conclusions'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 0
story_key: '8-3-manager-pp-maintain-assessments-and-conclusions'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-8-context.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** S12's assessment log has been read-only since Story 8.1 — nobody can add a new assessment entry or correct a conclusion, so FR-38's write side is entirely undelivered.

**Approach:** Add two endpoints on the existing `CDSAssessment` model — manager/PP create (append-only) and a conclusion-only edit — reusing the exact S12 write gate Story 8.2 already established for IDP records (S12 `RW`, or `maintain_cds_records` + S12 not `none`).

## Boundaries & Constraints

**Always:**
- New `CdsAssessmentsController` at `employees/:employeeId/cds-assessments`, gated by an `assertCanCreateOrUpdate` copied from `idp-records.controller.ts:101-126` (S12 `RW`, else `maintain_cds_records` + S12 not `'none'`, else 403) — Colleague and unrelated Manager/PP get 403 server-side regardless of what the UI exposes.
- Create appends a new `CDSAssessment` row (`date`, `assessor`, `resultLink`, `conclusion` all required, no partial rows) — never modifies an existing entry; no numeric score/rating field anywhere (registry-only boundary, unchanged since 8.1).
- Edit endpoint accepts only `conclusion` (required, non-empty, trimmed) via a dedicated DTO — `date`/`assessor`/`resultLink` are immutable through this endpoint; no new log entry is created; `forbidNonWhitelisted` rejects any other field in the body (mirrors `idp-records.controller.ts`'s `ValidationPipe`).
- `assessor` stays free-text `String` — Story 8.1's Design Notes already resolved the "free text or person reference" choice for the model; reuse it, don't reopen.
- Both endpoints 404 when `employeeId` doesn't exist, or when `assessmentId` doesn't exist or belongs to a different employee (mirror `assertSubjectEmployeeExists` / `findIdpRecordForSubject`'s not-found handling).

**Ask First:** If future auditing needs to show when a conclusion was last edited, flag to the human before adding an `updatedAt` column — `CDSAssessment` currently has none.

**Never:** a delete endpoint for assessments (unscoped by this story); any numeric score/rating field; changing `assessor`'s field type; IDP record fields/endpoints (done, Story 8.2); directory filters (Story 8.4); any new field on `CDSAssessment`/`CdsAssessmentEntryEntity` beyond what's needed for the conclusion edit (e.g. an `editedBy`/editor-tracking field).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| ASSESSMENT_CREATED | PP of John; log has 1 prior entry; date, assessor, resultLink, conclusion | New entry appended, listed newest-first by `date` (unchanged from 8.1's sort — not by creation time); prior entry unchanged | — |
| CREATE_DENIED | Colleague or unrelated Manager/PP posts a new entry | — | 403 |
| CONCLUSION_EDITED | John's Unit Manager edits an existing entry's conclusion | `conclusion` updated; date/assessor/resultLink unchanged; no new entry created | — |
| EDIT_DENIED | Colleague or unrelated Manager/PP edits a conclusion | — | 403 |
| MAINTAIN_PERMISSION_ALLOWS | Viewer lacks S12 RW but holds `maintain_cds_records` and S12 is `R` | Create and edit both succeed | — |
| MAINTAIN_PERMISSION_NO_S12 | Viewer holds `maintain_cds_records` but has no S12 access to the subject at all (S12 is `none`) | — | 403 |
| SUBJECT_NOT_FOUND | `employeeId` in the path doesn't exist | — | 404 |
| ASSESSMENT_NOT_FOUND | `assessmentId` doesn't exist, or belongs to a different `employeeId` | — | 404 |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/cds/dto/create-cds-assessment.dto.ts` -- **new**; `date` (reuse `IsIdpCalendarDate`/`isValidIdpDeadline` from `is-idp-calendar-date.validator.ts` — same `cds` module as the new DTOs, format-only ISO-date check, not IDP-specific despite the name; no not-in-future constraint — open question, see Design Notes), `assessor` (string, trim, `@IsNotEmpty()`, `MaxLength(200)` — required per Boundaries' "all required, no partial rows"; 200 is a fresh cap, not mirrored from any existing field), `resultLink` (string, trim, `IsUrl({ require_protocol: true, protocols: ['http','https'] })`, `MaxLength(2048)`, mirrors `create-idp-record.dto.ts`'s `fileUrl`), `conclusion` (string, trim, `MaxLength(10000)`, mirrors `update-management-note.dto.ts`'s `content`)
- `services/backend/src/modules/cds/dto/update-cds-assessment-conclusion.dto.ts` -- **new**; single required `conclusion` field, same validators
- `services/backend/src/modules/cds/cds.service.ts` -- add `createAssessment(subjectEmployeeId, dto)` (plain `create`, mirrors `createIdpRecord`) and `updateAssessmentConclusion(subjectEmployeeId, assessmentId, dto)` (guarded `updateMany` scoped `where: { id, employeeId }`, mirrors `updateIdpRecord`'s atomic-guard shape but with no completed-lock branch and no zero-changed-fields no-op — `conclusion` is always required, so that branch is dead by construction and shouldn't be copied in; `count === 0` resolves via a `findFirst` 404 assertion); reuse existing `toAssessmentDto`. List ordering is unchanged from Story 8.1's `loadAssessmentsForSubject` (sorts by assessment `date` desc, then `createdAt`, then `id`) — a backfilled entry dated earlier than the current latest won't sort to the top even though it was just created
- `services/backend/src/modules/cds/cds-assessments.controller.ts` -- **new**, `@Controller('employees/:employeeId/cds-assessments')`; `POST` create + `PATCH :assessmentId` conclusion-edit, both carrying `@UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))`, `@Param('employeeId', ParseUUIDPipe)` / `@Param('assessmentId', ParseUUIDPipe)`, and the copied `assertCanCreateOrUpdate`/`assertSubjectEmployeeExists`/`resolveViewerEmployeeId` trio from `idp-records.controller.ts`
- `services/backend/src/modules/cds/cds.module.ts` -- add `CdsAssessmentsController` to `controllers`
- `services/backend/src/modules/cds/__tests__/*`, `test/employee-profile.e2e-spec.ts` -- extend for I/O matrix rows, plus: assessor rejected when whitespace-only; a `maintain_cds_records` holder with S12 `none` still gets 403; PATCH with an empty body gets 400
- `services/frontend/src/types/employee-profile.ts:100-107` -- add `CreateCdsAssessmentPayload`, `UpdateCdsAssessmentConclusionPayload`
- `services/frontend/src/api/services/cds.service.ts` -- add `createAssessment`, `updateAssessmentConclusion`
- `services/frontend/src/api/hooks/useCds.ts`, `hooks/data/useCdsData.ts` -- add create/update-conclusion mutations, mirror the existing IDP mutation hooks
- `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/CdsSection.tsx` -- extend `CdsAssessmentEntryItem` with an inline conclusion-edit affordance, gated by reusing the `canWriteIdp` flag/pattern `CdsSectionCard` already computes for assessments too; `CdsAssessmentEntryItem` currently takes only `{ entry }` and needs new `employeeId` + write-flag props threaded in to call the mutation and gate the control — plus an "add assessment" form mirroring `AddIdpRecordForm`
- `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/schemas/`, `hooks/useCdsSection.ts` -- add `createCdsAssessmentFormSchema`/`editCdsAssessmentConclusionFormSchema` + `useAddCdsAssessmentForm`/`useCdsAssessmentConclusionEdit`, mirroring the IDP form hooks — but for the conclusion-edit hook, resync the form with `values` alone (no extra `useEffect(() => form.reset(...))`); `useIdpRecordItem`'s combo of both is a pre-existing minor inconsistency with `react-forms.md`'s guidance, don't propagate it
- `services/frontend/src/locales/en/translation.json:616-624` -- add `employeeProfile.s12.addAssessment.*`, `.editConclusion.*` keys

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/cds/dto/create-cds-assessment.dto.ts` + `update-cds-assessment-conclusion.dto.ts` -- new DTOs -- input validation
- [x] `services/backend/src/modules/cds/cds.service.ts` -- `createAssessment` + `updateAssessmentConclusion` -- delivers the write logic
- [x] `services/backend/src/modules/cds/cds-assessments.controller.ts` + `cds.module.ts` -- new endpoints, access-gated -- delivers the write surface
- [x] `services/backend/src/modules/cds/__tests__/*` + e2e extensions -- cover I/O matrix rows
- [x] `services/frontend/src/types/employee-profile.ts` + `api/services/cds.service.ts` + `api/hooks/useCds.ts` + `hooks/data/useCdsData.ts` -- typed client + mutations
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/CdsSection/CdsSection.tsx` + `schemas/` + `hooks/useCdsSection.ts` + `translation.json` -- add-assessment form, inline conclusion edit

**Acceptance Criteria:**
- Given I am the PP assigned to employee John, whose log has one prior entry, when I add a new record with date, assessor, result link, and a conclusion, then it is appended — the prior entry stays unchanged, and no numeric score/rating field exists anywhere in the model
- Given John's assessment record has an existing conclusion, when his Unit Manager edits the conclusion text, then the date/assessor/result-link stay unchanged, no new log entry is created, and a Colleague or unrelated Manager/PP cannot write to this at all — rejected server-side even if the UI never exposes the control
- Given a viewer holds the `maintain_cds_records` permission but has no Manager/PP relationship to John at all (S12 resolves to `none`), when they attempt to create or edit an assessment, then the request is rejected with 403 — permission alone, without any S12 access, is not sufficient (per Boundaries' write gate)

## Design Notes

The edit endpoint intentionally accepts only `conclusion`, not a general partial-update DTO like IDP's — epics.md's AC frames this as "editing a conclusion," not "editing a record," and `CDSAssessment` has no `completedAt`-style lock concept, so there's no reason to open `date`/`assessor`/`resultLink` to post-creation edits. `updateAssessmentConclusion` mirrors `updateIdpRecord`'s `updateMany`-then-assert-404 shape, minus the "already completed" 409 branch IDP needed.

Both endpoints check subject-employee existence (404) before checking write authorization (403), inherited unchanged from `idp-records.controller.ts`'s established order — not a new decision for this story.

Concurrent conclusion edits are last-write-wins (no optimistic locking), consistent with `updateIdpRecord`'s existing behavior — not a new gap introduced here.

**Open question for the human (not encoded in validation):** `date` has no not-in-future constraint — a future-dated assessment would currently be accepted. Flag before shipping if that's unintended.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- employee-profile.e2e-spec.ts` -- expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` -- expected: pass

**Manual checks (if no CLI):**
- As a seeded PP, open a report's profile, add an assessment entry, confirm it appears newest-first above the prior entry
- As that report's Unit Manager, edit the conclusion of an existing entry; confirm date/assessor/result link are unchanged and no duplicate entry appears
- As a Colleague viewer (or an unrelated manager), confirm no create/edit affordance renders, and a direct API call is rejected with 403
- As an unrelated manager with a real `assessmentId` belonging to someone else, confirm the 403 response body doesn't leak whether that assessment or employee exists

### Review Findings

- [x] [Review][Patch] Remove future-date validation (spec open question) — removed from `cds-assessment-input.ts`, service/e2e tests, and frontend zod/`max` attribute
- [x] [Review][Patch] Add date trim + correct validation message on `CreateCdsAssessmentDto.date` [`create-cds-assessment.dto.ts`]
- [x] [Review][Patch] Add controller unit tests for `updateConclusion` RW and maintain+R success paths [`cds-assessments.controller.spec.ts`]
- [x] [Review][Patch] Expand e2e denial matrix: Self S12 R, unrelated manager POST+PATCH, maintain+none PATCH [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Add e2e for PATCH whitespace-only conclusion and non-existent assessmentId [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Use PP persona for `ASSESSMENT_CREATED`; restore `peoplePartnerId` after reassignment test pollution fix [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Align frontend date validation to UTC calendar dates; add `editConclusion.validation` i18n keys; `keepDirtyValues` on conclusion edit form
- [x] [Review][Defer] `forbidNonWhitelisted` 400 on PATCH extra fields — global `ValidationPipe` whitelist strips unknown fields before route pipe runs (same as IDP); not observable at e2e boundary [`bootstrap.ts`]
- [x] [Review][Defer] Frontend Playwright for CDS write UI — no profile/CDS e2e harness in frontend (pre-existing gap, same as stories 8.1/8.2)
- [x] [Review][Dismiss] Zero-width Unicode / `S12` undefined edge cases — theoretical, no reachable call site in access resolver
- [x] [Review][Dismiss] Duplicated `assertCanCreateOrUpdate` — intentional per spec ("copied from idp-records")

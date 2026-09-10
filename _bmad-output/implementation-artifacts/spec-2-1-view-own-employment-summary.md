---
title: 'View Own Employment Summary'
type: 'feature'
created: '2026-09-10'
status: 'done'
review_loop_iteration: 1
story_key: '2-1-view-own-employment-summary'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-2-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

<!-- Renegotiated 2026-09-10 (bmad-review): added the ReportingLine/PP RW-exposure clause below — registering the S4 provider is audience-agnostic and immediately surfaces the pre-existing S4:'RW' resolver grant (access-resolver.service.ts:54,107) for those roles with a read-only-only implementation. -->


## Intent

**Problem:** No `SectionProvider` exists for S4 (Employment). `GET /employees/:id/profile` already grants Self `R` on S4 (`access-resolver.service.ts` `SELF_SECTIONS.S4`), but the registry has no S4 registrant, so it always resolves `unavailable`. Four of S4's required fields — seniority, English level, probation status, contract type — have no Prisma column anywhere in the schema.

**Approach:** Add four nullable current-value columns to `Employee` (not temporal — AD-7's history-table set is limited to grade/position/department/employment-type/employment-status). Add an `EmploymentSectionProvider` (`S4`) that reads grade/position/employmentType via the existing `currentHistoryValue()` pattern plus the four new plain columns, and register it. Render S4 read-only on the frontend, reusing the inline key/value pattern already used for S9/S10/S11 — no access-control or write-path change; Self is already capped at `R` on S4.

## Boundaries & Constraints

**Always:**
- Provider reads grade/position/employmentType current value the same way `field-registry.service.ts`'s `loadEmployeeSnapshots` does: fetch all rows of `gradeHistory`/`positionHistory`/`employmentTypeHistory` for the subject via Prisma, then resolve each with `currentHistoryValue()` (`directory/employee-query.helpers.ts`) — never re-implement "pick the open or latest row."
- New `Employee` columns: `seniority String?`, `englishLevel String?`, `probationStatus String?`, `contractType String?` — plain nullable strings, no enum/domain validation this story (mirrors AD-7's own untyped-`String` history values).
- Section payload always includes all seven keys (`grade`, `position`, `seniority`, `employmentType`, `englishLevel`, `probationStatus`, `contractType`) as `string | null` — never omit a key because the value is unset. Frontend renders an explicit empty-state string per null field; the section itself is never hidden or errored for missing values.
- S4 stays `accessLevel: 'R'` for Self in every case — no write endpoint, no edit affordance, regardless of any other role (Manager/PP) the same person holds elsewhere. This is already enforced by `SELF_SECTIONS.S4 = 'R'` in `access-resolver.service.ts` (Stories 1.1–1.5); this story adds no access-control logic.
- Register `@RegisterProvider('section', 'S4')` in the `access` module, following `IdentitySectionProvider`/`ProjectsSectionProvider`'s constructor/DI shape exactly.
- Frontend adds `S4` to `PROFILE_SECTION_RENDERERS` and `PROFILE_SECTION_TITLE_KEYS` in `profile-sections.tsx` as an inline read-only renderer — no new component folder.
- The S4 renderer stays read-only for every audience this story, regardless of `accessLevel` — `access-resolver.service.ts` already grants ReportingLine/ProjectLine/PP `S4: 'RW'` (Stories 1.1–1.5), and registering this story's provider is the first time that pre-existing grant becomes visible in a profile response for those roles, since the registry lookup is audience-agnostic. No write endpoint exists for any audience yet; do not add edit UI based on `accessLevel` for S4 in this story.

**Ask First:**
- Whether `seniority`, `englishLevel`, `probationStatus`, `contractType` should be constrained to an enum/select rather than free text — `access-model.md` and the backlog reconciliation name the fields but never define a value domain. HALT and confirm before choosing anything stricter than nullable `String`; if no answer is given, default to plain nullable `String`.
- Whether `probationStatus` is a boolean-like state (on/off probation) or a richer value — source material never specifies. Default to nullable `String` (e.g. free-text status) absent clarification.

**Never:**
- Do not add a write endpoint, PATCH route, or edit affordance for any S4 field in this story — RW for Manager line/PP is a distinct future story, not part of 2.1's scope.
- Do not model the four new fields as effective-dated history tables — AD-7 explicitly limits its temporal set to grade/position/department/employment-type/employment-status; adding a fifth/sixth/seventh history table contradicts that decision.
- Do not surface `Employee.employmentStatus` (active/dismissed, CAP-14) as an S4 field — Story 2.1's acceptance criteria list exactly seven fields and this is not one of them; it already governs access-capping elsewhere in `AccessResolver`.
- Do not duplicate grade/position/employmentType's full change history inside S4 — that already belongs to S9 (career timeline, Epic 7); S4 shows current value only.
- Do not extend the Story 1.16 seed tool for the four new columns beyond what manual verification needs.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Self views own profile | Employee X requests `GET /employees/:X/profile` | 200; `sections.S4` present with `accessLevel: 'R'` and all seven fields populated or null | 401 unauthenticated |
| Unset field | `seniority` column is `null` for X | `sections.S4.data.seniority === null`; frontend shows an explicit empty-state label, field is not omitted | N/A |
| No temporal row yet | X has zero `GradeHistory` rows (pre-seed edge case) | `currentHistoryValue([])` returns `null` → `grade: null`, no throw | N/A |
| Dual-role viewer | Manager M (RW on S4 for reports elsewhere) views their own profile | `audience.role === 'Self'`, `sections.S4.accessLevel === 'R'` — unaffected by M's other-profile grants | N/A |
| Non-self viewer, S4 none | Colleague views subject B's profile | `COLLEAGUE_SECTION_GRANTS.S4 = 'none'` — S4 key omitted entirely from the response; unaffected by this story | N/A |
| Non-self viewer, S4 RW | ReportingLine/ProjectLine/PP views subject B's profile | `sections.S4.accessLevel === 'RW'` (pre-existing resolver grant) but the payload and rendering stay read-only — same shape as Self's `R`; this story adds no write path for any audience | 404 if a write is attempted, not 403 |

</frozen-after-approval>

## Code Map

**Schema**
- `services/backend/prisma/schema.prisma` — `Employee` model (~line 43): add `seniority String?`, `englishLevel String?`, `probationStatus String?`, `contractType String?`.

**Access module**
- `services/backend/src/modules/access/identity-section.provider.ts` — pattern reference: constructor DI, `@RegisterProvider('section', Sx)` shape, `getSection(viewerId, subjectId, audience)` signature.
- `services/backend/src/modules/access/employment-section.provider.ts` (new) — `@RegisterProvider('section', 'S4')`; fetches subject's `gradeHistory`/`positionHistory`/`employmentTypeHistory` plus the four new columns via `prisma.employee.findUnique`, resolves current values with `currentHistoryValue()`.
- `services/backend/src/modules/access/entities/employment-section.entity.ts` (new) — `EmploymentSectionDto`; all seven fields are `@ApiProperty({ nullable: true }) field!: string | null` (required-but-nullable) — do NOT copy `identity-section.entity.ts`'s `@ApiPropertyOptional`/`field?:` pattern, which marks fields as possibly-absent and would contradict "always includes all seven keys."
- `services/backend/src/modules/access/access.module.ts` — register `EmploymentSectionProvider` alongside the existing S1/S11 provider registrations.
- `services/backend/src/modules/access/profile-assembler.service.ts` — no change; `ALL_SECTION_IDS` already includes `S4`, registry lookup + assembly is generic.

**Reused helpers**
- `services/backend/src/modules/directory/employee-query.helpers.ts:90-103` — `currentHistoryValue(rows)`; `HistoryRowSnapshot` type.
- `services/backend/src/modules/directory/field-registry.service.ts:754-796` — `loadEmployeeSnapshots` as the reference query/resolve shape (fetch full history array per dimension, then `currentHistoryValue`).

**Tests**
- `services/backend/src/modules/access/__tests__/employment-section.provider.spec.ts` (new) — current-value resolution, null-field passthrough, populated-field passthrough, empty-history-array case.
- `services/backend/test/employee-profile.e2e-spec.ts` — extend with a Self `/profile` request asserting `S4` present with `accessLevel: 'R'` and all seven keys (one populated, one null, to cover both matrix rows), plus a ReportingLine/PP request asserting `accessLevel: 'RW'` with the same read-only payload shape.

**Frontend**
- `services/frontend/src/types/employee-profile.ts` — add `EmploymentSection` interface (`grade`, `position`, `seniority`, `employmentType`, `englishLevel`, `probationStatus`, `contractType`, all `string | null`).
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — add `S4` to `PROFILE_SECTION_TITLE_KEYS` and `PROFILE_SECTION_RENDERERS`. S9/S10/S11 render `<ul>` lists of array data — S4 is a flat object of seven scalar fields, so there is no existing renderer to copy; build a small local key/value list (e.g. a `<dl>` or label/value row per field) instead. `PROFILE_SECTION_ORDER` already includes `'S4'` — no change needed there. Give the section a title distinct from the existing `employeeProfile.employmentSection` string (already rendered as an "Employment" heading on this page for the unrelated Functional Roles form, `EmployeeProfilePage.tsx:106-116`) — e.g. `employeeProfile.sections.employment` = "Employment details" — to avoid two identically-labeled sections on one page.
- `services/frontend/src/locales/en/translation.json:611` — add `employeeProfile.sections.employment` plus per-field labels and a dedicated field-level empty-value string (e.g. `employeeProfile.sections.employment.notSet` = "Not set"). Do not reuse `employeeProfile.emptySection` ("No records yet.") — that copy is phrased for an empty list, not a single unset field.
- `services/frontend/e2e/employee-profile-assembly.spec.ts` — extend with an assertion that S4 renders read-only with all seven fields for the Self viewer, covering both a populated and a null field.

## Tasks & Acceptance

**Execution:**
- [x] Check `services/backend/prisma/schema.prisma` and `prisma/migrations/` for any of `seniority`/`englishLevel`/`probationStatus`/`contractType` already present -- guards against colliding with a prior partial attempt before generating a new migration
- [x] Confirm the plain-`String` default was applied for both Boundaries "Ask First" items (no stricter enum/domain type, no boolean `probationStatus`) since no clarification was requested -- keeps an audit trail for unattended execution
- [x] `services/backend/prisma/schema.prisma` -- add `seniority`, `englishLevel`, `probationStatus`, `contractType` nullable columns to `Employee` -- unblocks the provider; no schema exists for these fields today
- [x] `npm run db:migrate` (from `services/backend`) -- generate + apply the migration -- required before the provider can read the columns
- [x] `services/backend/src/modules/access/entities/employment-section.entity.ts` -- new `EmploymentSectionDto` -- wire-safe Swagger-documented shape for the S4 payload
- [x] `services/backend/src/modules/access/employment-section.provider.ts` -- new `EmploymentSectionProvider` (`S4`) -- resolves grade/position/employmentType via `currentHistoryValue`, passes through the four new columns, all fields nullable
- [x] `services/backend/src/modules/access/access.module.ts` -- register `EmploymentSectionProvider` -- makes it discoverable via the registry lookup the assembler already calls
- [x] Backend tests: `employment-section.provider.spec.ts` (unit) and extend `employee-profile.e2e-spec.ts` -- cover the I/O matrix rows, including the ReportingLine/PP `accessLevel: 'RW'` read-only-payload row
- [x] `services/frontend/src/types/employee-profile.ts` -- add `EmploymentSection` type -- typed consumption in the profile page
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` -- add S4 title key + read-only key/value renderer -- surfaces the section in the UI
- [x] `services/frontend/src/locales/en/translation.json` -- add S4 labels and a dedicated field-level empty-value string -- i18n discipline (UX-DR17)
- [x] `services/frontend/e2e/employee-profile-assembly.spec.ts` -- extend with S4 Self assertions (populated + null field) -- confirms read-only rendering end-to-end

### Review Findings

- [x] [Review][Patch] Self-contained S4 e2e fixture helper [test/employee-profile.e2e-spec.ts]
- [x] [Review][Patch] Dual-role Self S4 access cap test [test/employee-profile.e2e-spec.ts]
- [x] [Review][Patch] ProjectLine S4 RW accessLevel test [test/employee-profile.e2e-spec.ts]
- [x] [Review][Patch] Empty temporal history S4 test [test/employee-profile.e2e-spec.ts]
- [x] [Review][Patch] PATCH profile route returns 404 for S4 writes [test/employee-profile.e2e-spec.ts]
- [x] [Review][Patch] Unit test for latest effectiveFrom without open row [employment-section.provider.spec.ts]
- [x] [Review][Patch] Unit test asserting exactly seven S4 keys [employment-section.provider.spec.ts]
- [x] [Review][Patch] Treat empty-string field values as "Not set" [profile-sections.tsx]
- [x] [Review][Patch] Playwright asserts all seven S4 fields + section title [employee-profile-assembly.spec.ts]
- [x] [Review][Patch] Playwright RW ReportingLine read-only S4 test [employee-profile-assembly.spec.ts]
- [x] [Review][Patch] Align shared-link S4 label with profile title [translation.json]
- [x] [Review][Defer] Manual seed verification after `db:seed` — deferred, manual gate only
- [x] [Review][Defer] Wire `EmploymentSectionDto` into `EmployeeProfileEntity` OpenAPI — deferred, pre-existing `Record<string, unknown>` pattern for section payloads

**Acceptance Criteria:**
- Given I open my own profile, when the S4 section renders, then I see grade, position, seniority, employment type, English level, probation status, and contract type as read-only text, with no edit affordance *(matrix: Self views own profile)*
- Given a field in S4 has no value set, when the section renders, then it displays a clear empty state for that field rather than omitting the field or the section *(matrix: Unset field)*
- Given no write endpoint for S4 exists, when any client attempts to modify an S4 field for Self, then the request has no route to hit — 404, not a 403 from a guard *(Boundaries: Always — S4 stays R)*
- Given I hold RW on S4 for someone else as Manager/PP, when I view my own profile, then S4 is still `R` — my other-profile role never widens my Self access *(matrix: Dual-role viewer)*
- Given I am a ReportingLine/ProjectLine/PP viewer on someone else's profile, when S4 renders, then `accessLevel` is `'RW'` but the section shows as read-only with no edit affordance, and any write attempt has no route to hit — 404, not 403 *(matrix: Non-self viewer, S4 RW)*

## Design Notes

**Current-value resolution reuse.** The reason `field-registry.service.ts` fetches full history arrays instead of filtering `where: { effectiveTo: null }` in Prisma is pre-1.20-migration edge data — a row can be "current" by being the latest `effectiveFrom` even when no row is open. The new S4 provider follows the same shape (see Boundaries "Always" and Code Map "Reused helpers") to stay consistent with the one other place this data is read.

**`contractType` vs. `employmentType`.** These are distinct concepts, not duplicates: `employmentType` is one of AD-7's five temporal history dimensions (e.g. full-time/part-time, audited via `EmploymentTypeHistory`); `contractType` is a new, untyped, non-temporal column this story adds (e.g. employment contract vs. B2B/contractor) with no change history. Both surface side by side on S4 — worth keeping distinct in any future labeling/help text so they don't read as redundant.

**Schema migration is additive only.** All four new columns are nullable with no default beyond `null` — no backfill, no data migration required. Existing seeded employees (Story 1.16, 24 accounts) will simply show empty-state values until an HR Admin or a later write-path story populates them — verify this against the real seed data, not only e2e fixtures (see Verification).

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint` -- expect clean build/lint after the schema + provider changes
- `cd services/backend && npm run db:migrate` -- expect a new migration file generated and applied against the local dev DB
- `cd services/backend && npm test -- employment-section.provider` -- expect the new unit spec passing
- `cd services/backend && npm run test:e2e -- employee-profile` -- expect the extended e2e assertions passing
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` -- expect clean typecheck/lint/build after the new type + renderer
- `cd services/frontend && npm run test -- employee-profile-assembly` -- expect the extended Playwright assertion passing
- Manual: after `npm run db:seed`, open a real seeded employee's own profile and confirm S4 renders the empty-state copy for every unset field -- fixture-based e2e alone doesn't cover real seed data

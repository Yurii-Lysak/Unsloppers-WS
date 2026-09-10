---
title: 'Edit Own Personal and Emergency Contacts'
type: 'feature'
created: '2026-09-10'
status: 'done'
review_loop_iteration: 1
story_key: '2-2-edit-own-personal-and-emergency-contacts'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-2-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-2-1-view-own-employment-summary.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

<!-- Renegotiated 2026-09-10 (bmad-review): fixed MESSENGER values being wrongly forced through phone-format validation; corrected the "mirrors CDSAssessment/IDPRecord" claim to note the intentional Cascade-vs-Restrict divergence; clarified the type-conditional validation mechanism (per-field @ValidateIf on the DTO, not "service layer", since the global ValidationPipe runs first), relationship/label/contactPerson non-empty checks, no-uniqueness-constraint scope, last-write-wins concurrency for simultaneous Self/PP edits, type-change re-validation on update, and that employmentStatus doesn't gate S2/S3 writes; added PP_EDIT_S2, SELF_EDIT_S3, COLLEAGUE_WRITE_DENIED, and PROJECT_LINE_WRITE_DENIED matrix rows plus a malformed-ID/no-existence-leak clarification on SUBJECT_NOT_FOUND and ENTRY_NOT_FOUND; copy-edited two sentences for grammar/clarity. -->

<!-- Renegotiated 2026-09-10 (human confirmation via bmad-review): resolved the "Ask First" data-shape item — human confirmed the unified PersonalContactMethod model over three separate per-type models. Moved from Ask First into Always as a settled decision. -->

## Intent

**Problem:** No `SectionProvider` exists for S2 (Personal contacts) or S3 (Emergency contacts) — both always resolve `unavailable` even though `AccessResolver` already grants Self and PP `RW` on both (`access-resolver.service.ts` `SELF_SECTIONS`/`PP_SECTIONS`, Stories 1.1–1.5). No Prisma model backs either section, and this is the platform's first genuinely free-form Self-write path (contrast S12/S13's narrow checkbox/flag writes, and S1's still-deferred photo RW).

**Approach:** Add two new Prisma models — `PersonalContactMethod` (repeatable phone/email/messenger entries) and `EmergencyContact` (repeatable contact-person entries) — plus two plain nullable columns on `Employee` (`residentialAddress`, `placeOfStay`). Add a new `personal-contacts` module with `PersonalContactsSectionProvider` (S2) and `EmergencyContactsSectionProvider` (S3), each registered via `@RegisterProvider('section', Sx)` (mirrors `EmploymentSectionProvider`, spec-2-1). Add two controllers exposing add/edit/remove for the repeatable entries and a PATCH for the two address fields, gated by the existing `SectionAccessGate` — no `AccessResolver` change needed, since S2/S3 grants are already correct.

## Boundaries & Constraints

**Always:**
- Every write route calls `sectionGate.requireSection(viewerEmployeeId, employeeId, 'S2'|'S3', 'RW')` (`mentorship.controller.ts` pattern) — this alone correctly allows Self and PP and rejects ReportingLine/ProjectLine/Colleague, since the resolver already returns `R`/`none` for them. No extra `viewer === employeeId` check (unlike mentorship's self-only flag) — PP must also pass.
- `PersonalContactMethod`: `id`, `employeeId` FK (`onDelete: Cascade`), `type` (`PersonalContactMethodType`: `PHONE | EMAIL | MESSENGER`), `label` String (e.g. "Mobile", "Telegram"), `value` String, timestamps. `EmergencyContact`: `id`, `employeeId` FK (`onDelete: Cascade`), `contactPerson`, `relationship`, `phone` (all String), timestamps. Both are flat, single-FK shapes, mirroring `CDSAssessment`/`IDPRecord`'s table layout — not their delete behavior: those use `onDelete: Restrict`, these two intentionally use `Cascade` since contact/emergency-contact rows have no standalone value once the owning employee record is gone. No uniqueness constraint on `(employeeId, type, value)` or across `EmergencyContact` rows this story — duplicate entries are allowed.
- `value`/`phone` get format validation depending on type: `@IsEmail()` when `type === EMAIL`; `@IsPhoneNumber(undefined)` (region-less; `libphonenumber-js` is already a direct dependency of `class-validator` — see `package-lock.json`) for `PHONE` values and for `EmergencyContact.phone`. `MESSENGER` values get only a non-empty-string check (`@IsNotEmpty()`) — a messenger handle (e.g. a Telegram `@handle`) is not a phone number and must not be forced through phone-format validation. `label`, `contactPerson`, and `relationship` also get `@IsNotEmpty()`; `relationship` stays free text with no domain/enum constraint this story. Implement the `type`-conditional checks as per-field `@ValidateIf()` decorators on the DTO, not a single blanket decorator applied regardless of `type` — NestJS's global `ValidationPipe` runs before the controller method, so service-layer-only enforcement would let malformed values pass the pipe unvalidated. On update, changing `type` re-validates `value` against the new type's rule, not only the type at creation time.
- `residentialAddress`/`placeOfStay` are optional — never require them to save a contact-method or emergency-contact edit, and never require a contact-method or emergency-contact entry to exist to save an address field (`spec-2-1`'s "no field forces another to be set" precedent).
- S2/S3 section payloads always include `residentialAddress`/`placeOfStay` as `string | null` plus `contactMethods`/`contacts` as `[]` when empty — never omitted (S4's zero-key "unavailable" trap, spec-2-1).
- No caching of S2/S3 data beyond the existing per-request resolution — an edit must be visible on the very next fetch (access-model.md, no TTL).
- Concurrent writes to the same `PersonalContactMethod`/`EmergencyContact` row (e.g. Self and PP editing simultaneously) use last-write-wins — no optimistic concurrency check this story, unlike the access-switch fields in `access-model.md`'s D15; this is a deliberate scope decision, not an oversight.
- `sectionGate.requireSection` governs S2/S3 write access purely on the RW/R/none grant per audience — it does not additionally gate on the subject's `employmentStatus`; editing a departed (non-active) employee's contacts stays permitted unless a future story adds an explicit employment-status check.
- Data shape for phone/email/messenger is one unified repeatable `PersonalContactMethod` list (typed `PHONE`/`EMAIL`/`MESSENGER`), not three separate lists or three singular columns, per `backlog_review_draft.md`'s "should not assume exactly one of each" — confirmed by human review 2026-09-10; do not split into per-type models.

**Never:**
- No write endpoint may touch any other section (S1/S4/etc.) — this story only ever writes `PersonalContactMethod`, `EmergencyContact`, `residentialAddress`, `placeOfStay`.
- Never expose S2/S3 to Colleague or ProjectLine, and never add S3 to `shared-link-matrix.ts`'s shareable set — it is already structurally excluded there (Story 1.11/1.12); do not touch that file.
- No delete-cascade side effects beyond the two new tables — `onDelete: Cascade` is scoped to the employee's own rows only.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| SELF_EDIT_S2 | Employee adds/edits a `PersonalContactMethod` or PATCHes `residentialAddress` | 200/201; S2 reflects the change on next fetch for Self (RW), ReportingLine (R), PP (RW) | 400 on malformed phone/email |
| PP_EDIT_S2 | PP adds/edits a `PersonalContactMethod` for their assigned employee | 200/201; visible to the employee (RW) and PP (RW), read-only for manager line | 400 on malformed phone/email |
| PP_EDIT_S3 | PP adds an `EmergencyContact` for their assigned employee | 201; visible to the employee (RW) and PP (RW), read-only for manager line | — |
| SELF_EDIT_S3 | Employee adds/edits an `EmergencyContact` for themself | 200/201; S3 reflects the change on next fetch for Self (RW), ReportingLine (R), PP (RW) | — |
| REPORTING_LINE_WRITE_DENIED | ReportingLine manager attempts POST/PATCH on S2 or S3 | — | 403 (section is `R`, not `RW`) |
| COLLEAGUE_WRITE_DENIED | Colleague attempts POST/PATCH on S2 or S3 | — | 403 (section is `none`, not `RW`) |
| PROJECT_LINE_WRITE_DENIED | ProjectLine viewer attempts POST/PATCH on S2 or S3 | — | 403 (section is `none`, not `RW`) |
| COLLEAGUE_READ | Colleague requests the profile | S2/S3 keys absent entirely from the response | — |
| PROJECT_LINE_READ | ProjectLine viewer requests the profile | S2/S3 keys absent (resolver already grants `none`) | — |
| EMPTY_STATE | Employee has zero contact methods and zero emergency contacts | `contactMethods: []`, `contacts: []`, address fields `null` — section still renders, not omitted | — |
| SUBJECT_NOT_FOUND | `employeeId` in path does not exist | — | 404 if well-formed but missing; 400 if it fails `ParseUUIDPipe` before the lookup runs |
| ENTRY_NOT_FOUND | `contactId`/`emergencyContactId` does not exist or belongs to a different employee | — | 404 in both cases — a wrong-employee id returns the same 404 as a nonexistent one, never a distinct response that would leak the record's existence; 400 if the id fails `ParseUUIDPipe` |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — add `PersonalContactMethod`, `EmergencyContact` models + `PersonalContactMethodType` enum + `residentialAddress String?`/`placeOfStay String?` on `Employee` (~line 43) + back-relations; migration.
- `services/backend/src/modules/personal-contacts/` — **new module**, mirrors `mentorship/` layout: `personal-contacts-section.provider.ts` (S2, `@RegisterProvider('section','S2')`), `emergency-contacts-section.provider.ts` (S3, `@RegisterProvider('section','S3')`), `entities/`, `dto/create-contact-method.dto.ts`/`update-contact-method.dto.ts`/`create-emergency-contact.dto.ts`/`update-emergency-contact.dto.ts`/`update-address.dto.ts` (update DTOs use `PartialType(CreateXDto)`; a partial update still re-validates `type`+`value` together whenever either is supplied), `personal-contacts.controller.ts` (`@Controller('employees/:employeeId/personal-contacts')`, add/edit/remove routes for `PersonalContactMethod` plus `PATCH employees/:employeeId/personal-contacts/address` for the two address fields), `emergency-contacts.controller.ts` (`@Controller('employees/:employeeId/emergency-contacts')`), `personal-contacts.module.ts`.
- `services/backend/src/modules/access/identity-section.provider.ts`, `employment-section.provider.ts` — pattern reference for provider shape (constructor DI, `getSection`).
- `services/backend/src/modules/mentorship/mentorship.controller.ts` — pattern reference: `sectionGate.requireSection(...)` gating, `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` on writes.
- `services/backend/src/app.module.ts` — register `PersonalContactsModule`.
- `services/backend/test/employee-profile.e2e-spec.ts`, new `__tests__/*.spec.ts` under `personal-contacts/` — I/O matrix coverage.
- `services/frontend/src/types/employee-profile.ts` — add `PersonalContactsSection`, `EmergencyContactsSection`, `EmergencyContact`, `PersonalContactMethod` types (mirror `MentorshipSection`, L217).
- `services/frontend/src/api/services/personal-contacts.service.ts` — **new**, mirror `mentorship.service.ts`.
- `services/frontend/src/api/hooks/usePersonalContacts.ts`, `hooks/data/usePersonalContactsData.ts` — **new**, mirror `useMentorshipMutations.ts`/`useMentorshipData.ts`; `usePersonalContacts.ts` owns all S2 mutations, including the `residentialAddress`/`placeOfStay` PATCH, since it targets the same section and controller as the contact-method CRUD.
- `services/frontend/src/pages/EmployeeProfilePage/components/PersonalContactsSection/PersonalContactsSection.tsx`, `.../EmergencyContactsSection/EmergencyContactsSection.tsx` — **new**; mirror `ManagementNotesSection.tsx`'s per-item inline-form + bottom "add" form pattern for the repeatable lists, `S4`'s `<dl>` key/value block for `residentialAddress`/`placeOfStay`.
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — add `S2`/`S3` to `PROFILE_SECTION_TITLE_KEYS` and `PROFILE_SECTION_RENDERERS` (S2/S3 already present in `PROFILE_SECTION_ORDER`, L28-31 — no change there).
- `services/frontend/src/locales/en/translation.json:611` — add `employeeProfile.sections.personalContacts.*`, `employeeProfile.sections.emergencyContacts.*` keys.

## Tasks & Acceptance

**Execution:**
- [x] Confirm the unified `PersonalContactMethod` shape (one typed, repeatable list, not three separate models) — human-confirmed 2026-09-10 via bmad-review
- [x] `services/backend/prisma/schema.prisma` + migration — new models/enum/columns — foundation
- [x] `personal-contacts/dto/*` — create/update DTOs, conditional phone/email validation — input validation
- [x] `personal-contacts/*-section.provider.ts` + `entities/` — S2/S3 payload assembly — read path
- [x] `personal-contacts/*.controller.ts` + `personal-contacts.module.ts` + `app.module.ts` — add/edit/remove endpoints, RW-gated — write path
- [x] Backend `__tests__/*` + `employee-profile.e2e-spec.ts` extension — cover I/O matrix
- [x] `services/frontend/src/types/employee-profile.ts` + `api/services/personal-contacts.service.ts` + hooks — typed client
- [x] `PersonalContactsSection.tsx` + `EmergencyContactsSection.tsx` + `profile-sections.tsx` + `translation.json` — UI

### Review Findings

- [x] [Review][Patch] Fix type-only PATCH failing at ValidationPipe — `is-personal-contact-value.validator.ts`, `create-contact-method.dto.ts`
- [x] [Review][Patch] Trim/normalize contact `value` on create and whitespace-only address PATCH to null — `create-contact-method.dto.ts`, `update-address.dto.ts`
- [x] [Review][Patch] Reject null PATCH fields and map Prisma P2025 to 404 — `personal-contacts.service.ts`
- [x] [Review][Patch] Add missing I/O matrix e2e coverage (malformed UUID 400, S3 phone 400, type-change revalidation, emergency ENTRY_NOT_FOUND, reporting-line/PP read visibility) — `employee-profile.e2e-spec.ts`
- [x] [Review][Patch] Restore `managerId` after manager reassignment test to prevent downstream pollution — `employee-profile.e2e-spec.ts`
- [x] [Review][Patch] Add client-side phone format validation and i18n keys — `lib/phone.ts`, form schemas, `translation.json`
- [x] [Review][Defer] Playwright e2e for S2/S3 profile UI — deferred, pre-existing gap (no profile page e2e harness yet)
- [x] [Review][Defer] Dedicated service/controller unit tests beyond section providers — deferred, e2e matrix covers write paths

**Acceptance Criteria:**
- Given I edit my personal phone number and save, then S2 reflects the change immediately for me (RW) and my PP (RW), and read-only for my manager line
- Given I add/edit/remove an emergency contact and save, then S3 updates the same way, visible only to me (RW) and my PP (RW), never to a colleague, and read-only for manager line
- Given my profile is shared via a shared link, when the link is generated, then S3 cannot be enabled under any configuration and S2 defaults to excluded (pre-existing behavior, unchanged by this story)

## Design Notes

`PersonalContactMethod` unifies phone/email/messenger into one typed, repeatable table rather than three separate structures — a messenger and a phone number are the same shape (label + value); human-confirmed 2026-09-10 (see Boundaries). `residentialAddress`/`placeOfStay` stay plain scalar columns on `Employee` (like S4's four fields) since nothing in the source material suggests an employee has more than one current address.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run db:migrate` — expect clean build/lint, migration applies
- `cd services/backend && npm test -- personal-contacts && npm run test:e2e -- employee-profile` — expect new + extended specs passing
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expect clean after new types/components

**Manual checks (if no CLI):**
- As a seeded employee, open your own profile, add a messenger and an emergency contact, confirm both appear immediately; switch to that employee's manager and confirm both sections render read-only with no edit affordance

---
title: 'Record Feedback with a Visibility Flag'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '85ccfa9343dbcb3c6e49a5d8954f1d8aa5a4d56f'
backend_baseline_commit: '85ccfa9343dbcb3c6e49a5d8954f1d8aa5a4d56f'
frontend_baseline_commit: 'f09f554cceb90a2962c1382d6d4e3fe68f940a20'
story_key: '11-1-record-feedback-with-a-visibility-flag'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-11-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-9-management-notes-with-visibility-flags.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-5-1-record-a-risk.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C1 grants S8 (`RW` for ReportingLine/PP/ProjectLine, `R` for Self with record-level filtering, `none` for Colleague), but no `FeedbackRecord` model, `feedbacks` module, or S8 provider exists — profile assembly returns `status: 'unavailable'`. Managers and PPs cannot record feedback; employees have no leak path today only because data does not exist.

**Approach:** Add `FeedbackRecord` persistence, a `feedbacks` Nest module with `@RegisterProvider('section', 'S8')`, parallel CRUD routes gated by `SectionAccessGate` plus optional `CREATE_FEEDBACK` permission fallback, and an S8 profile card mirroring management-notes — single visibility flag, simpler filter matrix (no PM gate).

## Boundaries & Constraints

**Always:**
- **Model** (`FeedbackRecord`): `id`, `subjectEmployeeId`, `authorEmployeeId`, `recordedAt` (`@db.Date`), `context` (trimmed non-empty, max **500**), `body` (trimmed non-empty, max **10_000**), `sharedWithEmployee` (default `false`), `createdAt`, `updatedAt`. Index `subjectEmployeeId`. FKs → `Employee` `onDelete: Restrict`. `authorEmployeeId` set on create from viewer; never changed on PATCH.
- **CRUD authority:** any viewer with `audience.sections.S8 === 'RW'` may create, read, update, or delete **any** feedback record on that subject (not author-only).
- **Flag filtering in provider** (server-side):
  - `S8 === 'RW'`: return all records sorted `recordedAt` desc, `createdAt` desc, `id` asc; full DTO includes `sharedWithEmployee`.
  - `S8 === 'R'` + Self: return only `sharedWithEmployee === true` records; omit flag from each DTO.
  - Colleague / `none`: section omitted by assembler (no provider call).
  - **No** `visibleForPm`, **no** `hasHiddenNotes`, **no** PM Section Gate — S8 ProjectLine is full `RW` per access matrix.
- **S8 wire DTO:**
  ```typescript
  type FeedbackRecordReadDto = {
    id: string;
    recordedAt: string; // ISO date
    context: string;
    body: string;
    author: { id: string; displayName: string };
    createdAt: string;
    updatedAt: string;
  };
  type FeedbackRecordDto = FeedbackRecordReadDto & {
    sharedWithEmployee: boolean;
  };
  type FeedbackSectionDto = { records: FeedbackRecordReadDto[] | FeedbackRecordDto[] };
  ```
- **Author resolution:** when `authorEmployeeId` has no resolvable display name, emit `author.displayName: 'Unknown author'` (same as S7).
- **Create auth** (`POST`): `requireSection(S8, 'RW')` **or** (`hasPermission(CREATE_FEEDBACK)` **and** `audience.sections.S8 !== 'none'` **and** `audience.sections.S8 !== 'R'`). Self `R` + functional permission → **403**. Colleague-only → **403** even with functional permission.
- **Routes** (mirror `management-notes.controller.ts`; under `employees/:employeeId/feedbacks`):
  - Resolve viewer via `resolveViewerEmployeeId` — no linked `Employee` → **403** before gate.
  - `assertSubjectEmployeeExists` → **404** before gate; malformed UUID → **400** (`ParseUUIDPipe` on `employeeId` and `feedbackId`).
  - `GET` — `requireSection(S8)`; delegate to provider; **200** with `{ records: [...] }`.
  - `POST` — create auth above; body `{ recordedAt, context, body, sharedWithEmployee? }`; defaults flag `false`; trim `context` and `body`; reject empty/whitespace-only → **400**; `recordedAt` ISO `YYYY-MM-DD`, today or past only → **201**.
  - `PATCH /:feedbackId` — `requireSection(S8, 'RW')`; partial — at least one of `recordedAt`, `context`, `body`, `sharedWithEmployee`; reject empty body → **400**; `recordedAt` future date → **400**; **200** on success; **404** when id unknown or subject mismatch.
  - `DELETE /:feedbackId` — `requireSection(S8, 'RW')`; **204**; **404** as above.
  - Provider/DB failure on parallel `GET` → **503**; profile maps to `status: 'unavailable'`.
  - Multi-audience union (Rule 10): when union yields `S8: 'RW'`, apply the RW filter path — all records; no PM read carve-out.
- **Provider registration:** `FeedbacksModule` imported in `app.module.ts`.
- **Frontend:** wire `PROFILE_SECTION_RENDERERS.S8` + title key `employeeProfile.sections.feedback`. `accessLevel === 'RW'`: add form (`recordedAt`, context, body), record list, single `Switch` "Shared with employee" with immediate PATCH. `accessLevel === 'R'` (Self): section always present; read shared records only (`records: []` when none shared), no write, no flag on DTOs. Empty state copy: "No feedback recorded." Mutations invalidate profile query. Forms: `react-hook-form` + `zod`.
- **E2e:** `feedbacks.e2e-spec.ts` — epic AC rows, Self filter, colleague deny, profile vs parallel GET parity, RW edit by non-author, flag-only PATCH, validation 400s, provider failure 503. Use ad-hoc employees + `projectAssignment` + `peoplePartnerId` pattern from `timeline.e2e-spec.ts` (bootcamp seed has no PM/DM rows). Frontend `e2e/feedback-visibility.spec.ts` — stub profile, single toggle, Self sees only shared (no PM gate test).

**Ask First:**
- If note body needs rich text or attachments, HALT — plain text only in this story.

**Never:**
- Period-comparison UI or date-range picker (Story 11.2); "Request feedback…" campaign action (Story 11.3); dual PM/employee flags (S7 pattern); mentorship pair closing feedback (D6); seed feedback in bootcamp `seed.service.ts`; client-side-only access checks; auto-create feedback from campaigns.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Default visibility | Manager POST with flag omitted | **201**; record saved `sharedWithEmployee: false`; absent from B's Self profile | 401; 403 no Employee; 404 subject |
| PP shares record | PATCH `sharedWithEmployee: true` on existing record | **200**; B's next profile load includes that record in S8 | N/A |
| Self unshared | B GET own profile; record exists but unshared | S8 present with `accessLevel: 'R'`; `records` excludes unshared record; no `sharedWithEmployee` on DTOs | 401 |
| Self empty shared set | B GET own profile; no shared records exist | S8 present with `accessLevel: 'R'` and `records: []` | N/A |
| Self direct API | B GET `/employees/B/feedbacks` | Only shared records returned (`records: []` when none) | N/A |
| Self write | B POST/PATCH/DELETE | — | 403 via gate `RW` |
| PM full RW | PM (ProjectLine, S8 `RW`) GET/POST/PATCH/DELETE for subject B | All records; full CRUD on any record — **no** PM read carve-out (contrast S7) | 404 unknown subject/feedback |
| Colleague | Colleague GET/POST for B | — | 403 / S8 absent from profile |
| Functional perm, no C1 | Colleague + `CREATE_FEEDBACK` POST | — | 403 |
| Functional perm, Self R | B + `CREATE_FEEDBACK` POST own feedback | — | 403 (`S8` is `R`, not `RW`) |
| Future date | POST or PATCH `recordedAt` after today | — | 400 |
| Whitespace body | POST/PATCH `{ body: "   ", ... }` | — | 400 |
| Whitespace context | POST/PATCH `{ context: "   ", ... }` | — | 400 |
| PATCH empty body | PATCH `{}` | — | 400 |
| PATCH flag only | PATCH `{ sharedWithEmployee: true }` without other fields | **200**; flag updated | N/A |
| RW non-author edit | Manager PATCHes record authored by PP | **200**; any RW holder may edit any record | N/A |
| Wrong-subject id | PATCH feedback belonging to another employee | — | 404 |
| Viewer without Employee | Authenticated user, no linked `Employee` row | — | 403 before gate |
| Unknown employee | Valid UUID not in DB | — | 404 before gate |
| Malformed employeeId | Non-UUID `employeeId` | — | 400 |
| Malformed feedbackId | Non-UUID `feedbackId` on PATCH/DELETE | — | 400 |
| Multi-audience union | Viewer is PM ∪ ReportingLine for B | Union `S8: 'RW'`; all records (union least-restrictive) | N/A |
| Profile vs parallel GET | Same viewer/subject via `/profile` and `/feedbacks` | Identical filtered `records` | N/A |
| Author missing | `authorEmployeeId` has no resolvable display name | `author.displayName: 'Unknown author'` | N/A |
| Provider failure | DB error in provider | Profile: `sections.S8.status: 'unavailable'`; parallel GET: **503** | N/A |
| Dismissed subject writes | Subject employment `dismissed` (when C1 cap lands) | Reads per capped `R`; writes **403** | Deferred — see Design Notes |

</frozen-after-approval>

## Code Map

Normative behavior lives in **Boundaries** and **I/O & Edge-Case Matrix** above; paths below are execution hints only.

**New / modified (backend):**
- `services/backend/prisma/schema.prisma` — `FeedbackRecord` model + `Employee` relations; migration.
- `services/backend/src/modules/feedbacks/` (new) — mirror `management-notes/` layout: `feedbacks-section.provider.ts` (`@RegisterProvider('section', 'S8')`), `feedbacks.service.ts` (`buildSection`, CRUD, `toSectionDto` filter), `feedbacks.controller.ts`, DTOs, entities, `feedbacks.swagger.ts`, constants.
- `services/backend/src/app.module.ts` — import `FeedbacksModule`.
- `services/backend/test/feedbacks.e2e-spec.ts` (new) — adapt `management-notes.e2e-spec.ts` (drop PM-gate / dual-flag cases).
- `services/backend/src/modules/feedbacks/__tests__/feedbacks-section.provider.spec.ts` — filter matrix unit tests.

**New / modified (frontend):**
- `services/frontend/src/types/employee-profile.ts` — `FeedbackRecord`, `FeedbackSection`, payload types.
- `services/frontend/src/api/services/feedback.service.ts` + `api/hooks/useFeedbackMutations.ts` + `hooks/data/useFeedbackData.ts`.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/` — card, add form, list, single visibility `Switch`.
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — `S8` renderer + `employeeProfile.sections.feedback` title key.
- `services/frontend/src/locales/en/translation.json` — `employeeProfile.s8.*`.
- `services/frontend/e2e/feedback-visibility.spec.ts` (new).

**Existing infrastructure (reference only — do not modify unless required):**
- `services/backend/src/modules/access/access-resolver.service.ts` — S8 grants (`SELF_SECTIONS` `R`, reporting/PP/project `RW`; only S7 has PM carve-out at L555).
- `services/backend/src/modules/access/profile-assembler.service.ts` — provider loop; missing provider → `unavailable` L159–162.
- `services/backend/src/modules/management-notes/` — parallel-route + section-provider + flag-filter template (simplify for single flag).
- `services/backend/src/modules/risks/risks.controller.ts` — `assertCanCreate` permission fallback L79–102.
- `services/backend/src/modules/contracts/permission-keys.ts` — `CREATE_FEEDBACK` L18/L78–79.
- `services/frontend/src/pages/EmployeeProfilePage/components/ManagementNotesSection/` — UI pattern for list + toggle + forms.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` — `FeedbackRecord` + migration — persistence substrate
- [x] `services/backend/src/modules/feedbacks/feedbacks.service.ts` — `buildSection`, CRUD, Self/RW filter paths — core logic
- [x] `services/backend/src/modules/feedbacks/feedbacks-section.provider.ts` — `@RegisterProvider('section', 'S8')` — profile assembly
- [x] `services/backend/src/modules/feedbacks/feedbacks.controller.ts` — GET/POST/PATCH/DELETE + `assertCanCreate` — API surface
- [x] `services/backend/src/modules/feedbacks/feedbacks.module.ts` + `app.module.ts` — wire module — bootstrap
- [x] `services/backend/src/modules/feedbacks/__tests__/` — provider filter + service validation unit tests
- [x] `services/backend/test/feedbacks.e2e-spec.ts` — epic AC, Self filter, colleague deny, profile parity — backend verification
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/` — S8 card + add form + visibility toggle — UJ-1 UX
- [x] `services/frontend/src/api/services/feedback.service.ts` + mutation hooks — write path + cache invalidation
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — wire S8 renderer — profile integration
- [x] `services/frontend/e2e/feedback-visibility.spec.ts` — Self shared-only + RW toggle PATCH stub

**Acceptance Criteria:**
- Given I am the manager of employee B, when I create a feedback record about B with context "Q3 project retrospective" and a body leaving the visibility flag unset, then it saves as management-only and is not visible to B on B's own profile
- Given a management-only feedback record about B exists, when B's PP PATCHes `sharedWithEmployee` to true, then B can see that specific record on their own profile on the next load
- Given I am employee B, when I GET my own profile, then unshared feedback records are entirely absent from S8 `records` (section may still be present with `records: []`)
- Given I am a colleague of B, when I GET B's profile or `/employees/B/feedbacks`, then S8 is absent and the API returns 403

### Review Findings

- [x] [Review][Decision] Shared-link S8 filtering uses creator audience — **Resolved:** assembler now passes `responseAudience` to section providers; `FeedbacksService` filters all `S8: R` reads to `sharedWithEmployee` records (covers SharedLink recipients).

- [x] [Review][Decision] `CREATE_FEEDBACK` + Self `S8:R` POST auth — **Resolved:** matrix wins — functional permission requires `S8 === 'RW'`; Self `R` + permission → 403.

- [x] [Review][Patch] Deny Self POST via functional permission when `S8` is `R` [`feedbacks.controller.ts:145`]

- [x] [Review][Patch] Playwright colleague S8-absence test uses wrong test-id casing [`feedback-visibility.spec.ts:185`]

- [x] [Review][Patch] Backend e2e missing spec I/O matrix rows [`feedbacks.e2e-spec.ts`] — added coverage for permission denials, validation, Self PATCH/DELETE, union RW, no-employee 403, profile-after-share.

- [x] [Review][Patch] Visibility toggle appears stale until profile refetch [`FeedbackSection.tsx:101`] — optimistic local state in `useFeedbackItem`.

- [x] [Review][Patch] Add form allows future/invalid calendar dates client-side [`feedback-form.schema.ts:7`, `FeedbackSection.tsx:148`]

- [x] [Review][Patch] `PATCH { sharedWithEmployee: null }` may reach Prisma [`update-feedback-record.dto.ts:44-47`]

- [x] [Review][Patch] Swagger misdocuments RW section payload [`feedback-record.entity.ts:39-41`]

- [x] [Review][Defer] `formatFeedbackCalendarDate` uses `toISOString().slice(0,10)` [`feedback-input.ts:6`] — deferred, same UTC calendar pattern as other date fields; timezone edge cases pre-existing class of issue.

- [x] [Review][Defer] Unused `getFeedbacks()` client method [`feedback.service.ts:10`] — deferred, parallel GET not required for profile-only UX in 11.1.

## Spec Change Log

- 2026-09-08 — bmad-review: aligned I/O matrix with spec-1-9 template.
- 2026-09-08 — code review: assembler passes response audience; S8 R filter; CREATE_FEEDBACK auth; e2e/UX/swagger fixes.
- 2026-09-08 — e2e: fixed `recordedAt` dates for `DEFAULT_TEST_INSTANT` (2026-01-05); wrong-subject test grants PP on both employees.

## Design Notes

S8 is intentionally simpler than S7: one flag, no PM read carve-out, no Section Gate. Reuse management-notes file layout and risks' `assertCanCreate` pattern — swap `CREATE_EDIT_RISKS`/`S6` for `CREATE_FEEDBACK`/`S8`. Chronological list only in 11.1; do not scaffold period-comparison column layout (reserved for 11.2). Joining-interview feedback (epic context) has no separate record type in 11.1 — treat as a normal feedback record when that flow is specified. Dismissed-subject write cap follows the same deferred pattern as spec-1-9 until C1 employment-cap story lands.

## Verification

**Commands:**
- `cd services/backend && npm run test:e2e -- feedbacks.e2e-spec.ts` — expected: all pass
- `cd services/backend && npm test -- feedbacks` — expected: unit tests pass
- `cd services/frontend && npm run typecheck` — expected: clean
- `cd services/frontend && npm run test -- feedback-visibility` — expected: Playwright pass

**Manual checks (if no CLI):**
- Manager creates feedback on direct report with default flag → record visible in S8 for manager, absent for employee Self view
- Flip "Shared with employee" → employee Self profile shows that record immediately after refresh

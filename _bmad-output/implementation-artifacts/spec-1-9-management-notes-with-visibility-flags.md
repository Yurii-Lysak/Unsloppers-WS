---
title: 'Management Notes with Visibility Flags'
type: 'feature'
created: '2026-09-01'
status: 'done'
review_loop_iteration: 1
baseline_commit: '76ff65b4f6f2b306dc501cb2ac6830a1ffac42a0'
backend_baseline_commit: '5501c983a7aa2caa3252bb6ca6ee81a453230fba'
frontend_baseline_commit: '5bf345e66500df0818cca631574c81f61b773b53'
story_key: '1-9-management-notes-with-visibility-flags'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-8-enforce-the-colleague-whitelist-everywhere.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C1 grants S7 access (including the PM read-only carve-out) and the profile assembler returns `status: 'unavailable'` for S7, but no persistence, API, or UI exists for management notes or their dual visibility flags. Epic AC requires flag-gated reads on every surface — UI and direct API — with PM seeing only `visibleForPm` notes read-only while ReportingLine, ProjectLine DMs, and PP retain full RW on all notes.

**Approach:** Add a `ManagementNote` Prisma model and a `management-notes` Nest module with an `@RegisterProvider('section', 'S7')` that filters notes by `audience` (Self / ProjectLine PM-only `R` / full `RW`). Expose CRUD parallel routes gated by `SectionAccessGate`. Extend the profile SPA with an S7 card: note list, add/edit for RW viewers, dual visibility toggles, and a PM Section Gate when hidden notes exist (no count leak). Profile `GET` and parallel `GET /management-notes` both delegate to the same provider filter logic.

## Boundaries & Constraints

**Always:**
- **Model** (`ManagementNote`): `id`, `subjectEmployeeId`, `authorEmployeeId`, `content` (trimmed non-empty text, max **10_000** characters), `visibleForEmployee` (default `false`), `visibleForPm` (default `false`), `createdAt`, `updatedAt`. Index `subjectEmployeeId`. FK `subjectEmployeeId` → `Employee` and `authorEmployeeId` → `Employee`, both `onDelete: Restrict` — notes are not cascade-deleted when an employee row is removed; explicit cleanup belongs to the future `employment` departure cascade.
- **CRUD authority:** any viewer with `audience.sections.S7 === 'RW'` for the subject may create, read, update, or delete **any** note on that subject (not author-only). `authorEmployeeId` is set on create from the viewer and is never changed on PATCH.
- **Flag filtering in provider** (server-side, never fetch-then-strip in controllers):
  - `audience.sections.S7 === 'RW'` (ReportingLine, PP, ProjectLine DM): return **all** notes for subject, sorted `createdAt` descending, `id` ascending tiebreak.
  - `audience.sections.S7 === 'R'` + Self: return only `visibleForEmployee === true` notes (same sort).
  - `audience.sections.S7 === 'R'` + ProjectLine (PM-only — C1 resolved `S7: 'R'` because viewer matched `pmId` but not `dmId`): return only `visibleForPm === true` notes (same sort); set `hasHiddenNotes: true` when **any** note for the subject has `visibleForPm === false` (boolean only — no count, no ids). `hasHiddenNotes` is `false` when the subject has no notes. Omit `hasHiddenNotes` from non-PM responses.
  - Colleague / `none`: section omitted by assembler (no provider call).
  - Multi-audience union (Rule 10): when union yields `S7: 'RW'` (e.g. ProjectLine PM + ReportingLine or PP for the same subject), apply the RW path — all notes, no `hasHiddenNotes` gate.
- **S7 wire DTO** (profile + routes):
  ```typescript
  type ManagementNoteReadDto = {
    id: string;
    content: string;
    author: { id: string; displayName: string };
    createdAt: string; // ISO
    updatedAt: string;
  };
  type ManagementNoteDto = ManagementNoteReadDto & {
    visibleForEmployee: boolean;
    visibleForPm: boolean;
  };
  type ManagementNotesSectionDto = {
    notes: ManagementNoteReadDto[] | ManagementNoteDto[];
    hasHiddenNotes?: boolean; // PM-only gate signal; present only on PM R responses
  };
  ```
  All `accessLevel: 'R'` viewers (Self and PM): omit `visibleFor*` fields from each note DTO (read-only, no toggle affordance). `accessLevel: 'RW'` viewers receive full `ManagementNoteDto` and may PATCH flags.
- **Routes** (mirror `timeline.controller.ts`; all under `employees/:employeeId/management-notes`):
  - Resolve viewer via `resolveViewerEmployeeId` — authenticated user without linked `Employee` → **403** before gate (same as timeline/profile).
  - `assertSubjectEmployeeExists` → **404** for unknown `employeeId` **before** gate.
  - `ParseUUIDPipe` on `employeeId` and `noteId` → **400** for malformed UUIDs.
  - `GET` — `requireSection(S7)`; delegate to provider `getSection` (same code path as profile assembler).
  - `POST` — `requireSection(S7, 'RW')`; body `{ content, visibleForEmployee?, visibleForPm? }`; defaults flags `false`; trim `content`; reject empty/whitespace-only → **400**.
  - `PATCH /:noteId` — `requireSection(S7, 'RW')`; partial update — at least one of `content`, `visibleForEmployee`, `visibleForPm` required; **404** when note id unknown or `subjectEmployeeId` mismatch.
  - `DELETE /:noteId` — `requireSection(S7, 'RW')`; **404** as above.
  - Provider/DB failure on parallel `GET` → **503**; profile assembler maps the same failure to `status: 'unavailable'`.
- **Provider registration:** new `management-notes` module imported in `app.module.ts`; provider listed in module `providers`.
- **Profile assembler:** S7 registered → returns `{ accessLevel, data: ManagementNotesSectionDto }` instead of `unavailable`.
- **Frontend:** render S7 only when key present in profile `sections`. `accessLevel === 'RW'`: add note, edit content, dual toggles ("Visible to employee" / "Visible to PM") with immediate PATCH — both add-note and edit-note forms use `react-hook-form`, `@hookform/resolvers`, and `zod`. `accessLevel === 'R'` + PM: read flagged notes only; when `hasHiddenNotes`, show Section Gate per Design Notes — no content leak. Self `R`: read employee-flagged notes only, no write. Install `react-hook-form`, `@hookform/resolvers`, `zod` when implementing forms.
- **E2e:** new `management-notes.e2e-spec.ts` covering epic AC rows and matrix rows marked for backend e2e. Use ad-hoc employees + `projectAssignment` + `peoplePartnerId` pattern from `timeline.e2e-spec.ts` (bootcamp seed has no PM/DM rows).

**Ask First:**
- If note content needs rich text, attachments, or author-only edit restrictions, HALT — plain text only in this story; any section-RW holder may edit any note.

**Never:**
- Do not leak unflagged note content, ids, or counts to PM or Self via any route.
- Do not add a C8 functional-role permission gate for S7 reads or writes — section access (C1) is sufficient here. PRD §2.3: functional roles unlock **features**, not section visibility; `access-model.md` Rule 3 assigns S7 RW to ReportingLine, DM, and PP via the matrix alone. (Epic context's dual-axis rule applies where a feature permission exists; S7 has no separate C8 key.)
- Do not build S8 feedback flags (Epic 11), shared-link S7 exclusion tests (1.11), or cross-audience leak harness (1.14).
- Do not seed management notes in bootcamp `seed.service.ts` — e2e creates fixtures inline.
- Do not show Section Gate to RW viewers (ReportingLine, PP, ProjectLine DM, or any union that yields `S7: 'RW'`).

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PP creates unflagged note | PP POST note, both flags false | 201; note stored | 401; 403 no Employee; 404 subject |
| Self cannot see unflagged | Self GET profile or `/management-notes` for own id | S7 `notes` excludes unflagged note; no `visibleFor*` on returned DTOs | 401 |
| Self write denied | Self POST/PATCH/DELETE own `/management-notes` | — | 403 via gate `RW` |
| PM cannot see unflagged | PM (ProjectLine `R` only) GET for subject B | Only `visibleForPm` notes; `hasHiddenNotes: true` when any note has `visibleForPm: false` | 403 if no S7 grant |
| PM sees flagged only | One flagged + one unflagged note | `notes` length 1; flagged content only; `hasHiddenNotes: true` | N/A |
| PM gate empty list | Only unflagged notes exist | `notes: []`; `hasHiddenNotes: true`; gate renders | N/A |
| PM read-only | PM PATCH or DELETE or POST | — | 403 via gate `RW` |
| PM ∪ ReportingLine union | Viewer is PM on B and ReportingLine on B | Union `S7: 'RW'`; all notes; no `hasHiddenNotes`; no gate | N/A |
| PM ∪ PP union | Viewer is PM on B and PP on B | Union `S7: 'RW'`; all notes; no `hasHiddenNotes`; no gate | N/A |
| ReportingLine / PP / DM RW | Manager GET/POST/PATCH/DELETE | All notes; full CRUD on any note | 404 unknown subject/note |
| Self flagged read | Note with `visibleForEmployee: true` | Self sees note in S7; no write; no `visibleFor*` fields | N/A |
| Colleague denied | Colleague GET `/management-notes` or profile | No S7 key in profile; parallel route 403 | 403 |
| Viewer without Employee | Authenticated user, no linked `Employee` row | — | 403 before gate |
| Unknown employee | Valid UUID not in DB | — | 404 before gate |
| Malformed employeeId | Non-UUID `employeeId` | — | 400 |
| Malformed noteId | Non-UUID `noteId` on PATCH/DELETE | — | 400 |
| Empty / whitespace content | POST/PATCH `content: ''` or `'   '` | — | 400 validation |
| PATCH flags only | PATCH `{ visibleForPm: true }` without `content` | 200; flag updated | 400 when body empty |
| Note wrong subject | Valid `noteId` for different `subjectEmployeeId` | — | 404 |
| Multi-audience union | Viewer is ReportingLine ∪ PP for B | `RW`; all notes (union least-restrictive) | N/A |
| Profile vs parallel GET | Same viewer/subject via `/profile` and `/management-notes` | Identical filtered `notes` and `hasHiddenNotes` | N/A |
| Author missing | `authorEmployeeId` has no resolvable `User` | `author.displayName: 'Unknown author'` | N/A |
| Provider failure | DB error in provider | Profile: `sections.S7.status: 'unavailable'`; parallel GET: 503 | N/A |
| Dismissed subject writes | Subject employment `dismissed` (when C1 cap lands) | Reads per capped `R`; writes 403 | Deferred — see Design Notes |

</frozen-after-approval>

## Code Map

Normative behavior lives in **Boundaries** and **I/O & Edge-Case Matrix** above; paths below are execution hints only.

- `services/backend/prisma/schema.prisma` — add `ManagementNote` model + `Employee` relations (`onDelete: Restrict`).
- `services/backend/src/modules/management-notes/` (new) — module, service, controller, DTOs, entities, swagger.
- `services/backend/src/modules/management-notes/management-notes-section.provider.ts` — `@RegisterProvider('section', 'S7')`; flag filter + `hasHiddenNotes`.
- `services/backend/src/modules/access/section-access-gate.service.ts:48-67` — reuse `requireSection(..., 'S7'|'RW')`.
- `services/backend/src/modules/access/access-resolver.service.ts:357-360` — PM `R` vs DM `RW` on S7 (already done; provider consumes `audience.sections.S7`).
- `services/backend/src/modules/access/profile-assembler.service.ts:121-128` — S7 will resolve once provider registered.
- `services/backend/src/modules/contracts/section-provider.contract.ts` — extend only if `getSection` signature needs no change (pass `audience`).
- `services/backend/src/modules/timeline/timeline.controller.ts` — parallel-route + 404-before-gate + `resolveViewerEmployeeId` pattern to copy.
- `services/backend/test/timeline.e2e-spec.ts:110-129` — PM/DM/PP fixture pattern for e2e.
- `services/frontend/src/types/employee-profile.ts` — `ManagementNotesSection`, `ManagementNote` / `ManagementNoteRead` types.
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — add S7 renderer + title key.
- `services/frontend/src/pages/EmployeeProfilePage/components/ManagementNotesSection.tsx` — card, add/edit forms, toggles, gate.
- `services/frontend/src/api/hooks/` — `useManagementNotes` mutations (or inline in page hook if profile-driven only).

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — `ManagementNote` model with `onDelete: Restrict` FKs
- [x] `services/backend/src/modules/management-notes/management-notes.service.ts` — CRUD + query helpers; trim/length validation; author displayName with fallback
- [x] `services/backend/src/modules/management-notes/management-notes-section.provider.ts` — S7 provider with audience filtering, sort order, `hasHiddenNotes`
- [x] `services/backend/src/modules/management-notes/management-notes.controller.ts` — gated REST routes; 404-before-gate; 403 no-employee; `ParseUUIDPipe` on ids
- [x] `services/backend/src/modules/management-notes/management-notes.swagger.ts` — Swagger decorators for all routes
- [x] `services/backend/src/modules/management-notes/management-notes.module.ts` — wire module; import in `app.module.ts`
- [x] `services/backend/src/modules/management-notes/__tests__/management-notes-section.provider.spec.ts` — unit: Self/PM/RW filter matrix + `hasHiddenNotes` edge cases (no notes, all flagged, hidden-only)
- [x] `services/backend/test/management-notes.e2e-spec.ts` — matrix rows: PP unflagged, PM flagged-only, PM gate empty list, RW all, Self write denied, union PM+ReportingLine, profile vs parallel GET parity
- [x] `services/frontend/package.json` — add `react-hook-form`, `@hookform/resolvers`, `zod`
- [x] `services/frontend/src/types/employee-profile.ts` — S7 types aligned to wire DTO (`ManagementNoteRead` vs `ManagementNote`)
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/ManagementNotesSection.tsx` — card UI with add/edit forms, toggles, PM gate
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — register S7 renderer
- [x] `services/frontend/src/locales/en/translation.json` — keys: `employeeProfile.sections.managementNotes`, `employeeProfile.s7.gated`, `employeeProfile.s7.toggleVisibleToEmployee`, `employeeProfile.s7.toggleVisibleToPm`, `employeeProfile.s7.addNote`, `employeeProfile.s7.empty`
- [x] `services/frontend/e2e/management-notes-visibility.spec.ts` — PM gate (no count leak), RW toggles, colleague S7 absent

**Acceptance Criteria:**
- Given PP creates a note with both flags off, when B views own profile and B's PM calls S7 directly, then neither sees the note in UI or API *(matrix: PP creates unflagged / Self cannot see / PM cannot see)*
- Given PM is not ReportingLine/PP/DM and one note is flagged visible-for-PM while another is not, when PM opens S7, then PM sees exactly the flagged note read-only, `hasHiddenNotes: true`, gate when only hidden notes exist, and cannot POST/PATCH/DELETE *(matrix: PM sees flagged only / PM gate empty list / PM read-only)*
- Given ReportingLine, PP, or ProjectLine DM viewer, when they open S7, then all notes are visible and CRUD is allowed per `accessLevel: 'RW'` on any note *(matrix: ReportingLine / PP / DM RW)*
- Given a viewer holds PM `R` and ReportingLine or PP `RW` union on B, when they open S7, then all notes are visible with full CRUD and no gate *(matrix: PM ∪ ReportingLine / PM ∪ PP)*
- Given Colleague V and subject B, when V requests profile or `/management-notes`, then S7 is absent or 403 *(matrix: Colleague denied)*

### Review Findings

- [x] [Review][Patch] Clarify all `R` viewers omit `visibleFor*` DTO fields, not PM only [Boundaries: S7 wire DTO]
- [x] [Review][Patch] Add PM ∪ ReportingLine and PM ∪ PP union matrix rows [I/O matrix]
- [x] [Review][Patch] Specify any section-RW holder may edit/delete any note (not author-only) [Boundaries: CRUD authority]
- [x] [Review][Patch] Add max content length (10_000) and whitespace trim validation [Boundaries: Model / Routes]
- [x] [Review][Patch] Specify note sort order `createdAt` desc, `id` asc [Boundaries: flag filtering]
- [x] [Review][Patch] Define `hasHiddenNotes` edge semantics (exists unflagged-for-PM; false when no notes) [Boundaries / matrix: PM gate empty list]
- [x] [Review][Patch] Document dismissed-subject write cap as deferred until C1 employment cap lands [I/O matrix + Design Notes]
- [x] [Review][Patch] Require frontend e2e spec `management-notes-visibility.spec.ts` [Tasks]
- [x] [Review][Patch] Enumerate i18n keys including `employeeProfile.s7.gated` [Tasks]
- [x] [Review][Patch] Specify `onDelete: Restrict` FK lifecycle for notes [Boundaries: Model]
- [x] [Review][Patch] State add-note and edit-note both use react-hook-form + zod [Boundaries: Frontend]
- [x] [Review][Patch] Reconcile C8 dual-axis with PRD §2.3 section-only S7 gate [Boundaries: Never]
- [x] [Review][Patch] Add viewer-without-Employee → 403 matrix row [I/O matrix]
- [x] [Review][Patch] Add malformed `noteId` → 400 matrix row [I/O matrix / Routes]
- [x] [Review][Patch] Add Self write denied matrix row [I/O matrix]
- [x] [Review][Patch] Add whitespace-only content and PATCH-flags-only matrix rows [I/O matrix]
- [x] [Review][Patch] Add note wrong-subject → 404 matrix row [I/O matrix]
- [x] [Review][Patch] Add author-missing fallback displayName [I/O matrix / service task]
- [x] [Review][Patch] Add profile vs parallel GET parity matrix row [I/O matrix / Intent]
- [x] [Review][Patch] Add provider failure behavior (profile unavailable / parallel 503) [I/O matrix / Routes]
- [x] [Review][Patch] Add swagger task and expand unit/e2e coverage tasks [Tasks]
- [x] [Review][Patch] Merge PM gate UI behavior into Design Notes; Boundaries references it [Design Notes]
- [x] [Review][Patch] Note Code Map is non-normative; Boundaries + matrix are authoritative [Code Map preamble]
- [x] [Review][Patch] Add frontend e2e to Verification commands [Verification]
- [x] [Review][Patch] Use consistent `ReportingLine` / `ProjectLine` role labels [Intent / Boundaries]
- [x] [Review][Patch] Add visibility toggles to add-note form (spec requires dual toggles on create, not only on edit) [ManagementNotesSection.tsx:198]
- [x] [Review][Patch] Fix toggle UX — disable checkboxes while update mutation is pending to avoid stale controlled state and race conditions [ManagementNotesSection.tsx:142]
- [x] [Review][Patch] Add mutation error handling for create/update/delete (surface failures to user) [useManagementNotesMutations.ts:13]
- [x] [Review][Patch] Reset edit-note form when note content changes after profile refetch [ManagementNotesSection.tsx:108]
- [x] [Review][Patch] Replace `toSectionDto` fallback that returns all notes for unknown `R` audiences with explicit error [management-notes.service.ts:128]
- [x] [Review][Patch] Expand backend e2e to cover remaining matrix rows (PM∪PP union, DM RW CRUD, Self flagged read, PATCH flags-only, wrong-subject 404, malformed noteId 400, whitespace 400, viewer-without-Employee 403, successful PATCH/DELETE, provider failure 503) [management-notes.e2e-spec.ts]
- [x] [Review][Patch] Strengthen frontend e2e — exercise toggle PATCH and PM gate with empty note list [management-notes-visibility.spec.ts]
- [x] [Review][Patch] Add unit tests for `createNote`, `updateNote`, `deleteNote`, and `assertPatchHasFields` [management-notes.service.ts]
- [x] [Review][Patch] Add Swagger `ApiBadRequestResponse` for create/update routes [management-notes.swagger.ts]
- [x] [Review][Patch] Map zod validation errors to i18n keys instead of raw English messages [ManagementNotesSection.tsx:135]
- [x] [Review][Patch] Add `@IsString()` on DTO `content` fields before `@Transform` trim [create-management-note.dto.ts:7]

## Design Notes

**PM gate vs absent section:** S7 is granted (`R`) for ProjectLine PM when they have project-line access — the section card renders. `hasHiddenNotes` drives the gate banner per EXPERIENCE.md (`employeeProfile.s7.gated`: "A management note exists here. Not shared with your role."); flagged notes still list below. When no flagged notes and only hidden ones exist, card shows gate with an empty list. RW viewers (ReportingLine, PP, ProjectLine DM, or any union yielding `RW`) never see the gate.

**Author display:** Resolve `author.displayName` from `Employee` → `User.name` (fallback email, then `'Unknown author'`) in service layer — same pattern as identity section.

**Dismissed employment cap:** When the `employment` module applies C1's dismissed cap (epic-1-context), S7 reads follow the capped `R` grant and writes return 403. No implementation in this story until that C1 hook exists — matrix row documents expected behavior.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run depcruise` — expected: pass
- `cd services/backend && npm test -- management-notes` — expected: unit specs pass
- `cd services/backend && npm run test:e2e -- management-notes` — expected: matrix e2e pass
- `cd services/frontend && npm run build && npm run lint` — expected: pass
- `cd services/frontend && npm run test -- management-notes-visibility` — expected: S7 PM gate, RW toggles, colleague absence pass; existing e2e unchanged

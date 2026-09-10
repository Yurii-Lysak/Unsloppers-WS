---
title: 'Upload Photo and Certificates'
type: 'feature'
created: '2026-09-10'
status: 'done'
review_loop_iteration: 0
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-2-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-2-2-edit-own-personal-and-emergency-contacts.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<!-- Renegotiated 2026-09-10 (bmad-review, full review before implementation started): fixed a real access-control leak — DocumentsSectionProvider must filter Document rows by audience the same way IdentitySectionProvider filters mentor, so ProjectLine's CV+certs-only restriction applies to the S5 list payload, not only the file-download endpoint; aligned the file-download 403 to a 404 for ProjectLine's restricted-type case to close a matching existence-leak through the HTTP status code; clarified that POST write routes call sectionGate.requireSection (R-level) in addition to the self-only check, mirroring idp-records.controller.ts's double-check convention; added the missing multer/@types/multer dependency and a multer-level fileSize limit (DoS hardening) to the Code Map; added magic-byte content validation alongside the MIME/extension allowlist; clarified DocumentType is validated via @IsEnum on the DTO (client-supplied, not silently coerced); added originalFilename length capping; clarified 404 handling for GET photo with no photoStorageKey set and for a missing on-disk file (ENOENT); clarified old-file-delete-on-replace is best-effort and that concurrent same-employee uploads may orphan a file as an accepted scope limitation (mirrors spec-2-2's last-write-wins precedent); scoped shared-link viewers out of the new streaming endpoints this story (Never); scoped photo removal (revert-to-no-photo) out of this story (Never); clarified OTHER_EMPLOYEE_UPLOAD_DENIED and DOCUMENT_NOT_FOUND matrix rows; promoted the self-only write rule to lead the Always list and condensed a duplicated rationale, per the structure/prose passes; several small wording fixes. -->

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** S1's photo field and S5 (Documents) have no write path at all — `IdentitySectionProvider` explicitly stubs photo out ("deferred until their owning stories add Prisma columns"), and no `SectionProvider` exists for S5, so S5 always resolves `unavailable` even though PP already holds `RW` there. No file-storage mechanism (disk, object storage, or otherwise) exists anywhere in the backend yet. *(Note: epics.md tags this story `FR-15`, but PRD `FR-15` is Epic 3's "colleague mode of the list" — the correct requirement is PRD `FR-18`.)*

**Approach:** Add a small local-disk `FileStorageService` (no cloud infra exists in this bootcamp environment — `docker-compose.yml` only provisions Postgres), reused by both a new S1 photo write path in `IdentitySectionProvider`'s module and a new `documents` module owning S5 (`DocumentsSectionProvider`, mirrors `PersonalContactsSectionProvider`/spec-2-2). Both writes gate on Self-only (`viewer === employeeId`) plus the underlying `R`-level section grant, following a self-only-check pattern (see Code Map) — not the section-RW gate, since Self's S1/S5 grants are `R` with narrow field/type exceptions, not `RW`.

## Boundaries & Constraints

**Always:**
- `POST employees/:employeeId/identity/photo` (multipart, `FileInterceptor`) and `POST employees/:employeeId/documents` (multipart, `type` fixed to `CERTIFICATE`): both require `viewer === employeeId` (`ForbiddenException` otherwise, mentorship-controller pattern) — applied uniformly, including against a Full profile access holder or any PP/manager relationship; no PP/manager/full-access write path this story (S1/S5's broader `RW¹`/`RW` grants for ReportingLine/PP/ProjectLine belong to a future story, not this one). After the self-only check, each route also calls `sectionGate.requireSection(viewerEmployeeId, employeeId, 'S1'|'S5', 'R')`, mirroring `idp-records.controller.ts`'s `complete()` double-check convention, even though Self always holds at least `R` on both sections.
- `Employee.photoStorageKey String?` (nullable) — new column. `Document` model: `id`, `employeeId` FK (`onDelete: Cascade`, mirrors `PersonalContactMethod` — no standalone value once the employee is gone), `type` (`DocumentType`: `CONTRACT | W8 | COOPERATION_FORM | DIIA_CITY | CV | CERTIFICATE`, full S5 catalog per access-model.md even though only `CERTIFICATE` is writable this story — avoids a future migration), `originalFilename` (`@db.VarChar(255)` — sourced from multer's `file.originalname`, not a JSON DTO field, so the length cap is enforced by truncating/rejecting in `FileStorageService`/the controller, not via a class-validator decorator; stored verbatim otherwise, since only the on-disk storage key is UUID-generated), `storageKey`, `mimeType`, `sizeBytes Int`, `uploadedAt DateTime @default(now())`. Both additions carry a `// Story 2.3 — ...` schema comment, matching every prior field/model addition's convention in `schema.prisma`.
- `FileStorageService` (new, `src/modules/storage/`): writes to `UPLOADS_DIR` env var (default `./uploads`, gitignored), generates a UUID-based filename (never trusts the client's original filename for the on-disk path — prevents path traversal), validates extension+MIME allowlist, size, and actual file content (magic-byte signature check for JPEG/PNG/WEBP/PDF — the client-declared MIME type and extension are both spoofable, so the allowlist alone isn't sufficient) before writing. Photo: `image/jpeg|png|webp`, max 5MB. Certificate: `application/pdf`, `image/jpeg|png`, max 10MB (PRD FR-18 note: source states no limits; sane, conventional defaults apply). `FileInterceptor` is additionally configured with a matching `limits: { fileSize }` so multer itself rejects an oversized body mid-stream, not only after `FileStorageService`'s post-hoc check (unbounded buffering is a memory/disk-exhaustion DoS vector on an any-employee-reachable endpoint).
- Files are served through access-gated streaming endpoints, never static hosting — `GET employees/:employeeId/identity/photo` gated via `sectionGate.requireSection(..., 'S1', 'R')` (broad: Self/ReportingLine/ProjectLine/PP/Colleague/shared-link all hold at least `R` on S1); returns 404 when the subject has no `photoStorageKey` set. `GET employees/:employeeId/documents/:documentId/file` gated the same way at `'S5'` (narrow: Colleague `none`, ProjectLine sees CV+certs only — the section gate only checks the section-level grant, not per-type, so the controller additionally treats a ProjectLine request for a non-CV/non-certificate document as a 404, not a 403, matching `DOCUMENT_NOT_FOUND`'s no-existence-leak rule rather than revealing that a forbidden-type document exists). Both streaming endpoints treat a row whose on-disk file is missing (`ENOENT`) as a 404, not an unhandled error.
- `IdentitySectionDto` gains `photoUrl?: string | null` — the download endpoint path when `photoStorageKey` is set, `null` otherwise; never the raw storage key.
- `DocumentsSectionProvider` (S5) returns the subject's own `Document` rows (all types, not certificate-only — a future story may populate CV/contract/etc.) as `{ id, type, originalFilename, uploadedAt, downloadUrl }[]`, **filtered by the resolved audience the same way `IdentitySectionProvider` filters `mentor`** — a ProjectLine viewer's list contains only `CV`/`CERTIFICATE` rows, since the section-list payload is a surface that access-model.md Rule 1 covers just as much as the file-download endpoint (Colleague never reaches this provider at all, since S5 is `none` for Colleague); empty array when none (S4's empty-state precedent, spec-2-1/2-2).
- Any request to upload/edit a non-`CERTIFICATE` S5 document type, or edit any S1 field other than photo, is rejected with 400/403 (AC). `type` on `POST employees/:employeeId/documents` is accepted as client input and validated with `@IsEnum(DocumentType)` on the create-document DTO, so an out-of-enum value 400s via the global `ValidationPipe` the same way every other type-conditional check in this codebase does — the server never silently coerces a client-supplied non-`CERTIFICATE` value, it rejects it.
- Replacing the photo deletes the old file from disk (via `FileStorageService`) after the new `photoStorageKey` is persisted — best-effort: a delete failure (missing file, permission error) is logged and does not fail the response, since the DB write already succeeded. Concurrent uploads from the same Self viewer (e.g. a double-click) may race and leave one of the two newly-written files unreferenced on disk — an accepted, deliberate scope limitation for this story (no per-employee upload lock this story), mirroring spec-2-2's last-write-wins precedent for concurrent S2/S3 writes, not a guarantee this story provides beyond the single-request case (AC: "replaces my prior photo immediately").

**Never:**
- No delete/remove endpoint for uploaded certificates this story — AC only covers upload; deletion is a future story's scope.
- No photo-removal endpoint this story either — S1's photo `RW` is scoped to upload/replace only; reverting `photoStorageKey` to `null` is out of scope, mirroring the certificate no-delete boundary above.
- No write endpoint may touch any other section — mirrors spec-2-2's boundary.
- No shared-link auth path for the new streaming endpoints this story. `GET identity/photo` needs none — S1 is `R` for organic Colleague already, so a shared-link recipient (always an authenticated employee, per access-model.md) reaches it via their ordinary Colleague resolution regardless of the link's own S1 setting. `GET documents/:documentId/file` is the real gap: S5 is `none` for organic Colleague, so `sectionGate.requireSection` 403s a shared-link recipient even when their link's `cfg` grants S5 — `sectionGate.requireSection` only resolves organic Self/ReportingLine/ProjectLine/PP/Colleague relationships, never a shared-link token. A shared-link recipient's profile JSON may list a `downloadUrl` that 403s if clicked; reconciling `SectionAccessGate` with `SharedLinkService`'s separate audience-resolution path is a future story's scope, not this one's.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| SELF_UPLOAD_PHOTO | Employee POSTs a valid JPEG to their own `identity/photo` | 200/201; old file deleted, `photoUrl` updated, visible wherever S1 renders for any entitled audience | 400 on invalid type/size |
| SELF_UPLOAD_CERTIFICATE | Employee POSTs a valid PDF to their own `documents` (type `CERTIFICATE`) | 201; appears in their own S5 list immediately | 400 on invalid type/size |
| NON_CERTIFICATE_TYPE_REJECTED | Employee attempts to POST `type: CV` (or any non-certificate, including a value outside the `DocumentType` enum) to `documents` | — | 400 |
| MISSING_FILE_IN_UPLOAD | Employee POSTs to `identity/photo` or `documents` with no file field present in the multipart body | — | 400 |
| SELF_EDIT_S1_NON_PHOTO_REJECTED | Employee attempts to PATCH any other S1 field | — | 403/404 (no such route exists — S1 has no general write endpoint) |
| OTHER_EMPLOYEE_UPLOAD_DENIED | Employee (including a Full profile access holder) attempts to POST photo/certificate for a different `employeeId` (including their own PP/manager) | — | 403 (`viewer !== employeeId`, applied uniformly regardless of any other role the viewer holds) |
| PROJECT_LINE_LIST_FILTERED | ProjectLine viewer GETs the subject's profile (S5 section) | S5 list contains only `CV`/`CERTIFICATE` rows — other types are entirely absent from the array, not merely undownloadable | — |
| PROJECT_LINE_READ_CERT_ONLY | ProjectLine viewer GETs a non-certificate, non-CV document's file | — | 404, not 403 — same no-existence-leak rule as `DOCUMENT_NOT_FOUND` (S5 grant for ProjectLine is CV+certs only) |
| COLLEAGUE_READ_PHOTO | Colleague GETs `identity/photo` | 200 (S1 is `R` for Colleague) | — |
| COLLEAGUE_READ_DOCUMENT | Colleague GETs a `documents/:id/file` | — | 403 (S5 is `none` for Colleague) |
| PHOTO_NOT_SET | Entitled viewer GETs `identity/photo`, subject has no `photoStorageKey` | — | 404 |
| SUBJECT_NOT_FOUND | `employeeId` does not exist | — | 404 if well-formed but missing; 400 if it fails `ParseUUIDPipe` |
| DOCUMENT_NOT_FOUND | `documentId` does not exist or belongs to a different employee | — | 404 in both cases, no existence leak (spec-2-2 precedent); 400 if `documentId` fails `ParseUUIDPipe` before the lookup runs |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — add `Employee.photoStorageKey String?` (~line 56, alongside S2's address columns), `Document` model + `DocumentType` enum, back-relation; migration. Both additions carry a `// Story 2.3 — ...` comment per existing convention.
- `services/backend/package.json` — add `multer` to `dependencies` and `@types/multer` to `devDependencies` (pin to versions compatible with `@nestjs/platform-express`'s bundled multer; today the package is present only as an undeclared transitive dependency, and `@types/multer` isn't installed at all — `Express.Multer.File` won't typecheck under this repo's full-strict TS without it).
- `services/backend/src/modules/storage/` — **new module**: `file-storage.service.ts` (save/delete/read-stream by key, extension+MIME+size+magic-byte validation), `storage.module.ts` (exported, imported by `access` and `documents` modules).
- `services/backend/src/modules/access/identity-section.provider.ts` — resolve `photoUrl` from `photoStorageKey`.
- `services/backend/src/modules/access/identity.controller.ts` — **new**: `@Controller('employees/:employeeId/identity')`, `POST photo` (`FileInterceptor('photo', { limits: { fileSize: 5 * 1024 * 1024 } })`), `GET photo` (stream; 404 when unset or file missing on disk).
- `services/backend/src/modules/access/entities/identity-section.entity.ts` — add `photoUrl`.
- `services/backend/src/modules/documents/` — **new module**, mirrors `personal-contacts/`: `documents-section.provider.ts` (S5, `@RegisterProvider('section','S5')`, filters rows by `audience.role` — CV/CERTIFICATE only for ProjectLine, mirroring `identity-section.provider.ts`'s `MENTOR_VISIBLE_ROLES` filtering pattern), `documents.controller.ts` (`@Controller('employees/:employeeId/documents')`, `POST` upload cert with `@IsEnum(DocumentType)`-validated `type` and a matching 10MB `FileInterceptor` limit, `GET :documentId/file` stream — 404, not 403, for a ProjectLine-restricted type or a missing on-disk file), `documents.service.ts`, `entities/document.entity.ts`, `documents.module.ts`.
- `services/backend/src/modules/mentorship/mentorship.controller.ts`, `src/modules/cds/idp-records.controller.ts` (`complete()` method) — pattern reference: self-only check + `sectionGate.requireSection` at `'R'`, not `'RW'` — both checks present, not either/or.
- `services/backend/src/app.module.ts` — register `StorageModule`, `DocumentsModule`.
- `services/backend/.env.example` — document `UPLOADS_DIR` (optional, default `./uploads`).
- `services/backend/.gitignore` — add `uploads/`.
- `services/backend/test/employee-profile.e2e-spec.ts`, new `__tests__/*.spec.ts` under `storage/` and `documents/` — I/O matrix coverage.
- `services/frontend/src/types/employee-profile.ts` — add `photoUrl?: string | null` to `IdentitySection` (L208); add `DocumentsSection`, `DocumentRecord`, `DocumentType` types.
- `services/frontend/src/api/services/identity.service.ts`, `documents.service.ts` — **new**, `FormData` multipart POST (mirror `mentorshipApiService`'s `apiClient` usage).
- `services/frontend/src/api/hooks/`, `hooks/data/` — **new** upload hooks mirroring `usePersonalContacts.ts`.
- `services/frontend/src/pages/EmployeeProfilePage/components/ProfileHeader/ProfileHeader.tsx` — add avatar image (`photoUrl` else placeholder) + upload control, gated on a new `audienceRole` prop (`=== 'Self'`) threaded from `EmployeeProfilePage.tsx` (`employeeProfile.audience.role`, L72).
- `services/frontend/src/pages/EmployeeProfilePage/components/DocumentsSection/DocumentsSection.tsx` — **new**; list + certificate-only upload control.
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — add `S5` to `PROFILE_SECTION_TITLE_KEYS`/`PROFILE_SECTION_RENDERERS` (S1 stays `() => null` — renders via `ProfileHeader`, not a card).
- `services/frontend/src/locales/en/translation.json` — add `employeeProfile.sections.documents.*`, `employeeProfile.header.photo*` keys.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — `photoStorageKey`, `Document`/`DocumentType` — foundation
- [x] `storage/file-storage.service.ts` + `storage.module.ts` — local-disk save/delete/stream, validation — shared foundation
- [x] `access/identity.controller.ts` + provider/entity updates — S1 photo write/read path
- [x] `documents/*` — S5 provider, controller, service, module — write/read path
- [x] `app.module.ts`, `.env.example`, `.gitignore` — wiring
- [x] Backend `__tests__/*` + `employee-profile.e2e-spec.ts` extension — I/O matrix
- [x] Frontend types + `identity.service.ts`/`documents.service.ts` + hooks — typed client
- [x] `ProfileHeader.tsx` photo UI + `DocumentsSection.tsx` + `profile-sections.tsx` + `translation.json` — UI

**Acceptance Criteria:**
- Given I upload a new photo, when it saves, then it replaces my prior photo immediately, visible wherever S1 is rendered for any entitled audience
- Given I attempt to upload/edit a non-certificate S5 document type, or edit any S1 field other than photo, when I submit the request, then it is rejected — my write access is scoped to photo (S1) and certificate upload only (S5)

### Review Findings

- [x] [Review][Patch] Add missing I/O matrix e2e coverage [`test/employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Extract shared ProjectLine document allowlist [`documents.constants.ts`]
- [x] [Review][Patch] Set photo stream Content-Type and Multer oversize 400 handling [`identity.controller.ts`, `multer-exception.filter.ts`]
- [x] [Review][Patch] Roll back saved files when DB writes fail [`identity.service.ts`, `documents.service.ts`]
- [x] [Review][Patch] Harden upload validation (MIME normalization, buffer size, disposition escaping) [`file-storage.service.ts`]
- [x] [Review][Patch] Fix stale S1 provider comment and reuse `IdentityService.buildPhotoUrl` [`identity-section.provider.ts`]
- [x] [Review][Patch] Add `DocumentsSectionProvider` unit coverage [`documents-section.provider.spec.ts`]
- [x] [Review][Patch] Cache-bust avatar after upload [`ProfileHeader.tsx`]
- [x] [Review][Defer] Frontend Playwright upload UI coverage — deferred, out of spec verification commands
- [x] [Review][Defer] Swagger multipart docs for new routes — deferred, pre-existing pattern gap

## Design NotesLocal disk over object storage (see Intent > Approach for why): no S3/MinIO service and no cloud credentials exist in `.env.example` either. A `FileStorageService` abstraction keeps the on-disk choice swappable later without touching callers. Files are served through access-gated controller routes rather than Express static middleware specifically because S5's per-audience matrix (Colleague `none`, ProjectLine CV+certs-only) cannot be expressed as a static file permission — a public `/uploads/...` URL would leak certificates to anyone with the link — and the ProjectLine narrowing applies to the S5 list payload itself, not only the file bytes, since a static permission model can't express per-row type filtering either.

Concurrent same-employee photo uploads (e.g. a double-click) may orphan one of the two newly-written files — accepted for this story's scope, matching spec-2-2's last-write-wins precedent for concurrent S2/S3 writes; no upload lock or optimistic concurrency this story. Shared-link viewers are similarly out of scope for the new streaming endpoints — extending `SectionAccessGate` to recognize a shared-link token alongside organic relationships is left to a future story.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run db:migrate` — expect clean build/lint, migration applies
- `cd services/backend && npm test -- storage documents && npm run test:e2e -- employee-profile` — expect new + extended specs passing
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expect clean after new types/components

**Manual checks (if no CLI):**
- As a seeded employee, open your own profile, upload a photo and a PDF certificate; confirm the avatar updates immediately and the certificate appears in the Documents section; switch to a colleague's view and confirm the photo is visible but Documents is absent entirely

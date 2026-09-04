---
title: 'Record a Risk'
type: 'feature'
created: '2026-09-04'
status: 'done'
review_loop_iteration: 0
baseline_commit: '0e5597166c17496e98385e3e22238786ff1a05cf'
story_key: '5-1-record-a-risk'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-5-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-9-management-notes-with-visibility-flags.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C1 grants S6 to Manager/PP (`RW`) and Self (`none`), but no `RiskRecord` model, `risks` module, or S6 provider exists — profile assembly returns `status: 'unavailable'`. Managers and PPs cannot record append-only risk history; employees have no leak path today only because data does not exist.

**Approach:** Add `RiskRecord` persistence, a `risks` Nest module with `@RegisterProvider('section', 'S6')`, parallel `GET/POST` routes gated by `SectionAccessGate`, and an S6 profile card mirroring management-notes — backend + e2e first, then frontend S6 UI.

## Boundaries & Constraints

**Always:**
- **Model** (`RiskRecord`): `id`, `subjectEmployeeId`, `authorEmployeeId`, `level` enum (`low`|`need_attention`|`medium`|`high`|`leaver`), `description` (trimmed, 1–500), `details` (trimmed, 1–5000), `recordedAt` (`@db.Date`), `createdAt`. FKs → `Employee` `onDelete: Restrict`. Index `subjectEmployeeId`. **Append-only** — no update/delete endpoints.
- **Create payload:** `{ level, description, details, recordedAt }` — ISO calendar date `YYYY-MM-DD`; allow today and past dates.
- **Create auth:** `requireSection(viewer, subjectId, 'S6', 'RW')` **or** (`hasPermission(viewer, CREATE_EDIT_RISKS)` **and** `audience.sections.S6 !== 'none'`). Colleague-only → **403** even with functional permission.
- **S6 provider:** return all records for subject sorted `recordedAt` desc, `createdAt` desc; `currentLevel` = most recent record's level (omit when no records). Self: section omitted by assembler — provider never called.
- **S6 wire DTO:**
  ```typescript
  type RiskRecordReadDto = {
    id: string;
    level: 'low' | 'need_attention' | 'medium' | 'high' | 'leaver';
    description: string;
    details: string;
    recordedAt: string; // ISO date
    author: { id: string; displayName: string };
    createdAt: string;
  };
  type RisksSectionDto = {
    records: RiskRecordReadDto[];
    currentLevel?: RiskRecordReadDto['level'];
  };
  ```
- **Routes** (mirror `management-notes.controller.ts`; param `:employeeId`):
  - `GET /employees/:employeeId/risks` — `requireSection(S6)`; delegate to provider.
  - `POST /employees/:employeeId/risks` — create auth above; **201** with `RiskRecordReadDto`.
  - Viewer without linked `Employee` → **403**; unknown subject → **404** before gate; malformed UUID → **400**.
  - Provider/DB failure on parallel GET → **503**; profile maps to `status: 'unavailable'`.
- **Frontend:** wire `PROFILE_SECTION_RENDERERS.S6` + title key `employeeProfile.sections.risks`. `accessLevel === 'RW'`: append form (level select, description, details, date). `R`: read-only history. Empty state: "No risk history recorded." — no add CTA for read-only viewers. Mutations invalidate `employeeProfileQueryKey`. Level labels via i18n; severity color tokens optional in 5.1 (text label required per a11y).
- **E2e:** `risks.e2e-spec.ts` — UM append for direct report; PP create; colleague 403; Self 403 on direct POST and no S6 in own profile; validation 400s; profile S6 `available` with data after create.

**Ask First:**
- If `EmployeeProfilePage` S6 wiring conflicts with an in-flight profile refactor, ship backend + e2e first.

**Never:**
- Trend arrow or `trend` field (Story 5.2); `DashboardSummaryProvider` or Risk Dashboard page (5.3); All Employees risk column; update/delete risk records; client-side-only access checks; conflate `leaver` level with `dismissed` employment status.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| First record | No prior history POST valid payload | 201; `currentLevel` equals new level | N/A |
| Self direct API | Employee E POST `/employees/E/risks` | — | 403 |
| Self profile | E GET own profile | No `S6` key in `sections` | N/A |
| Functional perm, no C1 | Colleague + `create_edit_risks` POST | — | 403 |
| Whitespace description | POST `{ description: "   ", ... }` | — | 400 |
| Future recordedAt | POST date after today | — | 400 |

</frozen-after-approval>

## Code Map

**New / modified (backend):**
- `services/backend/prisma/schema.prisma` — `RiskLevel` enum + `RiskRecord` model; migration.
- `services/backend/src/modules/risks/` (new) — module, service, controller, DTOs, swagger, `risks-section.provider.ts` (`@RegisterProvider('section', 'S6')`).
- `services/backend/src/app.module.ts` — import `RisksModule`.
- `services/backend/test/risks.e2e-spec.ts` (new) — epic AC + S6 matrix rows.
- `services/backend/test/support/access-matrix.ts` — S6 assertion rows if missing.

**New / modified (frontend):**
- `services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/` — card, form, history list (mirror `ManagementNotesSection/`).
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — `S6` renderer + title key.
- `services/frontend/src/types/employee-profile.ts` — `RiskLevel`, `RisksSection`, payload types.
- `services/frontend/src/api/services/risk.service.ts` + `api/hooks/useRiskMutations.ts` + `hooks/data/useRisksData.ts`.
- `services/frontend/src/locales/en/translation.json` — `employeeProfile.s6.*`, level labels.

**Existing infrastructure (reference only):**
- `services/backend/src/modules/access/access-resolver.service.ts` — S6 grants L56/L75/L109.
- `services/backend/src/modules/management-notes/` — parallel-route + section-provider template.
- `services/backend/src/modules/action-items/action-items.controller.ts` — `assertCanCreate` permission fallback pattern L123–148.
- `services/backend/src/modules/contracts/permission-keys.ts` — `CREATE_EDIT_RISKS` L10/L46–48.
- `services/backend/src/modules/access/profile-assembler.service.ts` — omits `none` sections L105–109; missing provider → `unavailable` L159–162.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` — `RiskRecord` + migration — persistence substrate
- [x] `services/backend/src/modules/risks/risks.service.ts` — `buildSection`, `createRecord` — core logic
- [x] `services/backend/src/modules/risks/risks-section.provider.ts` — `@RegisterProvider('section', 'S6')` — profile assembly
- [x] `services/backend/src/modules/risks/risks.controller.ts` — GET/POST parallel routes — API surface
- [x] `services/backend/src/modules/risks/risks.module.ts` + `app.module.ts` — wire module — bootstrap
- [x] `services/backend/src/modules/risks/__tests__/risks.service.spec.ts` — unit tests for auth + validation
- [x] `services/backend/test/risks.e2e-spec.ts` — UM/PP create, Self deny, colleague deny, profile S6 — epic AC
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/` — S6 card + append form — UJ-1 UX
- [x] `services/frontend/src/api/services/risk.service.ts` + mutation hooks — write path + cache invalidation
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — wire S6 renderer — profile integration

**Acceptance Criteria:**
- Given I hold Manager access over employee E, when I POST level `high` with description, details, and a date to `/employees/E/risks`, then the record is appended, becomes E's `currentLevel`, and appears in E's S6 via profile and `GET /employees/E/risks`
- Given I am employee E, when I GET my own profile or POST `/employees/E/risks`, then S6 is absent from the profile response and the POST returns 403
- Given E has no prior risk records, when a Manager POSTs the first record, then `currentLevel` matches the new level and history contains one entry
- Given I am a colleague with `create_edit_risks` but no Manager/PP relationship to E, when I POST for E, then the server returns 403

## Spec Change Log

## Design Notes

Append-only history: each POST creates a new row; `currentLevel` is always derived from the latest `recordedAt` (tiebreak `createdAt`). Do not add PATCH — risk levels only move forward/back via new records per FR-24.

## Verification

**Commands:**
- `cd services/backend && npm run db:migrate && npm test && npm run test:e2e -- risks` — expected: all pass
- `cd services/frontend && npm run lint && npm test` — expected: pass
- `cd services/frontend && npm run test:e2e -- risks-section` — expected: S6 record flow passes (when e2e added)

**Manual checks:**
- Manager opens employee profile → S6 shows history + append form; Self profile has no Risks section.

## Suggested Review Order

**Data model & persistence**

- Append-only risk history with five-level enum and employee FKs
  [`schema.prisma:396`](../../services/backend/prisma/schema.prisma#L396)

**Backend core**

- Section assembly: sorted history plus derived currentLevel
  [`risks.service.ts:29`](../../services/backend/src/modules/risks/risks.service.ts#L29)

- S6 provider registration for profile assembler
  [`risks-section.provider.ts:11`](../../services/backend/src/modules/risks/risks-section.provider.ts#L11)

- Parallel routes with RW gate and CREATE_EDIT_RISKS fallback
  [`risks.controller.ts:39`](../../services/backend/src/modules/risks/risks.controller.ts#L39)

**Frontend S6 card**

- Profile renderer wiring for Manager/PP viewers
  [`profile-sections.tsx:67`](../../services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx#L67)

- History list, current level, and RW append form
  [`RisksSection.tsx:28`](../../services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/RisksSection.tsx#L28)

**Tests & access harness**

- Epic AC e2e: create, Self deny, validation, provider failure
  [`risks.e2e-spec.ts:96`](../../services/backend/test/risks.e2e-spec.ts#L96)

- S6 parallel route in leak harness inventory
  [`matrix-leak-assertions.ts:9`](../../services/backend/test/support/matrix-leak-assertions.ts#L9)

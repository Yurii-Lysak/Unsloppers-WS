---
title: 'Manually Create an Action Item'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 0
baseline_commit: '34ba5ce39dc25c6c0c20fcf8cac2662aac5e8259'
story_key: '4-1-manually-create-an-action-item'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-4-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-9-management-notes-with-visibility-flags.md'
  - '{project-root}/docs/project-requirements.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C6 `ActionItemCreation` is a non-persisting stub, there is no `ActionItem` model, no `action-items` module, and S14 returns `unavailable` in profile assembly. Managers and PPs cannot assign tasks with due dates; Epic 10 cannot call C6 for campaign activation.

**Approach:** Add `ActionItem` persistence, a real C6 implementation, S14 section provider, and parallel REST routes — mirroring the management-notes module pattern. Ship backend + e2e first; wire frontend S14 inline-create when `EmployeeProfilePage` exists.

## Boundaries & Constraints

**Always:**
- **Model** (`ActionItem`): `id`, `assigneeId`, `authorId`, `title` (trimmed, 1–200 chars; reject empty/whitespace-only after trim), `description?` (trimmed, max 2000), `dueDate` (`@db.Date`), `link?` (valid URL, max 2048; treat `''` as omitted), `status` enum default `open`, `source` enum (`manual`|`campaign`), `campaignId?` (nullable UUID, no FK in 4.1 — Story 4.4 adds `Campaign` FK), `completedAt?`, `cancelledAt?`, `cancelledReason?`, `createdAt`, `updatedAt`. FKs `assigneeId` and `authorId` → `Employee` with `onDelete: Restrict`. Index `assigneeId`, `authorId`.
- **Create payload:** `{ title, description?, dueDate, link? }` — assignee from `:employeeId` route param; `authorId` from viewer; `source: 'manual'`; `status: 'open'`.
- **Validation:** `dueDate` required as ISO calendar date (`YYYY-MM-DD`); allow today and past dates (overdue is derived in 4.3); reject datetime-with-offset strings. `title` trimmed then non-empty. `link` validated only when present.
- **Assignee:** must be an existing active `Employee` row; unknown UUID → **404** before gate.
- **Create auth:** `SectionAccessGate.requireSection(viewer, employeeId, 'S14', 'RW')` **or** (`PermissionChecker.hasPermission(viewerUserId, 'create_action_items')` **and** resolved `audience.sections.S14 !== 'none'`). Colleague-only viewers → **403** even with the functional permission. PM-only ProjectLine `R` + `create_action_items` is the primary PM create path.
- **S14 provider:** Self `R` → items where `assigneeId === subjectId`; RW → all items for subject; **all statuses** in 4.1; sort `dueDate` asc, `createdAt` asc. Empty list when none.
- **S14 wire DTO** (profile + parallel routes):
  ```typescript
  type ActionItemReadDto = {
    id: string;
    title: string;
    description?: string;
    dueDate: string; // ISO date
    link?: string;
    status: 'open' | 'completed' | 'cancelled';
    source: 'manual' | 'campaign';
    author: { id: string; displayName: string };
    createdAt: string; // ISO
    updatedAt: string;
  };
  type AuthoredActionItemReadDto = ActionItemReadDto & {
    assignee: { id: string; displayName: string };
  };
  type ActionItemsSectionDto = { items: ActionItemReadDto[] };
  ```
- **Routes** (mirror `management-notes.controller.ts`; param is `:employeeId` throughout):
  - `GET /employees/:employeeId/action-items` — `requireSection(viewer, employeeId, 'S14')` (minimum `R`, including Self); delegate to provider.
  - `POST /employees/:employeeId/action-items` — create auth above; **201** with `ActionItemReadDto`.
  - `GET /me/authored-action-items` — authenticated viewer's employee; returns **open** items where `authorId === viewer` and live C1 still grants non-`none` S14 on the assignee (ReportingLine, PP, or ProjectLine — per-row `resolveAudience`); sorted `dueDate` asc; each row is `AuthoredActionItemReadDto` (assignee summary for Epic 12 dashboard widget).
  - Resolve viewer via `CurrentUserProvider`; no linked `Employee` → **403**. Malformed UUID → **400** (`ParseUUIDPipe`).
  - Provider/DB failure on parallel `GET` → **503**; profile assembler maps the same failure to `status: 'unavailable'`.
- **C6:** `ActionItemsService.createActionItem` persists and returns DTO including `id`, `status`, `createdAt`. Rebind `ContractsModule` from stub to real service.
- **Profile assembler:** S14 returns `{ accessLevel, data: ActionItemsSectionDto }` instead of `unavailable`.
- **E2e:** `action-items.e2e-spec.ts` — UM create for direct report; PM ProjectLine `R` + `create_action_items` create; PM out of scope deny; PP create; colleague 403; functional permission with S14 `none` deny; validation 400s (empty title, title >200, invalid URL, bad date format); authored-list AC with assignee fields; S14 profile + parallel GET.

**Ask First:**
- If `EmployeeProfilePage` is still absent in `services/frontend`, stop frontend work and ship backend + e2e first, then wire S14 UI in the same branch once profile infrastructure merges.

**Never:**
- Complete, cancel, or overdue UI/logic (Stories 4.2–4.3); campaign-sourced creation (4.4); departure-hook auto-cancel (AD-18, defer to 4.2 or employment); dashboard page/widget components (Epic 12); notifications; client-side-only access checks.

## I/O & Edge-Case Matrix

Delta scenarios not fully spelled out in Boundaries above:

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PM ProjectLine create | PM with S14 `R` + `create_action_items` POST for project talent | 201 | 403 if S14 `none` |
| Self-assignment | Viewer with `create_action_items` + Self S14 `R` POST for self | 201 | 403 if S14 `none` |
| Authored list scope drift | Author created item, then lost live S14 on assignee | Item omitted from `/me/authored-action-items` | N/A |
| Whitespace title | POST `{ title: "   ", dueDate }` | — | 400 |
| Past due date | POST with `dueDate` before today | 201; overdue derived in 4.3 | 400 only on bad format |
| Title at boundary | 201 chars | — | 400 |
| Empty link | POST `{ link: "" }` | Stored as no link | N/A |
| Provider failure | DB error on parallel GET | — | 503; profile `unavailable` |

</frozen-after-approval>

## Code Map

**New / modified:**
- `services/backend/prisma/schema.prisma` — add `ActionItem` model + `Employee` relations; new migration.
- `services/backend/src/modules/action-items/` (new) — module, service (C6 impl), controller, DTOs, swagger, section provider.
- `services/backend/src/modules/contracts/action-item-creation.contract.ts` — extend `ActionItemDto` with `status`, `completedAt?` (read shape for consumers).
- `services/backend/src/modules/contracts/contracts.module.ts` — rebind `ActionItemCreation` to real service (remove stub).
- `services/backend/src/app.module.ts` — import `ActionItemsModule`.
- `services/backend/test/support/access-matrix.ts` — S14 matrix rows for e2e assertions.
- `services/backend/test/action-items.e2e-spec.ts` (new) — epic AC + matrix rows.
- `services/frontend/src/pages/EmployeeProfilePage/components/ActionItemsSection.tsx` — inline create (only when profile page exists; mirror spec-1-9 section wiring).

**Existing infrastructure (reference only — no changes expected):**
- `services/backend/src/modules/access/access-resolver.service.ts` — S14 grants at L64/L83/L117.
- `services/backend/src/modules/access/permission-checker.service.ts` — `CREATE_ACTION_ITEMS` check.
- `services/backend/src/modules/management-notes/management-notes.controller.ts` — parallel-route template.
- `services/backend/src/modules/management-notes/management-notes-section.provider.ts` — `@RegisterProvider('section', …)` pattern.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` — `ActionItem` model + migration — persistence substrate
- [x] `services/backend/src/modules/action-items/action-items.service.ts` — C6 impl + create/list queries — core logic
- [x] `services/backend/src/modules/action-items/action-items-section.provider.ts` — `@RegisterProvider('section', 'S14')` — profile assembly
- [x] `services/backend/src/modules/action-items/action-items.controller.ts` — POST/GET parallel routes + `GET /me/authored-action-items` — API surface
- [x] `services/backend/src/modules/action-items/action-items.module.ts` + `app.module.ts` — wire module; rebind C6 — bootstrap
- [x] `services/backend/src/modules/action-items/__tests__/action-items.service.spec.ts` — unit tests for all auth matrix rows
- [x] `services/backend/test/action-items.e2e-spec.ts` — full matrix: UM, PM in/out scope, PP, colleague, validation, authored list — epic AC
- [ ] `services/frontend/src/pages/EmployeeProfilePage/components/ActionItemsSection.tsx` — inline create (when profile page exists) — UJ-1 UX

**Acceptance Criteria:**
- Given I am UM for employee B, when I POST title "Submit Q3 self-review" with a due date and no link to `POST /employees/B/action-items`, then the item is created with `status: open`, `authorId` me, `assigneeId` B, `source: manual`, and appears in B's S14 via profile and `GET /employees/B/action-items`
- Given I authored an item for B, when I GET `/me/authored-action-items`, then the item is returned sorted by due date with `assignee: { id, displayName }` (dashboard widget data ready for Epic 12)
- Given I am a PM with ProjectLine S14 `R` and `create_action_items`, when I POST an action item for a project talent, then the server returns 201
- Given I am a PM with no ReportingLine, PP, or ProjectLine relationship to C, when I POST an action item for C, then the server returns 403 using live C1 resolution

### Review Findings

- [x] [Review][Patch] Provider failure 503 e2e missing [`test/action-items.e2e-spec.ts`] — spec requires parallel GET → 503 and profile S14 → `unavailable`; mirror `management-notes.e2e-spec.ts` providerOverrides block
- [x] [Review][Patch] Invalid calendar dates accepted [`create-action-item.dto.ts:35`, `action-items.service.ts:204`] — `2026-02-30` passes regex; `parseDueDate` has no round-trip validation → wrong dates or 500
- [x] [Review][Patch] C6 path bypasses REST validation [`action-items.service.ts:40`] — `createActionItem` does not enforce title/description/link/dueDate rules that DTO applies to POST (Epic 10 campaign path)
- [x] [Review][Patch] Assignee active check missing [`action-items.controller.ts:117`] — spec requires active `Employee`; `findUnique` by id only allows dismissed assignees
- [x] [Review][Patch] Provider omits audience threading [`action-items-section.provider.ts:37`, `action-items.service.ts:79`] — unlike S7, `buildSection` gets accessLevel only; R/RW branches are identical dead code
- [x] [Review][Patch] Duplicate create implementations [`action-items.service.ts:40`, `action-items.service.ts:58`] — `createManualItem` and `createActionItem` duplicate Prisma create with different normalization
- [x] [Review][Patch] Authored-list sort order untested [`action-items.e2e-spec.ts:159`] — AC requires due-date sort; no multi-item ordering assertion
- [x] [Review][Patch] `assignee.displayName` not asserted in e2e [`action-items.e2e-spec.ts:164`] — AC requires assignee summary for Epic 12 widget; only `assignee.id` checked
- [x] [Review][Patch] Self S14 profile read e2e missing [`test/action-items.e2e-spec.ts`] — no test that assignee sees own items via `GET …/profile` with `accessLevel: 'R'`
- [x] [Review][Patch] S14 denied-matrix provider unit tests missing [`action-items-section.provider.spec.ts`] — no `deniedMatrixCells().filter(S14)` / `recordDeniedCoverage` pattern from S7
- [x] [Review][Patch] PM create e2e minimal [`action-items.e2e-spec.ts:170`] — asserts 201 only; no response body, parallel GET, or profile S14 persistence check
- [x] [Review][Defer] Frontend `ActionItemsSection.tsx` not implemented — deferred per spec Ask First when `EmployeeProfilePage` absent

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm run lint` — expected: no errors
- `cd services/backend && npm test -- --testPathPatterns=action-items` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- action-items.e2e-spec` — expected: e2e pass (Postgres up)

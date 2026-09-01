---
title: 'Sortable, Filterable Employee List'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'd7f1a6890f2652cc816a1baec72798a6b56afe6d'
story_key: '3-1-sortable-filterable-employee-list'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/all-employees.html'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/services/frontend/.claude/rules/react-pages.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 3.2 delivered custom-field substrate, but there is no All Employees list — no REST endpoint, no FieldRegistry catalog for built-in/derived fields, and no Directory page. Managers and PPs cannot sort or filter employees by profile fields.

**Approach:** Extend C2 `FieldRegistry` with a field catalog and paginated list query over built-in history columns, one derived tenure field, and custom fields; expose `GET /api/v1/employees` with server-driven sort/filter/pagination and per-viewer field masking via C1; build the Directory SPA page with a shadcn data table and column filter popovers.

## Boundaries & Constraints

**Always:**
- Server-driven sort, filter, and pagination — no infinite scroll; display `{shown} of {total}` count (EXPERIENCE.md).
- FieldRegistry is the sole type-branching point — built-in, derived, and custom fields share one query API (AD-6).
- Per-field visibility enforced server-side on catalog and row payloads — never client-side CSS hiding (S16 + custom visibility from 3.2).
- Directory module depends only on `contracts` + Nest core (AD-1); consume C1/C8 via injection, do not import other feature modules.
- Wave-1 built-in columns: `name`, `grade`, `position`, `department`, `employment_type`, plus derived `years_with_company` (computed from earliest `effectiveFrom` across the four history tables).

**Ask First:**
- Whether dismissed employees are excluded from the default row set (C11/employment module not built — default to include all employees unless product says otherwise).
- Whether Colleague whitelist AC must pass end-to-end now (Story 1.8 not done — Colleague S16 is all `none` today).

**Never:** inline cell editing (Story 3.3); saved-view tabs or sharing (3.4); Export button behavior (3.5); Colleague card layout at `sm` (3.6); NFR-2 load testing (3.7); FieldProvider registrations from risks/CDS/employment modules; real `Department` entity / C12 filters; profile drill-through page (`/employees/:id`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Derived tenure filter | Manager filters `years_with_company > 3` | Only employees whose computed tenure exceeds 3 years returned; same filter shape as stored fields | 400 on unknown field or unsupported operator |
| Column sort | `sort=grade&order=asc` | Rows ordered by current grade value | 400 if field not sortable or not visible to viewer |
| Pagination | `page=2&pageSize=50` with 128 matches | Page 2 of rows + `total: 128` | 400 on invalid page/pageSize |
| Custom field filter | Select custom field + equals value | Rows filtered via FieldRegistry custom-value columns | Field absent from catalog if viewer lacks visibility |
| Management field hidden | Colleague viewer | Management/custom-management fields absent from `fields[]` and row cells | No filter side-channel via result counts |
| Empty filter | Valid filters matching zero rows | `{ rows: [], total: 0 }` + UI empty state copy | N/A |

</frozen-after-approval>

## Code Map

- [`services/backend/src/modules/contracts/field-registry.contract.ts`](../../services/backend/src/modules/contracts/field-registry.contract.ts) — extend C2 with `FieldSpec`, list-query DTOs, `listFields()`, `queryEmployees()` (keep existing `query()` for value fetch)
- [`services/backend/src/modules/directory/field-registry.service.ts`](../../services/backend/src/modules/directory/field-registry.service.ts) — built-in/derived field registration + SQL filter/sort engine (Prisma)
- [`services/backend/src/modules/directory/custom-field-visibility.service.ts`](../../services/backend/src/modules/directory/custom-field-visibility.service.ts) — reuse `canViewFieldDefinition` / `canViewFieldForSubject` for catalog + cell gating
- [`services/backend/src/modules/directory/employees.controller.ts`](../../services/backend/src/modules/directory/employees.controller.ts) — new `GET /employees` (mirror `custom-fields.controller.ts` auth/Swagger patterns)
- [`services/backend/src/modules/directory/employees.service.ts`](../../services/backend/src/modules/directory/employees.service.ts) — orchestrates C1 audience + visible field set + registry query
- [`services/backend/src/modules/directory/directory.module.ts`](../../services/backend/src/modules/directory/directory.module.ts) — register new controller/service
- [`services/backend/src/modules/access/access-resolver.service.ts`](../../services/backend/src/modules/access/access-resolver.service.ts) — real C1; per-row section grants for cell masking (`resolveAudience` ~L194)
- [`services/backend/prisma/schema.prisma`](../../services/backend/prisma/schema.prisma) — `Employee`, `User`, `*History` tables supply built-in column sources (no new models expected)
- [`services/backend/test/custom-fields.e2e-spec.ts`](../../services/backend/test/custom-fields.e2e-spec.ts) — e2e harness pattern (`createTestApp`, `loginAsOperator`)
- [`services/frontend/src/router/index.tsx`](../../services/frontend/src/router/index.tsx) — add `/directory` child route
- [`services/frontend/src/components/SideMenu/SideMenu.tsx`](../../services/frontend/src/components/SideMenu/SideMenu.tsx) — add Directory nav item (EXPERIENCE.md IA)
- [`services/frontend/src/pages/AdminRolesPage/AdminRolesPage.tsx`](../../services/frontend/src/pages/AdminRolesPage/AdminRolesPage.tsx) — page shell / loading-error pattern to mirror
- [`services/frontend/src/api/hooks/useFunctionalRoles.ts`](../../services/frontend/src/api/hooks/useFunctionalRoles.ts) — TanStack Query hook pattern
- [`_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/all-employees.html`](../../_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/all-employees.html) — visual reference (toolbar, compact rows, filter popovers)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/field-registry.contract.ts` — add field catalog + list query types — uniform C2 surface
- [x] `services/backend/src/modules/directory/field-registry.service.ts` — implement built-in/derived/custom list query — core filter engine
- [x] `services/backend/src/modules/directory/employees.service.ts` — viewer-scoped orchestration — C1 + visibility masking
- [x] `services/backend/src/modules/directory/employees.controller.ts` — REST + DTOs/Swagger — API surface
- [x] `services/backend/src/modules/directory/directory.module.ts` — wire controller/service — module registration
- [x] `services/backend/src/modules/directory/__tests__/employees.service.spec.ts` — unit tests for filter/sort/masking matrix rows
- [x] `services/backend/test/employees.e2e-spec.ts` — e2e for list endpoint auth + tenure filter
- [x] `services/frontend/src/api/employees.ts` + `src/api/hooks/useEmployeeList.ts` — typed client + query hook
- [x] `services/frontend/src/pages/AllEmployeesPage/` — table, column picker, filter popovers, pagination — Directory UI
- [x] `services/frontend/src/router/index.tsx` + `SideMenu.tsx` + `locales/en/translation.json` — route, nav, i18n keys

**Acceptance Criteria:**
- Given employees with only stored history join dates, when a manager filters by `years_with_company > 3`, then only employees whose computed tenure exceeds 3 years are returned
- Given a manager opens All Employees, when they sort by grade ascending, then rows reorder server-side and the UI reflects the active sort indicator
- Given a custom field with visibility `management`, when a Colleague-level viewer requests the field catalog, then that field is absent and cannot be used as a filter
- Given 128 matching employees with `pageSize=50`, when the viewer requests page 2, then 50 rows return with `total: 128` and the UI shows pagination controls (no infinite scroll)

## Design Notes

Tenure derivation: `years_with_company = floor((today − min(effectiveFrom)) / 365.25)` across `GradeHistory`, `PositionHistory`, `DepartmentHistory`, and `EmploymentTypeHistory` for the employee. Filter operators for Wave-1: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains` (text/select), `in` (multi_select). Frontend adds shadcn `table` via CLI; filter popovers per column follow EXPERIENCE.md Data Table pattern — all filter state lives in URL/searchParams and refetches via TanStack Query.

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm run lint` — expected: no errors
- `cd services/backend && npm test -- --testPathPatterns=directory` — expected: all directory unit tests pass
- `cd services/backend && npm run test:e2e -- employees.e2e-spec` — expected: list endpoint e2e pass (Postgres up)
- `cd services/frontend && npm run typecheck` — expected: no TS errors
- `cd services/frontend && npm run lint` — expected: no errors
- `cd services/frontend && npm run build` — expected: production build succeeds

**Manual checks:**
- Log in as a manager/PP seed user, open `/directory`, add a tenure filter and grade sort; confirm `{shown} of {total}` updates and management-only custom fields are absent for Colleague test user

### Review Findings

- [x] [Review][Decision] Management-tier custom fields absent from catalog for manager/PP viewers — **Resolved: option 2 (Self-scoped catalog).** Management fields stay hidden from catalog unless viewer has `manage_custom_fields`; subject-scoped visibility applies at row level via `canViewFieldForSubject` only.

- [x] [Review][Patch] Custom-field filter never matches rows [`services/backend/src/modules/directory/field-registry.service.ts:467-468`, `employee-query.helpers.ts:243-244`] — fixed: load custom values before filter/pagination.

- [x] [Review][Patch] Custom-field sort is a no-op [`services/backend/src/modules/directory/field-registry.service.ts:470-471`, `employee-query.helpers.ts:96-98`] — fixed: custom values passed into `sortSnapshots`.

- [x] [Review][Patch] Built-in history columns use unordered `history[0]` [`services/backend/src/modules/directory/field-registry.service.ts:546-564`] — fixed: `currentHistoryValue()` prefers `effectiveTo: null`.

- [x] [Review][Patch] Tenure derivation ignores older history rows [`services/backend/src/modules/directory/field-registry.service.ts:573-578`] — fixed: `min(effectiveFrom)` across all history rows.

- [x] [Review][Patch] `multi_select` `in` filter compares joined string, not array membership [`services/backend/src/modules/directory/employee-query.helpers.ts:193-199`] — fixed: array-aware `in` matching.

- [x] [Review][Patch] Frontend omits `in` operator for select/multi_select [`ColumnFilterPopover.tsx`] — fixed.

- [x] [Review][Patch] Select fields use free-text input, not option picker [`ColumnFilterPopover.tsx`] — fixed: option dropdown / checkbox list.

- [x] [Review][Patch] Filter popover shows stale operator/value after URL changes [`ColumnFilterPopover.tsx`] — fixed: sync on open.

- [x] [Review][Patch] No unit/e2e test for custom-field filter or sort [`field-registry.service.spec.ts`, `employees.e2e-spec.ts`] — fixed: unit tests added.

- [x] [Review][Patch] No e2e proving filter on hidden management field returns 400 [`employees.e2e-spec.ts`] — fixed.

- [x] [Review][Patch] No e2e proving management field values absent from row cells for colleague [`employees.e2e-spec.ts`] — fixed.

- [x] [Review][Patch] No Playwright coverage for `/directory` UI [`services/frontend/e2e/directory.spec.ts`] — fixed: spec added (requires `npx playwright install` locally).

- [x] [Review][Patch] Frontend `pageSize` not capped at backend max 100 [`useAllEmployeesPage.ts`] — fixed.

- [x] [Review][Patch] `formatCellValue` hardcodes display strings [`useAllEmployeesPage.ts`] — fixed: i18n keys.

- [x] [Review][Patch] `loadEmployeeSnapshots` assumes `employee.user` always present [`field-registry.service.ts:568`] — fixed: optional chaining.

- [x] [Review][Defer] In-memory full-table filter/sort vs SQL/Prisma engine [`field-registry.service.ts:467-483`] — deferred; NFR-2 performance validation scoped to Story 3.7.

- [x] [Review][Defer] C1 per-field masking for Wave-1 built-in columns [`employees.service.ts:61-63`] — deferred; built-ins always included in catalog; section-level S16 gating for built-ins belongs to Track A (Stories 1.6/1.8).

## Spec Change Log

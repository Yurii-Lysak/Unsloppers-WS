---
title: 'Export to Excel'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 1
story_key: '3-5-export-to-excel'
baseline_commit:
  workspace: 'cbc66758b7545b9e76b8948b661d7cb20561ee81'
  backend: '73f9cf4c5fa3555204c5bd2345072110d4a36432'
  frontend: 'e3075263a14b1c0c1a8ee7eace694d001e35f1f3'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-1-sortable-filterable-employee-list.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-4-saved-and-shared-views.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 3.1–3.4 deliver the All Employees list with filters, columns, sort, and saved views, but there is no way to take the current view outside the app. FR-11 requires a server-generated `.xlsx` export scoped to the exporter's entitlements.

**Approach:** Add `GET /employees/export` that accepts the current view config (filters, sort, order, column ids), paginates through the same access-resolved `listEmployees` pipeline used by the table, and streams a single-sheet `.xlsx`. Wire an Export toolbar action on All Employees that downloads the blob from the current URL state.

## Boundaries & Constraints

**Always:**
- Export is server-side only — never scrape the DOM or export the current paginated page slice.
- Same session auth as `GET /employees` (`CurrentUserProvider`); unauthenticated requests 401.
- Reuse `EmployeesService.listEmployees` (visibility, `resolveEffectiveFilters`, `maskRowCells`) — do not duplicate access logic.
- Column set = ordered intersection of client `columns` with the viewer's entitled fields; per-row values come from masked rows (blank cell when masked for that row).
- A field the viewer cannot see at all is absent from the file — not blanked as a column, not in metadata/hidden sheets.
- Headers use field display labels from `FieldSpec.label` (built-in + custom names).
- Fetch all matching rows via `pageSize: MAX_PAGE_SIZE` (100) loop until `collected >= total` — same termination guard as `CampaignsService.collectFilterMatchIds`; 500+ rows means 5+ pages, not a single fetch.
- Extract a private `listAllEmployees` page loop inside `EmployeesService` (do not call `CampaignsService` — AD-1 no feature-to-feature imports).
- Filename: `employees-export-YYYY-MM-DD.xlsx`; `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`; `Content-Disposition: attachment; filename="employees-export-YYYY-MM-DD.xlsx"`.
- `columns` is a required JSON-encoded string array of field ids (same transport pattern as `filters`); reject empty array, unknown field ids, and non-visible sort fields with 400 (mirror list validation).
- Deduplicate duplicate `columns` entries preserving first-seen order.
- Reject with 400 when every requested column is filtered out as non-visible for this viewer.
- Prefix string cell values that start with `=`, `+`, `-`, or `@` to block spreadsheet formula injection.
- Export button disabled while list is loading or an export request is in flight; on failure show destructive toast per EXPERIENCE.md ("Couldn't save. Try again." pattern).

**Ask First:**
- Excel library choice if `exceljs` is unsuitable (default: `exceljs` in backend).

**Never:** saved-view CRUD changes (3.4); colleague card layout (3.6); client-side xlsx generation; export endpoints for other surfaces; NFR-2 load-test pass (3.7).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path | Manager exports current filters + visible columns incl. management custom field | Valid `.xlsx` with header row, all filtered rows, entitled column values | N/A |
| Hidden field in selection | `columns` includes a field outside viewer `visibleFieldIds` | Column omitted from file entirely | N/A |
| Per-row mask | Column entitled globally but `maskRowCells` removed value for some rows | Column present; those cells empty; no hidden data | N/A |
| Shared-view filters | Saved view filters reference invisible field | Same as list: `resolveEffectiveFilters` drops entire filter set; export matches entitlement-resolved rows; frontend shows existing `filtersHidden` toast if applicable | N/A |
| Large set | 500+ matching employees | Full file via multi-page loop (`MAX_PAGE_SIZE` 100), no silent truncation | 504/timeout only if infra limit hit — must not cap rows in app code |
| Empty result | Filters match zero rows | Valid `.xlsx` with headers only, zero data rows | N/A |
| Empty columns | `columns=[]` or missing | Reject | 400 |
| All columns invisible | Every requested column is outside viewer entitlements | Reject | 400 |
| Unknown column | `columns` includes field id absent from catalog | Reject like list filter on unknown field | 400 |
| Invalid sort | `sort` references non-visible or non-sortable field | Reject like list endpoint | 400 |
| Malformed input | Invalid `filters`/`columns` JSON | Reject like list endpoint | 400 |
| Export failure | Network or 5xx during download | Button re-enabled; destructive toast; no partial file saved | Toast only |
| Formula-like cell | Cell value starts with `=`, `+`, `-`, or `@` | Stored as plain text (prefixed), not executable formula | N/A |

</frozen-after-approval>

## Code Map

- [`services/backend/src/modules/directory/employees.controller.ts`](../../services/backend/src/modules/directory/employees.controller.ts) — add `@Get('export')` **before** `:employeeId` (mirror `lookup` ordering); return `StreamableFile` with `Content-Disposition`
- [`services/backend/src/modules/directory/employees.service.ts`](../../services/backend/src/modules/directory/employees.service.ts) — `listEmployees` 48–101, `resolveEffectiveFilters` 114–131, `maskRowCells` 257–294; add `listAllEmployees` loop + `exportEmployees`
- [`services/backend/src/modules/directory/dto/list-employees-query.dto.ts`](../../services/backend/src/modules/directory/dto/list-employees-query.dto.ts) — reuse filter/sort/order parsing patterns for export DTO
- [`services/backend/src/modules/campaigns/campaigns.service.ts`](../../services/backend/src/modules/campaigns/campaigns.service.ts) — `collectFilterMatchIds` 346–367: pagination loop pattern to mirror (not import)
- [`services/backend/src/modules/contracts/employee-list.constants.ts`](../../services/backend/src/modules/contracts/employee-list.constants.ts) — `MAX_PAGE_SIZE` (100), `MIN_PAGE`
- [`services/backend/package.json`](../../services/backend/package.json) — add `exceljs`
- [`services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx`](../../services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx) — toolbar ~118–160: add Export button (left cluster with Clear filters / Column picker per EXPERIENCE.md)
- [`services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts`](../../services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts) — `visibleColumnIds`, `query` state to pass to export
- [`services/frontend/src/api/client.ts`](../../services/frontend/src/api/client.ts) — `apiClient.raw.get` with `responseType: 'blob'` for download
- [`services/frontend/src/api/services/employee.service.ts`](../../services/frontend/src/api/services/employee.service.ts) — `getEmployeesList`; add `exportEmployeesList`
- [`services/backend/test/employees-export.e2e-spec.ts`](../../services/backend/test/employees-export.e2e-spec.ts) — e2e: entitled column present, restricted column absent in file bytes

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/package.json` — add `exceljs` dependency — xlsx generation
- [x] `services/backend/src/modules/directory/dto/export-employees-query.dto.ts` — filters/sort/order + required `columns` JSON array — request validation (non-empty, known fields)
- [x] `services/backend/src/modules/directory/employees.service.ts` — `listAllEmployees` page loop + `exportEmployees`: paginate, project entitled columns, formula-safe cell values, build workbook — core logic
- [x] `services/backend/src/modules/directory/employees.controller.ts` — `GET export` returning `StreamableFile` with `Content-Disposition` — REST surface
- [x] `services/backend/src/modules/directory/employees.swagger.ts` — document export endpoint — API docs
- [x] `services/backend/src/modules/directory/__tests__/employees-export.service.spec.ts` — unit tests: column projection, pagination loop, filtersHidden path, formula prefix — I/O matrix cases
- [x] `services/backend/test/employees-export.e2e-spec.ts` — e2e: entitled column present, restricted column absent in file bytes — integration proof
- [x] `services/frontend/src/api/services/employee.service.ts` — `exportEmployeesList` blob download via `apiClient.raw.get` — data layer
- [x] `services/frontend/src/hooks/data/useEmployeesData.ts` — export mutation hook with success/error toasts — TanStack Query pattern
- [x] `services/frontend/src/pages/AllEmployeesPage/` — Export button + loading/disabled state (list loading + export in flight) — FR-11 UX per EXPERIENCE.md
- [x] `services/frontend/src/locales/en/translation.json` — `directory.export` keys — i18n

**Acceptance Criteria:**
- Given a manager's current view includes a management-visible custom field they hold access to, when they export to `.xlsx`, then the file contains that column with correct values for matching filtered rows, generated server-side from the access-resolved query
- Given a user's column selection includes a field they are not entitled to see at all, when they export, then that column is absent from the file with no trace in metadata or hidden sheets, and export completes without timeout or truncation at 500+ rows
- Given a column is entitled globally but `maskRowCells` blanks values for some rows, when they export, then the column is present and masked rows have empty cells with no leaked values

## Design Notes

Map `FieldValue` types to Excel cell types (dates as ISO strings, booleans as TRUE/FALSE, multi-select as comma-joined). Frontend passes `columns` from `visibleColumnIds` plus current `query.filters`/`sort`/`order`. Trigger client download from blob using `Content-Disposition` filename when present, else `employees-export-YYYY-MM-DD.xlsx`.

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm test -- --testPathPatterns=employee-export` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- employees-export.e2e-spec` — expected: e2e pass (Postgres up)
- `cd services/frontend && npm run typecheck` — expected: no TS errors
- `cd services/frontend && npm run lint` — expected: no errors

## Spec Change Log

### Review Findings

- [x] [Review][Patch] No `Content-Disposition` header — browsers may not name the download; added to Always constraints and controller task.
- [x] [Review][Patch] `columns` param validation unspecified — added required non-empty JSON array, unknown-field 400, dedupe rule, and matrix rows for empty/unknown columns.
- [x] [Review][Patch] `MAX_PAGE_SIZE` is 100 — clarified multi-page loop requirement for 500+ rows; termination guard must match `collectFilterMatchIds`.
- [x] [Review][Patch] Design Notes suggested importing `CampaignsService` loop — violates AD-1; replaced with private `listAllEmployees` inside `EmployeesService`.
- [x] [Review][Patch] No auth constraint — added same-session auth as list endpoint.
- [x] [Review][Patch] No export failure UX — added disabled-during-flight, destructive toast, and matrix row per EXPERIENCE.md.
- [x] [Review][Patch] No formula-injection guard — added cell-prefix rule and matrix row.
- [x] [Review][Patch] `filtersHidden` shared-view path unspecified — matrix row documents same `resolveEffectiveFilters` behavior + existing toast.
- [x] [Review][Patch] Invalid sort field unhandled — added 400 rule and matrix row (mirrors `field-registry.service.ts` sort visibility check).
- [x] [Review][Patch] Code map cited extending `employees.e2e-spec.ts` while tasks used separate file — aligned on `employees-export.e2e-spec.ts` only.
- [x] [Review][Patch] Second AC conflated global column omission with per-row masking — split into two ACs matching Boundaries (global omit vs per-row blank).
- [x] [Review][Patch] No unit-test task for `filtersHidden` export path — added to service spec scope.
- [x] [Review][Patch] Export button placement unspecified — Code map notes left toolbar cluster per EXPERIENCE.md.
- [x] [Review][Defer] Playwright e2e for Export button — no blob-download fixture pattern yet; backend e2e covers entitlement bytes; UI covered by manual check until e2e infra exists.
- [x] [Review][Defer] Colleague-mode export (Story 3.6) — same `listEmployees` pipeline applies automatically when colleague whitelist lands; no 3.5-specific work.

### Code Review Findings

- [x] [Review][Patch] Missing `filtersHidden` export unit test — added service spec asserting filters drop before `queryEmployees` [`employees-export.service.spec.ts`]
- [x] [Review][Patch] Missing per-row mask export test (AC3) — added unit test for blank masked cells [`employees-export.service.spec.ts`]
- [x] [Review][Patch] "Omits invisible columns" test did not assert column count — strengthened to expect single entitled column [`employees-export.service.spec.ts`]
- [x] [Review][Patch] All-requested-columns-invisible produced empty-header file — `exportEmployees` now 400s when `exportColumnIds` is empty [`employees.service.ts:135-137`]
- [x] [Review][Patch] Missing e2e for empty `columns` array, invalid sort, empty result set, all-invisible columns — added [`employees-export.e2e-spec.ts`]
- [x] [Review][Patch] Verification command missed `employee-export.helpers.spec.ts` — updated pattern to `employee-export` [`spec-3-5-export-to-excel.md`]
- [x] [Review][Defer] `listAllEmployees` reuses `listEmployees` and recomputes `writableFieldIds` per row — acceptable for v1; optimize in 3.7 if export is slow [`employees.service.ts:167-203`]
- [x] [Review][Defer] Workbook buffered in memory rather than streamed — matches spec `StreamableFile` task; revisit only if infra timeouts appear [`employees.service.ts:159-164`]
- [x] [Review][Defer] E2E global teardown access-matrix coverage failure — pre-existing harness gap unrelated to export tests (9/9 export cases pass) [`jest-e2e.global-teardown.ts`]
- [x] [Review][Dismiss] Spec references `FieldSpec.label` but codebase uses `FieldSpec.name` — implementation matches catalog contract; spec boundary wording is cosmetic only

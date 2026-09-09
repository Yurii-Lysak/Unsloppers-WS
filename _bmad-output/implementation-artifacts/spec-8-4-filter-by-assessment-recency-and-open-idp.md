---
title: 'Filter by Assessment Recency and Open IDP'
type: 'feature'
created: '2026-09-10'
status: 'done'
review_loop_iteration: 0
story_key: '8-4-filter-by-assessment-recency-and-open-idp'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-8-context.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The All Employees directory has no way to surface who hasn't been assessed recently or who has an open IDP — Epic 8's FR is undelivered, and CDS still isn't registered as a `FieldProvider`, so the Directory module has no registry-sanctioned way to read CDS-derived data at all.

**Approach:** Register CDS as a `FieldProvider` for two new derived fields (`last_assessment_date`, `has_open_idp`), consumed by `FieldRegistryService` through the existing provider registry (never a direct Prisma read from `directory`); extend the filter engine with date-range (`between`) and "no value" (`is_empty`) operators; and exclude, before pagination, any subject the viewer lacks S12 access to from CDS-filtered results.

## Boundaries & Constraints

**Always:**
- CDS registers one `FieldProvider` per derived field via `@RegisterProvider('field', <fieldId>)`; `FieldRegistryService` looks it up through `ProviderRegistryService.get('field', fieldId)` — never a direct CDS import from `directory` (AD-1).
- `last_assessment_date` = most recent `CDSAssessment.date` per employee, `null` if none. `has_open_idp` = `true` iff the employee has ≥1 `IDPRecord` with `completedAt: null`.
- Both `FieldSpec`s: `sectionId: 'S12'`, `source: 'derived'`, `filterable: true`, `sortable: true`, `editable: false` — reuses the existing `maskRowCells` S12 gate for cell display, no new masking code.
- `FilterOperator` gains `'between'` (value `[from, to]` ISO dates, inclusive) and `'is_empty'` (cellValue is `null`), both scoped to `type: 'date'` fields only — no other field type's operator list changes.
- When a request's filters reference `last_assessment_date` or `has_open_idp`, resolve the viewer's per-employee S12 access first and scope the query to only S12-visible employees *before* filtering/pagination, so both `total` and page contents exclude denied subjects — not just their cell values.
- Frontend: one filter entry per field stays the invariant — "between" is a single filter with a tuple value, never two simultaneous entries on the same `fieldId`.
- A `{status:'unavailable'}` result from `ProviderRegistryService.get('field', fieldId)` for either new field (a registration gap, AD-3) is never treated as `null`/`false`. `EmployeeListQueryResultDto`/`EmployeeDirectoryListResultDto` gain `fieldsUnavailable?: string[]` (mirrors `filtersHidden`) naming any requested field currently unavailable. A request that *filters* on an unavailable field is rejected outright (503) rather than silently matched over a placeholder value; a request that only displays or sorts the field still succeeds, with its cells rendered as an explicit "temporarily unavailable" state distinct from both a masked cell and a genuine `null`.
- A `between` value that isn't exactly two well-formed ISO dates with `from <= to`, or an `is_empty` filter carrying a non-null `value`, is a 400 Bad Request from `FieldRegistryService.validateFilters` — never a silent no-match.

**Resolved (2026-09-10):** `last_assessment_date` and `has_open_idp` are included in Story 3.5 export when the viewer's catalog includes S12 — same filters, masking, and `fieldsUnavailable` semantics as the All Employees list. Campaign-audience filters remain out of scope for this story.

**Never:** an "assessment count" or "overdue" derived field (unscoped); migrating `mentor_status`/`years_with_company` onto the new `FieldProvider` pattern (existing inline-computed fields, out of scope); any `CDSAssessment`/`IDPRecord` schema change; a maintenance UI for either new field (read-only derived, like `years_with_company`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| NEVER_ASSESSED_FILTER | Manager filters `last_assessment_date` `is_empty` | Only S12-visible employees with zero assessments | — |
| DATE_RANGE_FILTER | Manager filters `last_assessment_date` `between` [2026-01-01, 2026-06-30] | Only S12-visible employees whose most recent assessment date falls in range (inclusive) | — |
| OPEN_IDP_FILTER | Manager filters `has_open_idp` `eq` true | Only S12-visible employees with ≥1 open IDP | — |
| NO_S12_ACCESS_EXCLUDED | Unrelated viewer (S12 `none` for employee X) filters `has_open_idp` true; X has an open IDP | X absent from both rows and `total` | — |
| COLLEAGUE_VIEWER | Colleague-mode viewer applies either CDS filter | Zero rows match — S12 is `none` for every subject | — |
| UNFILTERED_SORT_STILL_MASKS | Manager sorts by `last_assessment_date`, no CDS filter active | S12-denied rows still appear (from other visible fields) with the CDS cell masked, per existing `maskRowCells` — no row exclusion without an active CDS filter | — |
| COMBINED_CDS_AND_NON_CDS_FILTER | Manager filters `has_open_idp` `eq` true AND `department` `eq` "Sales" | Only S12-visible Sales employees with ≥1 open IDP | — |
| BOTH_CDS_FILTERS_TOGETHER | Manager filters `last_assessment_date` `between` [...] AND `has_open_idp` `eq` true | Only S12-visible employees satisfying both conditions; one S12-narrowing pass serves both filters | — |
| MALFORMED_BETWEEN | `between` value is not `[from, to]` (wrong length, missing bound) or `from > to` | Request rejected | 400 Bad Request |
| IS_EMPTY_WITH_STRAY_VALUE | `is_empty` filter sent with a non-null `value` (direct API call) | Request rejected | 400 Bad Request |
| PROVIDER_UNAVAILABLE_FILTER | Manager filters on `last_assessment_date`/`has_open_idp` while that field's `FieldProvider` is unregistered/unavailable | Request rejected — never silently matched over `null` | 503, names the unavailable field |
| PROVIDER_UNAVAILABLE_DISPLAY | `last_assessment_date`/`has_open_idp` requested as a visible or sort column (no filter) while unavailable | Request succeeds; affected cells render "temporarily unavailable" (not masked, not empty); `fieldsUnavailable` names the field | — |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/contracts/field-provider.contract.ts` -- **new**; `abstract class FieldProvider { abstract queryValues(employeeIds: string[]): Promise<FieldQueryResultDto[]> }`, mirrors `section-provider.contract.ts:11-17`
- `services/backend/src/modules/contracts/field-registry.contract.ts` -- add `'between'`/`'is_empty'` to `FilterOperator` (:21-22); add `last_assessment_date`/`has_open_idp` to `BUILTIN_FIELD_IDS` (:87-97); add optional `employeeIds?: string[]` to `EmployeeListQueryOptions` (:66-73), mirroring `FieldQueryOptions.employeeIds` (:50); **also add `PROVIDER_BACKED_FIELD_IDS: ReadonlySet<string>` (mirrors `INTEGRATED_LIST_FIELD_IDS`, :100-103) naming these two ids — required because both are also going into `BUILTIN_FIELD_IDS`, and `isBuiltinFieldId()` gates the value-lookup paths touched below; without a second set to check, provider-sourced values can never reach `getCellValue` (review finding: BUILTIN_FIELD_IDS breaks provider read path)**; add `fieldsUnavailable?: string[]` to `EmployeeListQueryResultDto` (:80-85), mirroring the existing `filtersHidden` shape on `EmployeeDirectoryListResultDto`
- `services/backend/src/modules/cds/cds.service.ts` -- add `getLastAssessmentDates(employeeIds)` (Prisma `groupBy(['employeeId'], { _max: { date: true } })`, formatted as plain `YYYY-MM-DD` via the existing `formatCdsCalendarDate` pattern — this format is the contract `matchesDateFilter` below relies on) and `getOpenIdpEmployeeIds(employeeIds)` (`findMany` `distinct: ['employeeId']` where `completedAt: null`); **the field provider wrapping the latter must emit an explicit `value: false` for every requested employeeId not in that result, not just `true` for the ones that are — an omitted id would otherwise default to `null` downstream and break `eq true`/`eq false` matching (review finding: has_open_idp provider must emit false)**
- `services/backend/src/modules/cds/last-assessment-date-field.provider.ts`, `open-idp-field.provider.ts` -- **new**, `@RegisterProvider('field', 'last_assessment_date' | 'has_open_idp')`, each calling its `CdsService` method and mapping to `FieldQueryResultDto[]`
- `services/backend/src/modules/cds/cds.module.ts` -- add the two new field providers to `providers`
- `services/backend/src/modules/directory/field-catalog.ts` -- add `last_assessment_date` (`type: 'date'`) and `has_open_idp` (`type: 'boolean'`) to `BUILTIN_FIELD_SPECS`, mirroring `mentor_status` (:66-75)
- `services/backend/src/modules/directory/employee-query.helpers.ts` -- new `matchesDateFilter` (handles `between`/`is_empty` plus `eq/neq/gt/gte/lt/lte` via `YYYY-MM-DD` string comparison — see the format contract on `cds.service.ts` above), replacing `date`'s current fall-through to `matchesTextFilter` (`matchesFilter`, :255-256); add `DATE_FILTER_OPERATORS` and route `type: 'date'` to it in `allowedOperatorsForField` (:322-323) — **note this also changes the operator set for any pre-existing custom field of type `date`: `contains` is dropped and gt/gte/lt/lte/between/is_empty are added; confirm this cross-cutting change is intended, and add a regression test asserting existing custom date-field filters still behave as expected (review finding: date operator split affects existing fields)**; **`getCellValue` (:49-80): change the `customValueMap` gate at :55 from `if (customValueMap && !isBuiltinFieldId(fieldId))` to a map-membership check (e.g. `if (customValueMap?.has(`${snapshot.employeeId}:${fieldId}`))`) so the two new builtin-but-provider-backed ids aren't shadowed by the switch's `default: return null` (review finding: BUILTIN_FIELD_IDS breaks provider read path)**
- `services/backend/src/modules/directory/field-registry.service.ts` -- `collectCustomFieldIds` (:567-587) also gates on `!isBuiltinFieldId(fieldId)`, so it must additionally collect ids in `PROVIDER_BACKED_FIELD_IDS` from `visibleFieldIds`, `filters`, **and `sortFieldId`** — the last one matters because a sort-only request (no active CDS filter) still needs provider values loaded to sort correctly, per the UNFILTERED_SORT_STILL_MASKS row (review finding: sort-only path for provider values); `queryEmployees` (:468-565): source `last_assessment_date`/`has_open_idp` cell values via `ProviderRegistryService.get('field', fieldId)`, merging results into the same value map `getCellValue`'s fixed gate above reads from; **when the registry reports `{status:'unavailable'}` for a requested field id (visible, filtered, or sorted), add it to a `fieldsUnavailable` set on the result; if that field is also in `filters`, throw `ServiceUnavailableException` before querying rather than proceeding — filtering is meaningless without real values (Boundaries, PROVIDER_UNAVAILABLE_FILTER/_DISPLAY)**; `validateFilters` (:608-635) gains checks for `between` (exactly two well-formed ISO dates, `from <= to`) and `is_empty` (no non-null `value` present), throwing `BadRequestException` alongside the existing operator-allowlist check (Boundaries, MALFORMED_BETWEEN/IS_EMPTY_WITH_STRAY_VALUE); honor `options.employeeIds` by narrowing `loadEmployeeSnapshots()`'s result before `applyFilters`
- `services/backend/src/modules/directory/employees.service.ts` -- `listEmployees` (:64-126): when `query.filters` includes either new field, resolve `accessResolver.resolveAudience(viewerEmployeeId, employeeId).sections.S12 !== 'none'` per employee over the full roster and pass the resulting id set as `employeeIds` into `queryEmployees`; **reuse that same roster-wide audience-resolution result inside `maskRowCells`'s per-row `resolveAudience` call for this request instead of resolving it a second time — as drafted, the pre-filter step and the masking step each call `AccessResolver.resolveAudience` once per employee, doubling the cost Design Notes claims this mirrors (review finding: double resolveAudience calls per request)**; propagate `result.fieldsUnavailable` onto `EmployeeDirectoryListResultDto.fieldsUnavailable` (:20-28), mirroring how `filtersHidden` is already threaded through
- `services/backend/src/modules/directory/__tests__/*`, `services/backend/src/modules/cds/__tests__/*`, `test/employee-profile.e2e-spec.ts` (or a directory-list e2e spec) -- cover I/O matrix rows, **including the combined-filter, malformed-input, and provider-unavailable rows added above**
- `services/frontend/src/types/employees.ts` -- add `'between' | 'is_empty'` to `FilterOperator`; add `fieldsUnavailable?: string[]` to `EmployeeListResponse` (:55-63), mirroring `filtersHidden`
- `services/frontend/src/components/AudienceBuilder/ColumnFilterPopover/ColumnFilterPopover.tsx` -- `operatorsForField` (:24-41): `date` gains `'between'`, `'is_empty'`; render two `Input type="date"` for `between`, no value input for `is_empty` (mirrors the no-extra-input shape of the `boolean` branch at :199-206); **`apply()` (:134-164) needs two new branches: `operator === 'is_empty'` must submit without requiring `value` (currently falls into the trailing `else if (!value) return` at :154-156 and can never submit), and `operator === 'between'` needs its own two-date local state feeding a `[from, to]` array into `onApply` — the existing single `value: string` state (used by `valueToString`/`valueToSelectedOptions`) cannot hold a tuple (review finding: frontend apply() can't express new operators)**
- `services/frontend/src/components/AudienceBuilder/filter-utils.ts` -- `formatCellValue` (:29-47): today `null` and `undefined` both render as `t('directory.cellEmpty')`, which would conflate a genuine never-assessed/no-open-idp result with an unavailable provider; callers (`EmployeeTable.tsx`, `EmployeeCardList.tsx`) must check `fieldsUnavailable.includes(field.id)` first and render a distinct `t('directory.cellUnavailable')` instead of calling `formatCellValue` for that cell
- `services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx` -- mirror the existing `filtersHidden` toast effect (:97-106) for `fieldsUnavailable`: a non-empty list surfaces a distinct notice naming the unavailable field(s)
- `services/frontend/src/locales/en/translation.json` -- add `directory.operators.between`, `directory.operators.is_empty`, `directory.fields.last_assessment_date`, `directory.fields.has_open_idp`, `directory.cellUnavailable`, `directory.fieldsUnavailableNotice`

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/field-provider.contract.ts` -- new `FieldProvider` abstract -- registry contract for derived fields
- [x] `services/backend/src/modules/contracts/field-registry.contract.ts` -- add operators, field ids, `employeeIds` option -- shared contract surface
- [x] `services/backend/src/modules/cds/cds.service.ts` + 2 new field-provider files + `cds.module.ts` -- register CDS as a `FieldProvider` -- delivers the epic's named cross-module dependency
- [x] `services/backend/src/modules/directory/field-catalog.ts` -- register the two `FieldSpec`s -- makes the fields listable/filterable
- [x] `services/backend/src/modules/directory/employee-query.helpers.ts` -- date-aware matcher + operator list -- delivers `between`/`is_empty`
- [x] `services/backend/src/modules/directory/field-registry.service.ts` -- provider-sourced values + `employeeIds` scoping -- wires filters through the registry, keeps pagination/total correct
- [x] `services/backend/src/modules/directory/employees.service.ts` -- pre-resolve S12-visible ids for CDS filters -- closes the access leak
- [x] Backend unit tests + story-scoped e2e (`employees-cds-filters`, `employees-export`) -- I/O matrix rows covered; full-suite e2e skipped locally
- [x] `services/frontend/src/types/employees.ts` + `ColumnFilterPopover.tsx` + `filter-utils.ts` + `AllEmployeesPage.tsx` + `translation.json` -- frontend filter UI and `fieldsUnavailable` display

**Acceptance Criteria:**
- Given the All Employees list, when a viewer with S12 access opens the last-assessment-date filter, then they can choose "never assessed" as a distinct option or a before/after/between date range, and results update accordingly
- Given the same list, when a viewer applies has-open-IDP = yes, then only employees with at least one incomplete IDP appear, scoped to employees the viewer has Manager/PP access to
- Given a Colleague-mode viewer, when either CDS filter is applied, then the list returns zero matching rows and no CDS data is exposed via the filter

### Review Findings

Applied in code review (2026-09-10):

- [x] [Review][Patch] Add `between`/`is_empty` to `EmployeeFieldFilterDto` FILTER_OPERATORS [services/backend/src/modules/directory/dto/list-employees-query.dto.ts]
- [x] [Review][Patch] Add `between`/`is_empty` to frontend URL filter parsing [services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts]
- [x] [Review][Patch] Add `fieldsUnavailable` to `EmployeeListEntity` Swagger contract [services/backend/src/modules/directory/entities/employee-list.entity.ts]
- [x] [Review][Patch] Return zero rows when CDS filters are stripped for viewers without S12 field access [services/backend/src/modules/directory/employees.service.ts]
- [x] [Review][Patch] Suppress provider-backed sort values for S12-denied employees (sort-only, no CDS filter) [services/backend/src/modules/directory/employees.service.ts, field-registry.service.ts]
- [x] [Review][Patch] Wrap provider `queryValues` in try/catch → controlled unavailable state [services/backend/src/modules/directory/field-registry.service.ts]
- [x] [Review][Patch] Label last-assessment-date `is_empty` as "Never assessed" in filter UI [ColumnFilterPopover.tsx, translation.json]
- [x] [Review][Patch] Show translated field names in `fieldsUnavailable` toast [AllEmployeesPage.tsx]
- [x] [Review][Patch] Block `between` apply when from > to in ColumnFilterPopover [ColumnFilterPopover.tsx]
- [x] [Review][Patch] Do not retry list requests on HTTP 503 [useEmployeeList.ts]
- [x] [Review][Patch] Add unit tests for employeeIds narrowing, has_open_idp filter, suppress values, colleague zero-result, sort-only S12 suppress [field-registry.service.spec.ts, employees.service.spec.ts]

- [x] [Review][Patch] HTTP e2e coverage for CDS directory filters [services/backend/test/employees-cds-filters.e2e-spec.ts]
- [x] [Review][Patch] Frontend unit tests for filter apply, URL parsing, and cell display [vitest + column-filter-apply.test.ts, directory-query-parsing.test.ts, filter-utils.test.ts]
- [x] [Review][Patch] Export CDS columns with same filter/masking/`fieldsUnavailable` semantics as list [employees.service.ts, employee-export.helpers.ts, employees-cds-filters.e2e-spec.ts]
- [x] [Review][Defer] Campaign-audience CDS filters — out of story scope
- [x] [Review][Defer] Provider-unavailable HTTP e2e — deferred (no provider-unregister harness; unit tests cover fail-closed path)

## Design Notes

`employeeIds` lands on the shared `EmployeeListQueryOptions`/`FieldRegistry.queryEmployees` contract rather than a CDS-only side-channel because `queryEmployees` is a pure, viewer-agnostic engine today (no viewer identity flows through it); the new field is an additive, optional extension shaped exactly like the existing `FieldQueryOptions.employeeIds`, so campaigns/saved-views/export — none of which reference these two fields — see no behavior change today. That holds only until the "Ask First" question above is resolved in favor of exposing these fields there; this note is not itself an answer to that question.

Per-employee `resolveAudience` (no batch API exists on `AccessResolver`) is run over the full roster instead of one page, and only when a request actually filters on `last_assessment_date`/`has_open_idp` — not on every list request. As drafted in the Code Map, this is an *additional* pass on top of `maskRowCells`'s existing per-page `resolveAudience` calls, not a reuse of them — the two should be consolidated (see the `employees.service.ts` Code Map bullet) or this doubles resolution cost on every CDS-filtered request rather than matching `maskRowCells`'s cost profile. If Ask First later extends these fields to Export, note that `exportEmployees` already paginates internally (`listAllEmployees`'s loop) — a roster-wide resolution per page would multiply this cost further and needs its own caching plan before that extension ships.

`between` is one filter entry with a `[from, to]` tuple value, not two entries on the same `fieldId`: the frontend's `upsertFilter` already replaces any existing filter for a given field, so two simultaneous entries would silently collide.

This is CDS's first `FieldProvider` registration, so the registration-gap behavior required by epic AD-3 has no existing precedent to follow in this codebase; `fieldsUnavailable` (Boundaries, Code Map) is a new mechanism, not a reuse of one. Filtering on an unavailable field fails closed (503) rather than degrading to an empty or unfiltered result, because either of those would misrepresent the data as confidently as returning it would.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e` -- expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` -- expected: pass

**Manual checks (if no CLI):**
- As a seeded Unit Manager with reports missing assessments, open All Employees, filter last-assessment-date = "never assessed"; confirm only never-assessed reports appear
- Apply has-open-IDP = yes; confirm only reports with an open IDP appear. Then confirm an unrelated manager/PP without S12 access to a given report never sees that person surface under either filter
- Combine has-open-IDP = yes with a non-CDS filter (e.g. department); confirm the intersection is correct and S12-denied reports stay excluded
- Sort by last-assessment-date with no CDS filter active; confirm S12-denied reports still appear (masked cell) rather than disappearing or sorting as if never assessed
- Submit a between filter with the "to" date before the "from" date, and (via API client) an is_empty filter carrying a value; confirm both are rejected with a 400, not a silent empty result
- With one of the two CDS field providers temporarily unregistered (e.g. module not loaded), confirm: filtering on that field is rejected rather than matching everyone/no one, while displaying or sorting it still works and shows an explicit "temporarily unavailable" cell state

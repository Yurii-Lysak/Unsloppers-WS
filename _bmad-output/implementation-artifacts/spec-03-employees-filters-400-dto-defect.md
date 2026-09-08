---
title: 'Employee list filters query param 400 — DTO transform'
type: 'bugfix'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '3fd89342e3bc0ec69b96f2869823c821d4a40827'
context:
  - '{project-root}/services/backend/defects/bugs/03-employees-filters-400-dto-defect.md'
  - '{project-root}/_bmad-output/test-artifacts/known-red-diagnosis.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `GET /api/v1/employees?filters=[...]` returned `400 Unknown field "undefined"` because the custom `@Transform` on `ListEmployeesQueryDto.filters` JSON-parsed the query string but left array items as plain objects; the global `ValidationPipe({ whitelist: true })` then stripped `fieldId`, `operator`, and `value` before nested validation ran.

**Approach:** Ensure parsed filter array items are instantiated as `EmployeeFieldFilterDto` inside the `@Transform` (via `plainToInstance`), verify the two existing e2e cases pass, and close the defect record.

## Boundaries & Constraints

**Always:**
- Fix stays in `list-employees-query.dto.ts` only — no changes to `field-registry.service.ts` or controller logic.
- Existing e2e tests in `test/employees.e2e-spec.ts` are the acceptance gate; do not add redundant unit tests unless a gap is found.
- Saved-view DTOs (`create-saved-view.dto.ts`, `update-saved-view.dto.ts`) receive JSON bodies and use `@Type` without a competing `@Transform` — out of scope.

**Ask First:** none identified.

**Never:**
- Weaken global `whitelist: true` on `ValidationPipe`.
- Change filter validation semantics in `field-registry.service.ts`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Valid filter query | `filters=[{"fieldId":"years_with_company","operator":"gt","value":3}]` | `200` with filtered `rows` | N/A |
| No matching rows | filter value excludes all employees | `200` with `total: 0`, empty `rows` | N/A |
| Malformed JSON | `filters=not-json` | `400` with `Invalid filters JSON` | `BadRequestException` from Transform |
| Non-array JSON | `filters={"fieldId":"name"}` | `400` with `filters must be a JSON-encoded array` | `BadRequestException` from Transform |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/directory/dto/list-employees-query.dto.ts:97-121` — `@Transform` on `filters`; fix is `plainToInstance(EmployeeFieldFilterDto, parsed)` after `JSON.parse` (landed in `85ccfa9` / PR #47).
- `services/backend/src/bootstrap.ts:20-24` — global `ValidationPipe({ whitelist: true, transform: true })` — root cause context.
- `services/backend/src/modules/directory/field-registry.service.ts:~615` — `validateFilters` throws `Unknown field "undefined"` when `fieldId` is stripped (symptom, not fix site).
- `services/backend/test/employees.e2e-spec.ts:153-197` — `"filters employees by derived years_with_company > 3"`.
- `services/backend/test/employees.e2e-spec.ts` — `"returns an empty row set when filters match no employees"` (second failing case from diagnosis).
- `services/backend/defects/bugs/03-employees-filters-400-dto-defect.md` — defect tracker; update status to **Fixed** when verified.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/directory/dto/list-employees-query.dto.ts` — confirm `plainToInstance` in `@Transform` is present; add only if missing on current branch.
- [x] `services/backend/defects/bugs/03-employees-filters-400-dto-defect.md` — set **Status: Fixed**, note resolution commit/PR.
- [x] `services/backend/test/employees.e2e-spec.ts` — run the two filter e2e cases; no edits expected.

**Acceptance Criteria:**
- Given an authenticated employee viewer and `filters` query with a valid `fieldId`/`operator`/`value`, when `GET /api/v1/employees` is called, then response is `200` with correctly filtered rows (not `400 Unknown field "undefined"`).
- Given a filter that matches no employees, when `GET /api/v1/employees` is called, then response is `200` with `total: 0`.

## Verification

**Commands:**
- `cd services/backend && npm run test:e2e:serial -- --testPathPatterns=employees.e2e-spec --testNamePattern="filters employees|empty row set when filters"` — expected: both filter tests pass (global teardown matrix warning on filtered runs is acceptable for this scoped check).

## Suggested Review Order

- `plainToInstance` restores class metadata so whitelist keeps filter fields
  [`list-employees-query.dto.ts:113`](../../services/backend/src/modules/directory/dto/list-employees-query.dto.ts#L113)

- Defect closed with resolution commit and verification note
  [`03-employees-filters-400-dto-defect.md:57`](../../services/backend/defects/bugs/03-employees-filters-400-dto-defect.md#L57)

## Manual Test Paths

**Employee list filters** — filtering the All Employees directory by tenure and other fields.

The list endpoint accepts a JSON filter in the query string and returns only matching employees.

1. Sign in as a user who can view the employee directory.
2. Open the All Employees list (or call the employees API with a filter such as years with company greater than 3).
3. Confirm the list shows only employees matching the filter, not an error message.
4. Apply a filter that matches nobody and confirm you see an empty list, not an error.

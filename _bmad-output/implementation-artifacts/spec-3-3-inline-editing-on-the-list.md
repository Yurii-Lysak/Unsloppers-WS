---
title: 'Inline Editing on the List'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 0
story_key: '3-3-inline-editing-on-the-list'
baseline_commit: '34ba5ce39dc25c6c0c20fcf8cac2662aac5e8259' # services/backend HEAD
frontend_baseline_commit: 'f51d6781e396fe66bf6530feddb3595213d4a647' # services/frontend HEAD
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-1-sortable-filterable-employee-list.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 3-1 delivered a read-only All Employees directory. Managers and PPs must open a full profile to change grade, position, or custom fields — violating FR-9 inline-edit UX.

**Approach:** Extend the list API with per-row writability metadata and a unified field-update endpoint (built-ins via temporal history + S4 gate; custom fields via existing C2/S16 path). Build click-to-edit cells on the Directory table with keyboard support and server-side rejection for unauthorized writes.

## Boundaries & Constraints

**Always:**
- Built-in inline-editable fields: `grade`, `position`, `employment_type` only — writes append via the temporal-history Prisma extension (same persistence as profile S4).
- `department`, `name`, `years_with_company`, manager, and PP columns are never inline-editable (AD-14 / CAP-2).
- Custom field writes reuse S16 `RW` + visibility checks; unified `PATCH /employees/:employeeId/fields/:fieldId`.
- List response exposes `writableFieldIds` per row so the UI never guesses editability.
- Server rejects unauthorized writes with 403 regardless of UI state.
- Inline edit UX: click to edit, Enter/blur saves, Esc reverts; destructive toast on failure, cell stays editable.

**Ask First:**
- Whether Self can inline-edit S4 built-ins on own row (default: yes when S4 is `RW` for Self).

**Never:** saved views (3.4); export (3.5); colleague card layout (3.6); NFR-2 perf pass (3.7); department inline edit; client-only edit gating without server enforcement.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Manager edits report grade | ReportingLine viewer, S4 `RW`, PATCH `{ value: "Senior" }` on `grade` | New history row; list cell shows "Senior" on refetch | 403 if S4 not `RW` |
| Colleague blocked | Colleague viewer PATCH on unrelated employee | 403 | UI cell non-editable (no writableFieldIds entry) |
| Custom field write | Manager with S16 `RW` + visibility match | Value persisted via FieldRegistry.setValue | 403 when visibility/S16 deny |
| Invalid built-in field | PATCH on `department` | 400/403 — field not writable | N/A |
| Empty grade | PATCH `{ value: "" }` | 400 validation error | Toast + cell stays editable |

</frozen-after-approval>

## Code Map

- [`services/backend/src/modules/contracts/field-registry.contract.ts`](../../services/backend/src/modules/contracts/field-registry.contract.ts) — add `editable` on `FieldSpec`, `BUILTIN_EDITABLE_FIELD_IDS`
- [`services/backend/src/modules/directory/field-catalog.ts`](../../services/backend/src/modules/directory/field-catalog.ts) — set `editable` flags on built-ins
- [`services/backend/src/modules/directory/field-registry.service.ts`](../../services/backend/src/modules/directory/field-registry.service.ts) — `setBuiltinFieldValue()` via history `create`
- [`services/backend/src/modules/directory/employees.service.ts`](../../services/backend/src/modules/directory/employees.service.ts) — `writableFieldIds` per row + `updateEmployeeField()`
- [`services/backend/src/modules/directory/employees.controller.ts`](../../services/backend/src/modules/directory/employees.controller.ts) — `PATCH :employeeId/fields/:fieldId`
- [`services/backend/src/modules/directory/custom-fields.service.ts`](../../services/backend/src/modules/directory/custom-fields.service.ts) — reuse `setValue()` for custom fields
- [`services/frontend/src/pages/AllEmployeesPage/components/EditableCell/EditableCell.tsx`](../../services/frontend/src/pages/AllEmployeesPage/components/EditableCell/EditableCell.tsx) — click/keyboard inline editor
- [`services/frontend/src/pages/AllEmployeesPage/components/EmployeeTable/EmployeeTable.tsx`](../../services/frontend/src/pages/AllEmployeesPage/components/EmployeeTable/EmployeeTable.tsx) — wire editable cells
- [`services/frontend/src/api/hooks/useUpdateEmployeeField.ts`](../../services/frontend/src/api/hooks/useUpdateEmployeeField.ts) — mutation + list invalidation

## Tasks & Acceptance

**Execution:**
- [x] Backend contract + catalog + `setBuiltinFieldValue` — legal write path for S4 fields
- [x] Backend list `writableFieldIds` + PATCH endpoint — server-driven editability
- [x] Backend unit + e2e tests — manager write + colleague deny
- [x] Frontend EditableCell + mutation hook — FR-9 UX
- [x] Frontend wire EmployeeTable + i18n — Directory integration

**Acceptance Criteria:**
- Given a Unit Manager with write access to grade for a direct report, when they edit the grade cell inline and save, then the change persists and appears on list refetch
- Given a Colleague views a column configured as inline-editable, when they attempt to edit a cell for someone they hold no Manager/PP relationship with, then the cell renders non-editable and a direct API write is rejected server-side

## Design Notes

Built-in writes use `clock.now()` as `effectiveFrom` on a new history row — the temporal-history extension closes the prior open row and records the timeline event. Custom fields route through the existing `CustomFieldsService.setValue` after the controller applies S16 `RW`.

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm test -- --testPathPatterns=directory` — expected: directory unit tests pass
- `cd services/frontend && npm run typecheck` — expected: no TS errors
- `cd services/frontend && npm run lint` — expected: no errors

## Spec Change Log

## Suggested Review Order

**Server-driven editability**

- Unified PATCH endpoint delegates built-in vs custom field writes
  [`employees.controller.ts:55`](../../services/backend/src/modules/directory/employees.controller.ts#L55)

- Per-row `writableFieldIds` computed from S4/S16 access resolution
  [`employees.service.ts:72`](../../services/backend/src/modules/directory/employees.service.ts#L72)

- Built-in writes append temporal history rows via `setBuiltinFieldValue`
  [`field-registry.service.ts:144`](../../services/backend/src/modules/directory/field-registry.service.ts#L144)

**Frontend inline edit UX**

- Click-to-edit cell with Enter/blur save and Esc revert
  [`EditableCell.tsx:27`](../../services/frontend/src/pages/AllEmployeesPage/components/EditableCell/EditableCell.tsx#L27)

- Mutation hook invalidates list query and shows destructive toast on failure
  [`useUpdateEmployeeField.ts:8`](../../services/frontend/src/api/hooks/useUpdateEmployeeField.ts#L8)

- Table wires writable cells from server `writableFieldIds`
  [`EmployeeTable.tsx:62`](../../services/frontend/src/pages/AllEmployeesPage/components/EmployeeTable/EmployeeTable.tsx#L62)

**Tests**

- Unit coverage for write paths, access denial, and validation
  [`employees.service.spec.ts:306`](../../services/backend/src/modules/directory/__tests__/employees.service.spec.ts#L306)

- E2e PATCH scenarios for manager write and colleague deny
  [`employees.e2e-spec.ts`](../../services/backend/test/employees.e2e-spec.ts)

### Review Findings

- [x] [Review][Patch] `manage_custom_fields` bypass marks custom fields writable without S16 RW — `resolveWritableFieldIds` grants writability via `canManage` but `updateEmployeeField` also requires `sectionGate` S16 `RW`, so holders of the functional permission can see editable cells that PATCH rejects with 403 [`employees.service.ts:252`](../../services/backend/src/modules/directory/employees.service.ts#L252)
- [x] [Review][Patch] Esc revert missing for boolean/select/multi_select editors — spec requires Esc reverts; only text/number/date inputs wire `handleKeyDown` for Escape [`EditableCell.tsx:144`](../../services/frontend/src/pages/AllEmployeesPage/components/EditableCell/EditableCell.tsx#L144)
- [x] [Review][Patch] Blur saves unchanged values — `commitEdit` fires PATCH on blur even when draft equals the current cell value, causing needless API traffic [`EditableCell.tsx:216`](../../services/frontend/src/pages/AllEmployeesPage/components/EditableCell/EditableCell.tsx#L216)
- [x] [Review][Defer] Per-row `writableFieldIds` resolves access per row × field (N+1) — acceptable for Wave 1; Story 3.7 owns list perf at scale [`employees.service.ts:69`](../../services/backend/src/modules/directory/employees.service.ts#L69) — deferred, Story 3.7
- [x] [Review][Defer] Self-edit S4 built-ins not covered by e2e — spec default is yes when S4 is RW; no regression signal yet — deferred, low risk given access-resolver unit coverage elsewhere
- [x] [Review][Defer] Frontend e2e mocks API for inline edit — custom-field and non-text types not exercised in Playwright; backend e2e covers custom text PATCH — deferred, acceptable split for v1

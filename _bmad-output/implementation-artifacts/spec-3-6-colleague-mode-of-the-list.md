---
title: 'Colleague Mode of the List'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 1
story_key: '3-6-colleague-mode-of-the-list'
baseline_commit:
  workspace: '4bbf11819c239f7aaa6e9b0a32c25f351bb7bb0e'
  backend: 'f51ea1674568cad7188549b73095a4bd6a4bbea7'
  frontend: '187ca6c348ca071b67fded693e598c9db93a6855'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-8-enforce-the-colleague-whitelist-everywhere.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-5-export-to-excel.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 3.1–3.5 deliver the All Employees list, but C1 whitelist enforcement stops at custom fields (S16). Built-in columns (`grade`, `employment_type`, `years_with_company`, etc.) are returned to every authenticated viewer, and `maskRowCells` does not vary cells per subject audience. Colleague-level users therefore see management data in API payloads — violating FR-15 and `access-model.md` Rule 4.

**Approach:** Wire C1 section grants into the existing `listEmployees` pipeline: map each list field to a profile section, filter the catalog to sections the viewer can ever see, mask each row's cells per `resolveAudience(viewer, subject)`, and add S10/S11 derived list columns. Frontend renders only server-returned fields and adds the stacked-card layout below the `md` breakpoint (768px) per EXPERIENCE.md.

## Boundaries & Constraints

**Always:**
- Whitelist per `access-model.md` Rule 4 / `COLLEAGUE_SECTION_GRANTS`: **S1** identity (excluding the mentor field per Rule 8 / D5), **S10 dates only** (no leave type), **S11 project name only**, plus S16 custom fields whose visibility is `colleague` (Story 1.10). Override stale `epics.md` wording ("S10 leave type").
- Server-side enforcement only — restricted fields must be **absent** from `fields[]` and row `cells`, never omitted by frontend CSS/JS (`buildDirectoryDisplayData` is display projection only).
- Reuse `AccessResolver.resolveAudience` + `SectionAccessGate.listGrantedSections` — do not duplicate C1 logic or import feature modules (AD-1). Inject `ProjectAssignment` (C3) for S11 cell values.
- Add `sectionId: SectionId` to `FieldSpec` and map built-ins: `name`/`position`/`department` → S1; `grade`/`employment_type` → S4; `years_with_company` → **exclude** (not whitelisted); new `current_leave_dates` → S10; new `project_names` → S11. Custom fields → S16 via `CustomFieldVisibilityService` (C1 Colleague S16 is `none`; colleague-tier custom visibility is separate from `listGrantedSections`).
- **Catalog (`fields[]`)**: include a field when the viewer has non-`none` grant for its section toward **at least one** active employee. Union across all relationship classes the viewer holds (Self, Reporting line, Project line, PP, Full access). Pure colleagues get the colleague whitelist plus **Self-only S4** columns (`grade`, `employment_type`) when Self grants S4 — `years_with_company` stays excluded; those Self-only S4 columns are **display-only** (`filterable: false`, `sortable: false`) so peers cannot be inferred via filter side-channels.
- **Per-row cells**: for each row, `resolveAudience(viewer, subject)` with multi-audience union per `access-model.md` Rule 10; delete the cell when the effective section grant is `none`; format S10 for Colleague as date ranges only; S11 as comma-separated project names (no PM/DM/period).
- **S10/S11 v1 defaults** (resolves Ask First unless product reopens): `sortable: false`, `filterable: false`, display-only; leaves enrichment via a narrow `contracts` read surface (preferred over inline `TimetrackerClient` in directory per AD-1).
- Filter/sort validation already uses `visibleFieldIds` — non-catalog fields return 400 (existing `field-registry.service.ts` behavior).
- Saved views (3.4): intersect stored `columnIds` with the viewer's entitled catalog on apply — omit non-visible columns silently (same entitlement rules as export); do not 400 on stale shared-view columns.
- Inline edit (3.3): list cells are read-only when the viewer lacks `RW` on that section for that subject — Colleague toward others never receives edit affordances on whitelist fields.
- Export (`GET /employees/export`) inherits the same pipeline automatically — no separate export logic.
- Timetracker/integration unavailable: S10/S11 cells use the existing fail-soft display pattern (empty or "Temporarily unavailable" label per EXPERIENCE.md State Patterns) — access decisions do not soften.
- Row name link to `/employees/:id` already exists; profile assembly whitelist is Story 1.6/1.8 — do not rebuild profile UI here.
- Supersedes the interim owner decision in `colleague-whitelist.e2e-spec.ts:240–245` — paginated list **is** whitelist-scoped as of this story.

**Ask First:**
- Reopen S10/S11 filter/sort support only if product explicitly requests it after v1 ships display-only.
- Reopen leaves list enrichment shape only if the contracts abstraction proves insufficient during implementation.

**Never:** row-set filtering (all employees remain visible); client-only column hiding as security; widening Colleague S10 to include leave type; NFR-2 load test (3.7); saved-view CRUD changes (3.4); new profile sections; campaign-sender S14 exception widening on the list (Rule 7 applies only to campaign surfaces).

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Pure colleague | Viewer with no Manager/PP/ProjectLine/FullAccess toward anyone | `fields[]` = S1 (no mentor) + S10 + S11 + colleague-tier custom + Self-only S4 display columns (`employment_type`, `grade`; not `years_with_company`); peer row cells whitelist-only; own row includes Self S4 values | Filter/sort on Self-only S4 columns → 400 (`not filterable`) |
| Self in list | Colleague toward peers; Self toward own row | Own row includes Self-granted sections (e.g. `employment_type`); peer rows remain whitelist-only | N/A |
| Mixed audience | Manager of X, colleague of Y in same response | Catalog includes S4 fields; X's row has `grade`; Y's row omits `grade` (key absent) | N/A |
| Project line | PM/DM with project-line (not reporting-line) access to Z | Catalog includes project-line sections per Rule 2; Z's row respects project-line narrowing (no S2/S3 fields in list) | N/A |
| Full access | Holder of full profile access | Full entitled catalog; no per-row builtin masking beyond normal C1 | N/A |
| API tamper | Colleague sends `sort=grade` or filter on `employment_type` | 400 — field not in catalog | 400 |
| Saved view columns | Colleague applies saved view whose `columnIds` includes `grade` | `grade` omitted from applied columns; list loads with entitled columns only | N/A |
| Export inherit | Colleague exports with `columns` including `grade` | Column omitted from `.xlsx` (same as 3.5 entitlement rules) | 400 if all columns invisible |
| S10 narrowing | Colleague views peer with active leave | `current_leave_dates` shows date ranges (e.g. `"2026-09-01 – 2026-09-05"`; multiple ranges semicolon-separated); no leave type in cell value | Empty string when no leaves |
| S11 narrowing | Colleague views peer on projects | `project_names` = names only, comma-separated | Empty when unassigned |
| Integration down | Timetracker unavailable for S10/S11 | Fail-soft cell per EXPERIENCE.md; no type/PM/DM leak | N/A |
| Custom S16 | Colleague + `management` custom field | Absent from catalog and cells (existing S16 rules) | Filter on mgmt field → 400 |
| Responsive UI | Viewport `< md` (768px) | Stacked employee cards with whitelist columns only; card tap navigates to profile | N/A |

</frozen-after-approval>

## Code Map

- [`services/backend/src/modules/contracts/field-registry.contract.ts`](../../services/backend/src/modules/contracts/field-registry.contract.ts) — add `sectionId` to `FieldSpec`; extend `BUILTIN_FIELD_IDS` with `current_leave_dates`, `project_names`
- [`services/backend/src/modules/directory/field-catalog.ts`](../../services/backend/src/modules/directory/field-catalog.ts) — section mappings; add S10/S11 derived specs (display-only in v1)
- [`services/backend/src/modules/directory/employees.service.ts`](../../services/backend/src/modules/directory/employees.service.ts) — `filterVisibleFields` 328–358 (built-ins always included today); `maskRowCells` 361–398 (custom-only today); extend both + add S10/S11 enrichment post-query
- [`services/backend/src/modules/directory/field-registry.service.ts`](../../services/backend/src/modules/directory/field-registry.service.ts) — `queryEmployees`; may need hook for derived S10/S11 cell population
- [`services/backend/src/modules/contracts/access-resolver.contract.ts`](../../services/backend/src/modules/contracts/access-resolver.contract.ts) — `COLLEAGUE_SECTION_GRANTS` 67–84; `ResolvedAudience`
- [`services/backend/src/modules/access/section-access-gate.service.ts`](../../services/backend/src/modules/access/section-access-gate.service.ts) — `listGrantedSections` 70–72
- [`services/backend/src/modules/access/access-resolver.service.ts`](../../services/backend/src/modules/access/access-resolver.service.ts) — `resolveAudience` per viewer×subject
- [`services/backend/src/modules/integrations/leaves-section.provider.ts`](../../services/backend/src/modules/integrations/leaves-section.provider.ts) — S10 Colleague dates-only narrowing pattern 38–57
- [`services/backend/src/modules/access/projects-section.provider.ts`](../../services/backend/src/modules/access/projects-section.provider.ts) — S11 name-only pattern
- [`services/backend/src/modules/contracts/project-assignment.contract.ts`](../../services/backend/src/modules/contracts/project-assignment.contract.ts) — C3 for S11 list values
- [`services/backend/src/modules/directory/custom-field-visibility.service.ts`](../../services/backend/src/modules/directory/custom-field-visibility.service.ts) — S16 per-field gating (keep; integrate with section map)
- [`services/backend/test/colleague-whitelist.e2e-spec.ts`](../../services/backend/test/colleague-whitelist.e2e-spec.ts) — update 240–245 comment; add list whitelist cases
- [`services/backend/test/employees.e2e-spec.ts`](../../services/backend/test/employees.e2e-spec.ts) — colleague custom-field tests 351–463; extend for builtin whitelist + mixed audience + self row
- [`services/backend/test/employees-export.e2e-spec.ts`](../../services/backend/test/employees-export.e2e-spec.ts) — colleague export 348–398; add builtin column omission
- [`services/backend/src/modules/directory/__tests__/employees.service.spec.ts`](../../services/backend/src/modules/directory/__tests__/employees.service.spec.ts) — unit tests for catalog union + per-row builtin mask
- [`services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts`](../../services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts) — `buildDirectoryDisplayData` 85–102 (display-only; not security); saved-view column intersection with server `fields`
- [`services/frontend/src/pages/AllEmployeesPage/components/EmployeeTable/EmployeeTable.tsx`](../../services/frontend/src/pages/AllEmployeesPage/components/EmployeeTable/EmployeeTable.tsx) — table at `≥ md`; name `Link` to profile 115–121
- [`services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx`](../../services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx) — add responsive card list branch below `md` breakpoint
- [`_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md`](../../_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md) — line 23 colleague auto-apply; line 120 stacked cards below `md` (768px)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/field-registry.contract.ts` — add `sectionId` to `FieldSpec`, new builtin ids — uniform section mapping
- [x] `services/backend/src/modules/directory/field-catalog.ts` — map sections; add S10/S11 derived fields — whitelist completeness
- [x] `services/backend/src/modules/directory/employees.service.ts` — C1 catalog union + per-row builtin/S10/S11 mask + enrichment — core enforcement
- [x] `services/backend/src/modules/directory/field-registry.service.ts` — support S10/S11 derived values in query pipeline if enrichment lives here — keep FieldRegistry as type branch point
- [x] `services/backend/src/modules/directory/__tests__/employees.service.spec.ts` — unit: pure colleague catalog, mixed-audience per-row mask, self row, S10/S11 narrowing — I/O matrix
- [x] `services/backend/test/employees.e2e-spec.ts` — e2e: colleague builtin whitelist absent from payload; manager mixed rows; self row — integration proof
- [x] `services/backend/test/employees-export.e2e-spec.ts` — e2e: colleague cannot export `grade` — export inherit
- [x] `services/backend/test/colleague-whitelist.e2e-spec.ts` — replace interim list comment; add list whitelist cases — align with Story 3.6
- [x] `services/frontend/src/pages/AllEmployeesPage/` — `EmployeeCardList` below `md`, server `fields` authoritative, saved-view column sanitization, card tap → profile — EXPERIENCE.md responsive rule
- [x] `services/frontend/src/locales/en/translation.json` — card layout i18n keys — i18n

**Acceptance Criteria:**
- Given a user holds no Manager, PP, Project line, or Full-access relationship with respect to anyone in the list, when they open All Employees, then S1 identity fields (excluding mentor), S10 dates, S11 project names, colleague-tier custom fields, and Self-only S4 display columns (`employment_type`, `grade` — not `years_with_company`) appear in `fields[]`; peer row `cells` contain only the colleague whitelist; the viewer's own row includes Self-granted S4 values; Self-only S4 columns reject filter/sort via API
- Given a user is the manager of Employee X but a plain colleague to Employee Y, when both appear in the same list response, then X's row includes manager-line fields (e.g. `grade`) while Y's row omits them, within one render
- Given the same user appears in their own list row, when they view All Employees, then their own row reflects Self-granted sections while other rows remain colleague-whitelist scoped
- Given a colleague viewer, when they filter or sort by a non-whitelist field via API parameters, then the request is rejected with 400
- Given a colleague applies a saved view that references non-entitled columns, when the view loads, then non-entitled columns are omitted and the list renders without error

### Review Findings

- [x] [Review][Defer] Self-row AC #3 — **resolved**: pure-colleague catalog unions Self S4; S4 columns are display-only (no filter/sort); `years_with_company` excluded; per-row mask hides S4 on peer rows
- [x] [Review][Patch] `ListCatalogAccessService` duplicated `listGrantedSections` logic — fixed: now delegates to `SectionAccessGate.listGrantedSections` [`list-catalog-access.service.ts`]
- [x] [Review][Patch] S10 list enrichment included all leave periods, not active-only — fixed: filter to dates overlapping today in `EmployeeListLeavesService`
- [x] [Review][Patch] Integrated field enrichment gated on placeholder cell keys — fixed: enrich when column is visible and section grant is non-`none` [`employees.service.ts`]
- [x] [Review][Patch] Missing `ListCatalogAccessService` unit tests — added `list-catalog-access.service.spec.ts`
- [x] [Review][Patch] Missing S10/S11 enrichment unit test — added to `employees.service.spec.ts`
- [x] [Review][Patch] E2e gaps: colleague filter on `employment_type`, S10/S11 catalog fields — added to `employees.e2e-spec.ts` and `colleague-whitelist.e2e-spec.ts`
- [x] [Review][Defer] No frontend component unit tests for `EmployeeCardList` — **resolved**: Playwright coverage in `e2e/directory-card-layout.spec.ts` (mobile cards, desktop table, profile navigation)

## Design Notes

Catalog union algorithm: for each relationship class the viewer holds toward at least one active employee (Self, Reporting line, Project line, PP, Full access), resolve audience against one representative subject and union `listGrantedSections` results; add custom fields via `CustomFieldVisibilityService` where the viewer can see the definition toward any subject. Avoids O(n) per-employee catalog scans. Per-row masking still calls `resolveAudience` per returned row (acceptable at current scale; optimize in 3.7 if needed).

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm test -- --testPathPatterns=employees.service` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- employees.e2e-spec` — expected: colleague whitelist e2e pass (Postgres up)
- `cd services/backend && npm run test:e2e -- employees-export.e2e-spec` — expected: colleague export e2e pass
- `cd services/frontend && npm run typecheck` — expected: no TS errors
- `cd services/frontend && npm run lint` — expected: no errors
- `cd services/frontend && npm test -- e2e/directory-card-layout.spec.ts` — expected: card layout e2e pass

**Manual checks:**
- Log in as bootcamp colleague account; confirm network tab shows no `grade` in list response; resize below 768px and confirm card layout; apply a shared saved view with management columns and confirm they are omitted

## Spec Change Log

| Date | Iteration | Change |
|------|-----------|--------|
| 2026-09-08 | 1 | Full bmad-review: expanded I/O matrix (Self, project line, full access, saved views, integration down); resolved Ask First defaults; S16/catalog union clarity; Rule 8 mentor exclusion; saved-view column sanitization; inline-edit read-only rule; responsive breakpoint alignment; verification additions |
| 2026-09-08 | — | Code review: SectionAccessGate catalog union, active-leave filter, enrichment guard fix, list-catalog + S10/S11 tests, e2e coverage gaps closed; AC #3 Self-row deferred (filter side-channel) |
| 2026-09-08 | — | Deferred items closed: Self-only S4 catalog + display-only guard (AC #3); Playwright `directory-card-layout.spec.ts`; spec AC/matrix aligned |

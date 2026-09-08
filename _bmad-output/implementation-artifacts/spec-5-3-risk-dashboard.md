---
title: 'Risk Dashboard'
type: 'feature'
created: '2026-09-04'
status: 'in-progress'
review_loop_iteration: 0
baseline_commit: '6a1d5ee1f6679c548a90b98669dfe5ebd28e0325'
story_key: '5-3-risk-dashboard'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-5-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-5-2-show-risk-trend.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 5.1–5.2 deliver per-employee S6 risk history and trend, but there is no aggregated Risk Dashboard — managers and PPs cannot scan severity counts or drill through their population by level, filters, or profile link.

**Approach:** Add a backend `GET /risks/dashboard` (counts + paginated rows) and `GET /risks/dashboard/access` scoped via C1 to subjects where the viewer holds Reporting-line, Project-line, or PP access (S6 ≠ `none`); register `@RegisterProvider('dashboard-summary', 'risks')` sharing the same count logic for Epic 12. Build `pages/RiskDashboard/` with counter cards, comfortable-density table (`RiskBadge` + `TrendArrow`), URL-driven filters, and nav gated on the access probe.

## Boundaries & Constraints

**Always:**
- **Audience:** include only employees where C1 grants S6 (`ReportingLine`, `ProjectLine`, or `PP` over the subject). Exclude Self-only and Colleague viewers entirely — **403** on dashboard routes; never return the viewer's own risk row.
- **Active risk:** level above `low` (FR-26). Count cards show `need_attention`, `medium`, `high`, `leaver` plus `totalActive`; `low` and no-history employees appear in the table only when unfiltered, never in active counters.
- **Counts + rows:** derive `currentLevel`, `trend`, and `recordedAt` from each subject's latest `RiskRecord` using the same sort as `risks.service.ts` (`recordedAt desc`, `createdAt desc`, `id desc`) and `computeRiskTrend` from `risk-input.ts`.
- **Default table sort:** severity desc (`leaver` → `low` per `RISK_LEVELS`), then `recordedAt` desc. Clicking a counter card sets `level` filter and refetches.
- **Filters (query params):** `departmentId`, `managerId`, `peoplePartnerId`, `projectId` — AND-combined, applied after audience scoping. Department uses current `DepartmentHistory`; manager/PP via `Employee` FKs; project via active confirmed `ProjectAssignment` rows (same freshness rules as C1 project-line).
- **Row drill-through:** row click navigates to `/employees/:employeeId`.
- **API DTO:** `{ counts, rows: [{ employeeId, displayName, currentLevel, trend?, recordedAt, department?, managerName?, peoplePartnerName? }], total, page, pageSize }`.
- **Frontend:** `table-row-comfortable` density; counter cards visually emphasise `medium`/`high`/`leaver`; empty copy `"No risk history recorded."` — no add-risk CTA. Reuse `RiskBadge` + `TrendArrow`; i18n keys under `riskDashboard.*`.
- **Nav:** SideMenu "Risks" item visible only when `GET /risks/dashboard/access` returns `{ canAccess: true }`; direct URL without access redirects home (no permission-denied page).
- **a11y:** severity never color-only; trend direction via existing aria labels.

**Ask First:**
- If extracting a shared `FilterAudienceBuilder` component blocks delivery, ship page-local filter controls mirroring All Employees URL-param patterns and defer shared extraction until Campaign audience (FR-41).

**Never:**
- Client-side audience or trend derivation; functional-role permission widening data access; All Employees risk column (`FieldProvider`, Epic 3); role Home dashboard widgets (Epic 12); update/delete risk records; conflate `leaver` with `dismissed`; duplicate badge/arrow markup.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PP with scoped people | 3 assigned at `high`, 1 at `low` | `counts.high === 3`, `totalActive === 3`; table lists all 4 when unfiltered | N/A |
| Counter drill-through | Click `high` card | `level=high` query applied; table shows only high subjects, severity sort preserved | N/A |
| Colleague-only viewer | No Manager/PP/Project-line over anyone | `canAccess: false`; `GET /risks/dashboard` → **403**; nav hidden | N/A |
| Self row exclusion | Viewer is PP for self + others | Dashboard never includes viewer's own row even if self has risk records | N/A |
| Filter intersection | `departmentId` + `managerId` both set | Only subjects matching both within audience | 400 on unknown UUID filter ids |
| No history | Subject in audience, zero records | Row omitted from table; not counted in level cards | N/A |
| Row navigation | Click table row | Navigate to employee profile | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/contracts/section-access-gate.contract.ts` — extend with `canAccessRiskDashboard(viewerId)` + `listS6SubjectIds(viewerId)` (exclude self) — C1-scoped audience listing without cross-module imports
- `services/backend/src/modules/access/section-access-gate.service.ts` — implement inverse Reporting/PP/Project-line subject queries reusing `AccessResolver` / `ProjectAssignment` patterns from `access-resolver.service.ts` L346–557
- `services/backend/src/modules/contracts/dashboard-summary-provider.contract.ts` (new) — `DashboardSummaryProvider.getSummary(viewerEmployeeId)` per AD-3
- `services/backend/src/modules/risks/risks-dashboard.service.ts` (new) — aggregate counts, filter/sort/paginate rows over scoped subjects + latest records
- `services/backend/src/modules/risks/risks-dashboard.controller.ts` (new) — `GET /risks/dashboard`, `GET /risks/dashboard/access`
- `services/backend/src/modules/risks/risks-dashboard-summary.provider.ts` (new) — `@RegisterProvider('dashboard-summary', 'risks')`; shares count logic with dashboard service
- `services/backend/src/modules/risks/entities/risk-dashboard.entity.ts` (new) — Swagger entities for dashboard DTOs
- `services/backend/src/modules/risks/risk-input.ts` L10–16 — `RISK_LEVELS` severity ordering for sort + active-risk definition
- `services/backend/src/modules/risks/risks.service.ts` L73–82 — `toSectionDto` trend derivation pattern to mirror per row
- `services/backend/src/modules/risks/risks.module.ts` — register dashboard controller, service, summary provider
- `services/backend/src/modules/risks/__tests__/risks-dashboard.service.spec.ts` — unit tests for counts, filters, sort, self-exclusion
- `services/backend/test/risks-dashboard.e2e-spec.ts` — PP scoped counts + drill-through; colleague 403; self omission
- `services/frontend/src/pages/RiskDashboard/` (new) — page shell, counter cards, table, filters hook
- `services/frontend/src/api/services/risk.service.ts` — add dashboard + access client methods
- `services/frontend/src/hooks/data/useRiskDashboardData.ts` (new) — TanStack Query hooks
- `services/frontend/src/components/RiskBadge/RiskBadge.tsx` + `TrendArrow/TrendArrow.tsx` — reuse unchanged
- `services/frontend/src/router/index.tsx` L36–64 — add `/risks` route
- `services/frontend/src/components/SideMenu/SideMenu.tsx` L43–60 — Risks nav item gated on access probe
- `services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts` — URL search-param filter/sort pattern to mirror
- `services/frontend/e2e/risk-dashboard.spec.ts` (new) — counter filter + row navigation smoke
- `services/frontend/src/locales/en/translation.json` — `riskDashboard.*`, `sidebar.risks`

**Read-only (5.1/5.2):** append-only model, per-employee GET/POST routes, S6 provider, trend computation helper.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/section-access-gate.contract.ts` + `access/section-access-gate.service.ts` — S6 audience listing + dashboard access probe — C1 gate without N× resolve
- [x] `services/backend/src/modules/contracts/dashboard-summary-provider.contract.ts` — AD-3 summary contract — registry family
- [x] `services/backend/src/modules/risks/risks-dashboard.service.ts` — counts, filters, severity sort, pagination — core aggregation
- [x] `services/backend/src/modules/risks/risks-dashboard-summary.provider.ts` — `@RegisterProvider('dashboard-summary', 'risks')` — Epic 12 hook
- [x] `services/backend/src/modules/risks/risks-dashboard.controller.ts` + entities + swagger — REST surface
- [x] `services/backend/src/modules/risks/risks.module.ts` — wire dashboard providers/controllers
- [x] `services/backend/src/modules/risks/__tests__/risks-dashboard.service.spec.ts` — unit coverage for matrix rows
- [x] `services/backend/test/risks-dashboard.e2e-spec.ts` — PP counts, colleague deny, self exclusion — epic AC
- [x] `services/frontend/src/pages/RiskDashboard/` — counter cards, table, filters, empty state — UJ-1 drill-through UX
- [x] `services/frontend/src/api/services/risk.service.ts` + `hooks/data/useRiskDashboardData.ts` — typed client + queries
- [x] `services/frontend/src/router/index.tsx` + `SideMenu.tsx` + `translation.json` — route, gated nav, i18n
- [x] `services/frontend/e2e/risk-dashboard.spec.ts` — Playwright counter + row click

**Acceptance Criteria:**
- Given I am a PP with assigned employees at `high` risk within my scope, when I open the Risk Dashboard, then I see `high` count scoped to my people, clicking it filters the table to those employees sorted by severity then date, and clicking a row opens the profile
- Given I am an ordinary employee with no Manager/PP/Project-line relationship over anyone, when I navigate to `/risks` or call `GET /risks/dashboard`, then I am redirected home (or receive 403) and the response exposes no risk data for myself or others
- Given a subject in my audience has no risk records, when I view the unfiltered table, then they do not appear and are not counted in any level card

## Spec Change Log

## Design Notes

Audience resolution lives in `access` (new inverse queries on `Employee.managerId`, `peoplePartnerId`, reporting-line closure, and confirmed project assignments) — the `risks` module consumes the contract only. Counter cards use shadcn `Card` with risk-token borders on `medium`/`high`/`leaver`; `need_attention` uses default card styling. Filter UI is a toolbar of `Select` controls (not the not-yet-built shared Filter/Audience Builder); filter state serializes to URL search params like All Employees.

## Verification

**Commands:**
- `cd services/backend && npm test -- risks-dashboard` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- risks-dashboard` — expected: PP scope + colleague 403 pass
- `cd services/frontend && npm run lint && npm run typecheck` — expected: pass
- `cd services/frontend && npm run test -- risk-dashboard` — expected: Playwright spec passes

**Manual checks:**
- Log in as PP seed user → Risks nav visible → counter cards match scoped population → click `high` filters table → row opens profile. Colleague seed user → no Risks nav, `/risks` redirects home.

---
title: 'Build Shared Dashboard Engine'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 2
baseline_commit: '2f04487efde50b9c020c220221c64c28d3fa0cc8'
story_key: '12-1-build-shared-dashboard-engine'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-12-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-5-3-risk-dashboard.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/dashboard-um.html'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Four role home dashboards (UM/DM/PM/PP) are specified (**FR-53**; role-specific counters in FR-54–57 land in 12.2–12.5) but no shared engine exists — only a placeholder `HomePage` and a separate Risk Dashboard with page-local counter/table components. Without a config-driven shell and backend orchestrator, Stories 12.2–12.5 would duplicate layout and access logic.

**Approach:** Add a backend `dashboards` module that resolves role config, scopes audience via C1 (no bypass), and merges `DashboardSummaryProvider` fragments from the registry **for that variant's subject set**. Build frontend shared widgets (counter grid, scoped table, own action items, quick nav) plus a `DashboardEngine` shell driven by typed config — prove with UM (people-grouped) and DM (project-grouped) variants on `/`, reusing existing risk summary and authored-action-items feeds; defer full counter sets and project selector to 12.2–12.5.

## Boundaries & Constraints

**Always:**
- **Access:** every population query goes through `dashboard-audience` + C1 contracts — UM uses `listManagerSubordinateIds(viewerId)` (transitive reporting-line descendants only; **excludes the viewer**; never PP- or project-line-only subjects). DM proof uses `listProjectGroups(viewerId)` scoped to project-responsibility subjects per FR-1. No client-side audience filtering.
- **Provider merge:** orchestrator consumes `@RegisterProvider('dashboard-summary', id)` via `ProviderRegistryService.get()` — same unavailable/available pattern as `ProfileAssemblerService.loadSection`. Reuse existing `risks` provider logic but **filter or re-aggregate counts/rows to the variant-scoped subject IDs** before merging (the registrant's default `getSummary(viewerId)` uses `listS6SubjectIds`, which is too broad for UM). Missing registry entries return `{ status: 'unavailable' }` and render fail-soft placeholders; caught provider errors (e.g. `ForbiddenException` when a provider rejects a viewer) map to unavailable fragments for that counter/column — never fabricated zeros and never widened access.
- **Own action items:** reuse `GET /api/v1/me/authored-action-items` — server already omits items whose assignee no longer has non-`none` S14 from the viewer; sorted by due date ascending; overdue uses server `isOverdue` + `OverdueIndicator`.
- **Table:** comfortable density (`table-row-comfortable`); risk column uses `RiskBadge` + `TrendArrow`; row click → `/employees/:id`. Leave/project columns show "temporarily unavailable" when provider missing. No pagination in 12.1 — render all scoped rows returned by the summary endpoint (pagination deferred to 12.2+).
- **Config contract:** `{ variant: 'um'|'dm'|'pm'|'pp', grouping: 'people'|'project', blocks: BlockId[], counters: CounterSpec[] }` where `CounterSpec = { id: string, providerId: string, labelKey: string }`. UM and DM configs ship in 12.1 as proof; PM/PP configs are stubs declaring grouping/blocks only.
- **Variant resolution:** backend maps viewer → variant from seed fixtures until functional-role UI gates land. **Priority when multiple match:** UM → DM → PM → PP. Response includes `{ variant, resolvedBy: 'seed-map'|'functional-role' }` for testability.
- **Frontend reuse:** extract shared components from `pages/RiskDashboardPage/` patterns into `components/DashboardEngine/` — do not refactor Risk Dashboard route in this story.
- **Home route:** replace `HomePage` with `DashboardPage` at `/`; variant comes only from `GET /api/v1/dashboards/config`.
- **API:** `GET /api/v1/dashboards/config` (viewer's variant + layout metadata), `GET /api/v1/dashboards/summary` (merged counters + table rows for scoped population). Swagger entities required. Frontend loads config first; if config is 403, do not call summary.
- **Tests:** parameterized access-matrix for UM subordinate scoping; unit tests for config merge, provider-unavailable handling, and S6-wide risk provider output filtered to UM scope; e2e smoke proving UM and DM render the same counter/table/action-item components with different grouping metadata.

**Ask First:**
- If extracting `RiskCounterCards` in place blocks delivery, copy the Card layout into `CounterTileGrid` and leave Risk Dashboard unchanged — do not rewrite 5.3.
- If server-side variant detection needs functional-role permissions not yet wired in frontend auth, ship a backend-only role resolver keyed to seed employee fixtures and document the gap in Design Notes.

**Never:**
- Four independently coded dashboard pages; dashboard-specific access bypass; duplicate overdue/risk badge markup; full UM/DM/PM/PP counter catalogs (12.2–12.5); project selector filter logic (12.3); PP resourcing exclusion widget (12.5); Risk Dashboard route changes; a11y/responsive audit (12.6).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| UM with 5 subordinates, 2 at active risk | `variant=um`, viewer is UM over 5 via reporting line | `summary.counters.headcount === 5`; table lists exactly 5 rows; risk counters reflect 2 active records scoped to those 5 only | N/A |
| UM also holds PP over outsiders | Viewer is UM for 5 reporting-line subordinates and PP for 10 others | Table and headcount still 5 — PP-only subjects absent | N/A |
| Non-subordinate employee | Person outside UM reporting closure | Absent from table and headcount; never flashes in response | N/A |
| UM with zero subordinates | Valid UM variant, empty reporting closure | `headcount === 0`; empty table with standard empty copy; config still 200 | N/A |
| UM vs DM same engine | Both configs requested | Identical React component tree (`CounterTileGrid`, `ScopedPeopleTable`, `OwnActionItemsWidget`); differs in `grouping` + `blocks` metadata only | N/A |
| DM with projects, no selector | `variant=dm`, viewer responsible for 2 projects | Summary includes project-group metadata and rows partitioned by project; empty project groups allowed | N/A |
| Missing leave provider | Registry lookup unavailable | Table leave column shows i18n unavailable copy; leave counters omitted or unavailable-styled | N/A |
| Registered provider throws | `risks` provider returns 403 for viewer | Orchestrator maps that fragment to unavailable — no 500 on summary endpoint | N/A |
| Authored items overdue | One item past due date | Widget lists item with `OverdueIndicator`; sorted by due date | N/A |
| Authored item, assignee lost S14 | Open item authored by viewer but assignee now Colleague-only | Item omitted from widget (same as `/me/authored-action-items`) | N/A |
| Multi-role seed employee | Matches UM and DM maps | Config returns higher-priority UM variant per resolver | N/A |
| Viewer with no dashboard role | No matching variant | `GET /api/v1/dashboards/config` → **403**; `/` shows existing access-denied pattern; summary not called | N/A |
| Config/summary ordering | Frontend mounts `/` | Config fetched before summary; summary uses config variant only | Summary skipped on config 403 |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/dashboards/` (new) — orchestrator module; depends on `contracts` + `registry` only
- `services/backend/src/modules/contracts/dashboard-audience.contract.ts` (new) — `listManagerSubordinateIds(viewerId)`, `listProjectGroups(viewerId)` for DM grouping scaffold
- `services/backend/src/modules/access/section-access-gate.service.ts` L125–144 — private `listReportingLineDescendantIds` pattern; expose Manager-reach listing via dashboard-audience implementation (not on `SectionAccessGate` public API)
- `services/backend/src/modules/contracts/dashboard-summary-provider.contract.ts` — tighten return typing with shared `DashboardSummaryFragment` union; optional overload or orchestrator-side filter for scoped subject IDs
- `services/backend/src/modules/registry/provider-registry.service.ts` L119–128 — `get('dashboard-summary', id)` consumer pattern
- `services/backend/src/modules/risks/risks-dashboard-summary.provider.ts` L8–22 — existing registrant; orchestrator re-scopes counts/rows to variant audience (provider default uses `listS6SubjectIds` — too broad for UM)
- `services/backend/src/modules/risks/risks-dashboard.service.ts` — read-only reference for aggregation + active-risk definition
- `services/backend/src/modules/action-items/action-items.controller.ts` L86 — `GET me/authored-action-items` feed for own-items widget
- `services/frontend/src/components/DashboardEngine/` (new) — `CounterTileGrid`, `ScopedPeopleTable`, `OwnActionItemsWidget`, `QuickNavLinks`, `DashboardEngineShell`
- `services/frontend/src/pages/DashboardPage/` (new) — replaces `HomePage`; hooks layer per `react-pages.md`
- `services/frontend/src/pages/RiskDashboardPage/components/RiskCounterCards/` — layout reference to generalize, not modify
- `services/frontend/src/components/RiskBadge/`, `TrendArrow/`, `OverdueIndicator/` — reuse unchanged
- `services/frontend/src/router/index.tsx` — `/` → `DashboardPage`
- `services/frontend/src/api/services/dashboard.service.ts` (new) + `hooks/data/useDashboardData.ts` — TanStack Query layer (config query gates summary query)
- `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/dashboard-um.html` — composition target

**Read-only:** `RiskDashboardPage/` route, `risks-dashboard.controller.ts`, Epic 5 e2e specs.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/dashboard-audience.contract.ts` + `access/` implementation — reporting-line subordinate listing for UM scope — C1 gate without dashboard bypass
- [x] `services/backend/src/modules/dashboards/` — config resolver (incl. multi-role priority), summary orchestrator with variant-scoped provider merge, controller (`GET config`, `GET summary`), entities — AD-1/AD-3 consumer
- [x] `services/backend/src/modules/dashboards/__tests__/` — unit tests for UM audience scoping (incl. PP-only exclusion), provider-unavailable merge, S6 provider output filtered to UM subjects, config variants
- [x] `services/backend/test/dashboards.e2e-spec.ts` — UM subordinate isolation; DM grouping metadata; 403 for unscoped viewer; config-before-summary behavior
- [x] `services/frontend/src/components/DashboardEngine/` — shared counter/table/action-items/quick-nav widgets — reusable across 12.2–12.5
- [x] `services/frontend/src/pages/DashboardPage/` + `api/services/dashboard.service.ts` + `hooks/data/useDashboardData.ts` — Home dashboard shell wired to backend; summary query enabled only when config succeeds
- [x] `services/frontend/src/router/index.tsx` — replace `HomePage` with `DashboardPage`; i18n keys under `dashboard.*`
- [x] `services/frontend/e2e/dashboard-engine.spec.ts` — UM table row count matches seed subordinates; shared components render for UM and DM configs

**Acceptance Criteria:**
- Given the dashboard engine is configured for both UM and DM variants, when each dashboard renders, then both use the same counter-tile, table, and action-item-widget components, differing only in grouping metadata and which blocks are configured to appear
- Given a UM opens their dashboard, when the subordinates table and counters render, then only people the UM holds Manager access over via the reporting hierarchy appear (excluding self and PP/project-line-only subjects), with no dashboard-specific bypass of the standard access-resolution layer
- Given a dashboard-summary provider is not registered or rejects the viewer for the scoped merge, when the summary endpoint is called, then affected counters/columns show the fail-soft unavailable treatment and no fabricated aggregate values
- Given the viewer has authored action items including one overdue item whose assignee still has non-`none` S14 access, when the own-action-items widget renders, then items are sorted by due date and the overdue item displays the `OverdueIndicator` treatment

## Spec Change Log

| Date | Change |
|------|--------|
| 2026-09-08 | Spec review pass: FR-53 traceability; `/api/v1` routes; variant-scoped provider merge; multi-role priority; expanded edge matrix |
| 2026-09-08 | Code review fixes: project-assignment freshness in DM/PM audience; PM vs DM responsibility param; registry-unavailable + audience unit tests; multi-role e2e; overdue row border |

## Review Findings

### Review Findings (2026-09-08 code review)

- [x] [Review][Patch] `listProjectGroups` omitted confirmed/freshness/active-window filters — unconfirmed rows appeared on DM dashboard [`dashboard-audience.service.ts`]
- [x] [Review][Patch] PM variant reused DM (`dmId`) project scope — added `responsibility: 'dm' | 'pm'` to audience contract and orchestrator [`dashboard-audience.contract.ts`, `dashboards.service.ts`]
- [x] [Review][Patch] Missing unit test for unregistered `dashboard-summary` provider merge [`dashboards.service.spec.ts`]
- [x] [Review][Patch] Missing `DashboardAudienceService` unit coverage for reporting-line listing and assignment filtering [`dashboard-audience.service.spec.ts`]
- [x] [Review][Patch] Missing e2e for UM-over-DM multi-role variant priority [`dashboards.e2e-spec.ts`]
- [x] [Review][Patch] DM e2e fixture `confirmedAt` fell outside 4h freshness window after filter landed [`dashboards.e2e-spec.ts`]
- [x] [Review][Patch] Overdue action-item rows lacked widget-level left-border treatment [`OwnActionItemsWidget.tsx`]
- [x] [Review][Defer] `getScopedData` trusts orchestrator-scoped `subjectIds` without re-validating S6 per subject [`risks-dashboard.service.ts`] — deferred; orchestrator is the sole caller via AD-1 boundary

## Design Notes

Backend variant resolution maps seed employees to `um`/`dm` configs until functional-role permission wiring selects the variant in production; document the seed map in test fixtures. DM proof ships project-grouped table **structure** (group headers + empty groups allowed) without the 12.3 project-selector filter. Counter tile grid uses shadcn `Card` matching `dashboard-um.html`; quick nav links are static config arrays pointing at existing routes (`/employees`, `/risks`, `/campaigns`, etc.).

## Verification

**Commands:**
- `cd services/backend && npm test -- dashboards` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- dashboards` — expected: UM scope + 403 cases pass
- `cd services/frontend && npm run lint && npm run typecheck` — expected: pass
- `cd services/frontend && npm run test -- dashboard-engine` — expected: Playwright smoke passes

**Manual checks:**
- Log in as UM seed user → `/` shows counter grid + subordinate table scoped to reporting line (no self row, no PP-only people) → action items widget lists authored items with overdue styling. Switch to DM seed → same component set, project-grouped layout metadata visible. Colleague-only seed → `/` access denied, no dashboard data flash.

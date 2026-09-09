---
title: 'Unit Manager Dashboard'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'affad9d41205290c67b326e60fbbbb69bac1e3db'
story_key: '12-2-unit-manager-dashboard'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-12-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-1-build-shared-dashboard-engine.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/dashboard-um.html'
  - '{project-root}/docs/project-requirements.md#441-unit-manager-dashboard--grouped-by-people'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 12.1 delivered a proof UM variant — headcount plus a single `totalActive` risk tile, unavailable leave/project columns, and three hardcoded quick-nav links. **FR-54** (PRD §5.12) and requirements **§4.4.1** require the full UM dashboard: per-level active-risk counters (`need_attention`, `medium`, `high`, `leaver`), team open/overdue action-item counters, resourcing and campaign counters, a populated subordinates table, six quick-nav links, and table pagination.

**Approach:** Extend the UM variant config and dashboards orchestrator with new scoped `dashboard-summary` providers; map per-level risk counters from the existing `risks` fragment; expose `quickNav` on the config response; add optional pagination to `GET /api/v1/dashboards/summary`. Reuse 12.1 `DashboardEngine` widgets — configuration and provider wiring only, no new page.

## Boundaries & Constraints

**Always:**
- **Scope:** `listManagerSubordinateIds(viewerId)` — reporting-line descendants only; excludes self and PP/project-line-only subjects. All aggregates are server-side over the full scoped population.
- **AD-1 boundary:** the `dashboards` module imports `contracts` + `registry` only. New aggregate providers live in their feature modules (`action-items`, `resourcing`, `campaigns`, `directory`) and register via `@RegisterProvider('dashboard-summary', id)`.
- **Provider merge:** `{ subjectIds }` passed into every provider call; missing registry entry or caught provider error (including `ForbiddenException`) → `{ status: 'unavailable' }` for that counter/column — never fabricated zeros and never widened access (same fail-soft pattern as 12.1).
- **UM counter catalog (nine tiles):** `headcount` (`audience`); `need_attention`, `medium`, `high`, `leaver` (`risks` — exclude `low`, matching `ACTIVE_LEVELS` in `risks-dashboard.service.ts`); `openActionItems`, `overdueActionItems` (`action-items`); `openResourcingRequests` (`resourcing`); `openCampaigns` (`campaigns`). Remove the 12.1 proof `totalActive` counter.
- **Action-item counters:** count `status: 'open'` items where `assigneeId ∈ subjectIds`; overdue uses the same server derivation as Story 4.3 (`isOverdue` on the action-item entity). Own-action-items widget stays on `GET /api/v1/me/authored-action-items`.
- **Resourcing counter:** count of `listAssigned(viewerId)` results with `status: 'open'` (Story 6.2 live routing).
- **Campaigns counter:** count of `FormCampaign` with `status: 'active'` and `creatorId = viewerId` (UM's own open campaigns; Epic 10).
- **Table:** all scoped subordinates appear (including `low` or no risk); risk column uses `RiskBadge` + `TrendArrow`; leave via a `leave` provider wrapping `EmployeeListLeavesReader.formatListCell`; project via an `employment` provider using the integrated `project_names` field (Story 3.6). Integration down → unavailable cell copy; sync lag with last-known data → AD-8 stale indicator (distinct from unavailable).
- **Quick nav (§4.4.1, six links on config):** All Employees `/employees`; Saved views `/employees` (same route, distinct `labelKey`; optional `?savedViews=open` query param — see Ask First); Resourcing `/resourcing`; Risk dashboard `/risks`; Mentorship hub `/mentorship`; Campaigns `/campaigns`. Frontend renders from `config.quickNav`, not a hardcoded array.
- **Pagination (UM `people` grouping only):** optional `page` (default 1, min 1) and `pageSize` (default 50, max 100) on `GET /api/v1/dashboards/summary`; counters aggregate the full scoped population; `rows` are the paginated slice; response includes `pagination: { page, pageSize, totalRows }`. Invalid `page`/`pageSize` → 400; `page` beyond last page → 200 with empty `rows`.
- **Layout:** `CounterTileGrid` uses a responsive grid that fits nine tiles (e.g. `sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-5`); mockup `dashboard-um.html` is a simplified five-tile composition reference, not the FR-54 counter set.

**Ask First:**
- Multi-project cell display (comma-join vs first project only).
- Saved-views deep-link query param (`?savedViews=open` or equivalent) if UX wants the picker to auto-open.

**Never:** Separate UM page; dashboard access bypass; DM/PM/PP variant changes (12.3–12.5); Risk Dashboard route changes; a11y pass (12.6); mentorship counters.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| 12 subordinates, 3 active risks | UM reporting closure | `headcount=12`; level counters sum to 3; table lists 12 rows with risk on 3 | N/A |
| PP-only outsiders | UM over 5, PP over 10 others | Scope stays 5 | N/A |
| Zero subordinates | Valid UM, empty closure | `headcount=0`; level/action-item/resourcing counters `0` when providers available; empty table with standard empty copy; config still 200 | N/A |
| Team action items | 4 open, 1 overdue on subordinates | `openActionItems=4`, `overdueActionItems=1` | N/A |
| Integration healthy | Leave/employment providers available | Table leave/project cells populated | N/A |
| Integration down | Leave reader unavailable | Unavailable cell; summary 200 | N/A |
| Sync lag (AD-8) | Leave/project stale but last-known present | Cell shows last-known value with stale indicator | N/A |
| Partial integration | Leave up, project down | Leave populated; project unavailable | N/A |
| Pagination | 60 subs, `page=2`, `pageSize=50` | 10 rows; counters still over 60 | `page<1` or `pageSize` out of range → 400 |
| Provider throws | `campaigns` provider rejects viewer | Campaign counter unavailable; other counters unaffected | No 500 on summary |
| Authored overdue | UM-authored overdue for subordinate with S14 | Widget shows `OverdueIndicator` | N/A |
| Low-risk subordinate | Employee at `low` risk | Appears in table with badge; excluded from active level counters | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/dashboards/dashboard-variant-config.ts` — full UM nine-counter specs + `quickNav`; remove proof `totalActive`
- `services/backend/src/modules/dashboards/dashboards.service.ts` — multi-provider merge, per-level risk counter mapping, pagination, leave/project row enrichment
- `services/backend/src/modules/dashboards/dashboards.controller.ts` — pagination query DTO on `GET /api/v1/dashboards/summary`
- `services/backend/src/modules/dashboards/entities/dashboard-config.entity.ts` — add `quickNav: DashboardQuickNavLinkEntity[]`
- `services/backend/src/modules/dashboards/entities/dashboard-summary.entity.ts` — add `pagination` object on UM responses
- `services/backend/src/modules/contracts/dashboard-summary.types.ts` — fragment types for `action-items`, `resourcing`, `campaigns`, `leave`, `employment`
- `services/backend/src/modules/risks/risks-dashboard-summary.provider.ts` — existing scoped risks; orchestrator maps `counts.need_attention|medium|high|leaver`
- `services/backend/src/modules/action-items/` (Epic 4) — new `action-items-dashboard-summary.provider.ts`; reuse Story 4.3 overdue derivation
- `services/backend/src/modules/resourcing/resourcing.service.ts` L124–154 — `listAssigned` for resourcing counter provider
- `services/backend/src/modules/campaigns/` (Epic 10) — new `campaigns-dashboard-summary.provider.ts`; active campaigns by `creatorId`
- `services/backend/src/modules/contracts/employee-list-leaves.contract.ts` + `directory/employees.service.ts` L475+ — leave provider pattern
- `services/backend/src/modules/contracts/field-registry.contract.ts` L96 — `project_names` for employment provider
- `services/backend/src/modules/access/dashboard-audience.service.ts` L22–26 — unchanged UM scope
- `services/frontend/src/components/DashboardEngine/CounterTileGrid/` — responsive grid for nine tiles
- `services/frontend/src/components/DashboardEngine/QuickNavLinks/` — render `config.quickNav` from API
- `services/frontend/src/components/DashboardEngine/ScopedPeopleTable/` — leave/project cells + pagination controls
- `services/frontend/src/pages/DashboardPage/` + `api/services/dashboard.service.ts` — pagination query params + config wiring
- `services/backend/test/dashboards.e2e-spec.ts`, `services/frontend/e2e/dashboard-engine.spec.ts` — extend UM scenarios

## Tasks & Acceptance

**Execution:**
- [x] `contracts/dashboard-summary.types.ts` — fragment types for `action-items`, `resourcing`, `campaigns`, `leave`, `employment`
- [x] New providers: `action-items-dashboard-summary.provider.ts`, `resourcing-dashboard-summary.provider.ts`, `campaigns-dashboard-summary.provider.ts`, `leave-dashboard-summary.provider.ts`, `employment-dashboard-summary.provider.ts` — register in respective modules
- [x] `dashboard-variant-config.ts` + `dashboards.service.ts` + controller + config entity — UM counter specs, per-level risk mapping, orchestration, pagination, `config.quickNav`
- [x] `dashboards/__tests__/` + `test/dashboards.e2e-spec.ts` — counter merge, pagination, scope matrix, per-provider unavailable paths
- [x] `DashboardEngine/` + `DashboardPage/` + i18n — config-driven nav, nine-tile grid, table cells, pagination controls
- [x] `e2e/dashboard-engine.spec.ts` — UM counters, six nav links, leave/project columns, pagination

**Acceptance Criteria:**
- Given a UM has 12 subordinates and three with active risk (`need_attention`, `medium`, `high`, or `leaver`), when they open the dashboard, then `headcount` is 12, each active level counter reflects only those three records, and the table lists all 12 with risk, project, and leave status (including subordinates at `low` or without risk)
- Given subordinates have 4 open action items and 1 overdue, when the UM opens the dashboard, then `openActionItems` is 4 and `overdueActionItems` is 1
- Given the UM has 2 open routed resourcing requests, when they open the dashboard, then `openResourcingRequests` is 2
- Given the UM has 1 active campaign they created, when they open the dashboard, then `openCampaigns` is 1
- Given the UM authored an overdue action item for a subordinate with non-`none` S14, when they open the dashboard, then it appears in the own-action-items widget sorted by due date with Story 4.3 overdue styling
- Given the dashboard renders, when quick nav is shown, then all six §4.4.1 links are present, labeled via i18n, and navigate to the configured paths
- Given more subordinates than `pageSize`, when paginating the table, then counters reflect the full scoped population and `pagination.totalRows` matches headcount
- Given a `dashboard-summary` provider is unregistered or throws, when summary is fetched, then only that provider's counters/columns are unavailable — no fabricated values and no 500

## Spec Change Log

| Date | Change |
|------|--------|
| 2026-09-09 | Full review pass: nine-tile counter catalog; `quickNav` on config; pagination contract; AD-1/AD-8; expanded edge matrix and acceptance; code-map path alignment |
| 2026-09-09 | Code review pass: employment AD-8 stale/freshness; table-provider gating; empty-cell UI; swagger 400; unit/e2e coverage expanded |
| 2026-09-09 | Deferred follow-up: leave AD-8 stale via sync cache fallback; UM pagination before leave/employment cell enrichment |
| 2026-09-09 | Story complete: per-provider unit tests; quick-nav click-through e2e; backend dashboards e2e verified (5/5; global teardown matrix gap pre-existing) |

## Review Findings

### Review Findings (2026-09-09 spec review)

- [x] [Review][Spec] Nine-tile counter catalog and per-level risk mapping documented [`Boundaries`, `Tasks`]
- [x] [Review][Spec] `quickNav` exposed on config; frontend reads from API [`Boundaries`, `Code Map`]
- [x] [Review][Spec] Pagination query/response contract and validation rules [`Boundaries`, `I/O matrix`]
- [x] [Review][Spec] Provider fail-soft and AD-1 module boundary restated [`Boundaries`]
- [x] [Review][Spec] Campaigns counter scoped to viewer-created active campaigns [`Boundaries`]
- [x] [Review][Spec] Leave vs employment provider split; AD-8 stale vs unavailable [`Boundaries`, `I/O matrix`]
- [x] [Review][Spec] Acceptance criteria cover all counter types, nav, pagination, unavailable paths [`Tasks & Acceptance`]

### Review Findings (2026-09-09 code review)

- [x] [Review][Patch] Empty project/leave labels rendered blank instead of em dash [`ScopedPeopleTable.tsx`]
- [x] [Review][Patch] Employment provider stale logic uses AD-8 `confirmedAt` freshness window [`employment-dashboard-summary.provider.ts`]
- [x] [Review][Patch] Leave/employment providers loaded only when config includes `table` block [`dashboards.service.ts`]
- [x] [Review][Patch] Swagger documents 400 for invalid pagination query [`dashboards.swagger.ts`]
- [x] [Review][Patch] Unit tests cover page-beyond-last and resourcing/campaigns happy path [`dashboards.service.spec.ts`]
- [x] [Review][Patch] Backend e2e asserts nine counters, six quick-nav links, pagination metadata [`dashboards.e2e-spec.ts`]
- [x] [Review][Patch] Frontend e2e asserts all nine counters, leave/project cells, pagination controls [`dashboard-engine.spec.ts`]
- [x] [Review][Patch] Leave AD-8 stale indicator propagated via `EmployeeListLeaveCell.stale` + expired-cache fallback in `LeavesSyncService` [`employee-list-leaves.contract.ts`, `leaves-sync.service.ts`, `leave-dashboard-summary.provider.ts`]
- [x] [Review][Patch] Per-provider unit tests for five new dashboard-summary providers [`action-items`, `resourcing`, `campaigns`, `leave`, `employment` provider specs]
- [x] [Review][Patch] Paginate UM `subjectIds` before leave/employment enrichment — table providers receive page slice only [`dashboards.service.ts`]
- [x] [Review][Patch] Quick-nav e2e click-through navigation to all six destination routes [`dashboard-engine.spec.ts`]

## Design Notes

`dashboard-um.html` shows five simplified counters (combined high/leaver); FR-54 requires nine — use the mockup for table/action-item layout only. `CounterTileGrid` currently uses `xl:grid-cols-5`; extend for nine tiles. Campaigns provider depends on Epic 10 module landing; until then the counter shows unavailable (AD-3). Saved views and All Employees share `/employees` until UX confirms a deep-link query param.

## Verification

**Commands:**
- `cd services/backend && npm test -- dashboards` — unit tests pass
- `cd services/backend && npm run test:e2e -- dashboards` — UM scenarios pass
- `cd services/frontend && npm run lint && npm run typecheck` — pass
- `cd services/frontend && npm run test -- dashboard-engine` — Playwright pass

**Manual checks:**
- UM seed on `/` — nine counters, six nav links, leave/project when integrations up, pagination when team > 50; PP-only people never appear; `page` beyond last returns empty rows without error.

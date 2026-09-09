---
title: 'Delivery Manager Dashboard with Project Selector'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 1
baseline_commit: '216d31d6463690c1693630668b47566ecaf52af5'
story_key: '12-3-delivery-manager-dashboard-with-project-selector'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-12-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-1-build-shared-dashboard-engine.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-2-unit-manager-dashboard.md'
  - '{project-root}/docs/project-requirements.md#442-delivery-manager-dashboard--grouped-by-project'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 12.1 shipped a DM proof variant — project-grouped tables and two counters (`headcount`, `totalActive`) with no project selector, no per-level risk breakdown, no resourcing counter or block, and no **Unassigned** bucket. **FR-49** (epics backlog) and requirements **§4.4.2** require a DM home dashboard where a project selector (default *All projects*) filters the **whole page** — tables and every counter — and a resourcing block shows the DM's own requests plus PM-created requests on their projects (Epic 6 visibility).

**Approach:** Extend the existing `dm` variant config and dashboards orchestrator with optional `projectId` scoping on `GET /api/v1/dashboards/summary`, the full DM counter catalog, an **Unassigned** virtual project in the selector, and a `resourcingRequests` dashboard block backed by a scoped resourcing summary provider. Add `ProjectSelector` and `ResourcingRequestsWidget` to `DashboardEngine`; wire DM-only — no PM/PP changes (12.4/12.5).

## Boundaries & Constraints

**Always:**
- **Scope:** DM audience via `listProjectGroups(viewerId, 'dm')` — active confirmed project assignments only (existing freshness rules in `dashboard-audience.service.ts`). People outside DM project responsibility never appear. Resourcing visibility follows `ResourcingService.listRequests` (author's own + PM-authored on DM-managed projects per Story 6.1); never `listAssigned` (UM routing).
- **AD-1:** `dashboards` imports `contracts` + `registry` only. Extend or add providers in `resourcing` / `risks` feature modules; orchestrator merges fragments with `{ subjectIds, projectId? }`.
- **Project selector:** optional `projectId` query on `GET /api/v1/dashboards/summary`. Omit, absent, or `all` → aggregate all DM projects + Unassigned resourcing in counters; filter tables to all project groups. Specific `projectId` → counters and people tables scoped to that project only. Sentinel `unassigned` → no people tables; headcount and per-level risk counters are `0` when providers are available; `openResourcingRequests` counts only requests with `projectId = null`; resourcing block lists only those requests. Empty string or whitespace `projectId` → 400.
- **Selector options:** config or summary exposes the DM project list for the selector: one entry per `listProjectGroups` result (sorted by `projectId` ascending, matching the audience service) plus **Unassigned** (i18n label). Default selection is *All projects* (no `projectId` param). Clearing selection restores the all-projects view without a full page reload (client state only). Project display name defaults to `projectId` until a lookup source exists (see Ask First).
- **Headcount deduplication:** in the all-projects view, `headcount` counts unique `employeeId` values across all visible project groups (a person on two projects counts once). Per-project and single-project-filter views count subjects in that group's `subjectIds` only.
- **DM counter catalog (six tiles):** `headcount` (`audience`); `need_attention`, `medium`, `high`, `leaver` (`risks` — active levels only, same as UM; exclude `low`); `openResourcingRequests` (`resourcing` — `status === 'open'` only, matching UM; filtered by `projectId` when set). Remove proof `totalActive` counter from DM config. **Layout:** `CounterTileGrid` responsive grid for six tiles (e.g. `sm:grid-cols-2 lg:grid-cols-3`).
- **Resourcing block:** new block id `resourcingRequests` on DM config only. Lists DM-visible requests with `status` in `open` or `pending_dm_review`, sorted `createdAt` desc. Each row shows status, vacancy summary, author, project label or Unassigned copy, and a link to `/resourcing/:id`. Filtered by selected `projectId` including `unassigned`. Compensation band never rendered (entity omits it for non-author viewers). Empty list → standard dashboard empty copy (not unavailable).
- **Tables:** one section per visible project group (hidden when a single project or `unassigned` is selected); reuse `ScopedPeopleTable`. All scoped people appear in tables, including `low` or no risk; active-level counters exclude `low`. Leave/project columns use existing providers with scoped `subjectIds`. Integration down → unavailable cell copy; sync lag with last-known data → AD-8 stale indicator (distinct from unavailable).
- **Fail-soft:** missing or throwing provider → unavailable counter/column/block; no fabricated zeros; summary stays 200. Partial failure (e.g. resourcing counter unavailable but risks available) affects only the failing fragment.
- **Own action items + quick nav:** unchanged from 12.1/12.2 patterns; not filtered by project selector. Quick nav remains the existing three-link `DM_QUICK_NAV_LINKS` set (Employees, Risks, Campaigns) from config — no new links in this story.
- **Frontend selector UX:** `projectId` is client state only (not URL-persisted in 12.3). On selector change, cancel or ignore stale in-flight summary requests so a rapid change cannot render an outdated filter.
- **Tests:** backend unit + e2e for project filter, Unassigned, resourcing visibility (DM own + PM on shared project), headcount deduplication, counter recalculation; frontend e2e for selector UX.

**Ask First:**
- Project display name when timetracker supplies a human-readable name distinct from `projectId` (lookup table vs inline on `DashboardProjectGroup`).

**Never:** PM/PP variant or config changes (12.4/12.5); separate DM page; dashboard access bypass; Risk Dashboard route changes; a11y pass (12.6); approve/reject workflow UI (6.3); pagination on DM project tables; URL query persistence for `projectId`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| All projects default | DM with 3 projects, 18 unique people (2 on multiple projects), 2 active risks, 3 open resourcing requests | Three project tables; `headcount=18` (deduped); risk counters cover 2 records; resourcing counter `3`; block shows DM + PM requests | N/A |
| Select one project | `projectId=proj-atlas` | Only Atlas table; counters recalc to Atlas people/risks/requests | Unknown or not-DM-managed `projectId` → 400 |
| Clear selection | After single-project filter | All tables and aggregated counters restore | N/A |
| Unassigned filter | `projectId=unassigned` | No people tables; headcount and risk counters `0`; resourcing block shows only `projectId=null` requests with `open` or `pending_dm_review`; open counter counts `open` only | N/A |
| PM request visibility | PM on DM project created request | Request appears in DM resourcing block and all-projects counter when `status=open` | N/A |
| Pending DM review | Request in `pending_dm_review` on DM project | Appears in resourcing block; excluded from `openResourcingRequests` counter | N/A |
| DM-only outsider project | Project DM does not manage | Absent from selector and tables; passing its id → 400 | N/A |
| Zero projects | Valid DM, no assignments | Empty tables; counters `0` when available; selector shows All projects + Unassigned only | N/A |
| Duplicate person | Employee on Project A and B | Two table rows (one per project); all-projects `headcount=1` | N/A |
| Provider down | Resourcing provider throws | Resourcing counter and block show unavailable copy | No 500 |
| Partial integration | Leave up, project down | Leave populated; project unavailable with AD-8 when stale | N/A |
| Action items unchanged | DM selects a project | Own action-items widget and quick nav unchanged; still full authored list | N/A |
| Empty resourcing | No visible requests for filter | Block shows empty copy; open counter `0` when provider available | N/A |
| Invalid `projectId` | `projectId=""` or whitespace | Rejected before orchestration | 400 |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/dashboards/dashboard-variant-config.ts` — replace `DM_COUNTERS` proof set; add `resourcingRequests` to DM `blocks`
- `services/backend/src/modules/dashboards/dto/get-dashboard-summary-query.dto.ts` — add optional `projectId` (`all` | project id | `unassigned`)
- `services/backend/src/modules/dashboards/dashboards.service.ts` — apply `projectId` filter to `subjectIds`, groups, provider scope, counter merge; headcount dedup for all-projects
- `services/backend/src/modules/contracts/dashboard-summary.types.ts` — extend `DashboardSummaryScope` with `projectId?: string`
- `services/backend/src/modules/resourcing/resourcing-dashboard-summary.provider.ts` — DM path: `listRequests` + `projectId` filter + `open` count; UM path unchanged (`listAssigned`)
- `services/backend/src/modules/resourcing/` — new list fragment provider or extend contract for `resourcingRequests` block rows (register `dashboard-summary` id)
- `services/backend/src/modules/resourcing/resourcing.service.ts` — `listRequests` + `buildListWhere` DM visibility (reuse, do not duplicate)
- `services/backend/src/modules/access/dashboard-audience.service.ts` — project groups for selector; Unassigned is virtual (not in this service)
- `services/backend/src/modules/dashboards/entities/dashboard-config.entity.ts` — add `resourcingRequests` to `DashboardBlockId`; optional `selectorProjects` on config response
- `services/backend/src/modules/dashboards/entities/dashboard-summary.entity.ts` — optional `resourcingRequests` array + `selectorProjects` metadata
- `services/frontend/src/components/DashboardEngine/ProjectSelector/` (new) — All projects + project list + Unassigned
- `services/frontend/src/components/DashboardEngine/ResourcingRequestsWidget/` (new) — block renderer; link to `/resourcing/:id`
- `services/frontend/src/components/DashboardEngine/DashboardEngineShell/` — mount selector (DM only), resourcing block, respect filtered groups
- `services/frontend/src/pages/DashboardPage/hooks/useDashboardPage.ts` — `projectId` state; pass to summary query; refetch on change; abort stale fetches
- `services/frontend/src/api/services/dashboard.service.ts` + `types/dashboard.ts` — `projectId` query param and response fields
- `services/backend/test/dashboards.e2e-spec.ts` — extend DM scenarios (filter, Unassigned, resourcing, dedup)
- `services/frontend/e2e/dashboard-engine.spec.ts` — DM selector + resourcing block smoke

## Tasks & Acceptance

**Execution:**
- [x] `dashboard-summary.types.ts` + summary query DTO + summary/config entities — `projectId` scope, `resourcingRequests` block payload, selector metadata
- [x] `resourcing-dashboard-summary.provider.ts` (+ new list provider if split) — variant-aware counting; `listRequests` for DM; scope filter by `projectId`
- [x] `dashboard-variant-config.ts` + `dashboards.service.ts` + controller/swagger — DM six-counter catalog, project filter orchestration, Unassigned rules, headcount dedup, resourcing block merge
- [x] `dashboards/__tests__/` + `test/dashboards.e2e-spec.ts` — filter matrix, Unassigned, DM+PM resourcing visibility, dedup, 400 on bad `projectId`
- [x] `ProjectSelector/` + `ResourcingRequestsWidget/` + `DashboardEngineShell/` + `DashboardPage/` + i18n — DM-only UI; selector refetches summary; resourcing block; stale-fetch guard
- [x] `e2e/dashboard-engine.spec.ts` — DM all-projects, single-project filter, resourcing rows visible

**Acceptance Criteria:**
- Given a DM is responsible for three projects with 18 combined unique people and 2 active risk records, when they open the dashboard with no project selected, then three project tables render and counters show headcount 18 and risk counts covering both records
- Given the dashboard shows all projects, when the DM selects a specific project in the selector, then only that project's table remains and every counter recalculates for that project alone; clearing the selection restores the all-projects view
- Given the DM's resourcing block renders, when requests exist from the DM and from PMs on their projects, then both appear in the all-projects view and the open resourcing counter includes only `open` requests
- Given project-less open resourcing requests exist, when the DM selects Unassigned, then only those requests appear in the block and people/risk counters read zero
- Given a person appears on two DM projects, when the all-projects view is shown, then headcount counts that person once while each project table still lists them
- Given the DM selects a project, when the own action-items widget and quick nav render, then they are unchanged from the all-projects view (not filtered by project)
- Given a `dashboard-summary` provider is unavailable, when summary is fetched, then affected counters or the resourcing block show unavailable treatment without fabricated values or a 500 response

## Spec Change Log

| Date | Iteration | Change |
|------|-----------|--------|
| 2026-09-09 | 1 | bmad-review: headcount dedup, resourcing status split (counter vs block), `projectId` validation, AD-8/stale columns, quick-nav scope, selector UX guards, expanded I/O matrix, acceptance criteria |

## Design Notes

PM dashboard reuses this engine in 12.4 — keep `projectId` filtering logic variant-agnostic in the orchestrator but only enable selector UI for `variant === 'dm'` in this story. Variant resolution for multi-role viewers stays UM → DM → PM → PP (12.1); this story does not add client-side variant switching.

## Verification

**Commands:**
- `cd services/backend && npm test -- dashboards` — unit tests pass
- `cd services/backend && npm run test:e2e -- dashboards` — DM filter + resourcing scenarios pass
- `cd services/frontend && npm run lint && npm run typecheck` — pass
- `cd services/frontend && npm run test -- dashboard-engine` — DM selector e2e pass

**Manual checks:**
- DM seed on `/` — project selector defaults to All projects; selecting a project filters tables and counters; resourcing block lists DM + PM requests (`open` and `pending_dm_review`); Unassigned shows project-less requests only; duplicate-person headcount dedupes; UM dashboard unchanged.

### Review Findings

- [x] [Review][Patch] Project selector refetch showed full-page loading instead of inline summary loading [`useDashboardPage.ts`]
- [x] [Review][Patch] Project selector hidden during summary refetch — moved to `DashboardPage` [`DashboardPage.tsx`]
- [x] [Review][Patch] Counter grid used UM nine-tile layout for six DM tiles [`CounterTileGrid.tsx`]
- [x] [Review][Patch] Duplicate `unassigned` sentinel constant in `ProjectSelector` [`types/dashboard.ts`]
- [x] [Review][Patch] Missing backend e2e for headcount dedup and unassigned filter [`dashboards.e2e-spec.ts`]
- [x] [Review][Defer] Missing frontend e2e for unassigned/clear-selection paths [`dashboard-engine.spec.ts`] — deferred, selector refetch e2e covers primary flow

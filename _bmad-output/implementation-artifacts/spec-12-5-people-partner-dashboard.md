---
title: 'People Partner Dashboard'
type: 'feature'
created: '2026-09-10'
status: 'in-progress'
review_loop_iteration: 0
story_key: '12-5-people-partner-dashboard'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-12-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-1-build-shared-dashboard-engine.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-2-unit-manager-dashboard.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-3-assign-people-partner-relationships.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-8-2-idp-records.md'
  - '{project-root}/docs/project-requirements.md#444-people-partner-dashboard'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The `pp` dashboard variant is a stub — empty counter catalog, no `DashboardAudience` method to resolve its population, and `dashboards.service.ts#resolveSubjectIds` returns `[]` for `pp` (variant resolution itself already works via the `PEOPLE_PARTNER` functional role, unchanged since 12.1). **FR-51**/**§4.4.4** require a working PP home dashboard scoped to everyone the PP holds real People Partner access over — direct assignment *and* the HR-line walk `AccessResolverService.resolvePp`/`isInHrLine` grants on profiles (broader than the Risk Dashboard's `listS6SubjectIds`, which only matches direct `peoplePartnerId` today). It must be groupable by department or project, with **no resourcing block anywhere**, plus carry the design-freedom IDPs-approaching-deadline widget as first cut.

**Approach:** Add `listPpAssignedIds` to `DashboardAudience`, implemented as a downward BFS from the viewer through consecutively-HR-department direct-report chains (the exact inverse of `isInHrLine`'s upward walk), matched against `Employee.peoplePartnerId`. Wire it into the `pp` branch of `resolveSubjectIds`; give `pp` the UM counter catalog minus `openResourcingRequests`; add a `department` table-cell provider (directory) and an `idp` block provider (cds) via the existing `dashboard-summary` registry; reuse `ScopedPeopleTable` flat (like UM) and add a client-side department/project group-by toggle over the same fetched rows — no new grouping mode, no selector query param.

## Boundaries & Constraints

**Always:**
- **Audience correctness:** `listPpAssignedIds(viewerId)` must return exactly the population `AccessResolverService.resolvePp` would grant PP section-access to — that is, every subject whose `peoplePartnerId` is reachable from `viewerId` by the downward HR-line walk described above (viewer's own department need not be HR for direct `peoplePartnerId === viewerId` matches; it must be HR to extend the walk to indirect matches; a reached assignee's own department is never itself gated — only the ancestor chain up to `viewerId` must be HR). Never fall back to `listS6SubjectIds`'s direct-only shortcut. The walk uses `getOpenDepartmentValue`'s open-row-only semantics (not `currentHistoryValue`'s latest-row fallback used by the department display provider below); a subject with no open department-history row is treated as non-HR for walk purposes even if the display column resolves a historical label for them.
- **Counters:** reuse `UM_COUNTERS` for `pp.counters` minus `openResourcingRequests` (8 tiles: `headcount`, `need_attention`, `medium`, `high`, `leaver`, `openActionItems`, `overdueActionItems`, `openCampaigns`) — same semantics as UM, scoped to the PP population.
- **Blocks:** `pp.blocks = ['counters', 'table', 'idpDeadlines', 'ownActionItems', 'quickNav']` — never `resourcingRequests`, matching FR-51's "no resourcing block appears anywhere" on the PP dashboard itself. A PP who also holds a higher-priority functional role (UM/DM/PM) is routed to that variant's dashboard instead (per existing `VARIANT_PRIORITY`), which may include its own resourcing block — this story's guarantee covers only the `pp` variant's own config.
- **IDP widget:** new `dashboard-summary` provider `idp` (cds module) returns open (`completedAt IS NULL`) `IDPRecord` rows with `deadline` between "today" and "+30 days" inclusive (both boundaries normalized to UTC midnight, matching `isAssignmentCurrentlyActive`'s `Date.UTC` convention), sorted `deadline` ascending, scoped to `scope.subjectIds`. One row per qualifying `IDPRecord` — an employee with multiple open IDPs in-window appears once per record, not deduplicated. Each row joins `Employee`/`User` for a display name (mirroring `ResourcingRequestsDashboardSummaryProvider`'s `authorDisplayName` join), batched across `scope.subjectIds` rather than per-row. New block id `idpDeadlines`; unavailable/empty follow existing fail-soft and empty-copy conventions.
- **Department display/grouping:** new `dashboard-summary` provider `department` (directory module) returns one cell per subject via `currentHistoryValue(departmentHistory)` (reuse `employee-query.helpers.ts`'s exported helper — same shape/pattern as `EmploymentDashboardSummaryProvider`). Add to `DashboardTableRowEntity` as `departmentStatus`/`departmentLabel`/`departmentStale`, loaded in `ensureTableFragments` only when `scope.variant === 'pp'`. A subject resolves to the "Unassigned" grouping bucket both when `departmentStatus` is unavailable and when it resolves with an empty/blank `departmentLabel`; if the `department` fragment itself is down for the request, the Department toggle option still renders but sections everyone under "Unassigned" (the Project toggle remains unaffected).
- **Grouping UI:** PP keeps `grouping: 'people'` (flat `GET /summary` response, `rows`, no `projectId` param). Frontend renders a "Group by: Department | Project" toggle (client state, default Department) visible only for `config.variant === 'pp'`; it re-sections the already-fetched flat `rows` by `departmentLabel` or `projectLabel` (both compared case-insensitively/trimmed, so differently-cased values for the same department or project merge into one section) — no new API call, no new query param, no filtering of population. `projectLabel` is a comma-joined string of all of a subject's concurrent active projects (per `EmploymentDashboardSummaryProvider`); a subject on 2+ concurrent projects groups once under that combined label rather than appearing under each project separately — a known limitation of this story's client-side grouping.
- **Quick nav:** new `PP_QUICK_NAV_LINKS` (Employees, Risks, Mentorship, Campaigns) — no Resourcing entry. Note this differs from `DM_QUICK_NAV_LINKS`, which excludes both Resourcing and Mentorship; PP excludes only Resourcing and keeps Mentorship.
- **Tests:** backend unit for `listPpAssignedIds` (direct match, multi-level HR-line match, blocked by a non-HR intermediate manager, viewer with non-HR own department only matching direct assignees); unit + e2e for `idp`/`department` providers and `pp` config/summary wiring; frontend e2e for the group-by toggle and IDP widget.

**Ask First:** none — both open design questions were resolved with the human before drafting: audience scope → full HR-line walk; grouping mechanism → client-side toggle.

**Never:** a resourcing block or resourcing counter for `pp`, in any form; a `projectId` selector or query param for `pp` (grouping stays flat/client-side); changes to `um`/`dm`/`pm` configs, the project-grouped orchestrator path, or `DashboardVariantResolverService` (already wired for `pp` since 12.1); reuse of `listS6SubjectIds` for scoping; overdue-IDP handling beyond the 30-day-approaching window (design-freedom first cut only); a11y/responsive pass (12.6); editing department/IDP data from the dashboard (read-only display).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Direct PP assignment | 40 employees have `peoplePartnerId = viewer` across 2 departments | `headcount = 40`; table lists all 40, groupable by department or project; no resourcing block/counter anywhere | N/A |
| HR-line indirect assignment | Employee E has `peoplePartnerId = X`; X's manager chain to viewer is 2 levels, all HR department | E appears in viewer's population | N/A |
| HR-line broken by non-HR manager | Same as above but one intermediate manager is not HR department | E absent from viewer's population | N/A |
| Viewer not in HR department | Viewer has direct PP assignees but their own department ≠ HR | Direct assignees still appear; no indirect (HR-line) assignees added | N/A |
| IDPs approaching deadline | 3 of the PP's population have open IDPs with deadlines in 5, 20, and 45 days | Widget lists the 5- and 20-day IDPs only, sorted ascending; 45-day one absent | N/A |
| No IDPs due | PP population has no open IDP within 30 days | Widget shows standard empty copy, not unavailable | N/A |
| Group by department | PP selects "Department" | Table sections by `departmentLabel`; unresolved department renders under an "Unassigned" section, not dropped | N/A |
| Group by project | PP selects "Project" | Same rows re-sectioned by `projectLabel`; people with no active project group under "Unassigned" | N/A |
| Multi-role viewer | Person holds PP and also DM functional roles | Resolves to DM dashboard per existing UM→DM→PM→PP priority; PP behavior in this story never executes for them | N/A |
| Provider down | `idp` or `department` provider throws | Affected block/column shows unavailable treatment; other blocks/counters unaffected; no 500 | No 500 |
| Zero PP assignees | Valid PP viewer, empty population | Counters `0` where available; empty table and empty IDP widget with standard empty copy | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/contracts/dashboard-audience.contract.ts` — add `listPpAssignedIds(viewerEmployeeId): Promise<string[]>`
- `services/backend/src/modules/access/dashboard-audience.service.ts` — implement via downward HR-line BFS (mirror `AccessResolverService.isInHrLine`/`getOpenDepartmentValue` at `access-resolver.service.ts:436-497`); do not duplicate the upward walk, invert it
- `services/backend/src/modules/dashboards/dashboards.service.ts:313-330` (`resolveSubjectIds`) — add `pp` branch calling `listPpAssignedIds`
- `services/backend/src/modules/dashboards/dashboard-variant-config.ts` — `pp.counters` = `UM_COUNTERS` minus `openResourcingRequests`; `pp.blocks` add `idpDeadlines`; new `PP_QUICK_NAV_LINKS`
- `services/backend/src/modules/contracts/dashboard-summary.types.ts` — new fragment variants `{ providerId: 'idp'; status: 'available'; rows: DashboardIdpRowFragment[] }` and `{ providerId: 'department'; status: 'available'; cells: Record<string, DashboardTableCellFragment> }`
- `services/backend/src/modules/cds/idp-dashboard-summary.provider.ts` (new) — `@RegisterProvider('dashboard-summary', 'idp')`, mirrors `resourcing-requests-dashboard-summary.provider.ts`'s row-list structure (not the aggregate-count shape of `action-items-dashboard-summary.provider.ts`); filters `IDPRecord` by `completedAt: null`, `deadline` window; joins `Employee`/`User` for display name, batched across `scope.subjectIds`
- `services/backend/src/modules/directory/department-dashboard-summary.provider.ts` (new) — `@RegisterProvider('dashboard-summary', 'department')`, mirrors `employment-dashboard-summary.provider.ts`; reuse `currentHistoryValue` from `employee-query.helpers.ts`
- `services/backend/src/modules/cds/cds.module.ts` — add `IdpDashboardSummaryProvider` to `providers` (the `@RegisterProvider` decorator alone doesn't register it with Nest's DI)
- `services/backend/src/modules/directory/directory.module.ts` — add `DepartmentDashboardSummaryProvider` to `providers`
- `services/backend/src/modules/dashboards/dashboards.service.ts` (`ensureTableFragments`, `buildTableRows`) — load `department` fragment when `scope.variant === 'pp'`; map into row's `departmentStatus`/`departmentLabel`/`departmentStale`; merge `idpDeadlines` block fragment into summary when `pp.blocks` includes it
- `services/backend/src/modules/dashboards/dashboards.service.ts` (`resolvePeopleTablePagination`) — currently gates pagination to `variant === 'um'` only; `pp` ships this story without table pagination (unpaginated single fetch), matching `dm`/`pm`'s existing unpaginated flat variants — called out here as an accepted limitation since PP populations can span larger HR-line subtrees than typical UM teams
- `services/backend/src/modules/dashboards/entities/dashboard-config.entity.ts` — `DashboardBlockId` add `'idpDeadlines'`
- `services/backend/src/modules/dashboards/entities/dashboard-summary.entity.ts` — `DashboardTableRowEntity` add department fields; new `DashboardIdpRowEntity`; `DashboardSummaryEntity` add `idpDeadlines?: DashboardIdpRowEntity[]`
- `services/frontend/src/components/DashboardEngine/GroupedPeopleTable/` (new) — client-side department/project group-by toggle wrapping `ScopedPeopleTable`; used only for `variant === 'pp'`
- `services/frontend/src/components/DashboardEngine/ScopedPeopleTable/ScopedPeopleTable.tsx` — add a Department column (rendered when `variant === 'pp'`) alongside the existing Name/Risk/Leave/Project columns; `GroupedPeopleTable` wraps this component for grouping, but the column itself must render here
- `services/frontend/src/components/DashboardEngine/IdpDeadlinesWidget/` (new) — mirrors `ResourcingRequestsWidget` structure
- `services/frontend/src/components/DashboardEngine/DashboardEngineShell/DashboardEngineShell.tsx` — render `GroupedPeopleTable` instead of flat `ScopedPeopleTable` when `config.variant === 'pp'`; render `idpDeadlines` block when present
- `services/frontend/src/types/dashboard.ts` + `api/services/dashboard.service.ts` — mirror new entity fields
- `services/backend/test/dashboards.e2e-spec.ts` + `services/backend/src/modules/access/__tests__/dashboard-audience.service.spec.ts` — PP scenarios per I/O matrix
- `services/frontend/e2e/dashboard-engine.spec.ts` — PP group-by toggle + IDP widget scenario (mocked fixtures)

## Tasks & Acceptance

**Execution:**
- [x] `dashboard-audience.contract.ts` + `dashboard-audience.service.ts` -- add `listPpAssignedIds` via downward HR-line BFS -- gives the orchestrator a C1-correct PP population
- [x] `dashboards.service.ts` -- wire `pp` into `resolveSubjectIds`; load `department` fragment for `pp`; merge `idpDeadlines` block -- closes the empty-population and missing-widget gaps
- [x] `dashboard-variant-config.ts` -- PP counter catalog, blocks, `PP_QUICK_NAV_LINKS` -- FR-51 shape
- [x] `idp-dashboard-summary.provider.ts` (cds) + `department-dashboard-summary.provider.ts` (directory) -- new registry providers -- powers the IDP widget and department grouping/display
- [x] `dashboard-summary.types.ts` + config/summary entities -- new fragment and DTO shapes
- [x] `dashboard-audience.service.spec.ts` + `dashboards.e2e-spec.ts` -- PP audience matrix (direct, indirect, blocked-chain, non-HR viewer), IDP window, provider-down cases
- [x] `GroupedPeopleTable/` + `IdpDeadlinesWidget/` + `DashboardEngineShell.tsx` + `types/dashboard.ts` + i18n -- PP-only UI composition
- [x] `dashboard-engine.spec.ts` -- PP scenario: counters, group-by toggle, IDP widget, no resourcing block/link

### Review Findings

- [x] [Review][Patch] Restore removed `listManagerSubordinateIds` and `listProjectGroups` unit tests [`dashboard-audience.service.spec.ts`]
- [x] [Review][Patch] Fix `GroupedPeopleTable` splitting unavailable rows into multiple Unassigned buckets [`GroupedPeopleTable.tsx:25-33`]
- [x] [Review][Patch] Show empty-table copy when PP population is zero [`GroupedPeopleTable.tsx`]
- [x] [Review][Patch] Strengthen IDP provider tests for window filter and ascending deadline order [`idp-dashboard-summary.provider.spec.ts`]
- [x] [Review][Patch] Cover closed-row `currentHistoryValue` fallback in department provider tests [`department-dashboard-summary.provider.spec.ts`]
- [x] [Review][Patch] Add PP provider-down wiring tests in `dashboards.service.spec.ts` [`dashboards.service.spec.ts`]
- [x] [Review][Patch] Add backend e2e for zero PP assignees and non-HR PP viewer [`dashboards.e2e-spec.ts`]
- [x] [Review][Patch] Add cycle-guard test for `listPpAssignedIds` HR walk [`dashboard-audience.service.spec.ts`]
- [x] [Review][Patch] Add frontend e2e for case-insensitive department merge and single Unassigned bucket [`dashboard-engine.spec.ts`]
- [x] [Review][Defer] Extract shared HR-line walk helper with `AccessResolverService` — deferred, pre-existing architectural drift accepted for this story per design notes

**Acceptance Criteria:**
- Satisfies I/O & Edge-Case Matrix rows "Direct PP assignment" and "Multi-role viewer" — population scoping (headcount, table, group-by), no resourcing block anywhere on the `pp` dashboard itself, and correct routing for a PP who separately holds a higher-priority functional role (UM/DM/PM)
- Satisfies I/O & Edge-Case Matrix row "IDPs approaching deadline" — the IDPs-approaching-deadline widget lists only the PP's assigned population, within the 30-day window
- Satisfies I/O & Edge-Case Matrix rows "HR-line indirect assignment" and "HR-line broken by non-HR manager" — audience correctness for the downward HR-line walk

## Design Notes

`listPpAssignedIds`'s BFS: start at `viewerId` (always eligible as a direct-match anchor); if `viewerId`'s own department is HR, recurse into its direct reports (`managerId = viewerId`), continuing recursion only through nodes whose own department is HR. Collect every visited node ID into `anchors`; the result is every employee whose `peoplePartnerId ∈ anchors`. This is the precise inverse of `isInHrLine` and must stay in lock-step with it if that method ever changes.

Guard the walk the same way `isInHrLine` guards its upward walk: track a `visited` set of node IDs and stop recursing (with the same warning-log pattern) on a repeat, including the degenerate case of a node whose own `managerId` equals its own ID. A viewer with no `Employee` row, or no open `departmentHistory` row, is treated as non-HR — the walk does not extend past the viewer (matches `getOpenDepartmentValue`'s null fallback). Reuse the same HR-department-value constant `AccessResolverService` derives from `HR_DEPARTMENT_VALUE` rather than re-deriving it independently, so the two implementations cannot drift apart. Like the existing reporting-line walks in this service, this issues one query per visited node (no batch-fetch, no cache) — acceptable for this story given the existing `um`/`dm` walks follow the same pattern, but worth revisiting if PP populations prove large in practice.

## Verification

**Commands:**
- `cd services/backend && npm test -- dashboards access cds directory` -- unit tests pass, including new `pp`/`idp`/`department` cases
- `cd services/backend && npm run test:e2e -- dashboards` -- PP scenarios pass alongside UM/DM/PM
- `cd services/frontend && npm run lint && npm run typecheck` -- pass
- `cd services/frontend && npm run test -- dashboard-engine` -- PP e2e scenario passes

**Manual checks:**
- Assign the "People Partner" functional role to a seed employee with direct and HR-line-indirect assignees (same `assignBuiltInRole` pattern as prior stories); then confirm on `/`:
  - only that population appears (counters and table)
  - the group-by toggle switches sections correctly between Department and Project
  - the IDPs-approaching-deadline widget lists only within-30-day open IDPs
  - no resourcing block/link is visible anywhere on the page


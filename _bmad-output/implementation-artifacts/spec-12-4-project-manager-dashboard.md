---
title: 'Project Manager Dashboard'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 1
baseline_commit: '2de8ef8e29d5f2d3b83b953ea33b7e85058c73a2'
story_key: '12-4-project-manager-dashboard'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-12-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-1-build-shared-dashboard-engine.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-12-3-delivery-manager-dashboard-with-project-selector.md'
  - '{project-root}/docs/project-requirements.md#443-project-manager-dashboard'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The `pm` dashboard variant is scaffolded (`dashboard-variant-config.ts`, orchestrator branches, `listProjectGroups(viewerId, 'pm')`) but incomplete: `pm` has an empty counter catalog, no `resourcingRequests` block, and both resourcing dashboard-summary providers gate their real logic to `scope.variant === 'dm'` — a PM viewer silently falls back to `listAssigned` (UM department-routing lookup), the wrong population entirely. **FR-50** / requirements **§4.4.3** require the PM home dashboard identical in shape to the DM dashboard (Story 12.3), scoped to the PM's own projects, with a resourcing block limited to requests the PM personally authored (Story 6.1 PM scoping) — no DM-level breadth, no project selector.

**Approach:** Reuse the DM six-tile counter catalog and `resourcingRequests` block for `pm`; widen both resourcing dashboard-summary providers to treat `pm` like `dm` for population source (`ResourcingService.listRequests`, which already resolves to author-own-only for a PM viewer per Story 6.1). No orchestrator, audience, or entity changes — `dashboards.service.ts` and `dashboard-audience.service.ts` are already variant-agnostic for `pm`. No project selector for PM. A viewer holding both Delivery Manager and Project Manager functional roles resolves to the DM dashboard under the existing UM→DM→PM→PP priority (12.1) — unchanged by this story and out of scope for PM verification.

## Boundaries & Constraints

**Always:**
- **Scope:** PM audience via `listProjectGroups(viewerId, 'pm')` (unchanged) — projects where `ProjectAssignment.pmId === viewerId` only; mere `employeeId` membership never appears. Resourcing visibility follows `ResourcingService.listRequests` (the DM-routing OR-clause in `buildListWhere` only matches when the viewer is also a `dmId`, so a PM-only viewer naturally sees own-authored only) — never `listAssigned` (UM routing) for `pm`.
- **Counter catalog:** reuse the DM six-tile catalog verbatim for `pm` in `DASHBOARD_VARIANT_DEFINITIONS` — `headcount` (deduped across all PM projects), `need_attention`/`medium`/`high`/`leaver` (active risk levels, exclude `low`), `openResourcingRequests` (`status === 'open'`, PM-authored population).
- **Blocks:** add `resourcingRequests` to `pm.blocks` (currently missing); keep `counters`/`table`/`ownActionItems`/`quickNav` as-is — no selector block.
- **Resourcing providers:** in `resourcing-dashboard-summary.provider.ts`, extend `scope?.variant === 'dm'` to `scope?.variant === 'dm' || scope?.variant === 'pm'`. In `resourcing-requests-dashboard-summary.provider.ts`, extend the `scope?.variant !== 'dm'` unavailable-gate to also allow `pm`. Both already accept optional `scope.projectId`, which stays `undefined` for PM (no selector).
- **Direct `projectId` filtering (no UI, but API-reachable):** `dashboards.service.ts`'s project-grouped orchestration already validates and filters `projectId` (including `unassigned`) for any `grouping: 'project'` variant, `pm` included — a PM viewer can pass their own `projectId` or `unassigned` directly against `GET /api/v1/dashboards/summary` even with no selector rendered. This is existing, variant-agnostic orchestrator behavior (no new code) and stays allowed; an outsider project id still 400s via `validateProjectFilter`.
- **Resourcing block rows:** unchanged shape/statuses (`open` or `pending_dm_review`, `createdAt` desc, no comp band for non-author) once the variant gate widens. For `pm`, every visible row is self-authored (author-own-only population), so the comp-band field is always present per `canViewExpectedCompBand`'s author branch — the non-author carve-out is unreachable for this variant and not separately testable here.
- **Frontend table heading:** `DashboardEngineShell.tsx`'s `tableTitle` keys off `config.variant === 'dm'`; change to `config.grouping === 'project'` so `pm` also renders `dashboard.table.projectsTitle` ("Your projects"). This is a grouping-based generalization, not a DM/UM/PP config or provider-gating change (per Never below) — `um`/`pp` stay `grouping: 'people'` today so their heading is unaffected; renegotiate if a future variant adopts `grouping: 'project'` without wanting this heading.
- **No selector UI:** `ProjectSelector` stays gated to `config.variant === 'dm'`; `getConfig` keeps building `selectorProjects` only for `dm`. Do not extend either to `pm`.
- **Tests:** backend unit `pm` cases for both resourcing providers (population = `listRequests`, not `listAssigned`; block available for `pm`); backend e2e PM scenarios in `dashboards.e2e-spec.ts` mirroring the DM block; frontend e2e PM scenario in `dashboard-engine.spec.ts` (mocked fixtures, no seed/timetracker dependency).

**Ask First:** none — reuse decisions and no-selector scope are settled by FR-50's AC and the existing DM-only gating already in the codebase.

**Never:** a project selector for PM (only DM got that story); DM/UM/PP config or provider-gating changes beyond the shared `dm`/`pm` widening; changes to orchestration, audience service, or entities (already variant-agnostic); changes to `ResourcingService.listRequests`/`buildListWhere` (Story 6.1, already correct); a11y pass (12.6); a PM entry in `SEED_VARIANT_BY_MANIFEST_ID` (e2e tests assign the role directly via `assignBuiltInRole`).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PM with two projects, team member on a third | PM on projects A and B (`pmId`), plain `employeeId` on project C | Only A and B render as project tables; C is absent entirely | N/A |
| PM-authored resourcing requests | PM created 2 requests (1 `open`, 1 `pending_dm_review`); a DM on one of PM's projects created a 3rd request | Resourcing block shows only the PM's 2 requests; the DM's request is absent; `openResourcingRequests` counter is `1` | N/A |
| Duplicate person across PM's projects | Employee on both PM projects A and B | `headcount` counts them once; each project table still lists them | N/A |
| Six-tile counters | PM with active risk records at each level and PM-authored open requests | All six counters (`headcount`, `need_attention`, `medium`, `high`, `leaver`, `openResourcingRequests`) populate matching DM semantics | N/A |
| Zero PM projects | Valid PM viewer, no `ProjectAssignment` rows with `pmId` set | Empty tables; counters `0` where providers available; resourcing block empty | N/A |
| Resourcing provider(s) down | `resourcing` and/or `resourcing-requests` throws for a PM viewer — the two are separate registrations and can fail independently | Only the failing provider's counter or block shows unavailable treatment; the other stays available if it succeeds; no 500, no fabricated zero | No 500 |
| No project selector | PM opens dashboard | No `ProjectSelector` rendered; table heading reads "Your projects" | N/A |
| Direct `projectId` query (no selector rendered) | PM viewer requests `GET /dashboards/summary?projectId=<own-project>` or `?projectId=unassigned` | Filtered result for that project or Unassigned (empty tables, PM's own unassigned-project requests only) — same orchestrator path as DM | Unknown or not-PM-managed `projectId` → 400 |
| Multi-role viewer (PM + DM) | Person holds both Delivery Manager and Project Manager functional roles | Resolves to the DM dashboard (existing UM→DM→PM→PP priority); PM-specific behavior in this story never executes for them | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/dashboards/dashboard-variant-config.ts` — `pm.counters: []` → reuse `DM_COUNTERS`; add `'resourcingRequests'` to `pm.blocks`
- `services/backend/src/modules/resourcing/resourcing-dashboard-summary.provider.ts:18` — widen `scope?.variant === 'dm'` to include `'pm'`
- `services/backend/src/modules/resourcing/resourcing-requests-dashboard-summary.provider.ts:18` — widen the `scope?.variant !== 'dm'` unavailable-gate to allow `'pm'`
- `services/backend/src/modules/resourcing/__tests__/resourcing-dashboard-summary.provider.spec.ts` — add `pm` case: asserts `listRequests` (not `listAssigned`) is called
- `services/backend/src/modules/resourcing/__tests__/resourcing-requests-dashboard-summary.provider.spec.ts` — add `pm` case: availability + PM-authored population; also add a negative case for a third variant (e.g. `pp`) to re-prove the widened gate is `dm`/`pm`-only, not any-variant, now that the existing `um`-only negative case under-specifies it
- `services/backend/src/modules/dashboards/__tests__/dashboards.service.spec.ts` — add a `pm` `getConfig` unit case asserting `counters` matches the DM six-id list (same ids, same order) and `blocks` includes `resourcingRequests` — fast, Postgres-free check against a wiring regression
- `services/backend/test/dashboards.e2e-spec.ts` — add PM scenarios mirroring the DM block; `assignBuiltInRole(testApp, employeeId, BUILT_IN_ROLE_NAMES.PROJECT_MANAGER)`, `pmId` on `ProjectAssignment` rows; assert `config.counters` ids equal the same ordered six-id list used in the DM config test
- `services/frontend/src/components/DashboardEngine/DashboardEngineShell/DashboardEngineShell.tsx:34` — `tableTitle` condition `config.variant === 'dm'` → `config.grouping === 'project'`
- `services/frontend/e2e/dashboard-engine.spec.ts` — add a PM scenario (mocked fixtures): counters, "Your projects" heading, resourcing block, no selector — assert `queryByTestId('dashboard-project-selector')` is absent at the page level, matching where `ProjectSelector` actually mounts in `DashboardPage.tsx` (not inside `DashboardEngineShell`)
- Verified already variant-agnostic for `pm` (no changes needed): `dashboards.service.ts`, `dashboard-audience.service.ts`, dashboard entities, `resourcing.service.ts`, `DashboardPage/*`, `CounterTileGrid.tsx`

## Tasks & Acceptance

**Execution:**
- [x] `dashboard-variant-config.ts` -- reuse `DM_COUNTERS` for `pm.counters`; add `resourcingRequests` to `pm.blocks` -- closes the empty-catalog and missing-block gap so PM gets the DM-identical shape FR-50 requires
- [x] `resourcing-dashboard-summary.provider.ts` + `resourcing-requests-dashboard-summary.provider.ts` -- widen `dm`-only branches/gates to include `pm` -- routes PM through `listRequests` (author-scoped) instead of the UM `listAssigned` routing lookup
- [x] `resourcing-dashboard-summary.provider.spec.ts` + `resourcing-requests-dashboard-summary.provider.spec.ts` -- add `pm` scope unit cases, a third-variant negative case, and a `pm` provider-throws case in each -- locks in the population-source fix, proves the widened gate stays `dm`/`pm`-only, and covers the matrix's provider-down row for `pm`
- [x] `dashboards.service.spec.ts` -- add `pm` `getConfig` unit case (six counters, same order as DM; `resourcingRequests` in `blocks`) -- fast, Postgres-free wiring check
- [x] `dashboards.e2e-spec.ts` -- add PM scenarios per the I/O matrix, plus a DM+PM multi-role precedence case mirroring the existing UM+DM one -- end-to-end proof of PM scoping, resourcing visibility, dedup, six counters, and resolver precedence
- [x] `DashboardEngineShell.tsx` -- fix `tableTitle` to key off `grouping` not `variant === 'dm'` -- PM tables get correct "Your projects" copy
- [x] `dashboard-engine.spec.ts` -- add a PM frontend e2e scenario -- confirms UI composition and absence of the selector

**Acceptance Criteria:**
- Given a person is PM on two projects and an ordinary team member on a third, when they open their PM dashboard, then only the two PM projects appear as project tables, reusing the DM engine's rendering path
- Given the PM's dashboard renders its resourcing block, when requests are shown, then only requests the PM personally created appear — not DM-level visibility, not UM-routed requests
- Given the PM dashboard shows six counters, when risk and resourcing data are available, then all six populate using the same semantics as the DM dashboard (active risk levels only, `open`-status resourcing count)
- Given a `dashboard-summary` provider is unavailable for a PM viewer, when the summary is fetched, then the affected counter or block shows unavailable treatment without a fabricated zero or a 500 response

## Spec Change Log

| Date | Iteration | Change |
|------|-----------|--------|
| 2026-09-09 | 1 | bmad-review: multi-role (PM+DM) precedence note, self-authored comp-band clarification, direct `projectId`/`unassigned` API reachability documented, split provider-down row + two new I/O matrix rows, sharper test assertions (exact counter order, selector-absence target, gate negative case), added config-level `pm` unit test task, manual-check seed guidance |

## Design Notes

Nearly all shared-engine plumbing for `pm` already exists from Stories 12.1/12.3 (orchestrator's `config.variant === 'pm' ? 'pm' : 'dm'` responsibility branch, `resolveSubjectIds`, `listProjectGroups(viewerId, 'pm')`, headcount dedup, `DashboardBlockId`/`DashboardVariant` types). This story is a narrow "finish the wiring" pass, not new engine work — resist the temptation to touch the orchestrator or audience service.

## Verification

**Commands:**
- `cd services/backend && npm test -- dashboards resourcing` -- unit tests pass, including new `pm` provider cases
- `cd services/backend && npm run test:e2e -- dashboards` -- PM scenarios pass alongside existing UM/DM ones
- `cd services/frontend && npm run lint && npm run typecheck` -- pass
- `cd services/frontend && npm run test -- dashboard-engine` -- PM e2e scenario passes

**Manual checks:**
- No seed account currently resolves to `pm` via `SEED_VARIANT_BY_MANIFEST_ID` (only manifest ids 2→um, 3→dm) — assign the "Project Manager" functional role directly to a seed employee (same `assignBuiltInRole` pattern the e2e suite uses) with `ProjectAssignment.pmId` rows on 1-2 projects, then confirm `/` shows those project tables only, six counters, a resourcing block limited to that PM's own requests, "Your projects" heading, and no project selector; UM/DM dashboards remain unchanged.

## Suggested Review Order

**PM variant catalog & block wiring**

- Entry point — reuses the DM six-tile catalog and adds the missing `resourcingRequests` block for `pm`, closing the empty-catalog gap FR-50 requires.
  [`dashboard-variant-config.ts:137`](../../services/backend/src/modules/dashboards/dashboard-variant-config.ts#L137)

**Resourcing provider scoping (the actual bug fix)**

- Routes PM through the author-scoped `listRequests`, not the UM department-routing `listAssigned` lookup, for the `openResourcingRequests` counter.
  [`resourcing-dashboard-summary.provider.ts:18`](../../services/backend/src/modules/resourcing/resourcing-dashboard-summary.provider.ts#L18)

- Same fix for the `resourcingRequests` block's own availability gate.
  [`resourcing-requests-dashboard-summary.provider.ts:18`](../../services/backend/src/modules/resourcing/resourcing-requests-dashboard-summary.provider.ts#L18)

**Frontend table heading**

- Generalizes the "Your projects" heading from a `dm`-only check to any project-grouped variant, covering `pm` without touching `um`/`pp`.
  [`DashboardEngineShell.tsx:34`](../../services/frontend/src/components/DashboardEngine/DashboardEngineShell/DashboardEngineShell.tsx#L34)

**Tests — backend unit**

- Config-level regression guard: six ordered counter ids, `resourcingRequests` in blocks, no selector.
  [`dashboards.service.spec.ts:505`](../../services/backend/src/modules/dashboards/__tests__/dashboards.service.spec.ts#L505)

- Proves the counter provider now calls `listRequests` (not `listAssigned`) for `pm`, plus the provider-down case.
  [`resourcing-dashboard-summary.provider.spec.ts:76`](../../services/backend/src/modules/resourcing/__tests__/resourcing-dashboard-summary.provider.spec.ts#L76)

- Proves the block-gate widened to exactly `dm`/`pm` (negative case on a third variant), plus the provider-down case.
  [`resourcing-requests-dashboard-summary.provider.spec.ts:119`](../../services/backend/src/modules/resourcing/__tests__/resourcing-requests-dashboard-summary.provider.spec.ts#L119)

**Tests — backend e2e**

- Config shape end-to-end for a real PM viewer.
  [`dashboards.e2e-spec.ts:471`](../../services/backend/test/dashboards.e2e-spec.ts#L471)

- Core scoping proof: PM-managed projects only, excluding mere team/`dmId` membership.
  [`dashboards.e2e-spec.ts:501`](../../services/backend/test/dashboards.e2e-spec.ts#L501)

- Resourcing visibility proof: PM-authored only, a DM's request on the same project is excluded.
  [`dashboards.e2e-spec.ts:568`](../../services/backend/test/dashboards.e2e-spec.ts#L568)

- Headcount dedup, empty-state, and project-filter/400 edge cases.
  [`dashboards.e2e-spec.ts:661`](../../services/backend/test/dashboards.e2e-spec.ts#L661)

- Multi-role precedence: DM wins over PM, mirroring the existing UM-over-DM case.
  [`dashboards.e2e-spec.ts:794`](../../services/backend/test/dashboards.e2e-spec.ts#L794)

**Tests — frontend e2e**

- UI composition proof: six counters, "Your projects" heading, resourcing block, no selector.
  [`dashboard-engine.spec.ts:523`](../../services/frontend/e2e/dashboard-engine.spec.ts#L523)

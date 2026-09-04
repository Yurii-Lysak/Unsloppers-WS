---
title: 'Build and Freeze Campaign Audience'
type: 'feature'
created: '2026-09-04'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'bd4c4d1840397240048c5d6c5f3134876f524022' # services/backend; services/frontend: e7c43616dda11cbba6ce97dfa24e1a7abee83cfb
story_key: '10-2-build-and-freeze-campaign-audience'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-10-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-10-1-create-a-form-campaign.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-4-auto-generate-action-item-on-form-campaign-activation.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 10.1 created draft `FormCampaign` records with metadata only — no way to define who receives the form. Epic 10 requires audience building via the All Employees filter engine (C2 — FieldRegistry query DSL), live preview, and manual add/remove before activation, scoped to the same employee universe All Employees shows today. `CampaignRecipient` rows and the immutable freeze happen at activation (Story 10.3), but 10.3 needs a persisted draft definition and a server-side resolver that returns the final `employeeId[]`.

**Approach:** Add draft-audience columns on `FormCampaign`, extend `campaigns` with save/preview/resolve endpoints (draft-only, creator-scoped), orchestrate preview through the same `EmployeesService.listEmployees` visibility pipeline (not a bespoke filter), extract a shared Filter/Audience Builder UI from All Employees, and add a campaign detail route where creators build and confirm the audience. Defer saved-view-as-audience until Story 3.4 ships.

## Boundaries & Constraints

**Always:**
- Draft audience shape: `filters: FieldFilter[]` (C2 DSL), `addedEmployeeIds: string[]`, `excludedEmployeeIds: string[]`. Persisted on `FormCampaign`; editable only while `status === 'draft'` (same atomic `updateMany` + **409** guard as metadata PATCH in `campaigns.service.ts:79-85`).
- Resolved set = `(filter matches via C2) ∪ added − excluded`. Deduplicate; excluded wins when an id appears in multiple sets.
- Preview `total` and `resolve` output use the **full resolved set**, not filter-match count alone. Paginated preview **rows** come from that resolved set (same column shape as All Employees list rows — `employeeId` + visible `cells`).
- Preview and resolve run through `EmployeesService.listEmployees(viewerUserId, { filters, page, pageSize })` for the filter leg — reuse `filterVisibleFields`, `queryEmployees`, `maskRowCells`; do not call `FieldRegistry.queryEmployees` directly from campaigns. Merge adds/excludes in `CampaignsService` after collecting filter-match ids.
- `GET :campaignId` (existing Story 10.1 read) returns the persisted audience definition (`filters`, `addedEmployeeIds`, `excludedEmployeeIds`) so the detail page can reload without a separate audience GET.
- `addedEmployeeIds` may include people outside the current filter result (epic AC). Each id must resolve to an `employmentStatus: 'active'` employee present in the same employee universe as today's unfiltered All Employees list (Story 3.6 colleague row filtering is backlog — match current list behavior, do not invent a new access model). Reject unknown, inactive, or duplicate ids with **400** and `{ invalidEmployeeIds: string[] }`. The campaign creator may include their own employee id (Story 4.4 allows sender in audience).
- `excludedEmployeeIds` may only reference ids in the current filter-match set (not manually added-only ids — remove those by editing `addedEmployeeIds`). Reject ids not in filter results with **400** and `{ invalidExcludedEmployeeIds: string[] }`. Reject duplicate ids in either array with **400**.
- If the same id appears in both `addedEmployeeIds` and `excludedEmployeeIds`, normalize on save by dropping it from `addedEmployeeIds` (exclude wins).
- Endpoints (creator-scoped, non-owned → **404**): `PUT :campaignId/audience` (save definition), `GET :campaignId/audience/preview` (paginated resolved rows + resolved `total`), `GET :campaignId/audience/resolve` (full resolved `employeeIds[]` for 10.3 — unpaginated, bootcamp-scale ≤500 ids). All three reject non-draft campaigns with **409**.
- Export `CampaignsService.resolveAudienceEmployeeIds(campaignId, creatorId)` for Story 10.3 — same logic as the resolve endpoint, callable without HTTP during activation.
- Frontend: new `/campaigns/:campaignId` detail page with metadata summary + audience builder; draft list rows navigate there (keep metadata edit dialog or inline edit on detail — either is fine). Live preview count reflects the resolved set whenever filters or add/remove change.
- Extract shared filter UI from `AllEmployeesPage` into `components/AudienceBuilder/` (filter chips + preview table + add/remove controls). All Employees page adopts the shared component for column filters only — column picker, sort, and URL column state stay page-local.
- Tests: unit tests for resolve algebra and validation; backend e2e for the matrix below; Playwright flow under `e2e/flows/campaigns/` for filter + adjust preview.

**Ask First:**
- Story 3.4 (saved views) is still backlog — confirm shipping filter-only audience in 10.2 and adding `savedViewId` import when 3.4 lands, versus blocking 10.2 until 3.4 is done.

**Never:**
- Activation, `active` status transition, `CampaignRecipient` table rows, or C6 `createCampaignActionItems` calls (Story 10.3).
- Re-resolve or mutate draft audience on `active` campaigns — Story 10.3 owns the frozen snapshot.
- Saved view import as audience starting point (Story 3.4 — no stubs exist).
- Separate audience-filter implementation or reimplemented field catalog.
- Row-level colleague whitelist beyond what All Employees already enforces (Story 3.6) — match current list behavior; do not invent a new access model here.
- Completion-tracking table (Story 10.4).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Save audience on draft | Creator PUTs filters + add/remove | 200, definition persisted | N/A |
| Save on non-draft | `status !== 'draft'` | — | 409 |
| Preview/resolve on non-draft | `status !== 'draft'` | — | 409 |
| Filter preview | `department=Engineering` filters | Paginated resolved rows + `total` = filter matches minus exclusions plus adds | N/A |
| Manual adjust | Remove one filter match via exclude, add one non-match via `addedEmployeeIds` | Preview reflects adjusted resolved set | N/A |
| Invalid manual add | `addedEmployeeIds` contains unknown or inactive uuid | — | 400 + `invalidEmployeeIds` |
| Duplicate manual add | Duplicate uuid in `addedEmployeeIds` | — | 400 + `invalidEmployeeIds` |
| Exclude stranger | `excludedEmployeeIds` not in filter matches | — | 400 + `invalidExcludedEmployeeIds` |
| Overlap add + exclude | Same id in both arrays on PUT | Saved with id excluded (dropped from adds) | N/A |
| Invalid filters | Malformed filter JSON or unknown `fieldId` | — | 400 (same messages as `ListEmployeesQueryDto`) |
| Empty audience | No filters, no adds | `total: 0`, `resolve` returns `[]` | N/A |
| Adds only | No filters, non-empty adds | `total` = valid added count; preview lists added employees | N/A |
| Non-creator | Another user hits audience routes | — | 404 |
| Reload detail | Creator GET campaign after save | Response includes persisted audience definition | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma:472-489` -- `FormCampaign`; add JSON/array columns for draft audience (`audienceFilters`, `audienceAddedEmployeeIds`, `audienceExcludedEmployeeIds`).
- `services/backend/src/modules/campaigns/campaigns.service.ts:65-88` -- draft-only `updateMany` guard pattern to mirror for audience writes.
- `services/backend/src/modules/campaigns/campaigns.module.ts:10-14` -- inject `EmployeesService` (`DirectoryModule` is `@Global()` but explicit import/doc here is fine).
- `services/backend/src/modules/directory/employees.service.ts:40-88` -- `listEmployees` visibility pipeline to reuse for preview (inject, do not duplicate).
- `services/backend/src/modules/directory/field-registry.service.ts:467-531` -- underlying C2 query; reached only via `EmployeesService`.
- `services/backend/src/modules/contracts/field-registry.contract.ts:56-69` -- `FieldFilter` / `EmployeeListQueryOptions` types for DTOs.
- `services/backend/src/modules/directory/dto/list-employees-query.dto.ts` -- filter JSON parsing pattern for audience DTOs.
- `services/backend/src/modules/directory/field-catalog.ts:6-58` -- built-in filterable fields (no `country`; use `department`/`grade` in tests).
- `services/backend/src/modules/action-items/action-items.service.ts` (Story 4.4) -- `employmentStatus: 'active'` assignee validation `resolveAudienceEmployeeIds` must satisfy before 10.3 activation.
- `services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts:47-195` -- filter state/URL helpers to extract.
- `services/frontend/src/pages/AllEmployeesPage/components/ColumnFilterPopover/` -- per-field operator UI to lift into shared builder.
- `services/frontend/src/pages/CampaignsPage/CampaignsPage.tsx:56-90` -- list rows; navigate draft rows to detail instead of only metadata dialog.
- `services/frontend/src/types/employees.ts` -- mirror `FieldFilter` types for audience state.
- `services/frontend/src/router/index.tsx` -- add `/campaigns/:campaignId` route.

## Tasks & Acceptance

**Execution:** (backend schema → service → tests before frontend detail page)
- [x] `services/backend/prisma/schema.prisma` -- draft audience columns + migration
- [x] `services/backend/src/modules/campaigns/{dto/,entities/}` -- audience DTOs/entities; extend `CampaignReadEntity` with definition fields + preview response type
- [x] `services/backend/src/modules/campaigns/campaigns.service.ts` -- `saveAudience`, `previewAudience`, `resolveAudienceEmployeeIds`; inject `EmployeesService`
- [x] `services/backend/src/modules/campaigns/campaigns.controller.ts` -- PUT/GET audience routes; extend GET-one read model
- [x] `services/backend/src/modules/campaigns/campaigns.module.ts` -- wire `EmployeesService`
- [x] `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts` + `services/backend/test/campaigns.e2e-spec.ts` -- audience matrix
- [x] `services/frontend/src/components/AudienceBuilder/` -- shared filter/preview/add-remove UI extracted from All Employees
- [x] `services/frontend/src/pages/AllEmployeesPage/` -- adopt shared `AudienceBuilder` for column filters only
- [x] `services/frontend/src/pages/CampaignDetailPage/` -- detail + audience section, hooks, API wiring
- [x] `services/frontend/src/api/services/campaign.service.ts` + hooks -- audience mutations/queries
- [x] `services/frontend/src/router/index.tsx` + `types/campaigns.ts` -- route and types
- [x] `services/frontend/e2e/flows/campaigns/` -- extend for audience build/adjust flow

**Acceptance Criteria:**
- Given I filter for `department = Engineering` and preview, when I exclude one filter match and add someone who did not match, then the preview `total` and rows show exactly that adjusted resolved set and remain scoped to employees I can see on All Employees today
- Given a draft campaign has a resolved preview of 50 people, when Story 10.3 activates it later, then those 50 ids are what `resolveAudienceEmployeeIds` returns — new hires matching the filter afterward are not included (freeze semantics owned by 10.3 persisting this resolved set)
- Given I save an audience and reload the campaign detail page, when the GET campaign response arrives, then filters and add/remove lists match what I saved
- Given I am not the campaign creator, when I call any audience endpoint for that campaign, then I receive 404

### Review Findings

- [x] [Review][Patch] Preview `total` must count the resolved set (filter ∪ add − exclude), not filter matches alone [`Boundaries & Constraints`, `I/O & Edge-Case Matrix`]
- [x] [Review][Patch] Extend GET campaign read model with persisted audience definition — no separate audience GET [`Boundaries & Constraints`, `Tasks & Acceptance`]
- [x] [Review][Patch] Clarify exclude vs add-remove: exclusions only for filter matches; undo manual adds via `addedEmployeeIds` [`Boundaries & Constraints`, `Design Notes`]
- [x] [Review][Patch] Require `employmentStatus: 'active'` on manual adds to align with Story 4.4 activation validation [`Boundaries & Constraints`, `Code Map`]
- [x] [Review][Patch] Reject duplicate ids; normalize add+exclude overlap (exclude wins) [`Boundaries & Constraints`, `I/O & Edge-Case Matrix`]
- [x] [Review][Patch] Audience routes return **409** on non-draft campaigns; never re-resolve active campaigns [`Boundaries & Constraints`, `Never`]
- [x] [Review][Patch] Mirror atomic draft-only `updateMany` guard for audience PUT [`Boundaries & Constraints`]
- [x] [Review][Patch] Add matrix rows: invalid filters, adds-only audience, reload via GET, preview/resolve 409 [`I/O & Edge-Case Matrix`]
- [x] [Review][Patch] Scope AudienceBuilder extraction to filters — leave column picker/sort URL state on All Employees [`Boundaries & Constraints`, `Tasks & Acceptance`]
- [x] [Review][Patch] Fix frontend verification command to Playwright path filter [`Verification`]
- [x] [Review][Patch] Allow campaign creator in audience (Story 4.4 sender-in-audience) [`Boundaries & Constraints`]
- [x] [Review][Structure] Add Review Findings + Spec Change Log; note `DirectoryModule` is `@Global()` [`Code Map`]
- [x] [Review][Prose] Define C2 on first use; replace ambiguous "unfiltered employee list" wording [`Intent`, `Boundaries & Constraints`]

- [x] [Review][Decision] Live preview while editing — **Decision B (2026-09-04):** client-side resolved count/table from `employeesList` + local definition via `resolve-audience-preview.ts`; Save persists to backend.

- [x] [Review][Patch] Preview query disabled until first Save in session — removed server preview dependency from detail hook; client preview computes immediately [`useCampaignAudienceSection.ts`]

- [x] [Review][Patch] Filter operator UI gated on preview response — `fieldCatalog` from employees list feeds filter chips [`AudienceBuilder.tsx`, `useCampaignAudienceSection.ts`]

- [x] [Review][Patch] Playwright audience flow does not exercise filter/exclude/add AC — extended e2e with department filter, remove, add, and save assertions [`campaigns.spec.ts`, `helpers.ts`, `fixtures.ts`]

- [x] [Review][Patch] Backend tests omit inactive/duplicate `addedEmployeeIds` matrix rows — added unit + e2e coverage [`campaigns.service.spec.ts`, `campaigns.e2e-spec.ts`]

- [x] [Review][Patch] Dead list edit entry point — removed unused `openEdit` / `dialogCampaign` from campaigns list hook [`useCampaignsPage.ts`, `CampaignsPage.tsx`]

- [x] [Review][Defer] `previewAudience` performs an extra `listEmployees` call solely to sample `fields` for the response [`services/backend/src/modules/campaigns/campaigns.service.ts:177-180`] — deferred, acceptable at bootcamp scale

## Spec Change Log

- **2026-09-04 (review loop 1):** Applied bmad-review findings — resolved-set preview semantics, GET campaign includes audience definition, active-employee validation, exclude/add rules, draft-only 409 on all audience routes, matrix expansion, AudienceBuilder scope boundary, verification command fix.
- **2026-09-04 (code review):** Story 10.2 implementation reviewed on `feature/10-2-build-and-freeze-campaign-audience` (backend baseline `bd4c4d1`, frontend baseline `e7c4361`, uncommitted). Backend audience API/tests largely match spec; frontend preview UX and e2e coverage gaps recorded below.
- **2026-09-04 (code review patches):** Decision B — client-side audience preview from employees list + local definition; enabled filter catalog on load; extended e2e and backend invalid-add tests; removed dead list edit hook.
- **2026-09-04:** Story marked `done` after code review patches applied.

## Design Notes

**"Freeze" split:** This story owns draft definition + deterministic resolution. Story 10.3 writes the immutable snapshot (`CampaignRecipient` per architecture spine) and calls C6. Title "Freeze" refers to epic behavior; do not write recipient rows here.

**Add vs exclude UX:** Use `excludedEmployeeIds` to drop filter matches. Use `addedEmployeeIds` edits to add or remove people who did not match the filter. Do not put manually added-only ids in `excludedEmployeeIds`.

**Preview at scale:** Paginate preview table over the resolved id list (`pageSize` ≤ 100). Compute resolved `total` from the full set. `resolveAudienceEmployeeIds` paginates through all filter matches internally, then applies add/exclude algebra (bootcamp ≤500 employees).

**Client-side live preview (Decision B):** The campaign detail page computes preview count and rows from the unfiltered employees list plus the in-memory audience definition (`resolve-audience-preview.ts`), updating immediately when filters or add/remove change. Save persists to the backend; the server preview/resolve endpoints remain the source of truth for activation (Story 10.3).

**10.3 handoff:** Activation should call `resolveAudienceEmployeeIds` once while the campaign is still `draft`, persist ids, then pass `assigneeIds` to `createCampaignActionItems` in the same transaction. All resolved ids must already satisfy Story 4.4's active-employee validation.

```typescript
// Resolved set (server-side only)
const filterIds = await paginateAllFilterMatches(viewerUserId, filters);
const added = normalizeUnique(activeAddedIds);
const excluded = new Set(excludedEmployeeIds);
const resolved = [...new Set([...filterIds, ...added])].filter(
  (id) => !excluded.has(id),
);
```

## Verification

**Commands:**
- `cd services/backend && npm run db:migrate && npm test && npm run test:e2e -- campaigns` -- expected: all pass
- `cd services/frontend && npm run lint && npm run build && npm run test e2e/flows/campaigns` -- expected: pass

**Manual checks:**
- As a manager, open a draft campaign detail, build a filter, confirm live resolved count, add/remove individuals, save, reload — definition persists via GET campaign.
- As another user, direct-call audience endpoints for that campaign id — 404.

---
title: 'Saved and Shared Views'
type: 'feature'
created: '2026-09-04'
status: 'done'
review_loop_iteration: 0
story_key: '3-4-saved-and-shared-views'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-1-sortable-filterable-employee-list.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 3-1 delivered filter/sort/column state in the URL only — managers rebuild useful list configurations every session. FR-10 requires named saved views as tabs and sharing with other managers.

**Approach:** Persist filter/column/sort configuration server-side (`SavedView` + `SavedViewShare`); expose CRUD/share REST under `directory`; add tab bar above the Directory toolbar that loads stored config into existing URL params and re-executes the access-resolved list query per viewer.

## Boundaries & Constraints

**Always:**
- Saved views store configuration only (`filters`, `columnIds`, `sort`, `order`) — never row IDs or cached results.
- Opening a shared view re-runs `GET /employees` with the viewer's own access — rows/columns may differ from the owner.
- Filter-based views only for v1 (`decisions.md` appendix) — no static membership lists.
- Owner can create, rename, update config, delete, and share; recipients are view-only.
- Ownerless views (`ownerEmployeeId` null on departure) remain usable by share recipients but are not editable until HR Admin adopts (defer adopt flow — read-only for v1).

**Ask First:**
- Whether to extend list API with `visibleFieldIds` server-side (default: keep client column projection from 3-1).

**Never:** export (3.5); colleague mode (3.6); campaign audience wiring (10.2); static bench rosters; materialized membership.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Save view | Manager saves current filters+columns as "Needs a conversation" | View persisted; appears as tab; reload restores config | 400 on empty name |
| Shared view access | Manager A shares management-field view with Manager B | B's tab shows only rows/columns B is entitled to | 404 if view deleted |
| Recipient read-only | Non-owner opens shared view | Can use tab; cannot rename/delete/share | 403 on PATCH/DELETE |
| Unshare | Owner removes B from shares | View tab disappears for B | 404 if share absent |
| Ownerless view | Creator departed (`ownerEmployeeId` null) | Recipients still see tab; edit/delete blocked | 403 on mutate |

</frozen-after-approval>

## Code Map

- [`services/backend/prisma/schema.prisma`](../../services/backend/prisma/schema.prisma) — `SavedView`, `SavedViewShare` models
- [`services/backend/src/modules/directory/saved-views.service.ts`](../../services/backend/src/modules/directory/saved-views.service.ts) — CRUD + share logic
- [`services/backend/src/modules/directory/saved-views.controller.ts`](../../services/backend/src/modules/directory/saved-views.controller.ts) — REST surface
- [`services/backend/src/modules/directory/directory.module.ts`](../../services/backend/src/modules/directory/directory.module.ts) — register controller/service
- [`services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts`](../../services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts) — `view` URL param + `applySavedView` / `getCurrentViewConfig`
- [`services/frontend/src/pages/AllEmployeesPage/components/ViewTabs/ViewTabs.tsx`](../../services/frontend/src/pages/AllEmployeesPage/components/ViewTabs/ViewTabs.tsx) — tab bar UI
- [`services/frontend/src/api/services/saved-view.service.ts`](../../services/frontend/src/api/services/saved-view.service.ts) — API client

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` — add models + migration — persistence layer
- [x] `services/backend/src/modules/directory/saved-views.*` — service, controller, DTOs, swagger — API
- [x] `services/backend/src/modules/directory/__tests__/saved-views.service.spec.ts` — unit tests — owner/share/access rules
- [x] `services/backend/test/saved-views.e2e-spec.ts` — e2e save + share — integration proof
- [x] `services/frontend/src/api/` — types, service, hooks — data layer
- [x] `services/frontend/src/pages/AllEmployeesPage/` — ViewTabs, SaveViewDialog, ShareViewDialog — FR-10 UX
- [x] `services/frontend/src/locales/en/translation.json` — i18n keys — user-facing copy

**Acceptance Criteria:**
- Given a manager configures filters and columns on All Employees, when they save as "Needs a conversation", then it appears as a tab, persists across sessions, and coexists with other saved views
- Given Manager A saves a view including a management-visible custom field and shares it with Manager B, when Manager B opens the shared view, then B sees only rows and columns B is personally entitled to see

## Design Notes

Tab selection sets URL `view=<id>` and hydrates `filters`, `columns`, `sort`, `order` from the saved record. List fetch unchanged — access resolution in `EmployeesService.listEmployees` ensures per-viewer results.

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm test -- --testPathPatterns=saved-views` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- saved-views.e2e-spec` — expected: e2e pass (Postgres up)
- `cd services/frontend && npm run typecheck` — expected: no TS errors
- `cd services/frontend && npm run lint` — expected: no errors

## Spec Change Log

### Review Findings

- [x] [Review][Patch] **Decision: drop the entire filter set + notice.** Filtering on a field the viewer can't see 400s the whole list call, breaking the core sharing AC — `FieldRegistryService.validateFilters` throws `BadRequestException` when a filter's `fieldId` is outside the viewer's `visibleFieldIds` [`services/backend/src/modules/directory/field-registry.service.ts:617-620`], and `applySavedView` writes the owner's stored filters straight into URL params [`services/frontend/src/pages/AllEmployeesPage/hooks/useAllEmployeesPage.ts:220-231`]. Fix: in `EmployeesService.listEmployees`, when any filter references a field that exists in the catalog but is outside the viewer's `visibleFieldIds`, drop the entire `filters` array (not per-filter) and set a `filtersHidden: true` flag on the list response; keep the existing 400 for filters referencing a field absent from the catalog entirely (client/malformed-input signal, unrelated to visibility). Frontend shows a toast/banner when `filtersHidden` is true (e.g. "Some filters in this view aren't visible to you and were skipped").
- [x] [Review][Patch] **Decision: dedicated lookup endpoint.** `ShareViewDialog`'s recipient picker is built only from the currently-loaded, paginated (50/page), possibly column-filtered `employeesList` rows [`services/frontend/src/pages/AllEmployeesPage/components/ShareViewDialog/ShareViewDialog.tsx:44-54`] — recipients already in `view.sharedWith` who aren't in that page never render as checked (silent un-share risk), and sharing with anyone outside page 1 is impossible. Fix: add a lightweight `GET /employees/lookup` endpoint (id + name for every employee, no pagination, no field-registry/visibility masking — name is baseline-visible everywhere per existing rules) and a matching frontend hook; rewire `ShareViewDialog`'s `recipientOptions` to source from it, still merging in any `view.sharedWith` entries defensively.
- [x] [Review][Patch] Whitespace-only view name bypasses the spec's "400 on empty name" rule — `CreateSavedViewDto`/`UpdateSavedViewDto` validate string length via `@MinLength(1)`, not trimmed content, and the service trims after validation [`services/backend/src/modules/directory/saved-views.service.ts:67,91`], persisting an empty name for a `" "` input.
- [x] [Review][Patch] Unsharing down to zero recipients is impossible on both API and UI — `ShareSavedViewDto.recipientEmployeeIds` has `@ArrayMinSize(1)` [`services/backend/src/modules/directory/dto/share-saved-view.dto.ts:7`], and `ShareViewDialogForm.handleShare` no-ops on an empty selection with the confirm button disabled at `selectedIds.length === 0` [`services/frontend/src/pages/AllEmployeesPage/components/ShareViewDialog/ShareViewDialog.tsx:64-66,116`] — the spec's "Owner removes B from shares → view disappears for B" is unreachable when B is the only recipient.
- [x] [Review][Patch] Real, reachable behaviors ship without test coverage: `SavedViewsService.remove()` success path (owner deletes, row actually gone) [`services/backend/src/modules/directory/saved-views.service.ts:106-112`]; `replaceShares()` reducing an existing share set (verify the dropped recipient actually loses access) [`saved-views.service.ts:114-146`]; the self-share `ForbiddenException` guard [`saved-views.service.ts:123-125`]; `update()`'s partial-field-merge success path [`saved-views.service.ts:89-100`].
- [x] [Review][Defer] Migration `20260904122634_add_saved_views` bundles unrelated schema drift (`access_graph_generation.id` default, `timeline_events.updatedAt` default drop, an index rename on `relationship_journal_entries`) alongside the Story 3.4 tables — confirmed absent from this branch's `schema.prisma` diff, so it's pre-existing DB/migration-history drift, not caused by this story — deferred, confirm with team before merge whether to split
- [x] [Review][Defer] No UI path exists to rename a view or re-save its config, even though "Owner can ... rename, update config" is an Always-constraint and the `PATCH` endpoint/hooks already exist end-to-end [`services/frontend/src/pages/AllEmployeesPage/AllEmployeesPage.tsx`] — deferred, not covered by either Acceptance Criterion for this story
- [x] [Review][Defer] Minor/cosmetic items bundled: share/delete icons only render when a view's tab is also active [`ViewTabs.tsx:64`]; a bookmarked `view=<deletedId>` URL degrades silently instead of erroring (mutate endpoints already 404 correctly); `sort`/`order` are unconstrained DB `String?` columns despite DTO-level `@IsIn` enum restriction; no per-owner uniqueness on view names; `columnIds` allows an empty array and `recipientEmployeeIds` has no upper bound; TOCTOU gap between `assertRecipientsExist` and the share transaction; `isSavedViewsError` is computed but never surfaced in the UI; tab bar pops in abruptly with no loading skeleton; inconsistent `null` vs `''` fallback between `ownerName` and a missing recipient's `name`; `directory.savedViews.sharedBy` i18n value is a bare `"{{name}}"` with no "shared by" wording — deferred, cosmetic/low-risk
- [x] [Review][Defer] Ownerless-view read-only enforcement, the controller's direct `PrismaService` injection for viewer-employee resolution, and the lack of an "HR Admin adopts an ownerless view" flow all match the spec's explicit "defer adopt flow — read-only for v1" boundary and the codebase's existing `custom-fields.controller.ts` pattern — dismissed as spec-compliant / consistent with convention, not defects
- [x] [Review][Defer] Spec's I/O matrix error column says "404 if share absent" for Unshare, but `replaceShares` is a full-replace endpoint that always returns 200 — a spec-wording mismatch against the chosen full-replace design, not a code defect; resolved in substance once the zero-recipient patch above lands (the described behavior — "view disappears for B" — becomes reachable) — dismissed, no code action

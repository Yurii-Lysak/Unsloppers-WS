---
title: 'View Shared Feedback, Flagged Notes and Own Action Items'
type: 'feature'
created: '2026-09-10'
status: 'done'
review_loop_iteration: 0
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-2-context.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Backend S7/S8/S14 are already correct for Self (`ManagementNotesService`/`FeedbacksService` filter to `visibleForEmployee`/`sharedWithEmployee`; `ActionItemsService.buildSection` filters to own items, `completeActionItem` is assignee-gated and never exposes edit/cancel). Frontend S7/S8 (`ManagementNotesSectionCard`, `FeedbackSectionCard`) are already wired into `profile-sections.tsx` and already render read-only whenever `accessLevel !== 'RW'` — Self's level. The one real gap: S14 has no frontend presence at all — missing from `PROFILE_SECTION_TITLE_KEYS`/`PROFILE_SECTION_RENDERERS`, no `ActionItemsSection` type, no mutation calling `POST /employees/:id/action-items/:itemId/complete` anywhere in the frontend — not even in the read-only manager-dashboard widget (`OwnActionItemsWidget.tsx`), which lists items but has no complete affordance.

**Approach:** Build the missing S14 self-service slice (type, API service, mutation, data hook, section component with mark-complete only), following the exact layering `ManagementNotesSection` uses for S7. Add Self-viewer test coverage for S7/S8/S14. No backend changes.

## Boundaries & Constraints

**Always:**
- S14's mark-complete button calls only the existing complete endpoint — never a PATCH that could touch title/dueDate.
- S14 shows only the viewer's own items (already server-filtered for Self) — never merge in `listAuthoredOpenItems` data.
- S7/S8 keep using their existing components/gate unchanged — Self's read-only view already falls out of `accessLevel === 'RW'` being false.
- New i18n: section title `employeeProfile.sections.actionItems` (distinct from the unrelated `dashboard.actionItems.*` manager-widget namespace), content under flat `employeeProfile.s14.*` (matching `s7`/`s8`), toasts under `employeeProfile.actionItems.complete.*` (matching sibling `employeeProfile.managementNotes.*`).

**Never:**
- No changes to `ManagementNotesService`, `FeedbacksService`, `ActionItemsService`, or their controllers/providers — confirmed correct for Self.
- No edit/cancel affordance on S14 for Self.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| SELF_VIEW_S7_FILTERED | One note `visibleForEmployee: true`, one `false` | S7 shows only the flagged note, read-only | — |
| SELF_VIEW_S8_FILTERED | One record `sharedWithEmployee: true`, one `false` | S8 shows only the shared record, read-only | — |
| SELF_VIEW_S14_OWN_ONLY | I'm assignee on 2 items, author on 1 other's item | S14 lists only my 2 items | — |
| SELF_COMPLETE_ACTION_ITEM | I mark my own open item complete | `completedAt` recorded/displayed; title/dueDate/link stay non-editable, no cancel control | 409 toast if already completed/cancelled |
| SELF_RISK_INVISIBLE | I have a risk record | S6 renders nowhere on my own profile (regression check, no S6 code touched) | — |

</frozen-after-approval>

## Code Map

- `services/frontend/src/types/employee-profile.ts` -- add `ActionItemAuthor{id,displayName}` and `ActionItemsSection{items: ActionItem[]}`, `ActionItem` mirroring backend `ActionItemReadEntity` (`services/backend/src/modules/action-items/entities/action-item.entity.ts:11-53`): `id,title,description?,dueDate,link?,status:'open'|'completed'|'cancelled',source:'manual'|'campaign',author,createdAt,updatedAt,completedAt?,cancelledAt?,cancelledReason?,isOverdue`
- `services/frontend/src/api/services/action-item.service.ts` -- new `ActionItemApiService.completeActionItem(employeeId, itemId)` → `apiClient.post('/api/v1/employees/${employeeId}/action-items/${itemId}/complete')`; copy `management-note.service.ts`'s shape
- `services/frontend/src/api/hooks/useActionItemMutations.ts` -- new `useCompleteActionItem(employeeId)`, `onSuccess` toasts + invalidates `employeeProfileQueryKey(employeeId)` (`@/api/hooks/useEmployeeProfile`); `onError` reads the 409 body's `status` field to distinguish an already-completed from an already-cancelled item in the toast, and also invalidates the query so a stale "open" item doesn't keep offering a dead button -- copy `useManagementNotesMutations.ts:11-28`
- `services/frontend/src/hooks/data/useActionItemsData.ts` -- new `useActionItemsData(employeeId)`: `completeItem(itemId)` + a per-item `pendingItemId` (not a single shared `isCompletingItem`) so only the clicked item's button disables while its own request is in flight -- copy `useManagementNotesData.ts`
- `services/frontend/src/pages/EmployeeProfilePage/components/ActionItemsSection/hooks/useActionItemsSection.ts` -- new section hook consuming `useActionItemsData`, per `react-pages.md`'s three-layer rule (component → section hook only, never `hooks/data/` directly) -- mirrors `ManagementNotesSection/hooks/useManagementNotesSection.ts`
- `services/frontend/src/pages/EmployeeProfilePage/components/ActionItemsSection/ActionItemsSection.tsx` -- new `ActionItemsSectionCard({employeeId, section, accessLevel})`: list items (title, description when present, dueDate with `<OverdueIndicator dueDate={item.dueDate} />` on open+overdue items, status, link when present); "mark complete" button per open item only when `accessLevel !== 'RW'`; cancelled items show `cancelledOn`/`cancelledReason` instead of a button; no edit/cancel controls for any audience -- structure after `ManagementNotesSectionCard` (`ManagementNotesSection.tsx:27-72`) minus the create form
- `profile-sections.tsx:23-32` (import block), `91-107,128-335` -- import the new card; add `S14: 'employeeProfile.sections.actionItems'` to `PROFILE_SECTION_TITLE_KEYS`; add an `S14` renderer entry shaped like the `S7` case (`S14` is already registered in `PROFILE_SECTION_ORDER` at line 48 -- no change needed there)
- `services/frontend/src/locales/en/translation.json` -- add `sections.actionItems` (sibling of `.managementNotes`/`.feedback`, ~line 655); flat `s14.*` keys (`empty`,`dueDate`,`markComplete`,`completedOn`,`cancelled`,`cancelledOn`,`cancelledReason`) -- reuse the existing `campaigns.completion.overdue` key via `<OverdueIndicator>` rather than adding a new `s14.overdue` key, so the profile page and the manager dashboard widget show identical overdue copy/formatting; `actionItems.complete.success`/`.error` (sibling of `managementNotes.*` toasts, ~line 584), with `.error` covering both the already-completed and already-cancelled 409 cases -- do not touch the pre-existing unrelated `dashboard.actionItems.*` (line 60)
- `services/backend/test/employee-profile.e2e-spec.ts` -- add the four Self scenarios above, plus a fifth: an employee with zero assigned items sees the S14 empty state; none of these five exist today (only S6/S15 denial and S9-S13 Self coverage exist)
- `services/frontend/e2e/employee-profile-assembly.spec.ts` -- add coverage for S7/S8 Self-viewer rendering (flagged/shared record shown, unflagged/unshared record absent -- no existing assertions touch `ManagementNotesSectionCard`/`FeedbackSectionCard` today) and for S14 rendering + mark complete

## Design Notes

`ActionItemsSectionCard` gates its "mark complete" button on `accessLevel !== 'RW'` rather than omitting an RW branch outright: the access matrix grants S14 `RW` to Reporting line and PP too, and the profile assembler loads the S14 envelope for any non-`none` access level, so a manager or PP viewing a report's profile does reach this card. Since `completeActionItem` stays assignee-gated server-side and no RW action-item management UI (create/edit/cancel) exists anywhere in the product yet, RW viewers see the same list read-only, with no dead button — building that management surface is a follow-up, not this spec's scope.

`buildSection`'s `role === 'Self'` filter (`action-items.service.ts:238-240`) is redundant with `loadItemsForAssignee`'s own `where: { assigneeId }` scoping — the query already returns only the subject's own items before the role check runs. `SELF_VIEW_S14_OWN_ONLY` is satisfied by that query scope, not by the role branch; don't mistake the role branch for the load-bearing gate if it's touched later.

Also already correct, with no changes needed: `management-notes-section.provider.ts`+`.service.ts` (S7), `feedbacks-section.provider.ts`+`.service.ts` (S8), `action-items-section.provider.ts`'s `getSection` + `action-items.service.ts`'s `buildSection`/`completeActionItem` (S14), and `ManagementNotesSectionCard`/`FeedbackSectionCard`.

## Tasks & Acceptance

**Execution:**
- [x] `types/employee-profile.ts` -- add `ActionItemAuthor`/`ActionItemsSection`/`ActionItem` types
- [x] `api/services/action-item.service.ts` + `api/hooks/useActionItemMutations.ts` + `hooks/data/useActionItemsData.ts` -- complete-item write path, per-item pending state, 409-aware error toast
- [x] `components/ActionItemsSection/hooks/useActionItemsSection.ts` + `components/ActionItemsSection/ActionItemsSection.tsx` -- renders own items, read-only for RW, mark complete for Self only
- [x] `profile-sections.tsx` wiring + `translation.json` additions -- closes the S14 rendering gap
- [x] `employee-profile.e2e-spec.ts` -- Self-viewer S7/S8/S14 coverage per I/O matrix, plus the S14 empty-state case
- [x] `employee-profile-assembly.spec.ts` -- Playwright coverage for S7/S8 rendering and the new S14 section + mark complete

### Review Findings

- [x] [Review][Patch] UTC-normalize date-only due dates in `ActionItemsSection` [`ActionItemsSection.tsx:19-30`]
- [x] [Review][Patch] Swallow `mutateAsync` rejection after `onError` toast handling [`useActionItemsData.ts:10-16`]
- [x] [Review][Patch] Backend e2e: 409 on double-complete and complete-cancelled [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Backend e2e: cancelled item metadata in S14 profile [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Backend e2e: assert `dueDate` unchanged after complete [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Playwright: S14 empty state, cancelled item, 409 toast, PP read-only, overdue UI [`employee-profile-assembly.spec.ts`]
- [x] [Review][Patch] Remove vacuous S7/S8 hidden-record assertions; rename test to read-only focus [`employee-profile-assembly.spec.ts`]
- [x] [Review][Defer] `SELF_RISK_INVISIBLE` dashboards/notifications scope — deferred, pre-existing: profile-only assertion matches story backend scope; dashboard risk surfaces are out of story 2.5

**Acceptance Criteria:** (human-readable restatement of the frozen I/O & Edge-Case Matrix, cross-referenced by scenario ID for traceability — keep both in sync if either changes)
- Given a feedback record about me has `sharedWithEmployee` set, and a note has `visibleForEmployee` set, and a second note has neither flag, when I view S7 and S8, then I see the shared feedback and the flagged note; the unflagged note is entirely absent (`SELF_VIEW_S7_FILTERED`, `SELF_VIEW_S8_FILTERED`)
- Given I have a risk record at any level, when I view any part of my own profile, including dashboards and notifications, then no risk level, trend, or history is visible or inferable anywhere (`SELF_RISK_INVISIBLE`)
- Given I mark my own action item complete, then a completion date is recorded and displayed, but I cannot edit its title or due date, or cancel it (`SELF_COMPLETE_ACTION_ITEM`)

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint` -- expect clean
- `cd services/backend && npm run test:e2e -- employee-profile` -- expect new Self S7/S8/S14 scenarios (including the empty-state case) to pass
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` -- expect clean
- `cd services/frontend && npm run test` -- expect new S7/S8/S14 Playwright coverage to pass

**Manual checks:**
- As a seeded employee with a flagged note, an unflagged note, a shared feedback record, an unshared one, and two open action items assigned to me: confirm S7/S8 show only the flagged/shared records, S14 shows both items with mark complete only, and completing one records a date without touching the other.
- As the same employee, confirm a third, already-cancelled action item shows its cancellation date/reason with no mark complete button, and that with zero assigned items S14 renders its empty state.
- As a Reporting line or PP viewer opening this employee's profile (S14 resolves `RW` for them): confirm S14 renders read-only with no mark complete button.
- As the same seeded employee, confirm the risk dashboard and any notification list never surfaces this employee's own risk level or trend.

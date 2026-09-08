---
title: 'Story 9.5 — View All Mentor-Mentee Pairs (Active and Ended)'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '19617612b21bb730c5ab6ba700c9bb40319d8314'
context:
  - '_bmad-output/implementation-artifacts/epic-9-context.md'
  - '_bmad-output/implementation-artifacts/spec-9-4-automatic-mentor-status-transitions.md'
  - 'services/backend/AGENTS.md'
  - 'services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The Mentorship Hub lists only active pairs with mentor/mentee names and an End action — no start/end dates, no ended pairs, no status filter, and names are plain text. Managers and PP cannot see the overall mentorship program state at a glance (FR-39).

**Approach:** Extend `GET /mentorship/pairs` to return scoped active, ended, and all pairs with dates and derived status. Replace the Hub's active-only table with a unified "All pairs" section: status filter chips, date columns, profile links on names, and End action retained only for active rows.

## Boundaries & Constraints

**Always:**
- Viewer must hold `assign_end_mentorships`; pairs scoped via `listS6SubjectIds` — include pair when mentor OR mentee is in scope (same rule as 9.2–9.3).
- Derived status: `endedAt == null` → `active`; else `ended`. No persisted status column.
- Response fields per row: id, mentorId, mentorDisplayName, menteeId, menteeDisplayName, startedAt, endedAt (null when active), status.
- `status` query param: `active` | `ended` | `all` (default `all`). Invalid value → 400.
- Profile links use normal `/employees/:id` navigation — no new access granted by this view.
- End-pair flow unchanged (9.3); active rows keep End action.

**Ask First:**
- Whether server-side pagination is needed now (recommend: defer — in-memory filter like Risk Dashboard until pair volume warrants it).

**Never:**
- Expose `closureFeedback` in the list API (manager-only on pair end / S13 history).
- Pairs outside viewer's Manager/PP scope.
- New DB columns or migrations.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Scoped all pairs | Viewer has access to mentor or mentee | All in-scope pairs returned with dates + status | Empty array if no subjects |
| Out-of-scope pair | Neither participant in subjects | Omitted from list | N/A |
| Filter active | `?status=active` | Only `endedAt: null` pairs | N/A |
| Filter ended | `?status=ended` | Only pairs with `endedAt` set | N/A |
| Invalid status | `?status=foo` | Rejected | 400 Bad Request |
| Profile link | Click mentor/mentee name | Navigate to profile if viewer has normal access | Standard access denial on profile page |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/mentorship/mentorship.controller.ts` L146–160 — relax status guard; default `all`
- `services/backend/src/modules/mentorship/mentorship-assignment.service.ts` L128–175 — generalize `listActivePairs` → `listPairs(viewerId, status)`
- `services/backend/src/modules/mentorship/entities/mentorship-pair.entity.ts` L18–41 — extend or add `MentorshipPairListItemEntity` with `endedAt`, `status`
- `services/backend/src/modules/mentorship/mentorship.swagger.ts` L61–68 — document status values + new fields
- `services/backend/prisma/schema.prisma` L623–637 — `startedAt`, `endedAt` sufficient; no migration
- `services/backend/test/mentorship.e2e-spec.ts` L723–759 — extend listing tests for ended/all/scope
- `services/backend/src/modules/mentorship/__tests__/mentorship-assignment.service.spec.ts` L241–268 — unit tests for status filters
- `services/frontend/src/api/services/mentorship.service.ts` L41–45 — `getPairs(status?)` replacing `getActivePairs`
- `services/frontend/src/types/mentorship.ts` L33–44 — extend pair list type with `endedAt`, `status`
- `services/frontend/src/api/hooks/useMentorship.ts` — query key includes status param
- `services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx` L60–109 — replace active-only table with all-pairs table + filter
- `services/frontend/src/pages/MentorshipHub/hooks/useMentorshipHubPage.ts` — status filter state
- `services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/MentorshipSection.tsx` L126–131 — profile `Link` pattern to reuse
- `services/frontend/src/pages/RiskDashboardPage/RiskDashboardPage.tsx` L54–68 — filter chip UX reference
- `services/frontend/src/locales/en/translation.json` L255–299 — add all-pairs, status, date column keys

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/mentorship/mentorship-assignment.service.ts` — `listPairs(viewerId, status)` with parameterized `endedAt` filter
- [x] `services/backend/src/modules/mentorship/entities/mentorship-pair.entity.ts` — list entity with `endedAt`, `status`
- [x] `services/backend/src/modules/mentorship/mentorship.controller.ts` + swagger — accept `active|ended|all`, default `all`
- [x] `services/backend/src/modules/mentorship/__tests__/mentorship-assignment.service.spec.ts` — status filter + scope matrix
- [x] `services/backend/test/mentorship.e2e-spec.ts` — e2e: ended/all listing, out-of-scope exclusion, invalid status
- [x] `services/frontend/src/types/mentorship.ts` + `api/services/mentorship.service.ts` + `api/hooks/useMentorship.ts` — typed `getPairs(status?)`
- [x] `services/frontend/src/pages/MentorshipHub/` — unified all-pairs table: status filter, dates, profile links, End on active only
- [x] `services/frontend/src/locales/en/translation.json` — column, filter, empty-state copy
- [x] Mutation invalidation — create/end pair invalidates all pairs query keys

**Acceptance Criteria:**
- Given several pairs exist, some within and some outside my Manager/PP access, when I open the All pairs view, then I see only in-scope pairs with mentor, mentee, start date, end date, and status
- Given the list contains both active and ended pairs, when I filter by status, then the table updates accordingly
- Given I click a mentor or mentee name, when I have normal profile access, then I navigate to that profile; no new access is granted by this view

## Design Notes

Status filter UX: mirror Risk Dashboard counter cards — three chips (All / Active / Ended) above the table; client refetches on change via `?status=` query param.

Active-row End action stays in the actions column; ended rows show em dash or omit the column action.

## Verification

**Commands:**
- `cd services/backend && npm test -- mentorship-assignment` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- mentorship` — expected: listing e2e pass
- `cd services/backend && npm run lint && npm run build` — expected: clean
- `cd services/frontend && npm run typecheck && npm run lint` — expected: clean

**Manual checks (if no CLI):**
- Hub: verify all/ended/active filters, date columns, profile links, End only on active rows; confirm out-of-scope pairs absent

## Suggested Review Order

**Scoped pair listing API**

- Generalized list query with status filter and derived lifecycle fields
  [`mentorship-assignment.service.ts:134`](../../services/backend/src/modules/mentorship/mentorship-assignment.service.ts#L134)

- Controller accepts active, ended, or all with default all
  [`mentorship.controller.ts:146`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L146)

- Swagger documents status query param and new response fields
  [`mentorship.swagger.ts:61`](../../services/backend/src/modules/mentorship/mentorship.swagger.ts#L61)

**Mentorship Hub UI**

- Unified all-pairs table with section-scoped loading and filter chips
  [`MentorshipHubPage.tsx:68`](../../services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx#L68)

- Status filter state wired into pairs query hook
  [`useMentorshipHubPage.ts:8`](../../services/frontend/src/pages/MentorshipHub/hooks/useMentorshipHubPage.ts#L8)

- Pairs query key and API client accept status parameter
  [`useMentorship.ts:7`](../../services/frontend/src/api/hooks/useMentorship.ts#L7)

**Tests**

- Unit coverage for active, ended, and all filter branches
  [`mentorship-assignment.service.spec.ts:241`](../../services/backend/src/modules/mentorship/__tests__/mentorship-assignment.service.spec.ts#L241)

- E2e scope, filter, and closure-feedback exclusion assertions
  [`mentorship.e2e-spec.ts:723`](../../services/backend/test/mentorship.e2e-spec.ts#L723)

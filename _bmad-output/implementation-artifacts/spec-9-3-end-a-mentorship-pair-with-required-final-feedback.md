---
title: 'Story 9.3 — End a Mentorship Pair with Required Final Feedback'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'cd5367d4d6a4c11a7b8e71da88a243a4e9e6f34e'
context:
  - '_bmad-output/implementation-artifacts/epic-9-context.md'
  - '_bmad-output/implementation-artifacts/spec-9-2-assign-a-mentor-mentee-pair.md'
  - 'services/backend/AGENTS.md'
  - 'services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 9.2 creates active pairs, but there is no way to end them. `MentorshipPairService.endActivePairForMentee` is a Story 1.7 stub that sets `endedAt` without feedback, permission checks, timeline events, or conflict protection. Managers/PP cannot close relationships or capture outcomes.

**Approach:** Add `closureFeedback` to `MentorshipPair`, expose a permission-gated end-pair API that rejects empty feedback and already-ended pairs (409), atomically writes `mentorshipEnd` timeline events for both employees, and returns updated derived `mentorStatus`. Extend Mentorship Hub and profile S13 with an "End pair" dialog (primary disabled until feedback entered) and read-only ended-pair history with closure notes visible only to reporting/project line and PP (D6).

## Boundaries & Constraints

**Always:**
- End pair requires `assign_end_mentorships` and viewer access to the mentor or mentee via `SectionAccessGate.listS6SubjectIds`.
- `closureFeedback` is mandatory (trimmed non-empty) before `endedAt` is set; empty/missing → 400, pair stays active.
- Store closure feedback on `MentorshipPair` only — never create Epic 11 Feedback records (D6).
- Redact `closureFeedback` for Self, Colleague, and SharedLink audiences; expose only to ReportingLine, ProjectLine, PP, and FullAccess.
- End + dual `mentorshipEnd` timeline events commit in one `$transaction` with `tx` passed to `TimelineEventWriter`.
- Concurrent end of an already-ended pair → 409 Conflict (D20); do not overwrite existing closure note.
- After end, `deriveMentorStatus` reflects revert (last active pair ended → `openToMentoring` or `none` per self-flag).
- Ended pairs remain visible in S13 history on both profiles; active mentor/mentee links clear when pair ends.

**Ask First:**
- Whether Hub should list all scoped active pairs in a dedicated table vs. end actions only from profile S13 (recommend: both — minimal active-pairs list on Hub).

**Never:**
- Full organization-wide all-pairs table (Story 9.5).
- Departure auto-close hook (Epic 14).
- Direct mentor-status writes or Epic 11 feedback routing.
- Showing closure feedback to mentor, mentee, or colleagues.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path end | Active pair, non-empty feedback, viewer in scope | `endedAt` + `closureFeedback` set; `mentorshipEnd` on both timelines; mentor `mentorStatus` updated in response | N/A |
| Missing feedback | Active pair, empty/whitespace `closureFeedback` | Pair stays active | 400 Bad Request |
| Out of scope | Viewer lacks S6 access to mentor and mentee | Denied | 403 Forbidden |
| Already ended | `endedAt` already set (concurrent request) | No overwrite of feedback | 409 Conflict |
| Missing permission | Viewer lacks `assign_end_mentorships` | Denied | 403 Forbidden |
| Closure visibility | Self views own ended pair in S13 | Pair history shown; `closureFeedback` omitted | N/A |
| Closure visibility (manager) | ReportingLine/PP views subject's ended pair | `closureFeedback` included | N/A |
| Last pair revert | Mentor's only active pair ends, flag still on | `mentorStatus` → `openToMentoring` | N/A |
| Last pair revert (flag off) | Mentor's only active pair ends, flag off | `mentorStatus` → `none` | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` L554–567 — add `closureFeedback String?`; migration required
- `services/backend/src/modules/mentorship/mentorship-pair.service.ts` L55–65 — replace `endActivePairForMentee` stub with transactional end accepting feedback + `endedAt: null` guard
- `services/backend/src/modules/mentorship/mentorship-assignment.service.ts` L76–153 — mirror `createPair` orchestration for `endPair`; reuse scope via `listS6SubjectIds`
- `services/backend/src/modules/mentorship/mentorship.controller.ts` L137–158 — add `PATCH mentorship/pairs/:pairId/end` and `GET mentorship/pairs?status=active`; reuse `assertAssignEndMentorshipsPermission` L172–185
- `services/backend/src/modules/mentorship/mentorship.service.ts` L17–44, L92–118 — extend `buildSection` with ended-pair history + audience-aware feedback redaction; `deriveMentorStatus` for response
- `services/backend/src/modules/mentorship/mentorship-section.provider.ts` L21–38 — pass `audience.role` into extended `buildSection`
- `services/backend/src/modules/mentorship/entities/mentorship-section.entity.ts` — add `MentorshipPairHistoryEntity` (id, counterpart, dates, status, optional `closureFeedback`)
- `services/backend/src/modules/timeline/timeline.constants.ts` L12–14 — `'mentorshipEnd'` already defined
- `services/backend/test/mentorship.e2e-spec.ts` L313+ — extend with end-flow matrix
- `services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx` — add active-pairs section + End pair action
- `services/frontend/src/pages/MentorshipHub/components/AssignMenteeDialog/` — dialog pattern to copy for `EndPairDialog`
- `services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/MentorshipSection.tsx` L64–102 — add end action (manager/PP) and ended-pair history display
- `services/frontend/src/api/services/mentorship.service.ts` — add `getActivePairs`, `endPair`
- `services/frontend/src/locales/en/translation.json` L195–220 — end-pair copy

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + new migration — add `closureFeedback` column
- [x] `services/backend/src/modules/mentorship/dto/end-mentorship-pair.dto.ts` — `{ closureFeedback: string }` with `@IsNotEmpty` after trim
- [x] `services/backend/src/modules/mentorship/mentorship-pair.service.ts` — transactional `endActivePair(pairId, feedback, endedAt, tx)` with `where: { id, endedAt: null }`
- [x] `services/backend/src/modules/mentorship/mentorship-assignment.service.ts` — `endPair(viewerId, pairId, feedback)` + `listActivePairs(viewerId)` scoped to S6 subjects
- [x] `services/backend/src/modules/mentorship/mentorship.controller.ts` + swagger — `PATCH .../end`, `GET .../pairs?status=active`
- [x] `services/backend/src/modules/mentorship/mentorship.service.ts` + entity — pair history with D6 redaction by `AccessRole`
- [x] `services/backend/src/modules/mentorship/__tests__/` + `test/mentorship.e2e-spec.ts` — unit + e2e for matrix rows
- [x] `services/frontend/src/api/services/mentorship.service.ts` + hooks — `getActivePairs`, `endPair` mutation with cache invalidation
- [x] `services/frontend/src/pages/MentorshipHub/components/EndPairDialog/` — required feedback textarea; submit disabled until non-empty
- [x] `services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx` — active pairs table with End action
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/` — end action on active mentees; ended history with conditional feedback
- [x] `services/frontend/src/locales/en/translation.json` — end-pair strings

**Acceptance Criteria:**
- Given an active pair, when a manager/PP attempts to end it without final feedback, then the pair stays active with no `endedAt`
- Given final feedback is provided and confirmed, when the pair ends, then `endedAt` and `closureFeedback` are persisted, `mentorshipEnd` events exist on both timelines, ended pairs appear in S13 history on both profiles, and if this was the mentor's last active pairing their derived status reverts per the self-flag rule
- Given a concurrent attempt to end an already-ended pair, when the second request arrives, then the API returns 409 and the original closure note is unchanged

## Design Notes

Timeline payload for pair end (mirror 9.2 start):

```ts
// type 'mentorshipEnd', effectiveDate = endedAt date-only ISO
// mentor row: newValue = menteeId; mentee row: newValue = mentorId
// source 'system', authorId = ender employeeId
```

Conflict pattern: `updateMany({ where: { id, endedAt: null }, data: { endedAt, closureFeedback } })` — if `count === 0`, throw `ConflictException`.

## Verification

**Commands:**
- `cd services/backend && npm test && npm run test:e2e -- mentorship` — expected: all pass including new end-pair cases
- `cd services/backend && npm run lint && npm run build` — expected: clean
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expected: clean

**Manual checks (if no CLI):**
- Hub: list active pairs, end with feedback, confirm mentor status revert and S13 history; confirm mentor/mentee cannot see closure note; manager/PP can

## Spec Change Log

- **2026-09-08:** Initial implementation complete; all I/O matrix rows covered by unit/e2e tests.

## Suggested Review Order

**Pair end orchestration**

- Atomic end with feedback, scope gate, and dual timeline writes
  [`mentorship-assignment.service.ts:211`](../../services/backend/src/modules/mentorship/mentorship-assignment.service.ts#L211)

- Conflict-safe pair write with `endedAt: null` guard
  [`mentorship-pair.service.ts:60`](../../services/backend/src/modules/mentorship/mentorship-pair.service.ts#L60)

**API surface**

- Permission-gated list/end routes for active pairs
  [`mentorship.controller.ts:143`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L143)

**Closure feedback visibility (D6)**

- Audience-aware pair history redaction in S13
  [`mentorship.service.ts:139`](../../services/backend/src/modules/mentorship/mentorship.service.ts#L139)

**Mentorship Hub UI**

- Active pairs table and end dialog with required feedback
  [`MentorshipHubPage.tsx:58`](../../services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx#L58)

- Submit disabled until non-empty feedback text
  [`EndPairDialog.tsx:48`](../../services/frontend/src/pages/MentorshipHub/components/EndPairDialog/EndPairDialog.tsx#L48)

**Profile S13**

- End actions and ended-pair history on employee profile
  [`MentorshipSection.tsx:119`](../../services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/MentorshipSection.tsx#L119)

**Tests**

- End-flow e2e matrix including visibility and status revert
  [`mentorship.e2e-spec.ts:579`](../../services/backend/test/mentorship.e2e-spec.ts#L579)

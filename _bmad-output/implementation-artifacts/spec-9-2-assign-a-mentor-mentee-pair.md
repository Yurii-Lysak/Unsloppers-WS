---
title: 'Story 9.2 — Assign a Mentor-Mentee Pair'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '72d6290330eb1fa74fef13321510bf3b6b97ce75'
context:
  - '_bmad-output/implementation-artifacts/epic-9-context.md'
  - '_bmad-output/implementation-artifacts/spec-9-1-self-flag-open-to-mentoring.md'
  - 'services/backend/AGENTS.md'
  - 'services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Managers and PP with `assign_end_mentorships` can list willing mentors (9.1) but cannot create pairs. `MentorshipPairService.createActivePair` is an internal stub with no permission, consent, access, or timeline integration. No Mentorship Hub UI exists for assignment.

**Approach:** Expose a permission-gated pair-creation API that validates mentor consent (`openToMentoring`), mentee access scope, and DB constraints inside a transaction that also writes `mentorshipStart` timeline events for both employees. Add a Mentorship Hub page listing willing mentors globally and a scoped mentee picker dialog to complete assignment.

## Boundaries & Constraints

**Always:**
- Pair creation requires `assign_end_mentorships` and mentee within viewer's resolved access (`SectionAccessGate.listS6SubjectIds` or equivalent S13 RW check).
- Reject pair creation when mentor `openToMentoring` is false at write time (D20), even if still visible elsewhere.
- Willing-mentor pool remains company-wide (`openToMentoring: true`); only mentee picker is scoped.
- Mentor status stays derived — first active pair yields `'mentor'` via existing `deriveMentorStatus`; never persist or accept direct status writes.
- Pair + both timeline events commit atomically in one `$transaction` with `tx` passed to `TimelineEventWriter`.
- Hub nav and assignment UI gated on `assign_end_mentorships`.

**Ask First:**
- Whether willing-mentor rows should also show derived `mentorStatus` badge (not required by AC).

**Never:**
- Pair end flow, closure feedback, or status revert rules (Stories 9.3–9.4).
- All-pairs table (9.5).
- Allowing a mentee to have two active pairs (DB partial unique already enforces one).
- Global/unscoped mentee picker (`listLookupOptions` alone is insufficient).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path assign | Mentor flagged open, mentee in scope, no active pair for mentee | Active pair with `startedAt`; mentor derived status `'mentor'`; `mentorshipStart` on both timelines | N/A |
| Mentor not willing | `openToMentoring: false` at POST time | Pair rejected | 400 Bad Request |
| Mentee out of scope | UM picks employee outside `listS6SubjectIds` | Pair rejected | 403 Forbidden |
| Self-pair | `mentorId === menteeId` | Pair rejected | 400 Bad Request |
| Duplicate active mentee | Mentee already has active pair | Pair rejected | 400 Bad Request |
| Missing permission | Viewer lacks `assign_end_mentorships` | Denied | 403 Forbidden |
| Scoped mentee list | UM with subordinates-only access | Picker/API returns only in-scope employees | Empty list if none in scope |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/mentorship/mentorship-pair.service.ts` L12–38 — reuse `createActivePair`; extend orchestration layer above it, not permission logic inside stub
- `services/backend/src/modules/mentorship/mentorship.controller.ts` L100–134 — `MentorshipPoolController` pattern for `assign_end_mentorships` gate on new routes
- `services/backend/src/modules/mentorship/mentorship.service.ts` L76–118 — `listWillingMentors`, `deriveMentorStatus`; mentor list unchanged (global, flag-only)
- `services/backend/src/modules/access/section-access-gate.service.ts` L95–123 — `listS6SubjectIds` for mentee scope (reporting + PP + project line)
- `services/backend/src/modules/contracts/timeline-event-writer.contract.ts` L49–58 — C4 `recordTimelineEvent` with optional `tx`
- `services/backend/src/modules/timeline/timeline.constants.ts` L12 — `'mentorshipStart'` event type
- `services/backend/src/modules/timeline/timeline-event-writer.service.ts` L21–60 — implementation; pass `tx` inside pair transaction
- `services/backend/src/modules/contracts/permission-keys.ts` L15 — `ASSIGN_END_MENTORSHIPS`
- `services/backend/prisma/schema.prisma` L554–567 — `MentorshipPair` model; partial unique active mentee in migration
- `services/backend/test/mentorship.e2e-spec.ts` — extend with assignment, scope, consent, timeline assertions
- `services/frontend/src/router/index.tsx` L73–75 — mirror `resourcing` route pattern for `/mentorship`
- `services/frontend/src/pages/Resourcing/ResourcingPage.tsx` — hub list + dialog reference
- `services/frontend/src/components/SideMenu/` — add Mentorship nav item gated on permission
- `services/frontend/src/api/services/mentorship.service.ts` — add `getWillingMentors`, `getAssignableMentees`, `createPair`
- `services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/` — 9.1 S13; pair results visible read-only after assign

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/mentorship/dto/create-mentorship-pair.dto.ts` — `{ mentorId, menteeId }` UUIDs; whitelist validation
- [x] `services/backend/src/modules/mentorship/mentorship-assignment.service.ts` — orchestrate permission, consent, scope, transaction, timeline writes
- [x] `services/backend/src/modules/mentorship/mentorship.controller.ts` — `POST /mentorship/pairs`, `GET /mentorship/assignable-mentees`; swagger decorators
- [x] `services/backend/src/modules/mentorship/mentorship.module.ts` — register assignment service; inject `TimelineEventWriter` token
- [x] `services/backend/src/modules/mentorship/__tests__/mentorship-assignment.service.spec.ts` — unit tests for matrix edge cases
- [x] `services/backend/test/mentorship.e2e-spec.ts` — e2e: create pair, scope denial, consent denial, timeline rows for both employees
- [x] `services/frontend/src/api/services/mentorship.service.ts` + hooks — query/mutation primitives for hub APIs
- [x] `services/frontend/src/pages/MentorshipHub/` — willing-mentor table, assign dialog with scoped mentee combobox, success toast + cache invalidation
- [x] `services/frontend/src/router/index.tsx` + SideMenu — `/mentorship` route and nav gated on `assign_end_mentorships`
- [x] `services/frontend/src/locales/en/translation.json` — hub + assignment copy

**Acceptance Criteria:**
- Given a willing mentor with zero existing pairs and a mentee within my access scope, when I create the pair via Hub or API, then an active pair exists with a start date, the mentor's derived status is `mentor`, and `mentorshipStart` events exist on both timelines
- Given I am a UM whose access covers only my subordinates and their reports, when I open the assignment mentee picker, then only in-scope employees appear while the willing-mentor list remains company-wide

## Design Notes

Timeline payload for pair start (AD-7 raw values):

```ts
// mentor row: oldValue null, newValue = menteeId (UUID string)
// mentee row: oldValue null, newValue = mentorId (UUID string)
// type 'mentorshipStart', effectiveDate = startedAt ISO date-only, source 'system', authorId = assigner
```

Orchestration sketch: `$transaction(async (tx) => { assert consent + scope; create pair via tx; recordTimelineEvent ×2 with tx })`.

## Verification

**Commands:**
- `cd services/backend && npm test && npm run test:e2e -- mentorship` — expected: all pass including new assignment cases
- `cd services/backend && npm run lint && npm run build` — expected: clean
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expected: clean

**Manual checks (if no CLI):**
- Hub: list willing mentors, assign mentee in scope, confirm pair on both profiles S13/header; confirm out-of-scope mentee blocked

## Spec Change Log

- **2026-09-08:** Story marked `done` after PRs merged (BE #57, FE #40, WS #69).

## Suggested Review Order

**Pair assignment orchestration**

- Atomic pair create with consent, scope, and dual timeline writes
  [`mentorship-assignment.service.ts:62`](../../services/backend/src/modules/mentorship/mentorship-assignment.service.ts#L62)

- Internal pair write path reused with optional transaction client
  [`mentorship-pair.service.ts:14`](../../services/backend/src/modules/mentorship/mentorship-pair.service.ts#L14)

**API surface**

- Permission-gated POST/GET routes for pairs and assignable mentees
  [`mentorship.controller.ts:116`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L116)

**Mentorship Hub UI**

- Hub page with willing-mentor table and assign dialog
  [`MentorshipHubPage.tsx:7`](../../services/frontend/src/pages/MentorshipHub/MentorshipHubPage.tsx#L7)

- Scoped mentee picker excluding selected mentor
  [`AssignMenteeDialog.tsx:16`](../../services/frontend/src/pages/MentorshipHub/components/AssignMenteeDialog/AssignMenteeDialog.tsx#L16)

- Sidebar nav gated on `assign_end_mentorships`
  [`SideMenu.tsx:97`](../../services/frontend/src/components/SideMenu/SideMenu.tsx#L97)

**Tests**

- Assignment service unit matrix and transactional timeline assertions
  [`mentorship-assignment.service.spec.ts:14`](../../services/backend/src/modules/mentorship/__tests__/mentorship-assignment.service.spec.ts#L14)

- E2e pair creation, scope, consent, and permission denial
  [`mentorship.e2e-spec.ts:313`](../../services/backend/test/mentorship.e2e-spec.ts#L313)

---
title: 'Story 9.1 — Self-Flag Open to Mentoring'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '1c5408954468196ad7bf4e0fa268509e19b5710f'
context:
  - '_bmad-output/implementation-artifacts/epic-9-context.md'
  - 'services/backend/AGENTS.md'
  - 'services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Employees cannot declare willingness to mentor or view their own mentorship relationships. S13 is registered in access but has no provider, so the profile shows it as unavailable. The open-to-mentoring flag, willing-mentor pool, and self-view mentor/mentee data do not exist.

**Approach:** Add `Employee.openToMentoring`, an S13 section provider, and a self-only PATCH for the flag. Expose a permission-gated willing-mentors list API (identity-card fields + flag only). Widen S1 mentor visibility to include Self (Story 1.7 deferred this to 9.1). Compute mentor status read-only from flag + active pairs; reject any direct status write.

## Boundaries & Constraints

**Always:**
- Self may write only `openToMentoring` on own profile; pairs and closure notes are read-only for all audiences in 9.1.
- Willing-mentors list is company-wide but gated by `assign_end_mentorships`; expose identity-card data + flag only — never full S13 content.
- Mentor status is derived (`mentor` if active mentor pair exists; else `openToMentoring` if flag on; else `none`) — never persisted or accepted via API.
- S13 remains never-shareable; Colleague access stays `none`.
- Follow existing section-provider + parallel write-route pattern (S8 feedbacks).

**Ask First:**
- Adding a dedicated Mentorship Hub page shell (Epic 9 UX mentions Hub nav — defer unless PO wants minimal list UI in 9.1).

**Never:**
- Pair assignment/end flows (Stories 9.2–9.3), full status-transition engine polish (9.4), all-pairs table (9.5).
- Closure feedback field on pairs, timeline pair events, departure auto-close hooks.
- Direct `mentorStatus` column or accepting `mentorStatus` / `status` in request bodies.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Self enables flag | Authenticated self, `PATCH { openToMentoring: true }` | Flag persisted; S13 reflects change; employee appears in willing-mentors list | 403 if not self or S13 not RW |
| Self disables flag | Active mentor pair exists, flag off | Flag off; person removed from willing-mentors list; active pair unchanged; derived status stays `mentor` | N/A |
| Direct status write | Body includes `mentorStatus` or `status` | Request rejected | 400 Bad Request |
| Self views relationships | Active mentor + mentees in DB | S13 and S1 header show them read-only | Omit missing relations |
| Non-assigner lists pool | Viewer without `assign_end_mentorships` | List denied | 403 Forbidden |
| Colleague views S13 | Colleague audience | Section absent / 403 per matrix | Existing gate behavior |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — add `openToMentoring Boolean @default(false)` on `Employee`; `MentorshipPair` already at L522–535
- `services/backend/src/modules/mentorship/mentorship.module.ts` — register new provider, service, controller
- `services/backend/src/modules/mentorship/active-mentor-lookup.service.ts` — reuse for mentor resolution; add mentee lookup sibling
- `services/backend/src/modules/access/identity-section.provider.ts` L12–16 — widen `MENTOR_VISIBLE_ROLES` to include `Self`
- `services/backend/src/modules/feedbacks/feedbacks-section.provider.ts` — section provider + gate pattern reference
- `services/backend/src/modules/feedbacks/feedbacks.controller.ts` — parallel write-route + `sectionGate.requireSection` pattern
- `services/backend/test/support/access-matrix.ts` L180–190 — S13 field-level qualifier to enforce in provider/service
- `services/backend/test/employee-profile.e2e-spec.ts` L207–297 — update Self-omits-mentor tests after D5 widening
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — register S13 renderer (currently generic fallback L34)
- `services/frontend/src/types/employee-profile.ts` — add `MentorshipSection` types
- `services/frontend/src/pages/EmployeeProfilePage/components/ProfileHeader/ProfileHeader.tsx` L42–48 — already renders S1 mentor when present
- `services/frontend/src/pages/EmployeeProfilePage/components/ManagementNotesSection/` — Switch + RW gating reference for self-editable field

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — add `openToMentoring` on `Employee`
- [x] `services/backend/src/modules/mentorship/mentorship.service.ts` — derive status, load S13 payload, update flag, build willing-mentors list
- [x] `services/backend/src/modules/mentorship/mentorship-section.provider.ts` — `@RegisterProvider('section', 'S13')` with audience narrowing
- [x] `services/backend/src/modules/mentorship/mentorship.controller.ts` — `PATCH employees/:id/mentorship/open-to-mentoring`, `GET mentorship/willing-mentors`
- [x] `services/backend/src/modules/mentorship/entities/` + DTOs + swagger — section entity, patch DTO whitelist
- [x] `services/backend/src/modules/access/identity-section.provider.ts` — include Self in mentor visibility
- [x] `services/backend/src/modules/mentorship/__tests__/` + `test/mentorship.e2e-spec.ts` — unit + e2e for flag, list, status rejection, self mentor visibility
- [x] `services/frontend/src/api/services/mentorship.service.ts` + `api/hooks/` + `hooks/data/` — query/mutation primitives
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/` — Switch for flag, read-only mentor/mentees/status display
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — wire S13 renderer
- [x] `services/frontend/src/locales/en/translation.json` — S13 + mentorship copy

**Acceptance Criteria:**
- Given I view my own profile S13, when I toggle open to mentoring on and save, then the flag persists and I appear in the willing-mentors API response for holders of `assign_end_mentorships`
- Given I have an assigned mentor and/or mentees, when I view my own profile, then I see them read-only in S13 and in the profile header
- Given any write attempt includes `mentorStatus` or `status`, when submitted, then the server rejects with 400 and no data changes

## Design Notes

Derived status for 9.1 (9.4 will own full transition rules on pair end):

```ts
// mentor if any active pair where subject is mentorId
// else openToMentoring ? 'openToMentoring' : 'none'
```

PATCH DTO: `{ openToMentoring: boolean }` only — use `@Allow()` whitelist or separate DTO class; strip unknown keys via ValidationPipe.

## Verification

**Commands:**
- `cd services/backend && npm test && npm run test:e2e -- mentorship` — expected: all pass
- `cd services/backend && npm run lint && npm run build` — expected: clean
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expected: clean

**Manual checks (if no CLI):**
- Self profile: toggle flag, refresh, confirm S13 + header mentor/mentees; confirm willing-mentors list membership via API

## Spec Change Log

## Suggested Review Order

**Schema & derived status**

- Single source of truth for willingness flag on Employee
  [`schema.prisma:47`](../../services/backend/prisma/schema.prisma#L47)

- Derived mentor status from active pairs + flag, never persisted
  [`mentorship.service.ts:92`](../../services/backend/src/modules/mentorship/mentorship.service.ts#L92)

**API & access control**

- Self-only PATCH with S13 RW gate and status-write rejection
  [`mentorship.controller.ts:40`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L40)

- Permission-gated company-wide willing-mentors pool
  [`mentorship.controller.ts:110`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L110)

- S13 section provider registering profile assembly
  [`mentorship-section.provider.ts:12`](../../services/backend/src/modules/mentorship/mentorship-section.provider.ts#L12)

**Profile header (D5 widening)**

- Self now included in mentor visibility roles
  [`identity-section.provider.ts:12`](../../services/backend/src/modules/access/identity-section.provider.ts#L12)

**Frontend S13 section**

- Self-only Switch with read-only mentor/mentees display
  [`MentorshipSection.tsx:51`](../../services/frontend/src/pages/EmployeeProfilePage/components/MentorshipSection/MentorshipSection.tsx#L51)

- Profile section wiring and types
  [`profile-sections.tsx:157`](../../services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx#L157)

**Tests**

- E2E coverage for flag, pool, status rejection, relationships
  [`mentorship.e2e-spec.ts:75`](../../services/backend/test/mentorship.e2e-spec.ts#L75)

---
title: 'Profile Header Shows Manager, PP and Mentor'
type: 'feature'
created: '2026-09-02'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'a88a0cc3f1cb63b94181c433469cae778368f5c5'
frontend_baseline_commit: 'af21cd0739f00fd9c7ddfb285a949723341fb389'
story_key: '1-7-profile-header-shows-manager-pp-and-mentor'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.6 assembles profile sections and returns manager/PP inside S1, but the profile page header shows only `displayName` plus the Access Scope Chip — not the reporting context required by PRD §4.2 ("the profile header shows the manager, the people partner and the mentor"). Mentor is not persisted or resolved anywhere yet; `IdentitySectionProvider` explicitly defers it.

**Approach:** Add a minimal read-only mentorship substrate (`MentorshipPair` + `ActiveMentorLookup` contract) so S1 can resolve the current active mentor. Extend `IdentitySectionProvider` to populate `mentor` only for the ReportingLine, ProjectLine, and PP audiences (D5). Build a `ProfileHeader` SPA component that renders the manager, people partner, and mentor (when present in the S1 payload) as i18n-labeled links to `/employees/:id`, sourced exclusively from the assembled profile response — no second fetch.

## Boundaries & Constraints

**Always:**
- Header relationship data comes **only** from `GET /employees/:employeeId/profile` → `sections.S1.data` when S1 is present with `data` (not `unavailable`). Do not add parallel top-level header fields on the profile DTO. Top-level `displayName` remains the fallback title when S1 is `unavailable` or absent.
- **D5 mentor visibility (field-level exception — access-model Rule 8):** Include `mentor` in the S1 payload only when `audience.role` is `ReportingLine`, `ProjectLine`, or `PP`. Use an **allow-list** on display role — not a deny-list (`!== 'Colleague'` is wrong and would leak mentor to `Self`). Omit the `mentor` key entirely for `Colleague`, `Self`, `SharedLink`, `FullAccess` (pre-C13 Colleague-equivalent fallthrough per Story 1.6), and any other role. Never send `null` for `mentor` to forbidden audiences. D5 does not use section-grant union logic — it is a single-field carve-out keyed to `audience.role` (highest-rank matched audience from C1). When a viewer matches PP + Colleague, `audience.role` is `PP` and mentor is allowed; when only Colleague (or SharedLink / FullAccess fallthrough), mentor is omitted.
- **Active mentor resolution:** One active pair per mentee — the `MentorshipPair` row where `menteeId = subjectId` and `endedAt IS NULL`. Active means `endedAt IS NULL` only (no future-`endedAt` semantics). If multiple active rows exist (bad data), lookup picks the row with the latest `startedAt`. If none, omit `mentor` from the payload (not an error). Reject `mentorId === menteeId` in the internal pair-create helper.
- **Lookup failure / orphan data:** When `mentorId` references a missing `Employee`, or the mentor row has no resolvable display name, treat as no mentor — omit `mentor` from S1; do not 500 the profile. `displayName` on relations uses `user.name?.trim() || user.email` (same as manager/PP today).
- **Profile header UX (EXPERIENCE.md):** Render manager, PP, and mentor in the identity header block above section cards. Pattern: muted label + value, separated by `·`. Unassigned roles are omitted (no "—" placeholder). Each assigned person is a `Link` to `/employees/:targetId`; destination profile enforces C1 on navigation — no client-side access pre-check beyond using ids already returned by the server. Do not render `data-testid="profile-header-relationships"` when the strip would be empty (no manager, PP, or mentor segments).
- **De-duplication:** Remove manager/PP/mentor from the S1 section card renderer — header is the canonical surface for these three fields per §4.2. When S1 `data` has no identity fields beyond `displayName` (all relationships live in the header), render the S1 section card with its title only and **no body duplicate** of `displayName` (omit the card body, not the whole section key).
- **Wire shape:** Reuse existing `IdentityRelationEntity` (`{ id, displayName }`) for mentor. `manager` and `peoplePartner` may remain `null` in S1 when unassigned (existing provider shape); header omits null segments. `mentor` is never `null` — only present or omitted. ISO dates on `MentorshipPair`; no closure-note field required in this story.
- **Module boundary:** Declare abstract `ActiveMentorLookup` in `contracts`; implement in a new `@Global()` `mentorship` module (Epic 9 domain boundary — pair lifecycle, hub, and write APIs land there; unlike `PeoplePartnerAssignmentService` / `ProjectAssignmentService`, which are org-relationship writes colocated in `access`). `IdentitySectionProvider` injects the contract token only — no direct `MentorshipPair` Prisma access from `access`. Register `MentorshipModule` in `app.module.ts`; export the lookup token from the module.
- **i18n:** Add `employeeProfile.header.manager`, `.peoplePartner`, `.mentor` keys (labels only — names stay as API data).

**Ask First:**
- If Epic 9 later requires Self viewers to see their own mentor in the header (story 9-1 draft AC), that is a separate change — do not widen D5 in this story without human renegotiation.

**Never:**
- Do not build mentorship assignment UI, open-to-mentor flag, pair-ending, S13 section provider, or Mentorship Hub flows (Epic 9).
- Do not add mentor to list/export/search surfaces (Story 1.8 scope).
- Do not add photo/position/department to the header (deferred fields).
- Do not cache mentor resolution across requests.
- Do not render mentor in the header when the S1 payload omits it — no client-side inference from manager/PP ids or other sections.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Manager-line header | ReportingLine viewer; B has manager M, PP P, active mentor T | Header shows M, P, T as links; S1 payload includes all three relations | N/A |
| ProjectLine header | ProjectLine viewer; B has manager, PP, active mentor | Header shows all three; S1 includes `mentor` (D5 allow-list) | N/A |
| Colleague header | Colleague viewer; B has manager, PP, mentor | Header shows manager + PP only; S1 payload has no `mentor` key (D5) | N/A |
| Self header | Self viewer; B has active mentor | Header shows manager + PP if assigned; no mentor in S1 payload or header (D5) | N/A |
| PP viewer | PP viewer; B has all three assigned | Header shows manager, PP, mentor | N/A |
| SharedLink header | SharedLink viewer; B has active mentor; link exposes S1 | Header shows manager + PP if in payload; no `mentor` key (D5) | N/A |
| FullAccess fallthrough | Viewer resolves as Colleague-equivalent (pre-C13); B has mentor | No `mentor` key; header omits mentor segment | N/A |
| No mentor | Any D5-allowed audience; no active `MentorshipPair` for B | `mentor` key absent from S1; header omits mentor segment | N/A |
| No manager/PP | B has null `managerId` / `peoplePartnerId` | Corresponding header segment omitted; wire may still carry `manager: null` / `peoplePartner: null` | N/A |
| Ended pair only | Active pair ended (`endedAt` set); no other active pair | `mentor` omitted — ended mentors are not header-visible | N/A |
| Live org reassignment | B's `managerId` or `peoplePartnerId` changes between requests | Next profile response reflects new values immediately | N/A |
| Live mentorship change | Active pair created or ended between requests | Next profile response reflects current active mentor (or omission) | N/A |
| Multiple active pairs | Two rows with `endedAt IS NULL` for same mentee (bad data) | Lookup returns mentor from row with latest `startedAt`; header matches | N/A |
| Orphan mentor row | Pair references missing `mentorId` Employee | `mentor` omitted; profile still 200 | N/A |
| Provider without audience | `getSection` called without `audience` (dev path) | No `mentor` key — same as forbidden audience | N/A |
| Empty relationship strip | S1 `data` present; no manager, PP, or mentor segments | No `profile-header-relationships` testid in DOM | N/A |
| Wire mentor null | Serializer would emit `mentor: null` | Strip property — forbidden audiences never receive `mentor` | N/A |
| Self-pair create | Internal helper called with `mentorId === menteeId` | Reject create (test/seed helper) | N/A |
| Colleague manager-is-mentor | Colleague viewer; B's manager id equals active mentor id | Header shows manager link only; no mentor segment; no client inference | N/A |
| Link navigation | Viewer clicks header link to employee C | Navigates to `/employees/C`; C1 on that page governs access | 403/404 on destination if denied — no bypass |
| S1 unavailable | S1 grant exists but `status: 'unavailable'` | Header shows top-level `displayName` only; no relationship strip | N/A |
| S1 denied | Viewer lacks S1 grant (not expected on profile today) | No relationship strip; section absent per 1.6 rules | N/A |

</frozen-after-approval>

> **Execution sections** (Tasks, Code Map, Design Notes, Verification) are outside the frozen block and may be updated during implementation without renegotiating intent.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` — add `MentorshipPair` with partial unique on active mentee (`menteeId` where `endedAt IS NULL`); inverse `Employee` relations; run `npm run db:migrate` in `services/backend`
- [x] `services/backend/src/modules/contracts/active-mentor-lookup.contract.ts` — abstract lookup: `getActiveMentorForMentee(menteeId) → { id, displayName } | null`
- [x] `services/backend/src/modules/mentorship/` — `MentorshipModule` (`@Global()`), `ActiveMentorLookupService`, internal pair-create helper for tests/seeds (same pattern as PP/project assignment helpers in `access`; reject self-pairs; no REST, no C8 gates); export lookup token
- [x] `services/backend/src/app.module.ts` — register `MentorshipModule`
- [x] `services/backend/src/modules/access/identity-section.provider.ts` — inject `ActiveMentorLookup`; **allow-list** D5 gate (`ReportingLine` \| `ProjectLine` \| `PP` only); default no mentor when `audience` missing
- [x] `services/backend/src/modules/access/entities/identity-section.entity.ts` — update Swagger description for `mentor` (no longer "deferred")
- [x] `services/backend/src/modules/access/__tests__/identity-section.provider.spec.ts` — mentor included for ReportingLine and ProjectLine when lookup returns; omitted for Colleague, Self, and undefined `audience`; mock `ActiveMentorLookup`
- [x] `services/backend/test/employee-profile.e2e-spec.ts` — HTTP-level D5: ReportingLine and ProjectLine include mentor in S1; Colleague omits; mentorship pair create/end updates next response
- [x] Optional: bootcamp seed or e2e fixture — at least one active `MentorshipPair` so demo profiles can show mentor in header without manual DB edits
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/ProfileHeader/ProfileHeader.tsx` (new) — relationship strip from `sections.S1.data` only when envelope has `data`; omit testid when strip empty; links with `displayName` fallback
- [x] `services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx` — compose `ProfileHeader` below title/access chip
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — remove manager/PP/mentor from S1 card body; omit card body when only `displayName` would duplicate `h1`
- [x] `services/frontend/src/locales/en/translation.json` — `employeeProfile.header.*` label keys
- [x] `services/frontend/e2e/employee-profile-header.spec.ts` (new) — Colleague vs ReportingLine header assertions; `profile-header-relationships` presence/absence; mentor omission for Colleague

**Acceptance Criteria:**
- Given Manager M holds ReportingLine access to B and B has assigned manager, PP, and active mentor, when M opens B's profile, then the header displays all three as navigable links *(matrix: Manager-line header)*
- Given DM D holds ProjectLine access to B and B has an active mentor, when D opens B's profile, then the header displays mentor and S1 includes `mentor` *(matrix: ProjectLine header)*
- Given Viewer V is a Colleague of B and B has an assigned mentor, when V opens B's profile, then the header shows manager and PP but not mentor *(matrix: Colleague header; D5)*
- Given B views own profile and has an active mentor, when B loads the profile, then S1 has no `mentor` key and the header omits mentor *(matrix: Self header; D5)*
- Given B has no active mentorship pair, when any D5-allowed viewer opens B's profile, then the mentor segment is absent and S1 has no `mentor` key *(matrix: No mentor)*
- Given B's manager, PP, or active mentorship pair changes, when the next profile request is made, then the header reflects the live values *(matrix: Live org reassignment / Live mentorship change)*

## Code Map

- `services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx:44-67` — header today is title + Access Scope Chip only; compose `ProfileHeader` in identity block.
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx:64-87` — S1 renderer duplicates manager/PP; strip relationship fields; omit body when only `displayName` remains.
- `services/frontend/src/locales/en/translation.json` — add `employeeProfile.header.*` keys.
- `services/frontend/src/types/employee-profile.ts:70-75` — `IdentitySection.mentor` already typed; no DTO change.
- `services/frontend/e2e/employee-profile-header.spec.ts` (new) — Playwright header relationship strip tests (canonical e2e path for this story).
- `services/backend/src/modules/access/identity-section.provider.ts:59-61` — mentor stub uses wrong deny-list; replace with allow-list + `ActiveMentorLookup`.
- `services/backend/src/modules/access/identity-section.provider.ts` + `__tests__/identity-section.provider.spec.ts` — D5 allow-list; ReportingLine, ProjectLine, Colleague, Self, missing `audience`.
- `services/backend/prisma/schema.prisma` — add `MentorshipPair` per Design Notes.
- `services/backend/src/modules/contracts/active-mentor-lookup.contract.ts` (new).
- `services/backend/src/modules/mentorship/` (new) — lookup service + test/seed pair helper; Epic 9 domain module (not colocated in `access` like PP assignment).
- `services/backend/src/app.module.ts` — import `MentorshipModule`.
- `services/backend/test/employee-profile.e2e-spec.ts` — extend credentialed mentor visibility and live pair change cases.

## Design Notes

**`MentorshipPair` schema (implement in Prisma task):** `id`, `mentorId`, `menteeId`, `startedAt`, `endedAt?`; inverse relations on `Employee`. Enforce at most one active pair per mentee via partial unique index on `menteeId` where `endedAt IS NULL`. Lookup: `endedAt IS NULL` only; `ORDER BY startedAt DESC` if constraint violated. Epic 9 adds closure note, consent checks, and write APIs.

**Header layout:** Follow `mockups/employee-profile.html` subtitle pattern (`Manager: Alex Kim · PP: Daniela Voss`) beneath the `h1`, inside the existing bordered header row — **note:** that mockup omits mentor even for the PP viewer; append `· Mentor: {name}` when S1 includes `mentor`. Access Scope Chip stays as today (below `h1` in current SPA, not the mockup top bar).

**Self vs D5:** `access-model.md` rule 8 and decision D5 scope mentor visibility to reporting/project line and PP only. Epic 9 story 9-1 draft mentions Self seeing mentor in the header — treat that as future work; this story does not add a Self exception.

## Verification

```bash
cd services/backend && npm run build && npm run lint
cd services/backend && npm test -- identity-section.provider
cd services/backend && npm run test:e2e -- employee-profile

cd services/frontend && npm run typecheck && npm run lint && npm run build
cd services/frontend && npm run test -- employee-profile-header
```

### Review Findings

- [x] [Review][Patch] Mentor lookup failure marks entire S1 unavailable [`services/backend/src/modules/access/identity-section.provider.ts:72`] — wrapped lookup in try/catch; S1 still returns manager/PP when mentor resolution fails.
- [x] [Review][Patch] `createActivePair` does not guard against duplicate active pairs [`services/backend/src/modules/mentorship/mentorship-pair.service.ts:22`] — rejects second active pair with `BadRequestException`.
- [x] [Review][Patch] PP audience mentor visibility lacks HTTP-level e2e [`services/backend/test/employee-profile.e2e-spec.ts`] — added PP viewer credentialed profile GET assertion.
- [x] [Review][Patch] Backend HTTP gaps for D5 edge audiences [`services/backend/test/employee-profile.e2e-spec.ts`] — added mentor lookup failure e2e plus live manager/PP reassignment tests (SharedLink/FullAccess remain C1-unreachable; covered by unit tests).
- [x] [Review][Patch] Frontend Playwright gaps for audience and header UX [`services/frontend/e2e/employee-profile-header.spec.ts`] — added Self, ProjectLine, PP segment, no-mentor, link hrefs, S1 unavailable, and S1 dedup cases.
- [x] [Review][Patch] Bootcamp seed has no `MentorshipPair` rows [`services/backend/src/prisma/seed/`] — added `seed.mentorship.ts` demo pair seeding.
- [x] [Review][Patch] `IdentitySection.mentor` typed as nullable [`services/frontend/src/types/employee-profile.ts:74`] — removed `| null` from mentor type.
- [x] [Review][Patch] Orphaned i18n keys after S1 card dedup [`services/frontend/src/locales/en/translation.json:177`] — removed unused `employeeProfile.fields.*` keys.
- [x] [Review][Patch] `endActivePairForMentee` has no unit tests [`services/backend/src/modules/mentorship/__tests__/mentorship-pair.service.spec.ts`] — added end-pair and duplicate-active-pair tests.

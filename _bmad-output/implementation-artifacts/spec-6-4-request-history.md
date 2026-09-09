---
title: 'Request History'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 2
story_key: '6-4-request-history'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-6-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-6-3-dm-reviews-and-approves-rejects-candidates.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 6.1–6.3 persist proposals and DM decisions, and request detail already lists them — but profile section S15 has no `SectionProvider`, so ReportingLine/ProjectLine/PP viewers (and shared links that include S15) get `unavailable` instead of history. Epic 6's "full proposal history on profile S15" requirement is therefore only half-delivered.

**Approach:** Register an S15 section provider in `resourcing` that reads existing `ResourcingProposal` rows for internal candidates (with parent-request context, never expected comp band), add `decidedAt` on decide transitions, and render a read-only S15 card on the employee profile (and shared-link consume path). Request-detail proposals remain the Resourcing → Requests history surface — no separate history route.

## Boundaries & Constraints

**Always:**
- `@RegisterProvider('section', 'S15')` in `resourcing` module; mirror S6 access gate (`ForbiddenException` when grant is `none`): `risks-section.provider.ts:32-35`. Self and Colleague are gated by `AccessResolver` (`S15: none` — key omitted from the profile payload); the provider gate covers any other `none` edge case.
- `buildRequestHistorySection(subjectEmployeeId)` queries `ResourcingProposal` where `candidateEmployeeId = subjectEmployeeId`, includes parent `ResourcingRequest` fields needed for display (`vacancyDetails`, `department`, `projectId`, `status`) — **never** `expectedCompBand`. Use an explicit `select`/DTO mapping so comp band cannot leak via Prisma `include`.
- S15 lists all proposal rows for the subject (status `proposed`, `approved`, or `rejected`), newest `createdAt` first. External proposals (`peopleForceCandidateUrl` only) are excluded — no employee profile exists for them. When no rows match, return `{ entries: [] }` (authorized viewers still receive S15 with data, not `unavailable`).
- Add nullable `decidedAt` on `ResourcingProposal`; set in `decide()` via `this.clock.now()` whenever status becomes `approved` or `rejected` (including reversal from `approved` → `rejected` — `decidedAt` reflects the latest decision timestamp); leave `null` while `proposed`. Expose on `ResourcingProposalEntity` and S15 entry DTOs. Pre-6.4 rows that were already decided before this migration ship with `decidedAt: null` — acceptable; no backfill migration.
- S15 entry DTO includes: proposal `id`, `requestId`, `status`, `decisionReason` (only when `rejected`; `null` on `proposed`/`approved` per 6.3), `proposedAt` (`createdAt`), `decidedAt`, request `vacancyDetails`, `department`, optional `requestStatus` (`open` | `pending_dm_review`), optional `projectName` — when `projectId` is set, use `projectId` as the display label (same S11 stub pattern in `projects-section.provider.ts:9-11`; request detail today exposes `projectId` only, not a separate name resolver).
- Self must never receive an `S15` key (`access-resolver.service.ts:84`, existing e2e `employee-profile.e2e-spec.ts:230-238`). Colleague stays `none` (`access-resolver.contract.ts:82`). ReportingLine, ProjectLine, and PP all have `S15: R`.
- Approval must not write `ProjectAssignment` — regression assertion only; decide path already compliant (`spec-6-3`).
- Add Swagger decorators for `RequestHistorySectionEntity` / entry DTOs in `resourcing.swagger.ts` (mirror other resourcing entities).

**Ask First:** If bootcamp seed has no internal candidate with a decided proposal to manually verify S15 against, flag to the human rather than inventing new seed fixtures beyond what 6.1–6.3 establish.

**Never:** Separate Resourcing history page/tab; append-only audit/event table (current proposal row is the history record; reversals overwrite status/reason per 6.3); `CLOSE_RESOURCING_REQUESTS` / `closed` request status; expected comp band in S15, shared links, or exports; S15 for external candidates; un-rejecting `rejected`; timetracker sync or S11 project surfacing (Epic 13 — verify unchanged only).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| S15_MANAGER_AFTER_DECIDE | ReportingLine viewer, internal candidate with approved/rejected proposal | Profile `sections.S15.data.entries` contains row with request context, status, `decidedAt` | — |
| S15_PP_AFTER_DECIDE | PP viewer, same candidate | Same S15 payload as ReportingLine | — |
| S15_PROJECT_LINE | ProjectLine viewer, same candidate | Same S15 payload as ReportingLine | — |
| S15_SELF_DENIED | Subject views own profile | `sections` has no `S15` key | — |
| S15_COLLEAGUE_DENIED | Colleague viewer | `sections` has no `S15` key | — |
| S15_NO_COMP_BAND | Any S15 consumer | Payload never includes `expectedCompBand` | — |
| S15_EMPTY | Authorized viewer, internal candidate with zero proposals | `sections.S15.data.entries` is `[]` (section present, not `unavailable`) | — |
| S15_PROPOSED_VISIBLE | Manager-line viewer, proposal still `proposed` | Entry shown with `status: proposed`, `decidedAt: null` | — |
| S15_SHARED_LINK_CFG | Shared link with S15 enabled, creator had `R` on S15 | Consume response includes S15 data via same provider | 403 if creator cannot create links (e.g. Colleague); 400 if an authorized creator requests sections beyond their grant; section omitted when clamped out |
| DECIDE_SETS_DECIDED_AT | DM approves or rejects | `decidedAt` populated on proposal row and returned DTO | — |
| DECIDE_REVERSAL_UPDATES_DECIDED_AT | DM reverses `approved` → `rejected` | `decidedAt` updates to reversal time; `decisionReason` set | — |
| NO_PROJECT_WRITE | DM approves internal candidate | No new `ProjectAssignment` row; S11 unchanged on profile | — |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma:574-593` — add `decidedAt DateTime?` on `ResourcingProposal`; migration (no backfill for existing decided rows)
- `services/backend/src/modules/resourcing/resourcing.service.ts:270-355` (`decide`) — set `decidedAt` via `this.clock.now()` on approve/reject/reverse; `:727-754` (`toProposalEntity`) — map `decidedAt`
- `services/backend/src/modules/resourcing/resourcing.module.ts:18-22` — register new section provider alongside dashboard providers
- `services/backend/src/modules/resourcing/request-history-section.provider.ts` — **new** `@RegisterProvider('section', 'S15')`, inject `ResourcingService` + `AccessResolver`
- `services/backend/src/modules/resourcing/entities/request-history-section.entity.ts` — **new** section + entry DTOs
- `services/backend/src/modules/resourcing/resourcing.service.ts` — **new** `buildRequestHistorySection(subjectEmployeeId)` (Prisma query + comp-band omission + `projectName` from `projectId`)
- `services/backend/src/modules/resourcing/resourcing.swagger.ts` — Swagger decorators for S15 section/entry entities
- `services/backend/src/modules/risks/risks-section.provider.ts:11-38` — provider shell pattern to copy
- `services/backend/src/modules/access/projects-section.provider.ts:9-11` — `projectId`-as-label pattern for `projectName`
- `services/backend/src/modules/access/profile-assembler.service.ts:100-113` — assembly loop; S15 already in `ALL_SECTION_IDS` (`:35`)
- `services/backend/test/employee-profile.e2e-spec.ts:230-238` — Self S15 absent (extend with positive ReportingLine/PP S15 case post-decide; Colleague absent; shared-link consume with S15 cfg)
- `services/backend/test/resourcing.e2e-spec.ts` — `decidedAt` + reversal timestamp + no-`ProjectAssignment`-on-approve regression
- `services/backend/src/modules/resourcing/__tests__/request-history-section.provider.spec.ts` + `resourcing.service.spec.ts` — matrix rows; assert query/DTO never surface `expectedCompBand`
- `services/frontend/src/types/employee-profile.ts` — `RequestHistorySection` interface + entry type
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx:42-52`, `:73-165` — add `S15` title key + renderer (today falls through to empty)
- `services/frontend/src/pages/EmployeeProfilePage/components/RequestHistorySection/` — **new** read-only card (comfortable table rows per `DESIGN.md` spacing note)
- `services/frontend/src/pages/EmployeeProfilePage/shared-link-sections.ts:14` — S15 already cfg-eligible; consume reuses profile renderer
- `services/frontend/src/locales/en/translation.json` — add `employeeProfile.sections.requestHistory` (replace generic `Section S15` fallback in `PROFILE_SECTION_TITLE_KEYS`)
- `services/frontend/src/pages/ResourcingDetailPage/components/ProposalList/ProposalList.tsx` — optional: show `decidedAt` when present (same DTO field)
- `services/frontend/src/api/hooks/useResourcingMutations.ts` + `useEmployeeProfile.ts` — on decide success, invalidate `employeeProfileQueryKey(candidateEmployeeId)` when `candidateEmployeeId` is known (alongside existing resourcing detail invalidation)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — `decidedAt` on `ResourcingProposal` (nullable, no backfill)
- [x] `services/backend/src/modules/resourcing/resourcing.service.ts` — `buildRequestHistorySection()`; set/map `decidedAt` in `decide()` + `toProposalEntity()`
- [x] `services/backend/src/modules/resourcing/request-history-section.provider.ts` + `entities/request-history-section.entity.ts` + `resourcing.swagger.ts` — S15 provider, DTOs, Swagger
- [x] `services/backend/src/modules/resourcing/resourcing.module.ts` — register provider
- [x] `services/backend/src/modules/resourcing/__tests__/request-history-section.provider.spec.ts` + `resourcing.service.spec.ts` + extend `resourcing.e2e-spec.ts` / `employee-profile.e2e-spec.ts` — matrix rows
- [x] `services/frontend/src/types/employee-profile.ts` + `profile-sections.tsx` + `RequestHistorySection/` component + `translation.json` — S15 read UI
- [x] `services/frontend/src/pages/ResourcingDetailPage/components/ProposalList/ProposalList.tsx` + `useResourcingMutations.ts` — `decidedAt` display + `employeeProfileQueryKey` invalidation

**Acceptance Criteria:**
- Given a DM approves an internal candidate on a request, when the decision is saved, then it appears immediately on Resourcing request detail and in the candidate's S15 for ReportingLine/ProjectLine/PP viewers, with no local project assignment and unchanged S11
- Given an employee was proposed and rejected with a written reason, when they view their own profile, then S15 is absent from the API response, while their manager line and PP see the rejection and reason in S15
- Given a shared link explicitly includes S15 and the creator had read access, when the recipient consumes the link, then S15 entries render without `expectedCompBand`
- Given an authorized viewer opens an internal candidate with no resourcing proposals, when the profile loads, then S15 is present with `entries: []`

### Review Findings

- [x] [Review][Patch] `projectName` resolution — request detail has no name resolver; use `projectId` as display label (S11 stub pattern) [`Boundaries`, `Code Map`]
- [x] [Review][Patch] `decidedAt` semantics — set via `Clock` on every approve/reject/reverse; reversal updates timestamp; pre-6.4 rows stay null [`Boundaries`, matrix `DECIDE_REVERSAL_UPDATES_DECIDED_AT`]
- [x] [Review][Patch] Empty S15 — return `{ entries: [] }`, not `unavailable`, for authorized viewers with no rows [`Boundaries`, matrix `S15_EMPTY`]
- [x] [Review][Patch] Access audiences — explicit ReportingLine, ProjectLine, PP, Colleague, Self matrix rows; Self/Colleague gated by resolver not provider [`Boundaries`, matrix]
- [x] [Review][Patch] Comp-band leak guard — explicit `select`/DTO mapping in query [`Boundaries`, service spec task]
- [x] [Review][Patch] Swagger for S15 entities [`Boundaries`, `Code Map`, tasks]
- [x] [Review][Patch] Profile cache invalidation — `employeeProfileQueryKey(candidateEmployeeId)` in decide mutation [`Code Map`, tasks]
- [x] [Review][Patch] i18n key — `employeeProfile.sections.requestHistory` instead of generic fallback [`Code Map`]
- [x] [Review][Patch] FR-32 reference — replaced with epic-6-context wording (FR numbering is CDS elsewhere) [`Intent`]
- [x] [Review][Patch] `requestStatus` on entry DTO — include parent request status for display context [`Boundaries`]
- [x] [Review][Patch] Shared-link consume test — extend `employee-profile.e2e-spec.ts` for S15 cfg/clamp [`Code Map`, matrix `S15_SHARED_LINK_CFG`]
- [x] [Review][Patch] Service-level unit test for comp-band omission [`Code Map`, tasks]

### Code Review Findings (2026-09-09)

- [x] [Review][Patch] ProjectLine S15 e2e — assert `dmAgent` ProjectLine profile in post-decide test [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] `S15_PROPOSED_VISIBLE` e2e — proposed row with `decidedAt: null` before submit/decide [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] AC2 PP coverage — reversal test asserts PP sees `decisionReason` in S15 [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] AC1 S11 regression — compare `sections.S11` before/after approve in setup helper [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] Shared-link Colleague denied — Colleague creator cannot create links (403) [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] Profile cache on propose — invalidate `employeeProfileQueryKey` in `useCreateResourcingProposal` [`useResourcingMutations.ts`]
- [x] [Review][Patch] S15 UI — render `requestStatus`; guard malformed `entries`/dates/status labels [`RequestHistorySection.tsx`]
- [x] [Review][Patch] Provider gate — treat missing `sections.S15` as denied [`request-history-section.provider.ts`]
- [x] [Review][Patch] Service unit tests — proposed mapping + `orderBy` newest-first [`resourcing.service.spec.ts`]
- [x] [Review][Patch] E2e entry DTO fields — assert `proposedAt` and `requestStatus` on S15 entries [`resourcing.e2e-spec.ts`]
- [x] [Review][Patch] Matrix correction — shared-link over-grant is HTTP 400, not 403 [`I/O Matrix`]
- [x] [Review][Defer] S15 OpenAPI on profile endpoint — section entities documented but profile assembly stays generic envelope like other sections — deferred, pre-existing pattern
- [x] [Review][Defer] Frontend Playwright for S15 card — no component-test harness in frontend — deferred, pre-existing
- [x] [Review][Defer] Bootcamp seed manual S15 fixture — no decided-proposal seed row; flag at manual QA time per Ask First — deferred, accepted gap
- [x] [Review][Defer] FixedClock reversal timestamp distinctness — e2e uses single instant; unit tests assert `decidedAt` write on each decide — deferred, covered at unit layer

## Design Notes

History source of truth is the mutable `ResourcingProposal` row (6.3 semantics) — not an event log. Reversal to `rejected` updates the same row; S15 shows the current final state plus `proposedAt`/`decidedAt` (with `decidedAt` reflecting the latest decision). Request detail's embedded `proposals[]` is the Resourcing-side history UI; 6.4 does not add a second surface.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- resourcing.e2e-spec.ts employee-profile.e2e-spec.ts` — expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` — expected: pass

**Manual checks (if no CLI):**
- Bootcamp manager opens an internal candidate's profile after a DM decision; S15 shows request title/department, status, rejection reason when applicable; candidate's own profile has no S15 section

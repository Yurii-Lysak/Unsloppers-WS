---
title: 'Fulfil a Request with Internal or External Candidates'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '8bf7efe0b36cc9e81a6e406b37fe71fa2176ca64'
baseline_revision: '8bf7efe0b36cc9e81a6e406b37fe71fa2176ca64'
story_key: '6-2-fulfil-a-request-with-internal-or-external-candidates'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-6-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-6-1-create-a-resourcing-request.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/SPEC.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<intent-contract>

## Intent

**Problem:** Story 6.1 created `ResourcingRequest` create/list for DMs/PMs, but UMs cannot see routed requests, attach internal or external candidates, or submit proposals for DM review — `ResourcingProposal` persistence, C12 UM routing, fulfil APIs, and fulfilment UI are absent.

**Approach:** Add minimal C12 (`Department` + `DepartmentDirectory` contract) for live UM routing by department name; add `ResourcingProposal` model and fulfil endpoints gated by `fulfil_resourcing_requests`; extend request status to `pending_dm_review` on submit; build a Resourcing detail page where UMs attach internal unit employees or external PeopleForce-link candidates, then submit — frontend orchestrates shared-link creation for each internal candidate naming the reviewing DM as recipient.

## Boundaries & Constraints

**Always:**
- **C12 minimal (access module):** `Department` model (`id`, `name` unique, optional `parentId`, `managerId` → Employee); `DepartmentDirectory` contract token in `contracts/` with `getDepartmentByName(name)`, `getManagedDepartmentIds(employeeId)` (direct `managerId` rows plus descendant ids); implementation in `access/`; seed departments from distinct current `DepartmentHistory.value` strings in bootcamp population, assigning each department's `managerId` to the bootcamp UM employee for that unit when resolvable.
- **Routing:** `ResourcingRequest.department` remains the string persisted in 6.1; UM assignment resolves live via `getDepartmentByName(request.department).managerId === viewerEmployeeId` (re-resolved every list/detail read, never pinned).
- **Comp band visibility:** extend 6.1 rule — include `expectedCompBand` when viewer is author, DM on request's `projectId`, **or** viewer is the live routed UM for `request.department`.
- **Proposal model (`ResourcingProposal`):** `id`, `requestId` FK, `proposedById` FK (UM), `candidateEmployeeId` nullable FK (internal), `peopleForceCandidateId` optional string max 128, `peopleForceCandidateUrl` optional string max 2000, `status` enum (`proposed` only in 6.2), `createdAt`. Exactly one of internal (`candidateEmployeeId`) or external (`peopleForceCandidateUrl` required; `peopleForceCandidateId` optional) per row.
- **Request status enum:** add `pending_dm_review`; reachable only via explicit submit while status is `open` and at least one `proposed` proposal exists; `open` requests accept new proposals.
- **Reviewing DM resolution (for shared-link recipient and future 6.3):** if `projectId` set → active `ProjectAssignment.dmId` for that project; else if author holds DM functional role → `authorId`; else if author is PM → `author.managerId` (reports-to DM). Store resolved id on submit in a new nullable `reviewingDmId` column on `ResourcingRequest` (set once at submit, immutable in 6.2).
- **Fulfil auth:** `FULFIL_RESOURCING_REQUESTS` via C8 for assigned list, detail read, proposal create, submit. Create/list from 6.1 remain on `CREATE_RESOURCING_REQUESTS`.
- **Internal candidate guard:** `candidateEmployeeId` must belong to UM's managed department tree — employee's current `DepartmentHistory` row (latest `effectiveFrom`) department **name** must match a `Department` whose id is in `getManagedDepartmentIds(proposerId)` or is a descendant-managed department name.
- **External candidate guard:** `peopleForceCandidateUrl` required (trimmed, valid `http`/`https` URL, max 2000); `peopleForceCandidateId` optional trimmed max 128; C5 lookup optional when id present — never required for submit.
- **Endpoints** (under `/api/v1/resourcing/requests`): `GET /assigned` (UM routed open requests), `GET /:id` (detail + proposals, role-scoped), `POST /:id/proposals` (attach one candidate), `POST /:id/submit` (transition to `pending_dm_review`, set `reviewingDmId`). Existing `GET /` and `POST /` unchanged for create permission holders.
- **Shared link (frontend-orchestrated, AD-11):** on successful submit, for each internal proposal sequentially call `POST /employees/:candidateEmployeeId/shared-links` with `recipientEmployeeId = reviewingDmId` and sections `['S1','S4','S11','S12','S5']` (CV+certs). Backend resourcing module does not import `access`.
- **Frontend:** `/resourcing/:requestId` detail page; list rows navigate to detail; nav/route visible when viewer has `create_resourcing_requests` **or** `fulfil_resourcing_requests`; add `route-permissions.ts` entries for both; UM sees assigned list (or assigned section on same page when fulfil-only).
- **Never write `ProjectAssignment`** from resourcing; no approve/reject, no S15 provider, no dashboard providers, no PeopleForce API integration, no proposal status beyond `proposed`, no headcount consumption, no departed-UM/DM backstop (C13/D18).

**Block If:** Bootcamp seed cannot yield at least one `Department` row with a resolvable UM `managerId` — HALT rather than inventing department names.

**Never:** DM approve/reject UI (6.3); request history/S15 (6.4); auto-close on headcount; PP resourcing nav; comp band in shared links/exports; backend-side shared-link generation inside resourcing module.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| HAPPY_INTERNAL | UM routed to request, valid unit employee, submit | Proposal `proposed`, status `pending_dm_review`, shared links created for DM | — |
| HAPPY_EXTERNAL_LINK | UM routed, valid `peopleForceCandidateUrl`, submit | External proposal stored, status `pending_dm_review` | — |
| FORBIDDEN_FULFIL | Viewer lacks `fulfil_resourcing_requests` on assigned routes | — | 403 |
| NOT_ROUTED | UM not current manager of request department | — | 403 on detail/propose/submit |
| INTERNAL_OUT_OF_UNIT | Candidate outside UM managed departments | — | 400 |
| EXTERNAL_NO_URL | External proposal without URL | — | 400 |
| SUBMIT_EMPTY | Submit with zero proposals | — | 400 |
| SUBMIT_NOT_OPEN | Submit when status not `open` | — | 409 |
| CREATE_LIST_UNCHANGED | DM/PM still use existing POST/GET list | 6.1 behavior preserved | — |

</intent-contract>

## Code Map

- `services/backend/prisma/schema.prisma` — add `Department`, `ResourcingProposal`, extend `ResourcingRequestStatus`, `reviewingDmId` on `ResourcingRequest`
- `services/backend/src/modules/contracts/department-directory.contract.ts` — new C12 token + DTOs
- `services/backend/src/modules/access/department-directory.service.ts` — C12 implementation
- `services/backend/src/modules/access/access.module.ts` — bind C12 provider
- `services/backend/src/prisma/seed/seed.departments.ts` — seed from bootcamp department history values
- `services/backend/src/modules/resourcing/resourcing.service.ts` — routing, proposals, submit, comp-band UM rule
- `services/backend/src/modules/resourcing/resourcing.controller.ts` — fulfil routes + split auth asserts
- `services/backend/src/modules/resourcing/dto/` — proposal + detail DTOs
- `services/backend/src/modules/resourcing/entities/` — detail read entity with proposals
- `services/backend/src/modules/access/shared-link.service.ts:106-156` — `createLink` reuse from frontend
- `services/backend/src/modules/access/dto/create-shared-link.dto.ts` — shared-link payload shape
- `services/backend/src/modules/contracts/external-identity-mapping.contract.ts` — optional C5 on external id
- `services/backend/src/modules/resourcing/__tests__/resourcing.service.spec.ts` — routing + proposal unit tests
- `services/backend/test/resourcing.e2e-spec.ts` — matrix rows (new file)
- `services/frontend/src/pages/Resourcing/ResourcingPage.tsx` — row navigation to detail
- `services/frontend/src/pages/ResourcingDetailPage/` — fulfilment UI (new)
- `services/frontend/src/api/services/resourcing.service.ts` — assigned/detail/propose/submit
- `services/frontend/src/api/hooks/useResourcing.ts` + `useResourcingMutations.ts` — query/mutation keys
- `services/frontend/src/hooks/data/useResourcingData.ts` — fulfil hooks
- `services/frontend/src/types/permissions.ts` + `usePermissionsData.ts` + `useSideMenu.ts` + `route-permissions.ts` — fulfil permission gate
- `services/frontend/src/pages/CampaignDetailPage/` — detail page layout pattern
- `services/frontend/src/components/AudienceBuilder/` — internal employee picker pattern
- `services/frontend/src/api/services/shared-link.service.ts` — post-submit link creation

## Tasks & Acceptance

**Execution:**
- `services/backend/prisma/schema.prisma` — Department, ResourcingProposal, status enum, reviewingDmId; migration
- `services/backend/src/modules/contracts/department-directory.contract.ts` — C12 contract
- `services/backend/src/modules/access/department-directory.service.ts` + module wiring — C12 impl
- `services/backend/src/prisma/seed/seed.departments.ts` + `seed.ts` — bootcamp department seed
- `services/backend/src/modules/resourcing/` — fulfil service methods, DTOs, controller routes, swagger
- `services/backend/test/resourcing.e2e-spec.ts` — I/O matrix + AC scenarios
- `services/backend/src/modules/resourcing/__tests__/resourcing.service.spec.ts` — unit coverage for routing/proposals
- `services/frontend/src/pages/ResourcingDetailPage/` — candidate attach + submit UI
- `services/frontend/src/pages/Resourcing/ResourcingPage.tsx` — navigable rows + UM assigned section
- `services/frontend/src/api/` + `hooks/data/useResourcingData.ts` — fulfil API layer
- `services/frontend/src/types/resourcing.ts` — proposal + status types
- `services/frontend/src/router/index.tsx` + permissions/side menu/route-permissions — fulfil access
- `services/frontend/src/locales/en/translation.json` — detail/candidate/submit i18n keys

**Acceptance Criteria:**
- Given a resourcing request routed to me as UM, when I attach a specialist from my unit and submit, then the candidate is recorded as proposed and the request moves to `pending_dm_review`
- Given PeopleForce integration is not live, when I attach an external candidate using an outbound PeopleForce link, then the candidate is recorded with that link and the request can be submitted for DM review
- Given a UM without `create_resourcing_requests`, when they open Resourcing, then they see assigned routed requests and can fulfil but cannot create requests
- Given a DM/PM with create permission, when they use the existing list/create flow, then 6.1 behavior is unchanged

## Design Notes

Reviewing DM for unattached PM requests uses `author.managerId` as the pragmatic bootcamp default until project-based routing applies. Shared-link sections follow CAP-6 SPEC: S1, S4, S11, S12, S5 (CV+certs); never S2/S3/S7/S8 on auto-generated links.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- resourcing.e2e-spec.ts` — expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` — expected: pass

**Manual checks (if no CLI):**
- Bootcamp UM account sees routed open request, attaches internal + external candidates, submits; status shows pending DM review

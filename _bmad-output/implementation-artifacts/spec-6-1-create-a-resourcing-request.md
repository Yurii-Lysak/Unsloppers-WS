---
title: 'Create a Resourcing Request'
type: 'feature'
created: '2026-09-08'
status: 'in-review'
review_loop_iteration: 0
baseline_commit: '2f04487efde50b9c020c220221c64c28d3fa0cc8'
story_key: '6-1-create-a-resourcing-request'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-6-context.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<intent-contract>

## Intent

**Problem:** CAP-6 resourcing has no `ResourcingRequest` model, `resourcing` module, or UI — DMs/PMs cannot create staffing requests; unattached (no-project) requests and DM/PM list scoping from Story 1.2 are unimplemented.

**Approach:** Add `ResourcingRequest` persistence, a `resourcing` Nest module with `POST/GET /resourcing/requests` gated by `CREATE_RESOURCING_REQUESTS`, list scoping (PM = own; DM = own + PM requests on DM-managed projects), then a Resourcing list page with create dialog mirroring Campaigns (rhf + zod).

## Boundaries & Constraints

**Always:**
- **Model** (`ResourcingRequest`): `id`, `authorId` (FK Employee), `vacancyDetails` (trimmed, 1–5000), `expectedCompBand` (trimmed, 1–200), `duration` (trimmed, 1–200), `workload` (trimmed, 1–200), `headcount` (int, 1–99, default 1), `department` (trimmed, 1–200, required — matches built-in department string values), `projectId` (optional string, max 128, opaque timetracker id), `status` enum (`open` only reachable in 6.1), `createdAt`, `updatedAt`. FK `authorId` → `Employee` `onDelete: Restrict`. Index `authorId`, `projectId`.
- **Create payload:** `{ vacancyDetails, expectedCompBand, duration, workload, headcount?, department, projectId? }` — `projectId` omitted/null is valid; no blocking validation on project.
- **Create auth:** `PermissionChecker.hasPermission(viewerUserId, CREATE_RESOURCING_REQUESTS)` only — no section gate, no manager/PP widening (unlike campaigns).
- **List auth:** same permission as create for 6.1 scope.
- **List scoping:** PM (and any creator without DM assignments): `authorId === viewerEmployeeId`. DM additionally sees requests where `authorId` is `pmId` on any `ProjectAssignment` row with `dmId === viewerEmployeeId` (active rows: `endDate IS NULL OR endDate >= today`).
- **Comp band visibility in read DTO:** include `expectedCompBand` when viewer is the author OR viewer is `dmId` on a `ProjectAssignment` for the request's `projectId` (when set); otherwise omit the field from JSON (do not return null placeholder).
- **Routes:** `POST /resourcing/requests`, `GET /resourcing/requests` under global `/api/v1` prefix.
- **Module imports:** `ResourcingModule` depends only on global `ContractsModule`/`PrismaModule` patterns — inject `PermissionChecker`, `ProjectAssignment` contract, `PrismaService`, `CurrentUserProvider`; never import other feature modules.
- **Frontend:** `pages/Resourcing/` list + create dialog; nav entry when `/permissions/me` includes `create_resourcing_requests`; route `/resourcing`; i18n keys under `resourcing.*`.
- **Never write `ProjectAssignment`** from resourcing.

**Block If:** Department-to-UM routing contract (C12) is required beyond storing the department string — defer UM routing resolution to Story 6.2; 6.1 only persists `department`.

**Never:** `ResourcingProposal` model, fulfilment/approval flows, S15 provider, shared-link auto-generation, PeopleForce integration, dashboard providers, comp band in exports/shared links, PP resourcing nav.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| HAPPY_PATH_UNATTACHED | DM with permission, valid body, no `projectId` | 201, status `open`, request in GET list | — |
| HAPPY_PATH_ATTACHED | PM with permission, valid body + known `projectId` from assignment | 201, `projectId` stored | — |
| FORBIDDEN_CREATE | Employee without `create_resourcing_requests` | — | 403 |
| VALIDATION_MISSING_DEPT | Empty/missing `department` | — | 400 |
| PM_LIST_SCOPE | PM A and PM B each create requests | PM A GET list shows only A's rows | — |
| DM_SEES_PM_REQUEST | DM on project X, PM on project X creates request with that `projectId` | DM GET includes PM's request | — |
| DM_INVISIBLE_OTHER_PM | PM on project Y (DM not responsible) creates request | DM GET excludes it | — |

</intent-contract>

## Code Map

- `services/backend/prisma/schema.prisma` — add `ResourcingRequestStatus`, `ResourcingRequest`; new migration
- `services/backend/src/modules/campaigns/campaigns.controller.ts:156-177` — permission-only auth + employee resolve pattern
- `services/backend/src/modules/campaigns/campaigns.module.ts` — minimal module shape
- `services/backend/src/modules/contracts/permission-keys.ts:11` — `CREATE_RESOURCING_REQUESTS` already defined
- `services/backend/src/prisma/seed/seed.functional-roles.ts:40,50` — DM/PM already granted key
- `services/backend/src/modules/access/project-assignment.service.ts:45-49` — `listByProject`; query `dmId`/`pmId` for list scoping
- `services/backend/src/modules/contracts/project-assignment.contract.ts` — read-only C3
- `services/backend/src/app.module.ts` — register `ResourcingModule`
- `services/backend/test/campaigns.e2e-spec.ts:80-95` — functional-role grant helper pattern for e2e
- `services/frontend/src/pages/CampaignsPage/` — list + dialog + form stack to mirror
- `services/frontend/src/router/index.tsx` — add `/resourcing` route
- `services/frontend/src/components/SideMenu/hooks/useSideMenu.ts` — nav flag from permissions
- `services/frontend/src/types/permissions.ts` — extend `PERMISSION_KEYS` + hook

## Tasks & Acceptance

**Execution:**
- `services/backend/prisma/schema.prisma` — add model + enum; run migration
- `services/backend/src/modules/resourcing/` — module, controller, service, DTOs, entity mapper, swagger
- `services/backend/src/app.module.ts` — import `ResourcingModule`
- `services/backend/test/resourcing.e2e-spec.ts` — matrix rows + AC scenarios
- `services/backend/src/modules/resourcing/__tests__/resourcing.service.spec.ts` — list scoping unit tests
- `services/frontend/src/pages/Resourcing/` — page, form dialog, schema, hooks
- `services/frontend/src/api/services/resourcing.service.ts` + `api/hooks/useResourcingMutations.ts` + `hooks/data/useResourcingData.ts`
- `services/frontend/src/router/index.tsx` + SideMenu + i18n + permissions types

**Acceptance Criteria:**
- Given a DM with `create_resourcing_requests`, when they POST without `projectId`, then 201 and GET list includes the unattached request with status `open`
- Given a DM responsible for Project X and a PM of Project X who created a request for Project X, when the DM GETs `/resourcing/requests`, then the PM's request appears alongside the DM's own
- Given a PM who created a request, when another PM GETs the list, then the first PM's request is absent
- Given a viewer without `create_resourcing_requests`, when they POST, then 403
- Given the Resourcing page, when a permitted user creates a request via the dialog, then the new row appears in the list without page reload beyond query invalidation

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- resourcing.e2e-spec.ts` — expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` — expected: pass

**Manual checks (if no CLI):**
- Resourcing nav visible for bootcamp DM/PM seed accounts; hidden for PP

## Manual Test Paths

**Create a Resourcing Request** — DMs and PMs can raise staffing requests with or without a project link.

1. Sign in as a Delivery Manager or Project Manager account that has the create resourcing requests permission.
2. Open **Resourcing** from the sidebar (the item appears only for permitted roles).
3. Click **New request**, fill in vacancy details, expected compensation band, duration, workload, department, and optionally pick a project, then save.
4. Confirm the new request appears in the list with status open; if you skipped the project, the row shows no project reference.
5. Sign in as the Delivery Manager responsible for that project’s PM, open **Resourcing**, and confirm you see both your own requests and the PM’s request for the shared project.
6. Sign in as a different PM who did not create the request, open **Resourcing**, and confirm the other PM’s request is not listed.
7. Sign in as a People Partner or another role without create permission and confirm **Resourcing** is hidden and the create API returns forbidden.

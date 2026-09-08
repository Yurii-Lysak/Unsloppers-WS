---
title: 'Create a Form Campaign'
type: 'feature'
created: '2026-09-04'
status: 'done'
review_loop_iteration: 0
baseline_commit: '39b1a35f8918649aea88d158abfdde5ca600cb36' # services/backend HEAD on feature/10-1-create-a-form-campaign; services/frontend HEAD at story start: 6f642d0df1cfce5acc97ae95010971ce76651bbc
story_key: '10-1-create-a-form-campaign'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-10-context.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/interface-contracts.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/docs/PRD_parallel_delivery_plan.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** No `FormCampaign` model or `campaigns` backend module exists yet. The frontend already has a `/campaigns` route, nav item, and permission gate wired to `CREATE_FORM_CAMPAIGNS` — a stub page today. But that functional-permission check today only recognizes an explicit role grant, not "is a manager or PP." So PPs/managers without a granted role can't actually reach a feature the PRD says they get by default.

**Approach:** Add the `FormCampaign` model (draft-only) and a `campaigns` module (create/list/get/update-while-draft), extend `PermissionCheckerService` so `CREATE_FORM_CAMPAIGNS` is also satisfied by manager/PP status, and replace `CampaignsStubPage` with a real list + create + edit-draft UI using the existing react-hook-form/zod pattern.

## Boundaries & Constraints

**Always:**
- Fields: `title` (≤200), `description` (≤500, "short description"), `purpose` (≤2000), `link` (required URL, `IsUrl({require_protocol:true})`, ≤2048), `dueDate` (ISO calendar date `YYYY-MM-DD`, same validator shape as `action-items/dto/is-calendar-date.validator.ts` — no past/future constraint, mirroring `ActionItem.dueDate`). All five fields are required — none is optional.
- `status` defaults `draft`; `PATCH` on any of the five fields (title, description, purpose, link, dueDate) succeeds only while `status === 'draft'` (409 otherwise) — a no-op guard today since nothing sets `active` until Story 10.3, but required so 10.3 doesn't have to add it later.
- Create auth: `PermissionChecker.hasPermission(viewerId, CREATE_FORM_CAMPAIGNS)`, widened per Design Notes so PP/manager status also satisfies it, per PRD "available to PP and managers by default, plus any functional role granted the permission."
- List/Get/Patch are scoped to `creatorId === viewer's employeeId`; a campaign that exists but isn't the viewer's → 404 (no existence leak).
- Add the FK the schema already anticipates: `ActionItem.campaign` relation to `FormCampaign` (`schema.prisma:432`'s own comment). Do not populate `campaignId` anywhere — that's Story 10.3/4.4.
- Frontend: replace `CampaignsStubPage` with a list page (empty state "No campaigns yet." + "New campaign" action) and a create/edit form; reuse `AdminRolesPage`'s react-hook-form + zod pattern and `risk.service.ts` / `useRiskMutations.ts`'s API+cache-invalidation pattern.

**Ask First:**
- Widening `PermissionCheckerService` (owned by Epic 1's `access` module) from within this Epic 10 story is a cross-epic edit — confirm this is acceptable before merging, versus adding the manager/PP check as a campaigns-module-local helper instead.

**Never:**
- Audience building, the filter engine, or saved views (Story 10.2).
- Activation, `active` status transition, or any action-item generation (Story 10.3).
- The completion-tracking table (Story 10.4).
- A delete endpoint, or any dueDate temporal validation beyond valid ISO calendar-date format.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PP creates | PP, valid fields, POST | 201, draft campaign | N/A |
| Manager, no role grant | `managerId` points to viewer on ≥1 employee, POST | 201 | N/A |
| Role-granted, no reports | Holds `CREATE_FORM_CAMPAIGNS` via functional role, 0 reports, POST | 201 | N/A |
| No manager/PP/permission | Plain colleague, POST | — | 403 |
| Missing link | POST without `link` | — | 400 |
| Edit while draft | Creator PATCHes `title` | 200, updated | N/A |
| Edit someone else's | Non-creator PATCHes another's campaign | — | 404 |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma:432-458` -- `ActionItem.campaignId` comment marks this as the trigger to add `FormCampaign`; add model + enum + the FK here.
- `services/backend/src/modules/risks/*` -- module-shape template (module/controller/service/dto/entities/swagger/`__tests__`) to copy for `campaigns`.
- `services/backend/src/modules/action-items/dto/{create-action-item.dto.ts,is-calendar-date.validator.ts}` -- reuse pattern for `link`/`dueDate` validators (define local campaigns equivalents, do not import cross-module). Campaign fields are required where action-items' are optional: drop the empty-string-to-undefined `link` transform, and apply the same trim + `@IsNotEmpty()` used on `title` to `description`/`purpose` too.
- `services/backend/src/modules/access/permission-checker.service.ts:24-56` -- `getGrantedPermissions`; add the manager/PP-derived widening here, scoped to `CREATE_FORM_CAMPAIGNS`.
- `services/backend/src/app.module.ts` -- register `CampaignsModule`.
- `services/frontend/src/router/index.tsx:58-60` -- swap `CampaignsStubPage` for the real page (permission mapping in `route-permissions.ts` and nav flag in `SideMenu/hooks/useSideMenu.ts` are already correct, no change needed).
- `services/frontend/src/pages/AdminRolesPage/{schemas/role-form.schema.ts,hooks/useRoleForm.ts}` -- react-hook-form + zod pattern to copy for the campaign form.
- `services/frontend/src/api/services/risk.service.ts` + `api/hooks/useRiskMutations.ts` -- API client + mutation-hook pattern to copy.

## Tasks & Acceptance

**Execution:** (complete in order below -- later tasks depend on earlier ones, e.g. schema before service, service before its tests)
- [x] `services/backend/prisma/schema.prisma` -- `FormCampaignStatus` enum (`draft`, `active` -- only `draft` is reachable this story; `active` exists now so Story 10.3 doesn't need a second migration), `FormCampaign` model, `ActionItem.campaign` FK (`onDelete: Restrict`, matching `ActionItem.assignee`/`.author`), `Employee.createdCampaigns FormCampaign[]` back-relation, migration
- [x] `services/backend/src/modules/campaigns/{campaigns.module.ts,campaigns.controller.ts,campaigns.service.ts,dto/,entities/,campaigns.swagger.ts}` -- POST/GET/GET-one/PATCH
- [x] `services/backend/src/modules/access/permission-checker.service.ts` -- manager/PP widening for `CREATE_FORM_CAMPAIGNS`
- [x] `services/backend/src/app.module.ts` -- register `CampaignsModule`
- [x] `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts` + `access/__tests__/permission-checker.service.spec.ts` -- unit tests for auth + validation + widening
- [x] `services/backend/test/campaigns.e2e-spec.ts` -- matrix rows above
- [x] `services/frontend/src/pages/CampaignsPage/{CampaignsPage.tsx,components/CampaignForm/,schemas/campaign-form.schema.ts,hooks/}` -- list + create + edit-draft UI, replaces stub
- [x] `services/frontend/src/api/services/campaign.service.ts` + `api/hooks/useCampaignMutations.ts` + `hooks/data/useCampaignsData.ts` -- write path + cache invalidation
- [x] `services/frontend/src/router/index.tsx` -- swap stub for `CampaignsPage`

**Acceptance Criteria:**
- Given I am a PP, when I create a campaign with title, description, purpose, an external form link, and a due date, then it saves in `draft` state with no action items generated and no employees notified
- Given I manage at least one direct report but hold no functional-role grant, when I POST a valid campaign, then creation succeeds (manager default access, not role-grant-based)
- Given I hold a functional role granting `create form campaigns` but manage no one and am no one's PP, when I POST a valid campaign, then creation still succeeds
- Given a campaign is in `draft`, when its creator edits any of title/description/purpose/link/dueDate, then the change is saved and the campaign remains `draft`

### Review Findings

- [x] [Review][Decision] Cross-epic `PermissionCheckerService` widening — Approved 2026-09-04: cross-epic edit in Epic 1 `access` module is acceptable per Design Notes intent.

- [x] [Review][Patch] Frontend `dueDate` validation weaker than backend [`services/frontend/src/pages/CampaignsPage/schemas/campaign-form.schema.ts:42`]
- [x] [Review][Patch] Modal can dismiss during in-flight save [`services/frontend/src/pages/CampaignsPage/components/CampaignFormDialog/CampaignFormDialog.tsx:26`]
- [x] [Review][Patch] Stale mutation error shown after dialog reopen [`services/frontend/src/pages/CampaignsPage/hooks/useCampaignForm.ts:55`]
- [x] [Review][Patch] Duplicate save-failure feedback (toast + inline `rootError`) [`services/frontend/src/api/hooks/useCampaignMutations.ts:19`]
- [x] [Review][Patch] No Playwright e2e for campaigns create/list/edit UI [`services/frontend/src/pages/CampaignsPage/CampaignsPage.tsx`]
- [x] [Review][Patch] List-ordering e2e test name overclaims — only asserts creator scoping, not `createdAt desc` [`services/backend/test/campaigns.e2e-spec.ts:384`]
- [x] [Review][Patch] Save-failure copy does not match epic UX ("Couldn't save. Try again.") [`services/frontend/src/locales/en/translation.json`]

- [x] [Review][Defer] `npm run test:e2e -- campaigns` exits non-zero due to pre-existing global teardown access-matrix gaps — deferred, pre-existing
- [x] [Review][Defer] `PermissionCheckerService` runs three extra `count` queries for every `getGrantedPermissions` caller without short-circuit — deferred, acceptable design tradeoff for Story 10.1
- [x] [Review][Defer] Non-draft list rows disabled with no explanatory copy — deferred, Story 10.3 when `active` campaigns are common
- [x] [Review][Defer] `title`/`link` lack DB-level length constraints (app validation only) — deferred, low risk for draft-only story

## Spec Change Log

## Design Notes

`PermissionCheckerService.getGrantedPermissions` currently derives grants purely from `FunctionalRoleAssignment` rows. This story adds a second, narrow source for exactly one key: if the viewer manages ≥1 employee (reporting line or PM/DM on a `ProjectAssignment`) or is PP for ≥1 employee, `CREATE_FORM_CAMPAIGNS` is included in the returned set even with zero role assignments. Because `hasPermission` and the `/me` "my permissions" endpoint both read from this one function, both the create-authorization check and the frontend's existing `canCreateFormCampaigns` nav/route gate pick up the correct behavior. No frontend changes are needed.

Widening scope, made explicit to avoid divergent implementations:
- "Manages" means direct reports only (`Employee.managerId === viewer.id`), not the full transitive `ReportingLine` closure `AccessResolver` uses elsewhere -- a skip-level manager qualifies only if also a direct manager of someone, or PP of someone.
- `ProjectAssignment` PM/DM rows only count while active: `endDate IS NULL OR endDate >= today`. An ended assignment no longer grants the permission.
- Direct-report/PP-assignee `employmentStatus` is not filtered -- a report or PP-assignee who has left still counts, consistent with how `FunctionalRoleAssignment`-derived grants are never auto-revoked either.
- A malformed (non-UUID) `campaignId` path param on GET-one/PATCH returns Nest's standard `ParseUUIDPipe` 400, distinct from the ownership-scoped 404 for a well-formed but non-existent/non-owned id.
- PATCH accepts a partial payload (`PartialType(CreateCampaignDto)`): any subset of the five editable fields, not all five on every call.
- GET-list is unpaginated for this story, ordered `createdAt desc`; revisit once creators accumulate enough campaigns for this to matter.

## Verification

**Commands:**
- `cd services/backend && npm run db:migrate && npm test && npm run test:e2e -- campaigns` -- expected: all pass
- `cd services/frontend && npm run lint && npm run build` -- expected: pass

**Manual checks:**
- Log in as a manager with no functional roles; confirm the Campaigns nav item appears and creation succeeds.
- Log in as a colleague with no reports, no PP assignments, and no functional roles; confirm the nav item is absent and a direct POST returns 403.

## Suggested Review Order

**Data model**

- New entity: draft-only campaign, `active` reserved so Story 10.3 needs no second migration
  [`schema.prisma:472`](../../services/backend/prisma/schema.prisma#L472)

- Completes `ActionItem.campaignId`'s FK, closing a comment left by Story 4.4
  [`schema.prisma:456`](../../services/backend/prisma/schema.prisma#L456)

**Permission widening (the one real design decision)**

- Manager/PP status now satisfies `CREATE_FORM_CAMPAIGNS` without a role grant
  [`permission-checker.service.ts:82`](../../services/backend/src/modules/access/permission-checker.service.ts#L82)

- Widening folds into the one function every permission check already reads
  [`permission-checker.service.ts:60`](../../services/backend/src/modules/access/permission-checker.service.ts#L60)

**Campaign service — write-path safety**

- Atomic draft-only PATCH guard, closing a TOCTOU race found in review
  [`campaigns.service.ts:79`](../../services/backend/src/modules/campaigns/campaigns.service.ts#L79)

- Ownership check underlies every read/write path — no permission re-check by design
  [`campaigns.service.ts:98`](../../services/backend/src/modules/campaigns/campaigns.service.ts#L98)

- Link accepted only for http/https, closing a second review finding
  [`campaign-input.ts:71`](../../services/backend/src/modules/campaigns/campaign-input.ts#L71)

- Same allow-list enforced at the DTO boundary
  [`create-campaign.dto.ts:35`](../../services/backend/src/modules/campaigns/dto/create-campaign.dto.ts#L35)

**API surface**

- Four routes: create/list/get/patch, no delete — matches frozen scope
  [`campaigns.controller.ts:30`](../../services/backend/src/modules/campaigns/campaigns.controller.ts#L30)

**Frontend — campaign UI**

- Row action only opens the edit dialog for draft campaigns
  [`CampaignsPage.tsx:60`](../../services/frontend/src/pages/CampaignsPage/CampaignsPage.tsx#L60)

- Same http/https allow-list mirrored client-side
  [`campaign-form.schema.ts:7`](../../services/frontend/src/pages/CampaignsPage/schemas/campaign-form.schema.ts#L7)

- Cancel now disabled mid-submit, avoiding a stale-dialog race
  [`CampaignFormDialog.tsx:36`](../../services/frontend/src/pages/CampaignsPage/components/CampaignFormDialog/CampaignFormDialog.tsx#L36)

- Route swapped from the stub page onto the real feature
  [`router/index.tsx:59`](../../services/frontend/src/router/index.tsx#L59)

**Tests**

- Full I/O matrix plus the review's new no-employee-record 403 case
  [`campaigns.e2e-spec.ts:215`](../../services/backend/test/campaigns.e2e-spec.ts#L215)

- Fixture fix for Story 4.4's own tests, now that a real FK exists
  [`action-items.e2e-spec.ts:57`](../../services/backend/test/action-items.e2e-spec.ts#L57)

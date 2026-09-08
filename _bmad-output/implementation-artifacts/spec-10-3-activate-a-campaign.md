---
title: 'Activate a Campaign'
type: 'feature'
created: '2026-09-07'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'c4306b6cd2aba390644ed154156dab261daf6790' # services/backend HEAD on feature/10-3-activate-a-campaign; services/frontend HEAD at story start: c6c620d55b505ce0c166e8336f5ce64c19f17df1
story_key: '10-3-activate-a-campaign'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-10-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-10-2-build-and-freeze-campaign-audience.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-4-auto-generate-action-item-on-form-campaign-activation.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 10.2 gave campaigns a resolvable draft audience and Story 4.4 gave C6 a bulk action-item path, but nothing connects them — there is no way to turn a draft campaign into an active one, so recipients never receive anything.

**Approach:** Add a creator-scoped `POST :campaignId/activate` endpoint that, in one transaction, atomically flips the campaign to `active` and calls C6 with the resolved audience; wire an Activate action, confirmation, and locked view on the campaign detail page.

## Boundaries & Constraints

**Always:**
- Resolve the audience via existing `resolveAudienceEmployeeIdsForDefinition` while the campaign is still `draft` — same algebra Story 10.2 already validates. This read happens outside the transaction (like `previewAudience` today); staleness is fine because C6 re-validates every assignee is active inside the transaction and 400s if not.
- One `prisma.$transaction`: (1) atomic `formCampaign.updateMany({ where: { id, status: 'draft' }, data: { status: 'active' } })` — 0-count → 409, same pattern as `updateDraft`/`saveAudience`; (2) call C6 `createCampaignActionItems({ campaignId, authorId: creatorId, title, description, dueDate, link, assigneeIds }, tx)` with the same `tx`. Any C6 rejection (400 invalid assignees, 409 pre-existing items) rolls back the whole transaction — campaign stays `draft`.
- No new persistence for "frozen recipients": the `ActionItem` rows C6 creates (unique on `campaignId`+`assigneeId`) already are the frozen snapshot — do not add a `CampaignRecipient` table, despite that name in epic-context's technical decisions.
- Endpoint is creator-scoped like existing campaign routes (non-owned → 404); returns the updated `CampaignReadEntity`.
- Once `active`, the existing `updateDraft`/`saveAudience`/`previewAudience`/`resolveAudienceEmployeeIds` draft-only guards already 409 — no new lock code needed, and re-activation/reversion is blocked for free.
- Frontend: Activate button on `CampaignDetailPage` (draft only) behind a `ConfirmationModal` (irreversible, destructive style); on success invalidate campaign + audience-preview queries so the page re-renders the locked view (audience builder is already conditional on `status === 'draft'`).
- Tests: unit coverage for happy path, rollback on C6 rejection, concurrent-activation 409; e2e for `POST /activate`; Playwright activate flow.

**Ask First:**
- Whether to block activation when the resolved audience is empty. Neither PRD/epics nor C6 mandate a minimum — C6 already accepts `assigneeIds: []` → `[]`. Default: allow zero-recipient activation.

**Never:**
- Completion tracking table (Story 10.4).
- Changes to C6 itself, or to `updateDraft`/`saveAudience` guards (already sufficient).
- A deactivate / revert-to-draft path.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path | Draft campaign, resolved audience of 30 active ids | 200; status `active`; 30 `ActionItem` rows (`source: 'campaign'`) created | N/A |
| Already active | `POST /activate` on `status: 'active'` campaign | — | 409 |
| Non-creator | Another user calls activate | — | 404 |
| C6 rejects mid-activation | An assignee went inactive since audience save | — | 400; transaction rolled back; campaign stays `draft` |
| Concurrent activation | Two `POST /activate` calls race | Exactly one succeeds | Second gets 409 |
| Empty audience | No filters, no adds | 200; status `active`; 0 action items | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/campaigns/campaigns.service.ts` -- `resolveAudienceEmployeeIdsForDefinition` (private, :205-214, needs a `viewerUserId` from `resolveCreatorUserId` and a `definition` from `toAudienceDefinition` -- see the public `resolveAudienceEmployeeIds` at :193-203 for the call shape), `findOwnedDraftCampaign` (:370), `toAudienceDefinition` (:392) to reuse; add `activateCampaign(campaignId, creatorId)`
- `services/backend/src/modules/campaigns/campaigns.controller.ts` -- route pattern at :109-122; add `POST :campaignId/activate`
- `services/backend/src/modules/contracts/action-item-creation.contract.ts` -- C6 `createCampaignActionItems(input, tx?)` + `ActionItemWriteContext` shape; `tx` is resolved via `action-items.module.ts`'s `@Global()` export, no `campaigns.module.ts` change needed
- `services/backend/src/modules/action-items/action-items.service.ts:80-99` -- `createCampaignActionItems` dispatcher (accepts optional `tx`, delegates to `runCampaignActivation`); its guard/createMany/rollback logic is `runCampaignActivation` at :452-507
- `services/backend/src/modules/campaigns/campaigns.swagger.ts` -- add `SwaggerActivateCampaign` (include an `ApiBadRequestResponse` documenting C6's `{ message, invalidAssigneeIds }` rejection body)
- `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts`, `services/backend/test/campaigns.e2e-spec.ts` -- extend `PrismaMock` (`$transaction`) + `ActionItemCreation` mock; activation matrix
- `services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:61-70,96-110` -- add Activate button + `ConfirmationModal` (pattern: `AllEmployeesPage.tsx`)
- `services/frontend/src/api/services/campaign.service.ts`, `api/hooks/useCampaigns.ts`, `useCampaignMutations.ts`, `hooks/data/useCampaignsData.ts` -- add `activateCampaign` call → `useActivateCampaign` mutation (mirror `useSaveCampaignAudience`) → `useActivateCampaignData`
- `services/frontend/src/locales/en/translation.json` (`campaigns.*`, :30-90) -- add `campaigns.activate.*` keys
- `services/frontend/e2e/flows/campaigns/campaigns.spec.ts`, `helpers.ts`, `fixtures.ts` -- extend for activate flow

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/campaigns/campaigns.service.ts` -- add `activateCampaign` -- transaction per Boundaries
- [x] `services/backend/src/modules/campaigns/campaigns.controller.ts` -- `POST :campaignId/activate` route
- [x] `services/backend/src/modules/campaigns/campaigns.swagger.ts` -- `SwaggerActivateCampaign`
- [x] `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts` + `services/backend/test/campaigns.e2e-spec.ts` -- activation matrix
- [x] `services/frontend/src/api/services/campaign.service.ts` + `api/hooks/` + `hooks/data/` -- `activateCampaign` mutation
- [x] `services/frontend/src/pages/CampaignDetailPage/` -- Activate button + confirmation (disable confirm while the mutation is pending) + locked-view wiring; on a 409 response, refetch the campaign so a stale second tab/window drops the Activate button
- [x] `services/frontend/src/locales/en/translation.json` -- activation copy
- [x] `services/frontend/e2e/flows/campaigns/` -- extend for activate flow

**Acceptance Criteria:**
- Given a draft campaign has a resolved audience of 30 people, when the creator activates it, then the campaign becomes `active`, its fields lock, and exactly 30 `ActionItem` rows exist for that `campaignId`
- Given activation fails partway (e.g. an assignee went inactive), when the transaction rolls back, then the campaign remains a fully-editable draft and no action items exist for it
- Given a campaign is already `active`, when activate is called again, then the response is 409 and no new action items are created
- Given I am not the campaign's creator, when I call activate for that campaign, then I receive 404
- Given a draft campaign has an empty resolved audience, when the creator activates it, then the campaign becomes `active` with zero `ActionItem` rows
- Given two activation requests race on the same draft campaign, when both are processed, then exactly one succeeds and the other receives 409

### Review Findings

- [x] [Review][Patch] `useActivateCampaign`'s `onError` shows the generic activation-failure toast unconditionally, then stacks a second, more specific toast when the response carries `invalidAssigneeIds` — two toasts for one failure [`services/frontend/src/api/hooks/useCampaignMutations.ts:91-111`]
- [x] [Review][Patch] `campaigns.activate.invalidAssignees` interpolates `"{{count}} recipient(s)…"` instead of using the project's own i18next plural-suffix convention (`_one`/`_other`, per `.claude/rules/react-i18n.md`) — reads ungrammatically at `count === 1` and won't localize [`services/frontend/src/locales/en/translation.json:100`]
- [x] [Review][Patch] `CampaignDetailPage.tsx` imports `useActivateCampaignData` from `hooks/data/` directly and defines activation business logic (`handleActivate`, `activateOpen` state) inline, violating the documented convention ("Never import `@/api/hooks/` or `@/hooks/data/` from a page or section component file" / "Page component consumes page hook only" in `.claude/rules/react-pages.md`) — extends a pre-existing violation (`useCampaignData`) rather than correcting it [`services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:8,30,39-47`]
- [x] [Review][Patch] Activation-failure toast copy ("Couldn't activate the campaign. Try again.") deviates from `EXPERIENCE.md`'s State Patterns table, which specifies "Couldn't save. Try again." for "Action/save failure" on any write and explicitly names campaign activation in that same row [`services/frontend/src/locales/en/translation.json:99`]
- [x] [Review][Patch] No test (Playwright or otherwise) verifies the Activate confirm button is actually disabled while the mutation is pending, despite the spec's own task list requiring it — a regression dropping `confirmDisabled={isActivatingCampaign}` would ship without any test failing [`services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:159`, `services/frontend/e2e/flows/campaigns/campaigns.spec.ts`]
- [x] [Review][Patch] On 409/404 activation failure, `handleActivate` intentionally keeps `activateOpen` true (catch block never closes the dialog) while `useActivateCampaign`'s `onError` refetches an already-active campaign — the destructive confirmation modal can remain open over the locked active view; the Playwright 409 test asserts recovery of draft controls but not dialog dismissal [`services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:39-47`, `services/frontend/e2e/flows/campaigns/campaigns.spec.ts:116-155`]
- [x] [Review][Defer] Audience `employeeId[]` is resolved before the transaction opens, so a concurrent `PUT :campaignId/audience` save landing in that gap is not detected — the frozen recipients can end up one save behind the campaign's persisted audience definition; spec's staleness sanction covers only per-employee active-status drift (which C6 re-validates), not drift in the audience definition itself — deferred, closing it needs a tx-scoped `EmployeeDirectory.listEmployees`, a design decision beyond this story [`services/backend/src/modules/campaigns/campaigns.service.ts:219-222`]
- [x] [Review][Defer] No e2e races a real concurrent `PATCH :campaignId` against `POST :campaignId/activate` against real Postgres — deferred, the in-transaction re-fetch behavior is already deterministically covered by a mocked unit test; a true DB-timing race test would be flaky [`services/backend/src/modules/campaigns/campaigns.service.ts:237`]
- [x] [Review][Defer] No frontend test exercises the 400 invalid-assignees branch of `onError` — deferred, low priority; backend e2e already covers the C6-rejection scenario end to end
- [x] [Review][Defer] `NotFoundException` branch for `freshCampaign === null` inside the transaction is untested — deferred, effectively unreachable (no campaign-delete endpoint exists; the row was just updated in the same transaction) [`services/backend/src/modules/campaigns/campaigns.service.ts:240-242`]
- [x] [Review][Defer] `SwaggerActivateCampaign`'s 409 doc only describes "already active," not the race-loss path the e2e concurrency test covers — deferred, minor doc completeness, same response body either way [`services/backend/src/modules/campaigns/campaigns.swagger.ts:87`]
- [x] [Review][Defer] No frontend test covers a generic network/5xx failure during activation — deferred, low priority; the generic toast path is untested but low-risk
- [x] [Review][Defer] Theoretical double-submit race if a user double-clicks before React commits the `disabled` state — deferred, low real-world risk given React 18 event batching plus the backend's own 409 race guard as a backstop [`services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:39-47`]
- [x] [Review][Defer] `useActivateCampaign`'s `onError` doesn't special-case a 403 (no employee record) response — deferred, pre-existing pattern shared by every other campaign mutation hook, not a regression introduced here [`services/frontend/src/api/hooks/useCampaignMutations.ts:91-123`]
- [x] [Review][Defer] Diff bundles unrelated Prettier-only formatting churn (several `directory`/`risks` files, `saved-views.service.ts`) alongside the activation feature — deferred, hygiene note for commit hygiene, not a code defect
- [x] [Review][Defer] `useActivateCampaign` `onError` invalidates only `campaignQueryKey` on 409/404, not `campaignsListQueryKey` — the campaigns list can still show `draft` after a concurrent activation elsewhere until manually refreshed [`services/frontend/src/api/hooks/useCampaignMutations.ts:117-121`]
- [x] [Review][Defer] Backend activate happy-path e2e asserts action-item count/source/status/assignee only, not persisted `title`/`description`/`link`/`dueDate` metadata — unit test checks the C6 call args instead [`services/backend/test/campaigns.e2e-spec.ts:870-881`]
- [x] [Review][Defer] `POST :campaignId/activate` has no 403 e2e for users without an employee record — create/list 403 cases exist; same `resolveViewerEmployeeId` path [`services/backend/test/campaigns.e2e-spec.ts`]
- [x] [Review][Defer] Playwright 409 recovery test uses unscoped `getByText('Active')` — brittle if other UI copy contains the same word [`services/frontend/e2e/flows/campaigns/campaigns.spec.ts:110,151`]
- [x] [Review][Defer] Activation confirmation copy does not surface the resolved recipient count or call out the zero-recipient case, even though empty-audience activation is allowed [`services/frontend/src/locales/en/translation.json:95`, `CampaignDetailPage.tsx:150-151`]
- [x] [Review][Defer] A creator can activate while the audience builder holds unsaved local changes — `POST /activate` resolves the persisted server audience, not the in-memory draft shown in the UI [`CampaignDetailPage.tsx`, `useCampaignAudienceSection.ts`]
- [x] [Review][Defer] Playwright happy-path activate test creates a campaign and activates immediately without saving an audience first — does not exercise the primary build-audience-then-activate flow [`services/frontend/e2e/flows/campaigns/campaigns.spec.ts:83-114`]

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm test -- --testPathPatterns=campaigns && npm run test:e2e -- campaigns` -- expected: all pass
- `cd services/frontend && npm run lint && npm run build && npm run test e2e/flows/campaigns` -- expected: pass

**Manual checks:**
- As creator, activate a draft campaign with a filtered audience; confirm locked view. There is no frontend surface yet for a recipient's own action items, so confirm delivery via `GET employees/:employeeId/action-items` for a couple of resolved recipients. As another user, `POST /campaigns/:id/activate` for someone else's campaign -- expect 404.

## Suggested Review Order

**Atomic activation transaction (backend)**

- Entry point — ownership check, draft guard, pre-transaction audience resolution.
  [`campaigns.service.ts:207`](../../services/backend/src/modules/campaigns/campaigns.service.ts#L207)

- Atomic `updateMany` (draft→active) inside the transaction; race-loss 409.
  [`campaigns.service.ts:224`](../../services/backend/src/modules/campaigns/campaigns.service.ts#L224)

- In-transaction re-fetch closes the metadata staleness race (review patch).
  [`campaigns.service.ts:237`](../../services/backend/src/modules/campaigns/campaigns.service.ts#L237)

- Route wiring — `HttpCode(OK)` overrides Nest's default 201 for `POST`.
  [`campaigns.controller.ts:127`](../../services/backend/src/modules/campaigns/campaigns.controller.ts#L127)

**API contract**

- Swagger response matrix for the new endpoint (200/400/403/404/409).
  [`campaigns.swagger.ts:87`](../../services/backend/src/modules/campaigns/campaigns.swagger.ts#L87)

**Frontend activation flow**

- Mutation hook — success invalidation, 409/404 refetch, 400 detail toast (review patch).
  [`useCampaignMutations.ts:74`](../../services/frontend/src/api/hooks/useCampaignMutations.ts#L74)

- `onError` — the specific status-code branching added by review.
  [`useCampaignMutations.ts:91`](../../services/frontend/src/api/hooks/useCampaignMutations.ts#L91)

- Confirm handler — try/catch keeps the dialog open on failure (review patch).
  [`CampaignDetailPage.tsx:39`](../../services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx#L39)

- Destructive-style confirmation dialog per spec's irreversibility requirement.
  [`CampaignDetailPage.tsx:143`](../../services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx#L143)

- Thin API call backing the mutation.
  [`campaign.service.ts:44`](../../services/frontend/src/api/services/campaign.service.ts#L44)

**Tests**

- Backend unit matrix — happy path, empty audience, 404/409, race-loss, C6 rollback.
  [`campaigns.service.spec.ts:417`](../../services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts#L417)

- Backend e2e matrix against real Postgres — same scenarios end-to-end.
  [`campaigns.e2e-spec.ts:818`](../../services/backend/test/campaigns.e2e-spec.ts#L818)

- Playwright happy path — activate and lock the detail page.
  [`campaigns.spec.ts:83`](../../services/frontend/e2e/flows/campaigns/campaigns.spec.ts#L83)

- Playwright 409 path — added by review to close a verification gap.
  [`campaigns.spec.ts:116`](../../services/frontend/e2e/flows/campaigns/campaigns.spec.ts#L116)

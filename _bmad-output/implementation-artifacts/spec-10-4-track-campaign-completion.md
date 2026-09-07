---
title: 'Track Campaign Completion'
type: 'feature'
created: '2026-09-07'
status: 'done'
review_loop_iteration: 1
baseline_commit: '58aaf6fd321fb3d45d2f223c7ef56a6da5000612' # services/backend HEAD; services/frontend HEAD: beec6c8d378a6ba20a68522dd9d0ee9b4316795c
story_key: '10-4-track-campaign-completion'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-10-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-10-3-activate-a-campaign.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-3-overdue-highlighting.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 10.3 activates campaigns and creates one `ActionItem` per frozen recipient, but the creator has no progress view — `CampaignDetailPage` shows only a locked summary for `active` campaigns, and `CampaignsPage` blocks navigation to them.

**Approach:** Add a creator-scoped `GET :campaignId/completion` endpoint that reads campaign action items and maps each to a recipient row (`status` + `isOverdue` via `isActionItemOverdue`), then render a live-updating completion table on the campaign detail page using shared `OverdueIndicator` and positive `StatusBadge` components.

## Boundaries & Constraints

**Always:**
- **Read model:** frozen recipients are existing `ActionItem` rows for the `campaignId` (unique on `campaignId`+`assigneeId`) — no `CampaignRecipient` table.
- **Endpoint:** `GET /campaigns/:campaignId/completion` — same creator-scoped auth as other campaign routes (`findOwnedCampaign` → non-owner or unknown id **404**; authenticated user without an employee record **403**, same as other campaign routes). Campaign must be `active`; `draft` → **409**. Response body: `{ recipients: CampaignCompletionRow[] }` — no pagination wrapper.
- **Overdue:** call `isActionItemOverdue(status, dueDate, clock)` from `action-item-input.ts` at map time with injected `Clock` — never duplicate date logic on frontend or in campaigns. Open items due **today** (UTC calendar) → `isOverdue: false` (same rule as spec-4-3).
- **Row shape:** `actionItemId`, `assignee: { id, displayName }`, `status`, `dueDate` (ISO `YYYY-MM-DD`), `isOverdue`, optional `completedAt`. `displayName` uses the same `displayName(user)` fallback as action items (trimmed `name`, else `email`). Order by assignee `displayName` ascending, then `actionItemId` ascending (stable tie-break). Return the full recipient set (no pagination).
- **Display buckets:** `completed` (`status === 'completed'`), `overdue` (`open` + `isOverdue`), `not yet completed` (`open` + `!isOverdue`). `cancelled` rows render with a muted cancelled badge (not the positive completed badge) and are excluded from the three AC buckets.
- **Frontend:** render the completion table when `campaign.status === 'active'` on the locked detail view (summary read-only; no edit/activate affordances — already draft-gated in `CampaignDetailPage.tsx:63-82`). Table columns: assignee name, status (Overdue Indicator / positive Status Badge / neutral open label / cancelled badge). Trust backend `isOverdue` and `dueDate` for OverdueIndicator copy (no client derivation). Poll with `refetchInterval: 60_000` while the detail page is mounted and the campaign is `active` (same interval as `useAuthSession.ts`; extract a shared constant if preferred). Enable the completion query only when `campaign?.status === 'active'` — never prefetch on draft. Show inline loading/error for the completion query; stop polling on terminal 404/409. Use `components/ui/table.tsx` for layout. No summary-count strip above the table — bucket breakdown is visible from row treatments alone.
- **Shared UI:** create `OverdueIndicator` (3px destructive left border + label including formatted `dueDate`, per `DESIGN.md:137` / spec-4-3) and positive `StatusBadge` (per `DESIGN.md:138`) — first surfaces for these primitives; campaign table and future action-item rows share them. Cancelled rows use a separate muted badge variant (not `StatusBadge`).
- **Navigation:** `CampaignsPage` must allow opening `active` campaign detail (today draft-only at `:60-70`).
- **Visibility:** creator-only for this story — matches the existing campaigns module and intentionally defers epic-10-context's "equivalent-access holders" Colleague grant; no access-resolver exception in this story.

**Never:**
- Verifying or reading the external form URL.
- Access-resolver scoped Colleague grant (`epic-10-context.md` — separate work).
- Expanding visibility to non-creator "equivalent-access holders" without a module-wide auth change.
- Pagination, SSE/WebSockets, or a `closed` campaign status.
- Changes to C6 activation or overdue derivation rules.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path | Active campaign, 30 action items: 10 completed, 5 open+overdue, 15 open+not overdue | 200; `{ recipients }` with 30 rows, correct `status`/`isOverdue` buckets, sorted by `displayName` then `actionItemId` | N/A |
| Draft campaign | `GET /completion` on `status: 'draft'` | — | 409 |
| Unknown campaign id | Valid creator auth, random UUID | — | 404 |
| Non-creator | Another user requests completion for an existing campaign | — | 404 |
| No employee record | Authenticated user without linked `Employee` | — | 403 |
| Empty audience | Active campaign with 0 action items (10.3 allows) | 200; `{ recipients: [] }` | N/A |
| Open due today | `status: 'open'`, `dueDate` equals clock's UTC calendar day | Row `isOverdue: false` | N/A |
| Marked complete without filling form | Recipient completes their item | Row shows `completed` + positive badge | N/A |
| Cancelled recipient | Departed employee's item auto-cancelled | Row shows cancelled badge; not counted as overdue | N/A |
| Completed/cancelled past due | Terminal item with past `dueDate` | `isOverdue: false` | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/campaigns/campaigns.controller.ts:127-137` -- route pattern for `POST :campaignId/activate`; add `GET :campaignId/completion` alongside
- `services/backend/src/modules/campaigns/campaigns.service.ts:466-478` -- `findOwnedCampaign` reuse; add `getCompletion(campaignId, creatorId)` querying `prisma.actionItem.findMany({ where: { campaignId }, include: assignee.user })`
- `services/backend/src/modules/campaigns/campaigns.service.ts:207-259` -- `activateCampaign` confirms action items are the frozen snapshot
- `services/backend/src/modules/campaigns/entities/campaign.entity.ts` -- add `CampaignCompletionEntity` (`{ recipients: CampaignCompletionRow[] }`) + row DTOs (`dueDate`, `isOverdue`, optional `completedAt`) with Swagger `@ApiProperty`
- `services/backend/src/modules/campaigns/campaigns.swagger.ts` -- `SwaggerGetCampaignCompletion` (200/403/404/409)
- `services/backend/src/modules/action-items/action-item-input.ts:24-35` -- `isActionItemOverdue` — sole overdue derivation
- `services/backend/src/modules/action-items/action-items.service.ts:349-375,422-428` -- `toReadDto` / `displayName` pattern to mirror for assignee names
- `services/backend/prisma/schema.prisma:441-466` -- `ActionItem` model + `@@unique([campaignId, assigneeId])`
- `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts` + `services/backend/test/campaigns.e2e-spec.ts` -- completion matrix (extend existing campaign test harness)
- `services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:93-120` -- render completion section when `status === 'active'` (replaces activate-only gap below summary)
- `services/frontend/src/pages/CampaignsPage/CampaignsPage.tsx:60-70` -- enable row navigation for `active` campaigns
- `services/frontend/src/api/services/campaign.service.ts:44-46` -- add `getCampaignCompletion`; mirror hook chain: `api/hooks/useCampaigns.ts` → `hooks/data/useCampaignsData.ts` → page hook
- `services/frontend/src/types/campaigns.ts` -- completion row types
- `services/frontend/src/components/OverdueIndicator/` -- **new** shared component
- `services/frontend/src/components/StatusBadge/` -- **new** positive completed badge (follow `RiskBadge.tsx` wrapper pattern)
- `services/frontend/src/components/ui/table.tsx` -- table primitives
- `services/frontend/src/locales/en/translation.json` -- `campaigns.completion.*` keys
- `services/frontend/e2e/flows/campaigns/campaigns.spec.ts` -- assert completion table after activate

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/campaigns/entities/campaign.entity.ts` -- add completion DTOs -- API contract
- [x] `services/backend/src/modules/campaigns/campaigns.service.ts` -- `getCompletion` with `isActionItemOverdue` mapping -- core read path
- [x] `services/backend/src/modules/campaigns/campaigns.controller.ts` + `campaigns.swagger.ts` -- `GET :campaignId/completion` route + Swagger -- expose endpoint
- [x] `services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts` + `services/backend/test/campaigns.e2e-spec.ts` -- completion matrix incl. due-today, sort stability, 403/404 -- backend coverage
- [x] `services/frontend/src/types/campaigns.ts` + `api/services/campaign.service.ts` + `api/hooks/` + `hooks/data/` -- completion fetch + polled query (`enabled: status === 'active'`, `refetchInterval: 60_000`) -- data layer
- [x] `services/frontend/src/components/OverdueIndicator/` + `StatusBadge/` + cancelled badge variant -- shared status primitives -- UX parity with DESIGN.md
- [x] `services/frontend/src/pages/CampaignDetailPage/` -- `CampaignCompletionTable` section + polled query (`enabled: status === 'active'`, `refetchInterval: 60_000`) + loading/error states -- primary UI
- [x] `services/frontend/src/pages/CampaignsPage/CampaignsPage.tsx` -- allow `active` row navigation -- unblock access to tracking view
- [x] `services/frontend/src/locales/en/translation.json` -- completion copy -- i18n
- [x] `services/frontend/e2e/flows/campaigns/campaigns.spec.ts` -- completion table assertions -- e2e

**Acceptance Criteria:**
- Given a campaign was activated for 30 recipients, 10 of whom marked their action item complete and 5 of whom are past due and still open, when the creator opens the tracking table, then they see 10 completed, 5 overdue, and 15 not-yet-completed-but-not-overdue, live-updating with no manual refresh
- Given a recipient has not actually filled in the external form but marks their action item complete anyway, when the creator views the tracking table, then that recipient shows as completed
- Given a campaign is still in draft, when the creator requests the completion endpoint, then the response is 409
- Given I am not the campaign creator, when I request completion for that campaign, then I receive 404
- Given an active campaign was activated with zero recipients, when the creator opens the tracking table, then an explicit empty state is shown

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm test -- --testPathPatterns=campaigns && npm run test:e2e -- campaigns` -- expected: all pass
- `cd services/frontend && npm run lint && npm run build && npm run test e2e/flows/campaigns` -- expected: pass

**Manual checks:**
- Activate a campaign with a mixed audience; open detail as creator and confirm table buckets match action-item states (including an open item due today showing as not overdue). Open the same URL as another user — expect 404 on the completion API. Leave the detail page open while completing a recipient's item via `POST .../action-items/:id/complete` — table should update within 60s without refresh.

### Review Findings

- [x] [Review][Patch] Completion table hidden when background refetch fails [`services/frontend/src/pages/CampaignDetailPage/CampaignDetailPage.tsx:139`] — gate is `completion && !isCompletionError`; a failed 60s poll clears the table even when stale `completion` data is still in cache. Show the table whenever `completion` exists and surface the error alongside.
- [x] [Review][Patch] Missing backend e2e 403 for completion endpoint [`services/backend/test/campaigns.e2e-spec.ts`] — spec I/O matrix requires 403 for authenticated user without employee record; existing no-employee block covers create/list only. Extend with `GET .../completion` → 403.
- [x] [Review][Patch] Cancelled recipient row untested in frontend e2e [`services/frontend/e2e/flows/campaigns/campaigns.spec.ts`] — `CampaignCompletionTable` renders `CancelledBadge` for `status === 'cancelled'` but no e2e fixture/assertion covers it.
- [x] [Review][Patch] Active campaign list navigation untested [`services/frontend/e2e/flows/campaigns/campaigns.spec.ts`] — `CampaignsPage` enables row click for `active` campaigns; all e2e flows enter detail while still `draft`. Add test that opens a pre-seeded `active` row from the list.
- [x] [Review][Patch] OverdueIndicator hardcodes `en-US` date locale [`services/frontend/src/components/OverdueIndicator/OverdueIndicator.tsx:11`] — `toLocaleDateString('en-US', …)` bypasses active i18n locale; use `i18n.language` (or shared date helper) for the due-date fragment inside the translated overdue label.
- [x] [Review][Patch] `displayName` email fallback untested [`services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts`] — `getCompletion` fixtures always supply non-empty `name`; add case with blank/`null` name and email present to pin spec fallback.
- [x] [Review][Defer] Completion loading/error and polling behavior untested in e2e — inline states and `refetchInterval` are wired correctly; automated timer/poll assertions deferred as brittle/low ROI for this story.
- [x] [Review][Defer] Draft detail completion prefetch untested — `enabled: status === 'active'` is correct in hook chain; no network-count assertion added.
- [x] [Review][Defer] `displayName` duplicated in `CampaignsService` and `ActionItemsService` — mirrors spec intent ("same fallback as action items"); shared helper is a refactor outside story scope.
- [x] [Review][Defer] Pre-existing backend e2e failure in `rejects inactive added employee ids` — `employmentStatus: "inactive"` Prisma validation error unrelated to story 10.4 completion changes.

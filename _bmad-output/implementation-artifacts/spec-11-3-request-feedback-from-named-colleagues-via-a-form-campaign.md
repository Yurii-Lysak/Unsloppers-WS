---
title: 'Request Feedback from Named Colleagues via a Form Campaign'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 2
baseline_commit: '73f9cf4c5fa3555204c5bd2345072110d4a36432'
backend_baseline_commit: '73f9cf4c5fa3555204c5bd2345072110d4a36432'
frontend_baseline_commit: '1b8f597f05c8737c463cf252643537712df58dc1'
story_key: '11-3-request-feedback-from-named-colleagues-via-a-form-campaign'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-11-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/epic-10-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-11-2-view-feedback-over-time-and-compare-periods.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 11.1–11.2 deliver S8 CRUD and period comparison, but managers and PPs still cannot request feedback from named colleagues via the existing Form Campaign mechanism (PRD FR-52 / UX Feedback Panel).

**Approach:** Add a **Request feedback…** action on the S8 Feedback Panel (list mode) that frontend-orchestrates Epic 10: create a draft campaign with subject-scoped default copy, save audience via `addedEmployeeIds` only (no filters), then navigate to the existing campaign detail page for activation and completion tracking (Stories 10.3–10.4). No backend changes — AD-11 keeps `feedbacks` and `campaigns` decoupled.

## Boundaries & Constraints

**Always:**
- **Frontend-only.** Reuse existing APIs: `POST /api/v1/campaigns`, `PUT /api/v1/campaigns/:id/audience`, then user activates on `CampaignDetailPage`. No new backend fields, routes, or `feedbacks`↔`campaigns` coupling.
- **Entry point:** S8 Feedback Panel, **list mode only** — hidden in compare mode and for `accessLevel !== 'RW'`.
- **Who sees the action:** `profile.audience.role` is **`ReportingLine` or `PP` only** (PRD FR-52 / EXPERIENCE.md — not `ProjectLine`-only PM viewers even when S8 is RW). D13 union: when a viewer matches both `ReportingLine` and `ProjectLine`, the profile payload's single `audience.role` label is the higher-ranked match (`ReportingLine` > `PP` > `ProjectLine`), so dual-relationship managers still see the action. **`FullAccess`, `Self`, `Colleague`, and `SharedLink` never see it** — EXPERIENCE limits the entry to manager/PP relationship context, not org-wide admin. Backend `CREATE_FORM_CAMPAIGNS` still enforced on create (403 → destructive toast).
- **Subject association:** encode in campaign metadata only — pre-fill `title`, `description`, and `purpose` via i18n templates (`employeeProfile.s8.requestFeedback.defaults.*`) interpolating the profile subject's `displayName` (`profile.displayName` on the viewed employee). **`FormCampaign` has no `subjectEmployeeId`** today; do not add one in this story. Metadata defaults on dialog open: `link` and `dueDate` start empty — user must supply both (same required validation as `createCampaignFormSchema`).
- **Dialog contents:** (1) campaign metadata fields — same validation as `createCampaignFormSchema` (`title` ≤200, `description` ≤500, `purpose` ≤2000, http(s) `link`, `dueDate` `YYYY-MM-DD`); (2) named-colleague multi-select from the requester's visible employee list (`useEmployeesListData`, page 1, size 100 — same source as `useCampaignAudienceSection`). **Exclude the profile subject (`employeeId`)** from selectable colleagues; the requester may include their own employee id. While employees are loading, disable submit and show the shared loading pattern; on employees-list fetch failure, show inline error under the picker and keep submit disabled.
- **Audience on submit:** `{ filters: [], addedEmployeeIds: selectedColleagueIds, excludedEmployeeIds: [] }` — named individuals only, never filter-resolved audience in this flow. Deduplicate selected ids client-side before submit.
- **Colleague validation:** require **≥1** selected colleague before submit; inline message under picker when empty.
- **Orchestration sequence:** create campaign → save audience → invalidate campaigns list query (`queryClient.invalidateQueries` for campaigns list key) → `navigate(\`/campaigns/${campaignId}\`)`. Campaign remains **draft** — user activates via existing detail UI. One consolidated success toast (`employeeProfile.s8.requestFeedback.success`) after both steps succeed; single destructive toast on any failure (no partial silent state). Dialog mirrors `CampaignFormDialog`: block close/cancel while submit in flight; disable submit button during orchestration.
- **No auto Feedback record:** completing campaign action items never creates S8 records — requester manually authors feedback (Story 11.1) after reading the external form; completion table (Story 10.4) is the signal only.
- **i18n:** keys under `employeeProfile.s8.requestFeedback.*`; reuse `campaigns.form.*` validation messages for metadata fields.

**Ask First:**
- If product requires a persisted `subjectEmployeeId` on `FormCampaign` (queryable subject link beyond title copy), HALT — needs a backend migration and is out of AD-11 thin-reuse scope.
- If product requires FullAccess HR admins to launch feedback requests from S8, HALT — needs PO override of the manager/PP-only UX gate.

**Never:**
- Auto-activate after create; filter-based audience in the request dialog; duplicate Epic 10 activation/completion UI; backend module coupling; auto-create `FeedbackRecord` from campaign completion; seed data changes; PM-only (`ProjectLine`) request entry without PO override; paginated colleague search beyond the existing page-1/size-100 employee list (out of v1 scope).

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Request from three colleagues | PP; subject B; three valid colleague IDs selected; valid metadata | Draft campaign created; audience `addedEmployeeIds` = those three; redirect to `/campaigns/:id`; subject named in pre-filled title/purpose | N/A |
| Zero colleagues | Submit with empty selection | Inline validation; no API calls | N/A |
| Subject not in picker | Subject B on profile B | B absent from colleague options | N/A |
| Requester selects self | Requester id in colleague multi-select | Allowed in `addedEmployeeIds` (Epic 10 parity); subject still excluded | N/A |
| Duplicate colleague picks | Same id selected twice in UI | Client dedupes before PUT; server never sees duplicates | N/A |
| Invalid colleague ID | Stale/invisible ID submitted | — | 400 + toast (`invalidEmployeeIds` or generic save failure) |
| Invalid metadata | Missing/invalid link or dueDate on submit | Inline field validation; no API calls | N/A |
| Create fails | POST returns 400/403 before audience save | Destructive toast; dialog stays open; no navigation | toast |
| No campaign permission | ReportingLine viewer without `CREATE_FORM_CAMPAIGNS` | — | 403 + destructive toast |
| PM ProjectLine only | S8 RW; `audience.role === 'ProjectLine'` | Request button not rendered | N/A |
| Dual ReportingLine + ProjectLine | S8 RW; union resolves `audience.role === 'ReportingLine'` | Request button rendered | N/A |
| FullAccess admin | S8 RW; `audience.role === 'FullAccess'` | Request button not rendered | N/A |
| Self / Colleague / SharedLink | Any non-manager audience on profile | Request button not rendered (section may be `R` or absent) | N/A |
| Compare mode | User in compare view | Request button not rendered | N/A |
| Employees list loading | `useEmployeesListData` in flight | Picker shows loading; submit disabled | N/A |
| Employees list failure | Employee list query errors | Inline picker error; submit disabled | N/A |
| Empty colleague universe | List returns zero rows (excluding subject) | Picker empty; submit blocked by ≥1 colleague rule | N/A |
| Completion without auto feedback | Activated campaign; all assignees complete action items | Completion table shows completed; **no** new `FeedbackRecord` on subject profile | N/A |
| Audience save fails after create | Create 201; audience PUT fails | Destructive toast; user can fix audience on campaign detail (draft exists) | toast |

</frozen-after-approval>

## Code Map

Normative behavior lives in **Boundaries** and **I/O & Edge-Case Matrix** above; paths below are execution hints only.

- `services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx` — pass `profile.audience.role` and subject `displayName` into `ProfileSections` / S8 renderer (today only `employeeId` + section envelope reach `FeedbackSectionCard`).
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — extend `SectionRenderer` props with `subjectDisplayName` + `audienceRole`; wire S8 with subject context.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/FeedbackSection.tsx` — **Request feedback…** button (list + ReportingLine/PP + RW); open dialog.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/RequestFeedbackDialog.tsx` (new) — metadata form + colleague multi-select; submit triggers orchestration hook; block dismiss while submitting (mirror `CampaignFormDialog.tsx:33-36`).
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/hooks/useRequestFeedbackFlow.ts` (new) — `campaignApiService.createCampaign` → `saveCampaignAudience` → campaigns-list cache invalidation → navigate; consolidated toasts; returns/errors for dialog.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/schemas/request-feedback.schema.ts` (new) — extends campaign fields + `colleagueIds: string[]` min 1.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/schemas/__tests__/request-feedback.schema.test.ts` (new) — colleague min-1 + metadata parity with campaign schema.
- `services/frontend/src/pages/CampaignsPage/schemas/campaign-form.schema.ts` — reuse field validators / compose into request-feedback schema.
- `services/frontend/src/hooks/data/useEmployeesData.ts` — `useEmployeesListData({ page: 1, pageSize: 100 })` for colleague options (mirror `useCampaignAudienceSection.ts:16–30`).
- `services/frontend/src/pages/CampaignDetailPage/hooks/useCampaignAudienceSection.ts` — reference for `addedEmployeeIds`-only audience pattern and add-candidate filtering.
- `services/frontend/src/api/services/campaign.service.ts` — `createCampaign`, `saveCampaignAudience` (orchestration calls service directly to capture returned `Campaign.id` without double mutation toasts from `useCampaignMutations`).
- `services/frontend/src/hooks/data/useCampaignsData.ts` — export/query key used for post-create list invalidation.
- `services/frontend/src/components/AudienceBuilder/AudienceBuilder.tsx` — read-only reference for individual-add semantics (`addEmployee` lines 86–95); request dialog uses simpler multi-select, not full filter UI.
- `services/backend/prisma/schema.prisma` (`FormCampaign` ~497–517) — **read-only:** no `subjectEmployeeId`; audience stored in `audienceAddedEmployeeIds`.
- `services/backend/test/campaigns.e2e-spec.ts` (~532–611) — adds-only audience e2e pattern for behavioral parity.

**Read-only (Epic 10 — do not change unless bugfix):** `CampaignDetailPage`, `useActivateCampaign`, `CampaignCompletionTable`, backend `campaigns.service.ts`.

## Tasks & Acceptance

**Execution:**
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` + `EmployeeProfilePage.tsx` — pass `subjectDisplayName` + `audienceRole` to S8 renderer — role gate + subject copy
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/schemas/request-feedback.schema.ts` — campaign fields + colleagueIds validation — form contract
- [x] `services/frontend/e2e/request-feedback.schema.spec.ts` — schema validation tests (Playwright; repo has no Vitest) — validation contract
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/hooks/useRequestFeedbackFlow.ts` — create + save audience + invalidate + navigate orchestration — AD-11 sequencing
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/RequestFeedbackDialog.tsx` — dialog UI (metadata + colleague picker + loading/error) — FR-52 entry point
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/FeedbackSection.tsx` — wire button + dialog; hide in compare mode — UX placement
- [x] `services/frontend/src/locales/en/translation.json` — `employeeProfile.s8.requestFeedback.*` (including `defaults.*` and `success`) — i18n contract
- [x] `services/frontend/e2e/request-feedback-campaign.spec.ts` (new) — stub profile + employees; assert dialog → campaign detail with named audience — epic AC

**Acceptance Criteria:**
- Given I am a PP viewing employee B's profile, when I click Request feedback…, select three colleagues by name, fill campaign metadata, and submit, then a draft form campaign exists targeted at exactly those three via `addedEmployeeIds`, with copy referencing B, and I land on `/campaigns/:id` ready to activate
- Given the feedback-request campaign is activated and all three colleagues mark their action items complete, when I open the campaign completion table, then all three show completed and no Feedback record was auto-created on B's profile
- Given I am a PM with S8 RW via ProjectLine only, when I view B's Feedback Panel, then the Request feedback… action is not shown
- Given I am a direct manager (ReportingLine) who is also PM on B via ProjectLine, when I view B's Feedback Panel in list mode, then the Request feedback… action is shown
- Given I am in compare mode on S8, when the panel renders, then Request feedback… is not shown and compare behavior from Story 11.2 is unchanged

## Spec Change Log

- 2026-09-08 — bmad-review (iteration 1): D13 role-rank gate, FullAccess exclusion, metadata defaults, dialog submit UX, cache invalidation, expanded matrix rows, schema unit test, canonical matrix note, i18n success/defaults keys.

- 2026-09-08 — bmad-code-review (iteration 2): lazy dialog mount, query-key import layer, e2e payload assertions, FullAccess/audience-failure coverage, GET stub method fix.

## Design Notes

**PRD FR-52** here is the Epic 11 feedback-request requirement (named-colleague Form Campaign), not the Timetracker FR-52 referenced elsewhere in planning artifacts. Subject linkage lives in human-readable campaign copy because AD-11 forbids backend cross-module coupling and `FormCampaign` has no subject FK. The colleague picker reuses the same visible-universe employee list as campaign audience building (page 1, size 100) so server-side `invalidEmployeeIds` checks stay aligned; colleagues beyond that page are out of v1 scope.

## Verification

**Commands:**
- `cd services/frontend && npm run typecheck` — expected: clean
- `cd services/frontend && npm test -- request-feedback.schema` — expected: schema unit tests pass
- `cd services/frontend && npm run test -- request-feedback-campaign` — expected: Playwright pass

**Manual checks (if no CLI):**
- PP on employee profile → S8 list mode → Request feedback… → pick 3 colleagues → submit → campaign detail shows 3 recipients in audience, draft status, Activate available
- Complete assignee action items → completion table updates → subject S8 still has no auto-added records
- Direct manager who is also PM on subject → Request feedback… visible in list mode

### Review Findings

- [x] [Review][Patch] D13 union role rank — document `ReportingLine`/`PP` gate against single `audience.role` label [`Boundaries & Constraints`]
- [x] [Review][Patch] Exclude FullAccess from entry gate; add Ask First for PO override [`Boundaries & Constraints`]
- [x] [Review][Patch] Specify i18n default templates + empty link/dueDate defaults [`Boundaries & Constraints`]
- [x] [Review][Patch] Dialog in-flight dismiss guard + success toast key [`Boundaries & Constraints`, `RequestFeedbackDialog.tsx`]
- [x] [Review][Patch] Invalidate campaigns list cache after orchestration [`useRequestFeedbackFlow.ts`]
- [x] [Review][Patch] Expand matrix: metadata validation, create failure, loading/error, dedupe, dual role, FullAccess, Self/Colleague [`I/O & Edge-Case Matrix`]
- [x] [Review][Patch] Add schema unit test task + verification command [`Tasks & Acceptance`, `Verification`]
- [x] [Review][Patch] Canonical matrix intro + Code Map normative note (parity with 11.2) [`I/O & Edge-Case Matrix`, `Code Map`]
- [x] [Review][Patch] AC for ReportingLine+ProjectLine dual relationship [`Acceptance Criteria`]
- [x] [Review][Defer] Paginated colleague search beyond 100-row employee list — deferred, matches Epic 10 audience builder scope
- [x] [Review][Defer] FullAccess HR admin entry — deferred pending PO; Ask First documents override path

#### Code review (2026-09-08)

- [x] [Review][Patch] Lazy-mount `RequestFeedbackDialog` to avoid fetching employees list on every S8 profile view [`FeedbackSection.tsx`]
- [x] [Review][Patch] Import `campaignsListQueryKey` via `hooks/data/useCampaignsData` not `@/api/hooks` [`useRequestFeedbackFlow.ts`]
- [x] [Review][Patch] Assert create/audience request payloads in happy-path e2e [`request-feedback-campaign.spec.ts`]
- [x] [Review][Patch] E2e for FullAccess role gate [`request-feedback-campaign.spec.ts`]
- [x] [Review][Patch] E2e for audience PUT failure → navigate to draft detail [`request-feedback-campaign.spec.ts`]
- [x] [Review][Patch] Scope campaign detail stub to `GET` so it does not swallow audience `PUT` [`request-feedback-campaign.spec.ts`]
- [x] [Review][Defer] E2e for employees-list fetch error inline state [`RequestFeedbackDialog.tsx`] — deferred, picker error path covered manually; network stub ordering fragile

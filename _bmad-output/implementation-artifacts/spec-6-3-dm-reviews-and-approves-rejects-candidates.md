---
title: 'DM Reviews and Approves/Rejects Candidates'
type: 'feature'
created: '2026-09-09'
status: 'done'
review_loop_iteration: 1
baseline_commit: '41abb76707fa7c67aff4ec78b9f841f77cbb0da2'
story_key: '6-3-dm-reviews-and-approves-rejects-candidates'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-6-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-6-2-fulfil-a-request-with-internal-or-external-candidates.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/decisions.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Proposals submitted in 6.2 sit permanently at `proposed` — the reviewing DM cannot even open the request (`GET /:id` is gated to the routed UM only), let alone approve or reject a candidate, so `pending_dm_review` is a dead end.

**Approach:** Let `GET /:id` admit the resolved `reviewingDmId` as well as the routed UM; add a decide endpoint that transitions a proposal to `approved`/`rejected` (rejection and reversal-from-approved both require a written reason) gated by headcount remaining; surface each internal candidate's existing shared link (created at 6.2 submit) to the DM for review instead of a direct profile.

## Boundaries & Constraints

**Always:**
- `ResourcingProposalStatus` gains `approved`, `rejected` (alongside `proposed`); `ResourcingProposal` gains nullable `decisionReason` text.
- `POST /resourcing/requests/:id/proposals/:proposalId/decide` — body `{ decision: 'approved'|'rejected', reason?: string }`, `decision` validated against exactly these two values (400 on anything else); gated by `APPROVE_REJECT_CANDIDATES` permission (already seeded on Delivery Manager) **and** `viewerEmployeeId === request.reviewingDmId` (403 `NOT_REVIEWING_DM` otherwise); request must be `pending_dm_review` (409 otherwise). The loaded proposal must belong to `:id` — 404 if `proposalId` resolves under a different request (closes an IDOR-shaped gap: a DM authorized on one request must not be able to decide a proposal that lives under a different one they don't review).
- **Controller-level gate on `GET /:id` must widen too, not just the service check below.** The route is currently gated by `assertCanFulfilResourcing` (`FULFIL_RESOURCING_REQUESTS`) alone — a permission the reviewing DM does not hold (only `APPROVE_REJECT_CANDIDATES`, seeded on Delivery Manager, not `FULFIL_RESOURCING_REQUESTS`, seeded on Unit Manager). Without widening this controller-level check to accept either permission, the DM is 403'd before the service's `assertRouted`-OR-`reviewingDmId` check is ever reached, making the entire reviewing-DM feature unreachable. `POST .../decide` gets its own new, correct gate (`APPROVE_REJECT_CANDIDATES` above) and needs no change here.
- Rejecting a `proposed` row, or reversing an `approved` row to `rejected`, requires a non-empty trimmed `reason` (400 otherwise, max 2000 chars — mirrors `peopleForceCandidateUrl`'s existing bound). Approving requires the proposal be currently `proposed` (400 if already decided — see `APPROVE_ALREADY_DECIDED` matrix row).
- Headcount gate: approve is blocked (409) when the request's live count of `approved` proposals is already `>= headcount`; reversing an approval frees its slot immediately (count is recomputed live, never cached). **Both the headcount recount and the proposal's status transition must happen inside one DB transaction with a conditional update (`UPDATE ... WHERE status = 'proposed'`, checking rows-affected) rather than a plain read-then-write** — otherwise two concurrent `decide` calls (two approvals racing the same free slot, or an approve/reject race on the same row) can each pass their pre-write check before either commits, over-filling headcount or silently discarding one decision.
- `rejected` is terminal — any further `decide` call on it returns 409.
- `getDetail`: extend the existing UM `assertRouted` check with `OR viewerEmployeeId === request.reviewingDmId`, so the reviewing DM can load `GET /:id` for their assigned request (closes a 6.2 gap — today only the UM can reach this route). This is the service-level half of the fix; the controller-level gate above must widen too, or the DM never reaches this code path.
- Proposal read DTO adds `decisionReason`. For internal candidates **only when the viewer is the reviewing DM**, it also adds `sharedLinkToken`, resolved via:
  ```
  prisma.sharedLink.findFirst({
    where: {
      subjectEmployeeId: candidateEmployeeId,
      recipientEmployeeId: viewerEmployeeId,
      revokedAt: null,
      expiresAt: { gt: now },
    },
    orderBy: { createdAt: 'desc' },
  })
  ```
  Reuses the link 6.2's submit flow already creates; no new write path. **Known limitation, accept as-is:** `SharedLink` carries no linkage back to a specific resourcing proposal/request, so if the same internal candidate is ever proposed on two different requests reviewed by the same DM, this lookup returns whichever link is most recently created for *either* request — the token/expiry shown could belong to the other request. Not worth a schema change for this story; document rather than fix.
  Detail DTO separately adds `approvedCount`, and a server-computed `viewerIsReviewingDm: boolean` (`viewerEmployeeId === request.reviewingDmId`) — **required** because the frontend has no other source for its own `viewerEmployeeId` to perform that comparison itself (no endpoint or hook in the frontend codebase currently exposes the signed-in user's employee id); the boolean must do the comparison server-side and hand the frontend a ready-to-branch-on flag, the same pattern already used for `expectedCompBand`/`sharedLinkToken`.
- Fix a 6.2 shipped defect blocking this story: `useResourcingFulfilData.submitRequest`'s shared-link creation call must pass `expiresInHours: 168` (the DTO's hard max) instead of the default 24h, so the link plausibly survives to review time — the closest available stand-in for "until the request is decided."
- Frontend reviewing-DM view of `/resourcing/:requestId` (gated on the detail DTO's `viewerIsReviewingDm` flag — see below, not a client-side `viewerEmployeeId` comparison the frontend has no way to perform — independent of the existing UM view): Approve/Reject per `proposed` row, Reverse-to-rejected per `approved` row, a required-reason dialog (primary action disabled until text entered) for reject/reverse, and a "Review candidate profile" link to `/shared-links/:sharedLinkToken` for internal candidates (a clear fallback state when `sharedLinkToken` is null). External candidates keep the existing PeopleForce link unchanged.
- Add `PERMISSION_KEYS.APPROVE_REJECT_CANDIDATES` to the `/resourcing` and `/resourcing/:requestId` entries in `route-permissions.ts`.

**Ask First:** If bootcamp seed data has no `pending_dm_review` request with a resolvable `reviewingDmId` to manually verify the reviewing-DM view against, flag to the human rather than inventing new seed fixtures beyond what 6.1/6.2 already establish. Same for any `pending_dm_review` request whose internal candidates' shared links (created under 6.2's pre-fix 24h default) have already expired by the time this story is verified — flag rather than silently re-issuing links or fabricating fresher seed data.

**Never:** Explicit DM "close request" (`CLOSE_RESOURCING_REQUESTS` is seeded but unowned by any epic-6 story — tracked in `deferred-work.md`); S15 profile section / full proposal-attempt history (Story 6.4); headcount editing or its shrink-under-water guard (no edit endpoint exists yet); departed-UM/DM backstop reassignment (D18 — no story owns it, PO sign-off defers it); un-rejecting a `rejected` proposal; writing `ProjectAssignment` on approval.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| APPROVE_HAPPY | Reviewing DM, proposal `proposed`, remaining headcount > 0 | Proposal → `approved`, `approvedCount` +1 | — |
| REJECT_HAPPY | Reviewing DM, proposal `proposed`, reason given | Proposal → `rejected`, reason stored | — |
| REJECT_NO_REASON | Reject/reverse with empty reason | — | 400 |
| APPROVE_HEADCOUNT_FULL | `approvedCount` already `== headcount` | — | 409 |
| REVERSE_APPROVAL | Approved proposal, reason given | Proposal → `rejected`, slot freed, `approvedCount` −1 | — |
| DECIDE_ON_REJECTED | Proposal already `rejected` | — | 409 |
| APPROVE_ALREADY_DECIDED | Approve a proposal already `approved` | — | 400 |
| DECIDE_WRONG_REQUEST | `proposalId` resolves under a different request than `:id` | — | 404 |
| INVALID_DECISION_VALUE | `decision` is neither `'approved'` nor `'rejected'` | — | 400 |
| NOT_REVIEWING_DM | Viewer ≠ `reviewingDmId` | — | 403 |
| REQUEST_NOT_PENDING | `request.status` is `open` | — | 409 |
| REVIEWING_DM_UNRESOLVED | `reviewingDmId` is `null` (6.2 couldn't resolve one at submit) | Request permanently undecidable — no viewer can equal `null` | Accepted gap, not fixed this story — see Design Notes |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — extend `ResourcingProposalStatus`; add `decisionReason` to `ResourcingProposal`; migration
- `services/backend/src/modules/resourcing/resourcing.controller.ts:78-85` (`getDetail`) — **widen its permission gate to accept `FULFIL_RESOURCING_REQUESTS` OR `APPROVE_REJECT_CANDIDATES`**, not `assertCanFulfilResourcing` alone (verified: today it 403s any Delivery Manager, who holds only `APPROVE_REJECT_CANDIDATES`); `:87-98` — add `decide` route beside `createProposal`, gated by its own `assertCanApproveRejectCandidates` (`APPROVE_REJECT_CANDIDATES` only)
- `services/backend/src/modules/resourcing/resourcing.service.ts:160-192` (`getDetail`), `:250-278` (`submit` — pattern to mirror for the new method), `:338-349` (`assertRouted` — pattern for the new reviewing-DM check) — `decide()` wraps its status/headcount check-and-transition in one `prisma.$transaction` using a conditional update (rows-affected check) to close the concurrent-approve and concurrent-decide races; also verifies the loaded proposal's `requestId` matches `:id`
- `services/backend/src/modules/resourcing/dto/create-resourcing-proposal.dto.ts` — sibling `decide-resourcing-proposal.dto.ts` (`decision` via `@IsIn(['approved','rejected'])`, `reason` via `@MaxLength(2000)`)
- `services/backend/src/modules/resourcing/entities/resourcing-proposal.entity.ts` — add `decisionReason`, `sharedLinkToken`
- `services/backend/src/modules/resourcing/entities/resourcing-request-detail.entity.ts` — add `approvedCount`, `viewerIsReviewingDm`
- `services/backend/src/modules/resourcing/resourcing.swagger.ts` — `SwaggerDecideResourcingProposal`
- `services/backend/src/modules/resourcing/__tests__/resourcing.service.spec.ts`, `services/backend/test/resourcing.e2e-spec.ts` — matrix rows
- `services/backend/src/modules/contracts/permission-keys.ts:13` — `APPROVE_REJECT_CANDIDATES` already defined; `seed.functional-roles.ts:34` already grants it to DM
- `services/frontend/src/hooks/data/useResourcingData.ts:91-113` (`submitRequest` — add `expiresInHours: 168`), add `decideProposal`
- `services/frontend/src/api/hooks/useResourcingMutations.ts` — `useDecideResourcingProposal`
- `services/frontend/src/pages/ResourcingDetailPage/hooks/useResourcingDetailPage.ts`, `.tsx` — reviewing-DM branch
- `services/frontend/src/pages/ResourcingDetailPage/components/ProposalList/ProposalList.tsx` — Approve/Reject/Reverse actions, shared-link/profile link
- `services/frontend/src/pages/ResourcingDetailPage/components/` — new `DecisionReasonDialog` (mirrors required-reason pattern)
- `services/frontend/src/types/resourcing.ts` — extend `status`, add `decisionReason`, `sharedLinkToken`, `approvedCount`, `viewerIsReviewingDm`
- `services/frontend/src/router/route-permissions.ts:30-42` — add `APPROVE_REJECT_CANDIDATES`
- `services/frontend/src/locales/en/translation.json` (`resourcing.detail.*`) — decision/reason/link i18n keys

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — enum + `decisionReason`
- [x] `services/backend/src/modules/resourcing/resourcing.service.ts` — `decide()` method (auth, proposal-belongs-to-request check, status/headcount validation inside one transaction with a conditional update, transition); widen `getDetail` auth; add `sharedLinkToken`/`approvedCount`/`viewerIsReviewingDm` to DTOs
- [x] `services/backend/src/modules/resourcing/resourcing.controller.ts` — **widen `GET /:id`'s permission gate** to accept `FULFIL_RESOURCING_REQUESTS` OR `APPROVE_REJECT_CANDIDATES`; new `decide` route gated by `APPROVE_REJECT_CANDIDATES` + `dto/decide-resourcing-proposal.dto.ts` (`decision` enum-validated, `reason` max-length-bound) + swagger
- [x] `services/backend/test/resourcing.e2e-spec.ts` + `__tests__/resourcing.service.spec.ts` — matrix + AC coverage
- [x] `services/frontend/src/hooks/data/useResourcingData.ts` — fix submit's `expiresInHours`; add `decideProposal`
- [x] `services/frontend/src/api/hooks/useResourcingMutations.ts` + `types/resourcing.ts` — decide mutation + types
- [x] `services/frontend/src/pages/ResourcingDetailPage/` — reviewing-DM UI (approve/reject/reverse, reason dialog, candidate profile link)
- [x] `services/frontend/src/router/route-permissions.ts` + `locales/en/translation.json` — permission + i18n

**Acceptance Criteria:**
- Given I am the DM reviewing a request with a proposed internal candidate I don't hold standing access to, when I open the candidate's card, then I'm offered the shared-link review flow instead of a direct profile, and I can approve after reviewing it
- Given I am reviewing a proposed candidate, when I choose to reject without entering a reason, then the rejection is blocked; entering a reason succeeds and each candidate's decision on a multi-candidate request is recorded independently
- Given a request's headcount is already fully approved, when I try to approve another candidate, then the action is blocked until I reverse an existing approval
- Given I previously approved a candidate, when I reverse that approval with a reason, then it becomes rejected and the freed slot allows a new approval

### Review Findings

- [x] [Review][Patch] DM pending-review inbox — added `GET /pending-review` and `/resourcing` list section for `approve_reject_candidates` holders [`resourcing.service.ts`, `ResourcingPage.tsx`]
- [x] [Review][Patch] Headcount UX — show `approvedCount` of `headcount`, disable Approve when full, headcount-specific error toast [`ProposalList.tsx`, `ResourcingDetailPage.tsx`, `useResourcingMutations.ts`]
- [x] [Review][Patch] Decision reason max length — client `maxLength={2000}` and confirm guard [`DecisionReasonDialog.tsx`]
- [x] [Review][Patch] Dual-role candidatePool leak — omit pool when viewer is reviewing DM even if also routed UM [`resourcing.service.ts:195-197`]
- [x] [Review][Patch] External candidate null URL — no `href="#"` fallback [`ProposalList.tsx`]
- [x] [Review][Patch] Stale detail entity JSDoc — updated for 6.3 widened auth [`resourcing-request-detail.entity.ts`]
- [x] [Review][Patch] Misleading getDetail 403 message — generic unauthorized wording [`resourcing.service.ts`]
- [x] [Review][Patch] Verification gaps — reverse-without-reason unit+e2e, reason max-length e2e, pending-review list e2e, dual-role candidatePool unit [`resourcing.service.spec.ts`, `resourcing.e2e-spec.ts`]
- [x] [Review][Defer] Prisma migration for 6.3 enum/`decisionReason` — already present at `prisma/migrations/20260909100724_story_6_3_decide_proposal_status/`; false positive — deferred, pre-existing
- [x] [Review][Defer] Concurrent headcount-race e2e under real DB contention — covered by transactional unit tests; full parallel e2e disproportionate — deferred, pre-existing
- [x] [Review][Defer] Frontend Playwright coverage for reviewing-DM UI — no component-test harness in frontend; manual/bootcamp verification — deferred, pre-existing
- [x] [Review][Defer] Partial shared-link creation on submit failure — 6.2 orchestration pattern; atomic rollback out of 6.3 scope — deferred, pre-existing
- [x] [Review][Defer] Frontend unit test for `expiresInHours: 168` — no vitest in frontend; backend consume path covered — deferred, pre-existing

## Design Notes

`getDetail`'s widened auth means a UM and the reviewing DM now share one endpoint; `sharedLinkToken`/`candidatePool`/`viewerIsReviewingDm` stay viewer-scoped fields (null/false for whichever role doesn't apply) rather than splitting into two endpoints — mirrors how `expectedCompBand` is already conditionally included in the 6.1/6.2 read DTOs. `viewerIsReviewingDm` exists specifically because the frontend has no independent way to know its own `viewerEmployeeId`; the backend must do that comparison and hand over a boolean.

**Two accepted gaps, deliberately not fixed this story:** (1) a request whose `reviewingDmId` resolved to `null` at 6.2 submit time becomes permanently undecidable once `pending_dm_review` — no viewer can equal `null` (see `REVIEWING_DM_UNRESOLVED` matrix row); full backstop reassignment for a departed/unresolvable DM is D18/out-of-scope, so this is the same class of gap, just surfacing one story earlier. (2) `sharedLinkToken` resolution has no linkage back to a specific resourcing proposal, so the same internal candidate proposed on two requests reviewed by the same DM can show either request's link. Both are cheap-to-explain, expensive-to-fix-properly limitations; flag rather than silently patch around them with unscoped schema changes.

**Module-boundary trade-off, accepted:** `resourcing.service` querying `prisma.sharedLink` directly is a read-only reach into the access module's schema, which the epic's "resourcing depends only on `contracts`/`registry`" rule and 6.2's "resourcing does not import `access`" boundary would otherwise forbid. No `DepartmentDirectory`-style contract exists for shared links, and adding one is disproportionate to a single `findFirst` read. Accepted as a pragmatic exception for this story rather than introducing a new C-token contract.

## Verification

**Commands:**
- `cd services/backend && npm run lint && npm test && npm run test:e2e -- resourcing.e2e-spec.ts` — expected: pass
- `cd services/frontend && npm run typecheck && npm run lint` — expected: pass

**Manual checks (if no CLI):**
- Bootcamp DM account opens an assigned `pending_dm_review` request, reviews an internal candidate via the shared-link flow, approves one candidate and rejects another with a reason, then reverses the approval and confirms the slot frees up

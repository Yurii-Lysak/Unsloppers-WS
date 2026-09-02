---
title: 'Shared Link Expiry, Logging and Revocation'
type: 'feature'
created: '2026-09-02'
status: 'done'
review_loop_iteration: 1
baseline_commit: '1df0c3f06c1edc340f04ec0a79cb909b05620a64'
backend_baseline_commit: '1df0c3f06c1edc340f04ec0a79cb909b05620a64'
frontend_baseline_commit: '8f54ac4f2c4b677377f71af8f0c5e0acd2de2d83'
story_key: '1-12-shared-link-expiry-logging-and-revocation'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-11-generate-a-shareable-profile-link.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/interface-contracts.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/docs/project-requirements-v2.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/services/backend/.claude/rules/nest-e2e.md'
  - '{project-root}/services/frontend/.claude/rules/react-pages.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.11 persists `expiresAt` (default +24h) and `revokedAt` but does not enforce expiry, log access attempts, or support revocation. Managers cannot time-bound or audit shared profile exposure, and inactive links still return profiles.

**Approach:** Enforce link lifecycle on every consumption attempt, record each attempt via C10 `RelationshipJournal` (`shared_link_access`), expose revoke/list/log APIs gated by current relationship access over the subject, and complete UX-DR12 in the Shared Link Manager (expiry picker, active links, revoke, access log).

## Boundaries & Constraints

**Always:**
- **Story 1.11 create gate unchanged:** only viewers holding **Reporting-line, Project-line, or PP** access over the subject may create links (spec-1-11). Full-access-only holders remain rejected at create.
- **Expiry enforcement:** before profile assembly, reject when `clock.now() >= expiresAt` or `revokedAt IS NOT NULL`. Boundary is **inclusive at `expiresAt`** — at the exact expiry instant the link is inactive. Default remains +24h when create omits custom duration.
- **Configurable expiry at create:** optional `expiresInHours` on `POST /employees/:employeeId/shared-links` — integer **1–168** (1 hour to 7 days). Omitted → 24. Reject non-integer values (e.g. `1.5`) with **400**. Compute `expiresAt = clock.now() + expiresInHours × 3_600_000 ms` via injected `Clock`, never `Date.now()` directly. Create response shape stays `{ token: string; url: string }` (spec-1-11).
- **Consume pipeline order (normative):** (1) require authenticated session → **401** when absent; (2) validate token format → **400**; (3) load link row by token → **404** when unknown; (4) lifecycle check (`expired` / `revoked`) → journal `denied` + **404** uniform message; (5) recipient check → journal `denied` + **403**; (6) D14 re-clamp + profile assembly. Lifecycle runs **before** recipient check so inactive links always return **404**, even for a non-recipient viewer.
- **Uniform inactive-link responses:** expired, revoked, and unknown token all return **404** with the same message (`Shared link not found`). Malformed token stays **400** (Story 1.11). Wrong recipient on an **active** link stays **403** (authenticated but not named recipient). Unauthenticated stays **401** before any DB lookup or journal write.
- **Journal on every resolved attempt:** when step (3) finds a DB link row (valid token format + known token), write one C10 `shared_link_access` entry **before** returning the HTTP response — granted or denied. No journal for malformed tokens, unknown tokens, or unauthenticated requests (auth fails at step 1). `after` payload (JSON):
  ```typescript
  {
    sharedLinkId: string;
    outcome: 'granted' | 'denied';
    denialReason?: 'expired' | 'revoked' | 'wrong_recipient';
    originIp: string | null; // first X-Forwarded-For hop when present, else request.ip
    recipientEmployeeId: string | null; // viewer employee id when authenticated, else null
  }
  ```
  `actorId` = authenticated viewer's employee id (consume always requires auth before journal-eligible steps). `subjectId` = link's `subjectEmployeeId`. `before` = `null`.
- **C10 minimal implementation:** add `relationship-journal.contract.ts` (C10) + `RelationshipJournalEntry` Prisma model (`actorEmployeeId`, `subjectEmployeeId`, `kind`, `before` Json?, `after` Json, `createdAt`). Add index on `(subjectEmployeeId, kind, createdAt DESC)` for `readFor`. Implement `record` + `readFor(subjectId, readerId)` in `access` module. Other journal kinds (`manager`, `full_access_grant`, …) are out of scope — only `shared_link_access` in this story.
- **`readFor` gate:** `readFor(subjectId, readerId)` returns entries when C1 resolves `readerId` to `ReportingLine`, `ProjectLine`, `PP`, or `FullAccess` over `subjectId` (architecture spine AD-16 / PRD §4.8 backstop). Filter returned entries to `kind === 'shared_link_access'` for the link-access-log API.
- **Manage API gate (list / revoke / access-log):** same gate as `readFor` — Reporting-line, Project-line, PP, or Full-access over `:employeeId` (the subject). Not limited to original creator.
- **Revocation:** `POST /employees/:employeeId/shared-links/:linkId/revoke` sets `revokedAt = clock.now()` idempotently (second call is **200** with `{ revoked: true }` if already revoked). Return **404** when `:linkId` is unknown or does not belong to `:employeeId`. Return **403** when the viewer lacks manage gate access.
- **List active links:** `GET /employees/:employeeId/shared-links` returns **200** `{ links: LinkSummary[] }` for that subject where `revokedAt IS NULL` and `expiresAt > clock.now()`, newest `createdAt` first. Empty array when none active. **403** when viewer lacks manage gate; **404** when subject employee not found. `LinkSummary`:
  ```typescript
  {
    id: string;
    recipient: { id: string; displayName: string };
    creator: { id: string; displayName: string };
    expiresAt: string; // ISO
    createdAt: string;
    sectionIds: SectionId[];
  }
  ```
- **Access log API:** `GET /employees/:employeeId/shared-links/:linkId/access-log` → **200** `{ entries: Array<{ accessedAt: string; outcome: 'granted' | 'denied'; denialReason?: 'expired' | 'revoked' | 'wrong_recipient'; originIp: string | null; recipientEmployeeId: string | null }> }` sorted `accessedAt` desc. Return journal rows where `kind === 'shared_link_access'` **and** `after.sharedLinkId === :linkId`. **404** when link unknown or not for subject; **403** when viewer lacks manage gate.
- **D14 unchanged:** live creator-access re-clamp still runs after lifecycle checks pass; empty sections after clamp returns **200** with `sections: {}` (Story 1.11).
- **Frontend (UX-DR12 completion):** extend `SharedLinkManagerDialog` with expiry select (default **24h** plus presets **1h / 24h / 48h / 168h**), tabs or sections for **Create** vs **Manage**: active-link table (recipient, expires, sections summary, Revoke button), expandable per-link access log from access-log API. `SharedLinkViewPage` shows generic unavailable copy for 404/403 consume errors — no "expired" vs "revoked" wording. i18n keys only.

**Ask First:** none identified.

**Never:** resourcing auto-generated links or request-bound lifetime (Epic 6); departure-cascade bulk revoke (CAP-14); journal kinds other than `shared_link_access`; C9 `OrgRelationshipWriter`; full-access grant/revoke UI; changing Story 1.11 create/consume matrix rules or D14 behavior; pagination on list or access-log APIs (bootcamp scale).

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Default expiry | Create with no `expiresInHours`; consume at T+24h+1ms | Denied; journal `denied/expired` | 404 uniform message |
| Exact expiry boundary | Consume at `clock.now() === expiresAt` | Denied; journal `denied/expired` | 404 uniform message |
| Custom expiry | `expiresInHours: 48`; consume at T+47h | Profile returned | N/A |
| Expiry out of range | `expiresInHours: 0` or `200` | — | 400 |
| Non-integer expiry | `expiresInHours: 1.5` or `"48"` | — | 400 |
| Revoke active link | Manager revokes; recipient consumes | Denied; journal `denied/revoked` | 404 uniform |
| Idempotent revoke | Second revoke on same link | `revokedAt` unchanged | 200 `{ revoked: true }` |
| PP revokes creator's link | PP holds access; creator departed | Revoke succeeds | N/A |
| Full-access revoke | Full-access holder POST revoke | Revoke succeeds | N/A |
| Colleague revoke | Colleague POST revoke | — | 403 |
| Wrong recipient on expired link | Expired link; non-recipient opens | Denied; journal `denied/expired` (lifecycle before recipient) | 404 uniform |
| Wrong recipient consume | Valid active link; other employee opens | Denied; journal `denied/wrong_recipient` | 403 |
| Granted consume | Named recipient; active link | Profile + journal `granted` | N/A |
| List filters inactive | One active, one expired, one revoked | List returns only active | N/A |
| List empty | No active links for subject | `{ links: [] }` | 200 |
| Revoke wrong subject | `:linkId` exists but for another employee | — | 404 |
| Access log scoped | Two links; log API for one linkId | Only entries for that `sharedLinkId` | N/A |
| Access log | Two consume attempts (one denied) | Log API returns both entries newest-first | N/A |
| Unknown token | Random valid-format token | No journal (no subject row) | 404 uniform |
| Unauthenticated consume | No session | No DB lookup; no journal | 401 |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma:405-423` — `SharedLink` already has `expiresAt`, `revokedAt`; add `RelationshipJournalEntry` model + migration; index `(subjectEmployeeId, kind, createdAt)`
- `services/backend/src/modules/contracts/relationship-journal.contract.ts` (new) — C10 abstract token per `interface-contracts.md` (`record`, `readFor(subjectId, readerId)`)
- `services/backend/src/modules/contracts/contracts.module.ts` — export C10 DI token
- `services/backend/src/modules/access/relationship-journal.service.ts` (new) — C10 implementation; `readFor` uses `AccessResolver.resolveAudience` manage gate
- `services/backend/src/modules/access/shared-link.service.ts:54-61,104-120,123-144` — extend `SharedLinkRecord` with `expiresAt`/`revokedAt`; add `assertConsumable`, `revokeLink`, `listActiveForSubject`, `resolveExpiresAt(dto)`; enforce lifecycle in consume path per pipeline order
- `services/backend/src/modules/access/shared-link.controller.ts:43-53` — consume: pipeline order → journal → assembly; add list/revoke/access-log routes; extract client IP helper
- `services/backend/src/modules/access/dto/create-shared-link.dto.ts` — add optional `expiresInHours` with `@IsInt()` `@Min(1)` `@Max(168)` validators
- `services/backend/src/modules/access/shared-link.swagger.ts:22` — remove "not enforced until 1.12" note; document list/revoke/access-log endpoints and response shapes
- `services/backend/src/clock/clock.service.ts` — reuse `Clock` / `FixedClock` in unit tests for expiry boundaries
- `services/backend/src/modules/access/__tests__/shared-link.service.spec.ts` — expiry, revoke, journal record, pipeline-order cases
- `services/backend/src/modules/access/__tests__/relationship-journal.service.spec.ts` (new) — `readFor` gate + `shared_link_access` filter
- `services/backend/test/shared-links.e2e-spec.ts` — expiry denial, revoke denial, uniform 404, journal/list/revoke happy paths, wrong-subject linkId 404
- `services/frontend/src/api/services/shared-link.service.ts` — add list/revoke/access-log client methods + types
- `services/frontend/src/api/hooks/useSharedLinks.ts` — queries/mutations for manage surface
- `services/frontend/src/pages/EmployeeProfilePage/components/SharedLinkManagerDialog/` — expiry UI, active-link table, revoke + log (extend hook + dialog)
- `services/frontend/src/pages/SharedLinkViewPage/SharedLinkViewPage.tsx:58-61` — generic error copy for inactive links
- `services/frontend/e2e/shared-link-create.spec.ts` — stub new manage APIs or extend flow smoke test

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — `RelationshipJournalEntry` model + index — C10 persistence
- [x] `services/backend/src/modules/contracts/relationship-journal.contract.ts` + `access.module.ts` — C10 DI token wired to real impl in `access.module.ts`
- [x] `services/backend/src/modules/access/relationship-journal.service.ts` — `record` + gated `readFor` for `shared_link_access`
- [x] `services/backend/src/modules/access/shared-link.service.ts` — expiry math, `assertConsumable`, revoke/list helpers, journal hook on consume per pipeline order
- [x] `services/backend/src/modules/access/shared-link.controller.ts` + DTOs — list, revoke, access-log routes; IP capture on consume
- [x] `services/backend/src/modules/access/dto/create-shared-link.dto.ts` — `expiresInHours` optional field
- [x] `services/backend/src/modules/access/shared-link.swagger.ts` — document new endpoints; remove deferred-expiry note
- [x] `services/backend/src/modules/access/__tests__/shared-link.service.spec.ts` + `relationship-journal.service.spec.ts` — matrix unit coverage
- [x] `services/backend/test/shared-links.e2e-spec.ts` — expiry, revoke, uniform 404, journal rows, linkId scoping
- [x] `services/frontend/src/api/services/shared-link.service.ts` + `useSharedLinks.ts` — manage API wiring
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/SharedLinkManagerDialog/` — UX-DR12 expiry + manage UI
- [x] `services/frontend/src/pages/SharedLinkViewPage/SharedLinkViewPage.tsx` — uniform inactive-link error copy
- [x] `services/frontend/e2e/shared-link-create.spec.ts` — manage-tab smoke with stubbed APIs

**Acceptance Criteria:**
- Given a share link is created with no custom expiry, when 24 hours elapse and the named recipient opens it, then access is denied with the same 404 response shape as a revoked or unknown link, and a journal entry records the denied attempt with timestamp and origin IP
- Given a share link is created with `expiresInHours: 48`, when the recipient opens it within 48 hours, then the profile is returned and a journal entry records a granted access with timestamp and origin IP
- Given an active share link, when a viewer holding Reporting-line, Project-line, PP, or Full-access over the subject revokes it and the recipient next opens it, then access is denied with the uniform inactive-link response and the attempt is logged as denied
- Given a viewer holding Reporting-line, Project-line, PP, or Full-access over the subject, when they open the Shared Link Manager manage view, then they see active links for that employee with expiry time and can revoke a link and view its access log

## Design Notes

**Journal vs audit:** C10 is intentionally narrow — only relationship/access-changing events. Do not write to a general audit table.

**404 uniformity:** Log internally with real `denialReason`; never expose expired vs revoked vs missing in HTTP body or frontend copy. Lifecycle-before-recipient ordering prevents inactive links from leaking existence via 403.

**Consume pipeline:** Auth and token validation are pre-journal. Only DB-resolved links produce `shared_link_access` rows. Revoke actions do not journal separately — the next denied consume records `denied/revoked`.

**Access-log scoping:** Filter by `after.sharedLinkId` in journal JSON; do not return other links' entries for the same subject.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run depcruise` — PASS (verified)
- `cd services/backend && npm test -- --testPathPatterns="shared-link|relationship-journal"` — PASS, 29/29 (verified)
- `cd services/backend && npm run test:e2e -- shared-links` — NOT RUN: Node 22.x cannot execute backend e2e (`@nestjs/schedule` ESM / Jest require — same pre-existing blocker as Story 1.11). Run on CI or Node ≥24.9 before merge.
- `cd services/frontend && npm run build && npm run lint` — PASS (verified)
- `cd services/frontend && npx playwright test shared-link-create` — PASS, 3/3 (verified)

### Review Findings

- [x] [Review][Patch] Journal `after.recipientEmployeeId` typed `string` but documented as nullable — Fixed: `string | null` in payload and access-log response.
- [x] [Review][Patch] `actorId` rule self-contradictory (recipient fallback vs null) — Fixed: always viewer employee id; consume requires auth before journal.
- [x] [Review][Patch] Missing normative consume pipeline order — Fixed: auth → format → resolve → lifecycle → recipient → assembly; lifecycle before recipient.
- [x] [Review][Patch] Wrong recipient on expired link unspecified — Fixed: matrix row + lifecycle-first rule → 404 uniform.
- [x] [Review][Patch] `readFor`/manage gate vs acceptance criteria — Fixed: Full-access included in acceptance criteria; gate aligned to architecture spine.
- [x] [Review][Patch] Access-log API missing `sharedLinkId` filter — Fixed: filter `after.sharedLinkId === :linkId`.
- [x] [Review][Patch] Missing HTTP shapes for list/revoke — Fixed: `{ links: [] }`, `{ revoked: true }`, status codes documented.
- [x] [Review][Patch] Expiry boundary ambiguity — Fixed: inclusive at `expiresAt`; matrix row for exact boundary.
- [x] [Review][Patch] Non-integer `expiresInHours` unhandled — Fixed: `@IsInt()` + matrix row.
- [x] [Review][Patch] Unauthenticated matrix row conflated resolve vs auth — Fixed: no DB lookup, no journal, 401.
- [x] [Review][Patch] Missing matrix rows (Full-access revoke, empty list, wrong-subject linkId, log scoping) — Fixed.
- [x] [Review][Patch] Story 1.11 create gate not restated — Fixed: explicit inheritance bullet.
- [x] [Review][Structure] Matrix lacked canonical-source note — Fixed: matches spec-1-11 pattern.
- [x] [Review][Structure] Code map missing `contracts.module.ts`, journal spec file, swagger task — Fixed.
- [x] [Review][Prose] "Gate via revoke/list rights" vague — Fixed: renamed to manage API gate with explicit roles.

- [x] [Review][Decision] Full-access manage gate is unreachable — Resolved: implement `FullAccess` in C1 now (grant persistence + resolver path + frontend manage gate).

- [x] [Review][Patch] Implement FullAccess grant persistence and C1 resolution — `FullAccessGrant` model + migration; `AccessResolverService.hasActiveFullAccessGrant`; frontend `LINK_MANAGE_ROLES` separate from create gate.

- [x] [Review][Patch] Journal records `granted` before profile assembly — `shared-link.controller.ts` records journal after successful assembly.

- [x] [Review][Patch] Access log renders raw API enum strings — i18n keys for outcomes and denial reasons in `SharedLinkManagerDialog.tsx`.

- [x] [Review][Patch] Frontend conflates create and manage authorization — `EmployeeProfilePage.tsx` uses separate `canCreateSharedLink` / `canManageSharedLinks`.

- [x] [Review][Patch] Manage tab hides list and access-log API errors — `useSharedLinkManagerDialog.ts` surfaces `isLinksError` / `isAccessLogError`.

- [x] [Review][Patch] `extractClientIp` ignores array `X-Forwarded-For` — handles `string[]` header form.

- [x] [Review][Patch] String `expiresInHours` may bypass 400 rejection — `@IsStrictInteger()` rejects non-number JSON values.

- [x] [Review][Patch] Remove dead `assertRecipient` method — removed from `shared-link.service.ts`.

- [x] [Review][Patch] Spec matrix lacks automated test coverage — added unit tests for list filter/access-log scoping; e2e for FA holder, wrong-recipient journal, revoked journal, invalid expiry.

- [x] [Review][Patch] Manage tab shows raw section IDs — uses translated `sectionLabel` in manage list.

- [x] [Review][Patch] Revoke does not invalidate access-log cache — `useRevokeSharedLink` invalidates access-log query key.

- [x] [Review][Defer] Backend e2e suite not executed locally — `npm run test:e2e -- shared-links` blocked by pre-existing Node 22 / Jest ESM issue (noted in Verification); run on CI or Node ≥24.9 before merge.

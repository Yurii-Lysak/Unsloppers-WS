---
title: 'Generate a Shareable Profile Link'
type: 'feature'
created: '2026-09-02'
status: 'done'
review_loop_iteration: 2
baseline_commit: 'ab595a3c77fff2b2549ddf91d65ec8c5536ed3ca'
backend_baseline_commit: 'ab595a3c77fff2b2549ddf91d65ec8c5536ed3ca'
frontend_baseline_commit: '0045cda5cf6d5c56d84dd269569b4d2b2adce7e8'
story_key: '1-11-generate-a-shareable-profile-link'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-8-enforce-the-colleague-whitelist-everywhere.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/services/backend/.claude/rules/nest-e2e.md'
  - '{project-root}/services/frontend/.claude/rules/react-pages.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Viewers with Reporting-line, Project-line, or PP access cannot share a scoped, read-only profile view with someone who lacks that access over the subject — blocking flows like a DM reviewing an internal resourcing candidate. `SharedLink` exists only as a type stub; there is no persistence, creation API, consumption route, or Shared Link Manager UI.

**Approach:** Add `SharedLink`/`SharedLinkSection` Prisma models and an `access`-module service that validates section selection (reject `never` sections; `cfg` sections off unless explicitly enabled; S1 on by default), records a named recipient via `recipientEmployeeId`, and returns a copyable URL. Extend profile assembly with a link-token consumption path that builds a synthetic `SharedLink` audience (all grants `'R'`, sections = link intent ∩ creator's live access per D14) and wire the Employee Profile "Share…" dialog plus an authenticated `SharedLinkView` page.

## Boundaries & Constraints

**Always:**
- Only a viewer holding **Reporting-line, Project-line, or PP** access over the subject may create a link — gate via existing C1 before accepting `POST`. **Full-access** holders without one of those relationship roles over the subject are **not** link creators in this story.
- **Never-share sections** `{S3, S7, S13, S14}` are rejected server-side on create (AD-5: server-side grant enforcement) — not merely omitted from the UI checkbox list.
- **Cfg defaults** follow `test/support/access-matrix.ts`: S1 `sharedLinkDefault: 'on'`; S2/S4/S5/S6/S8/S9/S10/S11/S12/S15/S16 default `'off'` — unselected cfg sections must not appear in the consumed profile.
- At **create**, intersect the requested section set with the creator's live C1 grants — reject any section where the creator's grant is `'none'` (400). Path `:employeeId` is the sole subject source; ignore any body subject field.
- **Create request shape (normative):**
  ```typescript
  POST /employees/:employeeId/shared-links
  {
    recipientEmployeeId: string; // UUID of an active employee with a linked user account
    sections?: SectionId[];      // explicit cfg enables; S1 implied on when omitted; dedupe server-side
  }
  ```
- **Create response shape (normative):**
  ```typescript
  { token: string; url: string } // url = SPA path `/shared-links/{token}`; absolute copy URL optional
  ```
- **Consume response shape (normative):** identical to spec-1-6 `GET /employees/:employeeId/profile` profile DTO — `employeeId`, `displayName`, `audience: { role: 'SharedLink', sections: Record<SectionId, SectionAccessLevel> }`, and `sections` with only D14-clamped keys, each at `accessLevel: 'R'`. Apply spec-1-6 assembler normalization (`status: 'unavailable'` when a granted section's provider fails).
- Consumption requires an **authenticated session** whose employee is the link's named `recipientEmployeeId`; any other viewer gets **403**. **401** when unauthenticated.
- On every consumption, re-clamp each enabled section against `resolveAudience(creatorId, subjectId)` (D14); cap every granted section at `'R'` (access-model Rule 9). When D14 clamps every section to `'none'`, return **200** with `sections: {}` and `audience.sections` all `'none'` — not 403.
- Reuse `ProfileAssemblerService` + existing section providers — no duplicate assembly logic. Pass the **creator's** resolved audience tier into providers so **Project-line field narrowing** applies at assembly (no S2/S3; S5 CV+certs only; S7 PM visibility flags). When cfg **S16** is enabled, apply Story 1.10 per-field visibility rules in `CustomFieldsSectionProvider` — never emit fields the creator could not see.
- **Recipient validation at create:** `recipientEmployeeId` must reference an existing, non-dismissed `Employee` with a linked `User` (active account). Reject unknown UUID (400), dismissed recipient (400), and recipient without user account (400). Reject `recipientEmployeeId === creatorEmployeeId` (400) and `subjectId === creatorEmployeeId` (400) — links are for sharing outward, not self-view.
- Consumption is **only** via `GET /shared-links/:token/profile`. If the named recipient opens `GET /employees/:subjectId/profile` directly, normal C1 applies (typically Colleague view) — the link token does not widen standing access on other routes.
- Frontend: Shared Link Manager is a `Dialog` from the Access Scope Chip "Share…" action (**Reporting-line, Project-line, or PP** viewers only). **1.11 UX scope (partial UX-DR12):** section checkboxes (never sections absent), recipient picker (`GET /employees` id+displayName), create + copy URL — **no** expiry field, active-link list, or revoke controls (Story 1.12). `SharedLinkView` is an authenticated route (no nav entry) reusing `profile-sections.tsx` read-only renderers and showing an Access Scope Chip labeled from enabled section names (e.g. "Shared link — sections: Identity, Career timeline").
- i18n: all new copy via `react-i18next` keys.

**Never:**
- Expiry **enforcement**, access **logging**, **revocation**, active-link list UI, revoke actions, indistinguishable expired/revoked error shapes, C10 `RelationshipJournal` `shared_link_access` writes, or expiry/revoke controls in the Shared Link Manager — Story **1.12**.
- Auto-generated resourcing links (Epic 6) — separate story; this story delivers the manual flow only.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path — S1+S9 | Reporting-line manager creates link with `[S1,S9]` for subject B, names DM as recipient | Consumption returns profile with only S1 and S9 keys, all `accessLevel: 'R'`; `audience.role === 'SharedLink'` | N/A |
| PP creator | Assigned PP P creates link for subject B with defaults | 201; consumption works for named recipient | 403 if P lacks PP access over B |
| Cfg off by default | Create with only defaults (S1 on) | Consumed profile has S1 only; S2/S5/S6/S8 absent | N/A |
| Never sections rejected | Create payload includes S3, S7, S13, or S14 | Link not created | 400 Bad Request |
| Section creator lacks | Reporting-line creator POSTs S6 but live grant is `none` (e.g. Colleague actor bypass) | Link not created | 400 Bad Request |
| Duplicate section ids | Create payload lists S9 twice | Dedupe or reject | 400 if rejected |
| Wrong recipient | Authenticated user ≠ named recipient opens token URL | No profile data | 403 Forbidden |
| Unauthenticated consume | No session on `GET /shared-links/:token/profile` | — | 401 |
| Viewer without Employee | Authenticated user, no linked `Employee` row → create or consume | — | 403 before link logic |
| Unknown token | Valid-format token not in DB | — | 404 Not Found |
| Malformed token | Token fails format validation (not base64url / wrong length) | — | 400 Bad Request |
| Unknown recipient at create | `recipientEmployeeId` not found | — | 400 Bad Request |
| Dismissed recipient at create | Recipient employee is dismissed | — | 400 Bad Request |
| Self as recipient | Creator sets `recipientEmployeeId` to own id | — | 400 Bad Request |
| Self as subject | Creator POSTs link for own profile | — | 400 Bad Request |
| Creator access narrows | Link enabled S6; creator later loses S6 via tier change | Next consumption omits S6 (D14 re-clamp) | N/A |
| Creator loses all access | Creator becomes Colleague w.r.t. subject | Consumption returns 200, `sections: {}` | N/A |
| Project-line S5 narrowing | Project-line creator enables S5 | Consumed S5 is CV+certs only per Rule 2 | N/A |
| S16 cfg visibility | Creator enables S16; some fields management-only | Only fields visible to creator emitted | N/A |
| Provider unavailable | Enabled section has no working provider | `{ accessLevel: 'R', status: 'unavailable' }` per spec-1-6 | N/A |
| Non-manager creator | Colleague POSTs create for subject | Rejected | 403 Forbidden |
| Full-access only | Holder has FullAccess but no Reporting/Project/PP role over subject | Rejected | 403 Forbidden |
| Standing profile route | Named recipient opens `GET /employees/:subjectId/profile` | Normal C1 profile (typically Colleague) — not link scope | 401; 403 no Employee; 404 |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` — add `SharedLink` (token, subjectId, creatorEmployeeId, recipientEmployeeId, `expiresAt` default +24h via service, `revokedAt` nullable for 1-12) + `SharedLinkSection` (linkId, sectionId); migration required
- `services/backend/test/support/access-matrix.ts` — extend (or create) with `sharedLinkDefault` and `never` cells per `access-model.md`; drive unit validation tests
- `services/backend/src/modules/access/shared-link.service.ts` (new) — validate sections against `ACCESS_MATRIX` never/cfg rules; intersect with creator C1 grants; token format validation; generate URL-safe token
- `services/backend/src/modules/access/shared-link.controller.ts` (new) — `POST /employees/:employeeId/shared-links` (create), `GET /shared-links/:token/profile` (consume)
- `services/backend/src/modules/access/profile-assembler.service.ts:52-67` — add overload or sibling `assembleProfileViaSharedLink(token, viewerEmployeeId)` that builds synthetic `ResolvedAudience` instead of standard C1
- `services/backend/src/modules/access/access-resolver.service.ts:182-223` — unchanged for normal paths; shared-link path bypasses `resolveAudience` for the recipient
- `services/backend/src/modules/access/section-access-gate.service.ts:70-72` — reuse `listGrantedSections` to cap creator-offerable sections at create time
- `services/backend/src/modules/directory/custom-fields-section.provider.ts` — S16 per-field visibility when audience is `SharedLink` (Story 1.10 rules)
- `services/backend/src/modules/access/profile.controller.ts:32-46` — mirror auth/`resolveEmployeeId` pattern for consumption controller
- `services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx:58-67` — extend Access Scope Chip with "Share…" action (Reporting-line, Project-line, or PP only)
- `services/frontend/src/pages/EmployeeProfilePage/components/SharedLinkManagerDialog/` (new) — section checkboxes (never sections absent), recipient select, create + copy URL
- `services/frontend/src/pages/SharedLinkViewPage/` (new) — authenticated route; fetch `GET /shared-links/:token/profile`; reuse `profile-sections.tsx`; Access Scope Chip with enabled section labels
- `services/frontend/src/router/index.tsx` — add `/shared-links/:token` child route under `ProtectedRoute`
- `services/frontend/e2e/shared/selectors.ts:36-39` — wire `shared-link-view` test id on the new page

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/test/support/access-matrix.ts` — `sharedLinkDefault` + `never` cells aligned to `access-model.md` — validation source of truth
- [x] `services/backend/prisma/schema.prisma` + migration -- `SharedLink`/`SharedLinkSection` models -- persistence foundation
- [x] `services/backend/src/modules/access/shared-link.service.ts` -- section validation, token generation, D14 re-clamp helper, recipient/subject guards -- core business rules
- [x] `services/backend/src/modules/access/shared-link.controller.ts` + `access.module.ts` -- create + consume HTTP surface + Swagger notes on deferred `expiresAt` enforcement
- [x] `services/backend/src/modules/access/profile-assembler.service.ts` -- link-scoped assembly entry point -- reuse provider pipeline with synthetic audience and creator-tier narrowing
- [x] `services/backend/src/modules/access/__tests__/shared-link.service.spec.ts` -- never-section rejection (incl. S14), cfg defaults, D14 re-clamp, create-time grant intersection unit cases
- [x] `services/backend/test/shared-links.e2e-spec.ts` -- matrix rows: S1+S9 consumption, never-section 400 (S7+S14), wrong recipient 403, PP creator, unknown token 404
- [x] `services/frontend/src/api/hooks/useSharedLinks.ts` + service -- TanStack Query mutations/queries
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/SharedLinkManagerDialog/` -- creation UI from chip (partial UX-DR12 scope)
- [x] `services/frontend/src/pages/SharedLinkViewPage/` + `router/index.tsx` -- consumption surface + Access Scope Chip
- [x] `services/frontend/e2e/shared-link-create.spec.ts` -- Reporting-line manager creates link, recipient opens URL, sees only enabled sections and scope chip

**Acceptance Criteria:**
- Given a reporting-line manager holds access to employee B and opens the share-link creation flow, when they enable S1 and S9 only leaving S2/S5/S6/S8 at their default, then the link exposes S1 and S9 read-only and S2/S5/S6/S8 are excluded because they were never explicitly enabled
- Given a viewer with Reporting-line, Project-line, or PP access is configuring a share link for employee B, when they or a direct API call attempts to include S3, S7, S13, or S14, then the request is rejected server-side — these sections are never offered nor acceptable under any configuration
- Given an assigned PP holds access to employee B, when the PP creates a share link naming a colleague as recipient, then the recipient can consume the link and sees only the enabled sections at read-only access
- Given a share link names employee D as recipient, when any other authenticated employee opens the token URL, then they receive 403 and no profile sections
- Given a share link is consumed successfully, when the recipient views `SharedLinkView`, then an Access Scope Chip shows the shared-link role and the names of enabled sections

## Design Notes

**D14 re-clamp:** see Boundaries Always — store link sections as creator intent only; intersect with live `resolveAudience(creatorId, subjectId)` on every consumption.

**URL shape:** `/shared-links/{token}` on the SPA; API `GET /api/v1/shared-links/{token}/profile`. Token is opaque (32-byte base64url), unique index in Prisma; reject malformed tokens before DB lookup.

`expiresAt` is written at creation (default +24h via `Clock` / `clock.service`) but **not enforced** until Story 1-12 — document this in Swagger so QA does not treat pre-expiry denial as a regression.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run depcruise` -- PASS (verified)
- `cd services/backend && npm test -- --testPathPatterns=shared-link` -- PASS, 10/10 (verified)
- `cd services/backend && npm run test:e2e -- shared-links` -- NOT RUN: Node 22.23.2 cannot execute backend e2e (`@nestjs/schedule` ESM / Jest require — same pre-existing blocker as Story 1.10). Run on CI or Node ≥24.9 before merge.
- `cd services/frontend && npm run build && npm run lint` -- PASS (verified)
- `cd services/frontend && npx playwright test shared-link-create` -- PASS, 2/2 (verified)

### Review Findings

- [x] [Review][Decision] Dismissed-recipient rejection (400) — Resolved: added `EmploymentStatus` enum (`active` | `dismissed`) on `Employee` with migration `20260902143000_add_employee_employment_status`; `assertValidRecipient` rejects `dismissed` with 400.

- [x] [Review][Patch] `profile-assembler.service.spec.ts` broken after `SharedLinkService` injection [`services/backend/src/modules/access/__tests__/profile-assembler.service.spec.ts:45`] — Fixed: added `SharedLinkService` mock; all 11 assembler tests pass.

- [x] [Review][Patch] `shared-link-matrix.ts` duplicates `access-matrix.ts` instead of deriving rules [`services/backend/src/modules/access/shared-link-matrix.ts:1`] — Fixed: added `test/shared-link-matrix.sync.spec.ts` asserting never/default/cfg alignment with `ACCESS_MATRIX`.

- [x] [Review][Patch] `CreateSharedLinkDto.sections` lacks per-element validators [`services/backend/src/modules/access/dto/create-shared-link.dto.ts:16`] — Fixed: `@IsString({ each: true })` + `@IsIn(SECTION_IDS, { each: true })`.

- [x] [Review][Patch] `assembleProfileViaSharedLink` dual-audience path untested [`services/backend/src/modules/access/profile-assembler.service.ts:71`] — Fixed: unit tests verify creator `providerAudience` passed to providers and D14 empty-section response.

- [x] [Review][Patch] Create-time recipient/subject guards untested [`services/backend/src/modules/access/shared-link.service.ts:76`] — Fixed: unit tests for self-subject, self-recipient, unknown recipient, recipient-without-user, dismissed recipient.

- [x] [Review][Patch] D14 consumption narrowing untested at assembly boundary [`services/backend/src/modules/access/shared-link.service.ts:152`] — Fixed: `assembleProfileViaSharedLink returns empty sections when D14 clamps all` test.

- [x] [Review][Patch] Misnamed unit test [`services/backend/src/modules/access/__tests__/shared-link.service.spec.ts:124`] — Fixed: renamed to `createLink_rejectsColleagueCreatorRole`.

- [x] [Review][Patch] FullAccess-only creator rejection untested [`services/backend/src/modules/access/shared-link.service.ts:229`] — Fixed: `createLink_rejectsFullAccessOnlyCreator` unit test.

- [x] [Review][Patch] E2e spec bypasses shared selector catalogue [`services/frontend/e2e/shared-link-create.spec.ts:115`] — Fixed: uses `testIds.sharedLink.view` and `expectSectionAbsent`.

- [x] [Review][Defer] Backend shared-links e2e not run in verification [`services/backend/test/shared-links.e2e-spec.ts`] — deferred, pre-existing Node 22 / `@nestjs/schedule` e2e blocker documented in spec Verification.

- [x] [Review][Defer] Frontend Playwright tests fully stub APIs [`services/frontend/e2e/shared-link-create.spec.ts:59`] — deferred, UI wiring smoke only; no live backend integration in bootcamp test harness.

- [x] [Review][Defer] Project-line S5 narrowing untestable [`services/backend/src/modules/access/profile-assembler.service.ts:111`] — deferred, pre-existing: no registered S5 `SectionProvider` exists yet.

- [x] [Review][Defer] Recipient picker capped at 100 employees [`services/frontend/src/pages/EmployeeProfilePage/components/SharedLinkManagerDialog/hooks/useSharedLinkManagerDialog.ts:25`] — deferred, pre-existing bootcamp scale (24 accounts) makes pagination out of scope for 1.11.

---
title: 'Complete and Cancel Action Items'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'd1d6e92d561a3b532b37fc41ab401ec7cceb0bd5'
story_key: '4-2-complete-and-cancel-action-items'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-4-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-1-manually-create-an-action-item.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/decisions.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 4.1 persists `ActionItem` rows and supports create/list only. Assignees cannot mark items complete; authors cannot cancel with a reason. Terminal lifecycle fields (`completedAt`, `cancelledAt`, `cancelledReason`) exist in schema but are never written or returned.

**Approach:** Add assignee-scoped complete and author-scoped cancel mutations on the existing `action-items` module — mirror 4.1 controller patterns, use `Clock` for timestamps, extend read DTOs, and cover the authorship-survives-access-loss case via a `/me/...` cancel route that does not require live S14 on the assignee.

## Boundaries & Constraints

**Always:**

### Complete

- `POST employees/:employeeId/action-items/:itemId/complete` (no body).
- Pre-checks: active assignee exists (`assertSubjectEmployeeExists` — inactive/dismissed `:employeeId` → **404**); `sectionGate.requireSection(viewer, employeeId, 'S14')` (minimum `R`; Self assignee retains `R` on own profile); item belongs to `employeeId` (`assigneeId === employeeId`) or **404**; unknown `itemId` after scope check → **404**; `viewerEmployeeId === item.assigneeId` or **403** (author, manager, PP with `RW`, PM with ProjectLine `R`, colleague — none may complete on behalf of assignee); `status === 'open'` or **409** (`ConflictException`, body may include current `status`).
- Applies equally to `source: 'manual'` and `source: 'campaign'` open items.
- On success: **200** with `ActionItemReadEntity`; `status: 'completed'`, `completedAt: clock.now()`. Terminal state; items are never reopenable.
- Concurrent complete + cancel: first writer wins; loser gets **409** on terminal transition.

### Cancel

- `POST me/authored-action-items/:itemId/cancel` with body `{ reason }`.
- Pre-checks: viewer has employee record or **403**; malformed `:itemId` → **400** (`ParseUUIDPipe`); item exists with `authorId === viewerEmployeeId` or **404** (assignee who is not author → **404**, not **403**); **no** live S14 gate on assignee (authorship is historical per `decisions.md` appendix).
- If `status === 'cancelled'`: idempotent no-op — skip reason validation, ignore body, return existing row unchanged (D17; do not overwrite reason).
- If `status === 'completed'`: **409** (`ConflictException`).
- If `status === 'open'`: require trimmed non-empty `reason` (1–2000 chars) or **400** (missing key, `null`, whitespace-only, non-string, or >2000 chars); set `status: 'cancelled'`, `cancelledAt: clock.now()`, `cancelledReason`.
- On success: **200** with `ActionItemReadEntity`.
- Author cancel remains allowed when the assignee is dismissed or the author has lost live Manager/PP access; assignee complete on an inactive `:employeeId` still **404**s via `assertSubjectEmployeeExists`.

### Read / list

- Extend `ActionItemReadEntity` / parallel route responses with optional `completedAt?`, `cancelledAt?`, `cancelledReason?` (ISO date-time strings). `toReadDto` / `toAuthoredDto` / `toContractDto` must populate when present.
- S14 provider and `GET employees/:employeeId/action-items` continue returning **all statuses** (completed/cancelled remain visible with terminal fields). `GET /me/authored-action-items` stays **open-only** — completed/cancelled drop off automatically.

### Clock / e2e

- Inject `Clock` into `ActionItemsService`; never `new Date()` / `Date.now()` for mutation timestamps.
- E2e: use existing `FixedClock` harness; assert `completedAt`/`cancelledAt` against fixed instant.

**Ask First:**

- If `EmployeeProfilePage` is still absent in `services/frontend`, ship backend + e2e only (same deferral as 4.1); wire complete checkbox and cancel dialog (required-reason pattern per EXPERIENCE.md) when profile infrastructure merges; destructive Toast on save failure.

**Never:**

- Overdue derivation/UI (4.3); campaign-sourced creation (4.4); departure-hook auto-cancel (AD-18 / CAP-14 — no `departure-hook` registrant exists yet); reopening terminal items; author completing on behalf of assignee; notifications; dashboard widgets (Epic 12).

## I/O & Edge-Case Matrix

Delta scenarios not fully spelled out in Boundaries above:

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PP completes for report | PP with S14 `RW` POST complete | — | 403 |
| PM completes for project talent | PM with ProjectLine S14 `R` POST complete | — | 403 |
| Colleague completes | Colleague (S14 `none`) POST complete | — | 403 at S14 gate |
| Assignee cancels | Assignee (not author) POST cancel | — | 404 |
| Cancel idempotent + new reason | Item already `cancelled`; body has different `reason` | 200, stored reason unchanged | N/A |
| Cancel empty body | POST `{}` or missing `reason` key on open item | — | 400 |
| Cancel reason >2000 | Open item; reason 2001 chars | — | 400 |
| Cancel no employee record | Authenticated user without `Employee` row | — | 403 |
| Complete inactive assignee | `:employeeId` dismissed/inactive | — | 404 |
| Concurrent complete + cancel | Assignee completes while author cancels same open item | One terminal state; first wins | 409 for loser |

</frozen-after-approval>

## Code Map & Tasks

| Status | Path | Scope |
|--------|------|-------|
| [x] | `services/backend/src/modules/action-items/dto/cancel-action-item.dto.ts` (new) | `reason`: `@Transform(trim)`, `@IsNotEmpty`, `@MaxLength(2000)` — validated only when `status === 'open'` |
| [x] | `services/backend/src/modules/action-items/action-items.service.ts` | `completeActionItem`, `cancelActionItem`, `findItemForAssignee`, `findItemForAuthor`; inject `Clock`; extend `toReadDto`/`toAuthoredDto`/`toContractDto` with terminal fields |
| [x] | `services/backend/src/modules/action-items/action-items.controller.ts` | POST complete + POST cancel; reuse `resolveViewerEmployeeId`, `assertSubjectEmployeeExists` |
| [x] | `services/backend/src/modules/action-items/entities/action-item.entity.ts` | Optional `completedAt?`, `cancelledAt?`, `cancelledReason?` on read entity |
| [x] | `services/backend/src/modules/action-items/action-items.swagger.ts` | Decorators for new operations; document 200/400/403/404/409 responses |
| [x] | `services/backend/src/modules/action-items/action-items.module.ts` | Ensure `ClockModule` available (global via `app.module.ts`) |
| [x] | `services/backend/src/modules/contracts/action-item-creation.contract.ts` | Add `cancelledAt?`, `cancelledReason?` to `ActionItemDto` (symmetric with existing `completedAt?`) |
| [x] | `services/backend/src/modules/action-items/__tests__/action-items.service.spec.ts` | Unit tests: complete/cancel guards, idempotent cancel (skip validation), 409 paths |
| [x] | `services/backend/src/modules/action-items/__tests__/action-items-section.provider.spec.ts` | Terminal fields present on S14 provider read mapping |
| [x] | `services/backend/test/support/access-matrix.ts` | Extend S14 matrix with complete-deny qualifiers (manager, PP, author-not-assignee) |
| [x] | `services/backend/test/action-items.e2e-spec.ts` | Assignee complete, denial matrix, cancel validation, access-drift cancel (mirror authored-list drift e2e in `action-items.e2e-spec.ts`), terminal 409s, DTO fields on mutation + subsequent GET/profile |
| [ ] | `services/frontend/.../ActionItemsSection.tsx` (when profile page exists) | Complete checkbox + cancel dialog; `data-testid`: `action-item-complete`, `action-item-cancel` |

**Read-only references:**

- `services/backend/prisma/schema.prisma` — `ActionItem` already has terminal columns; no migration expected.
- `services/backend/src/clock/clock.service.ts` — `Clock` abstraction for timestamps.
- `services/backend/test/support/access-matrix.ts` — Self S14 qualifier documents assignee-only complete.
- `_bmad-output/specs/spec-people-management-platform/decisions.md` — D17 idempotency + historical authorship (appendix).

## Acceptance Criteria

- **AC-1:** Given I am the assignee of an open item (manual or campaign source), when I POST complete, then the response is **200**, `status` is `completed`, `completedAt` is set to now, and only I — not the author, manager, PP, PM, or colleague — can do this
- **AC-2:** Given I am the author of an open item, when I cancel without a valid reason, then the request is rejected with **400**; when I provide a trimmed reason (1–2000 chars), then the response is **200**, `status` is `cancelled`, the reason is stored, and this works even if I have since lost live Manager/PP access to the assignee
- **AC-3:** Given an item is `completed` or `cancelled`, when any client attempts to complete or cancel it again (except idempotent cancel on an already-cancelled item), then the mutation is rejected with **409** and the item is never reopened
- **AC-4:** Given an item is cancelled, when the same author POSTs cancel again (with any or no body), then the response is **200**, the existing cancellation reason is preserved, and no validation error occurs (idempotent no-op per D17)
- **AC-5:** Given I authored an open item that appears on `GET /me/authored-action-items`, when the assignee completes it or I cancel it, then the item no longer appears on that authored list
- **AC-6:** Given an item was completed or cancelled, when I GET `employees/:employeeId/action-items` or the assignee's profile S14, then the item is returned with the correct terminal fields (`completedAt` or `cancelledAt` + `cancelledReason`)

### Review Findings

- [x] [Review][Patch] Idempotent cancel DTO validation [`cancel-action-item.dto.ts`, `action-items.service.ts`] — skip `reason` validation when `status === 'cancelled'` before idempotent return
- [x] [Review][Patch] access-matrix complete-deny rows [`test/support/access-matrix.ts`] — extend S14 matrix for manager/PP/author-not-assignee complete denial
- [x] [Review][Patch] Section provider terminal fields [`action-items-section.provider.spec.ts`] — assert `completedAt`/`cancelledAt`/`cancelledReason` on S14 read path
- [x] [Review][Patch] Authored-list lifecycle e2e [`action-items.e2e-spec.ts`] — AC-5: terminal items absent from `/me/authored-action-items`
- [x] [Review][Patch] Post-mutation read surfaces e2e [`action-items.e2e-spec.ts`] — AC-6: terminal fields on parallel GET and profile S14 after complete/cancel
- [ ] [Review][Defer] Frontend `ActionItemsSection.tsx` — deferred per Ask First when `EmployeeProfilePage` absent
- [x] [Review][Patch] Optimistic locking on terminal transitions [`action-items.service.ts`:108-156] — `updateMany` with `status: 'open'` guard; 409 on race loser
- [x] [Review][Patch] E2e concurrent complete+cancel race [`action-items.e2e-spec.ts`] — `returns one terminal state when complete and cancel race`
- [x] [Review][Patch] E2e AC-6 cancel read-back on GET/profile [`action-items.e2e-spec.ts`] — assignee list + S14 after cancel in drift test
- [x] [Review][Patch] E2e colleague POST complete 403 [`action-items.e2e-spec.ts`] — extended colleague denial test
- [x] [Review][Patch] E2e campaign-source complete [`action-items.e2e-spec.ts`] — `lets the assignee complete a campaign-sourced open item`
- [x] [Review][Patch] E2e cancel without employee record 403 [`action-items.e2e-spec.ts`] — `returns 403 when canceling without an employee record`

## Verification

**Commands:**

- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm run lint` — expected: no errors
- `cd services/backend && npm test -- --testPathPatterns=action-items` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- action-items.e2e-spec` — expected: e2e pass (Postgres up)

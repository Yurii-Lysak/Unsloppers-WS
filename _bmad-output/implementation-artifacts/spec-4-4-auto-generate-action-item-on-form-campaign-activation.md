---
title: 'Auto-Generate Action Item on Form Campaign Activation'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'd94b313eaad325d80249a0dfcfe87e5b1e43dd7e'
story_key: '4-4-auto-generate-action-item-on-form-campaign-activation'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-4-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-1-manually-create-an-action-item.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-3-overdue-highlighting.md'
  - '{project-root}/_bmad-output/planning-artifacts/ARCHITECTURE-SPINE.md'
  - '{project-root}/docs/project-requirements.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C6 exposes only single-item `createActionItem`. Epic 10 campaign activation must atomically create exactly one `source: 'campaign'` item per frozen recipient (`epics.md` FR-18; PRD §4.5 and §4.12 point 3), but there is no bulk contract, no `(campaignId, assigneeId)` uniqueness guard, and no transaction hook for Epic 10 to participate in the same DB transaction as audience freeze. Builds on Story 4.1's `ActionItem` model and is the integration surface for Epic 10 Story 10.3.

**Approach:** Extend C6 with `createCampaignActionItems` — validate campaign payload once, verify author and all assignees are active employees, insert all rows in one transaction via `createMany` (optionally using a caller-supplied write context), and return contract DTOs. No campaign entity, REST routes, or UI in this story; Epic 10 Story 10.3 calls C6 directly.

## Boundaries & Constraints

**Always:**

- **C6 bulk signature:** `createCampaignActionItems({ campaignId, authorId, title, description?, dueDate, link?, assigneeIds: string[] }, tx?: ActionItemWriteContext) → ActionItemDto[]`.
- **Campaign existence guard (runs first):** if `ActionItem` already has **any** row for `campaignId` (any status — `open`, `completed`, or `cancelled`), reject with **409 Conflict** (`campaign already has action items`). Run this check inside the same transaction as inserts when `tx` is supplied. On concurrent activations, map Prisma unique-violation (`P2002` on `campaignId_assigneeId`) to the same **409** — never surface as **500**.
- **Empty audience:** when the guard passes and `assigneeIds` is empty → return `[]` without writes.
- **Input UUIDs:** `campaignId`, `authorId`, and every element of `assigneeIds` must be non-empty UUID strings; malformed values → **400** before any DB access.
- **Per-item shape:** each row has `source: 'campaign'`, `status: 'open'`, `campaignId` set, `completedAt`/`cancelledAt` null; field normalization reuses `normalizeActionItemFields` (same rules as manual create / Story 4.1).
- **Author:** `authorId` must resolve to an active `Employee` (`employmentStatus: 'active'`). Unknown or non-active author → **400** before assignee validation; zero rows persisted.
- **Assignees:** every id in `assigneeIds` must resolve to a distinct active employee. Reject duplicate ids in the input array with **400**. Reject any unknown or non-active assignee with **400** and body `{ message: string, invalidAssigneeIds: string[] }` (entire batch aborts — no partial inserts).
- **Sender in audience:** when `authorId` also appears in `assigneeIds`, allow it — the sender receives their own campaign action item like any other recipient.
- **Atomicity:** all inserts for one call commit or roll back together. Use `createMany` for the batch (no per-row round trips). When `tx` is supplied, use it; when omitted, wrap in `prisma.$transaction`. Bootcamp scale: audiences up to several hundred recipients must succeed in one call without a separate chunking story.
- **`ActionItemWriteContext`:** minimal ORM-agnostic transaction surface (same tx-delegate pattern as C4's `TimelineEventWriteContext`):
  ```typescript
  export interface ActionItemWriteContext {
    actionItem: {
      count(args: { where: { campaignId: string } }): Promise<number>;
      createMany(args: { data: Array<{ ... }> }): Promise<{ count: number }>;
      findMany(args: {
        where: { campaignId: string };
        orderBy: Array<
          | { dueDate: 'asc' }
          | { assigneeId: 'asc' }
          | { createdAt: 'asc' }
        >;
      }): Promise<CampaignPersistedActionItem[]>;
    };
    employee: {
      findFirst(args: {
        where: { id: string; employmentStatus: 'active' };
      }): Promise<{ id: string } | null>;
      findMany(args: {
        where: { id: { in: string[] }; employmentStatus: 'active' };
      }): Promise<Array<{ id: string }>>;
    };
  }
  ```
- **Uniqueness:** add `@@unique([campaignId, assigneeId])` on `ActionItem`. PostgreSQL treats `NULL` as distinct in unique constraints, so manual items (`campaignId` null) are unaffected without a partial index.
- **Single-item C6 guard:** `createActionItem` with `source: 'campaign'` → **400** (`use createCampaignActionItems for campaign activation`). Campaign items are bulk-only.
- **DTO return:** map through existing `toContractDto` (includes `isOverdue` per Story 4.3). Sort the **C6 return array only** `dueDate` asc, `assigneeId` asc, `createdAt` asc — S14 reads keep Story 4.1 ordering (`dueDate` asc, `createdAt` asc).
- **Stub (`ActionItemCreationStub`):** in-memory map keyed by `campaignId`; first bulk call for a campaign persists-shaped DTOs, second call for same `campaignId` throws `ConflictException`; when `tx` is passed, delegate to it if present else no-op writes (matches C4 stub pattern) so Epic 10 can integrate before the real service lands.
- **Tests:** unit coverage for validation, tx participation, conflict (including empty audience + existing rows), concurrent P2002 mapping, and author failure; e2e via `ActionItemsService` injected from the test app (no new HTTP routes).

**Ask First:**

- If Epic 10 lands `Campaign` model before this story ships, add the Prisma FK from `ActionItem.campaignId` → `Campaign.id` in the same migration — otherwise keep `campaignId` as opaque UUID (Story 4.1 deferral).

**Never:**

- Campaign draft/audience/activation HTTP or UI (Epic 10 Stories 10.1–10.3); per-person completion table (10.4); campaign-sender S14 exception wiring (Rule 7 — Epic 10); manual-create REST/auth changes beyond the `source: 'campaign'` guard; departure-hook auto-cancel (AD-18); frontend work; changes to Story 4.2 completion logic (campaign items use the existing path unchanged).

## I/O & Edge-Case Matrix

Delta scenarios not fully spelled out in Boundaries above:

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy path | 50 distinct active assignee ids, valid campaign fields | 50 persisted rows via `createMany`; DTO length 50 | N/A |
| Empty audience (new campaign) | `assigneeIds: []`, no rows for `campaignId` | `[]`; no DB writes | N/A |
| Empty audience (existing rows) | `assigneeIds: []`, `campaignId` already has rows | — | 409 before empty early return |
| Inactive author | `authorId` dismissed or unknown | — | 400; zero rows persisted |
| Assignee error body | One dismissed assignee among many | — | 400 with `invalidAssigneeIds` listing only bad ids |
| Malformed UUID | `campaignId` or assignee id not a UUID | — | 400 before DB |
| Field boundaries | `description` at 2001 chars or `link` at 2049 chars | — | 400 (shared normalizer) |
| Concurrent activation | Two parallel calls for same `campaignId` | Exactly one succeeds | Second gets 409 (pre-check or P2002) |
| Tx rollback | Caller passes `tx`; `createMany` throws | — | Caller transaction rolls back; no orphan rows |
| Overdue at create | `dueDate` before clock today | Each DTO `isOverdue: true` | N/A |
| Sender in audience | `authorId` ∈ `assigneeIds` | N items including one for sender | N/A |

## Code Map & Tasks

| Path | Scope | Done |
|------|-------|------|
| `services/backend/src/modules/contracts/action-item-creation.contract.ts` | Add `CreateCampaignActionItemsInput`, `ActionItemWriteContext`, abstract `createCampaignActionItems(...)`; document Epic 10 as consumer; update `ARCHITECTURE-SPINE.md` AD-2 C6 entry to match | [x] |
| `services/backend/src/modules/contracts/stubs/action-item-creation.stub.ts` | Stub bulk method per Boundaries stub semantics | [x] |
| `services/backend/prisma/schema.prisma` | `ActionItem`: `@@unique([campaignId, assigneeId])` + migration | [x] |
| `services/backend/src/modules/action-items/action-items.service.ts` | `createCampaignActionItems` + reject `source: 'campaign'` on `createActionItem` | [x] |
| `services/backend/src/modules/action-items/action-item-input.ts` | Read-only; reuse `normalizeActionItemFields` | — |
| `services/backend/src/modules/contracts/timeline-event-writer.contract.ts` | Reference: `TimelineEventWriteContext` tx-delegate pattern | — |
| `services/backend/src/modules/action-items/action-items.controller.ts:150` | Reference: active-employee check pattern | — |
| `services/backend/src/modules/action-items/__tests__/action-items.service.spec.ts` | Unit: happy path, empty new campaign, empty existing campaign 409, duplicate assignee, dismissed assignee, inactive author, re-activation 409, tx client used, P2002 → 409 | [x] |
| `services/backend/test/action-items.e2e-spec.ts` | E2e: bulk create 3 recipients → S14 + profile show `source: 'campaign'`; conflict on second call; complete one item via Story 4.2 path (regression) | [x] |
| `_bmad-output/planning-artifacts/epics.md` | Reference: Story 4.4 + 10.3 AC alignment | — |

</frozen-after-approval>

## Acceptance Criteria

- **AC-1:** Given a frozen audience of N active employee ids and valid campaign fields, when `createCampaignActionItems` runs, then exactly N action items are persisted with `source: 'campaign'`, `status: 'open'`, the campaign title/description/dueDate/link, and `authorId` equal to the campaign sender
- **AC-2:** Given activation input where any assignee is missing or not `active`, when `createCampaignActionItems` runs, then the call fails with **400**, `invalidAssigneeIds` names the bad ids, and no action items are created for that `campaignId`
- **AC-2b:** Given `authorId` is missing or not `active`, when `createCampaignActionItems` runs, then the call fails with **400** and no action items are created
- **AC-3:** Given any action item already exists for `campaignId` C (any status), when `createCampaignActionItems` is invoked for C (including with `assigneeIds: []`), then the call fails with **409** and the existing item set is unchanged
- **AC-4:** Given a caller supplies `ActionItemWriteContext` from an open transaction and a mid-batch failure occurs, when the caller rolls back, then no action items from that attempt remain
- **AC-5:** Given campaign-sourced items created with a past `dueDate`, when returned via C6 or `GET /employees/:id/action-items`, then `isOverdue` follows Story 4.3 derivation (no campaign-specific logic)

## Verification

**Commands:**

- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm run lint` — expected: no errors
- `cd services/backend && npm test -- --testPathPatterns=action-items` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- action-items.e2e-spec` — expected: e2e pass (Postgres up)

### Review Findings

- [x] [Review][Patch] `ActionItemWriteContext` missing `findMany` but service casts past contract [`action-item-creation.contract.ts`]
- [x] [Review][Patch] Author/assignee validation uses `this.prisma` when caller supplies `tx` [`action-items.service.ts`]
- [x] [Review][Patch] Stub blocks second call after empty audience; real service allows follow-up with assignees [`action-item-creation.stub.ts`]
- [x] [Review][Patch] Stub `createActionItem` does not reject `source: 'campaign'` [`action-item-creation.stub.ts`]
- [x] [Review][Patch] `P2002` handler treats any unique violation as campaign conflict [`action-items.service.ts`]
- [x] [Review][Patch] Migration file exists but is untracked — must be committed with schema change [`prisma/migrations/20260903150000_action_item_campaign_assignee_unique/migration.sql`]
- [x] [Review][Patch] No unit tests for malformed UUID rejection on bulk path [`action-items.service.spec.ts`]
- [x] [Review][Patch] No unit tests for field boundary validation on `createCampaignActionItems` [`action-items.service.spec.ts`]
- [x] [Review][Patch] No test for sender-in-audience (`authorId` ∈ `assigneeIds`) [`action-items.service.spec.ts`]
- [x] [Review][Patch] No assertion that omitted-`tx` path wraps in `$transaction` [`action-items.service.spec.ts`]
- [x] [Review][Patch] No test pinning C6 return-array sort order [`action-items.service.spec.ts`]
- [x] [Review][Patch] No e2e for AC-3 conflict when existing rows are completed/cancelled [`action-items.e2e-spec.ts`]
- [x] [Review][Patch] No e2e for AC-4 mid-batch `createMany` failure + caller rollback [`action-items.e2e-spec.ts`]
- [x] [Review][Patch] No e2e for bulk-path validation failures (inactive author, `invalidAssigneeIds`, empty+existing 409) [`action-items.e2e-spec.ts`]
- [x] [Review][Patch] E2e does not assert `isOverdue` on GET/S14 response (AC-5 read path) [`action-items.e2e-spec.ts`]
- [x] [Review][Defer] Concurrent activation race e2e against real Postgres unique index — deferred, hard to reproduce reliably in CI
- [x] [Review][Defer] Assignee employment status changing between validation and `createMany` — deferred, inherent race without serializable isolation
- [x] [Review][Defer] Migration preflight for pre-existing duplicate `(campaignId, assigneeId)` rows — deferred, greenfield bootcamp has none
- [x] [Review][Defer] Map `P2003` FK violations to `invalidAssigneeIds` — deferred, narrow window after validation
- [x] [Review][Defer] Error precedence when invalid fields and existing campaign rows both apply — deferred, invalid input should 400 first

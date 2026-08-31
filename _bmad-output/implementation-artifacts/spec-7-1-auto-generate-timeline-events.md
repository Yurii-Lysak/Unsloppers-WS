---
title: 'Auto-Generate Timeline Events'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'e0e9416e1e7a3eac1bc8085547d3f0af1b3ca71e' # services/backend HEAD — implementation happens in that submodule
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-7-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/services/backend/.claude/rules/nest-prisma.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.20 wired the temporal-history Prisma extension to call C4 (`TimelineEventWriter`) on every grade/position/department/employment-type change, but C4 is still a Wave-0 stub that logs and no-ops — no `timeline_events` rows are ever persisted, seed history writes produce an empty timeline, and the AD-7 structural coupling is incomplete.

**Approach:** Add a `@Global()` `timeline` module that implements C4 for real: persist system-generated `TimelineEvent` rows, implement `markSystemWriteSkipped` skip metadata, and extend the C4 contract with an optional transaction client so extension-mediated writes stay atomic (history row + timeline row commit or roll back together). External callers (future timetracker sync) get soft-fail behavior — log the failure for retry, never crash the caller.

## Boundaries & Constraints

**Always:**
- C4 remains the only write path for timeline rows from feature code — the temporal-history extension continues calling `recordTimelineEvent` / `markSystemWriteSkipped`; no service writes `timeline_events` directly except the C4 implementation itself.
- `recordTimelineEvent` inserts a row with `source: 'system'` (or the supplied `source`), storing `oldValue`/`newValue` as JSON (raw typed values, not formatted strings), and `effectiveDate` as date-only UTC.
- When the extension passes a transaction client, C4 uses it — a failure rolls back the enclosing Serializable transaction (close prior row + insert + timeline row).
- When no transaction client is passed (external/integration caller), C4 catches persistence errors, logs structured failure context for retry, and resolves without throwing — graceful degradation per Story 7.1 AC2.
- `markSystemWriteSkipped` updates the existing manual event's skip-metadata field (`systemWriteSkippedAt`); it never creates a separate timeline row.
- `TimelineModule` overrides the `TimelineEventWriter` DI token (mirrors `AccessModule` / `AuthModule` pattern); import it in `AppModule` after `ContractsModule`.
- Follow `nest-prisma.md`: schema change via new migration, commit generated SQL, never hand-edit applied migrations.

**Ask First:** none identified.

**Never:** no S9 read/write API or UI (Story 7.2); no `SectionProvider(S9)` registration yet; no manual-entry write paths; no joining/mentorship/extended-leave caller implementations (their epics own those C4 call sites); no change to the temporal-history extension's close-prior-row / conflict-suppression logic (Story 1.20 frozen); no `EmploymentStatusHistory` timeline coupling (AD-7/AD-18 explicitly exclude departures).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Grade change via extension | Open `GradeHistory` row exists; new `create` with later `effectiveFrom` | Prior row closed, new row inserted, `timeline_events` row with `type: 'grade'`, `source: 'system'`, old/new values, matching `effectiveDate` | N/A |
| First-ever dimension write | No prior history row | New row + timeline event with `oldValue: null` | N/A |
| Manual conflict (existing) | Manual `TimelineEvent` at same `(employeeId, type, effectiveDate, source: 'manual')` | Extension suppresses history write; C4 sets `systemWriteSkippedAt` on manual row; throws `ManualConflictSuppressedError` | Typed error to caller |
| C4 failure inside extension tx | DB constraint violation during `timelineEvent.create` inside tx | Whole Serializable transaction rolls back — no orphan history row | Prisma error propagates |
| External caller, transient DB error | `recordTimelineEvent(...)` called without tx (simulated timetracker path) | Error logged with employeeId/type/effectiveDate payload; promise resolves (caller continues) | No throw |
| Duplicate system event | Second system write with same `(employeeId, type, effectiveDate, source: 'system')` | Unique constraint violation | Inside tx: rollback; external: log and swallow |
| markSystemWriteSkipped | Valid manual event id | `systemWriteSkippedAt` set to parsed ISO timestamp | `NotFoundException` or Prisma P2025 if id missing (inside tx: propagate) |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/contracts/timeline-event-writer.contract.ts` -- C4 abstract class; extend both methods with optional final `tx?: Prisma.TransactionClient` param (Epic 7 owns C4; required for AD-7 atomicity per deferred-work item from Story 1.20).
- `services/backend/src/modules/contracts/stubs/timeline-event-writer.stub.ts` -- update stub signatures to match extended contract (still no-ops; used only if `TimelineModule` absent).
- `services/backend/src/prisma/extensions/temporal-history.extension.ts:307-339` -- passes tx to C4 calls today implicitly via closure; update call sites to pass `tx` explicitly after contract change.
- `services/backend/prisma/schema.prisma:151-166` -- `TimelineEvent` model; add `systemWriteSkippedAt DateTime?` for D2 skip affordance.
- `services/backend/src/modules/timeline/timeline.module.ts` -- new `@Global()` module; `{ provide: TimelineEventWriter, useClass: TimelineEventWriterService }`.
- `services/backend/src/modules/timeline/timeline-event-writer.service.ts` -- new C4 implementation: `recordTimelineEvent` → `timelineEvent.create`; `markSystemWriteSkipped` → `timelineEvent.update`; tx-aware client selection; soft-fail branch when no tx.
- `services/backend/src/app.module.ts:22-29` -- import `TimelineModule` after `ContractsModule`.
- `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- update mock signatures; optionally add one integration case with real `TimelineEventWriterService` instead of mock.
- `services/backend/src/modules/timeline/__tests__/timeline-event-writer.service.spec.ts` -- new unit tests for persistence, tx participation, soft-fail, skip metadata.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/timeline-event-writer.contract.ts` -- add optional `tx?: Prisma.TransactionClient` to both abstract methods -- enables AD-7 same-transaction coupling.
- [x] `services/backend/src/modules/contracts/stubs/timeline-event-writer.stub.ts` -- match new signatures -- keeps stub compilable.
- [x] `services/backend/prisma/schema.prisma` -- add `systemWriteSkippedAt DateTime?` to `TimelineEvent` -- D2 skip metadata storage.
- [x] `services/backend/prisma/migrations/**` -- generate and commit migration for `systemWriteSkippedAt` -- per nest-prisma.md.
- [x] `services/backend/src/modules/timeline/timeline-event-writer.service.ts` -- implement C4 with tx-aware writes and external soft-fail logging -- core Story 7.1 deliverable.
- [x] `services/backend/src/modules/timeline/timeline.module.ts` -- register and export `TimelineEventWriter` override -- DI wiring.
- [x] `services/backend/src/app.module.ts` -- import `TimelineModule` after `ContractsModule` -- ensures real impl wins over stub.
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- pass `tx` into C4 calls at lines 307-310 and 332-339 -- atomic extension path.
- [x] `services/backend/src/modules/timeline/__tests__/timeline-event-writer.service.spec.ts` -- unit-test I/O matrix rows for service layer -- proves persistence and soft-fail.
- [x] `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- add end-to-end case: real `TimelineEventWriterService` + `gradeHistory.create` → row in `timeline_events` -- proves AC1 integration.

### Review Findings

- [x] [Review][Patch] Stale extension docblock still claims manual-conflict runs inside Serializable tx [`temporal-history.extension.ts:14`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L14)
- [x] [Review][Patch] Stale schema comment says C4 is Wave-0 stub and timeline_events never populated [`schema.prisma:146`](../../services/backend/prisma/schema.prisma#L146)
- [x] [Review][Patch] Stale timeline.module comment says contracts binds Wave-0 stub [`timeline.module.ts:7`](../../services/backend/src/modules/timeline/timeline.module.ts#L7)
- [x] [Review][Patch] Normalize external `effectiveDate` to UTC date-only (match extension `isoDate`) [`timeline-event-writer.service.ts:38`](../../services/backend/src/modules/timeline/timeline-event-writer.service.ts#L38)
- [x] [Review][Patch] Add integration test: real C4 failure rolls back history write (duplicate system event at same key) [`timeline-event-writer.integration.spec.ts`](../../services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts)
- [x] [Review][Patch] Add unit test: duplicate system event without tx logs and swallows (soft-fail path) [`timeline-event-writer.service.spec.ts`](../../services/backend/src/modules/timeline/__tests__/timeline-event-writer.service.spec.ts)
- [x] [Review][Defer] Manual-conflict TOCTOU window (pre-tx check vs concurrent manual insert) [`temporal-history.extension.ts:290`] — deferred, documented tradeoff in Spec Change Log
- [x] [Review][Defer] No timetracker/extended-leave C4 caller yet [`timetracker module`] — deferred, Epic 13 scope per spec Never
- [x] [Review][Defer] Seed tests do not assert timeline_events populated after history writes — deferred, follow-up hardening

**Acceptance Criteria:**
- Given an employee currently has grade "Middle" and a write updates it to "Senior" effective 2026-09-01 via `gradeHistory.create` through the extended client, when the change is saved, then a `timeline_events` row exists with `type: 'grade'`, `oldValue: 'Middle'`, `newValue: 'Senior'`, `effectiveDate: 2026-09-01`, and `source: 'system'`.
- Given the timetracker sync (or any external caller) invokes `recordTimelineEvent` without a transaction and persistence fails transiently, when the call completes, then the error is logged with retry context and the promise resolves without throwing — other profile functionality is not blocked.

## Spec Change Log

- **2026-08-31 (implementation):** Moved manual-conflict detection and `markSystemWriteSkipped` outside the Serializable transaction so skip metadata commits when the real C4 writer replaces the Wave-0 stub (in-tx call + throw previously rolled back the update). Pre-transaction out-of-order guard preserved before conflict check per Story 1.20 ordering.

## Suggested Review Order

**C4 contract & DI wiring**

- ORM-agnostic transaction surface keeps contracts Prisma-free while enabling atomic writes
  [`timeline-event-writer.contract.ts:18`](../../services/backend/src/modules/contracts/timeline-event-writer.contract.ts#L18)

- C4 unbound in contracts; timeline module owns the real implementation (mirrors access/auth)
  [`contracts.module.ts:19`](../../services/backend/src/modules/contracts/contracts.module.ts#L19)

- Global module overrides C4; imported before PrismaModule so the extension gets the real writer
  [`timeline.module.ts:15`](../../services/backend/src/modules/timeline/timeline.module.ts#L15)

**Real writer persistence**

- Tx-aware creates; external callers soft-fail with TIMELINE_WRITE_RETRY logging
  [`timeline-event-writer.service.ts:21`](../../services/backend/src/modules/timeline/timeline-event-writer.service.ts#L21)

- Extension passes interactive tx into recordTimelineEvent for history+timeline atomicity
  [`temporal-history.extension.ts:338`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L338)

- Manual conflict + skip metadata run before Serializable tx so updates survive the throw
  [`temporal-history.extension.ts:291`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L291)

**Schema**

- D2 skip affordance column on timeline_events
  [`schema.prisma:160`](../../services/backend/prisma/schema.prisma#L160)

**Tests**

- Unit coverage for persistence, tx participation, and soft-fail logging
  [`timeline-event-writer.service.spec.ts:1`](../../services/backend/src/modules/timeline/__tests__/timeline-event-writer.service.spec.ts#L1)

- Integration proof: gradeHistory.create → timeline_events rows + skip metadata on conflict
  [`timeline-event-writer.integration.spec.ts:1`](../../services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts#L1)

## Design Notes

Internal event `type` strings stay as Story 1.20 defined (`grade`, `position`, `department`, `employmentType`) — the UI maps these to display labels like "grade change" at render time (AD-7). Do not rename types in this story.

`TimelineEventWriterService` selects the Prisma client as `const db = tx ?? this.prisma` (where `this.prisma` is the extended `PrismaService`). When `tx` is provided, use it directly — it is the interactive transaction client from the extension, not the extended outer client.

Soft-fail logging uses Nest `Logger.error` with a stable prefix (`TIMELINE_WRITE_RETRY`) and a JSON-serializable payload — no new retry-queue table in this story; Epic 13 can consume the log contract or add a table later.

## Verification

**Commands:**
- `cd services/backend && nvm use && npx tsc --noEmit` -- expected: PASS
- `cd services/backend && npm run lint` -- expected: PASS, 0 errors
- `cd services/backend && npm run db:migrate` -- expected: new migration applies cleanly
- `cd services/backend && npm test` -- expected: PASS, including new timeline writer tests and updated extension tests

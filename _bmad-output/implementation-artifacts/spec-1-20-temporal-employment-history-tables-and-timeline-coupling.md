---
title: 'Temporal Employment History Tables and Timeline Coupling'
type: 'feature'
created: '2026-08-27'
status: 'done'
review_loop_iteration: 2
baseline_commit: '49de34c16d5d9874c84bc6c77dd722b50e41f8fc' # services/backend HEAD — implementation happens in that submodule
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
  - '{project-root}/services/backend/.claude/rules/nest-prisma.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Grade, position, department, and employment-type currently have no persistence at all (no `Employee` model exists yet, since no prior story created one), and nothing structurally couples a change to those fields with a Career Timeline entry — any future write path could silently skip the timeline record.

**Approach:** Add a minimal `Employee` model (1:1 with `User`) and four effective-dated history tables (`GradeHistory`, `PositionHistory`, `DepartmentHistory`, `EmploymentTypeHistory`), plus a minimal `TimelineEvent` table. Wire a Prisma Client Extension on the four history tables' `create` method that closes the prior open row, calls C4 (`TimelineEventWriter.recordTimelineEvent`, the Wave-0 contract stub bound in Story 1.19's `ContractsModule`) automatically, and suppresses and flags a conflicting manual `TimelineEvent` via `markSystemWriteSkipped` instead of overwriting it.

## Boundaries & Constraints

**Always:**
- One history table per dimension, `(employeeId, value, effectiveFrom, effectiveTo)`; current = `effectiveTo IS NULL`; never a denormalized "current" column (AD-7 — the architecture decision requiring effective-dated history over a mutable current-state column).
- Each history table has a partial unique index enforcing at most one open row per `employeeId`: `UNIQUE (employeeId) WHERE effectiveTo IS NULL`. This is the DB-level backstop for the invariant above — application logic alone does not guarantee it.
- `Employee.userId` references `User` with `onDelete: Cascade` (an `Employee` has no lifecycle independent of its `User`). Each history table's `employeeId` FK references `Employee` with `onDelete: Restrict` (history rows must never be silently cascade-deleted; employees are not hard-deleted while history exists).
- The ONLY way a history row is created is through the Prisma Client Extension's intercepted `create` on the four models — no service method may write through an unextended client. The extension rejects (throws) every other operation on the four models except `create`: `update`, `updateMany`, `delete`, `deleteMany`, `upsert`, `createMany`, and Prisma's `createManyAndReturn`/`updateManyAndReturn` — history rows are append-only and closed-not-mutated; `create` is the only legal write shape. `findUnique`/`findUniqueOrThrow` are rejected too: the partial unique index only guarantees uniqueness among *open* rows, so Prisma's generated `findUnique` (which assumes true single-row uniqueness from the schema's `@@unique([employeeId])`) can silently return an arbitrary — possibly closed/stale — row once an employee has 2+ history rows; callers must use `findFirst`/`findMany` with an explicit `effectiveTo` filter instead.
- On create: extension closes any existing open row for the same `(employeeId, dimension)` by setting its `effectiveTo` to the new row's `effectiveFrom`, in the same transaction as the insert. Reject (throw) if the new row's `effectiveFrom` is less than or equal to the currently open row's `effectiveFrom` — out-of-order/backdated writes are not supported by this story.
- The close-prior-row read and the insert happen inside one serializable-or-stricter DB transaction (row lock via `SELECT ... FOR UPDATE` or equivalent), so two concurrent creates for the same `(employeeId, dimension)` cannot both observe "no open row" and both insert; the partial unique index above is the last-resort guard if the transaction-level protection is ever bypassed.
- Extension always calls C4 `recordTimelineEvent(employeeId, type, effectiveFrom, oldValue, newValue, 'system')` after a successful write, inside the same transaction as the history write and the prior-row close — if the C4 call throws, the whole transaction rolls back and no history row is ever left with a missing timeline event. `type` is one of `grade`, `position`, `department`, or `employmentType`, mapped one-to-one from the model name (`GradeHistory` → `grade`, `PositionHistory` → `position`, `DepartmentHistory` → `department`, `EmploymentTypeHistory` → `employmentType`). `oldValue` and `newValue` are the prior and new `value`; `oldValue` is `null` when there was no prior row.
- Before writing, extension checks `TimelineEvent` for an existing row with the same `(employeeId, type, effectiveDate)` and `source: 'manual'`, comparing `effectiveDate` and `effectiveFrom` as the same underlying type and precision (date, UTC, no time-of-day component); if found, suppress the history write, call C4 `markSystemWriteSkipped(manualEventId, now)`, and do not close the prior open row, create a new history row, or create a new timeline row. A `(employeeId, type, effectiveDate, source)` unique constraint on `TimelineEvent` keeps this lookup unambiguous — at most one manual entry can exist per key.
- A write whose `value` is unchanged from the current open row's `value` is not deduplicated — it still closes the prior row, creates a new row, and fires a timeline event. Wave-0 does not special-case no-op writes.
- `Employee` and `TimelineEvent` are minimal Wave-0 shapes (IDs, FKs, timestamps, and the fields C4's signature needs) — later stories (1.1, 1.6, Epic 7) extend them; do not add fields this story doesn't need.
- Follow `nest-prisma.md`: migration via `db:migrate`, commit generated SQL, never hand-edit an applied migration.

**Ask First:** none identified.

**Never:** no real `timeline` module or service (Epic 7 owns it — C4 stays the Wave-0 stub bound in Story 1.19's `ContractsModule`); no UI; no fields on `Employee` beyond the `User` link; no S4-section API endpoint (Story 1.6+, the future employee-profile "S4" UI section); no change to `PrismaService`'s connection lifecycle (`onModuleInit`/`onModuleDestroy`); no validation of `value` against a per-dimension domain or enum (callers are trusted for Wave-0; domain validation is a later story's concern); no interception of raw SQL (`$queryRaw`/`$executeRaw`) against the four history tables — the Prisma Client Extension is the only interception point this story guarantees, and raw SQL bypassing it is a known, accepted limitation, not this story's problem to solve.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| First-ever grade write | No prior `GradeHistory` row for employee | New row created, `effectiveTo` null; C4.recordTimelineEvent called with `oldValue: null` | N/A |
| Grade change | Existing open row (`effectiveTo` null) | Prior row's `effectiveTo` set to new `effectiveFrom`; new row created; C4 called with old+new values | N/A |
| Manual-entry conflict | A `TimelineEvent` with same `(employeeId, type, effectiveDate)` and `source: 'manual'` exists | History write suppressed; prior row left open; no new history or timeline row; C4.markSystemWriteSkipped called on that event's id | Extension throws `ManualConflictSuppressedError` (iteration 2 patch) — the manual entry stands, caller gets a typed error rather than a silent `null` |
| Same logic, other dimension | Write to `PositionHistory`/`DepartmentHistory`/`EmploymentTypeHistory` | Identical close-prior-row + C4-call behavior as `GradeHistory` | N/A |
| Out-of-order write | New `effectiveFrom` <= currently open row's `effectiveFrom` | Write rejected before any DB mutation | Extension throws a validation error; caller must supply a later `effectiveFrom` |
| Concurrent writes, same employee/dimension | Two `create` calls race for the same `(employeeId, dimension)` | Transaction/row-lock (and the partial unique index as backstop) admits exactly one; the loser observes the now-updated row or fails | Losing transaction throws `ConcurrentHistoryWriteError` (iteration 2 patch, wraps Postgres serialization failure `40001`/Prisma `P2034`) — a typed, retryable error rather than a raw driver error |
| No-op value write | New `value` equals the currently open row's `value` | Prior row still closed, new row still created, C4 still called — no deduplication | N/A |
| Unknown employee | `employeeId` has no corresponding `Employee` row | FK constraint violation | Extension does not catch this — it surfaces as a standard Prisma FK-violation error to the caller |
| C4 call fails | `recordTimelineEvent` throws after the history write is prepared | Whole transaction (close + insert + C4 call) rolls back | No history row or timeline event persists; caller receives the C4 error |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- only model so far is `User`; add the 6 new models + `User`'s back-relation.
- `services/backend/src/prisma/prisma.service.ts` -- current lifecycle-only `PrismaClient` subclass (`$connect`/`$disconnect`); keep intact, no business logic here.
- `services/backend/src/prisma/prisma.module.ts` -- `@Global()` provider; becomes the place the extended client is constructed and exported as `PrismaService`.
- `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- new file; the `Prisma.defineExtension` intercepting the four history models' writes, including the model→`type` string map (`GradeHistory` → `grade`, etc.).
- `services/backend/src/modules/contracts/timeline-event-writer.contract.ts` -- frozen C4 signature from Story 1.19; do not modify.
- `services/backend/src/modules/contracts/contracts.module.ts` -- existing binding of `TimelineEventWriter` to its stub; extension must inject the DI token, never the concrete class.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` -- add `Employee`, `TimelineEvent`, and the 4 history models, plus `User`'s back-relation -- FK targets the extension and future S4-section and timeline stories need. Prisma's schema DSL cannot express the per-table partial unique index (`effectiveTo IS NULL`) directly; the schema-level `@@unique` here is a placeholder full index that the migration step below narrows to a partial one.
- [x] `services/backend/prisma/migrations/**` -- **deviates from `nest-prisma.md`'s default one-shot `npm run db:migrate` for this migration only**: run `npx prisma migrate dev --create-only` to generate without applying, hand-edit the generated SQL to replace each `@@unique`-derived index with a partial one (`CREATE UNIQUE INDEX ... ON "..." ("employeeId") WHERE "effectiveTo" IS NULL`), then apply with `npm run db:migrate` (or `npx prisma migrate dev`) and commit the edited SQL. This is editing a migration *before* it's applied, not an applied one, so it does not violate the "never hand-edit an applied migration" rule -- only way to get a DB-level partial-unique backstop given Prisma has no native partial-index syntax and this repo's default `db:migrate` applies immediately
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- `Prisma.defineExtension` implementing close-prior-row + out-of-order rejection + manual-conflict-check + transactional C4-call + reject-on-`update`/`delete`/`upsert`/`createMany` for the 4 models, parameterized by an injected `TimelineEventWriter` -- the structural coupling AD-7 requires
- [x] `services/backend/src/prisma/prisma.module.ts` -- wire the extension so the exported `PrismaService` instance IS the extended client everywhere it's injected; verify with `tsc` that `OnModuleInit` and `OnModuleDestroy` still fire -- makes the extension the only path, not an opt-in
- [x] `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- covers every I/O matrix row (happy path, manual conflict, out-of-order rejection, concurrent-write race, no-op write, unknown employee, C4 failure/rollback) against a real Postgres test DB, parameterized across the 4 dimension models -- proves the extension for real, not just by inspection
- [x] add an architectural/DI-audit test asserting no unextended `PrismaClient` is reachable via Nest's DI container -- catches a future regression that silently reintroduces a bypass path (`services/backend/src/prisma/__tests__/prisma.module.spec.ts`)

**Patch (iteration 2, code-review renegotiation):**
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- expand `REJECTED_OPERATIONS` to `update`, `updateMany`, `delete`, `deleteMany`, `upsert`, `createMany`, `createManyAndReturn`, `updateManyAndReturn`, `findUnique`, `findUniqueOrThrow` -- closes the intent-gap bulk-mutation bypass and the `findUnique` stale-row landmine (renegotiated Boundaries above)
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- on manual-conflict suppression, throw a `ManualConflictSuppressedError(manualEventId)` instead of returning `null` -- a silent `null` from what looks like a non-nullable `.create()` is an NPE trap for the first real caller
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- catch Postgres serialization failure (SQLSTATE `40001`) around the transaction and rethrow as a typed `ConcurrentHistoryWriteError` -- gives a future caller something concrete to catch/retry on instead of a raw Prisma error
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- validate `value` is a non-empty string (same rigor already applied to `employeeId`/`effectiveFrom`) before use
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- after `toDateOnly`, reject an unparseable `effectiveFrom` (`Number.isNaN(effectiveFrom.getTime())`) instead of letting `NaN` silently pass the out-of-order comparison
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- reorder `handleHistoryCreate` so the out-of-order check runs before the manual-conflict check -- a backdated write that happens to match a manual entry must surface as `OutOfOrderEffectiveDateError`, not get silently swallowed as a conflict suppression
- [x] `services/backend/prisma/schema.prisma` + a follow-up migration -- add a plain `@@index([employeeId])` alongside the partial unique index on the 4 history tables -- an ordinary `findMany({ where: { employeeId } })` (used throughout the test suite) currently can't use the partial index and would sequential-scan as tables grow
- [x] `services/backend/src/prisma/__tests__/prisma.module.spec.ts` -- rewrite the `onModuleInit`/`onModuleDestroy` tests to spy on `raw.onModuleInit`/`raw.onModuleDestroy` (or `raw.$connect`/`raw.$disconnect`) and assert they were called, rather than inferring firing from query success -- Prisma's driver adapter connects lazily, so the current assertions pass even with the reattachment deleted entirely (confirmed by the verification-gap reviewer)
- [x] `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- assert the applied migration's partial-index `WHERE` clause is actually present in the live DB (query `pg_indexes`/`pg_get_indexdef`) -- guards against a future `prisma migrate dev` regeneration silently dropping the hand-edited partial clause
- [x] `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- add a case: a manual `TimelineEvent` exists before any history row for that employee (no prior open row) -- the existing manual-conflict test only covers the case where an open row already exists

**Acceptance Criteria:**
- Given an employee's grade changes, when the write lands in `GradeHistory` via the extension (the only write path exercised in this story — there is no service, endpoint, or UI caller yet), then the previous row's `effectiveTo` is set, a new row is created with `effectiveTo IS NULL`, and C4 is called automatically, all inside one transaction — no service method writes to a history table through any other path.
- Given a system-sourced write would overwrite a manual timeline correction in the same effective window, when the extension detects the conflict, then the history write is suppressed and C4's `markSystemWriteSkipped` attaches skip metadata to the existing manual entry — never a silent no-op, never a separate row. (See I/O & Edge-Case Matrix, "Manual-entry conflict" row, for the full behavior.)
- Given an out-of-order or concurrent write, when the extension's ordering check or transaction/unique-index guard triggers, then the write is rejected rather than silently corrupting the effective-date sequence. (See matrix rows "Out-of-order write" and "Concurrent writes, same employee/dimension".)
- Given the extension logic is dimension-agnostic, when `PositionHistory`, `DepartmentHistory`, or `EmploymentTypeHistory` are written, then the same behavior holds, verified by one parameterized test rather than four copies.

## Spec Change Log

- **2026-08-27 (iteration 1, code-review renegotiation):** Closed structural gaps found by adversarial/edge-case review: added a DB-level partial unique index enforcing one open row per employee/dimension; extension now rejects `update`/`delete`/`upsert`/`createMany` on the 4 history models; rejects out-of-order/backdated writes; wraps close-row + insert + C4 call in one transaction so a C4 failure rolls back the whole write; clarified the manual-conflict lookup's type/precision equivalence and added a uniqueness constraint on `TimelineEvent` for manual entries; explicitly scoped `value`-domain validation and raw-SQL bypass as out of this story's boundary; added FK `onDelete` behavior for `Employee`→`User` and the history tables→`Employee`. Editorial cleanup applied throughout (slash-conjunctions spelled out, undefined terms glossed at first use, symbols replaced with words). `baseline_commit` was checked against `services/backend` HEAD and confirmed correct (40-character SHA, no change needed). Verified against the installed Prisma 7 schema DSL and this repo's `db:migrate` script that a partial unique index cannot be expressed in `schema.prisma` and would otherwise be auto-applied unedited; added an explicit `--create-only` deviation to the schema/migration tasks and Design Notes so the partial index survives the migration step as intended.
- **2026-08-27 (iteration 2, post-implementation code review — Blind Hunter, Edge Case Hunter, Verification Gap):** **Intent gap (human-renegotiated):** the frozen rejected-operations enumeration (`update`/`delete`/`upsert`/`createMany`) was incomplete — `updateMany`/`deleteMany`/`createManyAndReturn`/`updateManyAndReturn` fell through to the raw query and could bulk-mutate "append-only" history rows, defeating the story's core guarantee; the human confirmed the intent was always "reject every non-`create` operation," so the Boundaries bullet above was expanded to the full enumeration plus `findUnique`/`findUniqueOrThrow` (a related landmine the edge-case reviewer found: the partial unique index only guarantees uniqueness among open rows, so Prisma's generated `findUnique` can silently return a stale/closed row). **Patch findings** (see new Patch task list above): manual-conflict suppression now throws instead of returning a silent `null`; Postgres serialization failures get a typed rethrow; `value` gets the same validation rigor as `employeeId`/`effectiveFrom`; an unparseable `effectiveFrom` is rejected instead of silently bypassing the out-of-order guard via `NaN`; the out-of-order check now runs before the manual-conflict check so a backdated write matching a manual entry is correctly flagged as out-of-order, not silently suppressed; added a plain index for ordinary `employeeId` lookups; the `onModuleInit`/`onModuleDestroy` DI-audit tests were rewritten to spy on invocation directly (the verification-gap reviewer proved empirically that the original assertions passed even with the reattachment deleted, because Prisma's driver adapter connects lazily); added a live-DB assertion that the partial index survives migration regeneration; added a first-write-vs-manual-conflict test case. **Rejected as noise:** the `Employee`(Cascade)/history-tables(Restrict) FK interaction — already the documented, intended behavior ("employees are not hard-deleted while history exists") with no current caller to be surprised by it. **KEEP:** the transaction structure (manual-conflict check → out-of-order validation and prior-row close → insert → C4 call, all inside one Serializable transaction via the pre-extension `internalClient`) is correct and worked well through review — only the check *order* within it changes, not the transaction shape itself; the DI-wiring approach (factory-constructed extended client as the sole `PrismaService` provider) survived review unchanged and should not be redesigned.

## Design Notes

`Employee` and `TimelineEvent` are added now, despite Epic 7 "owning" the real `timeline` module, because AD-7's conflict-detection AC requires querying an existing *manual* `TimelineEvent` row. C4 itself stays a no-op stub (per Story 1.19) — nothing in production populates `TimelineEvent` yet — but the table is Wave-0 substrate (this story's minimal schema, extended by later stories) so Epic 7 lands against a frozen shape, not a later migration.

All writes this extension intercepts are, by definition, `source: 'system'` — a "manual correction" is a human-entered `TimelineEvent` row (Epic 7's manual-edit flow, out of scope here), never a direct write to a history table. There is no `source` column on the history tables themselves; only `TimelineEvent` carries it.

Making the extended client the DI-injected `PrismaService` singleton (so `.gradeHistory.create()` is unreachable without the extension) must be verified against the installed Prisma 7 API — construct the extension inside `PrismaModule`'s provider and confirm `$extends()`'s return value still satisfies Nest's lifecycle-hook duck-typing. Treat this as unverified until `tsc` and the extension tests pass; if the duck-typing does not hold, fall back to manually re-attaching `onModuleInit`/`onModuleDestroy` on the extended client inside the provider factory rather than relying on `$extends()` to preserve them.

Prisma's schema DSL has no syntax for a partial/filtered unique index, and this repo's `db:migrate` script (`prisma migrate dev`) generates and applies a migration in one step by default — there's no built-in checkpoint to hand-edit the SQL before it lands. The Tasks step above resolves this by using `prisma migrate dev --create-only` once, for this migration only, to get that checkpoint; this is scoped to the one migration that needs a partial index, not a general departure from `nest-prisma.md`'s workflow.

## Verification

**Commands:**
- `cd services/backend && npx tsc --noEmit` -- expected: PASS
- `cd services/backend && npm run lint` -- expected: PASS, 0 errors
- `cd services/backend && npm run db:migrate` -- expected: migration applies cleanly against a fresh local DB
- `cd services/backend && npm test` -- expected: PASS, including the new extension tests (all I/O matrix rows) and the DI-audit test confirming no second/unextended Prisma client is reachable in DI

## Suggested Review Order

**Write path & structural coupling (the core of this story)**

- Entry point: the only legal write to the 4 history models — closes the prior row, checks for a manual conflict, calls C4, all in one transaction.
  [`temporal-history.extension.ts:227`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L227)

- Every other operation on the 4 models is rejected outright — the append-only guarantee's actual enforcement point (expanded in the iteration-2 patch).
  [`temporal-history.extension.ts:47`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L47)

- Out-of-order validation now runs before the manual-conflict check (iteration-2 reorder) — a backdated write matching a manual entry surfaces as out-of-order, not a silent suppression.
  [`temporal-history.extension.ts:271`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L271)

- Manual-conflict suppression now throws instead of returning `null` (iteration-2 patch) — no NPE trap for the first real caller.
  [`temporal-history.extension.ts:291`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L291)

- Typed errors for the 4 failure modes a caller needs to distinguish (out-of-order, rejected op, manual conflict, concurrent-write race).
  [`temporal-history.extension.ts:61`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L61)

**DI wiring — making the extension structurally unavoidable**

- The exported `PrismaService` token resolves to the extended client, never the raw one; lifecycle hooks re-attached explicitly since `$extends()` doesn't guarantee preserving them.
  [`prisma.module.ts:32`](../../services/backend/src/prisma/prisma.module.ts#L32)

**Schema — the temporal-history substrate**

- `Employee`: minimal Wave-0 shape, 1:1 with `User`, the FK target every history/timeline table needs.
  [`schema.prisma:27`](../../services/backend/prisma/schema.prisma#L27)

- `GradeHistory`: the per-dimension shape (identical across all 4) — partial unique index for "one open row" plus the iteration-2 plain index for ordinary lookups.
  [`schema.prisma:67`](../../services/backend/prisma/schema.prisma#L67)

- `TimelineEvent`: minimal substrate for AD-7's manual-conflict detection, ahead of Epic 7's real `timeline` module.
  [`schema.prisma:133`](../../services/backend/prisma/schema.prisma#L133)

**Tests — proving the guarantees for real**

- DI-audit: lifecycle-hook spies (iteration-2 rewrite) actually prove `onModuleInit`/`onModuleDestroy` fire on the resolved instance, not just that a query happens to succeed.
  [`prisma.module.spec.ts:72`](../../services/backend/src/prisma/__tests__/prisma.module.spec.ts#L72)

- Full I/O & Edge-Case Matrix, parameterized across all 4 dimensions, against a real Postgres DB.
  [`temporal-history.extension.spec.ts:1`](../../services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts#L1)

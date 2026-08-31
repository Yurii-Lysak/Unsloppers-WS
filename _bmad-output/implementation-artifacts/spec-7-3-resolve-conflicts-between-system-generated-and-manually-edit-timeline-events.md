---
title: 'Resolve Conflicts Between System-Generated and Manually-Edited Timeline Events'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'fbbb4a6c086e0c1aef23189ba0b6b7f72493fedb' # services/backend HEAD — implementation happens in that submodule
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-7-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-7-1-auto-generate-timeline-events.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-7-2-manual-timeline-edits.md'
  - '{project-root}/services/backend/.claude/rules/nest-e2e.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 7.1 and 1.20 wired D2 conflict suppression for an exact `(employeeId, type, effectiveDate)` manual key, but Epic 7 AC1 requires a PP date correction to survive a later re-sync that re-derives the same transition at the original inferred date — after PATCH moves the manual row off that date, the current lookup misses and the sync would write duplicate history. FR-30 / D2 also lack end-to-end proof through the Story 7.2 manual API and observable `systemWriteSkippedAt` in list responses.

**Approach:** Extend the temporal-history extension’s manual-conflict lookup with a guarded transition match when a system anchor event exists at the incoming `effectiveFrom`, then add integration and e2e tests that exercise the full correction → re-sync → skip-metadata path and the unrelated-manual non-suppression path. No new API surface or schema changes.

## Boundaries & Constraints

**Always:**
- D2 behavior stays in the Prisma extension + C4 `markSystemWriteSkipped` — no per-caller bespoke suppression.
- Exact-date manual conflict (Story 1.20 / 7.1) remains the first lookup; transition match is a fallback only when an active `source: 'system'` timeline row exists at the incoming `effectiveFrom` for the same `(employeeId, type)`.
- Transition fallback matches active manual rows with the same `(employeeId, type, oldValue, newValue)` as the history write would produce (`oldValue` from the open history row or `null`; `newValue` from the incoming `value`). JSON equality for `oldValue`/`newValue`.
- On conflict: suppress history write, call `markSystemWriteSkipped` on the matched manual row, throw `ManualConflictSuppressedError` — never create a skip row, never overwrite the manual row’s fields.
- Out-of-order check still runs before any conflict lookup (Story 1.20 ordering frozen).
- `systemWriteSkippedAt` continues to surface on REST GET / SectionProvider payloads (already mapped in `TimelineEventEntity`).

**Ask First:** none identified.

**Never:** no frontend S9 UI or skip affordance copy (EXPERIENCE.md render-time concern); no change to manual CRUD rules (Story 7.2 frozen); no semantic matching without the system-anchor guard (prevents unrelated historical backfills from suppressing future real changes); no hard delete; no changes to C4 contract signatures.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Exact-date conflict | Active manual at `(employeeId, type, effectiveFrom)` | History suppressed; `systemWriteSkippedAt` set on manual row | `ManualConflictSuppressedError` |
| Date-corrected manual | System timeline at 2026-01-10; manual PATCHed to 2026-01-15 with same transition; re-sync writes 2026-01-10 | Transition fallback matches manual at 2026-01-15; skip metadata on that row; no new history/timeline rows | `ManualConflictSuppressedError` |
| Unrelated manual backfill | Manual department change 2018-06-01; new grade history write 2026-09-01 | Normal system write + timeline row | N/A |
| Unrelated transition backfill | Manual grade Middle→Senior at 2015; no system anchor at incoming date; new Middle→Senior write 2026 | Normal write (anchor guard blocks false suppression) | N/A |
| Soft-deleted manual | Manual row soft-deleted at conflict date | No suppression; write proceeds | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/prisma/extensions/temporal-history.extension.ts:291-311` -- extend manual-conflict lookup with transition + system-anchor fallback after computing `oldValue`/`newValue`.
- `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts:289+` -- add cases for date-corrected manual + unrelated-type coexistence.
- `services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts:77+` -- add date-corrected manual conflict via real Postgres.
- `services/backend/test/timeline.e2e-spec.ts` -- add e2e: PP PATCH effectiveDate → re-attempt history write → GET shows one manual row with `systemWriteSkippedAt`.
- `services/backend/test/timeline.e2e-spec.ts` -- add e2e: unrelated manual department row does not block later grade history write.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/prisma/extensions/temporal-history.extension.ts` -- add transition + system-anchor fallback to manual-conflict lookup -- closes date-correction gap per Epic 7 AC1.
- [x] `services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts` -- unit/integration cases for transition fallback and unrelated-manual non-suppression -- proves extension I/O matrix.
- [x] `services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts` -- integration case for date-corrected manual + skip metadata -- real C4 path.
- [x] `services/backend/test/timeline.e2e-spec.ts` -- e2e AC1 + AC2 scenarios through REST + extended client -- FR-30 observable end-to-end.
- [x] `services/backend/test/support/app-harness.ts` -- wire temporal-history extension into e2e Prisma client -- history writes in e2e must use AD-7 path.

**Acceptance Criteria:**
- Given a system-generated grade-change event exists and a PP PATCHes a manual correction to a different effective date for the same transition, when a later history write re-derives that change at the original inferred date, then the write is suppressed, `systemWriteSkippedAt` is set on the manual row, and GET returns exactly one active grade event at the corrected date with skip metadata populated.
- Given a PP manually backfilled an unrelated department change and the employee later has a genuinely new grade change, when the extension processes the new grade write, then a new system-generated event is written normally and the earlier manual department entry is unchanged.

## Design Notes

Correction workflow with Story 7.2 API: PP POSTs a manual row (or PATCHes an existing manual row’s `effectiveDate`) to the corrected date while keeping the transition values; the original system timeline row at the inferred date is the anchor that enables transition fallback when sync retries that date.

Transition lookup uses Prisma JSON filters on `oldValue`/`newValue`; `null` oldValue uses `equals: Prisma.DbNull` or equivalent JSON-null filter consistent with existing seed/test patterns.

## Verification

**Commands:**
- `cd services/backend && nvm use && npx tsc --noEmit` -- expected: PASS
- `cd services/backend && npm run lint` -- expected: PASS, 0 errors
- `cd services/backend && npm test` -- expected: PASS, including new extension cases
- `cd services/backend && npm run test:e2e -- timeline.e2e-spec.ts` -- expected: PASS

## Spec Change Log

- **2026-08-31 (implementation):** Added e2e harness temporal-history extension wiring — e2e `PrismaService` override previously bypassed AD-7, so history writes in timeline e2e never exercised conflict suppression.

## Suggested Review Order

**Conflict resolution (D2 / FR-30)**

- Two-step lookup: exact date first, then transition fallback with system anchor
  [`temporal-history.extension.ts:291`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L291)

- Transition fallback matches manual rows by JSON old/new values at corrected dates
  [`temporal-history.extension.ts:387`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L387)

**E2e harness parity**

- E2e Prisma client now carries the same extension as production DI wiring
  [`app-harness.ts:81`](../../services/backend/test/support/app-harness.ts#L81)

**Tests**

- Extension matrix: date-corrected, unrelated manual, anchor guard, soft-delete
  [`temporal-history.extension.spec.ts:405`](../../services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts#L405)

- Real C4 integration for date-corrected skip metadata
  [`timeline-event-writer.integration.spec.ts:110`](../../services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts#L110)

- REST + history re-sync e2e for both epic ACs
  [`timeline.e2e-spec.ts:256`](../../services/backend/test/timeline.e2e-spec.ts#L256)
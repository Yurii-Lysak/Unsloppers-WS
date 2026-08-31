---
title: 'Manual Timeline Edits'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '1ab3a66f08365c85c763685675bcd76f00f49ccd' # services/backend HEAD — implementation happens in that submodule
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-7-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-7-1-auto-generate-timeline-events.md'
  - '{project-root}/services/backend/.claude/rules/nest-e2e.md'
  - '{project-root}/services/backend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 7.1 made C4 persist system-generated timeline rows, but there is still no way for PP or UM to backfill historical events or correct wrongly inferred ones — no manual write API, no S9 section provider, no soft-delete schema, and no write gate that narrows S9 beyond C1's coarse ProjectLine `RW`.

**Approach:** Extend the `timeline` module with REST endpoints and a registered `SectionProvider(S9)` so PP/UM can create, update, and soft-delete manual events (`source: 'manual'`), returned in chronological order by `effectiveDate`. Enforce the documented S9 write narrowing server-side: only `ReportingLine` (UM) and `PP` roles may write; ProjectLine-only viewers (e.g. a DM via project assignment) are rejected even when C1 grants S9 `RW`.

## Boundaries & Constraints

**Always:**
- Manual creates go through `TimelineEventWriter.recordTimelineEvent(..., source: 'manual', authorId: viewerId)` — same single write path as system events (C4); the timeline service never bypasses C4 for inserts.
- Reads (REST list + SectionProvider) filter `deletedAt IS NULL`, order by `effectiveDate ASC`, then `createdAt ASC` for same-day ties.
- Read access: C1 `resolveAudience` → `sections.S9 !== 'none'`.
- Write access: C1 role must be `ReportingLine` or `PP` (reject `ProjectLine`, `Self`, `Colleague`, etc.) — this is the one documented S9 narrowing; do not use C8 `PermissionChecker` as the write gate while it remains deny-all stubbed (Stories 1-4/1-5 backlog).
- Soft-delete sets `deletedAt` + `deletedById`; row stays in DB for audit. Updates set `updatedAt` + `updatedById`.
- Edit and soft-delete apply only to rows with `source: 'manual'`; system rows are immutable via this API (corrections use manual add + Story 7.3 conflict path).
- Valid manual `type` values: `grade`, `position`, `department`, `employmentType`, `joining`, `extendedLeave`, `mentorshipStart`, `mentorshipEnd` (same strings C4 uses; UI labels are render-time).
- `@RegisterProvider('section', 'S9')` on `TimelineSectionProvider`; register in `TimelineModule.providers`.
- Follow `nest-prisma.md`: schema change via new migration; never edit applied migration SQL.

**Ask First:** none identified.

**Never:** no frontend S9 UI (Story 1.6 profile assembly / `career-timeline` component not built yet); no change to temporal-history extension or C4 system-write path (Story 7.1 frozen); no Story 7.3 conflict-resolution changes beyond what existing extension already does; no `PermissionChecker` real implementation (Story 1-4 scope); no hard delete; no departure/joining auto-write callers (other epics); no `ProfileAssemblerService` (Story 1-6).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| PP manual backfill | POST with type, effectiveDate, old/new values | Row with `source: 'manual'`, `authorId`, appears in list at chronological position | N/A |
| DM ProjectLine write | Viewer role `ProjectLine`, S9 `RW` in C1 | 403 Forbidden on POST/PATCH/DELETE | `ForbiddenException` |
| UM ReportingLine write | Viewer role `ReportingLine` | Create/update/delete succeeds | N/A |
| Edit system event | PATCH on `source: 'system'` row | 403 or 404 | `ForbiddenException` |
| Soft delete manual | DELETE on manual row | `deletedAt`/`deletedById` set; row absent from GET | N/A |
| Duplicate manual key | Second manual row same `(employeeId, type, effectiveDate, source: 'manual')` | 409 or 400 | Prisma unique violation → mapped error |
| Re-create after soft delete | DELETE then POST same key | Succeeds (partial unique index excludes deleted rows) | N/A |
| Read without access | Colleague viewer | 403 on GET | `ForbiddenException` |
| SectionProvider | `getSection(viewerId, employeeId)` with S9 granted | `{ events: [...] }` sorted, excludes deleted | Throws if S9 `none` |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma:248-264` -- `TimelineEvent`; add `deletedAt`, `deletedById`, `updatedAt`, `updatedById`; replace `@@unique` with partial unique on active rows.
- `services/backend/src/modules/timeline/timeline.module.ts:14-21` -- reshape: add controller, service, section provider alongside existing C4 binding.
- `services/backend/src/modules/timeline/timeline-event-writer.service.ts:21-43` -- C4 insert path manual creates reuse.
- `services/backend/src/modules/contracts/timeline-event-writer.contract.ts:11` -- `TimelineEventSource = 'system' | 'manual'`.
- `services/backend/src/modules/access/access-resolver.service.ts:45-115` -- C1 S9 grants; write gate must override ProjectLine `RW`.
- `services/backend/src/modules/directory/custom-fields.controller.ts:33-37` -- `CurrentUserProvider` + `@Req()` pattern for controllers.
- `services/backend/src/modules/directory/custom-fields.service.ts:101-116` -- layered access assertion pattern (adapt for role-based write gate).
- `services/backend/src/modules/registry/register-provider.decorator.ts:20-27` -- `@RegisterProvider('section', 'S9')`.
- `services/backend/test/support/access-matrix.ts:148-154` -- S9 matrix; add write-narrowing exception test like S7.
- `services/backend/test/support/graph-factory.ts` -- e2e graph for PP/UM/DM scenarios.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` -- add soft-delete/audit columns + partial unique index -- supports FR-29 audit trail and re-create after delete.
- [x] `services/backend/prisma/migrations/**` -- generate migration -- per nest-prisma.md.
- [x] `services/backend/src/modules/timeline/timeline.constants.ts` -- `EDIT_CAREER_TIMELINE` permission key constant (for future C8) + allowed event types -- single source of truth.
- [x] `services/backend/src/modules/timeline/timeline.service.ts` -- CRUD + access gates (C1 read, role write narrowing) + chronological queries -- core business logic.
- [x] `services/backend/src/modules/timeline/timeline.controller.ts` -- REST under `employees/:employeeId/timeline` -- HTTP surface.
- [x] `services/backend/src/modules/timeline/dto/` + `entities/` + `timeline.swagger.ts` -- request/response shapes -- match users/directory conventions.
- [x] `services/backend/src/modules/timeline/timeline-section.provider.ts` -- `@RegisterProvider('section', 'S9')` with `getSection(viewerId, employeeId)` -- AD-3 registration.
- [x] `services/backend/src/modules/timeline/timeline.module.ts` -- wire controller, service, section provider -- module reshape.
- [x] `services/backend/src/modules/timeline/__tests__/timeline.service.spec.ts` -- unit tests for I/O matrix rows -- proves access gates and CRUD rules.
- [x] `services/backend/test/timeline.e2e-spec.ts` -- e2e: PP create, DM 403, soft-delete hidden, chronological order -- proves ACs end-to-end.

**Acceptance Criteria:**
- Given a PP viewing an employee with S9 access, when they POST a manual grade-change event dated 2019-03-15 with old/new values, then the event persists with `source: 'manual'`, and GET returns it sorted by `effectiveDate` (not append-only by `createdAt`).
- Given a DM with ProjectLine Manager access who is not the assigned UM or PP, when they attempt POST/PATCH/DELETE on any timeline event, then the server responds 403.

## Spec Change Log

## Design Notes

Partial unique index pattern for Postgres: `@@unique([employeeId, type, effectiveDate, source], map: "timeline_events_active_key")` with `WHERE deleted_at IS NULL` in migration SQL (Prisma may need raw SQL in migration for partial index — document in migration comment).

`TimelineSectionProvider.getSection` returns `{ events: TimelineEventEntity[] }` (never `null`) so consumers distinguish "empty timeline" from "unavailable section" per AD-3.

Write gate helper: `assertCanWriteTimeline(viewerId, employeeId)` → resolve audience → if `role` not in `['ReportingLine', 'PP']` throw 403; if `sections.S9 === 'none'` throw 403.

## Verification

**Commands:**
- `cd services/backend && nvm use && npx tsc --noEmit` -- expected: PASS
- `cd services/backend && npm run lint` -- expected: PASS, 0 errors
- `cd services/backend && npm run db:migrate` -- expected: new migration applies cleanly
- `cd services/backend && npm test` -- expected: PASS, including new timeline service tests
- `cd services/backend && npm run test:e2e -- timeline.e2e-spec.ts` -- expected: PASS

## Suggested Review Order

**Write gate and service layer**

- Role-narrowed S9 writes: ReportingLine/PP only, rejects ProjectLine
  [`timeline.service.ts:154`](../../services/backend/src/modules/timeline/timeline.service.ts#L154)

- Manual creates routed through C4, not direct Prisma inserts
  [`timeline.service.ts:52`](../../services/backend/src/modules/timeline/timeline.service.ts#L52)

**HTTP surface**

- REST CRUD under employees/:employeeId/timeline
  [`timeline.controller.ts:28`](../../services/backend/src/modules/timeline/timeline.controller.ts#L28)

**Section provider registration**

- S9 payload for future ProfileAssembler via registry
  [`timeline-section.provider.ts:7`](../../services/backend/src/modules/timeline/timeline-section.provider.ts#L7)

**Schema and extension coupling**

- Soft-delete columns; partial unique index in migration SQL
  [`migration.sql:1`](../../services/backend/prisma/migrations/20260831160000_add_timeline_soft_delete/migration.sql#L1)

- Manual-conflict query excludes soft-deleted rows
  [`temporal-history.extension.ts:293`](../../services/backend/src/prisma/extensions/temporal-history.extension.ts#L293)

**Tests**

- Unit coverage for access gates and CRUD rules
  [`timeline.service.spec.ts:1`](../../services/backend/src/modules/timeline/__tests__/timeline.service.spec.ts#L1)

- E2e: PP create, DM 403, soft-delete, re-create same key
  [`timeline.e2e-spec.ts:1`](../../services/backend/test/timeline.e2e-spec.ts#L1)


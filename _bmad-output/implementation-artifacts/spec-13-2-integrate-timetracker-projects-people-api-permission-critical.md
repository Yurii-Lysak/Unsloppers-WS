---
title: 'Integrate Timetracker Projects & People API (Permission-Critical)'
type: 'feature'
created: '2026-09-01'
status: 'done'
review_loop_iteration: 2
baseline_commit: '76ff65b4f6f2b306dc501cb2ac6830a1ffac42a0'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-13-context.md'
  - '{project-root}/docs/api-external-openapi.json'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Project-derived Manager access depends on C3 `ProjectAssignment` rows, but TimeTracker has no runtime writer. Assignments cannot be source-confirmed or safely expire when synchronization stops.

**Approach:** Sync at startup and every 15 minutes. Join feed emails to TimeTracker numeric IDs, resolve them through C5, then atomically confirm observed assignments and deconfirm missing TimeTracker-owned rows. Failures leave timestamps unchanged so the existing four-hour gate fails safe.

## Boundaries & Constraints

**Always:**
- Fetch Active/Support projects and use the current-month accounting report as TimeTracker's people directory.
- Email may join TimeTracker payloads; platform identity resolves only through C5 external IDs. Never query platform users by email.
- Successful passes are atomic and use one injected-clock timestamp. Failures change nothing.
- Track source ownership and use an idempotent key; never deconfirm manual rows.
- Run after bootstrap and every 15 minutes with the Nest scheduler; prevent overlap.
- Preserve the four-hour `confirmedAt` gate; successful observation resets it.
- Log sanitized aggregate/omission counts without PII.

**Ask First:**
- If manager strings are not emails in the TimeTracker directory, request a stable mapping source.
- If accounting cannot supply the people directory, stop; never fall back to platform email.

**Never:**
- Grant access from invalid/stale assignments or refresh timestamps after partial/failed passes.
- Delete missing assignments or modify non-TimeTracker rows.
- Add cumulative-outage tracking, frontend UI, PeopleForce, or cross-system email inference.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|---------------|---------------------------|----------------|
| Successful sync | Mapped active assignment | Source-owned row is upserted and confirmed | Atomic transaction |
| Removed member | Source-owned row absent from a complete feed | Row becomes unconfirmed | Manual rows unchanged |
| Unmapped identity | Missing directory ID or C5 mapping | Omit that assignment; commit valid rows | Log sanitized count |
| Feed failure | Endpoint, payload, or DB failure | No assignments change; scheduler survives | Log sanitized failure |
| Overlap | Sync already running | Skip new tick | Debug log |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/timetracker/timetracker.service.ts` — accounting/projects client, timeout, typed errors.
- `services/backend/src/modules/contracts/timetracker.types.ts` — employee, project, member, and status payloads.
- `services/backend/src/modules/integrations/external-identity-mapping.service.ts` — C5 resolver with supersession.
- `services/backend/src/modules/integrations/leaves-sync.service.ts` — `Clock`, error, and logging pattern.
- `services/backend/src/modules/integrations/integrations.module.ts` — new writer/scheduler owner.
- `services/backend/prisma/schema.prisma:88` — C3 lacks source ownership and a TimeTracker-only idempotent key; manual history may contain repeated project/employee periods.
- `services/backend/src/modules/access/access-resolver.service.ts:363` — preserve the existing access gate.
- `services/backend/src/modules/access/project-assignment.service.ts` — existing C3 reader/manual writer.
- `services/backend/test/support/external-boundary.ts` — boundary fault injection.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/package.json`, lockfile, `src/app.module.ts` — add/configure `@nestjs/schedule`.
- [x] `services/backend/prisma/schema.prisma` + migration — add source ownership and a nullable unique `sourceKey` populated only by TimeTracker; preserve unrestricted legacy/manual rows.
- [x] `services/backend/src/modules/integrations/project-assignment.mapper.ts` — perform TimeTracker-directory/C5 resolution and normalize rows.
- [x] `services/backend/src/modules/integrations/projects-sync.service.ts` — fetch, prevent overlap, and transactionally upsert/deconfirm.
- [x] `services/backend/src/modules/integrations/projects-sync.scheduler.ts`, `integrations.module.ts` — wire bootstrap and 15-minute runs.
- [x] `services/backend/src/modules/integrations/__tests__/` — cover the matrix, rollback, source-key idempotency, manual duplicates/coexistence, production module wiring, scoping, and fixed time.
- [x] `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` — verify fresh and over-four-hour ProjectLine behavior.

**Acceptance Criteria:**
- Given a mapped active assignment, when sync succeeds, then C3 confirms one source-owned row and mapped management receives ProjectLine access.
- Given an assignment is unobserved for over four hours, when access resolves, then project-derived Manager access is denied.
- Given a complete feed omits an assignment, when sync succeeds, then only that TimeTracker row is unconfirmed.
- Given either request or transaction fails, when sync ends, then assignments and confirmation timestamps are unchanged.

## Spec Change Log

- Review loop 1 — Migration review found that `@@unique([source, projectId, employeeId])` constrains all legacy/manual rows and can fail deployment on previously valid duplicates. The persistence task now requires a nullable unique TimeTracker-only `sourceKey`, avoiding destructive reconciliation and leaving manual history unrestricted. KEEP: C5-only directory joins, atomic source-scoped writes, scheduler cadence, overlap guard, sanitized logs, fixed-clock confirmation, and matrix coverage.
- Review loop 2 — Hardened malformed-payload rejection, deterministic duplicate handling, sanitized failure logs, enum ownership, initialized scheduler wiring, and database-backed sync/access and migration assertions. Frozen intent remains unchanged; genuine directory/C5 misses still omit invalid grants while valid rows commit atomically.

## Verification

**Commands:**
- `cd services/backend && npm test` — all tests pass.
- `cd services/backend && npm run lint` — zero errors.
- `cd services/backend && npm run build` — Nest/Prisma compile.
- `cd services/backend && npm run depcruise` — boundaries pass.

## Suggested Review Order

**Synchronization and fail-safe writes**

- Start with the complete-fetch, atomic-confirm, and source-scoped deconfirmation boundary.
  [`projects-sync.service.ts:32`](../../services/backend/src/modules/integrations/projects-sync.service.ts#L32)

- Verify bootstrap plus fixed 15-minute scheduling delegates to the overlap-safe service.
  [`projects-sync.scheduler.ts:8`](../../services/backend/src/modules/integrations/projects-sync.scheduler.ts#L8)

**Identity and payload safety**

- Review TimeTracker-only email joins, C5 resolution, and fail-closed payload validation.
  [`project-assignment.mapper.ts:46`](../../services/backend/src/modules/integrations/project-assignment.mapper.ts#L46)

- Check deterministic directory deduplication and ambiguous-identity handling.
  [`project-assignment.mapper.ts:143`](../../services/backend/src/modules/integrations/project-assignment.mapper.ts#L143)

**Persistence ownership**

- Confirm nullable source keys preserve unrestricted manual assignment history.
  [`schema.prisma:88`](../../services/backend/prisma/schema.prisma#L88)

- Inspect the legacy-safe ownership migration and database constraints.
  [`migration.sql:1`](../../services/backend/prisma/migrations/20260901103000_add_project_assignment_source_ownership/migration.sql#L1)

**Wiring and acceptance evidence**

- Verify scheduling is installed in the production application graph.
  [`app.module.ts:24`](../../services/backend/src/app.module.ts#L24)

- Follow synchronized persistence through fresh, stale, and unconfirmed access outcomes.
  [`project-assignment-sync.e2e-spec.ts:41`](../../services/backend/test/project-assignment-sync.e2e-spec.ts#L41)

- Confirm database ownership, uniqueness, and duplicate-manual-period behavior.
  [`project-assignment-sync.e2e-spec.ts:139`](../../services/backend/test/project-assignment-sync.e2e-spec.ts#L139)

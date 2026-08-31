---
title: 'Integrate Timetracker Leaves API'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '900a0d7b696e8b5beb0242a204042332635e21b6'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-13-context.md'
  - '{project-root}/docs/api-external-openapi.json'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Leave status (S10) is not sourced from TimeTracker at runtime — the existing `TimetrackerService` is seed-only, there is no `integrations` module, and profile/list surfaces cannot show accurate vacation/sick/parental leave data for entitled viewers.

**Approach:** Add an `integrations` module that (1) resolves platform employees to TimeTracker IDs via C5, (2) fetches and normalizes leave periods from the accounting report endpoint, (3) registers an S10 `@RegisterProvider` that returns leave data with a separate fail-soft `availability` flag, applying Colleague type-masking server-side. Wire a thin authenticated read endpoint so the provider is testable before Story 1.6's `ProfileAssemblerService` exists.

## Boundaries & Constraints

**Always:**
- Leave feed source is `POST /api/accounting/report` (`DayStatus` on `WorkingDay` rows) unless a newer TimeTracker endpoint is confirmed — group consecutive days with the same leave `DayStatus` into `{ type, startDate, endDate, approvalState }` periods.
- Map only leave-relevant `DayStatus` values (`Vacation`, `UnpaidLeave`, `Sick`, `OneDaySick`, etc.); exclude `WorkingDay`, `Weekend`, `Holiday`, remote-working statuses.
- S10 provider returns `{ availability: 'ok' | 'unavailable', leaves: LeavePeriodDto[], manageLeaveUrl?: string }` — ISO date strings, no `Date` objects in JSON.
- On `TimetrackerApiError`/timeout: set `availability: 'unavailable'`, return empty `leaves` (or last-known cache if implemented), **never throw** from `getSection` — profile assembly must not 500.
- Colleague audience: strip `type` from each leave period inside the provider (dates remain); use `AccessResolver.resolveAudience(viewerId, employeeId)` — this story supplies data, never changes who can see S10.
- C5 lookup uses `(system: 'timetracker', externalId)` → `employeeId`; seed must populate mappings from bootcamp manifest `BootcampIdentity.id` during seed run — **never email-only inference**.
- Reuse existing `TimetrackerService` HTTP/error handling; update its header comment to allow runtime use from `integrations`.
- Register `IntegrationsModule` in `app.module.ts`; implement real `ExternalIdentityMapping` here and remove C5 stub binding from `ContractsModule` (mirror C1/C4 pattern).
- Add `external_identities` Prisma model: `(system, externalId)` unique, `employeeId`, optional `supersededBy` — minimal Wave-1 shape per C5; Story 13.4 may refine.
- Add env key `TIMETRACKER_MANAGE_LEAVE_URL` (URI, optional) for self-service outbound link; validate in `env.validation.ts` + `.env.example`.
- Unit tests for leave-period grouping; integration tests using `test/support/external-boundary.ts` against accounting-report paths.

**Ask First:**
- If TimeTracker exposes a dedicated leaves endpoint not in `docs/api-external-openapi.json`, HALT and confirm whether to switch feed source before implementing.

**Never:**
- Touch `ProjectAssignment.confirmed` / access-resolution logic (Story 13.2).
- Implement All Employees list columns or `FieldProvider` leave filters (Stories 3.1 / 12.x).
- Build `ProfileAssemblerService` or frontend S10 UI (Stories 1.6 / frontend profile page).
- Show leave balances in-platform.
- Call TimeTracker synchronously on every HTTP request without bounded caching — at minimum cache per-process with TTL (e.g. 5 min) or batch fetch for requested employee set.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Approved vacation range | Accounting report returns `DayStatus.Vacation` for 2026-08-25..29 for employee B | S10 returns one period `{ type: 'vacation', startDate: '2026-08-25', endDate: '2026-08-29' }`, `availability: 'ok'` | N/A |
| Colleague viewer | Same data, viewer resolved as `Colleague` | Periods include dates, `type` field omitted/null | N/A |
| API unreachable | TimeTracker host down or 5xx | `availability: 'unavailable'`, `leaves: []`, HTTP 200 from S10 endpoint | No stack trace to client |
| Missing C5 mapping | Employee has no timetracker external id | `availability: 'ok'`, `leaves: []` (empty, not error) | Log at debug/warn |
| Timeout | Boundary `hang` behaviour | Same as unreachable after `REQUEST_TIMEOUT_MS` | `TimetrackerApiError` caught upstream |

</frozen-after-approval>

## Code Map

- `docs/api-external-openapi.json` — `DayStatus`, `WorkingDay`, `AccountingReportRequest/Response`; no separate leaves path
- `services/backend/src/modules/timetracker/timetracker.service.ts` — `fetchAccountingReport()`, `TimetrackerApiError`, 15s timeout; broaden from seed-only
- `services/backend/src/modules/timetracker/timetracker.types.ts` — `DayStatus`, `TimetrackerEmployee`, `WorkingDay`
- `services/backend/src/modules/contracts/external-identity-mapping.contract.ts` — C5 token; real impl moves to integrations
- `services/backend/src/modules/contracts/contracts.module.ts` — remove C5 stub binding when integrations provides real service
- `services/backend/src/modules/access/access-resolver.service.ts` — S10 grant levels; Colleague gets S10 `R` (type masking in provider)
- `services/backend/src/modules/registry/register-provider.decorator.ts` — `@RegisterProvider('section', 'S10')`
- `services/backend/src/prisma/seed/seed.manifest.ts` — `BootcampIdentity.id` is TimeTracker employee id for C5 seed rows
- `services/backend/test/support/external-boundary.ts` — fault-injection harness for NFR-4 tests
- `services/backend/src/modules/timeline/timeline-event-writer.service.ts` — C4; optional hook for extended-leave detection (log-and-continue on failure)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — add `ExternalIdentity` model (`system`, `externalId`, `employeeId`, `supersededBy?`) with `@@unique([system, externalId])`
- [x] `services/backend/src/prisma/seed/seed.service.ts` — upsert C5 rows from manifest `identity.id` → employee on each seed run
- [x] `services/backend/src/modules/integrations/` — new module: `integrations.module.ts`, `external-identity-mapping.service.ts` (C5), `leaves-sync.service.ts` (fetch + normalize + cache), `leaves-section.provider.ts` (`@RegisterProvider('section','S10')`), DTOs/entities/swagger
- [x] `services/backend/src/modules/integrations/leaves.controller.ts` — authenticated `GET /employees/:employeeId/leaves` delegating to S10 provider (dev/test surface until 1.6)
- [x] `services/backend/src/modules/contracts/contracts.module.ts` — drop C5 stub provider binding
- [x] `services/backend/src/config/env.validation.ts`, `.env.example` — `TIMETRACKER_MANAGE_LEAVE_URL`
- [x] `services/backend/src/app.module.ts` — import `IntegrationsModule`
- [x] `services/backend/src/modules/integrations/__tests__/` — grouping logic, Colleague masking, fail-soft on boundary timeout/offline

**Acceptance Criteria:**
- Given TimeTracker reports employee B on approved vacation 2026-08-25 to 2026-08-29, when an entitled viewer fetches S10 for B, then dates and type are returned from the feed with `availability: 'ok'`.
- Given the TimeTracker accounting API is unreachable, when a viewer fetches S10, then the response shows `availability: 'unavailable'` and the HTTP handler returns 200 without crashing the request.
- Given a Colleague-resolved viewer, when S10 is fetched, then leave dates are present and type is not exposed.

## Verification

**Commands:**
- `cd services/backend && npm test` — all unit tests pass including new integrations specs
- `cd services/backend && npm run lint` — zero errors
- `cd services/backend && npx tsc --noEmit` — clean compile

**Manual checks:**
- With TimeTracker test env configured, `GET /api/v1/employees/{id}/leaves` (authenticated) returns grouped leave periods for a seeded employee.

## Suggested Review Order

- S10 section provider: fail-soft display axis separate from access (AD-8).
  [`leaves-section.provider.ts:17`](../../services/backend/src/modules/integrations/leaves-section.provider.ts#L17)

- Accounting report fetch, cache, and leave-period normalization.
  [`leaves-sync.service.ts:36`](../../services/backend/src/modules/integrations/leaves-sync.service.ts#L36)

- Pure grouping logic from TimeTracker `DayStatus` rows.
  [`leave-period.mapper.ts:62`](../../services/backend/src/modules/integrations/leave-period.mapper.ts#L62)

- C5 real implementation replacing Wave-0 stub.
  [`external-identity-mapping.service.ts:11`](../../services/backend/src/modules/integrations/external-identity-mapping.service.ts#L11)

- Authenticated dev/test endpoint until ProfileAssembler (Story 1.6).
  [`leaves.controller.ts:27`](../../services/backend/src/modules/integrations/leaves.controller.ts#L27)

- Seed populates `(timetracker, manifest.id) → employee` mappings.
  [`seed.service.ts:122`](../../services/backend/src/prisma/seed/seed.service.ts#L122)

- C5 external_identities persistence shape.
  [`schema.prisma:278`](../../services/backend/prisma/schema.prisma#L278)

**Tests & wiring**

- Colleague type-masking and unavailable-state provider tests.
  [`leaves-section.provider.spec.ts:1`](../../services/backend/src/modules/integrations/__tests__/leaves-section.provider.spec.ts#L1)

- Module registration and C5 DI handoff from contracts.
  [`integrations.module.ts:1`](../../services/backend/src/modules/integrations/integrations.module.ts#L1)

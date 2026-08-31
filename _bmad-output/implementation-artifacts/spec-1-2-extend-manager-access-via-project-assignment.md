---
title: 'Extend Manager Access via Project Assignment'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '8a98d07e58fee155a17012000d696539cc87b0fc' # services/backend HEAD on feature/1-2-extend-manager-access-via-project-assignment — implementation happens in that submodule
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** `AccessResolver` (C1) only resolves `Self`/`ReportingLine` (Story 1.1) — Manager access via project assignment ("Project line": the PM/DM of a person's projects, and everyone above them through reports-to) doesn't exist. `ProjectAssignment` (C3) has a ratified contract but no persistence: it's bound to an empty Wave-0 stub.

**Approach:** Add a `ProjectAssignment` Prisma model, implement C3 for real in `access` (unbinding it from `ContractsModule`'s stub, mirroring how Story 1.1 unbound C1), and extend `AccessResolverService` to resolve `ProjectLine` from confirmed, fresh, still-active assignments — reusing the existing reports-to chain walk, rooted at each assignment's PM/DM instead of the subject.

## Boundaries & Constraints

**Always:**
- `ProjectLine` grants equal `ReportingLine`'s for every section except S2/S3 (`none`) and S7 (`RW` only via a DM leg, else `R`) — `access-model.md` Rule 2/3. S5's "CV/certs only" narrowing and S7's per-note "visible for PM" flag are out of scope: no `SectionProvider` exists yet to apply them (Story 1.6+).
- A row grants access only when `confirmed === true`, `confirmedAt` is non-null and within the freshness window (`Clock.now().getTime() - confirmedAt.getTime() <= 4h` — `decisions.md` D3/D19, boundary inclusive), and it's active (`startDate <= Clock.now()` **and** `endDate` null or `>= Clock.now()`) — re-checked every call, never cached. A null `confirmedAt` is always treated as not fresh, regardless of `confirmed`.
- Reuse `isInReportingLine`'s cycle-safe walk for "above the PM/DM" by rooting it at `pmId`/`dmId` — do not duplicate the algorithm.
- Time reads go through injectable `Clock` (never `new Date()`/`Date.now()`) — it already names the project-assignment freshness window as its reason for existing.
- C3 is unbound from `ContractsModule`, implemented for real in `access` (its CAP-1-owner per `interface-contracts.md`), mirroring Story 1.1's C1 move.

**Ask First:** none identified — proceed as scoped.

**Never:**
- Do not implement the department-management closure (AD-14) — reporting line stays reports-to only, as Story 1.1 scoped it.
- Do not build a `Project` entity/table — `projectId` stays an opaque string per C3's ratified shape.
- Do not touch `test/support/graph-factory.ts`/`access-matrix.ts` (still model a single `managerLine` audience, unused by any e2e suite today) — tracked in `deferred-work.md` as Story 1.15's job.
- Do not implement CAP-13's real timetracker sync or make `confirmedAt` self-refresh — `access` only writes rows here; Epic 13 becomes the writer later with zero consumer change.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior |
|----------|--------------|---------------------------|
| Direct DM | viewer=D; confirmed, active row: employeeId=B, dmId=D | `role: 'ProjectLine'`, S7 `'RW'` |
| Direct PM only | viewer=P, pmId=P; viewer not above dmId | `role: 'ProjectLine'`, S7 `'R'` |
| Above the DM | viewer=X, dmId=D, `D.managerId = X` | `role: 'ProjectLine'` via the reports-to walk rooted at D |
| Above the PM only | viewer=X, pmId=P, `P.managerId = X`; X is not D and not above D | `role: 'ProjectLine'` via the walk rooted at P, S7 `'R'` (no DM-leg match) |
| PM on one project, DM on another | two confirmed, active rows for the same subject; viewer matches `pmId` on row 1, `dmId` on row 2 | `role: 'ProjectLine'`, S7 `'RW'` — any surviving row's DM match sets `'RW'`, not just the first row checked |
| Unconfirmed | row has `confirmed = false` | falls through to `Colleague` |
| Stale confirmation | `confirmedAt` older than 4h, or `confirmedAt = null` | falls through to `Colleague` |
| Not yet started | `startDate` in the future, row otherwise confirmed and active | falls through to `Colleague` until `startDate` arrives |
| Ended assignment | `endDate` in the past | falls through to `Colleague`, no grace period |
| Ending exactly now | `endDate === Clock.now()` | still active — inclusive `>=` boundary — `role: 'ProjectLine'` |
| ReportingLine already granted | viewer also PM/DM of a shared project | `ReportingLine` short-circuits first — never narrower, no lost grant |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- no `ProjectAssignment` model yet
- `services/backend/src/modules/contracts/project-assignment.contract.ts` -- ratified C3 shape (`listByEmployee`/`listByProject`), unchanged
- `services/backend/src/modules/contracts/stubs/project-assignment.stub.ts` -- Wave-0 empty stub, stays unreferenced (precedent: `access-resolver.stub.ts`)
- `services/backend/src/modules/contracts/contracts.module.ts:33,44` -- current C3→stub binding
- `services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts` -- asserts that binding today
- `services/backend/src/clock/clock.service.ts` -- `Clock` abstraction, already built for this exact freshness-window case
- `services/backend/src/modules/access/access-resolver.service.ts` -- Story 1.1's `Self`/`ReportingLine` resolver; `isInReportingLine` (line 115) is the walk to reuse
- `services/backend/src/modules/access/access.module.ts` -- binds C1 only today
- `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` -- Story 1.1's mocked-Prisma suite to extend

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` -- add `ProjectAssignment`: `employeeId`/`pmId`/`dmId` as three distinct `Employee` relations, each with its own `@relation("...")` name (e.g. `"ProjectAssignmentEmployee"`/`"ProjectAssignmentPm"`/`"ProjectAssignmentDm"`, mirroring the existing `"ReportsTo"` precedent — Prisma requires distinct names to disambiguate 3 FKs to the same model) and `onDelete: Restrict`, `projectId String`, `startDate DateTime @db.Date` (non-nullable, matching C3's DTO), `endDate DateTime? @db.Date`, `confirmed Boolean @default(false)`, `confirmedAt DateTime?`, `@@index([employeeId])`, `@@index([projectId])` -- backs C3's ratified shape
- [x] Run `npm run db:migrate` inside `services/backend`
- [x] `services/backend/src/modules/access/project-assignment.service.ts` (new) -- implement `ProjectAssignment`: `listByEmployee`/`listByProject` mapping rows to `ProjectAssignmentDto`; plus `create(input)` (outside the contract) for the internal-write path, where `input` is `ProjectAssignmentDto` with `startDate`/`endDate`/`confirmedAt` as `Date`/`Date | null` instead of ISO strings, and `confirmed`/`confirmedAt` both caller-supplied (optional, defaulting to the schema's `false`/`null`) so a test or manual insert can produce an already-confirmed row directly -- the real C3 implementation
- [x] `services/backend/src/modules/access/access.module.ts` -- add `{ provide: ProjectAssignment, useClass: ProjectAssignmentService }` alongside C1
- [x] `services/backend/src/modules/contracts/contracts.module.ts` -- remove the `ProjectAssignment`/`ProjectAssignmentStub` import, provider entry, export
- [x] `services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts` -- remove the stub-binding assertion; add one asserting C3 is left unbound
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- inject `ProjectAssignment`/`Clock`; add `PROJECT_LINE_SECTIONS` (`REPORTING_LINE_SECTIONS` with S2/S3 `'none'`, S7 conditional); after the `ReportingLine` check, fetch `listByEmployee(subjectId)`, filter to confirmed+fresh+active rows, then for each surviving row check `isInReportingLine(viewerId, pmId)` and `(..., dmId)`; grant `ProjectLine` if either matched, S7 `'RW'` iff a `dmId` match occurred -- the story's core logic
- [x] `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` -- extend with the full I/O matrix above
- [x] `services/backend/src/modules/access/__tests__/project-assignment.service.spec.ts` (new) -- unit tests for `ProjectAssignmentService`, mocking `PrismaService`

**Acceptance Criteria:**
- Given employee B is assigned to Project X, whose PM is P and whose DM is D, when D's access with respect to B is resolved, then D holds `ProjectLine` access with respect to B
- Given PM P holds `ProjectLine` access to B solely via B's assignment to Project X, when B's assignment ends and P's access to B is resolved again, then P no longer holds `ProjectLine` access via that project, with no grace period
- Given `ProjectAssignment` rows exist with no timetracker integration live, when `listByEmployee`/`listByProject` are queried, then they return the seeded/manually-created rows unchanged — C3 is real domain data, not a passthrough to an external system

## Spec Change Log

**2026-08-31 — bmad-review resolution:**
- Boundaries & Constraints: the active-row rule now also checks `startDate <= Clock.now()` (previously only `endDate` was checked, silently granting access before an assignment's start).
- Boundaries & Constraints: a null `confirmedAt` is now explicitly not-fresh regardless of `confirmed`, and the 4h freshness boundary is now stated as inclusive (`<=`).
- I/O & Edge-Case Matrix: dropped the all-`N/A` "Error Handling" column; added rows for "above the PM only", "PM on one project/DM on another", "not yet started", and the inclusive `endDate === Clock.now()` boundary.
- Tasks: schema task now requires named `@relation`s for the three `Employee` FKs (Prisma requires disambiguation) and corrects `startDate` to non-nullable (matching C3's DTO shape; `endDate` stays nullable).
- Tasks: `ProjectAssignmentService.create(input)`'s shape is now specified explicitly (`Date`-typed dates, caller-supplied `confirmed`/`confirmedAt`).
- Acceptance Criteria: split the second AC into two — revocation-on-end, and `ProjectAssignment` querying independent of any live integration — each was one bundled assertion before.
- Verification: added a manual-check step exercising the revocation path (AC #2), not just the initial grant.
- Design Notes: added a note on why `resolveAudience`'s single-winning-role short-circuit is safe for this story only, and must become a real per-section union once PP (a non-nested audience) lands.

## Design Notes

**Why S7 is decided inside C1, but S5 isn't:** S7's DM/PM split is a section-level `RW`-vs-`R` difference — exactly what C1 already expresses. S5's "CV/certs only" is a document-type filter *within* one `R` grant, which the coarse `SectionAccessLevel` type can't express; it belongs to S5's future `SectionProvider` (Story 1.6+, branching on `ResolvedAudience.role`) — same precedent Story 1.1 set for field-level nuance.

**Why `resolveAudience` still returns one winning role, not a union:** `access-model.md` Rule 10 requires per-section union across all resolved audiences once more than one can independently apply — but `ReportingLine`'s section-set is a superset of `ProjectLine`'s (Rule 2), so short-circuiting to whichever is found first (`Self` > `ReportingLine` > `ProjectLine` > `Colleague`) loses nothing for this story specifically. This shape stops being safe once a non-nested audience (PP, Story 1.7+) joins the union — whoever adds PP needs to replace the short-circuit with a real per-section union, not extend the chain.

## Verification

**Commands:**
- `cd services/backend && npm run build` -- expected: compiles clean
- `cd services/backend && npm run lint` -- expected: no new lint errors
- `cd services/backend && npm test -- access-resolver` -- expected: extended unit tests pass, covering the full I/O matrix
- `cd services/backend && npm test -- project-assignment` -- expected: new C3 implementation unit tests pass
- `cd services/backend && npm test -- contracts.module` -- expected: updated to assert C3 left unbound

**Manual checks (if no CLI):**
- Confirm the generated migration only adds the `ProjectAssignment` table/FKs/indexes -- no unrelated schema drift
- Insert one confirmed, active `ProjectAssignment` row via `db:studio` or `ProjectAssignmentService.create`, then resolve access for its DM/PM and confirm `ProjectLine` resolves as expected
- Set that row's `endDate` to a past date (or null out `confirmedAt`), resolve access for the same DM/PM again, and confirm they fall back to `Colleague` with no grace period -- exercises AC #2's revocation path, not just the grant path above

### Review Findings

- [x] [Review][Patch] `resolveProjectLine` never short-circuits once `granted && dmMatched` are both already `true`, and re-walks `isInReportingLine` for every surviving row with no memoization across rows sharing a `pmId`/`dmId` [access-resolver.service.ts:154]
- [x] [Review][Patch] `isProjectAssignmentActive` reads `Clock` twice (`nowMs()` then `now()`) instead of taking one snapshot and deriving both values from it [access-resolver.service.ts:203]
- [x] [Review][Patch] No test exercises the malformed/unparseable `endDate` NaN-rejection guard (sibling `startDate` case is tested, `endDate` isn't) [access-resolver.service.ts:231]
- [x] [Review][Patch] No test exercises the malformed/unparseable `confirmedAt` NaN-rejection guard [access-resolver.service.ts:205]
- [x] [Review][Patch] `app.module.spec.ts`'s real-module-graph test (established by Story 1.1 specifically to catch a botched `ContractsModule`/`AccessModule` binding) wasn't extended to assert C3 `ProjectAssignment` resolves to `ProjectAssignmentService`, despite this story's own doc comments claiming to mirror that precedent [app.module.spec.ts]

## Suggested Review Order

**Core resolution logic**

- Entry point: after `Self`/`ReportingLine`, falls through to the new `ProjectLine` branch.
  [`access-resolver.service.ts:114`](../../services/backend/src/modules/access/access-resolver.service.ts#L114)

- Walks every surviving assignment (not just the first match) so a DM-leg match on any row sets S7 `RW`; reuses `isInReportingLine` rooted at `pmId`/`dmId` instead of the subject.
  [`access-resolver.service.ts:145`](../../services/backend/src/modules/access/access-resolver.service.ts#L145)

- The confirmed/fresh/active gate — patched post-review to compare date-only `startDate`/`endDate` against today's UTC midnight (not full-precision `now`), and to reject future-dated or unparseable inputs outright.
  [`access-resolver.service.ts:198`](../../services/backend/src/modules/access/access-resolver.service.ts#L198)

- Unchanged from Story 1.1 — the cycle-safe manager-chain walk this story reuses rooted at `pmId`/`dmId`.
  [`access-resolver.service.ts:253`](../../services/backend/src/modules/access/access-resolver.service.ts#L253)

**Schema change**

- `ProjectAssignment` — three distinct `Employee` relations (employee/PM/DM), `onDelete: Restrict`, `startDate`/`endDate` as `@db.Date` (the source of the date-only comparison bug the patch fixed).
  [`schema.prisma:76`](../../services/backend/prisma/schema.prisma#L76)

**Real C3 implementation**

- `ProjectAssignmentService` — real Prisma-backed `listByEmployee`/`listByProject`, CAP-1-owned.
  [`project-assignment.service.ts:27`](../../services/backend/src/modules/access/project-assignment.service.ts#L27)

- `create()` — outside the C3 contract, the internal-write path for seeding/manual inserts.
  [`project-assignment.service.ts:52`](../../services/backend/src/modules/access/project-assignment.service.ts#L52)

**Contract and DI rewiring**

- C3 deliberately left unbound here — `access` implements it directly, mirroring Story 1.1's C1 move.
  [`contracts.module.ts:23`](../../services/backend/src/modules/contracts/contracts.module.ts#L23)

- Real DI binding: `ProjectAssignment` → `ProjectAssignmentService`, alongside the existing C1 binding.
  [`access.module.ts:25`](../../services/backend/src/modules/access/access.module.ts#L25)

- Asserts C3 is left unbound in `ContractsModule`, mirroring the existing C1 test.
  [`contracts.module.spec.ts:48`](../../services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts#L48)

**Tests**

- Full `ProjectLine` I/O-matrix coverage, including the post-review boundary/edge-case additions.
  [`access-resolver.service.spec.ts:185`](../../services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts#L185)

- The date-only boundary fix's regression test — real `@db.Date` shape, not a full-precision timestamp.
  [`access-resolver.service.spec.ts:309`](../../services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts#L309)

- Confirms the freshness window's `>` (not `>=`) semantics at the exact 4h boundary.
  [`access-resolver.service.spec.ts:331`](../../services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts#L331)

- `ProjectAssignmentService` unit tests, mocking `PrismaService`.
  [`project-assignment.service.spec.ts`](../../services/backend/src/modules/access/__tests__/project-assignment.service.spec.ts#L1)

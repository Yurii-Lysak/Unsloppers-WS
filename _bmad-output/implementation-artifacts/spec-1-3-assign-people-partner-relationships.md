---
title: 'Assign People Partner Relationships'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '0ce33924548588af4ddb359f0bd17e32d4cd2f0d' # services/backend HEAD on main — implementation happens in that submodule
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C1 `AccessResolver` resolves `Self`/`ReportingLine`/`ProjectLine` only (Stories 1.1–1.2). There is no `peoplePartnerId` on `Employee`, so People Partner access — assignment plus the HR line above the assigned PP — cannot be resolved, and reassignment cannot take effect.

**Approach:** Add a singular `peoplePartnerId` self-relation on `Employee`, then extend `AccessResolverService` to resolve `PP` from the live assignment plus an HR-scoped walk up the assigned PP's `managerId` chain (not the subject's chain). Refactor `resolveAudience` to a per-section union (D13 / Rule 10), because `PP` overlaps `ProjectLine` and `ReportingLine` on some sections — replacing Story 1.2's single-winning-role short-circuit.

## Boundaries & Constraints

**Always:**
- `resolveAudience` reads live `peoplePartnerId` on every call — never cached (same "next request" rule as manager/PP platform-owned relations).
- Only one PP per employee — a nullable FK on `Employee`, not a history table.
- PP resolution for viewer V and subject B: grant when `B.peoplePartnerId === V`, **or** when V appears on the HR line above the assigned PP (walk `managerId` upward from `B.peoplePartnerId`, cycle-safe; at each node require open `DepartmentHistory.value === HR_DEPARTMENT_VALUE` before treating the node as on the HR line; stop at the first node outside HR — do not climb into the delivery hierarchy).
- `HR_DEPARTMENT_VALUE` comes from config (env, default `'HR'`) — Wave-0 uses exact, case-sensitive string match on the open department-history row; nested HR sub-departments wait for the Department entity story.
- `PP` section grants are the constants in Design Notes (sourced from `access-model.md` matrix PP column). Effective `sections` = least-restrictive union across every matched audience (`Self` excluded from union with others). `role` is a backward-compat label only (rank: Self > ReportingLine > PP > ProjectLine > Colleague); `sections` is authoritative.
- Reassigning or clearing `peoplePartnerId` revokes the previous PP (and their HR line) on the very next call — no grace period.

**Ask First:** none identified — proceed as scoped.

**Never:**
- Do not build C9 `OrgRelationshipWriter`, C10 `RelationshipJournal`, functional-permission checks, or admin/UI write screens — internal test/seed write helper only (mirrors Story 1.2's `ProjectAssignmentService.create()` pattern; C9 write guards such as no self-assignment are intentionally deferred — document that gap, do not enforce in the helper).
- Do not render the profile header (Story 1.7). This story only persists `peoplePartnerId` on `Employee`.
- Do not add bootcamp-seed PP demo rows — unit tests mock Prisma (deferred-work precedent from Story 1.2).
- Do not implement nested-department HR closure or the Department entity/admin screen — exact `HR_DEPARTMENT_VALUE` match only until that story lands.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior |
|----------|--------------|---------------------------|
| Direct PP | `B.peoplePartnerId = X`; viewer X | `role: 'PP'`, full PP section grant |
| HR line above PP | `B.peoplePartnerId = X`, `X.managerId = H`, H's open dept = HR; viewer H | `role: 'PP'` via HR-line walk |
| Multi-hop HR line | `X.managerId = H`, `H.managerId = G`, both H and G open dept = HR; viewer G | `role: 'PP'` via HR-line walk |
| HR line stops outside HR | `X.managerId = H`, H's open dept ≠ HR; viewer H | falls through — H gets no PP access via this path |
| Missing open department | node on HR walk has no `DepartmentHistory` row with `effectiveTo: null` | treat node as outside HR — stop walk, no PP via that path |
| Reassign PP | was `B.peoplePartnerId = X`, now `= Y`; resolve X→B | no PP access; Y→B grants PP |
| Clear PP | `B.peoplePartnerId` set null; resolve former PP X→B | `Colleague` (no PP leg) |
| PP + ProjectLine union | viewer is DM via project and PP via assignment; S2 ProjectLine `none`, PP `RW` | `sections.S2 = 'RW'`; `role: 'PP'` |
| ReportingLine + PP union | viewer matches ReportingLine and PP; S2 ReportingLine `R`, PP `RW` | `sections.S2 = 'RW'`; `role: 'ReportingLine'` |
| ReportingLine still wins rank | viewer matches ReportingLine and PP | unioned sections; `role: 'ReportingLine'` |
| Self unchanged | viewer === subject | `Self` short-circuit — PP leg not evaluated |
| Cyclical PP chain | HR-line walk revisits an id before viewer matched | log warning (mirror `isInReportingLine`); no PP via HR path; evaluate remaining audience legs |
| Deleted PP employee | assigned PP `Employee` row deleted (`onDelete: SetNull`) | `B.peoplePartnerId` null — no PP leg |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- `Employee` has no `peoplePartnerId` today -- add nullable FK + self-relation
- `services/backend/src/modules/access/access-resolver.service.ts:95` -- class comment still says PP lands in 1.7–1.10; update when implementing
- `services/backend/src/modules/access/access-resolver.service.ts:114` -- `resolveAudience` short-circuits one role -- refactor to union; add `resolvePp` + HR-line walk
- `services/backend/src/modules/access/access-resolver.service.ts:268` -- `isInReportingLine` -- reuse cycle-safe walk pattern for HR line (different root + dept gate)
- `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` -- extend with PP matrix + union regression; update Story 1.2 test at ~line 393 (ProjectLine may still skip when ReportingLine matches — PP must not)
- `services/backend/src/config/env.validation.ts` -- add optional `HR_DEPARTMENT_VALUE` (default `'HR'`)
- `services/backend/.env.example` -- document `HR_DEPARTMENT_VALUE`
- `services/backend/src/modules/access/project-assignment.service.ts` -- precedent for internal-write service in `access` module
- `services/backend/src/modules/access/access.module.ts` -- register new internal-write service if split out
- `_bmad-output/specs/spec-people-management-platform/access-model.md:25-26,96-111` -- normative PP / HR-line rules + PP column grants (workspace root, not under `services/backend`)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` -- add `peoplePartnerId String?`, `peoplePartner Employee? @relation("PeoplePartnerAssignment", ...)`, `ppAssignments Employee[] @relation("PeoplePartnerAssignment")`, `onDelete: SetNull`, `@@index([peoplePartnerId])` on `Employee`
- [x] Run `npm run db:migrate` inside `services/backend`
- [x] `services/backend/src/config/env.validation.ts` + `.env.example` -- add `HR_DEPARTMENT_VALUE` string, default `'HR'`
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- inject `ConfigService`; read `HR_DEPARTMENT_VALUE` (default `'HR'`) at construction
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- add `PP_SECTIONS` constant (Design Notes); add `getOpenDepartmentValue(employeeId)` via `departmentHistory.findFirst({ effectiveTo: null })`; add `isInHrLine(viewerId, assignedPpId)` (direct PP match + HR-gated upward walk per Design Notes); add `resolvePp(viewerId, subjectId)` reading `subject.peoplePartnerId` (null/missing subject → no PP leg, do not throw)
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- refactor `resolveAudience`: keep `Self` short-circuit; collect matched audiences (`ReportingLine`, `ProjectLine` w/ conditional S7, `PP`); union sections via helper (`none` < `R` < `RW`); set `role` by fixed rank above; **short-circuit rules:** `ProjectLine` may skip when `ReportingLine` already matched (ReportingLine section-set is a superset of ProjectLine per Rule 2); **`PP` must always be evaluated** when not `Self` — PP widens S2/S3/S5 over ReportingLine
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- update class/module doc comment: PP resolves here (Story 1.3), not 1.7–1.10
- [x] `services/backend/src/modules/access/people-partner-assignment.service.ts` (new) -- internal `assign(subjectId, peoplePartnerId | null)` doing a direct Prisma update (not C9) for tests/manual ops
- [x] `services/backend/src/modules/access/access.module.ts` -- register/export the new service
- [x] `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` -- cover full I/O matrix; add union cases for PP+ProjectLine and ReportingLine+PP; update Story 1.2 short-circuit test expectations per Design Notes short-circuit rules
- [x] `services/backend/src/modules/access/__tests__/people-partner-assignment.service.spec.ts` (new) -- unit test assign/reassign/clear

**Acceptance Criteria:**
- Given HR Admin assigns People Partner X to employee B, when X's access with respect to B is resolved, then X holds `PP` access to B, and X's HR line above them also resolves as holding `PP` access to B
- Given People Partner X currently holds PP access to employee B, when B's PP is reassigned to Y and X's access to B is resolved again, then X no longer holds PP access to B, and Y does, with no delay
- Given `peoplePartnerId` is persisted on `Employee`, when Story 1.7's list/profile consumers are implemented, then the current assignment is readable from B's row — no UI or DTO wiring required in this story

## Design Notes

PP section grants (coarse, section-level — `access-model.md` PP column):

- `PP`: S1 RW, S2 RW, S3 RW, S4 RW, S5 RW, S6 RW, S7 RW, S8 RW, S9 RW, S10 R, S11 R, S12 RW, S13 RW, S14 RW, S15 R, S16 RW

**Wave-0 `DepartmentHistory` shape:** the schema has `@@unique([employeeId])` — one row per employee today. `getOpenDepartmentValue` uses `findFirst({ where: { employeeId, effectiveTo: null } })`. A missing open row or a closed row (`effectiveTo` set) means the employee is treated as outside HR for the walk. A future multi-row history story may change this query; do not assume temporal rows exist yet.

HR-line walk (distinct from `isInReportingLine(subject)`):

```
assignedPpId = subject.peoplePartnerId; if !assignedPpId → no PP
if viewerId === assignedPpId → match
currentId = assignedPpId's managerId
while currentId:
  if getOpenDepartmentValue(currentId) !== HR_DEPARTMENT_VALUE → stop (outside HR department)
  if viewerId === currentId → match
  cycle-guard (revisit → log warning, stop — no PP via HR path)
  currentId = currentId.managerId
```

Union helper: start from `COLLEAGUE_SECTIONS`, for each matched audience merge with `maxAccess(a, b)` where `none < R < RW`. `ProjectLine` still contributes its conditional S7 before merging.

**Union short-circuit (Story 1.2 → 1.3):** Story 1.2's `ReportingLine`-before-`ProjectLine` short-circuit remains safe because ReportingLine's section-set is a superset of ProjectLine's (Rule 2). That stops being sufficient once PP joins: PP is *not* a subset of ReportingLine (PP grants S2/S3/S5 `RW` where ReportingLine grants `R`). After this story: always evaluate the PP leg (unless `Self` short-circuited); `ProjectLine` may still skip when ReportingLine matched.

## Verification

**Commands:**
- `cd services/backend && npm run build` -- expected: compiles clean
- `cd services/backend && npm run lint` -- expected: no new lint errors
- `cd services/backend && npm test -- access-resolver` -- expected: PP + union tests pass
- `cd services/backend && npm test -- people-partner-assignment` -- expected: internal-write tests pass

**Manual checks (if no CLI):**
- Confirm migration only adds `peoplePartnerId` column/FK/index
- Set `peoplePartnerId` via `db:studio`, resolve access for PP and their HR manager, then reassign and confirm immediate revocation

### Review Findings

- [x] [Review][Patch] Reassignment test does not simulate X→Y transition — fixed: test now assigns `X`, verifies PP, reassigns to `Y`, verifies revocation [`access-resolver.service.spec.ts:541`]
- [x] [Review][Patch] HR-line PP tests omit full section-grant assertions — fixed: HR-line happy-path tests now assert `sections` equal `PP_SECTIONS` [`access-resolver.service.spec.ts:497`]
- [x] [Review][Patch] `HR_DEPARTMENT_VALUE` config coverage incomplete — fixed: added non-default config and case-sensitive mismatch tests [`access-resolver.service.spec.ts:33`]
- [x] [Review][Patch] `effectiveTo: null` filter not pinned — fixed: department mock rejects non-null `effectiveTo`; HR-line test asserts call shape [`access-resolver.service.spec.ts:119`]
- [x] [Review][Patch] Cyclical HR-line test does not assert warning log — fixed: asserts `Logger.warn` on HR-only cycle [`access-resolver.service.spec.ts:595`]
- [x] [Review][Patch] Story 1.2 short-circuit test does not verify PP leg still runs — fixed: test sets assigned PP and asserts lookup + S2 union [`access-resolver.service.spec.ts:473`]
- [x] [Review][Defer] Removed resolver JSDoc on `isProjectAssignmentActive` / `isInReportingLine` — deferred, pre-existing doc regression from this diff [`access-resolver.service.ts`]
- [x] [Review][Defer] `access-resolver.contract.ts` lacks post–Story 1.3 union/role-rank documentation — deferred, contract doc update out of scope for this story [`access-resolver.contract.ts`]

---
title: 'Derive Manager Access from Reporting Hierarchy'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'bb9531cf6f7888602f49f3b220ef8fd2bf135323' # services/backend HEAD — implementation happens in that submodule; rebased mid-story from 9c376b3 onto a teammate's fast-forwarded PR (bb9531c, TEA e2e harness) with no conflicts
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Nothing in the system yet grants Manager access via the reporting chain — `Employee` has no manager relationship in the schema, and C1 `AccessResolver` is still bound to a deny-everything Wave-0 stub (`AccessResolverStub`).

**Approach:** Add a `managerId` self-relation to `Employee`, then implement the real `AccessResolver` in a new `access` module that resolves `Self` (viewer === subject) and `ReportingLine` (transitive, cycle-safe walk up the manager chain) — leaving every other role on today's safe deny-by-default until its own story lands.

## Boundaries & Constraints

**Always:**
- `resolveAudience` is recomputed from live `managerId` values on every call — never cached across requests or sessions (per AC and `access-model.md`'s "next request" revocation rule).
- `viewerId`/`subjectId` are `Employee.id` values (the profile id), consistent with `managerId` targeting `Employee.id`. Translating an authenticated `userId` (C7) to its `Employee.id` is the caller's job, not this contract's.
- The manager-chain walk is cycle-safe via a visited-id guard, independent of the write-time cycle guard (D15) — a corrupted chain must terminate, never loop, and must never let a cycle back to the subject read as a Manager-access grant.
- `AccessRole` is updated to the ratified 2026-08-26 set (`Self | ReportingLine | ProjectLine | PP | Colleague | SharedLink | FullAccess`) per `ARCHITECTURE-SPINE.md` AD-2 — the current contract file still has the stale `ManagerLine`/`HRAdmin` pre-split names.
- `AccessResolver`'s stub binding is removed from `ContractsModule`, mirroring how C7 `CurrentUserProvider` was left unbound for `auth` to implement directly.

**Ask First:** none identified — proceed as scoped.

**Never:**
- Do not implement the department-management closure (AD-14) or the project-assignment closure (Story 1.2) — this story is reports-to only.
- Do not implement the accurate Colleague whitelist (Story 1.8) or any field-level nuance (Story 1.7/1.9/1.10) — any role besides `Self`/`ReportingLine` keeps the existing coarse deny-all-sections default.
- Do not add manager relationships to the bootcamp seed data (`bootcamp-identities.json`) — out of scope; unit tests exercise the chain via mocked Prisma calls.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Direct report | viewer=M, subject=B, `B.managerId = M` | `role: 'ReportingLine'`, full Reporting-line section grant | N/A |
| Transitive (2 levels) | viewer=D, subject=B, `B.managerId = M`, `M.managerId = D` | `role: 'ReportingLine'` | N/A |
| Unrelated | viewer=A, subject=B, no overlap in B's manager chain | `role: 'Colleague'`, all sections `none` | N/A |
| Self | viewer=X, subject=X | `role: 'Self'` | N/A |
| Cyclical data | walking B's chain revisits an already-seen id before reaching viewer | `role: 'Colleague'` (safe default) — no infinite loop | terminate via visited-id set, never throw |
| Invalid id | subject's chain walk reaches an id with no matching Employee row (dangling/invalid id, or a row deleted mid-walk) | `role: 'Colleague'` (safe default) | stop the walk on the null lookup result, never dereference it, never throw |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- `Employee` model (no manager field today) -- add `managerId` self-relation
- `services/backend/src/modules/contracts/access-resolver.contract.ts` -- `AccessRole`/`resolveAudience` C1 contract -- update role enum to ratified set
- `services/backend/src/modules/contracts/contracts.module.ts` -- binds `AccessResolver` to `AccessResolverStub` today -- remove that binding (C1 becomes unbound here, like C7)
- `services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts` -- asserts the stub binding -- update to assert C1 is left unbound
- `services/backend/src/modules/contracts/stubs/access-resolver.stub.ts` -- Wave-0 deny-all stub -- leave in place, unreferenced (same precedent as `current-user-provider.stub.ts`)
- `services/backend/src/modules/registry/registry.module.ts` -- reference pattern for a controller-less `@Global()` module -- mirror its anatomy/comment for the new `access` module
- `services/backend/src/modules/users/users.service.ts` -- reference pattern for `PrismaService` DI in a feature service
- `services/backend/src/app.module.ts` -- module registration list -- add `AccessModule`
- `services/backend/_bmad-output/specs/spec-people-management-platform/access-model.md` -- Reporting-line and Self section columns (lines 94-111) -- source of the section-grant constants below (transcribed in Design Notes)

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` -- add `managerId String?`, `manager Employee? @relation("ReportsTo", fields: [managerId], references: [id], onDelete: SetNull)`, `directReports Employee[] @relation("ReportsTo")`, `@@index([managerId])` on `Employee` -- stores the reports-to edge the closure walks
- [x] Run `npm run db:migrate` inside `services/backend` to generate and apply the migration -- persists the schema change
- [x] `services/backend/src/modules/contracts/access-resolver.contract.ts` -- change `AccessRole` to `'Self' | 'ReportingLine' | 'ProjectLine' | 'PP' | 'Colleague' | 'SharedLink' | 'FullAccess'` -- aligns with `ARCHITECTURE-SPINE.md` AD-2
- [x] `services/backend/src/modules/access/access-resolver.service.ts` (new) -- implement `AccessResolver`: return `Self` when `viewerId === subjectId` and both are non-empty; else walk `subjectId`'s `managerId` chain (visited-id `Set` guard) looking for `viewerId`, returning `ReportingLine` on a match; if a chain lookup finds no Employee row for the current id, stop the walk and fall through to `Colleague` rather than dereferencing a null result; else `Colleague` with every section `none` -- the story's core logic. Section grants for `Self`/`ReportingLine` are the constants transcribed in Design Notes below (sourced verbatim from `access-model.md`'s matrix, coarse section-level R/RW/none, no field-level nuance)
- [x] `services/backend/src/modules/access/access-resolver.service.ts` -- when the visited-id guard actually breaks the walk (a real cycle, not just chain exhaustion), log a warning including `subjectId` -- surfaces org-data corruption instead of silently degrading to `Colleague`
- [x] `services/backend/src/modules/access/access.module.ts` (new) -- `@Global()`, no controller, `providers: [{ provide: AccessResolver, useClass: AccessResolverService }]`, `exports: [AccessResolver]` -- mirrors `registry.module.ts`'s documented anatomy exception
- [x] `services/backend/src/modules/contracts/contracts.module.ts` -- remove the `AccessResolver`/`AccessResolverStub` import, provider entry, and export -- leaves C1 unbound here for `access` to implement
- [x] `services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts` -- remove the `[AccessResolver, AccessResolverStub]` table row and the "denies every section by default" test; add `it('leaves C1 unbound for the access module to implement', ...)` mirroring the existing C7 test; also remove the now-unused `AccessResolver`/`AccessResolverStub` imports at the top of the file
- [x] `services/backend/src/app.module.ts` -- import and register `AccessModule`
- [x] `services/backend/src/modules/app.module.spec.ts` (new, or extend an existing app-level test) -- boot `AppModule` and assert `AccessResolver` resolves to `AccessResolverService`, not a stale stub -- catches a botched or skipped `ContractsModule` binding removal that a `ContractsModule`-only test can't see
- [x] `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts` (new) -- unit tests for the I/O matrix above, mocking `PrismaService.employee.findUnique`; also cover a chain lookup that returns no row (invalid/dangling id mid-walk resolves to `Colleague`, no throw) and empty-string `viewerId`/`subjectId` inputs

**Acceptance Criteria:**
- Given employee B reports to Manager M, and M reports to Director D, when D's access with respect to B is resolved, then D holds `ReportingLine` access with respect to B, without any explicit grant, recomputed on every call
- Given employee A and employee B share no reporting relationship, direct or transitive, when A's access with respect to B is resolved, then A does not hold `ReportingLine` access to B via this path, the closure never loops on a cyclical chain, and no cycle is ever read as granting `Self` access to self
- Given employee B reports directly to Manager M, when M's access with respect to B is resolved, then M holds `ReportingLine` access with respect to B
- Given viewer X and subject X are the same employee, when X's access with respect to X is resolved, then X holds `Self` access, and this check never depends on a `managerId` chain lookup

### Review Findings

- [x] [Review][Patch] ReportingLine section-grant test doesn't pin exact values — passes even if `REPORTING_LINE_SECTIONS` had an R/RW entry scrambled [services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts:60-63]
- [x] [Review][Patch] Spec's Verification command list omits `npm test -- app.module`, the test written specifically to catch a botched DI rewiring [_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md]
- [x] [Review][Patch] Missing edge-case test coverage: single-node self-cycle (`managerId` pointing at own id) and asymmetric empty ids [services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts:46,87]
- [x] [Review][Defer] AD-18's C1 "dismissed caps every section to R" rule isn't implemented or flagged as a gap in this story's "Never" boundary list (blocked on the not-yet-built `employment`/C11 module) [services/backend/src/modules/access/access-resolver.service.ts] — deferred, pre-existing
- [x] [Review][Defer] New `app.module.spec.ts` — this story's own DI-wiring regression guard — cannot execute under the repo's pinned Node 22 toolchain (`@nestjs/jwt` ESM build needs Node ≥24.9 for Jest's `require(ESM)`); confirmed pre-existing via the same failure on `src/__tests__/app-startup.spec.ts` [services/backend/src/modules/app.module.spec.ts] — deferred, pre-existing
- [x] [Review][Defer] Empty/falsy `viewerId`/`subjectId` silently resolve to `Colleague` instead of surfacing as a likely caller/auth-integration bug [services/backend/src/modules/access/access-resolver.service.ts:92] — deferred, pre-existing
- [x] [Review][Defer] No DB-level guard (e.g. `CHECK (managerId <> id)`) against a self-referencing `managerId`; only the read-time visited-set walk defends against it today, and the future write-time guard (C9/D15) doesn't exist yet [services/backend/prisma/schema.prisma] — deferred, pre-existing
- [x] [Review][Defer] No automated test exercises the real `onDelete: SetNull` FK behavior — all `AccessResolverService` tests mock Prisma, and the spec's own Verification section already scopes this to a manual check [services/backend/prisma/schema.prisma:52-56] — deferred, pre-existing

## Design Notes

Chain walk, not a recursive CTE: `managerId` is single-valued per employee, so resolving "does viewer appear above subject" is a plain loop — `let currentId = subjectId; while (currentId) { if (currentId === viewerId) return ReportingLine; if (visited.has(currentId)) break; visited.add(currentId); const emp = await lookup(currentId); if (!emp) break; currentId = emp.managerId; }` — one query per level, bounded by the visited set and by a null lookup result (a dangling/invalid id ends the walk, falls through to `Colleague`, never throws). A recursive SQL CTE would be a valid future optimization for very deep chains but is unwarranted complexity for this story.

Accepted risks, not addressed by this story: the walk is not wrapped in a single read transaction, so a concurrent `changeManager` write between two of its queries can (rarely) yield an inconsistent role for that one call; and because `resolveAudience` is never cached (per the "Always" constraint), each call costs one sequential query per chain level with no depth/latency ceiling defined. Both are acceptable for Wave-0 scope but worth flagging if org hierarchies grow deep or access checks become hot-path.

`onDelete: SetNull` on `managerId` means deleting a manager's Employee row orphans their direct reports (they become top-of-chain) rather than re-linking them to the grand-manager — manager offboarding silently narrows `ReportingLine` access for that subtree until someone manually reassigns it. No task in this story backfills that reassignment.

Section grant constants (coarse, section-level only — see `access-model.md` §"Section access matrix" for the full per-field detail future stories refine):

- `ReportingLine`: S1 RW, S2 R, S3 R, S4 RW, S5 R, S6 RW, S7 RW, S8 RW, S9 RW, S10 R, S11 R, S12 RW, S13 RW, S14 RW, S15 R, S16 RW
- `Self`: S1 R, S2 RW, S3 RW, S4 R, S5 R, S6 none, S7 R, S8 R, S9 R, S10 R, S11 R, S12 R, S13 RW, S14 R, S15 none, S16 R
- `Colleague` (unchanged default pending Story 1.8): every section `none`

## Verification

**Commands:**
- `cd services/backend && npm run build` -- expected: compiles clean, no TS errors from the contract/role-enum change
- `cd services/backend && npm run lint` -- expected: no new lint errors
- `cd services/backend && npm test -- access-resolver` -- expected: new unit tests pass, covering direct report, transitive report, unrelated employees, self, and a cyclical chain
- `cd services/backend && npm test -- contracts.module` -- expected: updated `contracts.module.spec.ts` passes with C1 left unbound
- `cd services/backend && npm test -- app.module` -- expected: asserts C1 resolves to `AccessResolverService` through the real module graph; currently cannot execute under the repo's pinned Node 22 toolchain (see Review Findings / deferred-work.md) until the `@nestjs/jwt` ESM/Jest incompatibility is resolved

**Manual checks (if no CLI):**
- Confirm the generated migration under `services/backend/prisma/migrations/` only adds the `managerId` column/FK/index — no unrelated schema drift
- Delete a manager's Employee row locally (or in a scratch DB) and confirm dependents' `managerId` is set to `null` rather than the delete being blocked — validates `onDelete: SetNull` actually behaves as specified, since the mocked unit tests can't exercise real FK behavior

## Suggested Review Order

**Core resolution logic**

- Entry point: role resolution — `Self` short-circuit, then the `ReportingLine` chain-walk delegate.
  [`access-resolver.service.ts:88`](../../services/backend/src/modules/access/access-resolver.service.ts#L88)

- The cycle-safe, dangling-id-safe manager-chain walk — the story's core algorithm.
  [`access-resolver.service.ts:115`](../../services/backend/src/modules/access/access-resolver.service.ts#L115)

- Warns on a genuine cycle instead of silently degrading — surfaces org-data corruption.
  [`access-resolver.service.ts:132`](../../services/backend/src/modules/access/access-resolver.service.ts#L132)

**Schema change**

- `managerId` self-relation on `Employee` — the reports-to edge the closure walks.
  [`schema.prisma:54`](../../services/backend/prisma/schema.prisma#L54)

- `onDelete: SetNull` orphans direct reports to top-of-chain, no auto-reassignment.
  [`schema.prisma:55`](../../services/backend/prisma/schema.prisma#L55)

**Contract and DI rewiring**

- `AccessRole` updated to the ratified `ReportingLine`/`ProjectLine`/`FullAccess` set, replacing the stale pre-split names.
  [`access-resolver.contract.ts:37`](../../services/backend/src/modules/contracts/access-resolver.contract.ts#L37)

- C1 deliberately left unbound here — `access` implements it directly, mirroring C7.
  [`contracts.module.ts:21`](../../services/backend/src/modules/contracts/contracts.module.ts#L21)

- Real DI binding: `AccessResolver` → `AccessResolverService`, no controller, mirrors `registry`.
  [`access.module.ts`](../../services/backend/src/modules/access/access.module.ts#L19)

- Wires `AccessModule` into the real app graph.
  [`app.module.ts:23`](../../services/backend/src/app.module.ts#L23)

**Tests**

- Full I/O-matrix coverage: direct report, transitive, unrelated, self, cycle, dangling id.
  [`access-resolver.service.spec.ts`](../../services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts#L39)

- Asserts C1 resolves through the *real* module graph, not a stale stub.
  [`app.module.spec.ts:26`](../../services/backend/src/modules/app.module.spec.ts#L26)

- Confirms `ContractsModule` no longer binds C1 — matches the C7 precedent.
  [`contracts.module.spec.ts:46`](../../services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts#L46)

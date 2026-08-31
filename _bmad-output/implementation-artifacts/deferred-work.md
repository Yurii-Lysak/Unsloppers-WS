- source_spec: `_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md`
  summary: Backend CI does not yet run the unit test suite — the workflow added by this story only runs `depcruise`.
  evidence: The spec's own frozen Boundaries & Constraints explicitly scoped this story's CI to "no full CI pipeline beyond the dependency-cruiser gate (build/test/lint stay out of scope)", so this is deliberate, not an oversight — but it means the 21 tests in `contracts`/`registry` (and any future test) won't be caught by CI until a follow-up story adds a test-execution job to `.github/workflows/ci.yml`.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md`
  summary: The dependency-cruiser module-boundary rules have no automated regression test proving they still catch a violation after a future config edit.
  evidence: Verified manually twice during this story (a deliberate `users`→`health` import and a deliberate `contracts`→`prisma` import, both caught, both reverted) — but that proof is a point-in-time manual check, not a committed fixture. A future edit to `.dependency-cruiser.cjs` (e.g. a regex typo) could silently stop catching violations with nothing in CI or tests to notice.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md`
  summary: C2 `FieldRegistry.query()` has no pagination/limit parameters, so a call with neither `employeeIds` nor `fieldIds` supplied is an unbounded full-company query.
  evidence: Flagged by review as a real scalability gap, but resolving it requires designing the actual query/pagination shape — that belongs with whoever builds the real `directory` module (Epic 3, Story 3.2), not this Wave-0 stub/contract story.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md`
  summary: C1 `AccessResolver.resolveAudience()` and C7 `CurrentUserProvider.getCurrentUser()` don't specify behavior for a nonexistent/unauthenticated subject (throw vs. null vs. sentinel).
  evidence: The Wave-0 stubs are safe either way (they deny/return a fixed value regardless of input validity), so this doesn't block Wave-1 today, but the real implementations (owned by `access` and Story 1.18/Authentication respectively) will need to decide this — C7's contract file already carries an explicit cross-story coordination note for the same reason.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-16-pseudonymized-seed-data-tool.md`
  summary: '`npm run start:prod` (`node dist/main`) is broken — `nest build`''s real compiled entrypoint is `dist/src/main.js`, not `dist/main.js`.'
  evidence: Reproduced directly (`node dist/main` → `MODULE_NOT_FOUND`). Confirmed pre-existing and unrelated to Story 1.16 — `nest-cli.json` and both `tsconfig*.json` files are untouched by this story's diff, so the same mismatch exists on the baseline commit. Surfaced incidentally by this story's review because it doubled the `nest build` invocation in `postbuild`, but the broken path is independent of that; whoever owns Story 1.17 (single-container deployment) needs `start:prod` actually working before the containerized deploy target can run the built app.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md`
  summary: `ProviderRegistryService.get<T>()` performs an unchecked `as T` cast with no per-family provider marker interface (e.g. `SectionProvider`/`FieldProvider`/`DashboardSummaryProvider`).
  evidence: Defining those interfaces now would be speculative — no concrete section/field/dashboard-summary provider exists yet to shape them against. Relevant once the first real provider in any family is designed (Epic 3+).

- source_spec: `_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md`
  summary: C4 `TimelineEventWriter.recordTimelineEvent()`/`markSystemWriteSkipped()` take no transaction/client parameter, so the "same-transaction, rollback-on-failure" coupling this story documents only holds while C4 is an in-process Wave-0 stub.
  evidence: Once Epic 7 supplies a real `timeline`-module implementation backed by its own `PrismaService` call, it structurally cannot participate in the calling transaction unless the contract signature changes to accept one — the atomicity guarantee will silently stop holding with no interface change to catch it. Flagged by the blind-hunter review; resolving it means renegotiating a Story-1.19-frozen contract, out of this story's scope.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md`
  summary: Manual-conflict detection is an exact match on `(employeeId, type, effectiveDate)`, not a range/overlap check — a manual correction on a nearby-but-different date doesn't stop a system write from closing the currently-open row and inserting a new one that effectively spans across it.
  evidence: This is the deliberately chosen, frozen design (C4's `effectiveDate` parameter is a single date, not a range), not an implementation defect — but it's a known limitation worth revisiting once Epic 7 designs the real manual-edit/conflict semantics for the `timeline` module.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md`
  summary: `TimelineEvent` has no write-protection extension of its own — the entire manual-conflict safeguard rests on the unenforced convention that `source: 'manual'` rows are only ever written by trusted code.
  evidence: Deliberately out of this story's scope (no real `timeline` module/service exists yet — Epic 7 owns it, per the frozen "Never" boundary). Real enforcement needs to land alongside whatever writes manual entries for real.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-20-temporal-employment-history-tables-and-timeline-coupling.md`
  summary: The temporal-history extension's `create()` on the four history models doesn't support caller-supplied `select`/`include`/extra fields — it rebuilds the write data from only `employeeId`/`value`/`effectiveFrom`, silently dropping anything else a caller passes.
  evidence: No real caller exists yet (no service/endpoint in this story's scope), so nothing is broken today, but the first real caller (Story 1.1/1.6+) that wants a shaped `select` response will need this extended. Flagged by the edge-case reviewer.

## Deferred from: code review of spec-1-16-pseudonymized-seed-data-tool (2026-08-28)

- source_spec: `_bmad-output/implementation-artifacts/spec-1-16-pseudonymized-seed-data-tool.md`
  summary: `TimelineEventWriterStub` no-ops — seed creates four history rows per employee but `timeline_events` stays empty until Epic 7 supplies a real C4 implementation.
  evidence: Pre-existing Wave-0 stub from Story 1.19; seed correctly calls extension-mediated `create`, but the stub writer records nothing. Career Timeline will have no events after seeding until Epic 7.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-16-pseudonymized-seed-data-tool.md`
  summary: Compiled seed entrypoint (`node dist/prisma/seed.js`) and `postbuild` migrate→seed chain have no automated CI smoke test.
  evidence: Same root cause as Story 1.19 CI gap — workflow runs depcruise only, not `npm test` or build/postbuild. All 121 unit tests pass locally but deploy-path regressions (broken DI in `prisma/seed.ts`, removed `&& npm run db:seed`) would not fail CI.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-16-pseudonymized-seed-data-tool.md`
  summary: `postbuild` unconditionally chains seed with no environment gate — runs manifest seed on every build.
  evidence: Deliberate for current single-environment scope per spec Design Notes; Story 1.17 owns deployment topology and environment gating when real prod/staging separation exists. **Post-pivot (2026-08-28):** seed no longer calls TimeTracker — manifest-only, so postbuild does not require VPN/TT keys.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-16-pseudonymized-seed-data-tool.md`
  summary: `ExternalIdentityMapping` (C5) not populated during seed — TimeTracker numeric `id` is not mapped to platform `employeeId`.
  evidence: Review decision 1B (2026-08-28): defer to Epic 13 / follow-up substrate story. Spec Code Map originally named C5 population as this story's job; amended to note seed stores identity on `User` by email only until Epic 13 owns project-assignment and external-id mapping.

## Deferred from: code review of spec-1-1-derive-manager-access-from-reporting-hierarchy (2026-08-31)

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: '5 test suites — including this story''s new `app.module.spec.ts` DI-wiring safety net — fail to even load under the pinned Node 22, because `@nestjs/jwt`/`@nestjs/passport` ship ESM-only dist builds that Jest''s `require(ESM)` cannot load synchronously below Node 24.9.'
  evidence: Reproduced directly (`npx cross-env NODE_OPTIONS=--experimental-vm-modules jest` on Node v22.23.2, matching `.nvmrc`). Pre-existing since Story 1.18 introduced `AuthModule`/`@nestjs/jwt` — already broke `auth.service.spec.ts`, `jwt.strategy.spec.ts`, `prisma.module.spec.ts`, and `src/__tests__/app-startup.spec.ts` before this story. Combined with the already-tracked CI gap above (CI only runs `depcruise`, never `npm test`), this means the one test written specifically to catch a botched C1 `AccessResolver` DI rewiring never actually executes anywhere today — locally or in CI.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: '`test/support/access-matrix.ts`/`graph-factory.ts` (the TEA e2e test-oracle harness merged to `main` the same day as this story, commit `bb9531c`) still model a single combined `managerLine` audience unioning reports-to and project PM/DM lines — not yet updated to the ratified `ReportingLine`/`ProjectLine` split this story''s contract now uses (`ARCHITECTURE-SPINE.md` AD-2, 2026-08-26).'
  evidence: Pre-existing in a file this story does not touch; the harness itself documents (via a `TODO(domain-schema)` comment) that it isn''t wired to real Prisma persistence yet — this story''s `Employee.managerId` is the schema piece that TODO was waiting on. Whoever wires `graph-factory.ts` to persistence next should split its role model to match.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: '`test/support/access-matrix.spec.ts` fails `tsc --noEmit` (`TS2345`, a type-narrowing issue in `test/support/access-matrix.ts`), unrelated to this story.'
  evidence: Surfaced incidentally while reviewing this story''s diff; no file under `test/support/` was touched here. Pre-existing in the same just-merged TEA e2e harness commit (`bb9531c`).

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: No DB-level `CHECK` constraint prevents `Employee.managerId = Employee.id` (a self-referencing manager) — only the app-level cycle-safe walk in `AccessResolverService` defends against it at read time.
  evidence: Flagged by the edge-case reviewer as defense-in-depth. Natural to add alongside the write-time cycle guard (D15, `access-model.md`) once a real write path for `managerId` exists — no such path exists yet in this story's scope (managerId is only set via migration/seed/manual DB ops today).

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: ARCHITECTURE-SPINE.md's AD-2 (C1 row) and AD-18 both specify that `AccessResolver` caps every section to `R` once the subject's employment status is `dismissed` — this story's "Never" boundary list doesn't call that out as an explicitly deferred concern the way it does for AD-14/Story-1.2/Story-1.8.
  evidence: Understandable gap — the `employment` module (C11 `EmploymentStatusService`) doesn't exist anywhere in the codebase yet, so there's nothing for `AccessResolverService` to consult. But it's an undocumented gap against a ratified invariant rather than an explicitly scoped-out one; whoever builds the `employment` module (CAP-14) needs to circle back and wire this cap into C1.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: Empty/falsy `viewerId`/`subjectId` inputs to `resolveAudience` silently resolve to `Colleague` (safe deny-by-default) rather than surfacing as a likely upstream caller bug (e.g. auth/session not resolving to a real `Employee.id`).
  evidence: Not required by the spec's I/O matrix or "Never" list — the unit tests already cover the both-empty case per the story's own task list, and the current behavior is safe, not incorrect. Flagged as a possible future hardening (throw/log on empty ids) rather than a defect; worth revisiting once a real caller (Story 1.6+) exists to observe whether this ever actually happens in practice.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-1-derive-manager-access-from-reporting-hierarchy.md`
  summary: No automated test exercises the real `onDelete: SetNull` FK behavior on `Employee.managerId` — every `AccessResolverService` test mocks `PrismaService.employee.findUnique`, and the only e2e delete path (`test/users.e2e-spec.ts`) deletes a user with no `directReports`.
  evidence: The spec's own Verification section already scopes this to a manual check ("delete a manager's Employee row locally... confirm dependents' managerId is set to null"), so this is an accepted, spec-sanctioned gap, not a violation. Flagged because an e2e test (in the style of `test/users.e2e-spec.ts`, using `testApp.prisma` directly) would be cheap and would catch a future accidental flip to `Restrict`/`Cascade` (the pattern already used by sibling history-table FKs in the same schema file) that nothing today would catch.

## Deferred from: spec-1-2-extend-manager-access-via-project-assignment planning (2026-08-31, token-budget split)

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: Bootcamp seed data (`seed.synthetic.ts`) doesn't include any demo `ProjectAssignment` rows, so `ProjectLine` access has no seeded example to click through after `db:seed`.
  evidence: Neither of this story's acceptance criteria require seeded demo data — both are proven by unit tests against the real C3 implementation. Carved off at the spec's token-budget checkpoint (drafted spec was 2810 tokens against the 900-1600 target) to keep the story to one cohesive goal: resolver logic + persistence, not persistence + demo-data curation. A follow-up can add 2-3 confirmed `ProjectAssignment` rows among the 24 bootcamp employees once this story ships.

## Deferred from: code review of spec-1-2-extend-manager-access-via-project-assignment (2026-08-31)

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: No `@@unique`/exclusion constraint on `ProjectAssignment` stops two confirmed, active rows for the same `(employeeId, projectId)` naming different PM/DM, and no validation rejects `pmId`/`dmId` equal to `employeeId` (self-managed project) or an `endDate` earlier than `startDate`.
  evidence: `resolveProjectLine`'s "grant if any surviving row matches" logic silently tolerates ambiguous/conflicting or nonsensical rows rather than rejecting them at write time. Not required by either acceptance criterion — all three are data-integrity hardening beyond this story's scope, flagged by the edge-case and blind-hunter review layers.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: `ProjectAssignmentService.create()` accepts `confirmed: true` with no `confirmedAt` (defaults to `null`), silently producing a row that can never resolve active — `isProjectAssignmentActive` always requires a non-null `confirmedAt`.
  evidence: Fails safe (denies access rather than granting it), so not a security defect, but a footgun for any future caller of the internal-write path. Flagged by the edge-case-hunter review layer; not required by either acceptance criterion.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: `AccessResolverService.resolveProjectLine`'s `listByEmployee(subjectId)` call fetches every historical `ProjectAssignment` row for the subject (no `confirmed`/date filtering pushed into the Prisma `where`), filtering only in application code; the reports-to walks per row (`isInReportingLine` for `pmId`/`dmId`) are also not de-duplicated across rows sharing the same PM/DM, nor does the loop short-circuit once both `granted` and `dmMatched` are already `true`.
  evidence: Every `resolveAudience` call re-fetches and re-walks the full assignment history with no caching or filtering — a real scalability gap once the table grows, mirroring the identical, already-deferred `FieldRegistry.query()` pagination gap from spec-1-19's review. Not a correctness bug; flagged by the blind-hunter review layer as a forward-looking performance concern.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: `resolveAudience`'s `ReportingLine` chain walk and `resolveProjectLine`'s `ProjectAssignment` reads are not wrapped in a single read transaction, so a concurrent write between them can (rarely) yield an inconsistent role for one call.
  evidence: Extends the identical accepted risk spec-1-1's Design Notes already documented for the `ReportingLine` walk alone ("acceptable for Wave-0 scope but worth flagging if... access checks become hot-path") to the new `ProjectAssignment` read path. Flagged by the edge-case-hunter review layer.

- source_spec: `_bmad-output/implementation-artifacts/spec-1-2-extend-manager-access-via-project-assignment.md`
  summary: No automated test exercises the real `onDelete: Restrict` FK behavior on `ProjectAssignment.employeeId`/`pmId`/`dmId` — every `ProjectAssignmentService` test mocks `PrismaService`.
  evidence: Mirrors the identical, already-deferred gap for `Employee.managerId`'s `onDelete: SetNull` from spec-1-1's review — the spec's own Verification section only covers this via manual migration inspection, not an automated FK-behavior test. Flagged by the blind-hunter review layer.

## Deferred from: code review of spec-3-2-define-custom-fields-at-runtime (2026-08-31)

- N+1 access resolution in `listDefinitions` / `listValuesForEmployee` loops — each definition triggers separate `resolveAudience` / `hasPermission` calls. Performance concern for large field sets; not blocking for Wave-1 bootcamp scope.

- `manage_custom_fields` permission not granted in bootcamp seed data — expected while C8 `PermissionChecker` remains deny-by-default stub until Track A functional-role stories land.

- Colleague-visible custom fields require S16 read for Colleague role — blocked until Story 1.8 (colleague whitelist) lands. Production `AccessResolverService` assigns Colleague S16 `none`; colleague-tier field reads return false until C1 colleague access is implemented.

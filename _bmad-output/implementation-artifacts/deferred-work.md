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

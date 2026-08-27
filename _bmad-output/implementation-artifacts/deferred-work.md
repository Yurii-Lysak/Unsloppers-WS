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

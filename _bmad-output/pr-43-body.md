## What this adds

- `build` job (`nest build`) and a full `e2e` job (all `test/*.e2e-spec.ts`) added to `.github/workflows/ci.yml`, alongside the existing `module-boundaries`, `unit-tests`, `lint`, `access-control-e2e` jobs.
- `concurrency` group so a new push cancels an in-flight run.
- `timeout-minutes` on every job.
- `lint` now runs `npm run lint:check` (new, non-autofixing script) instead of the autofixing `lint` script, so it actually gates instead of silently fixing and passing.
- Three test-only fixes, root-caused and verified (details in the companion WS PR's `known-red-diagnosis.md`):
  - schema-scoped the `pg_indexes` query in `temporal-history.extension.spec.ts` (was matching leftover schemas from unrelated e2e runs)
  - `jest-e2e.global-teardown.ts` now drops test schemas in a `finally`, so a thrown coverage assertion can't skip cleanup
  - `campaigns.e2e-spec.ts` fixture used `employmentStatus: 'inactive'`, not a valid enum value — corrected to `'dismissed'`

## Verification (local, Windows/Node 22/Postgres 18 in Docker, against this branch)

- Unit: **707/707**
- E2E: **325/335** (10 failures across 6 suites — down from 11/7 before the three fixes above)
- **Re-analysis (2026-09-07):** after merging `main` (includes [#45](https://github.com/Yurii-Lysak/Unsloppers-BE/pull/45)), `npx nest build` passes — TS2322 resolved.

## Not fixed here — needs an owner decision

These are pre-existing on `main`, not introduced by this branch, and the new `build`/`e2e` jobs are what surfaced them. Full write-up with file/line references: [`known-red-diagnosis.md`](https://github.com/Yurii-Lysak/Unsloppers-WS/blob/chore/testarch-ci-pipeline/_bmad-output/test-artifacts/known-red-diagnosis.md) in the companion workspace PR.

1. ~~**Build-blocking:** `TS2322` in `campaigns.service.ts:146`~~ — **Fixed on `main` via [#45](https://github.com/Yurii-Lysak/Unsloppers-BE/pull/45)** (merged 2026-09-07). `audienceFilters` cast to `Prisma.InputJsonValue`; `nest build` passes after merging `main` into this branch.
2. **3 × 403** on `custom-fields`/`users` routes — stale test fixtures, or the Story 1.8 employee-record hardening should exempt these routes? [details](https://github.com/Yurii-Lysak/Unsloppers-WS/blob/chore/testarch-ci-pipeline/_bmad-output/test-artifacts/known-red-diagnosis.md#L156)
3. **Real API defect:** `GET /api/v1/employees?filters=...` 400s with `Unknown field "undefined"` — DTO transform loses nested filter properties. [details](https://github.com/Yurii-Lysak/Unsloppers-WS/blob/chore/testarch-ci-pipeline/_bmad-output/test-artifacts/known-red-diagnosis.md#L176)
4. **Possible access-control gap:** `GET /api/v1/employees` has no per-row audience narrowing — every builtin field is visible to any authenticated viewer regardless of role, Colleague included. [details](https://github.com/Yurii-Lysak/Unsloppers-WS/blob/chore/testarch-ci-pipeline/_bmad-output/test-artifacts/known-red-diagnosis.md#L210)
5. **3 × 401**, not root-caused — a shared session cookie stops being accepted partway through `employee-profile.e2e-spec.ts`. [details](https://github.com/Yurii-Lysak/Unsloppers-WS/blob/chore/testarch-ci-pipeline/_bmad-output/test-artifacts/known-red-diagnosis.md#L252)

## Not merging anything

Opened for visibility/review only — no required status checks are configured, this doesn't change deploy behavior on its own, and the CI/required-checks decision is still open per the earlier team discussion.

**Next step:** push the `main` merge to `chore/testarch-ci-pipeline` to re-trigger CI — `build` should go green; `e2e`/`lint` may still show pre-existing failures.

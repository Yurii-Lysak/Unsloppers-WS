---
stepsCompleted:
  [
    'step-01-preflight',
    'step-02-generate-pipeline',
    'step-03-configure-quality-gates',
    'step-04-validate-and-summary',
  ]
lastStep: 'step-04-validate-and-summary'
lastSaved: '2026-09-04'
---

# CI/CD Pipeline Progress

Second run of `bmad-testarch-ci`. The first run (2026-08-31, see git history of
this file) scaffolded both pipelines on a branch that then sat 35-44 commits
behind `origin/main` in all three repos for four days. Rather than rebase a
stale diff onto four days of new features, `chore/testarch-ci-pipeline` was
fast-forwarded to `origin/main` in `Unsloppers-WS`, `Unsloppers-BE`, and
`Unsloppers-FE` (no branch carried commits of its own — the whole prior run was
an uncommitted working-tree diff, stashed rather than lost), and this run
re-applies the pipeline against the current code instead of restoring the old
YAML.

## Step 1: Preflight Checks

### Git Repository

All three repos have `.git/` and an `origin` remote. `chore/testarch-ci-pipeline`
fast-forwarded clean in each (branch had zero unique commits, so this was a
true fast-forward, not a rebase):

- `Unsloppers-WS`: `f9d9003` → `037e2a6` (44 commits)
- `Unsloppers-BE`: `2c30df2` → `77cfaca` (35 commits)
- `Unsloppers-FE`: `4fb973b` → `b4f454c` (18 commits)

### Stack Detection

Unchanged from the first run: `services/backend` is `backend`
(`prisma/schema.prisma`, `docker-compose.yml`, Jest); `services/frontend` is
`frontend` (`playwright.config.ts`, `vite.config.ts`).

### Test Framework

- Backend: Jest 30 / ts-jest, two tiers (`npm test` unit, `npm run test:e2e`
  e2e with per-worker Postgres schemas — see `test/README.md`).
- Frontend: Playwright 1.61, now 14 spec files / 71 tests (was 4 files / 24
  tests in the first run — the suite grew with the merged features).

### Local Test Run

Run against a live Postgres 18 container (`npm run db:up`, then
`npm run db:deploy` to apply the 19 migrations that had landed since the first
run) and, for the frontend, `npx playwright install --with-deps chromium`
(browsers were not present in this environment).

**Backend — build fails.** `npx nest build`:

```
src/modules/campaigns/campaigns.service.ts:146:9 - error TS2322:
Type 'FieldFilter[]' is not assignable to type 'JsonNullClass | InputJsonValue | undefined'.
```

Pre-existing on `origin/main` at `77cfaca` ("feat(campaigns): build draft
campaign audience", Story 10.2) — unrelated to this CI work. Reported to the
author; not fixed here (out of scope, not this run's code). The generated
`build` job below will be red on first run until this lands.

**Backend — unit tests: 704/707 passed.** 3 failures, all in
`src/prisma/extensions/temporal-history.extension.ts` (via
`temporal-history.extension.spec.ts` and
`timeline-event-writer.integration.spec.ts`): a real-Postgres transaction
deadlock (`Transaction failed due to a write conflict or a deadlock`) that
exceeds the 5s test timeout, and a follow-on `RangeError: Invalid string
length`. Looks like transaction-isolation contention in the extension itself,
not an environment artifact — reported alongside the build failure.

**Backend — e2e: 324/335 passed.** 11 failures across 7 suites, none of which
the pre-existing `access-control-e2e` job (see Step 2) would ever have run,
because it deliberately narrows to
`access-matrix|matrix-flag|shared-links|cross-feature-access`:

| Suite                                    | Failure                                                        |
| ----------------------------------------- | --------------------------------------------------------------- |
| `employees.e2e-spec.ts`                   | 2× filter query returns `400` instead of `200`                  |
| `campaigns.e2e-spec.ts`                   | `PrismaClientValidationError` in `employee.update()` via `relationship-graph.extension.ts` |
| `colleague-whitelist.e2e-spec.ts`         | matcher error — received `undefined`, expected a number         |
| `employee-profile-custom-fields.e2e-spec.ts` | **S16 data leak**: a colleague-visibility fixture leaks `"Shirt size": "L"` to a Colleague viewer |
| `employee-profile.e2e-spec.ts`            | 3× `401` instead of `200` on manager/PP/C8-role reassignment reads |
| `custom-fields.e2e-spec.ts`               | 2× `403` instead of `200`/`404`                                 |
| `auth.e2e-spec.ts`                        | `403` instead of `200` restoring a session from a cookie         |

The S16 leak is the one worth flagging loudest: it is exactly the class of bug
the Story 1.15 access-control test suite exists to catch, and it is not caught
by the narrow `access-control-e2e` job. Reported alongside the build failure.

**Frontend — lint, typecheck, all 71 e2e tests: green.** No findings.

Per user decision, all of the above are treated as **known-red gates** to
document and wire in, not blockers on generating the pipeline — the pipeline's
job is to surface exactly this class of regression on every PR going forward,
and it could not have caught these until it existed.

### CI Platform Detection

`ci_platform: github-actions`. `services/backend/.github/workflows/ci.yml`
already existed (`module-boundaries`, `unit-tests`, `lint`,
`access-control-e2e` — all real, all kept verbatim). `services/frontend` had no
CI. Decision: **extend** the backend file, **create** the frontend one.

### Environment Context

- Backend: `.nvmrc` pins Node 22 (already present).
- Frontend: no `.nvmrc` — added this run (see Step 2).

### TEA Config Flags

- `tea_use_playwright_utils` and `tea_use_pactjs_utils` were still `true` in
  `_bmad/tea/config.yaml` from before the first run (that run's flip to
  `false` was part of the stashed, uncommitted diff and never landed). Neither
  `@seontechnologies/playwright-utils` nor `@seontechnologies/pactjs-utils` is
  a dependency in either service — confirmed by grep, not assumed. Both flags
  set back to `false` with rationale recorded in the config file, so burn-in
  uses Playwright's native `--only-changed` instead of `runBurnIn`, and no
  contract-testing stage was generated.

## Step 2: Generate CI Pipeline

### Backend — `services/backend/.github/workflows/ci.yml` (extended)

| Job                   | Status     | What it does                                                                          |
| ---------------------- | ---------- | --------------------------------------------------------------------------------------- |
| `module-boundaries`   | kept as-is | `depcruise` — AD-1 module dependency boundaries                                        |
| `build`               | **added**  | `npx nest build` — compile gate. Not `npm run build`: its `postbuild` hook runs `prisma migrate deploy` against a live DB, so the compiler is invoked directly |
| `unit-tests`          | kept as-is | `npm test` against a Postgres 18 service container                                      |
| `lint`                | **changed**| now runs `npm run lint:check` (new script, no `--fix`) instead of `npm run lint`, which carries `--fix` and always exits 0 — it could never have gated anything |
| `access-control-e2e`  | kept as-is | narrow e2e subset (access-matrix/matrix-flag/shared-links/cross-feature-access), `--runInBand` |
| `e2e`                 | **added**  | `npm run test:e2e` — the full `test/*.e2e-spec.ts` tier, per-worker Postgres schemas. This is the job that would have caught the campaigns/employees/custom-fields/auth/S16 regressions found in Step 1 |

Also added: top-level `concurrency` group (cancels superseded runs on
force-push), `timeout-minutes` on every job (10 for lint/build/module-
boundaries, 15 for unit-tests/access-control-e2e, 20 for the new full `e2e`
job — sized off the local run times: unit ~80s, full e2e ~5.5min against a
fresh container).

`package.json` gained one script: `"lint:check": "eslint \"{src,apps,libs,test}/**/*.ts\""`.

### Frontend — `services/frontend/.github/workflows/test.yml` (new)

| Job                | Trigger                  | What it does                                                                 |
| ------------------- | ------------------------- | ------------------------------------------------------------------------------- |
| `lint-typecheck`   | PR, push to `main`        | `npm run lint`, `npm run typecheck`                                            |
| `e2e`              | PR, push to `main`        | 2-shard Playwright matrix (`--shard=N/2`), Chromium install, junit + HTML report artifacts |
| `e2e-count-check`  | PR, push to `main`        | sums `tests="N"` across both shards' junit files and compares against `npx playwright test --list --reporter=json`'s discovered count — fails if a shard silently ran fewer tests than exist, per the "runner manifest that names a subset is a coverage hole" rule |
| `burn-in`          | PR: changed specs only; weekly cron: full suite | 10 iterations. PR uses Playwright's native `--only-changed=origin/<base>` (no `playwright-utils` dependency, per the config decision above); cron runs the whole 71-test suite since it's the only trigger that would ever exercise untouched files |

Also added: `services/frontend/.nvmrc` pinning Node 22, to match the backend
and stop CI and local dev from silently drifting apart on Node version.

No contract-testing stage in either pipeline (`tea_use_pactjs_utils: false`,
no contract tests exist in either service).

## Step 3: Quality Gates & Notifications

- **Burn-in**: frontend/fullstack stack → enabled (see above). Backend is
  backend-only → skipped by default, per the stack-conditional rule (backend
  suites are deterministic; UI flakiness is what burn-in targets). No override
  requested.
- **Quality gates**: every job that runs tests can fail the build (no
  `continue-on-error` on a test-running step, only on artifact upload steps).
  P0/P1 pass-rate thresholds are not separately enforced in CI config — no
  risk-tagging convention exists in either test suite to key off yet.
- **Contract testing gate**: not applicable — `tea_use_pactjs_utils` is
  `false`.
- **Notifications**: none wired, unchanged from the first run's team decision
  (see below) — inventing a Slack webhook secret name would leave a
  permanently failing step. Failure signal remains the PR checks and the job
  summary.

## Step 4: Validate & Summary

### Checklist highlights

- [x] CI file at platform-correct path in both repos
- [x] YAML validated (parsed with `js-yaml`, both files) — no syntax errors
- [x] Stack-conditional steps: browser install present in the frontend `e2e`
  job only; absent from every backend job
- [x] Sharding: frontend 2-way matrix; backend intentionally single-runner per
  job (its e2e tier already isolates via per-worker Postgres schemas, not CI
  shards)
- [x] Burn-in: frontend enabled (PR + weekly), backend skipped (documented
  reason)
- [x] Artifacts: junit + HTML report always/on-failure as appropriate;
  retention 30 days
- [x] No secrets required in either pipeline
- [x] No `${{ inputs.* }}` or unsafe GitHub context interpolated directly into
  any `run:` block

### Team decisions (carried over from the first run, still current)

- **No CI/CD platform work.** Branch protection, required status checks, and
  failure notifications are still explicitly out of scope per the team's
  standing decision: part is considered unnecessary at this stage, and the
  deployment half lives on Vercel/Render/Neon (per `docs/deployment.md`, landed
  since the first run — see Story 1.17 closing out AD-12). Both workflows are
  quality gates only: they report, they do not block merges, and nothing needs
  configuring in GitHub settings.

### Known-red gates on first run (all pre-existing, all reported — see Step 1)

1. Backend `build` job — TS2322 in `campaigns.service.ts` (77cfaca)
2. Backend `unit-tests` job — 4 failures in `temporal-history.extension.spec.ts`
3. Backend `e2e` job (new) — 11 failures across 7 suites

None of these are caused by the pipeline itself — they are pre-existing on
`origin/main` and would have shipped silently without the `build` and `e2e`
jobs added this run.

**Follow-up investigation: `known-red-diagnosis.md`.** Two of the labels above
were wrong on this first pass and are corrected there — item 2 is not a
transaction deadlock (it is a leftover `tea_test_w1` schema plus a `pg_indexes`
query that does not filter by schema; the suite passes 65/65 once the schema is
dropped, and CI's separate job containers make it green there), and the S16
finding inside item 3 is not a data leak (the access control is correct; a raw
`not.toContain('L')` assertion matches the unrelated `manageLeaveUrl` key). Of
the 11 e2e failures, one is an unambiguous API defect — `filters` on
`GET /api/v1/employees` 400s with `Unknown field "undefined"` — three need an
owner decision, and the rest are test-suite debt.

### Completion Summary

- **Platform:** GitHub Actions, two repositories.
- **Paths:** `services/backend/.github/workflows/ci.yml` (extended, +2 jobs,
  1 job's command fixed), `services/frontend/.github/workflows/test.yml` (new,
  4 jobs).
- **Backend gates:** module boundaries, build, unit + Postgres, lint (now
  actually gating), access-control e2e (narrow), e2e (full, new).
- **Frontend gates:** lint + typecheck, 2-shard Playwright, coverage
  reconciliation, burn-in (PR: changed-only; weekly: full).
- **Required secrets:** none.
- **Also changed:** `lint:check` script in backend `package.json`; `.nvmrc` in
  frontend; both TEA utility flags flipped to `false` in
  `_bmad/tea/config.yaml` with rationale (packages not installed in either
  service).

**Next steps:**

1. Get the three known-red items above fixed or triaged by their owners; they
   are not blockers for merging this CI work, but they are real bugs the new
   `build` and `e2e` jobs will report on every PR until then.
2. Review and commit each service repository, then update the workspace
   gitlinks — staging and committing are explicit, separate asks per this
   workspace's policy.
3. Push branches and open PRs to see first runs; verify cache hits, shard
   distribution, and the `e2e-count-check` reconciliation actually catches a
   deliberately-broken shard (not verified live — only unit-tested locally
   against the real junit/list-JSON shapes).
4. Revisit only if the team's position on CI/CD changes: required status
   checks, failure notifications, and whether the frontend integration suite
   (`e2e/integration/**`, still `testIgnore`d) should get its own CI leg.

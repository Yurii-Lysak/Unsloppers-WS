# Test Framework Setup — Progress

**Date:** 2026-08-25
**Workflow:** `bmad-testarch-framework`, create mode
**Surfaces:** both — `services/backend` (Jest + Supertest) and `services/frontend` (Playwright)
**Branch:** `chore/tea-test-framework` in both submodules

## Framing

Both services already had a working framework, so this run extended them rather
than scaffolding new ones, as `test-design/people-management-handoff.md`
anticipated. The work was driven by the testability blockers that test design
raised, not by a generic template.

Two choices set the shape of everything below:

1. **Vanilla stack.** `_bmad/tea/config.yaml` sets `tea_use_playwright_utils`
   and `tea_use_pactjs_utils` to `true`, but neither package is installed. The
   decision was to stay on the installed stack, so the fixtures, interception
   helpers and factories are written here rather than imported. No dependency
   was added to either service.
2. **No contract testing.** Per `pactjs-utils-mandate.md`, a frontend calling
   its own backend is not a consumer/provider boundary — both live in this
   workspace and start from its own scripts. The genuinely external providers
   (timetracker, PeopleForce) are not ours to run in a provider verification.
   Pact scaffolding was therefore skipped, and the external boundaries are
   covered by a stubbing harness instead.

## Steps

### Step 1 — Preflight

Detected a fullstack workspace with two submodules. The skill's preflight halts
when an E2E framework already exists; that halt was overridden deliberately,
because extending the existing setup is the stated intent. Both submodules were
on `main`, so a `chore/tea-test-framework` branch was created in each before any
edit, per workspace policy.

### Step 2 — Backend: isolation, time, boundaries

- **Per-worker Postgres schemas.** `test/jest-e2e.global-setup.ts` drops and
  recreates `tea_test_w1..N`, one per worker Jest will spawn, and applies
  migrations to each. Provisioning goes through the Prisma CLI rather than the
  generated client, because `globalSetup` runs outside the module registry that
  resolves it. `test/jest-e2e.global-teardown.ts` drops them again.
- **Harness.** `test/support/app-harness.ts` boots the app with `PrismaService`
  overridden by a client pinned to the worker's schema through the adapter's
  `schema` option, and re-applies the global `ValidationPipe` that the test app
  does not inherit from `main.ts`. All three existing e2e specs were migrated
  onto it.
- **Injectable clock.** `src/clock/` adds a `Clock` abstraction and a global
  `ClockModule`; `test/support/fixed-clock.ts` is the test double.
- **External boundaries.** `test/support/external-boundary.ts` is a controllable
  HTTP server on loopback with `respond`, `malformed`, `hang` and `reset`
  behaviours plus `goOffline`/`comeBackOnline`. Zero new dependencies, and the
  client under test exercises its real HTTP path including timeouts.

### Step 3 — Backend: access matrix and graphs

- `test/support/access-matrix.ts` transcribes the S1–S16 × five-audience matrix
  from `specs/spec-people-management-platform/access-model.md`. The `Record`
  types make a missing section or audience a compile error;
  `assertMatrixCoverage()` makes an untested pair a test failure.
- `test/support/access-matrix.spec.ts` enforces two spec rules over the data
  itself: the colleague view is a whitelist (rule 3), and no shared link is ever
  writable (rule 7). The documented exceptions — the PM carve-out on S7, the
  mentor field on S1 — are attached to their cells and asserted present.
- `test/support/graph-factory.ts` builds the reporting / project / people-partner
  graph and computes the expected audience from the spec rules, independently of
  any resolver. It doubles as the oracle for matrix cases.

### Step 4 — Frontend: fixtures, selectors, config

- `e2e/shared/merged-fixtures.ts` is now the single import point for `test`,
  merging three small fixture modules: `api-fixture` (direct `APIRequestContext`
  for arranging state), `network-fixture` (`interceptNetworkCall` /
  `stubNetworkCall`, both declared before the triggering action), and
  `clock-fixture` (`freezeAt` / `advance` / `jumpTo`).
- `e2e/shared/selectors.ts` holds the test id catalogue plus three helpers that
  keep *absent*, *unavailable* and *rendered* distinguishable by section key
  rather than by rendered copy.
- `e2e/shared/factories.ts` ships deterministic factory primitives. Domain
  factories were deliberately not written: `src/types/api.ts` holds only
  `ApiError`, and a factory on a guessed shape is a fixture that gets rewritten
  the day the real shape lands.
- `playwright.config.ts` hardened: explicit timeouts, `retain-on-failure` traces
  and video, list/html/junit reporters, `failOnFlakyTests` in CI, and
  `workers: '50%'` replacing `workers: 1`, which had been cancelling out
  `fullyParallel`.
- `e2e/harness.spec.ts` is a self-test for the fixtures, so a silent breakage
  surfaces here rather than as a confusing failure in a feature spec.

### Step 5 — Documentation

`services/backend/test/README.md`, `services/frontend/e2e/README.md` and
`e2e/shared/README.md` were written or rewritten. `.env.example` in both services
gained the new variables. `services/backend/.claude/rules/nest-e2e.md` was
updated, because its previous guidance — prefix every value with `Date.now()`,
hand-delete rows in `afterAll` — is now actively wrong.

## Testability concerns from test design

| ID   | Concern                                  | Status |
| ---- | ---------------------------------------- | ------ |
| TC-1 | Machine-readable access matrix           | Done — plus a harness that fails on unmapped pairs |
| TC-2 | Parallel workers against shared Postgres | Done — one schema per worker, verified |
| TC-3 | Relationship-graph factory               | Partial — in-memory graph and oracle done; persistence blocked on the domain schema |
| TC-4 | Time-dependent requirements              | Done — injectable clock on both sides |
| TC-5 | External HTTP stubbing                   | Done — `ExternalBoundary`, no new dependency |
| TC-6 | API vs UI assertion split                | Decided — see below |
| TC-7 | Flaky tests masked by retries            | Done — `failOnFlakyTests` in CI |

### TC-6 decision

Response-shape assertions belong to the backend Supertest tier, which owns a real
database and a real application instance. Playwright starts only Vite, so the API
it can reach is whatever the developer happens to have running; its `api` fixture
is for arranging state, not for asserting on it. Every "this field is absent for
this audience" assertion is a backend assertion. The Playwright suite owns what
the browser renders.

## Deviations

- **Preflight halt overridden** — deliberate, see Step 1.
- **`test/support/*.spec.ts` run in the unit tier.** The unit Jest config moved
  from `rootDir: src` to `rootDir: .` with `roots: [src, test]`, so the harness
  has its own fast tests. `collectCoverageFrom` was narrowed to `src/**` to keep
  coverage measuring the application rather than the harness.
- **One production change.** `src/clock/` plus its registration in
  `app.module.ts`. Everything else is test-only. Without an injectable clock the
  time-dependent requirements cannot be tested without sleeping, so this is the
  minimum production surface that makes them testable.
- **`react-hooks/rules-of-hooks` disabled for `e2e/**`.** Playwright fixtures
  receive a callback named `use`, which the React rule mistakes for the `use()`
  hook. Scoped to the e2e folder, where there is no React.
- **Naming.** Frontend fixtures live in `e2e/shared/` to match the existing
  convention, not in a new `support/` folder.

## Deliberately skipped

- **Pact / contract testing** — the relevance gate is not met, see Framing.
- **`@seontechnologies/*` utils packages** — vanilla stack chosen.
- **Write-time enforcement hook.** The skill specifies a `PreToolUse` hook to
  block `describe.only`, hard waits and similar at write time. Cursor has no
  equivalent, so this was skipped rather than faked. `forbidOnly` in CI covers
  `.only`, and `bmad-testarch-test-review` remains the enforcement path.
- **Accessibility scanning.** WCAG 2.1 AA on List, Profile and Dashboards needs
  an axe integration that is not installed; adding it is a dependency decision,
  so it is a named follow-up rather than a silent install.

## Verification

All commands run on Node 22.23.2 against the docker Postgres.

| Command                             | Result |
| ----------------------------------- | ------ |
| backend `npm test`                  | 8 suites, 112 tests passed |
| backend `npm run test:e2e`          | 3 suites, 17 tests passed (parallel workers) |
| backend `npm run test:e2e:serial`   | 3 suites, 17 tests passed |
| backend `npm run lint`              | clean |
| backend `npm run build`             | clean |
| frontend `npm run test`             | 10 tests passed |
| frontend `npm run typecheck`        | clean |
| frontend `npm run lint`             | clean |
| frontend `npm run format:check`     | clean |

Three real defects were found and fixed during verification rather than
documented as passing:

1. `page.clock.install()` alone does not freeze time — it sets the starting
   instant and then lets time run, which drifted the assertion by the duration of
   the test. `freezeAt` pairs it with `pauseAt`.
2. `globalSetup` cannot import the generated Prisma client, because
   `moduleNameMapper` does not apply there. Provisioning moved to the Prisma CLI.
3. Jest's default 5 s timeout is too short to boot the Nest app; the e2e config
   now sets `testTimeout: 30000`.

## Environment notes

- **Node 22 is required and was missing.** The shell had Node 20.16.0 and no
  version manager. Prisma 7 refuses to install below 20.19/22.12, and Vite 8
  silently skips its native rolldown binding on an unsupported Node, which then
  fails at dev-server start. Node 22.23.2 was unpacked to
  `%LOCALAPPDATA%\node22-tea` and used via `PATH` for this session only — no
  installer, no change to the system Node, reversible by deleting that folder.
  Note that `services/backend/AGENTS.md` tells contributors to run `nvm use`;
  no `nvm` is present on this machine.
- **Frontend dependencies were reinstalled** under Node 22 to pick up the
  rolldown binding that the first install had skipped.
- **A local `.env` was created** in `services/backend` from the committed
  `.env.example`, because docker-compose requires it. It is gitignored and holds
  only the local defaults already published in the template.

## Follow-ups

1. **Graph persistence (TC-3).** Fill the `TODO(domain-schema)` in
   `graph-factory.ts` once `prisma/schema.prisma` has employees, projects and
   assignments.
2. **Access matrix cases.** The harness is in place but no resolver exists yet.
   When section access lands, drive the cases from `matrixCells()` and call
   `assertMatrixCoverage()` so an unmapped pair fails the build.
3. **Accessibility tooling** — decide on an axe integration.
4. **CI wiring.** Done — `bmad-testarch-ci` ran on 2026-09-04 and both suites are
   hooked to GitHub Actions: `services/backend/.github/workflows/ci.yml`
   (module boundaries, build, unit, lint, access-control e2e, full e2e) and
   `services/frontend/.github/workflows/test.yml` (lint/typecheck, 2-shard
   Playwright, coverage reconciliation, burn-in). `CI` is set by the runner, so
   `failOnFlakyTests` and the junit reporter are active there. See
   `ci-pipeline-progress.md` — the backend `build` and `e2e` jobs are red on
   first run against three pre-existing, reported bugs on `origin/main`.
5. **Full-stack Playwright runs.** Starting the backend and Postgres from
   `webServer` would let the frontend suite cover flows end to end.

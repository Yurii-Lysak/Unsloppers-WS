---
stepsCompleted:
  [
    'step-01-load-context',
    'step-02-define-thresholds',
    'step-03-gather-evidence',
    'step-04-evaluate-and-score',
    'step-04a-subagent-security',
    'step-04b-subagent-performance',
    'step-04c-subagent-reliability',
    'step-04d-subagent-maintainability',
    'step-04e-aggregate-nfr',
    'step-05-generate-report',
  ]
lastStep: 'step-05-generate-report'
lastSaved: '2026-09-05'
workflowType: 'testarch-nfr-assess'
inputDocuments:
  - _bmad/tea/config.yaml
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/adr-quality-readiness-checklist.md
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/ci-burn-in.md
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/test-quality.md
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/playwright-config.md
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/error-handling.md
  - .claude/skills/bmad-testarch-nfr/resources/knowledge/nfr-criteria.md
  - _bmad-output/test-artifacts/test-design-qa.md
  - _bmad-output/test-artifacts/test-design-architecture.md
  - _bmad-output/test-artifacts/known-red-diagnosis.md
  - _bmad-output/test-artifacts/ci-pipeline-progress.md
  - _bmad-output/test-artifacts/test-review.md
  - _bmad-output/test-artifacts/framework-setup-progress.md
  - docs/deployment.md
  - docs/project-requirements.md
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - services/backend/src/bootstrap.ts
  - services/backend/src/modules/auth/**
  - services/backend/src/modules/directory/employees.service.ts
  - services/backend/src/modules/directory/field-registry.service.ts
  - services/backend/src/modules/access/shared-link.service.ts
  - services/backend/.github/workflows/ci.yml
  - services/frontend/.github/workflows/test.yml
  - services/backend/package.json (+ live `npm audit`)
  - services/frontend/package.json (+ live `npm audit`)
  - services/backend/prisma/schema.prisma
  - services/backend/.env.example
---

# NFR Evidence Audit — People Management Platform (Backend + Frontend + CI + Deployment)

**Date:** 2026-09-05
**Story:** Cross-cutting (whole implemented system to date, not a single story)
**Overall Status:** FAIL ❌ (one confirmed live authorization exposure, plus a confirmed build failure currently blocking the backend's auto-deploy)

---

Note: This audit summarizes existing implementation evidence — code, config, tests, prior investigation
docs — read as of 2026-09-05. It does not run tests, spin up a live environment, or probe a running
instance (`tea_browser_automation` was set to `auto`, but per the run's own scope there is no live
deployed instance to point a browser at; all "Browser-Based Evidence Collection" sections in Steps 1/3
were skipped by design, not by oversight). NFR thresholds and planned evidence were taken primarily from
`test-design-qa.md` / `test-design-architecture.md` (Step 0/2 primary source), per the workflow's own
rule to prefer an existing test-design NFR plan over re-deriving thresholds from raw docs.

## Executive Summary

**Assessment:** 2 PASS, 9 CONCERNS, 1 FAIL, 8 N/A (across the 20 template rows below; domain-level detail
and the full ADR-8, 29-criteria table follow)

**Blockers:** 2 — (1) `GET /api/v1/employees` returns every builtin field (grade, position, department,
tenure) to any authenticated viewer regardless of relationship, with zero per-row audience narrowing
(confirmed directly in `employees.service.ts`, matches `known-red-diagnosis.md` §3e). This is a live
breach of SM-1 ("zero unauthorized data exposure"), the product's stated top success metric. (2) A
`TS2322` type error in `campaigns.service.ts` currently fails `nest build`, which per `docs/deployment.md`
is Render's exact build command — the backend's last push to `main` should have failed to deploy, and
production is very likely serving a stale build.

**High Priority Issues:** 6 — no rate limiting on `/auth/login` or shared-link consumption (R-016, still
open); a real API defect on `GET /api/v1/employees?filters=...` (400 "Unknown field \"undefined\""); the
employee-list query loads the *entire* employee table into memory before filtering/paginating (no DB-level
pagination); zero load-testing evidence for the one defined performance budget (NFR-2, 500+ records / 2s);
no CI gate for dependency vulnerabilities or test coverage despite both being measurable; the AD-10
generated-type diff gate that `test-design-qa.md` describes as "Ready" was not found wired into either
CI workflow actually on disk.

**Recommendation:** **FAIL — do not treat as release-ready.** The authorization gap and the build failure
are each independently disqualifying; neither requires new architecture to fix (per
`known-red-diagnosis.md`, both have a clear, scoped fix), but both are currently true in the code today,
not hypothetical.

---

## Defect Tracking

Each finding below is also tracked as an individual defect file in
`services/backend/defects/`, so a fix can reference the exact evidence
directly from the repo:

| Finding | Domain | Defect file |
| --- | --- | --- |
| `GET /api/v1/employees` authorization gap | Security (Blocker) | [`bugs/02-employees-list-authorization-gap.md`](../../services/backend/defects/bugs/02-employees-list-authorization-gap.md) |
| `TS2322` build failure | Reliability/Deployability (Blocker) | [`bugs/01-build-fails-ts2322-campaigns-service.md`](../../services/backend/defects/bugs/01-build-fails-ts2322-campaigns-service.md) |
| No rate limiting on `/auth/login`/shared-link (R-016) | Security | [`bugs/04-no-rate-limiting-auth-shared-link.md`](../../services/backend/defects/bugs/04-no-rate-limiting-auth-shared-link.md) |
| `queryEmployees` full-table load, no DB pagination | Performance | [`bugs/05-employees-query-full-table-load.md`](../../services/backend/defects/bugs/05-employees-query-full-table-load.md) |
| Employee list `filters` 400 (DTO defect) | Reliability/Security | [`bugs/03-employees-filters-400-dto-defect.md`](../../services/backend/defects/bugs/03-employees-filters-400-dto-defect.md) |
| `npm audit` not gated in CI | Security/Maintainability | [`ci-hardening/01-npm-audit-not-gated.md`](../../services/backend/defects/ci-hardening/01-npm-audit-not-gated.md) |
| Coverage not gated in CI | Maintainability | [`ci-hardening/02-coverage-not-gated.md`](../../services/backend/defects/ci-hardening/02-coverage-not-gated.md) |
| Unstructured logging, no error tracking | Maintainability | [`ci-hardening/03-unstructured-logging-no-error-tracking.md`](../../services/backend/defects/ci-hardening/03-unstructured-logging-no-error-tracking.md) |
| AD-10 type-diff gate not wired | Maintainability | [`ci-hardening/04-ad10-type-diff-gate-not-wired.md`](../../services/backend/defects/ci-hardening/04-ad10-type-diff-gate-not-wired.md) |
| 3× 401 session invalidation, not root-caused | Reliability | [`test-debt/03-investigation-401-employee-profile-session.md`](../../services/backend/defects/test-debt/03-investigation-401-employee-profile-session.md) |

---

## Domain Risk Breakdown

| Domain | Risk Level | Headline evidence |
| --- | --- | --- |
| **Security** | **HIGH** | Confirmed live unauthorized-exposure gap on the employee list endpoint; zero rate limiting anywhere in the API |
| **Performance** | **MEDIUM** | Zero profiling evidence for the one defined budget; the list query's implementation (full-table load, in-memory filter/paginate) is itself a scalability risk at 500+ records |
| **Reliability** | **HIGH** | Build currently fails to compile (blocks deploy); ~10 e2e failures were last recorded on `main`; the fault-injection/graceful-degradation infrastructure underneath is genuinely well built |
| **Maintainability** | **MEDIUM** | No CI gate for coverage, duplication, or dependency vulnerabilities; strong module boundaries, strict TypeScript, and an independently reviewed 81/100 test suite |

**Overall Risk Level: HIGH** (per the aggregation rule: any domain at HIGH forces overall HIGH).

---

## Performance Assessment

**Threshold source (Step 2):** `test-design-qa.md`'s NFR Test Coverage Plan — the only NFR-2/SM-4 target
that exists is **"All Employees list under 2 seconds at 500+ records with 3 active filters, permission
resolution included"** (P0-014, k6, blocked on Q-1 — the concurrency assumption). No other endpoint has a
latency target (test-design's own "Not in Scope" table: *"Load testing beyond the All Employees list... no
other target exists (Q-2)"*). This is a **known, defined** threshold for the one endpoint that has one —
not an UNKNOWN category — so the CONCERNS below come from **missing evidence against a real threshold**,
not from the UNKNOWN-threshold downgrade rule.

### Response Time (p95)

- **Status:** CONCERNS ⚠️
- **Threshold:** <2s at 500+ employee records, 3 active filters (one derived, one custom), permission
  resolution included (NFR-2/SM-4, `test-design-qa.md` P0-014)
- **Actual:** Not measured. No k6 script, load-test artifact, or profiling output exists anywhere in the
  repository (confirmed by repo-wide search — the only hits for "k6" are inside BMad skill/knowledge files
  and the test-design docs describing the *plan*, not an implementation).
- **Evidence:** Absence confirmed via `Grep` across the workspace; `test-design-qa.md`'s own Tooling table
  lists k6 as "Pending."
- **Findings:** The threshold is real and known, but per `nfr-status-definitions.md` an unmeasured target
  is never PASS. Additionally, `field-registry.service.ts`'s `queryEmployees` (the method backing this
  exact endpoint) loads the **entire** employee table via `prisma.employee.findMany()` — with four
  joined history tables per row — into memory, then applies `applyFilters`/`sortSnapshots` and finally
  `snapshots.slice(offset, offset + pageSize)` for pagination (`field-registry.service.ts:637-553`). There
  is no `WHERE` clause narrowing what's fetched and no DB-level `LIMIT`/`OFFSET` — filtering, sorting, and
  paging all happen in Node after the full table is materialized. At 500 records with 4 history joins each
  this may still complete comfortably inside 2s, but the implementation itself does not scale sub-linearly,
  and nothing in the repo demonstrates it holds the line — this is exactly the class of risk R-004
  ("resolution cost breaches the 2s budget", score 6) names.

### Throughput

- **Status:** CONCERNS ⚠️
- **Threshold:** UNKNOWN — `test-design-architecture.md` names the concurrency assumption for NFR-2 as
  Q-1, explicitly unanswered ("Unknown — no resolution strategy chosen... Concurrency assumption
  undefined").
- **Actual:** Not measured.
- **Evidence:** `test-design-architecture.md`'s NFR Testability Requirements table, row "Performance."
- **Findings:** Per the Default Rule for Undefined Thresholds, this is CONCERNS by construction — no
  guessed value was substituted.

### Resource Usage

- **CPU Usage**
  - **Status:** N/A
  - **Threshold:** Not defined anywhere in the reviewed docs.
  - **Actual:** No APM, no resource-usage monitoring of any kind exists (confirmed — no Datadog/New
    Relic/equivalent dependency in either `package.json`, no custom metrics code found).
  - **Evidence:** Absence confirmed by dependency and source search.
- **Memory Usage**
  - **Status:** N/A
  - **Threshold:** Not defined.
  - **Actual:** Not monitored. Same evidence as CPU Usage. Worth flagging alongside the full-table-load
    finding above: an unbounded `findMany()` with no `take`/`skip` is the shape most likely to show up
    first as a memory or latency regression as the dataset grows past the 500-record target, and nothing
    would currently catch that in CI.

---

## Security Assessment

**Threshold source:** `test-design-qa.md` NFR-1/SM-1 — "Zero unauthorized exposure; every denial holds on
every surface" — plus the ADR-8 Security category (AuthN/AuthZ, Encryption, Secrets, Input Validation).

### Authentication Strength

- **Status:** PASS ✅
- **Threshold:** Standard protocol, no plaintext credentials, resistant to timing-based user enumeration
  (ADR-8 §5.1)
- **Actual:** JWT issued via `@nestjs/jwt`, delivered as an `httpOnly` cookie (`auth-cookie.ts`), verified
  server-side on every request through a global `APP_GUARD` (`JwtAuthGuard`, wired in `auth.module.ts` —
  confirmed there is no route that bypasses this guard by omission, only `@Public()`-decorated ones, e.g.
  health). Passwords hashed with `bcryptjs` at cost 12 (`password.service.ts`). `AuthService.login`
  deliberately calls `verifyUnknownUser`/compares against a pre-computed dummy hash when the user does not
  exist or the input exceeds bcrypt's input-length limit, specifically to keep response timing
  indistinguishable between "no such user" and "wrong password" (`auth.service.ts:31-38`) — a real,
  non-obvious mitigation, not a template claim.
- **Evidence:** `services/backend/src/modules/auth/auth.service.ts`, `password.service.ts`,
  `jwt.strategy.ts`, `auth.module.ts`.
- **Findings:** Cross-site cookie handling is also correctly adapted for the real deployment topology —
  `sameSite: isProduction ? 'none' : 'strict'` paired with `secure: isProduction`
  (`auth-cookie.ts:23-32`), which `docs/deployment.md` §4 explains is required because Vercel (`*.vercel.app`)
  and Render (`*.onrender.com`) are genuinely cross-site, not just cross-port. This is deliberate, tested
  engineering, not a default.

### Authorization Controls

- **Status:** FAIL ❌
- **Threshold:** Zero unauthorized exposure — every one of the 96 section×audience cells in the access
  matrix must hold on **every** emission surface, not only the profile API (R-001, score 9, `test-design-
  architecture.md`).
- **Actual:** `GET /api/v1/employees` (`EmployeesService.listEmployees`) does **not** hold. Read directly
  in `employees.service.ts`:
  - `filterVisibleFields` (lines 169-200) only ever filters **custom** fields by visibility; every
    builtin field (`name`, `grade`, `position`, `department`, `employment_type`, `years_with_company`) is
    pushed into `visible` unconditionally for every viewer (line 182: `if (field.source !== 'custom') {
    visible.push(field); continue; }`).
  - `maskRowCells` (lines 202-240) only deletes cells for **custom** fields (`customFields.filter(...
    source === 'custom')`); it never touches a builtin cell.
  - Nothing in `listEmployees` calls `SectionAccessGate`/`AccessResolver` per row, unlike
    `updateEmployeeField`, which does gate `S16` writes and does check `audience.sections.S4` for builtin
    writes a few methods below.
  - Net effect: any authenticated caller with an `Employee` record — Colleague included — receives full
    builtin-field data (grade, position, department, tenure) for **every** employee in the directory, with
    no per-row narrowing by relationship at all.
- **Evidence:** `services/backend/src/modules/directory/employees.service.ts:169-240` (read directly for
  this audit); independently corroborated by `known-red-diagnosis.md` §3e, which found the same gap via a
  failing e2e assertion (`colleague-whitelist.e2e-spec.ts:241`, `returns S1-safe directory list entries`)
    and reached the same root cause by reading the same two methods. This is a live breach today, in the
    code as currently written, not a hypothetical from a template.
- **Findings:** This is precisely the risk category the product's own risk register scores highest
  (R-001, score 9) and precisely the metric the PRD names as the top success metric (SM-1: zero
  unauthorized exposure incidents). `known-red-diagnosis.md` explicitly recommends routing this to whoever
  owns the access matrix rather than silently patching the failing test to match current behavior — this
  audit concurs: the test (`colleague-whitelist.e2e-spec.ts`) is correct, and the product code is not.
- **Recommendation:** Route `listEmployees`'s builtin-field visibility and per-row masking through the
  same `AccessResolver`/`SectionAccessGate` path the profile endpoint already uses, mirroring the pattern
  `updateEmployeeField` follows for writes. Until fixed, this is a release blocker.

### Data Protection

- **Status:** CONCERNS ⚠️
- **Threshold:** Encryption in transit and at rest; no real PII outside a designated non-production tier
  (NFR-5, ADR-8 §5.2)
- **Actual:** In transit: TLS is provided by the managed platforms (Vercel, Render, Neon) rather than
  configured in application code — `docs/deployment.md` documents `sslmode=require` on the Neon connection
  string, and both Vercel and Render serve HTTPS by default; no code-level enforcement (e.g., HSTS header)
  was found, so this relies entirely on platform defaults rather than being asserted anywhere. At rest:
  Neon is a managed Postgres provider that encrypts storage by default per its own platform guarantees,
  but nothing in this repo configures, documents, or verifies that — it is inherited, not evidenced here.
  PII: the bootcamp seed is pseudonymized (per the workspace's own `AGENTS.md` pivot note and NFR-5), but
  `docs/deployment.md` §3 discloses a real, named gap: **local development's `DATABASE_URL` currently
  points at the same Neon database Render serves in production** — "there is no separate non-production
  database yet." The doc itself judges this acceptable only because the seeded data is already
  pseudonymized, not because the environments are actually separated.
- **Evidence:** `docs/deployment.md` §3 ("Neon connection string" subsection), `bootstrap.ts` (no HSTS/
  security-header middleware present).
- **Findings:** No violation of NFR-5 as currently scoped (no real PII exists anywhere), but the
  environment-separation gap is a named, self-reported risk in the deployment doc, not a discovery — it
  should be tracked, not just acknowledged.

### Vulnerability Management

- **Status:** CONCERNS ⚠️
- **Threshold:** 0 critical, minimal high vulnerabilities in production dependencies; a CI gate to catch
  regressions (ADR-8 §5, `nfr-criteria.md` Maintainability axis)
- **Actual:** Live `npm audit --json` run against both services' installed `node_modules` (2026-09-05):
  - **Backend:** 8 total (0 critical, **7 high**, 1 moderate). All are transitive — `js-yaml` (via
    `@nestjs/swagger`), `deepmerge-ts`/`mysql2` (via `prisma`'s own tooling, not the runtime Prisma
    client path this app uses), `fast-uri`, `qs`. None are in a code path this app's runtime HTTP handling
    actually exercises; all have `fixAvailable` (several via a Prisma major-version bump).
  - **Frontend:** 13 total (0 critical, **10 high**, 3 moderate). Mostly build-tooling/dev-time
    (`postcss`, `browserslist`, `js-yaml`, `nanoid`, `brace-expansion` — all transitive dev dependencies),
    plus one **direct** high: `react-router-dom` → `react-router` CSRF-adjacent advisory (GHSA-qwww-vcr4-
    c8h2, "RSC Mode CSRF Bypass," fix available, non-major).
  - **Neither CI workflow runs `npm audit`** — confirmed absent from both
    `services/backend/.github/workflows/ci.yml` and `services/frontend/.github/workflows/test.yml` (read
    in full for this audit). Nothing would flag a new critical vulnerability landing on `main` today.
- **Evidence:** Live `npm audit --json` output captured 2026-09-05 (not a template number); both CI YAML
  files read directly.
- **Findings:** No critical-severity findings today, but the "0 critical, minimal high" bar is met only by
  chance, not by a gate — an unmonitored dependency tree is not evidence it will stay this way.

### Compliance

- **Status:** N/A
- **Standards:** None. The PRD, architecture docs, and `docs/project-requirements.md` do not name SOC2,
  GDPR, HIPAA, PCI-DSS, or any other compliance framework as in scope for this project, and nothing in the
  codebase (no consent flows, no data-subject-request tooling, no PCI-scoped payment handling) suggests
  one was adopted informally either.
- **Actual:** Not applicable — this is a genuine N/A, not an unassessed gap. No compliance framework was
  ever adopted, so none is being audited against.
- **Evidence:** Absence confirmed across all planning-artifact and requirements documents reviewed for
  this audit.

**Other Security-relevant findings (not template rows, but material):**

- **No rate limiting anywhere.** Grep for `throttl`/`rate-limit`/`ThrottlerModule` across
  `services/backend/src` returns nothing; `auth.module.ts` registers exactly one `APP_GUARD`
  (`JwtAuthGuard`), not a throttler. R-016 ("Shared-link and login endpoints have no rate limiting,
  permitting token guessing or credential stuffing," score 4) is confirmed still open in the running
  code, not just the risk register.
- **Input validation is otherwise solid.** Global `ValidationPipe({ whitelist: true, transform: true })`
  (`bootstrap.ts:21`) plus Prisma's parametrized queries throughout — a repo-wide grep for
  `$queryRawUnsafe`/`$executeRawUnsafe` returns zero matches; the few `$queryRaw`/`$executeRaw` calls that
  do exist are tagged-template-literal (auto-parametrized) and confined to one test-only cleanup helper
  (`temporal-history.extension.spec.ts`), not production code. No SQL-injection surface was found.
- **XSS:** the frontend is React (auto-escaping by default) with zero `dangerouslySetInnerHTML` usages
  found anywhere in `services/frontend/src`.
- **The one dynamic-filter/injection-shaped surface (R-012) has a live, confirmed defect**, not a security
  breach but a correctness gap in exactly the area meant to be hardened: `GET /api/v1/employees?filters=...`
  400s with `"Unknown field \"undefined\""` because nested filter objects lose their properties during DTO
  transformation under the global `whitelist: true` pipe (`known-red-diagnosis.md` §3d, root-caused to
  `list-employees-query.dto.ts:83-105`). Not exploitable, but it means the allow-listing this risk depends
  on has not been exercised successfully end-to-end.

---

## Reliability Assessment

**Threshold source:** `test-design-qa.md` NFR-4 — "External sources degrade without taking the app down"
— plus CI-observed stability (`ci-pipeline-progress.md`, `known-red-diagnosis.md`).

### Availability (Uptime)

- **Status:** N/A
- **Threshold:** No SLA defined anywhere (`test-design-architecture.md` names this an open gap: "SLA
  Definitions... undefined").
- **Actual:** Pre-launch/demo-scale system; no production traffic history exists to measure against.
- **Evidence:** `docs/deployment.md` (single production environment, no staging tier).

### Error Rate

- **Status:** N/A
- **Threshold:** Not defined.
- **Actual:** No error-rate monitoring exists (no Sentry/Datadog — confirmed absent by dependency search).

### MTTR (Mean Time To Recovery)

- **Status:** N/A
- **Threshold:** Not defined; no incident history exists to derive one from.

### Fault Tolerance

- **Status:** PASS ✅ (scoped to external-integration resilience — see CI Burn-In below for the deploy-
  pipeline finding, which is separate and worse)
- **Threshold:** External sources (TimeTracker, PeopleForce) degrade the relevant feature without taking
  the app down (NFR-4)
- **Actual:** A genuine substitutable HTTP boundary exists (`test/support/external-boundary.ts` — a
  controllable loopback server with `respond`/`malformed`/`hang`/`reset`/`delayMs`/`goOffline`/
  `comeBackOnline`), and `ProjectsSyncService` uses it: on sync failure the service logs a warning and
  returns `{ status: 'failed' }` rather than throwing or crashing (`projects-sync.service.ts:125-128`),
  and its own spec (`projects-sync.service.spec.ts`, confirmed present) exercises the client boundary for
  timeout/failure paths per `test-review.md`'s independently-verified best-practice citation ("real
  substitutability... asserts on `AbortSignal`-bounded timeouts and typed `TimetrackerApiError` mapping").
- **Evidence:** `services/backend/src/modules/integrations/projects-sync.service.ts`,
  `test/support/external-boundary.ts`, `test-review.md` Best Practice #2-equivalent citation.
- **Findings:** This is real engineering, not a template claim — the fault-injection seam was purpose-
  built (per `framework-setup-progress.md`, TC-5) specifically because there was previously no way to test
  this path at all.

### CI Burn-In (Stability)

- **Status:** FAIL ❌ (mechanism vs. current state — both reported, see below)
- **Threshold:** A green, enforced CI pipeline is the mechanism NFR verification depends on (R-003, score
  9 — "no CI pipeline, so every CI-enforced invariant is advisory").
- **Actual:** CI now exists (`ci-pipeline-progress.md`, 2026-09-04 run) — `module-boundaries`, `build`,
  `unit-tests`, `lint`, `access-control-e2e`, and a new full `e2e` job on the backend; `lint-typecheck`,
  sharded `e2e`, `e2e-count-check`, and `burn-in` (PR: `--only-changed`, weekly: full suite, 10 iterations
  each) on the frontend. The mechanism itself is well designed — e.g. `e2e-count-check` specifically
  guards against a shard silently running fewer tests than exist.
  **But the last recorded state of `main` is red**: the backend `build` job fails to compile (`TS2322` in
  `campaigns.service.ts` — see Recommended Actions), and the backend `e2e` job had 10 failures across 6
  suites at last count (`known-red-diagnosis.md`'s fixed-up total, down from 11/7 after two test-only
  fixes). The narrow `access-control-e2e` job (which *is* presumably green, since it predates this run)
  would not have caught most of these — it deliberately scopes to
  `access-matrix|matrix-flag|shared-links|cross-feature-access`, not the full suite.
- **Evidence:** `ci-pipeline-progress.md` Step 1 (local reproduction) and Step 4 (known-red gates);
  `known-red-diagnosis.md` (root-caused, fixed-up totals); both CI YAML files read directly for this audit.
- **Findings:** A CI pipeline that would catch these regressions going forward now exists — that is a
  genuine improvement — but as of this audit the pipeline itself reports the backend `build` job red,
  which is the most severe possible CI signal (nothing downstream of a failed build can be trusted).

### Disaster Recovery

- **RTO (Recovery Time Objective)**
  - **Status:** CONCERNS ⚠️ (recorded here per the template; primarily an ADR-8 category, not scored by
    the automated Reliability worker — see the ADR-8 table below)
  - **Threshold:** UNKNOWN — never defined (`test-design-architecture.md` ADR-8 gap 4.1).
  - **Actual:** No RTO exists. Explicitly named as an accepted trade-off at this project's scale (single
    environment, no blue/green — `test-design-architecture.md` "Accepted Trade-offs").
- **RPO (Recovery Point Objective)**
  - **Status:** CONCERNS ⚠️
  - **Threshold:** UNKNOWN — same as above.
  - **Actual:** Neon (managed Postgres) provides its own backup/PITR capability per its platform, but
    nothing in this repo configures, documents, or has ever tested a restore. Untested backups are, by the
    standard framing, equivalent to no backups for planning purposes.

**Other Reliability-relevant findings (not template rows, but material):**

- **A real, currently-open API defect** on `GET /api/v1/employees?filters=...` (400 "Unknown field
  \"undefined\"") — see Security §Vulnerability/Input Validation above; cited here too because it is a
  functional reliability defect, not only a hardening gap.
- **Three unresolved 401s** in `employee-profile.e2e-spec.ts` (last three tests in the file) — a shared
  session cookie stops being accepted partway through the suite; `known-red-diagnosis.md` could not root-
  cause this and flags it as needing someone who knows the intended session-invalidation semantics. Left
  open here as an unresolved reliability question, not fabricated as fixed.
- **The health check endpoint is real and tested**: `GET /api/v1/health` via `@nestjs/terminus` +
  `PrismaHealthIndicator`, `@Public()`-decorated so it doesn't require auth (appropriate for a platform
  health probe), covered by `health.controller.spec.ts`. This is a genuine PASS-level finding, just not a
  template row on its own.

---

## Maintainability Assessment

**Threshold source:** `nfr-criteria.md` (coverage ≥80%, duplication <5%, 0 critical/high vulnerabilities,
structured logging + error tracking) and `test-design-qa.md`'s own exit criteria ("Coverage ≥80% backend
overall, ≥90% branch on `access`").

### Test Coverage

- **Status:** CONCERNS ⚠️
- **Threshold:** ≥80% backend overall, ≥90% branch coverage on the `access` module (`test-design-qa.md`
  Exit Criteria — a real, defined target, not UNKNOWN)
- **Actual:** Not measured in CI. `services/backend/package.json` defines a `test:cov` script
  (`jest --coverage`), but the CI `unit-tests` job runs plain `npm test`, not `test:cov` — confirmed by
  reading `ci.yml` directly. No `coverageThreshold` key exists in the Jest config block of `package.json`
  either, so even a local `test:cov` run would report a number without gating on it.
- **Evidence:** `services/backend/package.json` (`jest` config block, `scripts`), `ci.yml` (no `--coverage`
  flag anywhere).
- **Findings:** The underlying suite is qualitatively strong — `test-review.md` (2026-09-05, independent
  run) scored it 81/100 across 82 files, zero Critical findings, and specifically praised the access-matrix
  coverage-collector pattern (`assertMatrixCoverage`) for making an *untested matrix cell* fail the build
  by name. That is real evidence of thorough testing, but it is not the same as a measured line/branch
  percentage against the stated 80%/90% targets, and per the default rule an unmeasured target cannot be
  marked PASS.

### Code Duplication

- **Status:** CONCERNS ⚠️
- **Threshold:** <5% (`nfr-criteria.md` convention; no project-specific override found)
- **Actual:** No `jscpd` (or equivalent) configuration or dependency exists anywhere in either service —
  confirmed by repo-wide search, matches zero hits outside BMad's own skill/template files. No percentage
  exists to compare against a threshold. Manual review (`test-review.md`, M2 finding) did find one real,
  localized instance: `employees.e2e-spec.ts` inlines the same 6-call Prisma fixture sequence at least four
  times instead of using the shared `createEmployeeUser` factory ten other files already use.
- **Evidence:** Repo-wide search for `jscpd`; `test-review.md` Recommendation 4.
- **Findings:** The one confirmed instance is scoped and low-risk (identical, correct code, not divergent
  copies), not evidence of systemic duplication — but there is no tooling to know whether it's the only
  one.

### Vulnerability Scan

- **Status:** CONCERNS ⚠️
- **Threshold:** 0 critical, minimal high (shared with the Security domain's Vulnerability Management row
  — see that section for the full `npm audit` breakdown: backend 8 total/7 high/0 critical, frontend 13
  total/10 high/0 critical, neither gated in CI).
- **Actual/Evidence:** See Security §Vulnerability Management above (same underlying evidence; reported
  once, cross-referenced per both domains' template rows, matching the shared-finding convention
  `test-review.md` itself uses for its own H3 row).

### Observability

- **Status:** CONCERNS ⚠️
- **Threshold:** Structured logging + error tracking configured (`nfr-criteria.md`)
- **Actual:** Logging exists but is not structured: several backend services (`ProjectsSyncService`,
  `LeavesSyncService`, `TimelineEventWriterService`, and others) instantiate NestJS's built-in `Logger`
  class and call `.log()`/`.warn()`/`.debug()` with plain interpolated strings (e.g.
  `projects-sync.service.ts:22,34,108,125`) — this is real logging, not `console.log`, but it is
  human-readable text, not JSON, with no correlation/trace ID, and no documented schema. No Pino, Winston,
  or equivalent structured-logging library is a dependency of either service (confirmed by dependency
  search). No error-tracking service (Sentry, Datadog, Rollbar, or equivalent) is configured or a
  dependency anywhere — also confirmed absent by search.
- **Evidence:** Grep for `Logger`/`console\.` usage across `services/backend/src`; dependency search
  across both `package.json` files for `sentry|winston|pino|datadog|newrelic` (zero hits outside this
  audit's own search-artifact list).
- **Findings:** `test-design-architecture.md` names "No observability stack. Structured logs suffice;
  tracing is not warranted" as an *accepted trade-off* at this project's scale — that framing is
  reasonable for a demo-scale MVP, but "structured logs suffice" is not currently true in the literal
  sense: the logs that exist are plain text, not structured. This is a documented-intent-vs-implementation
  gap worth naming precisely rather than accepting the trade-off's wording at face value.

**Other Maintainability-relevant findings (not template rows, but material):**

- **Module boundaries are genuinely enforced.** `dependency-cruiser` runs as its own CI job
  (`module-boundaries`) against `.dependency-cruiser.cjs` — confirmed present and wired, not just
  configured.
- **Lint now actually gates.** `ci-pipeline-progress.md` documents (and `ci.yml` confirms) that the CI
  `lint` job was changed to run `lint:check` (no `--fix`) instead of the old `lint` script, which carried
  `--fix` and always exited 0 — a real, dated fix to a previously-cosmetic gate.
- **The AD-10 generated-type diff gate was not found wired into CI.** `test-design-qa.md`'s Tooling table
  marks `openapi-typescript` diff as "Already in the frontend toolchain... Ready," and P1-012 names it as
  a CI-enforced invariant. Reading both `ci.yml` and `test.yml` in full for this audit found no diff/openapi
  step in either — `lint-typecheck` on the frontend runs `eslint`/`tsc`, not a schema-diff check, and no
  such step exists on the backend either. This may simply not have been wired yet despite the tooling being
  present; recorded here as a gap between the plan and the CI file actually on disk, not asserted as
  definitely broken (the diff tooling itself was not independently re-verified beyond reading the two
  workflow files).
- **TypeScript strict mode is on for both services**, and `test-review.md`'s independent review found zero
  disabled/focused tests and zero hard waits across 82 backend test files — both genuine maintainability
  positives.
- **`action-items.e2e-spec.ts` is ~1,633 lines**, over the 1000-line ceiling `test-quality.md` sets
  (`test-review.md` H5) — a real, if narrow, maintainability cost.

---

## Custom NFR Evidence Audits

No `custom_nfr_categories` were supplied in `_bmad/tea/config.yaml` (confirmed empty). No custom NFR audit
performed; this section is intentionally empty rather than populated with invented categories.

---

## Other ADR-8 Categories (Recorded, Not Scored by an Automated Worker)

Per `step-02-define-thresholds.md`: Security, Performance, Reliability, and Maintainability get an
automated Step-4 worker (above). The remaining four ADR-8 categories are recorded here from whatever
evidence exists — not independently PASS/CONCERNS/FAIL-scored the way the four domains above are, per the
workflow's own scope boundary, but given a directional read where the evidence supports one.

### Testability & Automation — evidence strong

Per-worker Postgres schema isolation (`test/jest-e2e.global-setup.ts`), a substitutable external-HTTP
boundary for TimeTracker/PeopleForce (`test/support/external-boundary.ts`), an injectable clock
(`src/clock/`), a relationship-graph test factory (`test/support/graph-factory.ts`), and 100% of business
logic reachable via REST + Swagger docs. This is the one ADR-8 category with the deepest evidence trail in
this codebase — largely because `test-design-architecture.md` named these as foundation-phase blockers
(TC-1 through TC-5) and `framework-setup-progress.md` documents each being resolved.

### Test Data Strategy — evidence good, one confirmed gap

Faker-based factories with auto-cleanup fixtures used by ~72 of 82 reviewed test files
(`test-review.md`); pseudonymized bootcamp seed data (no real PII, NFR-5). Gap: `employees.e2e-spec.ts`
bypasses the shared factory (see Maintainability §Code Duplication), and — more materially — local
development points at the *same* Neon database as production (`docs/deployment.md` §3), so segregation
between "test data" and "real environment" is weaker than the matrix/CI-level isolation suggests.

### Scalability & Availability — evidence weak

Stateless auth (JWT, no server-side session store) is a genuine positive. Everything else is a named gap:
no load testing exists (bottlenecks, including the one this audit found in `queryEmployees`, are
unidentified by measurement), no availability SLA is defined, and circuit breakers exist only informally
(graceful failure returns in sync services, not a formal breaker pattern).

### Disaster Recovery — explicitly out of scope by design, not silently missing

No RTO/RPO, no failover rehearsal, backups unverified (Neon-managed, untested by this team). Distinct from
a silent gap: `test-design-architecture.md`'s "Accepted Trade-offs" section names this explicitly ("No
staging tier," "No blue/green or rollback rehearsal... a single containerized deploy is the stated shape")
as proportionate for a three-developer, demo-scale MVP. Worth revisiting only if the product moves past
that scale, per the same document.

### Monitorability, Debuggability & Manageability — evidence weak

Config is properly externalized (`ConfigService` + Joi validation, `env.validation.ts` — no hardcoded
config found). Everything else is absent: no distributed tracing, no dynamic log-level toggle, no
`/metrics` endpoint (only `/health`, which is a liveness/readiness check, not a RED-metrics endpoint).

### QoS & QoE — evidence weak

One latency target exists (the list endpoint's 2s/500-record budget), unmeasured (see Performance above).
No rate limiting/throttling exists anywhere (see Security above — this is the same underlying gap,
relevant to both domains). Frontend perceived-performance patterns (skeleton screens, optimistic updates)
and degradation UX (friendly error messages vs. raw stack traces) were **not independently reviewed** in
this audit — reviewing `services/frontend/src` component-level UX patterns was out of scope for the
evidence gathered here; recorded as **Not Assessed**, not scored either way.

### Deployability — evidence poor, one active blocker

Auto-deploy on push to `main` exists for both services (Vercel, Render — `docs/deployment.md` §1), which
is a real capability. But: no zero-downtime strategy (single instance, accepted trade-off), no automated
rollback, and — the active blocker — `prisma migrate deploy` runs automatically in the backend's `postbuild`
hook on every push, against the **same** shared Neon database local dev also points at, with **no staging
tier to rehearse a migration first** and a build that currently fails to compile at all. `known-red-
diagnosis.md` names this exact combination as the risky part of fixing the `TS2322` error: unblocking the
build also unblocks the next automatic migration run against production.

---

## Findings Summary

**Based on ADR Quality Readiness Checklist (8 categories, 29 criteria)**

| Category | Criteria Met | Status | Basis |
| --- | --- | --- | --- |
| 1. Testability & Automation | 4/4 | ✅ PASS | Isolation, headless API, seeding, sample requests (Swagger) all evidenced — see "Other ADR-8 Categories" |
| 2. Test Data Strategy | 2/3 | ⚠️ CONCERNS | Generation + teardown evidenced; segregation weakened by shared local/prod DB |
| 3. Scalability & Availability | 1/4 | ❌ CONCERNS | Only statelessness evidenced; no load test, no SLA, no formal circuit breaker |
| 4. Disaster Recovery | 0/3 | ⚠️ CONCERNS (accepted trade-off) | RTO/RPO/backup-restore all undefined, but explicitly documented as out of scope at this stage |
| 5. Security | 1/4 | ❌ FAIL | Secrets management evidenced; AuthN/AuthZ, encryption, and input-validation each carry a real, confirmed gap |
| 6. Monitorability, Debuggability & Manageability | 1/4 | ❌ CONCERNS | Config externalization evidenced; tracing, log levels, metrics all absent |
| 7. QoS & QoE | 0/4 (2 Not Assessed) | ❌ CONCERNS | Latency target defined but unmeasured; no throttling; UX-perception rows not assessed (frontend out of this audit's scope) |
| 8. Deployability | 0/3 | ❌ FAIL | Auto-deploy exists but is currently broken by a real build failure; no rollback; no DB/code decoupling |
| **Total** | **9/29 (31%)** | **❌ Significant gaps** | Per the scoring rubric: <20/29 (<69%) = Significant gaps |

**Criteria Met Scoring (from the checklist):**

- ≥26/29 (90%+) = Strong foundation
- 20-25/29 (69-86%) = Room for improvement
- **<20/29 (<69%) = Significant gaps** ← this system, at 9/29 (31%)

This is a harsher number than the qualitative engineering evidence elsewhere in this report might suggest
in isolation — the *test infrastructure* (clock injection, external-boundary fault injection, per-worker
DB isolation, self-verifying matrix coverage) is genuinely well built, and several ADR-8 gaps (DR, no
staging tier) are explicit, documented, proportionate trade-offs for a three-developer demo-scale MVP, not
oversights. But the checklist scores what exists today, not intent, and two categories (Security,
Deployability) carry a **confirmed, live** defect each, which is what pulls the total down to "Significant
gaps" rather than "Room for improvement."

---

## Gate YAML Snippet

```yaml
nfr_assessment:
  date: '2026-09-05'
  story_id: 'cross-cutting'
  feature_name: 'People Management Platform (backend + frontend + CI + deployment)'
  adr_checklist_score: '9/29' # ADR Quality Readiness Checklist
  categories:
    testability_automation: 'PASS'
    test_data_strategy: 'CONCERNS'
    scalability_availability: 'CONCERNS'
    disaster_recovery: 'CONCERNS' # accepted trade-off, documented
    security: 'FAIL'
    monitorability: 'CONCERNS'
    qos_qoe: 'CONCERNS'
    deployability: 'FAIL'
  domain_risk:
    security: 'HIGH'
    performance: 'MEDIUM'
    reliability: 'HIGH'
    maintainability: 'MEDIUM'
  overall_status: 'FAIL'
  overall_risk: 'HIGH'
  critical_issues: 2
  high_priority_issues: 6
  medium_priority_issues: 5
  concerns: 9
  blockers: true
  quick_wins: 2
  evidence_gaps: 6
  compliance: 'N/A — no compliance framework adopted for this project'
  recommendations:
    - 'Fix the GET /api/v1/employees authorization gap (per-row audience narrowing) before any release — SM-1 is violated today'
    - 'Fix the TS2322 build failure in campaigns.service.ts — it currently blocks the Render auto-deploy'
    - 'Add rate limiting to /auth/login and shared-link consumption (R-016)'
    - 'Build the k6 harness and capture real evidence for the NFR-2 (2s/500-record) budget before claiming it holds'
    - 'Wire npm audit and a coverage-threshold check into CI — both are measurable today and neither is gated'
```

---

## Cross-Domain Risks

1. **Security × Reliability (CRITICAL):** The live authorization gap on `GET /api/v1/employees` and the
   currently-broken build are both true at once. Even once the access gap is understood and a fix is
   written, that fix cannot ship until the unrelated `TS2322` build failure is resolved, because Render's
   build command (`npm install --include=dev && npm run build`) will reject any push, fix included, until
   `campaigns.service.ts` compiles. The two defects are unrelated in cause but compound in effect.

2. **Security × Maintainability (HIGH):** The one CI job scoped to catch access-control regressions
   (`access-control-e2e`) deliberately narrows to `access-matrix|matrix-flag|shared-links|cross-feature-
   access` — it would not have caught the `GET /api/v1/employees` gap, because that endpoint's suite
   (`colleague-whitelist.e2e-spec.ts`) apparently isn't in that filtered set, or wasn't failing loudly
   enough to be triaged before this audit. Meanwhile the broader `e2e` job that *would* run that spec is
   itself red for several unrelated, already-triaged reasons (`known-red-diagnosis.md`), which makes it
   easy for a genuine new regression to blend into a list of "known-red" noise instead of standing out.

3. **Performance × Maintainability (MEDIUM):** Neither the load-testing gap (Performance) nor the
   coverage/duplication/audit gaps (Maintainability) are enforced anywhere in CI. Both categories currently
   depend entirely on someone remembering to check manually — the same structural weakness (no automated
   gate) shows up independently in two different domains.

4. **Reliability × Deployability (HIGH):** `postbuild` runs `prisma migrate deploy` against the shared
   production Neon database automatically on every successful push to `main`, with no staging tier to
   rehearse a migration first. Combined with the currently-broken build, the *next* successful push (i.e.,
   whichever push finally fixes the `TS2322` error) will be the one that runs migrations — meaning the fix
   for one problem is also the trigger for the riskiest untested step in the deploy pipeline. This is not
   this audit's invention; `known-red-diagnosis.md` names the same caution in almost the same words.

---

## Compliance Summary

**N/A for all four domains.** No compliance framework (SOC2, GDPR, HIPAA, PCI-DSS, ISO 27001, or any
other) was found adopted, referenced, or implied as a requirement anywhere in `docs/project-requirements.md`,
the PRD, the architecture spine, or any BMad planning artifact reviewed for this audit. This is reported as
a genuine N/A rather than forced into a framework the project never claimed — per the user's own framing,
inventing a compliance status here would be exactly the kind of fabrication this audit is meant to avoid.

---

## Top Priority Actions

### Immediate (Release Blockers)

1. **Fix the `GET /api/v1/employees` authorization gap.** Route builtin-field visibility and per-row
   masking in `listEmployees` through the same `AccessResolver`/`SectionAccessGate` path
   `updateEmployeeField` already uses for writes. Owner: Backend (access-matrix owner). Validation: re-run
   `colleague-whitelist.e2e-spec.ts` and confirm `returns S1-safe directory list entries` passes without
   modification.
2. **Fix the `TS2322` build failure in `campaigns.service.ts`.** A known fix pattern already exists in this
   codebase (`toJsonValue` in `timeline.service.ts:245`) — apply the same pattern to `saveAudience`'s write
   path. Owner: Backend. Caution: this also unblocks the next automatic `prisma migrate deploy` against the
   shared Neon database — confirm the pending migration set is safe before merging, per
   `known-red-diagnosis.md`'s own caution.

### High Priority

3. **Add rate limiting** (e.g., `@nestjs/throttler`) to `/auth/login` and shared-link consumption
   endpoints (R-016).
4. **Fix the employee-list filter DTO defect** (400 "Unknown field \"undefined\"") — root-caused to
   `list-employees-query.dto.ts:83-105`'s interaction with the global `whitelist: true` pipe.
5. **Build a k6 harness and capture real evidence** for the NFR-2 500-record/2s budget before this budget
   can be marked PASS anywhere.
6. **Give `queryEmployees` DB-level pagination and filtering** instead of loading the full employee table
   into memory on every list request — this is the architectural root of the Performance CONCERNS above,
   independent of whether a load test currently shows a breach.

### Medium Priority

7. **Wire `npm audit` and a coverage-threshold check into CI** for both services — neither exists today,
   and both are cheap, well-understood gates.
8. **Adopt structured (JSON) logging** or explicitly narrow the "structured logs suffice" claim in
   `test-design-architecture.md` to match what's actually implemented (plain-text `Logger` output today).
9. **Wire the AD-10 generated-type diff gate into CI** if it is genuinely intended to be enforced — it was
   not found in either workflow file read for this audit despite being described as "Ready."

---

## Evidence Gaps

- [ ] **Load-test evidence for NFR-2** (Performance) — Owner: Backend/QA — Suggested Evidence: a k6
      script against the 500+ record seed dataset — Impact: the one performance target this project has
      cannot be claimed PASS until this exists.
- [ ] **Coverage percentage against the 80%/90% targets** (Maintainability) — Owner: Backend — Suggested
      Evidence: run `test:cov` in CI with a `coverageThreshold` gate — Impact: cannot distinguish "well
      tested" (qualitatively true per `test-review.md`) from "80% covered" (unmeasured) without this.
- [ ] **Backup-restore drill** (Disaster Recovery) — Owner: Team — Suggested Evidence: one documented Neon
      point-in-time-restore test — Impact: an untested backup is not a recovery plan.
- [ ] **Encryption-at-rest confirmation** (Security/Data Protection) — Owner: Team — Suggested Evidence:
      Neon dashboard/documentation confirming default at-rest encryption is active for this project's
      database — Impact: currently inherited, not verified.
- [ ] **Session-invalidation root cause** (Reliability) — Owner: Backend — Suggested Evidence: root-cause
      the three unresolved 401s in `employee-profile.e2e-spec.ts` — Impact: unknown whether this reflects a
      real session-handling bug or a test artifact.
- [ ] **Frontend perceived-performance/degradation UX review** (QoS/QoE) — Owner: Frontend/QA — Suggested
      Evidence: a targeted review of loading-state and error-boundary patterns in `services/frontend/src`
      — Impact: this audit could not assess these two ADR-8 criteria within its evidence scope.

---

## Related Artifacts

- **Test Design (QA):** `_bmad-output/test-artifacts/test-design-qa.md`
- **Test Design (Architecture):** `_bmad-output/test-artifacts/test-design-architecture.md`
- **Known-Red Diagnosis:** `_bmad-output/test-artifacts/known-red-diagnosis.md`
- **CI Pipeline Progress:** `_bmad-output/test-artifacts/ci-pipeline-progress.md`
- **Test Quality Review:** `_bmad-output/test-artifacts/test-review.md`
- **Framework Setup Progress:** `_bmad-output/test-artifacts/framework-setup-progress.md`
- **Deployment Reference:** `docs/deployment.md`
- **PRD:** `_bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md`
- **Evidence sources for this audit:** live `npm audit --json` (both services, 2026-09-05); direct reads of
  `services/backend/src/modules/{auth,directory,access}/**`, `bootstrap.ts`, `prisma/schema.prisma`,
  `.env.example`, both CI workflow YAML files; repo-wide greps for rate-limiting, observability tooling,
  raw-SQL usage, and `k6`/`jscpd` tooling.

---

## Recommendations Summary

**Release Blocker:** Two confirmed, live defects — the `GET /api/v1/employees` authorization gap and the
`campaigns.service.ts` build failure — must be fixed before this system is release-ready. Neither requires
new architecture.

**High Priority:** No rate limiting on auth/shared-link surfaces; a real filter-DTO defect on the employee
list; zero performance evidence for the one defined budget; an unbounded, non-paginated query backing that
same budget.

**Medium Priority:** No CI gate for dependency vulnerabilities, test coverage, or code duplication;
unstructured logging; an apparently-unwired AD-10 type-diff gate.

**Next Steps:** Fix the two blockers, then re-run this audit (or at minimum re-verify the Security and
Reliability sections) before considering the system release-ready. The test *infrastructure* underneath
all of this — injectable clock, external-boundary fault injection, per-worker DB isolation, self-verifying
matrix coverage — is genuinely strong and does not need rework; it is the two live defects and the missing
automated gates (coverage, audit, load test) that need attention, not the testing philosophy.

---

## Sign-Off

**NFR Evidence Audit:**

- Overall Status: FAIL ❌
- Critical Issues: 2 (authorization gap, build failure)
- High Priority Issues: 6
- Concerns: 9
- Evidence Gaps: 6

**Gate Status:** FAIL ❌

**Next Actions:**

- Two release blockers exist — fix both (see "Top Priority Actions — Immediate") before any release
  decision.
- Re-run `bmad-testarch-nfr` after the fixes land, particularly for Security and Reliability.
- Once both blockers clear and load-test evidence exists for NFR-2, this is a reasonable candidate for
  `bmad-testarch-trace` Phase 2 (release gate decision).

**Generated:** 2026-09-05
**Workflow:** `bmad-testarch-nfr` v5.0 (Create mode, sequential execution — subagent launch unavailable in
this session; Steps 04a-04d were performed directly, in sequence, in this document's construction, per the
run's own execution-mode override)

---

<!-- Powered by BMAD-CORE™ -->

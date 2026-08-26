---
runScope: 'system-level'
runKey: 'system'
workflowStatus: 'completed'
totalSteps: 5
stepsCompleted:
  [
    'step-01-detect-mode',
    'step-02-load-context',
    'step-03-risk-and-testability',
    'step-04-coverage-plan',
    'step-05-generate-output',
  ]
lastStep: 'step-05-generate-output'
nextStep: ''
lastSaved: '2026-08-25'
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - _bmad-output/specs/spec-people-management-platform/access-model.md
  - _bmad-output/planning-artifacts/epics.md
  - _bmad-output/implementation-artifacts/sprint-status.yaml
  - _bmad/tea/config.yaml
  - _bmad/config.toml
  - services/backend/package.json
  - services/backend/CLAUDE.md
  - services/frontend/package.json
  - services/frontend/playwright.config.ts
  - services/frontend/CLAUDE.md
  - knowledge/library-integration-mandate.md
  - knowledge/risk-governance.md
  - knowledge/probability-impact.md
  - knowledge/test-priorities-matrix.md
  - knowledge/test-levels-framework.md
  - knowledge/nfr-criteria.md
  - knowledge/test-quality.md
  - knowledge/adr-quality-readiness-checklist.md
---

# Step 1 Output — Mode Detection & Prerequisites

## Mode Decision

**Mode:** System-Level

**Rationale:** Both input sets are present (PRD + architecture decisions AND epics/stories +
`sprint-status.yaml`). File-based detection alone would point to Epic-Level because
`sprint-status.yaml` exists, but every epic is still `backlog`, no test artifacts exist yet, and
the workflow prefers System-Level first when PRD/ADR material is available. User explicitly
confirmed System-Level scope when asked.

## Run Identity

- `run_scope`: `system-level`
- `run_key`: `system`
- Checkpoint path: `_bmad-output/test-artifacts/test-design-progress-system.md`
- Pre-existing checkpoint at this path: none (fresh run)

## Prerequisite Check (System-Level)

| Requirement | Status | Source |
| --- | --- | --- |
| PRD (functional + non-functional requirements) | Available | `prds/prd-people-management-2026-08-21/prd.md` |
| ADR / architecture decision records | Available | `ARCHITECTURE-SPINE.md` (AD-1…AD-13), `specs/.../decisions.md` (D1…D10) |
| Architecture / tech-spec document | Available | `ARCHITECTURE-SPINE.md` + `specs/.../interface-contracts.md` (C1–C8) |

No halt conditions triggered.

## Requirements Scope Snapshot

- 55 functional requirements (FR-1 … FR-55)
- 8 non-functional requirements (NFR-1 … NFR-8)
- 17 UX design requirements (UX-DR1 … UX-DR17)
- 16 profile sections (S1 … S16), 6 viewer audiences
- 13 epics / 61 stories, each epic = one backend module

---

# Step 2 Output — Context & Knowledge Base

## Configuration Resolved

| Setting | Value | Source |
| --- | --- | --- |
| `test_artifacts` | `_bmad-output/test-artifacts` | `_bmad/tea/config.yaml` |
| `tea_use_playwright_utils` | `true` | config |
| `tea_use_pactjs_utils` | `true` | config |
| `tea_pact_mcp` | `mcp` | config |
| `tea_browser_automation` | `auto` | config |
| `tea_execution_mode` | `auto` | config |
| `test_stack_type` | `auto` → resolved `fullstack` | auto-detected |
| `risk_threshold` | `p1` | config |
| `ci_platform` | `auto` → **UNKNOWN** (no CI config committed) | auto-detected |
| `test_framework` | `auto` → Jest (backend) + Playwright (frontend) | auto-detected |

## Stack Detection

`detected_stack` = **fullstack**.

**Backend** (`services/backend`, gitlink `340a588`, branch `main`):

- NestJS 11 (Express, CommonJS), TypeScript strict, Node 22 pinned via `.nvmrc`
- Prisma 7 + `@prisma/adapter-pg`, PostgreSQL 18 via `docker-compose`
- `@nestjs/config` + Joi env validation; global `ValidationPipe`; Swagger at `/api/docs`;
  `@nestjs/terminus` health at `/api/v1/health`
- Tests: Jest 30 + ts-jest. Unit `testRegex: .*\.spec\.ts$` rooted at `src`; e2e via
  `test/jest-e2e.json` with supertest 7
- Jest scripts require `cross-env NODE_OPTIONS=--experimental-vm-modules` (Prisma 7 WASM client)

**Frontend** (`services/frontend`, gitlink `8b39eca`, branch `main`):

- React 19 + Vite 8, TypeScript 5.7, Tailwind v4 + shadcn/ui (radix-nova), React Router v7
- TanStack Query v5 + Axios, i18next / react-i18next
- Tests: `@playwright/test` 1.61, `testDir: ./e2e`, chromium-only project, `fullyParallel: true`,
  `retries: 2` in CI, `workers: 1` in CI, `baseURL http://127.0.0.1:4200`, Vite started via
  `webServer`

**Existing test inventory (baseline is effectively empty):**

| Location | Files | Notes |
| --- | --- | --- |
| `services/backend/test/` | `app.e2e-spec.ts`, `users.e2e-spec.ts`, `jest-e2e.json` | Starter e2e only |
| `services/backend/src/modules/*/__tests__/` | none found | Convention declared in AD-13, not yet populated |
| `services/frontend/e2e/` | `app.spec.ts`, `shared/test-env.ts` | Starter smoke spec only |

No component-test tooling, no k6, no axe/a11y tooling, no CI workflow files, no coverage
thresholds configured. Backend Jest `collectCoverageFrom` is set but no threshold gate exists.

## Library Gate Outcomes (per `library-integration-mandate.md` — two gates)

| Flag | Gate 1 (flag) | Gate 2 (package installed) | Binding? | Consequence |
| --- | --- | --- | --- | --- |
| `tea_use_playwright_utils` | ✅ `true` | ❌ `@seontechnologies/playwright-utils` absent from both manifests | **Does not bind** | Generate the vanilla Playwright path. Stated once here; recommend the `bmad-testarch-framework` workflow if the utilities are wanted. |
| `tea_use_pactjs_utils` | ✅ `true` | ❌ `@seontechnologies/pactjs-utils` absent from both manifests | **Does not bind** | See relevance assessment below. |
| `tea_pact_mcp` | ✅ `mcp` | ❌ No SmartBear/Pact MCP tools reachable in this session | **Does not bind** | `pact_mcp_reachable: false`. Reported once; no broker data inferred or invented. |

**Contract-testing relevance assessment (independent of the flags):** Pact is **not recommended**
for this system. The architecture already ratifies a different contract mechanism for the one
consumer/provider pair that exists — AD-10 generates frontend types from the backend's Swagger
surface via `openapi-typescript` and CI-diffs the committed file, failing the build on drift. There
is a single consumer, a single provider, and a single deploy unit (AD-12), so a broker-mediated
consumer-driven contract adds ceremony without adding a signal AD-10 does not already produce. The
`tea_use_pactjs_utils` default of `true` is not an instruction to add contract tests to this
project. **If** the AD-10 CI diff is not actually wired, that is a coverage gap to fix in AD-10's
own terms, and it is carried into the risk register rather than substituted with Pact.

**Browser exploration:** skipped, and not for a tooling reason — there is no running application to
explore. Every epic is `backlog` and the feature surfaces (11 per UX-DR14) do not exist yet. This
run is grounded in document and code analysis, as system-level mode intends.

## Knowledge Fragments Loaded

**System-Level required set:** `adr-quality-readiness-checklist.md`, `nfr-criteria.md`,
`test-levels-framework.md`, `risk-governance.md`, `test-quality.md`

**Additional (needed for scoring and prioritization in steps 3–4):** `library-integration-mandate.md`
(loaded first, per the mandate-ordering rule), `probability-impact.md`, `test-priorities-matrix.md`

**Deliberately not loaded:** Pact/Playwright-Utils per-utility fragments (both mandates non-binding
per the gate table above); mobile fragments (`maestro-*`, `mobile-*`) — PRD §6 puts native mobile
apps explicitly out of scope; webhook fragments — no webhook surface in any of the 55 FRs.

## Extracted: Integration Points

| Integration | Direction | Criticality | Governing rule |
| --- | --- | --- | --- |
| Timetracker Projects & People API | inbound → `ProjectAssignment` (C3) | **Permission-critical** — feeds Manager access resolution | AD-8, D3, FR-53 |
| Timetracker Leaves API | inbound → S10 | Display only | AD-8 fail-soft, FR-52 |
| PeopleForce Candidates/Vacancies | inbound → `resourcing` | Optional; outbound-link fallback is sanctioned permanent behavior | FR-54, D8 |
| Frontend ↔ Backend REST | internal | Type contract | AD-10 (`openapi-typescript`, CI-diffed) |
| Prisma Client Extension → `TimelineEventWriter` (C4) | internal, implicit | Structural coupling on 4 history tables | AD-7 |
| Provider Registry (`DiscoveryService`) | internal, bootstrap-time | Cross-module reads | AD-3 |

## Extracted: NFR Thresholds

| NFR | Threshold | Status | Evidence source planned |
| --- | --- | --- | --- |
| NFR-1 / SM-1 | Every audience × relationship-path × section combination, every `—` cell, every flag-gated record against both gated audiences has a passing negative test | **Measurable** | Parameterized Jest suites (AD-13) + Playwright negative e2e |
| NFR-2 / SM-4 | All Employees < 2s at 500+ records under arbitrary filters, **including permission resolution** | **Measurable** | k6 (system latency) + CI benchmark gate |
| NFR-3 / SM-7 | WCAG 2.1 AA on List, Profile, Dashboard | **Measurable standard, tooling UNKNOWN** | axe automated scan + manual keyboard/SR pass |
| NFR-4 | External integration failures degrade gracefully, never take the app down | **Qualitative — no explicit RTO/error-budget** | Fault-injection integration tests |
| NFR-5 | Non-prod uses pseudonymized data only; no real PII in agent contexts, logs, screenshots, repo | **Measurable as a prohibition** | Seed-tool tests + CI secret/PII scan |
| NFR-6 | Zero developer-waits-on-developer against the wave/track plan | **Process metric, not a test** | Sprint-status / PR history review |
| NFR-7 / SM-6 | Intelligent repository matches shipped behavior at demo (spot-checked) | **Process metric, not a test** | Doc-vs-behavior spot check |
| NFR-8 | Access evaluated server-side, per section, per request; any cache invalidates on generation-bump only, never bare TTL | **Measurable** | Unit + integration tests on `AccessResolver` |

**UNKNOWN thresholds** (recorded, not guessed — carried into step 3 as clarification items):
p95/p99 latency for endpoints other than the All Employees list; AD-8's freshness-window value
(mechanism fixed, number explicitly deferred); upload size/format limits for FR-15; concurrency
level behind the "2 second" target (single user vs. N concurrent); availability/SLA target; RTO/RPO.

## Confirmation of Loaded Inputs

All system-level prerequisites loaded successfully. Two capability gaps were found and are recorded
rather than worked around: no CI platform is configured in either repository (so every "CI-enforced"
rule in AD-1, AD-10, and AD-13 currently has no enforcement surface), and no running application
exists to explore.

---

# Step 3 Output — Testability Review, Risk Assessment, NFR Planning

## 🚨 Testability Concerns

Ordered by how much each one threatens a graded success metric. Every concern names the artifact it
was read from, so it can be argued with.

### TC-1 — The access matrix has no machine-readable source, so the suite AD-13 specifies cannot actually be data-driven

Story 1.14's acceptance criterion says the harness is "data-driven off `access-model.md`'s matrix
itself, so a future matrix change automatically extends or flags outdated coverage," and AD-13's
Consistency Conventions repeat it: the suite is "generated/maintained alongside `access-model.md` so
the two never drift." But `access-model.md`'s matrix is a Markdown table in prose, and nothing in the
stack parses Markdown. As written, the requirement has no implementation, which means the suite will
be hand-maintained and will drift — and SM-1 (the SPEC's primary metric) is exactly the claim that
depends on it not drifting.

**Required:** promote the matrix to a single machine-readable fixture (e.g.
`access-matrix.ts`/`.json` in `contracts`, exporting `Record<SectionId, Record<Audience, 'R'|'RW'|'none'>>`
plus the field-level and flag-gated exceptions), make `access-model.md` render from it or assert
against it, and have every `SectionProvider` suite plus the master `AccessResolver` suite iterate
that one fixture. A matrix cell that gains no test then becomes a compile/collection error rather
than an omission nobody notices.

### TC-2 — Parallel Jest workers against one shared Postgres will produce false greens on the matrix suite

The backend runs Jest 30 with default worker parallelism and e2e against a real Postgres
(`npm run db:up` then `npm run test:e2e`). The starter `users.e2e-spec.ts` already writes to that
shared database. AD-13 then loads a very large parameterized negative suite on top of it. Shared
mutable state plus parallel workers is the standard recipe for order-dependent passes — and a
false green on a negative access test is strictly worse than no test, because SM-1 gets attested
without being true.

**Required:** decide the isolation mechanism before the first `SectionProvider` suite is written.
Per-worker schema (`JEST_WORKER_ID`-suffixed Postgres schema, migrated once) or a
transaction-per-test rollback wrapper are both viable; the choice is less important than making it
before 61 stories' worth of suites exist. Also add an explicit "no cross-test data reuse" rule to
the matrix convention, since a matrix case that reads a relationship another case just mutated
fails nondeterministically.

### TC-3 — No per-test relationship-graph factory; the seed tool solves volume, not precision

Every cell of the access matrix is a pure function of the relationship graph: reports-to at
arbitrary depth, project assignment with a PM and a DM, PP assignment plus the HR line above it,
functional-role assignment, plus the S7/S8 flag states. Story 1.16's seed tool is specified for
realistic **volume** (500+ pseudonymized records) — that is what NFR-2 and NFR-5 need, and it is not
what a matrix case needs. A matrix case needs "D is two levels above B and nothing else is true."

**Required:** a typed graph factory alongside the seed tool — `aManager({ over })`,
`aProjectWith({ pm, dm, members })`, `aPeoplePartnerFor({ employee, hrLineAbove })`,
`aNoteAbout({ employee, visibleForEmployee, visibleForPm })` — composing to an exact graph per case
and tearing down with it. Without it, each of the three parallel developers invents their own
fixture shape and the matrix cases stop being comparable across modules.

### TC-4 — Three separate requirements are time-dependent, and no injectable clock is named

Four behaviours cannot be tested deterministically against a wall clock:

- AD-8's freshness window — a `ProjectAssignment` row confirmed before an outage must **age out** and
  stop granting Manager access. The window value is explicitly deferred ("a config number"), which is
  good, but the test also needs to move time, not just shorten the window.
- FR-6 / Story 1.12 — the 24-hour shared-link expiry.
- FR-20 / Story 4.3 — overdue derivation, which the PRD requires to be derived consistently and
  explicitly *not* a stored flag set by a batch job.
- FR-34 / Story 8.4 — "assessed before 2025-01-01" and "never assessed" as distinct filter states.

**Required:** a single injectable time source (a `ClockService` bound in `contracts`, or Nest's
provider override in tests) used by every one of those four paths. Frontend-side, Playwright's
`page.clock` covers the UI half. A test that sleeps, or one that only shrinks the config window,
verifies less than it appears to.

### TC-5 — External HTTP boundaries have no named stubbing mechanism

AD-2 gives every internal contract (C1–C8) a Wave-0 stub provider, which is genuinely strong. The
external boundaries have nothing equivalent: the timetracker Leaves API, the timetracker Projects &
People API, and PeopleForce are raw HTTP dependencies of `integrations`. NFR-4 and FR-52/FR-53
require testing unreachable-API and prolonged-outage behaviour, and SM-3 additionally requires the
timetracker to run against its **real** API in the demonstrated build — so real and stubbed modes
have to coexist deliberately rather than by accident.

**Required:** name the mechanism (an injected HTTP client interface with a fake implementation, or
`nock`/MSW at the transport layer) and name which suites run against real endpoints. SM-3's "real
API, not seed/mock data" claim needs a test tier that is unambiguously not stubbed, or the claim is
unfalsifiable.

### TC-6 — Frontend Playwright cannot exercise a real API, so "absent from the response" is unverifiable there

`playwright.config.ts` starts only `npx vite` on port 4200. There is no backend and no Postgres in
`webServer`. But the negative cases that matter most are precisely about the API response body:
Story 1.8 asserts a Colleague's direct profile-API call returns "only S1, S10, and S11 keys, with no
other section data present at any level," and Story 1.14 asserts a DM inspects both the rendered page
**and** the underlying API. Neither is reachable from today's e2e setup.

**Required:** either extend `webServer` to bring up backend + Postgres (docker-compose, which AD-12
already mandates for deployment), or place the response-shape negative cases in the backend's
supertest e2e tier and keep Playwright for the rendered-surface half. Both are defensible; leaving it
implicit means the assertion lands in neither tier.

### TC-7 — `retries: 2` plus `workers: 1` in CI hides flakiness and serialises the largest suite

`retries: process.env.CI ? 2 : 0` means a test that passes on the third attempt reports as passing.
For a suite whose entire purpose is proving negative access cases, a retry-masked intermittent pass
is indistinguishable from a real one. `workers: process.env.CI ? 1 : undefined` simultaneously
serialises execution, which is the wrong trade for a large matrix suite.

**Required:** keep retries for genuinely environmental failures but surface flaky-passes as a
build annotation, and revisit `workers: 1` once TC-2's isolation mechanism makes parallelism safe.
`fullyParallel: true` combined with `workers: 1` is currently self-cancelling.

### TC-8 — Two observability outputs the tests need to assert on are not modelled as data

- **Shared-link access log.** FR-6 and Story 1.12 require every access attempt, successful or not,
  to be logged with when and where it originated. The ER diagram in the Structural Seed has
  `SharedLink` and `SharedLinkSection` and no access-log entity. A `Logger.log()` line is not
  assertable without log capture, so as modelled this requirement has no test.
- **Provider-unavailable state.** AD-3 requires a missing registration to surface as an explicit
  "temporarily unavailable" state and calls a silent omission "a genuine data-leak-shaped bug" —
  correctly, because indistinguishable-from-not-granted is the failure mode. But C1's ratified
  signature returns only `{ role, sections }`, and no DTO shape for "granted but unavailable" appears
  in AD-2's contract table. Two developers will represent it two ways.

**Required:** add the access-log entity to the schema, and add the unavailable-state discriminator to
the contract before the first `SectionProvider` ships.

### TC-9 — Single environment means the perf/seed environment is the demo environment

AD-12 scopes exactly one environment (`production`), no staging tier. NFR-5 requires non-production
environments to use pseudonymized data only — but with one environment there is no "non-production"
to scope that to, and Story 1.16 seeds 500+ pseudonymized records into it. NFR-2's load test
(Story 3.7) then runs against the same instance that SM-3's real timetracker integration and the
final demo depend on.

This is a consequence of a deliberate scope decision, not an oversight, so the right response is a
stated policy rather than a new environment: name where the 500-record dataset lives, confirm the
load test runs against a disposable local docker-compose stack rather than the demo instance, and
record that "pseudonymized everywhere, always" is the actual NFR-5 posture under a single-environment
topology.

### TC-10 — Dynamic filter/sort over arbitrary field names is the one injection-shaped surface

FR-7 makes any built-in, derived, or custom field filterable and sortable, and FR-8 lets HR Admin
create fields at runtime. That means a client-supplied string reaches a query builder. Prisma
parameterizes **values**, not identifiers, and the global `ValidationPipe` cannot whitelist a field
set that is defined at runtime. AD-6 already puts the only type-branching point inside C2
`FieldRegistry`, which is the correct chokepoint.

**Required:** treat C2 as an allow-list oracle — a filter or sort naming a field C2 does not return
for this viewer is rejected before any query is built — and test it with unknown, malformed, and
not-authorized field names, plus the count-inference case in TC-11's risk entry.

## ✅ Testability Assessment Summary — what is already strong

These are load-bearing and should not be re-litigated; they are why this architecture is testable at
all.

| Strength | Why it matters for testing | Source |
| --- | --- | --- |
| Access resolution is one layer, called once per request | The matrix has exactly one place to test against, not sixteen re-implementations | AD-1, AD-3, AD-4 |
| Colleague is a resolved grant map, never a post-hoc filter | Removes the entire class of "assemble everything then trim" bugs, and makes the whitelist assertable at the resolver rather than at every surface | AD-5 |
| Every contract has a Wave-0 stub provider | Any module is testable in isolation before its dependencies exist — this is criterion 1.1 satisfied by design | AD-2 |
| Frontend holds no independent access logic | Nothing to keep in sync, so no frontend/backend divergence class of test is needed | Paradigm section, AD-10 |
| 100% of business logic is reachable over REST with Swagger | Criterion 1.2 fully covered; no UI-only logic forces an e2e test | AD-10, existing `modules/users` |
| History writes are structurally coupled to the timeline via a Prisma extension | The coupling cannot be forgotten per-call-site, so tests assert one mechanism rather than N call sites | AD-7 |
| A skipped system write is itself a recorded event | FR-30's hardest case is observable instead of being a silent no-op | AD-7, D2 |
| Registration collision fails at bootstrap | A whole class of ambiguity becomes a startup error, testable with one Nest test module | AD-3 |
| Custom fields need no migration | Runtime field definition is testable in-process; no schema-migration step inside a test | AD-6 |
| Secrets are env + Joi-validated, `.env` gitignored | No secret handling to test around; misconfiguration fails fast at boot | AD-12 |
| Risk `trend` is computed backend-side once | One assertion target instead of three frontend re-derivations | AD-3 |
| Errors use one JSON envelope from a global filter | Negative assertions have a stable shape | AD-2 conventions |

## Architecturally Significant Requirements (ASRs)

| ID | Requirement | Source | Classification |
| --- | --- | --- | --- |
| ASR-1 | Access is resolved server-side, per section, on every request — no session-stable assumption | NFR-8, AD-4, matrix rule 4 | **ACTIONABLE** |
| ASR-2 | Colleague whitelist is a resolved section-grant map, enforced at the data-access layer | FR-3, AD-5, matrix rule 3 | **ACTIONABLE** |
| ASR-3 | All Employees < 2s at 500+ records **including permission resolution** | NFR-2, SM-4 | **ACTIONABLE** |
| ASR-4 | Project-assignment access fails safe on stale `confirmedAt`; display fails soft, independently | FR-53, AD-8, D3 | **ACTIONABLE** |
| ASR-5 | Functional roles and custom fields are runtime data — zero deploy, zero migration | FR-5, FR-8, AD-6, SM-2 | **ACTIONABLE** |
| ASR-6 | No feature module imports another feature module (CI-enforced via `dependency-cruiser`) | AD-1 | **ACTIONABLE** |
| ASR-7 | Frontend API types are generated from Swagger and CI-diffed against the committed file | AD-10 | **ACTIONABLE** |
| ASR-8 | History-table writes fire `TimelineEventWriter` structurally, via a Prisma extension | AD-7, FR-28, FR-30 | **ACTIONABLE** |
| ASR-9 | Registry collision = bootstrap failure; missing registration = explicit unavailable, never silent | AD-3 | **ACTIONABLE** |
| ASR-10 | A Shared Link is read-only under every configuration; S3/S7/S13 rejected server-side | FR-6, AD-5, matrix rule 7 | **ACTIONABLE** |
| ASR-11 | Any access-resolution cache invalidates on a relationship-graph generation bump, never a bare TTL | NFR-8, D1, AD-4, SM-C2 | **ACTIONABLE** |
| ASR-12 | Auth answers only "who is this session"; `userId` reaches other modules only via C7 | AD-9, D9 | **ACTIONABLE** |
| ASR-13 | WCAG 2.1 AA on List, Profile, and Dashboard surfaces | NFR-3, SM-7, UX-DR16 | **ACTIONABLE** |
| ASR-14 | Single containerized deploy, one environment, no staging tier | AD-12 | FYI — constrains where NFR-2 and NFR-5 evidence can be produced (see TC-9) |
| ASR-15 | Wire-safety: every DTO crossing HTTP is plain-JSON-serializable (`Record`, not `Map`) | AD-2 | FYI — a serialization bug here surfaces as a missing section, which reads as a leak-shaped defect |

## Risk Register

Scored per `probability-impact.md`: probability 1 unlikely / 2 possible / 3 likely; impact 1 minor /
2 degraded / 3 critical. Action per score: 1–3 DOCUMENT, 4–5 MONITOR, 6–8 MITIGATE, 9 BLOCK.

"BLOCK" here means the item must be resolved before the **foundation-phase gate** PRD §1 already
requires, not that the project is failing. All three blockers are foundation-phase work by nature.

| ID | Cat | Risk | P | I | Score | Action |
| --- | --- | --- | --- | --- | --- | --- |
| R-001 | SEC | A section leaks through a surface other than the profile API — `.xlsx` export, a shared saved view, a filter/sort on a non-visible field, campaign audience preview, a dashboard aggregate, or a search result | 3 | 3 | **9** | BLOCK |
| R-002 | TECH | Parallel Jest workers on a shared Postgres yield order-dependent passes, producing a false green on the negative access suite | 3 | 3 | **9** | BLOCK |
| R-003 | OPS | No CI pipeline exists, so every CI-enforced invariant (AD-1 boundaries, AD-10 type diff, AD-13 matrix suite, NFR-2 benchmark, NFR-5 PII scan) is advisory only and SM-1 is unverifiable | 3 | 3 | **9** | BLOCK |
| R-004 | PERF | Transitive-closure access resolution per row at 500+ records breaches the 2s budget (NFR-2 / SM-4) | 3 | 2 | 6 | MITIGATE |
| R-005 | SEC | Under R-004 pressure, a cache is added with TTL invalidation, producing a stale permission grant — the SM-C2 counter-metric realised | 2 | 3 | 6 | MITIGATE |
| R-006 | SEC | A `ProjectAssignment` row confirmed before an outage keeps granting Manager access because the freshness window is unset, too wide, or untested | 2 | 3 | 6 | MITIGATE |
| R-007 | TECH | The access-matrix suites drift from `access-model.md` because the "data-driven" requirement has no machine-readable source (TC-1) | 3 | 2 | 6 | MITIGATE |
| R-008 | DATA | A history-table write bypasses the Prisma extension, or the extension double-fires, so the timeline silently diverges from employment history and FR-30's conflict rule is defeated | 2 | 3 | 6 | MITIGATE |
| R-009 | SEC | Custom-field visibility is inferable through filter result counts even though the field is excluded from columns and responses | 2 | 3 | 6 | MITIGATE |
| R-010 | SEC | Shared-link expiry, revocation, and invalid-token responses are distinguishable from each other, or access attempts are unlogged and therefore unauditable (TC-8) | 2 | 3 | 6 | MITIGATE |
| R-011 | TECH | Frontend e2e cannot assert API response shape, so the highest-value negative cases land in no tier (TC-6) | 3 | 2 | 6 | MITIGATE |
| R-012 | SEC | A dynamic filter or sort naming a runtime-defined field reaches the query builder unvalidated (TC-10) | 2 | 3 | 6 | MITIGATE |
| R-013 | DATA | A missing provider registration renders as an omission indistinguishable from "not granted", because the unavailable state has no contract shape (TC-8) | 2 | 3 | 6 | MITIGATE |
| R-014 | BUS | Campaign activation partially completes, splitting recipients between having an action item and not (FR-42) | 2 | 2 | 4 | MONITOR |
| R-015 | OPS | Time-dependent behaviour (freshness window, link expiry, overdue, assessment recency) is tested against a wall clock, so tests are slow, flaky, or vacuous (TC-4) | 3 | 2 | 6 | MITIGATE |
| R-016 | SEC | Shared-link and login endpoints have no rate limiting, permitting token guessing or credential stuffing | 2 | 2 | 4 | MONITOR |
| R-017 | OPS | Load-testing and 500-record seeding run against the only environment, which is also the demo environment (TC-9) | 2 | 2 | 4 | MONITOR |
| R-018 | TECH | Mentorship `endedAt` is set without the required closing feedback, which the spine itself flagged as below-altitude and unresolved | 2 | 2 | 4 | MONITOR |

### Mitigations for the three blockers

**R-001 — surface inventory, not just a profile suite.** The whitelist being a resolved grant map
(AD-5) removes the *resolver* as a leak source; it does not remove the surfaces. Enumerate every
surface that emits person data — profile API, All Employees rows, `.xlsx` export, saved-view
execution for a second viewer, campaign audience preview, all four dashboards, the shared-link view,
and error/empty states — and require each one to be covered by the same matrix fixture rather than by
one bespoke test per surface. Story 1.15 already scopes the strategy beyond the profile matrix into
Risks, Resourcing, and Action Items; this extends the same reasoning to output surfaces.
*Owner: test architecture (foundation phase). Timeline: before the first `SectionProvider` ships.*

**R-002 — pick the isolation mechanism first.** Per-worker Postgres schema or transaction-rollback
per test, decided and scaffolded before suite volume accumulates, plus a rule that no matrix case
reads state another case wrote. *Owner: backend substrate (Story 1.19's neighbourhood).
Timeline: Wave 0, ahead of Story 1.14/1.15.*

**R-003 — stand up the pipeline in the foundation phase.** Five specified invariants already name CI
as their enforcement point; AD-12 leaves the pipeline choice open as "a delivery-tooling choice for
Wave 0," which is exactly now. Minimum viable set: lint + typecheck, `dependency-cruiser`,
`openapi-typescript` regenerate-and-diff, Jest unit + e2e with the TC-2 isolation mechanism,
Playwright e2e, and a committed-focus grep (`.only`/`fdescribe`/`fit`). The NFR-2 benchmark and the
PII scan can follow. *Owner: Story 1.17's process setup. Timeline: foundation phase, before feature
work.*

## NFR Planning Assessment

Scope note per the workflow's own boundary: this plans NFR **validation**. It does not assign
PASS/CONCERNS/FAIL from implementation evidence — there is no implementation yet. Run `nfr-assess`
once evidence exists.

### ADR Quality Readiness — category rollup (8 categories, 29 criteria)

| Category | Criteria met | Verdict | Principal gap |
| --- | --- | --- | --- |
| 1. Testability & Automation | 1/4 | ⚠️ CONCERNS | No state-control mechanism for precise relationship graphs (TC-3); no external stubbing (TC-5) |
| 2. Test Data Strategy | 1/3 | ⚠️ CONCERNS | No teardown/isolation mechanism (TC-2); no test-data segregation under a single environment (TC-9) |
| 3. Scalability & Availability | 1/4 | ⚠️ CONCERNS | No availability target; no timeout/circuit-breaker policy for the two external clients |
| 4. Disaster Recovery | 0/3 | ⚠️ CONCERNS — **scope-accepted** | RTO/RPO and backups undefined; failover explicitly deferred by AD-12 |
| 5. Security | 3/4 | ✅ PASS with caveat | TLS / at-rest encryption unstated; FR-7's dynamic field names need the C2 allow-list (TC-10) |
| 6. Monitorability | 1/4 | ⚠️ CONCERNS | No metrics endpoint, no correlation IDs, no structured-log or dynamic-level statement |
| 7. QoS & QoE | 1/4 (1 not assessed) | ⚠️ CONCERNS | Only one latency target exists, and only for one endpoint; no rate limiting |
| 8. Deployability | 0/3 | ⚠️ CONCERNS — **scope-accepted** | Single container: no zero-downtime path, no N-1 schema policy, no health-gated rollback |

**Overall: 8/29 criteria met (28%) → CONCERNS.**

That number needs reading carefully rather than treating as a grade. Categories 4 and 8, and much of
3 and 6, are gaps *because* AD-12 deliberately scopes one container and one environment for this
iteration — recording them is right, but closing them is not this project's job. Sorting them:

**Gaps that threaten a graded success metric — close these:**

- Category 1 and 2 gaps → SM-1 (TC-1, TC-2, TC-3, R-001, R-002, R-007)
- Category 3.2 and 7.1 gaps → SM-4 (R-004, no baseline, no benchmark, no concurrency definition)
- Category 5.4's caveat → SM-1 again, via the filter surface (TC-10, R-012)
- Category 6.3's missing measurement point → SM-4 cannot be measured without one
- Absence of CI across all of them → SM-1, SM-4, and SM-7 all become self-attested (R-003)

**Gaps that are accepted consequences of AD-12 — record and move on:**

- Category 4 entirely (RTO/RPO, failover, backups) — single container, one environment, no data-loss
  liability at bootcamp scale
- Category 8 entirely (zero downtime, N-1 schema, automated rollback) — AD-6 already removes the
  highest-churn migration source by making custom fields migration-free
- Category 6.1 (distributed tracing) — one service, so correlation IDs buy little beyond the
  shared-link audit trail, which TC-8 addresses as data instead
- Category 3.1 is genuinely covered: JWT plus no server-side session means the app is already
  stateless

### Planned evidence sources per NFR

| NFR | Validation approach | Tool / tier | Threshold status |
| --- | --- | --- | --- |
| NFR-1 (access matrix) | Parameterized suite per section provider off one machine-readable matrix; master suite on `AccessResolver`; negative e2e per output surface | Jest unit + Jest/supertest e2e + Playwright | Measurable, exhaustive by construction |
| NFR-2 (2s @ 500+) | k6 for system latency under load; CI benchmark gate on the list endpoint; seeded 500+ dataset | k6 + CI job | **Partly UNKNOWN** — concurrency level undefined; p95/p99 undefined |
| NFR-3 (WCAG 2.1 AA) | Automated axe scan on List/Profile/Dashboard + manual keyboard and screen-reader pass; viewport projects for the `sm` collapse | Playwright + axe (tool not yet chosen) | Measurable standard; tooling UNKNOWN |
| NFR-4 (graceful degradation) | Fault injection at the external HTTP boundary: unreachable, slow, malformed, prolonged outage | Backend integration tests with a stubbed client (TC-5) | **UNKNOWN** — no timeout, retry, or error-budget number stated |
| NFR-5 (pseudonymized data) | Seed-tool assertions that no field carries real-looking PII; CI secret/PII scan; policy statement per TC-9 | Jest + CI scan | Measurable as a prohibition |
| NFR-6 (zero blocking) | Delivery-record review against the wave/track plan | Process review, not a test tier | Process metric |
| NFR-7 (repo fidelity) | Doc-vs-behaviour spot check at demo | Process review, not a test tier | Process metric |
| NFR-8 (per-request evaluation) | Unit tests asserting resolution is recomputed per call; integration test that a mid-session relationship change takes effect on the next request; a guard test that no cache exists, or that it invalidates on generation bump | Jest unit + e2e | Measurable |

### Clarification Items (UNKNOWN — recorded, not guessed)

| ID | Question | Blocks | Owner |
| --- | --- | --- | --- |
| Q-1 | What concurrency does "2 seconds at 500+ records" assume — one viewer, or N simultaneous? A single-user 2s target and a 2s p95 under load are different systems | NFR-2 test design, k6 scenario shape | PM + Architect |
| Q-2 | Are there latency targets for any endpoint other than the All Employees list (profile assembly, dashboards, export)? | Perf scope beyond SM-4 | PM |
| Q-3 | What is AD-8's freshness-window value, and what is the expected timetracker sync interval it derives from? | R-006, TC-4 | Architect + D8 investigation |
| Q-4 | Should a filter naming a non-visible custom field be **rejected** or **silently excluded**? Story 1.10's AC permits either, and they are externally distinguishable | R-009, R-012 | Architect |
| Q-5 | What timeout, retry, and failure-threshold policy do the timetracker and PeopleForce clients use? | NFR-4, criterion 3.4 | Architect |
| Q-6 | Which CI platform? Five invariants name CI as their enforcement point and AD-12 leaves it open | R-003 | Team (Story 1.17) |
| Q-7 | Where does the 500-record dataset live, and does the load test run against the demo instance? | TC-9, R-017, NFR-5 | Team |
| Q-8 | Is the shared-link access log an entity, and what does it record? | TC-8, R-010 | Architect |
| Q-9 | What wire shape distinguishes "granted but temporarily unavailable" from "not granted"? | TC-8, R-013 | Architect (AD-2/AD-3) |
| Q-10 | Are FR-15 upload size/format limits left to convention, or does the org have a requirement? | Upload negative tests | PM |

## Highest Risks — Summary

Three items must clear before feature work, and they are all foundation-phase by nature, which is
where PRD §1 already puts a gate:

1. **No CI (R-003).** Five architectural invariants name CI as their enforcement point. Until it
   exists, AD-1's module boundaries, AD-10's type diff, and AD-13's matrix suite are conventions, and
   SM-1 is a claim rather than a result.
2. **Test isolation (R-002).** Parallel Jest workers on a shared Postgres will make the negative
   access suite intermittently green. Decide the mechanism before 61 stories of suites exist, because
   retrofitting isolation across them later is the expensive version of this fix.
3. **Leak surface breadth (R-001).** AD-5 makes the resolver trustworthy; it does not make export,
   saved views, audience previews, and dashboard aggregates trustworthy. Each is its own emission
   path, and each needs to be driven off the same matrix fixture.

Underneath those, the sharpest single design gap is **TC-1**: the requirement that the suite be
data-driven off `access-model.md` currently has no implementation, because that file is prose. Fixing
it — one machine-readable matrix that both the doc and every suite derive from — is also the cheapest
mitigation available for R-001 and R-007 simultaneously, which is why it leads the coverage plan.

---

# Step 4 Output — Coverage Plan & Execution Strategy

## Counting Convention

Two units are used deliberately, because conflating them would inflate the estimate by an order of
magnitude:

- **Test group** — one hand-written, maintained artifact (a `describe` block, spec file, or harness).
- **Case** — one assertion instance. A parameterized group over the access matrix produces ~96 cases
  from one authored harness.

Effort tracks **groups**; confidence tracks **cases**. The access domain is where they diverge most:
it dominates case count and barely moves the effort line, which is exactly why the machine-readable
matrix (TC-1) is the highest-leverage item in the plan.

## Test-Level Assignment Rationale

Applying `test-levels-framework.md` to this architecture:

- **Unit (Jest, no DB)** — `AccessResolver` decisions, relationship-path derivation, overdue and risk
  derivations, AD-8 fail-safe branches. These are pure functions of a relationship graph and a matrix;
  they are the cheapest place to get combinatorial coverage, and they are where the 96-cell matrix
  belongs.
- **Integration (Jest + real Postgres)** — Prisma extension side effects, campaign activation
  atomicity, registry bootstrap, transactional invariants. Anything asserting "the database ended up
  in this state" cannot be a unit test here, because the extension is the mechanism under test.
- **API/e2e (Supertest against the Nest app)** — emission-surface assertions. The unit layer proves
  the resolver denies; only a response-body assertion proves the field never reached the wire. This
  distinction is the core of R-001 and justifies what would otherwise look like duplicate coverage.
- **E2E (Playwright)** — a thin set of critical journeys and the accessibility scans. Deliberately
  thin: `test-levels-framework.md` reserves E2E for critical paths, and every access permutation
  driven through a browser would be the slowest possible way to learn what a unit case already knows.

**Intentional overlap, declared once:** access rules are asserted at unit level (decision) *and* at
API level (emission). This is not redundancy — they can disagree, and the disagreement is the bug
class SM-1 exists to prevent. The overlap is bounded: unit covers all 96 cells, API covers the ~18
`—` cells plus one representative cell per emission surface.

## P0 — Critical

**Criteria:** security, data-integrity, or compliance impact with no safe workaround. In this system
that is almost entirely "the access model, on every surface that emits data."

| Test ID | Requirement | Test Level | Risk Link | Groups | Cases | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **P0-001** | Access matrix, all 16 sections × 6 audiences, driven off the machine-readable matrix | Unit | R-001, R-007 | 1 harness | ~96 | Blocked on TC-1. Fails loudly on an unmapped cell — a new section must not default to allowed |
| **P0-002** | Relationship-path resolution: direct, transitive, project PM, project DM, PP, union of legs | Unit | R-001, R-007 | 1 | ~12 | Needs the graph factory (TC-3) |
| **P0-003** | Negative `—` cells asserted on the wire (key absent, not null) | API | R-001 | ~4 | ~18 | Absence assertion, not falsy assertion |
| **P0-004** | Colleague whitelist holds on every emission surface: profile, list rows, xlsx export, saved-view re-execution, campaign audience preview, dashboard aggregate, shared link, search/autocomplete | API + E2E | R-001 | ~8 | ~16 | One per surface; the surface list is the deliverable |
| **P0-005** | S7 record-level filtering: employee flag × PM-visible flag × unconditional UM/DM/PP | Unit + API | R-001 | 2 | ~8 | Two independent flags on one section — the densest single rule in the matrix |
| **P0-006** | S8 `sharedWithEmployee` flip changes Self visibility on the next request | API | R-001, R-005 | 1 | 2 | Also covers the no-caching claim |
| **P0-007** | S1 mentor-field override denies Colleague while the section is otherwise readable | Unit + API | R-001 | 1 | 2 | Field-level exception inside a granted section |
| **P0-008** | S16 custom fields absent from profile, columns, filters, and export per visibility level; no existence inference via counts or errors | API | R-009, R-012 | 2 | ~8 | Resolution of Q-4 changes the expected shape |
| **P0-009** | Dynamic filter/sort field allow-listing: unknown, malformed, and unauthorized field names | API | R-012 | 1 | ~6 | Injection surface created by AD-11 |
| **P0-010** | Shared link: S3/S7/S13 rejected server-side under any payload; read-only under every config; expiry; revocation; uniform errors; access recorded | API | R-010 | ~3 | ~10 | Only unauthenticated data path in the system |
| **P0-011** | AD-8 fail-safe: confirmed+fresh grants, confirmed+stale denies, unconfirmed denies, display degrades independently of authorization | Unit + Integration | R-006 | 2 | ~6 | Blocked on Q-3 for the window value; branch logic testable now |
| **P0-012** | Per-request re-resolution: mid-session relationship change and project-assignment end both take effect immediately | API | R-005 | 1 | ~4 | NFR-8 evidence |
| **P0-013** | Authentication required on every protected route incl. shared-link consumption; `userId` never accepted from the client | API + static | R-016, ASR-12 | 2 | ~5 | Static half asserts no handler reads a client-supplied user id |
| **P0-014** | All Employees list under 2s at 500+ records with 3 filters incl. one derived and one custom, permission resolution included | k6 | R-004 | 1 | 2 | Scenario shape blocked on Q-1 |
| **P0-015** | Self denied S6 and S15 entirely, across profile, dashboards, exports, and notifications | API + E2E | R-001 | 1 | ~4 | Highest-consequence single denial in the matrix |

**Total P0:** ~30 groups / ~200 cases.

The case count is high and the group count is not, because P0-001 and P0-003 are table-driven. If
TC-1 is not resolved, this inverts: those ~114 cases become hand-written and P0 roughly doubles in
effort while getting less trustworthy. That trade is the single largest cost lever in this plan.

## P1 — High

| Test ID | Requirement | Test Level | Risk Link | Groups | Notes |
| --- | --- | --- | --- | --- | --- |
| **P1-001** | Graph edge cases: cycle rejection, self-is-not-manager, ended assignment revokes, multi-leg union | Unit | R-001 | ~2 | Negative cases the happy path hides |
| **P1-002** | Functional-role permissions grant/revoke immediately; a role never widens *data* access | API | R-001 | ~2 | Guards the role/relationship separation |
| **P1-003** | Registry: duplicate section key fails at bootstrap; unregistered section yields explicit unavailable, distinct from not-granted | Integration | R-013 | ~2 | Blocked on Q-9 for the wire shape |
| **P1-004** | Prisma extension: history write emits exactly one timeline event; no bypass path; system-write skip on manual conflict | Integration | R-008 | ~3 | The coupling AD-4 makes implicit |
| **P1-005** | FR-30 manual-vs-system conflict, both orderings | Integration | R-008 | ~2 | |
| **P1-006** | Overdue derivation identical across profile, self-service, dashboard, and campaign table; never overdue once complete or cancelled | Unit + API | R-015 | ~2 | Needs the injectable clock (TC-4) |
| **P1-007** | Campaign activation: all items or a fully-editable draft; audience frozen at activation; no double activation | Integration | R-014 | ~3 | |
| **P1-008** | External-source faults: unreachable, slow, malformed, partial — for timetracker feeds and PeopleForce | Integration | NFR-4 | ~3 | Blocked on Q-5 for thresholds |
| **P1-009** | Custom-field lifecycle with no migration: define → filter, column, export | API | R-009 | ~2 | |
| **P1-010** | Export .xlsx generated server-side from the access-resolved query; no truncation at 500+ | API | R-001, R-004 | ~2 | File-content assertion, not just 200 |
| **P1-011** | Saved view re-executes per viewer and is never a data snapshot | API | R-001 | 1 | Two viewers, same view, different rows |
| **P1-012** | AD-1 module boundaries and AD-10 generated-type drift enforced in CI | Static | R-003 | 2 | Blocked on R-003 |
| **P1-013** | Action items: assignee-only completion, author-only cancellation with reason, terminal states final, authorship survives access loss | API | R-001 | ~3 | Authorship outliving access is the non-obvious case |
| **P1-014** | Mentorship: derived status never directly settable; ending requires closing feedback; both writes atomic | API | R-018 | ~2 | |
| **P1-015** | WCAG 2.1 AA automated scan plus keyboard and screen-reader passes on List, Profile, Dashboards | E2E + axe | NFR-3 | ~3 | Automated scan is necessary, not sufficient |
| **P1-016** | Seed tool produces 500+ records, no real-looking PII, regenerable | Unit + CI scan | R-017 | ~2 | NFR-5 evidence |

**Total P1:** ~36 groups / ~120 cases.

## P2 — Medium

| Test ID | Requirement | Test Level | Risk Link | Groups |
| --- | --- | --- | --- | --- |
| **P2-001** | Risk trend computed server-side; no indicator on a first record; identical value on all three surfaces | Unit + API | R-015 | ~2 |
| **P2-002** | Dashboard scoping per UM/DM/PM/PP; DM project selector recalculates counters; PP sees no resourcing block | API + E2E | R-001 | ~3 |
| **P2-003** | Resourcing: unattached request valid, rejection requires a reason, approval creates no assignment | API | — | ~2 |
| **P2-004** | CDS: dictionary-link resolution, append-only assessment history, IDP self-completion with no reopen | API | — | ~3 |
| **P2-005** | Self-service writes scoped to photo and certificates only; S4 exposes no write route | API | R-001 | ~2 |
| **P2-006** | Feedback period comparison with an empty period renders an explicit empty column | API + Component | — | ~2 |
| **P2-007** | Responsive collapse at `sm`, profile accordion, sidebar sheet | E2E | NFR-3 | ~2 |
| **P2-008** | Error envelope shape consistent from the global filter across handler, guard, and pipe failures | API | R-013 | ~2 |
| **P2-009** | Assessment recency filters, with "never assessed" as a distinct option from "assessed long ago" | API | — | ~2 |

**Total P2:** ~20 groups / ~45 cases.

## P3 — Low

| Test ID | Requirement | Test Level | Notes |
| --- | --- | --- | --- |
| **P3-001** | Latency baselines for profile assembly, dashboard aggregate, and export at 500+ | k6 | Benchmark only — no threshold exists (Q-2). Record, do not gate |
| **P3-002** | Soak on the list endpoint | k6 | Catches per-request resolution cost accumulating |
| **P3-003** | Exploratory session on shared-link token enumeration and guessability | Manual | R-016; pairs with P0-010 |
| **P3-004** | No hardcoded UI strings outside the i18n layer | Static | Cheap, prevents a slow leak |

**Total P3:** ~5 groups.

## Coverage Totals

| Priority | Groups | Cases | Share of cases |
| --- | --- | --- | --- |
| P0 | ~30 | ~200 | ~55% |
| P1 | ~36 | ~120 | ~33% |
| P2 | ~20 | ~45 | ~12% |
| P3 | ~5 | — | — |
| **Total** | **~91** | **~365** | |

P0 is ~33% of groups, well above the ~10% rule of thumb in `test-priorities-matrix.md`. That is a
real deviation and it is deliberate: the product's stated top success metric (SM-1) and its primary
risk are the same thing — an access model with 96 declared cells and a hard zero-leak bar. Lowering
those cells out of P0 to satisfy a ratio would be misrepresenting the risk. The ratio guidance
assumes P0 is a small critical core inside a larger feature body; here the critical core *is* the
product's differentiator.

## NFR Coverage and Evidence Plan

| NFR | Threshold | Planned validation | Level / tool | Evidence artifact | Priority |
| --- | --- | --- | --- | --- | --- |
| NFR-1 / SM-1 — zero unauthorized exposure | Binary: every `—` cell denied on every surface | P0-001 through P0-010, P0-015 | Jest unit + Supertest + Playwright | Matrix suite report; per-surface negative results | P0 |
| NFR-2 / SM-4 — list < 2s at 500+ with 3 filters | 2s, concurrency UNKNOWN (Q-1) | P0-014 | k6 | Load report with the permission-resolution segment attributed | P0 |
| NFR-3 — WCAG 2.1 AA | AA on primary surfaces | P1-015 | Playwright + axe | Scan report plus a manual keyboard/SR checklist | P1 |
| NFR-4 — external-source resilience | No threshold defined (Q-5) | P1-008 | Jest integration, faults injected at the client boundary | Fault-injection results | P1 |
| NFR-5 — pseudonymized data | Prohibition: no real PII outside production | P1-016 | Jest + CI scan | Seed assertions plus scan output | P1 |
| NFR-8 — per-request evaluation | No cross-request grant persistence | P0-006, P0-012 | Supertest | Change-visibility test results | P0 |
| NFR-6, NFR-7 — parallel delivery, repo fidelity | Process metrics | Delivery review at demo | Not a test tier | Review notes | — |

**Missing thresholds carried forward:** Q-1 (NFR-2 concurrency), Q-2 (other endpoint targets), Q-3
(AD-8 freshness window), Q-5 (external-client timeout and retry policy). Each is recorded as a
clarification item, not filled with a guessed number. P0-014 and P1-008 can be *built* now and
*gated* once the values land.

**Assessment boundary:** this is the evidence plan. PASS / CONCERNS / FAIL belongs to `nfr-assess`
once implementation evidence exists.

## Execution Strategy

Philosophy: everything that fits in a PR runs in a PR. Only genuinely expensive or externally
dependent work defers.

### Every PR (target under 15 minutes)

- Jest unit suites, including the full access matrix harness. No database, so this is the fast
  majority of the case count and it is where a broken access rule should surface first.
- Jest integration and API suites against an ephemeral Postgres, sharded across workers. **Conditional
  on R-002 being resolved** — until then these cannot safely run in parallel, and running them
  serially is the fallback that keeps the PR under budget only while the suite is small.
- Playwright E2E: critical journeys and per-surface leak checks only.
- Static gates: dependency-cruiser (AD-1), generated-type diff (AD-10), `.only` detection,
  PII/secret scan.

### Nightly (30–60 minutes)

- k6 NFR-2 load scenario (P0-014) — needs a provisioned 500+ record dataset, too slow for a PR.
- Full-page axe sweep across all routes; the PR run covers the three primary surfaces only.
- Seed regeneration at 500+ records, to catch drift in the data generator itself.

### Weekly (hours)

- Integration run against the real timetracker and PeopleForce instances. Deliberately not in PRs:
  external rate limits and availability would make PR results non-deterministic, which is precisely
  the failure `test-quality.md` warns about. PR runs use stubs at the client boundary; the weekly run
  is what proves the stubs still resemble reality.
- Soak (P3-002) and the exploratory shared-link session (P3-003).

**Manual, excluded from automation:** screen-reader passes, delivery-process review (NFR-6, NFR-7),
and demo-time doc fidelity (SM-5).

## Resource Estimates

Test authoring, excluding the foundation infrastructure below.

| Priority | Groups | Effort |
| --- | --- | --- |
| P0 | ~30 | ~50–80 hours |
| P1 | ~36 | ~45–70 hours |
| P2 | ~20 | ~20–35 hours |
| P3 | ~5 | ~6–12 hours |
| **Total** | **~91** | **~120–200 hours** |

That is roughly **3–5 engineer-weeks** full-time. There is no dedicated QA on this team, so it
distributes across the three developers alongside feature work rather than compressing into a block.

**Foundation infrastructure — separate, and a prerequisite: ~40–70 hours.** This is not test
authoring and should not be estimated as if it were:

| Item | Concern | Effort |
| --- | --- | --- |
| Machine-readable access matrix + generator | TC-1 | ~8–16 h |
| Relationship-graph test factory | TC-3 | ~8–14 h |
| Parallel-safe DB isolation | TC-2 / R-002 | ~8–16 h |
| Injectable clock | TC-4 | ~3–6 h |
| External-client stubbing at the boundary | TC-5 | ~5–10 h |
| CI pipeline with the five static gates | TC-7 / R-003 | ~8–16 h |

The ranges widen where a decision is still open. DB isolation depends on the mechanism chosen; the
matrix generator depends on how much of `access-model.md` survives translation without ambiguity.

**Excluded:** ongoing maintenance (assume ~10%), and any effort owned by Backend or DevOps for the
production-code mitigations in the risk register.

## Quality Gates

Release gates, in the order they should be evaluated:

1. **P0 pass rate: 100%.** No exceptions. A failing P0 here is a data-exposure or auth failure.
2. **Every `—` cell in the matrix has a passing negative test.** Binary, not a percentage — SM-1 is
   stated as zero incidents, so a 95% negative-coverage figure would mean roughly one unguarded cell.
3. **P1 pass rate ≥ 95%,** with every failure triaged and explicitly accepted.
4. **No risk scoring 9 left open.** Currently R-001, R-002, R-003 — all three are foundation-phase
   and should clear before feature stories begin, not before release.
5. **Coverage ≥ 80% overall backend; ≥ 90% branch coverage on the `access` module.** The higher bar on
   `access` is where branch coverage actually correlates with the risk.
6. **All five CI-enforced invariants active** (module boundaries, type diff, matrix suite, focus
   detection, PII scan). An invariant enforced by convention is not enforced.
7. **NFR evidence artifact identified for every in-scope category.** Identified, not yet passing —
   the verdict is `nfr-assess`'s.

**Foundation-phase gate (before feature stories):** items 4, 6, and the TC-1/TC-2/TC-3 infrastructure.
PRD §1 already positions the foundation phase as a gate; this attaches specific criteria to it.

---

# Step 5 Output — Generation & Validation

## Execution Mode

`tea_execution_mode: auto` with `tea_capability_probe: true`. Subagent launching is available, so the
resolved mode is **subagent**. The step permits ("can") parallel generation of the two system-level
documents; I authored them sequentially instead. The reason is specific: both derive from one risk
register and one coverage matrix held in this session, cross-document consistency of risk IDs,
priorities, blockers, dates, and counts is an explicit validation requirement, and reconciling two
independently-written documents costs more than writing them consistently once.

## Outputs

| Document | Path | Lines |
| --- | --- | --- |
| Architecture concerns | `_bmad-output/test-artifacts/test-design-architecture.md` | ~430 |
| QA test plan | `_bmad-output/test-artifacts/test-design-qa.md` | ~541 |
| BMAD handoff | `_bmad-output/test-artifacts/test-design/people-management-handoff.md` | ~174 |
| Working notes | `_bmad-output/test-artifacts/test-design-progress-system.md` | this file |

## Checklist Validation

Passing: prerequisites (PRD, spine, epics, access model all present and read); risk assessment (18
risks, unique IDs, 1–3 scales, scores verified as P×I, ≥6 flagged, owners and timelines assigned);
NFR planning (categories identified, thresholds extracted, unknowns marked rather than guessed,
converted into Q-1…Q-10, evidence sources named, NFR risks folded into the main register); coverage
design (atomic scenarios, levels selected, priorities assigned, declared overlap justified); execution
strategy (simple PR/nightly/weekly, no re-listing, philosophy stated); estimates (all ranges, no false
precision); document structure (actionable-first in the architecture doc, recipe-focused in the QA
doc, no test code in the architecture doc, no quality-gate section in the QA doc).

### Deviations, with reasons

1. **Architecture doc is ~430 lines against a ~150–200 target.** The target presumes a handful of
   high-priority risks; this system has 12 scoring ≥6, and the checklist separately requires a
   mitigation plan with strategy, owner, timeline, status, and verification for each. Compressing to
   200 would mean dropping required content. Trimmed what was genuinely repeated — the per-plan
   "Status: Planned" boilerplate, the triple-stated high-priority recommendations, an empty
   low-priority subsection — and left the rest.
2. **P0 is ~33% of test groups, against a "<10% of scenarios" guideline.** The product's top success
   metric and its top risk are the same 96-cell access model with a zero-leak bar. The guideline
   assumes P0 is a small critical core inside a larger feature body; here that core is the product.
   Demoting cells to hit the ratio would misstate the risk.
3. **QA doc examples use vanilla Jest/Supertest/Playwright, not the configured utils packages.**
   `tea_use_playwright_utils` and `tea_use_pactjs_utils` are `true`, but neither package is installed
   in either service. Per the two-gate rule in `library-integration-mandate.md`, a flag alone does not
   bind; scaffolding imports against absent packages would produce code that cannot run. Stated once
   in the QA doc with a pointer to `bmad-testarch-framework`.
4. **Pact-based contract testing is out of scope.** AD-10 already generates frontend types from
   Swagger and CI-diffs them. With one consumer, one provider, and a single deploy unit, a broker adds
   ceremony without covering a failure mode the type diff misses. `tea_pact_mcp` was also unreachable,
   so no broker state was inferred.
5. **Quality gates live in the QA doc's Exit Criteria** rather than a section titled "Quality Gate
   Criteria", which the anti-bloat checklist prohibits in both documents while step 4 requires the
   thresholds. Exit Criteria carries them.
6. **R-011's mitigation is not in the architecture doc's mitigation plans.** It is QA-owned (a test-tier
   decision, not a production-code change), and the checklist directs QA-owned mitigations to the QA
   doc. It appears in the architecture doc's Architectural Improvements as TC-6.

## Open Assumptions Carried Forward

Ten clarification items (Q-1…Q-10) remain unresolved. Six change what a story's acceptance criteria
say — Q-1, Q-3, Q-4, Q-5, Q-8, Q-9 — and are flagged in the handoff for epic refinement. No threshold
was invented to close a gap; where a value is missing, the test is designed to be built now and gated
later with the threshold externalized as configuration.

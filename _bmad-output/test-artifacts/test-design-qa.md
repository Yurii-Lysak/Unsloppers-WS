---
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
workflowType: 'testarch-test-design'
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - _bmad-output/planning-artifacts/epics.md
  - _bmad-output/specs/spec-people-management-platform/access-model.md
---

# Test Design for QA: People Management Platform

**Purpose:** Test execution recipe. What to test, at which level, in what priority, and what is
needed from other teams first.

**Date:** 2026-08-25
**Author:** Unslopper (TEA Master Test Architect)
**Status:** Draft
**Project:** people management

**Related:** `test-design-architecture.md` holds the testability concerns, architectural blockers, and
full risk detail. Risk IDs and blocker references are shared between the two documents.

---

## Executive Summary

**Scope:** 13 epics, 61 stories, 57 functional and 8 non-functional requirements. The dominant
testing problem is a 16-section × 6-audience access model with a zero-leak bar (SM-1).

**Risk summary:**

- 18 risks — 12 scoring ≥6, 5 scoring 4, 1 lower.
- Concentrated in SEC (8 of 18) and in the foundation phase rather than in features.

**Coverage summary:**

Two units are used deliberately. A **group** is one authored, maintained artifact; a **case** is one
assertion instance. A parameterized group over the access matrix yields ~96 cases from one harness.
Effort tracks groups; confidence tracks cases.

| Priority | Groups | Cases |
| --- | --- | --- |
| P0 | ~30 | ~200 |
| P1 | ~36 | ~120 |
| P2 | ~20 | ~45 |
| P3 | ~5 | — |
| **Total** | **~91** | **~365** |

**Total effort:** ~120–200 hours of test authoring (~3–5 engineer-weeks), plus ~40–70 hours of test
infrastructure that must land first.

---

## Not in Scope

| Item | Reasoning | Mitigation |
| --- | --- | --- |
| **Contract testing (Pact)** | AD-10 already generates frontend types from Swagger and CI-diffs them. With one consumer, one provider, and a single deploy unit, a broker adds ceremony without adding a failure mode the type diff misses | The AD-10 diff gate (P1-012) |
| **Writes to timetracker or PeopleForce** | Both are read-only sources by design | Fault injection on the read path (P1-008) |
| **Blue/green, rollback, and failover rehearsal** | AD-12 defines a single containerized deploy with no staging tier | Accepted trade-off, recorded in the architecture doc |
| **Distributed tracing and APM assertions** | No observability stack in scope; structured logs only | Log-content assertions where a requirement names logging (P0-010) |
| **Cross-browser matrix** | Single internal audience; Chromium-only is proportionate | Accessibility and responsive checks still run (P1-015, P2-007) |
| **Load testing beyond the All Employees list** | NFR-2 sets a threshold only for that endpoint; no other target exists (Q-2) | Benchmarks recorded without a gate (P3-001) |
| **Penetration testing** | Out of scope for this milestone | Targeted negative tests on the auth and shared-link surfaces (P0-010, P0-013) |

Exclusions need confirmation from Dev and PM; they are proposed here, not settled.

---

## Dependencies & Test Blockers

### From Architecture and Backend — pre-implementation

Detail and mitigation plans are in the architecture doc's Quick Guide.

1. **Machine-readable access matrix (TC-1)** — Architect + Backend — foundation
   - A structured matrix that suites can import, covering all 16 sections, 6 audiences, field-level
     exceptions, and record-level flags.
   - Blocks P0-001 and P0-003 entirely — roughly 114 of the ~200 P0 cases. Without it those cases are
     hand-written, which roughly doubles P0 effort and loses the drift detection Story 1.14 asks for.

2. **Parallel-safe database isolation (TC-2 / R-002)** — Backend — foundation
   - A decided mechanism plus per-worker setup in the shared test harness.
   - Blocks every integration and API suite from running in parallel. Serial execution is a temporary
     fallback that only holds while the suite is small.

3. **Relationship-graph test factory (TC-3)** — Backend + QA — foundation
   - Build an exact graph per test: reports-to at arbitrary depth, a project with a PM and a DM, a PP
     assignment with the HR line above it, and an ended assignment.
   - Blocks P0-002 and P1-001. The seed tool gives volume; these need precision.

4. **Injectable clock (TC-4)** — Backend — foundation
   - Blocks deterministic testing of the AD-8 freshness window, shared-link expiry, overdue
     derivation, and assessment recency (P0-011, P1-006).

5. **Substitutable external client boundary (TC-5)** — Backend — foundation
   - Blocks NFR-4 fault injection (P1-008) entirely — there is currently no seam to inject at.

6. **CI pipeline (R-003)** — Team, Story 1.17 — foundation
   - Blocks P1-012 and every invariant gate. Also determines whether retries mask flakiness (TC-7).

7. **Access log and unavailable-state contract shapes (TC-8)** — Architect — pre-implementation
   - Blocks the log assertions in P0-010 and the distinguishability assertion in P1-003.

### QA infrastructure setup

1. **Test data factories** — faker-based employee, project, assignment, and mentorship builders with
   auto-cleanup fixtures for parallel safety. No real names, emails, or identifiers (NFR-5).
2. **Access matrix fixture** — a thin adapter over the machine-readable matrix, shared by unit and API
   suites so both assert the same source.
3. **Environments** — local: `npm run db:up` plus the Nest app; CI: ephemeral Postgres service
   container; there is no staging tier (AD-12), so the load dataset lives in the single environment and
   runs must be scheduled around demos (TC-9).

### A note on tooling

`tea_use_playwright_utils` and `tea_use_pactjs_utils` are both `true` in `_bmad/tea/config.yaml`, but
neither `@seontechnologies/playwright-utils` nor `@pactflow/pact-*` is present in either service's
`package.json`. Per the library integration mandate, a flag alone does not bind — the package must
also be installed. So every example below targets the stack that actually exists: Jest 30 with
Supertest on the backend, and vanilla Playwright on the frontend. If the team wants the utils
patterns, run `bmad-testarch-framework` first; this plan does not assume them.

---

## Risk Assessment

Full detail in the architecture doc. This is the QA-facing view: how each risk gets validated.

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Score | QA Test Coverage |
| --- | --- | --- | --- | --- |
| **R-001** | SEC | Leak through a non-profile surface | **9** | P0-001, P0-003, P0-004, P0-005, P0-007, P0-015, P1-010, P1-011 |
| **R-002** | TECH | Shared-DB parallelism false greens | **9** | Not a test — an infrastructure blocker. Verified by an order-dependent pair that must pass either way |
| **R-003** | OPS | No CI, so invariants are advisory | **9** | Not a test — verified by a PR that violates each gate and is rejected |
| **R-004** | PERF | Resolution cost breaches the 2s budget | 6 | P0-014, P3-002 |
| **R-005** | SEC | Stale grant from a TTL cache | 6 | P0-006, P0-012 |
| **R-006** | SEC | Stale assignment keeps granting access | 6 | P0-011 |
| **R-007** | TECH | Suites drift from the matrix | 6 | P0-001 fails loudly on an unmapped cell |
| **R-008** | DATA | Timeline diverges from history | 6 | P1-004, P1-005 |
| **R-009** | SEC | Custom-field existence inferable | 6 | P0-008 |
| **R-010** | SEC | Shared-link disclosure or unlogged access | 6 | P0-010 |
| **R-011** | TECH | Negative cases land in no tier | 6 | Resolved structurally: response-body assertions live in the backend API tier |
| **R-012** | SEC | Unvalidated dynamic filter or sort | 6 | P0-009 |
| **R-013** | DATA | Unavailable indistinguishable from denied | 6 | P1-003 |
| **R-015** | OPS | Wall-clock time dependence | 6 | P1-006, plus the clock used throughout P0-011 |

### Medium and Lower Risks

| Risk ID | Category | Description | Score | QA Test Coverage |
| --- | --- | --- | --- | --- |
| R-014 | BUS | Partial campaign activation | 4 | P1-007 |
| R-016 | SEC | No rate limiting on link or login | 4 | P0-013, P3-003 |
| R-017 | OPS | Perf and seed share the demo environment | 4 | P1-016; scheduling rather than a test |
| R-018 | TECH | Mentorship ended without closing feedback | 4 | P1-014 |

---

## Test Coverage Plan

**P0/P1/P2/P3 are priority and risk level** — what to focus on when time-constrained. They are not
execution timing; see Execution Strategy for when things run.

**Test level rationale.** Unit covers access decisions and derivations — pure functions of a
relationship graph and a matrix, and the cheapest place to get combinatorial coverage. API covers
emission: the unit layer proves the resolver denies, but only a response-body assertion proves the
field never reached the wire. Integration covers database-state invariants, where the Prisma extension
is itself the mechanism under test. E2E stays thin, reserved for critical journeys and accessibility.

**Declared overlap.** Access rules are asserted at unit level (decision) and API level (emission).
Not redundancy — the layers can disagree, and that disagreement is the bug class SM-1 exists to
prevent. The overlap is bounded: unit covers all 96 cells, API covers the ~18 denial cells plus one
representative cell per emission surface.

### P0 (Critical)

**Criteria:** security, data-integrity, or compliance impact with no safe workaround.

| Test ID | Requirement | Test Level | Risk Link | Groups | Cases | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| **P0-001** | Access matrix, 16 sections × 6 audiences, driven off the structured matrix | Unit | R-001, R-007 | 1 | ~96 | Blocked on TC-1. Must fail on an unmapped cell — a new section cannot default to allowed |
| **P0-002** | Relationship-path resolution: direct, transitive, project PM, project DM, PP, union of legs | Unit | R-001, R-007 | 1 | ~12 | Needs the graph factory |
| **P0-003** | Denial cells asserted on the wire — key absent, not null | API | R-001 | ~4 | ~18 | Absence, not falsiness |
| **P0-004** | Colleague whitelist holds on every emission surface: profile, list rows, export, saved-view re-execution, campaign audience preview, dashboard aggregate, shared link, search | API + E2E | R-001 | ~8 | ~16 | One per surface; the surface list is itself the deliverable |
| **P0-005** | S7 record-level filtering: employee flag × PM-visible flag × unconditional UM/DM/PP | Unit + API | R-001 | 2 | ~8 | Two independent flags on one section — the densest rule in the matrix |
| **P0-006** | S8 `sharedWithEmployee` flip changes Self visibility on the next request | API | R-001, R-005 | 1 | 2 | Also evidences the no-caching claim |
| **P0-007** | S1 mentor-field override denies Colleague while the section stays readable | Unit + API | R-001 | 1 | 2 | Field-level exception inside a granted section |
| **P0-008** | S16 custom fields absent from profile, columns, filters, and export per visibility level; no existence inference via counts or error shapes | API | R-009, R-012 | 2 | ~8 | Expected shape depends on Q-4 |
| **P0-009** | Dynamic filter and sort allow-listing: unknown, malformed, and unauthorized field names | API | R-012 | 1 | ~6 | The injection surface AD-11 creates |
| **P0-010** | Shared link: S3/S7/S13 rejected server-side under any payload; read-only under every config; expiry; revocation; uniform errors; access recorded | API | R-010 | ~3 | ~10 | The only unauthenticated data path |
| **P0-011** | AD-8 fail-safe: confirmed and fresh grants; confirmed and stale denies; unconfirmed denies; display degrades independently of authorization | Unit + Integration | R-006 | 2 | ~6 | Branch logic testable now; the window value needs Q-3 |
| **P0-012** | Per-request re-resolution: a mid-session relationship change and an ended project assignment both take effect immediately | API | R-005 | 1 | ~4 | NFR-8 evidence |
| **P0-013** | Authentication required on every protected route including shared-link consumption; `userId` never accepted from the client | API + static | R-016, ASR-12 | 2 | ~5 | The static half asserts no handler reads a client-supplied user id |
| **P0-014** | All Employees list under 2s at 500+ records with 3 filters including one derived and one custom, permission resolution included | k6 | R-004 | 1 | 2 | Scenario shape blocked on Q-1 |
| **P0-015** | Self denied S6 and S15 entirely — profile, dashboards, exports, notifications | API + E2E | R-001 | 1 | ~4 | Highest-consequence single denial in the matrix |

**Total P0:** ~30 groups / ~200 cases.

P0 is ~33% of groups, well above the ~10% rule of thumb. That is deliberate: the product's top success
metric and its top risk are the same thing — a 96-cell access model with a zero-leak bar. Demoting
cells out of P0 to hit a ratio would misrepresent the risk. The ratio assumes P0 is a small critical
core inside a larger feature body; here the critical core is the product's differentiator.

### P1 (High)

**Criteria:** core or complex behavior with material reach and limited workaround.

| Test ID | Requirement | Test Level | Risk Link | Groups | Notes |
| --- | --- | --- | --- | --- | --- |
| **P1-001** | Graph edge cases: cycle rejection, self-is-not-manager, ended assignment revokes, multi-leg union | Unit | R-001 | ~2 | Negative cases the happy path hides |
| **P1-002** | Functional-role permissions grant and revoke immediately; a role never widens data access | API | R-001 | ~2 | Guards the role/relationship separation |
| **P1-003** | Registry: duplicate section key fails at bootstrap; unregistered section yields explicit unavailable, distinct from not-granted | Integration | R-013 | ~2 | Wire shape blocked on Q-9 |
| **P1-004** | Prisma extension: a history write emits exactly one timeline event; no bypass path; system write skipped on manual conflict | Integration | R-008 | ~3 | The coupling AD-7 leaves implicit |
| **P1-005** | FR-30 manual-versus-system conflict, both orderings | Integration | R-008 | ~2 | |
| **P1-006** | Overdue derivation identical across profile, self-service, dashboard, and campaign table; never overdue once complete or cancelled | Unit + API | R-015 | ~2 | Needs the injectable clock |
| **P1-007** | Campaign activation: all items or a fully-editable draft; audience frozen at activation; no double activation | Integration | R-014 | ~3 | |
| **P1-008** | External-source faults — unreachable, slow, malformed, partial — for timetracker feeds and PeopleForce | Integration | NFR-4 | ~3 | Thresholds blocked on Q-5 |
| **P1-009** | Custom-field lifecycle with no migration: define, then filter, column, export | API | R-009 | ~2 | SM-2 evidence |
| **P1-010** | Export generated server-side from the access-resolved query; no truncation at 500+ | API | R-001, R-004 | ~2 | Assert file contents, not just a 200 |
| **P1-011** | A saved view re-executes per viewer and is never a data snapshot | API | R-001 | 1 | Two viewers, one view, different rows |
| **P1-012** | AD-1 module boundaries and AD-10 generated-type drift enforced in CI | Static | R-003 | 2 | Blocked on R-003 |
| **P1-013** | Action items: assignee-only completion, author-only cancellation with reason, terminal states final, authorship survives access loss | API | R-001 | ~3 | Authorship outliving access is the non-obvious case |
| **P1-014** | Mentorship: derived status never directly settable; ending requires closing feedback; both writes atomic | API | R-018 | ~2 | |
| **P1-015** | WCAG 2.1 AA automated scan plus keyboard and screen-reader passes on List, Profile, Dashboards | E2E + axe | NFR-3 | ~3 | The automated scan is necessary, not sufficient |
| **P1-016** | Seed tool produces 500+ records, no real-looking PII, regenerable | Unit + CI scan | R-017 | ~2 | NFR-5 evidence |

**Total P1:** ~36 groups / ~120 cases.

### P2 (Medium)

**Criteria:** secondary behavior, narrower reach, acceptable workaround.

| Test ID | Requirement | Test Level | Risk Link | Groups |
| --- | --- | --- | --- | --- |
| **P2-001** | Risk trend computed server-side; no indicator on a first record; identical value on all three surfaces | Unit + API | R-015 | ~2 |
| **P2-002** | Dashboard scoping per UM/DM/PM/PP; the DM project selector recalculates counters; PP sees no resourcing block | API + E2E | R-001 | ~3 |
| **P2-003** | Resourcing: an unattached request is valid, rejection requires a reason, approval creates no assignment | API | — | ~2 |
| **P2-004** | CDS: dictionary-link resolution, append-only assessment history, IDP self-completion with no reopen | API | — | ~3 |
| **P2-005** | Self-service writes scoped to photo and certificates only; S4 exposes no write route | API | R-001 | ~2 |
| **P2-006** | Feedback period comparison with an empty period renders an explicit empty column | API + Component | — | ~2 |
| **P2-007** | Responsive collapse at `sm`, profile accordion, sidebar sheet | E2E | NFR-3 | ~2 |
| **P2-008** | Error envelope shape consistent from the global filter across handler, guard, and pipe failures | API | R-013 | ~2 |
| **P2-009** | Assessment recency filters, with "never assessed" distinct from "assessed long ago" | API | — | ~2 |

**Total P2:** ~20 groups / ~45 cases.

### P3 (Low)

**Criteria:** rare, exploratory, or benchmark-only.

| Test ID | Requirement | Test Level | Notes |
| --- | --- | --- | --- |
| **P3-001** | Latency baselines for profile assembly, dashboard aggregate, and export at 500+ | k6 | Record only — no threshold exists (Q-2) |
| **P3-002** | Soak on the list endpoint | k6 | Catches per-request resolution cost accumulating |
| **P3-003** | Shared-link token enumeration and guessability | Manual | R-016; pairs with P0-010 |
| **P3-004** | No hardcoded UI strings outside the i18n layer | Static | Cheap, prevents a slow leak |

**Total P3:** ~5 groups.

---

## NFR Test Coverage Plan

Maps NFR requirements to planned validation and the evidence `nfr-assess` should later consume. No
PASS/CONCERNS/FAIL verdicts here.

| NFR Category | Requirement / Threshold | Planned Validation | Tool / Level | Evidence Artifact | Priority |
| --- | --- | --- | --- | --- | --- |
| Security (NFR-1 / SM-1) | Zero unauthorized exposure; every denial holds on every surface | P0-001 through P0-010, P0-015 | Jest unit + Supertest + Playwright | Matrix suite report; per-surface negative results | P0 |
| Performance (NFR-2 / SM-4) | List under 2s at 500+ records with 3 filters, resolution included | P0-014 | k6 | Load report with the resolution segment attributed | P0 |
| Reliability (NFR-4) | External sources degrade without taking the app down | P1-008 | Jest integration, faults at the client boundary | Fault-injection results | P1 |
| Accessibility (NFR-3 / SM-7) | WCAG 2.1 AA on List, Profile, Dashboards | P1-015 | Playwright + axe | Scan report plus a manual keyboard/SR checklist | P1 |
| Data protection (NFR-5) | No real PII outside production | P1-016 | Jest + CI scan | Seed assertions plus scan output | P1 |
| Per-request evaluation (NFR-8) | No grant persists across requests | P0-006, P0-012 | Supertest | Change-visibility results | P0 |
| Maintainability | Module boundaries and generated-type fidelity hold | P1-012 | dependency-cruiser + openapi diff in CI | CI gate output plus coverage report | P1 |
| Extensibility (SM-2) | A custom field is usable with no deploy or migration | P1-009 | Supertest | Lifecycle test result | P1 |

**Missing thresholds and evidence sources:** Q-1 (NFR-2 concurrency), Q-2 (targets for other
endpoints), Q-3 (AD-8 freshness window), Q-5 (external timeout and retry policy), Q-8 (access log
shape). P0-014 and P1-008 can be built now and gated when the values land — the threshold should be
configuration, not a literal.

NFR-6 (parallel delivery) and NFR-7 (repo fidelity) are process metrics reviewed at demo, not test
tiers.

---

## Execution Strategy

Philosophy: everything that fits in a PR runs in a PR. Only genuinely expensive or externally
dependent work defers. Jest with worker parallelism and Playwright with sharding both put hundreds of
tests inside a 10–15 minute budget.

### Every PR — target under 15 minutes

- **Jest unit suites**, including the full access matrix harness. No database, so this is most of the
  case count and the first place a broken access rule surfaces.
- **Jest integration and API suites** against an ephemeral Postgres, sharded across workers.
  Conditional on R-002: until isolation is decided, these must run serially, which stays inside budget
  only while the suite is small.
- **Playwright E2E**: critical journeys and per-surface leak checks only.
- **Static gates**: dependency-cruiser (AD-1), generated-type diff (AD-10), focus-mode detection, PII
  and secret scan.

Retries stay off for the access suites. A test that passes on the third attempt is not evidence of a
denial holding, and retry-laundered green is indistinguishable from a rare leak (TC-7).

### Nightly — 30–60 minutes

- **k6 NFR-2 load scenario** (P0-014) — needs the provisioned 500+ record dataset.
- **Full-page axe sweep** across all routes; the PR run covers the three primary surfaces only.
- **Seed regeneration** at 500+ records, to catch drift in the generator itself.

### Weekly — hours

- **Integration against the real timetracker and PeopleForce.** Deliberately not in PRs: external rate
  limits and availability would make PR results non-deterministic. PR runs use boundary stubs; this
  run is what proves the stubs still resemble reality.
- **Soak** (P3-002) and the **exploratory shared-link session** (P3-003).

**Manual, excluded from automation:** screen-reader passes, delivery-process review (NFR-6, NFR-7),
and doc fidelity at demo (SM-5).

---

## Entry Criteria

Test development cannot begin until all of the following hold:

- [ ] The machine-readable access matrix exists and is importable (TC-1)
- [ ] Database isolation is decided and implemented in the shared harness (TC-2 / R-002)
- [ ] The relationship-graph factory is available (TC-3)
- [ ] An injectable clock is wired wherever "now" is read (TC-4)
- [ ] The external client boundary is substitutable (TC-5)
- [ ] CI exists with the five invariant gates active (R-003)
- [ ] Q-4, Q-8, and Q-9 are answered — expected response shapes depend on them
- [ ] Requirements, exclusions, and assumptions agreed by Dev and PM

## Exit Criteria

The testing phase is complete when all of the following hold:

- [ ] **P0 pass rate 100%** — no exceptions; a failing P0 here is an exposure or auth failure
- [ ] **Every denial cell in the matrix has a passing negative test** — binary, not a percentage. SM-1
      is stated as zero incidents, so 95% negative coverage would mean roughly one unguarded cell
- [ ] **P1 pass rate ≥95%**, with every failure triaged and explicitly accepted
- [ ] **No score-9 risk open** — R-001, R-002, R-003 clear before feature stories, not before release
- [ ] **Coverage ≥80% backend overall, ≥90% branch coverage on the `access` module** — the higher bar
      is where branch coverage actually tracks the risk
- [ ] All five CI-enforced invariants active; an invariant enforced by convention is not enforced
- [ ] An evidence artifact identified for every in-scope NFR category — identified, not yet passing;
      the verdict belongs to `nfr-assess`
- [ ] No open high-severity defects
- [ ] No committed focus-mode tests

---

## Implementation Planning Handoff

| Work Item | Owner | Target Milestone | Dependencies / Notes |
| --- | --- | --- | --- |
| Machine-readable access matrix + generator | Architect + Backend | Foundation | ~8–16 h. Gates ~114 P0 cases |
| Relationship-graph test factory | Backend + QA | Foundation | ~8–14 h. Depends on the schema being settled |
| Parallel-safe DB isolation | Backend | Foundation | ~8–16 h. Mechanism choice drives the range |
| Injectable clock | Backend | Foundation | ~3–6 h |
| External-client stubbing seam | Backend | Foundation | ~5–10 h |
| CI pipeline with five static gates | Team | Foundation (Story 1.17) | ~8–16 h. Blocked on Q-6 |
| P0 access suites | Dev (shared) | Foundation → Wave 1 | Blocked on the first three items above |
| k6 harness and 500-record dataset | Dev | Wave 1 | Blocked on Q-1 and Q-7 |
| axe integration | Dev | Wave 2 | Independent of the access work |

---

## QA Effort Estimate

Test authoring only. The infrastructure above is a separate ~40–70 hours and is a prerequisite, not
part of this line.

| Priority | Groups | Effort Range | Notes |
| --- | --- | --- | --- |
| P0 | ~30 | ~50–80 hours | Matrix harness, per-surface negatives, k6 scenario |
| P1 | ~36 | ~45–70 hours | Integration invariants, fault injection, accessibility |
| P2 | ~20 | ~20–35 hours | Secondary flows, edge cases |
| P3 | ~5 | ~6–12 hours | Benchmarks, exploratory, static |
| **Total** | **~91** | **~120–200 hours** | **~3–5 engineer-weeks full-time** |

**Assumptions:**

- Includes design, implementation, debugging, and CI wiring.
- Excludes ongoing maintenance (assume ~10%).
- Assumes the entry-criteria infrastructure is in place. If TC-1 is not resolved, P0 roughly doubles
  and loses its drift detection — the single largest cost lever in this plan.
- There is no dedicated QA. This distributes across three developers alongside feature work rather
  than compressing into a block, so calendar time exceeds the engineer-week figure.

---

## Tooling & Access

| Tool or Service | Purpose | Access Required | Status |
| --- | --- | --- | --- |
| k6 | NFR-2 load scenario | Local binary; CI runner capacity | Pending |
| axe-core / `@axe-core/playwright` | WCAG 2.1 AA scanning | npm dependency | Pending |
| dependency-cruiser | AD-1 module boundary enforcement | npm dependency | Pending |
| `openapi-typescript` diff gate | AD-10 generated-type fidelity | Already in the frontend toolchain | Ready |
| CI platform | All invariant gates | Platform decision plus repo permissions | Blocked on Q-6 |
| Timetracker and PeopleForce sandboxes | Weekly real-integration run | Credentials and rate-limit headroom | Pending |

**Access requests needed:**

- [ ] CI platform and repository permissions (Q-6)
- [ ] Timetracker and PeopleForce non-production credentials
- [ ] Confirmation of a demo-safe window for load runs (TC-9)

---

## Interworking & Regression

| Service / Component | Impact | Regression Scope | Validation Steps |
| --- | --- | --- | --- |
| **`services/backend` access layer** | Every feature reads through it | All access suites | The full matrix harness must pass on any change touching resolution |
| **Prisma schema and extension** | History and timeline coupling | P1-004, P1-005 | Re-run extension integration tests on any schema change |
| **Generated frontend API types** | Every frontend data path | AD-10 diff gate | Regenerate and diff; a drift is a build failure |
| **Timetracker integration** | Absence, project, and assignment data | P1-008 plus the weekly real run | Fault-injection suite plus one live run per week |
| **PeopleForce integration** | Employee source data | P1-008 | Fault-injection suite |
| **Shared component library** | All UI surfaces | P1-015, P2-007 | Accessibility and responsive checks on the three primary surfaces |

**Regression strategy:** the access matrix harness is the regression gate for this system. Any change
to resolution, to a section provider, or to an emission surface re-runs it in full — it is cheap
(unit level, no database) and it is the suite whose failure matters most. Everything else regresses on
the normal PR tier.

---

## Appendix A: Code Examples & Tagging

Examples target the installed stack: Jest 30 with Supertest on the backend, vanilla Playwright on the
frontend. See the tooling note above for why the configured utils packages are not assumed.

**The matrix harness — one authored group, ~96 cases.** The shape matters more than the syntax: the
cases come from the matrix, so a matrix change extends coverage automatically and an unmapped cell
fails rather than silently passing.

```typescript
import { ACCESS_MATRIX, SECTIONS, AUDIENCES } from '../fixtures/access-matrix';
import { resolveSectionAccess } from '../../src/access/access-resolver';
import { buildGraph } from '../factories/relationship-graph';

describe('access matrix', () => {
  it('declares every section/audience pair', () => {
    const missing = SECTIONS.flatMap((s) => AUDIENCES.filter((a) => ACCESS_MATRIX[s]?.[a] === undefined).map((a) => `${s}/${a}`));
    expect(missing).toEqual([]);
  });

  describe.each(SECTIONS)('section %s', (section) => {
    it.each(AUDIENCES)('audience %s resolves to the declared grant', (audience) => {
      const graph = buildGraph(audience);

      expect(resolveSectionAccess({ section, graph, viewerId: graph.viewerId, subjectId: graph.subjectId })).toEqual(
        ACCESS_MATRIX[section][audience],
      );
    });
  });
});
```

**Emission assertion — absence on the wire.** This is the assertion the unit layer cannot make.
`toBeUndefined` would pass on a serialized `null`; `not.toHaveProperty` is what proves the key is gone.

```typescript
import * as request from 'supertest';

it('omits a denied section entirely for a colleague', async () => {
  const { body } = await request(app.getHttpServer())
    .get(`/employees/${subject.id}/profile`)
    .set('Authorization', `Bearer ${colleagueToken}`)
    .expect(200);

  expect(body).not.toHaveProperty('compensation');
  expect(Object.keys(body)).toEqual(expect.not.arrayContaining(['compensation']));
});
```

**Deterministic time — the clock is injected, never awaited.**

```typescript
it('denies access when the assignment confirmation is stale', () => {
  const clock = fixedClock('2026-08-25T12:00:00Z');
  const assignment = buildAssignment({ confirmedAt: '2026-08-20T12:00:00Z' });

  expect(resolveProjectAccess({ assignment, clock })).toEqual({ granted: false, reason: 'STALE_CONFIRMATION' });
});
```

**Tagging.** Jest selects by name pattern, Playwright by grep. Tag by priority and concern so CI tiers
can address them:

```bash
# Backend: P0 only
npx jest -t "@P0"

# Backend: the access matrix suite alone
npx jest --selectProjects unit -t "access matrix"

# Frontend: P0 plus P1 journeys
npx playwright test --grep "@P0|@P1"

# Frontend: accessibility sweep
npx playwright test --grep @a11y
```

---

## Appendix B: Knowledge Base References

- **`risk-governance.md`** — risk scoring, classification, and gate rules
- **`probability-impact.md`** — the 1–3 probability and impact scales used in the register
- **`test-levels-framework.md`** — unit vs API vs component vs E2E selection
- **`test-priorities-matrix.md`** — P0–P3 criteria
- **`test-quality.md`** — definition of done: no hard waits, deterministic, isolated, fast
- **`nfr-criteria.md`** — NFR category and threshold guidance
- **`library-integration-mandate.md`** — the two-gate rule behind the tooling note above

---

**Generated by:** BMad TEA Agent
**Workflow:** `bmad-testarch-test-design`

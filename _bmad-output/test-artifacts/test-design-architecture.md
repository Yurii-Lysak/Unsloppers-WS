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

# Test Design for Architecture: People Management Platform

**Purpose:** Architectural concerns, testability gaps, and NFR requirements for review by the
Architecture and Dev teams. This is the contract between test design and engineering on what must be
addressed before test development begins.

**Date:** 2026-08-25
**Author:** Unslopper (TEA Master Test Architect)
**Status:** Architecture Review Pending
**Project:** people management
**PRD Reference:** `_bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md`
**ADR Reference:** `_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md`

---

## Executive Summary

**Scope:** System-level test design for the full platform — 13 epics, 61 stories, 57 functional and
8 non-functional requirements, centred on a 16-section × 6-audience access model.

**Business context (from PRD):**

- **Problem:** People data is fragmented across systems with no single access-controlled view.
- **Primary success metric (SM-1):** zero unauthorized data exposure incidents.
- **Delivery shape:** three developers working parallel tracks; foundation phase gates feature work.

**Architecture (from the spine):**

- **AD-4 / AD-5:** access resolved server-side, per section, per request, at the data-access layer.
- **AD-6 / AD-11:** functional roles and custom fields are runtime data — no deploy, no migration.
- **AD-7:** history-table writes fire timeline events structurally, via a Prisma client extension.
- **AD-12:** a single containerized environment; no staging tier.

**Expected scale:** 500+ employee records; All Employees list under 2 seconds with three active
filters, permission resolution included.

**Risk summary:**

- **Total risks:** 18 — 12 high-priority (score ≥6), 5 medium, 1 low.
- **Score-9 risks:** 3, all foundation-phase.
- **Test effort:** ~91 test groups producing ~365 cases, ~120–200 hours, plus ~40–70 hours of test
  infrastructure that is a prerequisite rather than part of it.

---

## Quick Guide

### 🚨 BLOCKERS — Team Must Decide

These must be resolved before test development on the access model can begin.

1. **R-003 / TC-7: No CI pipeline exists.** Five architectural invariants name CI as their enforcement
   point (AD-1 boundaries, AD-10 type diff, AD-13 matrix suite, the NFR-2 benchmark, the NFR-5 PII
   scan). Until CI exists they are conventions, and SM-1 is a claim rather than a result. Also decide
   the platform — AD-12 leaves it open. (Owner: Team / Story 1.17)
2. **R-002 / TC-2: Test isolation against a shared Postgres is undecided.** Jest 30 runs parallel
   workers; the starter e2e spec already writes to a shared database. Choose the mechanism —
   schema-per-worker, transaction rollback, or template restore — before 61 stories of suites exist.
   Retrofitting isolation later is the expensive version of this fix. (Owner: Backend + Architect)
3. **TC-1: `access-model.md` has no machine-readable form.** Story 1.14 requires the harness be
   data-driven off the matrix so a matrix change automatically extends or flags coverage. The file is
   prose, so that requirement currently has no implementation. One structured source that both the
   document and every suite derive from is the cheapest simultaneous mitigation for R-001 and R-007.
   (Owner: Architect + Backend)

**What we need from the team:** these three resolved in the foundation phase. Test development on the
access model is blocked without them.

---

### ⚠️ HIGH PRIORITY — Team Should Validate

Recommendations with detail in the mitigation plans below. Each needs an approval or a
counter-proposal, not a discussion from scratch.

1. **R-001: the leak surface is wider than the profile API.** AD-5 makes the resolver trustworthy; it
   does not make export, saved views, audience previews, dashboard aggregates, or search trustworthy.
   Each is an independent emission path.
2. **R-004 → R-005: the performance/correctness collision.** The likeliest NFR-2 breach has caching as
   its natural fix, and caching is exactly the SM-C2 counter-metric. Settle the resolution strategy
   before the list endpoint is built.
3. **R-006 / TC-4: AD-8's freshness window has no value.** An assignment confirmed before an outage
   keeps granting access if the window is unset, too wide, or untested.
4. **R-012 / TC-10: dynamic filter and sort is the one injection-shaped surface.** FR-7 plus FR-8 mean
   a client-supplied field name reaches a query builder.
5. **R-013 / TC-8: "unavailable" and "not granted" have no distinct wire shape,** so a provider bug
   renders identically to a correct denial.
6. **R-008: the Prisma extension is a structural coupling with no declared bypass rule.**

**Approvers:** Architect for 1–5, Backend for 6.

---

### 📋 INFO ONLY — Solutions Provided

1. **Test strategy:** unit for access decisions (all 96 matrix cells), API for emission assertions
   (the ~18 denial cells plus one per surface), integration for database-state invariants, a
   deliberately thin E2E layer. Rationale in the QA doc.
2. **Coverage:** ~91 test groups producing ~365 cases, prioritized P0–P3 against the risk register.
3. **Execution tiers:** PR / nightly / weekly, with the external-integration run deferred to weekly so
   third-party availability cannot make PR results non-deterministic.
4. **Declared overlap:** access rules are asserted at both unit and API level. Not redundancy — the
   two layers can disagree, and that disagreement is the bug class SM-1 exists to prevent.

**What we need from the team:** review and acknowledge. No decisions required.

---

## For Architects and Devs — Open Topics

### Risk Assessment

**Total risks:** 18 — 12 scoring ≥6, 5 scoring 4, 1 scoring ≤2.

#### High-Priority Risks (Score ≥6) — Immediate Attention

| Risk ID | Category | Description | Prob | Impact | Score | Mitigation | Owner | Timeline |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **R-001** | SEC | A section leaks through a surface other than the profile API — export, shared saved view, filter on a non-visible field, campaign audience preview, dashboard aggregate, or search result | 3 | 3 | **9** | One resolved grant map shared by every emission path; per-surface negative assertions | Backend + Architect | Foundation |
| **R-002** | TECH | Parallel Jest workers on a shared Postgres yield order-dependent passes, producing a false green on the negative access suite | 3 | 3 | **9** | Decide the isolation mechanism before suites accumulate | Backend | Foundation |
| **R-003** | OPS | No CI pipeline, so every CI-enforced invariant is advisory and SM-1 is unverifiable | 3 | 3 | **9** | Stand up CI with the five static gates | Team | Foundation (Story 1.17) |
| **R-004** | PERF | Transitive-closure resolution per row at 500+ records breaches the 2s budget | 3 | 2 | 6 | Settle resolution strategy pre-build; benchmark in CI | Architect | Foundation |
| **R-005** | SEC | A cache added under R-004 pressure uses TTL invalidation, producing a stale grant — SM-C2 realised | 2 | 3 | 6 | Generation-bump invalidation only (ASR-11) | Architect | Pre-implementation |
| **R-006** | SEC | An assignment confirmed before an outage keeps granting access because the freshness window is unset or untested | 2 | 3 | 6 | Set the window; injectable clock | Architect | Foundation |
| **R-007** | TECH | Access suites drift from `access-model.md` because the data-driven requirement has no machine-readable source | 3 | 2 | 6 | Structured matrix as the single source | Architect | Foundation |
| **R-008** | DATA | A history write bypasses the extension, or it double-fires, so the timeline silently diverges and FR-30's conflict rule is defeated | 2 | 3 | 6 | Declare the no-bypass rule; assert it | Backend | Pre-implementation |
| **R-009** | SEC | Custom-field visibility is inferable from filter result counts even when excluded from responses | 2 | 3 | 6 | Decide reject-vs-exclude (Q-4) and apply consistently | Architect | Pre-implementation |
| **R-010** | SEC | Shared-link expiry, revocation, and invalid-token responses are distinguishable, or access attempts are unlogged | 2 | 3 | 6 | Uniform error responses; model the access log | Backend | Pre-implementation |
| **R-011** | TECH | Frontend E2E cannot assert API response shape, so the highest-value negative cases land in no tier | 3 | 2 | 6 | Move response-body assertions to backend API tests | QA/Dev | Foundation |
| **R-012** | SEC | A dynamic filter or sort naming a runtime-defined field reaches the query builder unvalidated | 2 | 3 | 6 | Server-side allow-listing from the resolved field set | Architect | Pre-implementation |
| **R-013** | DATA | A missing provider registration renders as an omission indistinguishable from "not granted" | 2 | 3 | 6 | Define the unavailable-state contract shape | Architect | Pre-implementation |
| **R-015** | OPS | Time-dependent behaviour is tested against a wall clock, making tests slow, flaky, or vacuous | 3 | 2 | 6 | Injectable clock | Backend | Foundation |

#### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Prob | Impact | Score | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R-014 | BUS | Campaign activation partially completes, splitting recipients between having an action item and not (FR-42) | 2 | 2 | 4 | Single transaction; all items or an editable draft | Backend |
| R-016 | SEC | Shared-link and login endpoints have no rate limiting, permitting token guessing or credential stuffing | 2 | 2 | 4 | Rate limit both; monitor | Backend |
| R-017 | OPS | Load testing and 500-record seeding run against the only environment, which is also the demo environment | 2 | 2 | 4 | Schedule around demos; pseudonymized seed only | Team |
| R-018 | TECH | Mentorship `endedAt` is set without the required closing feedback — flagged below-altitude by the spine and still unresolved | 2 | 2 | 4 | Resolve the rule; enforce atomically | Architect |

No risks below score 4 are tracked separately; lower-scored observations are folded into the concerns
above.

#### Risk Category Legend

- **TECH** — technical/architecture (flaws, integration, scalability)
- **SEC** — security (access control, auth, data exposure)
- **PERF** — performance (SLA violations, degradation)
- **DATA** — data integrity (loss, corruption, inconsistency)
- **BUS** — business impact (UX harm, logic errors)
- **OPS** — operations (deployment, config, monitoring)

---

### NFR Testability Requirements

What architecture must provide so NFR validation can be automated later. Planning guidance, not
evidence assessment.

| NFR Category | Threshold / Requirement | Current Design Support | Gap / Decision Needed | Planned Evidence |
| --- | --- | --- | --- | --- |
| Security (NFR-1, SM-1) | Zero unauthorized exposure; every denial in a 96-cell matrix holds on every surface | Partial — AD-5 covers the resolver, not the other emission paths | Machine-readable matrix (TC-1); enumerate every emission surface | Matrix suite report plus per-surface negative results |
| Performance (NFR-2, SM-4) | List under 2s at 500+ records, 3 filters, permission resolution included | Unknown — no resolution strategy chosen | Concurrency assumption undefined (Q-1); no CI benchmark | Load report attributing the permission-resolution segment |
| Reliability (NFR-4) | External sources degrade without taking the app down | Partial — internal contracts stubbed, external boundaries not | No timeout, retry, or failure-threshold policy (Q-5); no stubbing seam (TC-5) | Fault-injection results at the client boundary |
| Accessibility (NFR-3, SM-7) | WCAG 2.1 AA on List, Profile, Dashboards | Supported — component library is a reasonable base | None blocking | Automated scan plus a manual keyboard/screen-reader checklist |
| Data protection (NFR-5) | Non-production environments use pseudonymized data only | Contradicted — AD-12 defines exactly one environment (TC-9) | Restate as an absolute prohibition, or name the non-production tier | Seed-tool assertions plus a CI PII scan |
| Auditability | Shared-link access attempts logged with time and origin (FR-6) | Absent — not modelled as data (TC-8) | Is the access log an entity, and what does it record (Q-8) | Log-content assertions |
| Maintainability | Module boundaries and generated-type fidelity hold | Declared but unenforced | Enforcement point does not exist (R-003) | Static analysis output |

**Unknown thresholds:** NFR-2 concurrency (Q-1); latency targets for any endpoint other than the list
(Q-2); the AD-8 freshness window (Q-3); external-client timeout and retry policy (Q-5). Recorded as
clarification items, not filled with guessed values.

**Assessment boundary:** PASS / CONCERNS / FAIL belongs to `nfr-assess`, once implementation evidence
exists.

---

### Testability Concerns and Architectural Gaps

#### Blockers to Fast Feedback

| Concern | Impact on testing | What architecture must provide | Owner | Timeline |
| --- | --- | --- | --- | --- |
| **TC-1 — matrix is prose** | The data-driven suite AD-13 specifies cannot be built; coverage drifts silently from the spec | A structured, machine-readable matrix that both the document and every suite derive from | Architect + Backend | Foundation |
| **TC-2 — no DB isolation** | Parallel workers make the negative access suite intermittently green | A decided isolation mechanism with a documented per-worker setup | Backend | Foundation |
| **TC-3 — no relationship-graph factory** | Every matrix cell is a function of the graph; the seed tool gives volume, not the precise shapes | A factory that builds an exact graph — reports-to at depth, project with PM and DM, PP plus HR line — per test | Backend + QA | Foundation |
| **TC-4 — no injectable clock** | Freshness window, link expiry, overdue, and assessment recency are untestable deterministically | A clock abstraction injected wherever "now" is read | Backend | Foundation |
| **TC-5 — no external stubbing seam** | Timetracker and PeopleForce calls cannot be faulted, so NFR-4 has no test tier | A client boundary that can be substituted, matching the AD-2 stub pattern | Backend | Foundation |
| **TC-8 — observability not modelled** | The shared-link access log and the unavailable-state contract cannot be asserted on because neither is data | Both modelled explicitly | Architect | Pre-implementation |
| **TC-10 — unvalidated dynamic fields** | The injection-shaped surface has no defined validation point to test | Server-side allow-listing derived from the resolved visible-field set | Architect | Pre-implementation |

#### Architectural Improvements Needed

1. **TC-6 — the frontend E2E harness cannot reach a real API**
   - **Current problem:** `playwright.config.ts` starts only Vite on port 4200 — no backend, no
     Postgres. The negative cases that matter most are assertions about an API response body.
   - **Required change:** own response-shape assertions in the backend API tier; keep Playwright for
     journeys and accessibility.
   - **Impact if not fixed:** the highest-value negative cases land in no tier at all (R-011).
   - **Owner:** QA/Dev. **Timeline:** foundation.

2. **TC-7 — `retries: 2` with `workers: 1` in CI**
   - **Current problem:** a test passing on the third attempt reports as passing. For a suite whose
     purpose is proving negative access cases, a retry-masked intermittent pass is indistinguishable
     from a leak that happens to be rare.
   - **Required change:** no retries on the access suite; treat a retry as a failure signal elsewhere.
     Reconsider `workers: 1` once TC-2 is resolved.
   - **Impact if not fixed:** SM-1 cannot be evidenced, because the evidence is retry-laundered.
   - **Owner:** QA/Dev. **Timeline:** with CI setup.

3. **TC-9 — the performance and seed environment is the demo environment**
   - **Current problem:** AD-12 scopes one environment. NFR-5 requires non-production environments to
     use pseudonymized data, but with one environment there is no non-production tier for the rule to
     apply to.
   - **Required change:** restate NFR-5 as an absolute prohibition on real PII anywhere, and schedule
     load runs away from demos.
   - **Impact if not fixed:** NFR-5 is unenforceable as written, and a load run can degrade a demo.
   - **Owner:** Team. **Timeline:** pre-implementation.

---

### Testability Assessment Summary

#### What Works Well

- **AD-5 enforces at the data-access layer**, so access cannot be forgotten at a controller. That
  removes an entire defect class from scope.
- **AD-2's C1–C8 contracts with Wave-0 stub providers** make module isolation a design property rather
  than a testing afterthought.
- **AD-4's per-request resolution** leaves no session staleness to reason about — correctness is local
  to a request.
- **AD-3's registry-collision-as-bootstrap-failure** turns a config mistake into a startup crash, the
  cheapest place to catch it.
- **AD-10's generated frontend types** already enforce the frontend/backend contract, which is why
  Pact-style contract testing is not recommended.

#### Accepted Trade-offs

Acceptable at this scale — three developers, one environment, demo-facing MVP:

- **No staging tier** (AD-12). Constrains where NFR-2 and NFR-5 evidence can be produced.
- **No blue/green or rollback rehearsal.** A single containerized deploy is the stated shape.
- **No observability stack.** Structured logs suffice; tracing is not warranted.
- **Weekly rather than continuous external-integration verification,** so third-party availability
  cannot make PR results non-deterministic.

These are documented trade-offs, not debt to repay — except TC-9, which should be revisited if the
product moves past demo scale.

---

### Risk Mitigation Plans (Score ≥6)

Only production-code and infrastructure mitigations appear here; QA-owned work is in the QA doc. All
are status **Planned**. Related risks sharing one mitigation path are grouped.

#### R-001: Leak through a non-profile surface (Score 9) — CRITICAL

1. Enumerate every path that emits employee data: profile API, list rows, `.xlsx` export, saved-view
   re-execution, campaign audience preview, dashboard aggregates, search and autocomplete, shared
   link.
2. Route every one through the same resolved grant map from AD-5 — no path constructs its own query.
3. Make the grant map the only way to project fields, so a new surface inherits enforcement rather
   than reimplementing it.
4. Require any new emission surface to register in that list as a review gate.

**Owner:** Backend + Architect. **Timeline:** foundation.
**Verification:** every enumerated surface has a passing negative assertion for the same matrix cell.

#### R-002: False greens from shared-database parallelism (Score 9) — CRITICAL

1. Choose the isolation mechanism — schema-per-worker, per-test transaction rollback, or template
   database restore.
2. Implement it in the shared test setup before feature suites are written.
3. Prove it with a deliberately order-dependent pair of tests that must pass in either order.

**Owner:** Backend. **Timeline:** foundation.
**Verification:** the suite passes with randomized order and full worker parallelism, repeatedly.

#### R-003: No CI pipeline (Score 9) — CRITICAL

1. Choose the platform (Q-6) and stand up the pipeline in Story 1.17.
2. Wire the five invariant gates: module boundaries, generated-type diff, access-matrix suite, focus
   detection, PII scan.
3. Add the NFR-2 benchmark as a nightly job with a recorded baseline.

**Owner:** Team. **Timeline:** foundation, before feature stories.
**Verification:** a PR violating each invariant is rejected by CI.

#### R-007: Suites drift from the access matrix (Score 6)

1. Translate `access-model.md` into a structured, machine-readable matrix covering all 16 sections, 6
   audiences, field-level exceptions, and record-level flags.
2. Generate the human-readable document from that source, so the two cannot diverge.
3. Have suites import the same source, and fail on any section/audience pair the matrix does not
   declare — a new section must not default to allowed.

**Owner:** Architect + Backend. **Timeline:** foundation.
**Verification:** adding a section to the matrix without a corresponding grant fails the suite; adding
one to the code without a matrix entry also fails.

#### R-004 / R-005: Performance pressure producing a stale-grant cache (Score 6)

1. Decide the resolution strategy before the list endpoint is built — batch resolution per request
   rather than per row is the obvious first candidate.
2. Benchmark against 500+ records with three filters as an explicit acceptance step for the list
   story, not a later discovery.
3. If a cache becomes necessary, invalidate on a relationship-graph generation bump, never a bare TTL
   (ASR-11).

**Owner:** Architect. **Timeline:** foundation.
**Verification:** the benchmark passes and a mid-session relationship change is
visible on the next request.

#### R-006 / R-015: Untestable time boundaries (Score 6)

1. Set the AD-8 freshness window from the observed timetracker sync interval (Q-3).
2. Introduce a clock abstraction injected wherever "now" is read.
3. Keep authorization fail-safe and display fail-soft as independent paths, so a stale record denies
   access while still rendering something useful.

**Owner:** Architect + Backend. **Timeline:** foundation.
**Verification:** boundary cases on both sides of the window are asserted with a
controlled clock.

#### R-008: Timeline divergence (Score 6)

1. State explicitly that no write path may bypass the Prisma extension.
2. Assert exactly-once event emission per history write, and the FR-30 conflict rule in both orderings.
3. Add a guard against a second write path appearing later.

**Owner:** Backend. **Timeline:** pre-implementation.
**Verification:** integration tests over the extension, plus a static guard.

#### R-009 / R-012 / R-013: Inference, injection, and indistinguishable failures (Score 6)

1. Resolve Q-4 — a filter naming a non-visible field is either rejected or silently excluded, and the
   two are externally distinguishable, so pick one.
2. Allow-list filter and sort fields server-side from the caller's resolved visible-field set.
3. Give "granted but unavailable" a distinct wire shape from "not granted" (Q-9).

**Owner:** Architect. **Timeline:** pre-implementation.
**Verification:** a filter on a non-visible field returns the decided shape and
leaks nothing through result counts.

#### R-010: Shared-link disclosure (Score 6)

1. Return uniform responses for expired, revoked, and invalid tokens.
2. Model the access log as data, and record every attempt with time and origin (Q-8).
3. Reject S3, S7, and S13 server-side under any payload, independent of link configuration.

**Owner:** Backend. **Timeline:** pre-implementation.
**Verification:** the three token failure modes are externally identical, and
every attempt appears in the log.

---

### Assumptions and Dependencies

#### Assumptions

1. `access-model.md` is the authoritative access specification; where it and the PRD differ, it wins.
2. The 16 sections and 6 audiences are stable for this milestone — new sections are a matrix change,
   not a new mechanism.
3. AD-12's single environment is fixed for this milestone; no staging tier will appear.
4. The team stays at three developers with no dedicated QA, so test work distributes across feature
   work rather than running as a parallel track.
5. Timetracker and PeopleForce are read-only sources; the platform never writes back.

#### Dependencies

1. **CI platform decision (Q-6)** — required before the foundation-phase gate.
2. **Machine-readable access matrix (TC-1)** — required before access suites are written.
3. **AD-8 freshness window (Q-3)** — required before project-assignment access tests can assert
   boundaries.
4. **NFR-2 concurrency assumption (Q-1)** — required before the load scenario can be shaped.
5. **A 500+ record pseudonymized dataset** — required before the performance benchmark can run.

#### Risks to the Plan

- **Risk:** the foundation blockers slip into feature work.
  - **Impact:** access suites get written against a prose matrix and hand-maintained, roughly doubling
    P0 effort while lowering confidence.
  - **Contingency:** treat the foundation-phase gate as hard; if it must slip, write the matrix suite
    last so it can still be generated rather than hand-authored.
- **Risk:** the clarification items (Q-1 through Q-10) stay open.
  - **Impact:** NFR tests can be built but not gated, so NFR-2 and NFR-4 have no pass condition.
  - **Contingency:** build the scenarios now with the threshold externalized as configuration, and set
    the value when it lands.

---

**Next steps for the Architecture team:**

1. Review the Quick Guide and decide the three blockers.
2. Confirm owners and timelines for the twelve high-priority risks.
3. Answer the clarification items that gate NFR thresholds (Q-1, Q-3, Q-5) and access semantics
   (Q-4, Q-8, Q-9).
4. Validate the assumptions above, particularly the stability of the section and audience sets.

**Next steps for the QA/Dev team:**

1. See `test-design-qa.md` for scenarios, execution strategy, and effort.
2. Begin the foundation infrastructure — matrix generator, graph factory, DB isolation, clock — since
   it gates everything else.

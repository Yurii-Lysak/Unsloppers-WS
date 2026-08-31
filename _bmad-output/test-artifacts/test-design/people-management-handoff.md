---
title: 'TEA Test Design → BMAD Handoff Document'
version: '1.0'
workflowType: 'testarch-test-design-handoff'
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - _bmad-output/planning-artifacts/epics.md
  - _bmad-output/specs/spec-people-management-platform/access-model.md
sourceWorkflow: 'testarch-test-design'
generatedBy: 'TEA Master Test Architect'
generatedAt: '2026-08-25'
projectName: 'people management'
---

# TEA → BMAD Integration Handoff

## Purpose

Bridges this test design into BMAD's epic and story decomposition, so risk assessments, quality gates,
and test strategy reach implementation planning rather than sitting in a separate document.

## TEA Artifacts Inventory

| Artifact | Path | BMAD Integration Point |
| --- | --- | --- |
| Architecture concerns and risk register | `_bmad-output/test-artifacts/test-design-architecture.md` | Epic-level quality gates; foundation-phase blockers |
| QA test plan | `_bmad-output/test-artifacts/test-design-qa.md` | Story acceptance criteria; test-level assignment |
| Working notes (steps 1–4) | `_bmad-output/test-artifacts/test-design-progress-system.md` | Testability concerns TC-1…TC-10; clarification items Q-1…Q-10 |
| Risk register | Embedded in the architecture doc | Epic risk classification; story priority |
| Coverage matrix | Embedded in the QA doc | Story test requirements |

## Epic-Level Integration Guidance

### Risk References

Epic 1 carries a disproportionate share of the risk and should be treated as the gate for everything
else. Seven of the twelve high-priority risks resolve there, including all three scoring 9.

| Epic | High-priority risks | Notes for epic planning |
| --- | --- | --- |
| **1 — Access Control, Roles & Employee Profile** | R-001, R-002, R-003, R-004, R-005, R-006, R-007 | The foundation gate. Add the test infrastructure blockers (TC-1, TC-2, TC-3, TC-4, TC-5) as explicit stories, not as implicit setup |
| **3 — All Employees Directory & Custom Fields** | R-004, R-009, R-012 | Owns the 2s performance budget and the one injection-shaped surface |
| **7 — Career Timeline** | R-008 | Owns the Prisma extension coupling and the FR-30 conflict rule |
| **9 — Mentorship Hub** | R-018 | The closing-feedback rule is still unresolved at spine level |
| **10 — Forms & Survey Campaigns** | R-014 | Activation atomicity |
| **12 — Dashboards** | R-001 | Aggregates are an independent emission surface, not a view over a safe one |
| **13 — External Integrations** | R-011 (structural), NFR-4 | Owns the external boundary and its fault behaviour |
| **2, 4, 5, 6, 8, 11** | R-001 (surface-specific) | Each introduces at least one new data-emission path and inherits the whitelist obligation |

### Quality Gates

| Epic | Gate criteria |
| --- | --- |
| **1** | P0-001 through P0-003 and P0-011 through P0-013 passing. R-001, R-002, R-003 closed. CI active with all five invariant gates. This gate blocks every other epic |
| **2** | P2-005 passing — self-service writes scoped to photo and certificates only |
| **3** | P0-008, P0-009, P0-014, P1-009, P1-010 passing. The 2s budget met with permission resolution included |
| **4** | P1-013 passing, including authorship surviving access loss |
| **5** | P2-001 passing — risk trend identical on all three surfaces |
| **6** | P2-003 passing — approval creates no project assignment |
| **7** | P1-004, P1-005 passing — exactly-once event emission, no bypass path |
| **8** | P2-004 passing — append-only assessment history, no IDP reopen |
| **9** | P1-014 passing — derived status not settable, ending atomic with closing feedback |
| **10** | P1-007 passing — all items or a fully-editable draft; audience frozen |
| **11** | P2-006 passing — empty comparison period renders explicitly |
| **12** | P0-004 and P2-002 passing — dashboard aggregates leak nothing, scoping correct per role |
| **13** | P1-008 passing — the app survives every external fault mode |

## Story-Level Integration Guidance

### P0/P1 Test Scenarios That Must Become Acceptance Criteria

1. **Every story adding a data-emission surface** must carry an acceptance criterion that the surface
   is asserted against the access matrix fixture. The surface list is: profile, list rows, export,
   saved-view re-execution, campaign audience preview, dashboard aggregate, shared link, search. A new
   surface joins that list (P0-004).
2. **Story 1.14 (matrix suite)** must state that the harness fails on an unmapped section/audience
   pair. A new section defaulting to allowed is the failure mode this exists to prevent (P0-001).
3. **Any story touching project-assignment access** must assert the AD-8 fail-safe on both sides of
   the freshness boundary, and that display degrades independently of authorization (P0-011).
4. **The shared-link story** must assert that S3, S7, and S13 are rejected server-side under any
   payload, and that expired, revoked, and invalid tokens are externally indistinguishable (P0-010).
5. **Any story adding a filter or sort** must assert that unknown, malformed, and unauthorized field
   names are handled by the decided rule — the decision itself is Q-4 (P0-009).
6. **Any story writing to a history table** must assert exactly one timeline event, and that no other
   write path exists (P1-004).
7. **Any story rendering a derived value** (overdue, risk trend, mentorship status) must assert the
   value is identical on every surface that shows it, and computed server-side (P1-006, P2-001).
8. **Every story with a UI surface** on List, Profile, or Dashboards must include the WCAG 2.1 AA
   automated scan as an acceptance criterion (P1-015).

### Data-TestId Requirements

Recommended `data-testid` attributes, so E2E selectors do not depend on copy or layout:

- `employee-list-table`, `employee-list-row-{id}`, `employee-list-filter-{field}`
- `profile-section-{sectionKey}` — one per registered section, which lets a test assert absence by key
  rather than by inspecting rendered text
- `profile-section-unavailable-{sectionKey}` — distinct from absent, so the R-013 distinction is
  visible to a test
- `dashboard-counter-{metric}`, `dashboard-project-selector`
- `shared-link-view`, `shared-link-error`
- `action-item-{id}`, `action-item-complete`, `action-item-cancel`

The section-key attributes matter most: without them, "this section is not rendered" and "this section
rendered empty" are hard to distinguish in a browser test, which is exactly the ambiguity SM-1 cannot
afford.

## Risk-to-Story Mapping

| Risk ID | Category | P×I | Recommended Story / Epic | Test Level |
| --- | --- | --- | --- | --- |
| R-001 | SEC | 3×3 = **9** | Epic 1 (resolver) plus one criterion in every surface-adding story | Unit + API + E2E |
| R-002 | TECH | 3×3 = **9** | Epic 1 — new infrastructure story | Infrastructure |
| R-003 | OPS | 3×3 = **9** | Epic 1, Story 1.17 | CI |
| R-004 | PERF | 3×2 = 6 | Epic 3 — list endpoint story | k6 |
| R-005 | SEC | 2×3 = 6 | Epic 1 — resolution strategy story | API |
| R-006 | SEC | 2×3 = 6 | Epic 1 — project-assignment access story | Unit + Integration |
| R-007 | TECH | 3×2 = 6 | Epic 1, Story 1.14 plus a new matrix-source story | Unit |
| R-008 | DATA | 2×3 = 6 | Epic 7 — timeline extension story | Integration |
| R-009 | SEC | 2×3 = 6 | Epic 3 — custom field visibility story | API |
| R-010 | SEC | 2×3 = 6 | Epic 1, Story 1.12 | API |
| R-011 | TECH | 3×2 = 6 | Epic 1 — test tier decision, no product story | Structural |
| R-012 | SEC | 2×3 = 6 | Epic 3 — filter and sort story | API |
| R-013 | DATA | 2×3 = 6 | Epic 1 — registry contract story | Integration |
| R-014 | BUS | 2×2 = 4 | Epic 10 — campaign activation story | Integration |
| R-015 | OPS | 3×2 = 6 | Epic 1 — clock infrastructure story | Unit |
| R-016 | SEC | 2×2 = 4 | Epic 1 — auth and rate limiting story | API |
| R-017 | OPS | 2×2 = 4 | Epic 1 — seed tool story | Unit + CI |
| R-018 | TECH | 2×2 = 4 | Epic 9 — mentorship lifecycle story | API |

## Open Items That Should Shape Stories

Ten clarification items remain open. Six of them change what a story's acceptance criteria say, so
they should be resolved during epic refinement rather than during implementation:

| ID | Question | Affects |
| --- | --- | --- |
| Q-1 | Does "2s at 500+ records" assume one viewer or N concurrent? | Epic 3 list story AC and the k6 scenario shape |
| Q-3 | What is AD-8's freshness-window value? | Epic 1 project-assignment story AC |
| Q-4 | Is a filter on a non-visible field rejected or silently excluded? | Epic 3 filter story AC — the two are externally distinguishable |
| Q-5 | What timeout, retry, and failure-threshold policy do external clients use? | Epic 13 story AC |
| Q-8 | Is the shared-link access log an entity, and what does it record? | Epic 1 Story 1.12 AC |
| Q-9 | What wire shape distinguishes "granted but unavailable" from "not granted"? | Epic 1 registry story AC, and the `data-testid` guidance above |

Q-2, Q-6, Q-7, and Q-10 affect planning and tooling rather than acceptance criteria.

## Recommended BMAD → TEA Workflow Sequence

1. **TEA Test Design** — complete; produced this handoff.
2. **BMAD Create Epics & Stories** — consume this handoff. The epic list already exists, so this is a
   refinement pass that adds the quality gates and acceptance criteria above, plus the six
   infrastructure stories the foundation gate needs.
3. **TEA Framework** (`bmad-testarch-framework`) — run before ATDD. This plan targets the installed
   stack (Jest + Supertest + vanilla Playwright); if the configured utils packages are wanted, they
   need installing first.
4. **TEA ATDD** — generate red-phase acceptance tests for the P0 scenarios, starting with the matrix
   harness.
5. **BMAD Implementation** — build against the failing tests.
6. **TEA Automate** — expand into P1 and P2.
7. **TEA Trace** — verify coverage against the requirement set.
8. **TEA NFR Assess** — run once evidence exists; this design identifies the artifacts, it does not
   grade them.

## Phase Transition Quality Gates

| From Phase | To Phase | Gate Criteria |
| --- | --- | --- |
| Test Design | Epic/Story Creation | Every score-9 risk has an owned mitigation and a story to carry it |
| Epic/Story Creation | Framework | The six infrastructure stories exist in Epic 1 and Q-4, Q-8, Q-9 are answered |
| Framework | ATDD | Matrix fixture, graph factory, DB isolation, and injectable clock all available |
| ATDD | Implementation | Failing acceptance tests exist for all P0 scenarios |
| Implementation | Test Automation | All P0 acceptance tests pass; CI enforces all five invariants |
| Test Automation | Release | Every denial cell has a passing negative test; P0 at 100%, P1 at ≥95%; no score-9 risk open |

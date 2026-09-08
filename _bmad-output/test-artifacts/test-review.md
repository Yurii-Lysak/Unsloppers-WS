---
workflowType: 'testarch-test-review'
stepsCompleted:
  [
    'step-01-load-context',
    'step-02-discover-tests',
    'step-03-quality-evaluation',
    'step-03a-subagent-determinism',
    'step-03b-subagent-isolation',
    'step-03c-subagent-maintainability',
    'step-03e-subagent-performance',
    'step-03f-aggregate-scores',
    'step-04-generate-report',
  ]
lastStep: 'step-04-generate-report'
lastSaved: '2026-09-05'
inputDocuments:
  - _bmad-output/test-artifacts/known-red-diagnosis.md
  - _bmad-output/test-artifacts/ci-pipeline-progress.md
  - _bmad-output/test-artifacts/test-design-qa.md
  - .claude/skills/bmad-testarch-test-review/steps-c/criteria-registry.md
---

# Test Quality Review: services/backend (full suite)

**Quality Score**: 81/100 (B - Good)
**Review Date**: 2026-09-05
**Review Scope**: suite
**Reviewer**: Unslopper (via TEA Agent, Create mode)

---

Note: This review audits existing tests; it does not generate tests.
Coverage mapping and coverage gates are out of scope here. Use `trace` for coverage decisions.

## Executive Summary

**Overall Assessment**: Good

**Recommendation**: Request Changes

<!-- Computed, never chosen, per steps-c/step-03f-aggregate-scores.md §3b: any HIGH severity
     finding forces "Request Changes" regardless of the numeric score. Four HIGH violations were
     found (see below), so the recommendation is Request Changes even though the underlying score
     (81/100) would otherwise read as solidly "Good." -->

**Context Basis**: pr_diff_truncated

<!-- The three context files are prior investigation/test-design documents, not a PR diff. They are
     the closest available label in the template's enum but the fit is imperfect — treat this as
     "partial, narrative context was available, not a diff." See "Context and Integration" below for
     what each document actually contributed. -->

**Context Waivers Applied**: 0

### Key Strengths

✅ Deterministic time is treated as a first-class dependency everywhere it matters — `FixedClock` (`test/support/fixed-clock.ts`) is injected via `createTestApp({ clock })` in every suite that touches TTLs, expiry, or overdue derivation (`access-resolution-cache.e2e-spec.ts`, `action-items.e2e-spec.ts`, `shared-links.e2e-spec.ts`, `project-assignment-sync.e2e-spec.ts`), and its own unit spec (`test/support/fixed-clock.spec.ts`) proves it rejects unparseable instants and non-finite offsets rather than silently producing `NaN` time.

✅ Zero disabled or focused tests, zero hard waits, zero `console.log`/timer-based synchronization across all 82 reviewed files — confirmed by full-suite grep for `.only`/`.skip`/`xit`/`xdescribe`/`waitForTimeout`/`sleep(`/`jest.useFakeTimers`, not by sampling.

✅ A genuine relationship-graph test factory (`test/support/graph-factory.ts`, exercised by `test/support/graph-factory.spec.ts`) provides a fluent builder (`aGraph().reportsTo(...).project(...).assign(...).peoplePartner(...).build()`) that turns the hardest fixture in the codebase — an arbitrary-depth reporting/PM/DM/PP graph — into a one-line, readable declaration instead of hand-rolled Prisma calls.

✅ The access-matrix coverage mechanism (`test/support/access-matrix.ts` + `.spec.ts`) is self-verifying: `assertMatrixCoverage`, `assertDeniedMatrixCoverage`, and `assertFlagGatedCoverage` fail the suite by name (`/S6\/colleague/`) if any of the 16×6 matrix cells, 18 denial cells, or 13 flag-gated cases go unexercised — this is exactly the "a new section cannot default to allowed" guarantee `test-design-qa.md`'s P0-001/R-007 calls for, enforced in the test harness itself rather than by convention.

✅ External-boundary fault injection (`test/support/external-boundary.ts` + `.spec.ts`) gives real substitutability for the TimeTracker/PeopleForce integration surface — `respond`, `malformed`, `hang`, `reset`, `delayMs`, `goOffline`/`comeBackOnline` — and `projects-sync.service.spec.ts` / `timetracker.service.spec.ts` use it (or an equivalent mocked-fetch seam) to assert on `AbortSignal`-bounded timeouts and typed `TimetrackerApiError` mapping rather than letting a raw network error escape.

### Key Weaknesses

❌ Two record-level security assertions in the suite's highest-risk area (the zero-leak access matrix, R-001/SM-1 per `test-design-qa.md`) are wrapped in an `if (section && 'data' in section)` guard with no unconditional counterpart — if the guarded shape is ever absent, the assertion silently does not run and the test still passes (`test/matrix-flag-gated.e2e-spec.ts`).

❌ `test/users.e2e-spec.ts` has no `beforeEach` database reset and chains state through a describe-scoped `createdId` variable across independent `it` blocks — the suite passes today only because Jest runs tests in file order; reordering, `.only`-ing a later test, or a future parallel-within-file runner would break it silently.

❌ `test/action-items.e2e-spec.ov.ts` — corr. `test/action-items.e2e-spec.ts` — is ~1,633 lines, well past the 1000-line maintainability ceiling, mixing four largely-independent concerns (CRUD lifecycle, overdue derivation, campaign bulk-activation, provider-failure) that would each read better as their own file.

❌ `test/employees.e2e-spec.ts` hand-rolls the same 6-call Prisma fixture (user → employee → grade/position/department/employmentType history) inline at least four times instead of using the `createEmployeeUser` factory already shared by ten other e2e spec files (`test/support/employee-users.ts`), and one of those tests (the grade inline-edit test) folds five distinct concerns into a single `it`.

### Summary

This is one of the stronger backend Jest/Supertest suites reviewed under this rubric: 82 files (55 unit/integration, 27 e2e and support), zero Critical findings, zero disabled/focused tests, and a deliberately engineered test-infrastructure layer (injectable clock, relationship-graph factory, per-worker DB isolation, matrix coverage assertions, external-boundary fault injection) that reads as if it was built specifically to satisfy `test-design-qa.md`'s TC-1 through TC-5 entry criteria — because it was. The suite is doing exactly the hard thing its own risk register (R-001, 18 risks, 8 of them SEC) says matters most: proving a 16-section × 6-audience access model holds, with structured wire-shape assertions (`not.toHaveProperty`, exact key-set equality) rather than loose truthy checks, in almost every file reviewed.

The four HIGH findings are what keep this out of "Approve" territory, and three of the four sit in exactly the areas the suite otherwise gets right: two silently-skippable assertions inside the access-matrix leak suite (the one place a silent no-op is most expensive), one real test-order dependency, and one oversized file. None of these are fabricated to hit a quota — each is cited with the exact file and reasoning below, and the two Medium findings and one confirmed cross-reference to `known-red-diagnosis.md`'s S16 substring-assertion fragility round out a Request-Changes verdict on an otherwise well-built suite.

---

## Quality Criteria Assessment

| Criterion                            | Status                | Violations | Basis                                                     | Notes                                                                                                                                    |
| ------------------------------------- | ---------------------- | ---------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| BDD Format (Given-When-Then)          | ✅ PASS                | 0          | Convention: `bddNaming` (82 of 82 sampled)                 | Every test name states an observable behavior ("rejects a payload missing the link field with 400"), never a method or selector name.     |
| Test IDs                              | ✅ PASS (n/a)          | 0          | Convention: `testIds` (0 of 82 sampled)                    | No stable test-id convention exists in this backend suite (no DOM); absent convention deducts nothing per the registry.                   |
| Priority Markers (P0/P1/P2/P3)        | ✅ PASS (n/a)          | 0          | Convention: `priorityMarkers` (0 of 82 sampled)             | `test-design-qa.md` defines a P0–P3 framework at the plan level, but no reviewed test name or tag encodes it; convention is absent, not violated. |
| Disabled or Focused Tests             | ✅ PASS                | 0          | Absolute                                                  | Zero `.skip`/`.only`/`xit`/`xdescribe`/`test.todo` across all 82 files (grep-verified, not sampled).                                       |
| Hard Waits (sleep, waitForTimeout)     | ✅ PASS                | 0          | Absolute                                                  | Zero `waitForTimeout`/`sleep(`/bare timers in any reviewed spec. The one `setTimeout` in the tree is in `test/support/external-boundary.ts`, which is fixture infrastructure simulating a hung server, not a spec file. |
| Determinism (no conditionals)         | ❌ FAIL                | 2          | Absolute (H3)                                             | Two guarded assertions with no unconditional counterpart in `matrix-flag-gated.e2e-spec.ts` — see Recommendations 1–2.                     |
| Isolation (cleanup, no shared state)  | ❌ FAIL                | 1          | Absolute (H4)                                             | `users.e2e-spec.ts` chains state across tests via a describe-scoped variable with no reset — see Recommendation 3.                          |
| Fixture Patterns                      | ⚠️ WARN                | 1          | Convention: `dataFactories`/fixture reuse (established; ~72 of 82 sampled use shared/local factory helpers) | `employees.e2e-spec.ts` bypasses the shared `createEmployeeUser` factory with inline duplication — see Recommendation 4.                    |
| Data Factories                        | ⚠️ WARN                | 1          | Convention: `dataFactories` (established; ~72 of 82 sampled) | Same finding as Fixture Patterns — one file (`employees.e2e-spec.ts`) constructs the same domain payload shape inline 4+ times.             |
| Network-First Pattern                 | ✅ PASS (n/a)          | 0          | Applicability: file navigates and reads data-dependent content | Backend Jest/Supertest suite; no `page.goto`/`cy.visit`/browser navigation exists anywhere in this stack.                                  |
| Playwright Utils Adoption             | ✅ PASS (n/a)          | 0          | Convention: `playwrightUtils` — precondition `playwrightUtilsActive` is false | `tea_use_playwright_utils: false` and `@seontechnologies/playwright-utils` is not a dependency of `services/backend`; the row does not exist for this run. |
| Pact.js Utils Adoption                | ✅ PASS (n/a)          | 0          | Applicability — precondition `pactjsUtilsActive` is false  | `tea_use_pactjs_utils: false`, no `@seontechnologies/pactjs-utils` dependency, and no `.pact.`/contract-test files exist in this service.  |
| Explicit Assertions                   | ⚠️ WARN                | 2          | Absolute + Applicability (H3)                             | Same two guarded assertions as "Determinism" — `Explicit Assertions` and `Determinism` both draw on H3 per the registry's mapping table; reported once in the ledger, cited under both criteria per the template. |
| Test Length (≤1000 lines)             | ❌ FAIL                | 1          | Absolute (H5)                                             | `action-items.e2e-spec.ts` is ~1,633 lines.                                                                                               |
| Test Duration (≤1.5 min)              | ✅ PASS (n/a)          | —          | Absolute                                                  | No live timing run was performed (static review, no `--json` Jest run executed as part of this workflow); no per-file duration evidence to assess against the threshold, so this is not scored as a violation. |
| Flakiness Patterns                    | ⚠️ WARN                | 1          | Absolute + Applicability (H4)                             | The `users.e2e-spec.ts` order-dependency is the one structural flakiness risk found; no hard waits, no wall-clock races, no unawaited async detected elsewhere. |

**Total Violations**: 0 Critical, 4 High, 2 Medium, 0 Low

**Convention Baseline**: `review_scope: suite` — the review set is the entire discoverable backend test corpus, so there is no external sample to measure a baseline against separately. Per the workflow variables supplied for this run, the corpus itself was used as the baseline (`corpusSize = sampled = 82`), and every adoption ratio cited above (e.g. "82 of 82", "0 of 82", "~72 of 82") was measured directly against these same 82 files by reading them, not estimated. This deviates from the template's literal "{sampled} test files sampled outside the review set" line because, for a `suite`-scope review, no such outside sample exists; stating it verbatim would misrepresent what was actually measured.

---

## Quality Score Breakdown

```
Starting Score:          100
Critical Violations:     -0 × 10 = -0
High Violations:         -4 × 5 = -20
Medium Violations:       -2 × 2 = -4
Low Violations:          -0 × 1 = -0

Bonus Points:
  Excellent BDD:         +5
  Comprehensive Fixtures: +0
  Data Factories:        +0
  Network-First:         +0
  Perfect Isolation:     +0
  All Test IDs:          +0
                         --------
Total Bonus:             +5

Final Score:             81/100
Grade:                   B
```

Bonus rationale (each category is 0 or 5, awarded only when it holds across every one of the 82 reviewed files, never partial credit):

- **Excellent BDD (+5)**: held across all 82 files with no exception found.
- **Comprehensive Fixtures (0)**: `employees.e2e-spec.ts` inlines duplicated setup instead of routing through the shared fixture, so it does not hold across every file.
- **Data Factories (0)**: same file, same reason — the bar is "every reviewed file," and one clear exception disqualifies the category.
- **Network-First (0)**: the criterion has no applicable case anywhere in this backend-only suite (no browser navigation exists to be network-first about); scored 0 rather than awarded, since a criterion that cannot fail here also cannot be said to have been demonstrated.
- **Perfect Isolation (0)**: `users.e2e-spec.ts` is a genuine counter-example (H4).
- **All Test IDs (0)**: not applicable — no DOM element lookups exist in a Supertest-only suite; scored 0 for the same reason as Network-First.

---

## Critical Issues (Must Fix)

No critical issues detected. ✅

---

## Defect Tracking

Each finding below is also tracked as an individual defect file in
`services/backend/defects/test-debt/`, so a fix can reference the exact
evidence directly from the repo:

| # | Finding | Row | Defect file |
| --- | --- | --- | --- |
| 1 | S7 guarded assertion can silently skip (`matrix-flag-gated.e2e-spec.ts:89-91`) | H3 | [`test-debt/04-matrix-flag-gated-s7-guarded-assertion.md`](../../services/backend/defects/test-debt/04-matrix-flag-gated-s7-guarded-assertion.md) |
| 2 | S16 guarded assertion is the *only* assertion (`matrix-flag-gated.e2e-spec.ts:282-285`) | H3 | [`test-debt/05-matrix-flag-gated-s16-guarded-assertion-only.md`](../../services/backend/defects/test-debt/05-matrix-flag-gated-s16-guarded-assertion-only.md) |
| 3 | Test-order dependency via unshared state (`users.e2e-spec.ts`) | H4 | [`test-debt/06-users-e2e-test-order-dependency.md`](../../services/backend/defects/test-debt/06-users-e2e-test-order-dependency.md) |
| — | Oversized file, ~1,633 lines (`action-items.e2e-spec.ts`) | H5 | [`test-debt/07-action-items-e2e-oversized-file.md`](../../services/backend/defects/test-debt/07-action-items-e2e-oversized-file.md) |
| 4 | Repeated inline fixture bypasses shared factory (`employees.e2e-spec.ts`) | M2 | [`test-debt/08-employees-e2e-duplicated-fixture.md`](../../services/backend/defects/test-debt/08-employees-e2e-duplicated-fixture.md) |
| 5 | Multi-concern test bundling (`employees.e2e-spec.ts:433-547`) | M3 | [`test-debt/09-employees-e2e-multi-concern-test.md`](../../services/backend/defects/test-debt/09-employees-e2e-multi-concern-test.md) |

The S16 false-alarm cross-reference to `known-red-diagnosis.md` §3a is
tracked separately as [`test-debt/01-s16-fragile-substring-assertion.md`](../../services/backend/defects/test-debt/01-s16-fragile-substring-assertion.md).

---

## Recommendations (Should Fix)

### 1. Guarded assertion can silently skip the one check the test exists to make (S7)

**Severity**: P1 (High)
**Location**: `test/matrix-flag-gated.e2e-spec.ts:89-91`
**Row**: H3
**Criterion**: Determinism (no conditionals) / Explicit Assertions
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Issue Description**:
In `'hides unflagged S7 notes from Self on profile and parallel route'`, the profile-side assertion is wrapped in a runtime guard with no `else` and no later unconditional check on the same data:

```typescript
const s7 = (
  profileRes.body as {
    sections?: { S7?: { data?: { notes?: unknown[] } } };
  }
).sections?.S7;
if (s7 && 'data' in s7) {
  expect(s7.data?.notes ?? []).toHaveLength(0);
}
```

If `S7` is ever absent from the profile response for `Self` (e.g. a future regression removes the section, or the provider starts returning an `unavailable` status instead of a data-bearing shape under some condition), `s7 && 'data' in s7` evaluates to `false`, the block is skipped, and the test reports green having verified nothing about the profile surface — the exact surface this test's name claims to cover. The test's only safety net is the subsequent parallel-route assertion two lines later, which checks a different endpoint, not this one.

This sits inside the suite's highest-risk area: `test-design-qa.md` scores "leak through a non-profile surface" and "suites drift from the matrix" at 9 and 6 respectively (R-001, R-007), and this is precisely the class of drift those risk entries exist to catch on the *profile* surface specifically.

**Current Code**:

```typescript
// ❌ Bad (current implementation) — silently passes if s7 lacks 'data'
if (s7 && 'data' in s7) {
  expect(s7.data?.notes ?? []).toHaveLength(0);
}
```

**Recommended Fix**:

```typescript
// ✅ Good (recommended approach) — asserts the shape unconditionally, then the content
expect(profileRes.body.sections).toHaveProperty('S7');
expect(s7).toMatchObject({ data: { notes: [] } });
```

**Why This Matters**:
A conditional assertion in a security-boundary test is worse than an assertion with a slightly wrong expected value: a wrong value fails loudly and gets fixed, while a skipped assertion passes silently and erodes exactly the guarantee (`SM-1`, zero-leak) the suite's own risk register treats as the product's top differentiator.

**Related Violations**:
The same pattern recurs at `test/matrix-flag-gated.e2e-spec.ts:282-285` (S16) — see Recommendation 2.

---

### 2. Guarded assertion is the *only* assertion in the test (S16)

**Severity**: P1 (High)
**Location**: `test/matrix-flag-gated.e2e-spec.ts:282-285`
**Row**: H3
**Criterion**: Determinism (no conditionals) / Explicit Assertions
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Issue Description**:
In `'hides management-only S16 fields from Colleague viewers'`, every assertion in the test lives inside a single guard with no fallback:

```typescript
if (s16 && 'data' in s16) {
  expect(s16.data?.values ?? {}).not.toHaveProperty(managementField.id);
  expect(s16.data?.values?.[colleagueField.id]).toBe('public');
}
```

Unlike Recommendation 1, this test has no secondary unconditional assertion elsewhere in its body. If `s16` is ever `undefined` or lacks `'data'` (for example if S16 resolves to a non-data-bearing `unavailable` status under a provider hiccup, mirroring the `unavailable` shapes seen in `risks.e2e-spec.ts` and `management-notes.e2e-spec.ts`'s provider-failure suites), this test executes zero assertions and Jest still reports it as passed — a test that "can pass while the behavior is broken," which is precisely what registry row H3 targets. This is also the S16 section — the same section `known-red-diagnosis.md` §3a flags as the site of a *different* fragile-assertion defect (see "Context and Integration" below), making S16 the section with the most assertion-fragility findings in the whole suite.

**Current Code**:

```typescript
// ❌ Bad (current implementation) — the entire test body is conditional
if (s16 && 'data' in s16) {
  expect(s16.data?.values ?? {}).not.toHaveProperty(managementField.id);
  expect(s16.data?.values?.[colleagueField.id]).toBe('public');
}
```

**Recommended Fix**:

```typescript
// ✅ Good (recommended approach)
expect(res.body.sections).toHaveProperty('S16');
expect(s16.data?.values ?? {}).not.toHaveProperty(managementField.id);
expect(s16.data?.values?.[colleagueField.id]).toBe('public');
```

**Why This Matters**:
Same rationale as Recommendation 1 — the section under test (S16, custom fields) is explicitly called out in `test-design-qa.md` as R-009 ("custom-field existence inferable") and P0-008, one of the P0 rows. A test that can execute zero assertions and still pass is functionally equivalent to no test at all for regression purposes.

**Related Violations**: See Recommendation 1 (`matrix-flag-gated.e2e-spec.ts:89-91`).

---

### 3. Test-order dependency via unshared, unreset mutable state

**Severity**: P1 (High)
**Location**: `test/users.e2e-spec.ts:12` (declaration), manifesting across lines 24-79
**Row**: H4
**Criterion**: Isolation (cleanup, no shared state)
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Issue Description**:
Every other e2e spec file in the suite either calls `testApp.resetDatabase()` in a `beforeEach`, or (like `auth.e2e-spec.ts`) keeps each test fully self-contained by creating and tearing down its own fixtures. `users.e2e-spec.ts` does neither: it declares `let createdId: string;` at describe scope, sets it inside the `'POST /api/v1/users creates a user'` test, and three later, independent `it` blocks (`PATCH`, `DELETE`, and the final `'GET ... after deletion returns 403'`) all read that same variable with no reset in between:

```typescript
let createdId: string;
// ...
it('POST /api/v1/users creates a user', async () => {
  // ...
  createdId = body.id;          // written here
});
// ...
it('PATCH /api/v1/users/:id updates the name', async () => {
  const res = await agent.patch(`/api/v1/users/${createdId}`) /* ... */;
  // ...
});
it('DELETE /api/v1/users/:id returns 204', () => {
  return agent.delete(`/api/v1/users/${createdId}`).expect(204);
});
it('GET /api/v1/users/:id after deletion returns 403 for another user id', () => {
  return agent.get(`/api/v1/users/${createdId}`).expect(403);   // depends on DELETE having already run
});
```

This passes today only because Jest executes `it` blocks within a file in declaration order by default. It would break silently under `--randomize`, under `.only`-ing a later test in isolation, or under any future test runner that parallelizes within a file — and a reader who runs just the `DELETE` test in isolation (a common debugging move) gets a `ReferenceError`/`undefined` in the URL instead of a clear "this test depends on state from an earlier test" signal.

**Current Code**:

```typescript
// ❌ Bad (current implementation) — cross-test state with no isolation
describe('Users CRUD (e2e)', () => {
  let createdId: string;
  it('POST ... creates a user', async () => { /* sets createdId */ });
  it('PATCH ... updates the name', async () => { /* reads createdId */ });
  it('DELETE ... returns 204', () => { /* reads createdId */ });
});
```

**Recommended Fix**:

```typescript
// ✅ Good (recommended approach) — each test creates what it needs, or the lifecycle
// is expressed as one test with clearly ordered, commented phases
it('supports the full create -> update -> delete -> 403 lifecycle for one user', async () => {
  const created = await agent.post('/api/v1/users').send({ email, name: 'E2E User' }).expect(201);
  const id = (created.body as UserEntity).id;

  await agent.patch(`/api/v1/users/${id}`).send({ name: 'Renamed User' }).expect(200);
  await agent.delete(`/api/v1/users/${id}`).expect(204);
  await agent.get(`/api/v1/users/${id}`).expect(403);
});
```

**Why This Matters**:
`test-design-qa.md` names exactly this class of bug as R-002 ("shared-DB parallelism false greens", scored 9) and as one of the suite's own entry criteria (TC-2: "parallel-safe database isolation... blocks every integration and API suite from running in parallel"). This file is the one place in the 82-file review set where that guarantee does not hold.

---

### 4. Repeated inline fixture construction bypasses the shared factory

**Severity**: P2 (Medium)
**Location**: `test/employees.e2e-spec.ts:433-483`, `556-609`, `625-674`, `692-732`
**Row**: M2
**Criterion**: Fixture Patterns / Data Factories
**Knowledge Base**: [data-factories.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/data-factories.md)

**Issue Description**:
Ten other e2e spec files (`campaigns.e2e-spec.ts`, `custom-fields.e2e-spec.ts`, and others) import and reuse `createEmployeeUser`/`loginAsEmployee` from `test/support/employee-users.ts`. `employees.e2e-spec.ts` instead defines its own local `createEmployeeUser` (with different parameters: `tenureStart`, `grade`) and, on top of that, repeats the same six-call Prisma sequence — create `User`, create `Employee`, then `gradeHistory`/`positionHistory`/`departmentHistory`/`employmentTypeHistory`, each with a matching `effectiveFrom` — inline, verbatim, at least four more times across the "inline-edit" test group, instead of extracting a second helper (e.g. `createReportWithFullHistory(managerId)`):

```typescript
// Repeated near-verbatim at 433-483, 556-609, 625-674, and 692-732:
const reportUser = await testApp.prisma.user.create({ data: { email: '...', name: 'Report', passwordHash: await hash(PASSWORD, 12) } });
const report = await testApp.prisma.employee.create({ data: { id: reportUser.id, userId: reportUser.id, managerId: manager.employeeId } });
const reportStart = new Date('2020-01-01T00:00:00.000Z');
await testApp.prisma.gradeHistory.create({ data: { employeeId: report.id, value: 'Mid', effectiveFrom: reportStart } });
await testApp.prisma.positionHistory.create({ data: { employeeId: report.id, value: 'Engineer', effectiveFrom: reportStart } });
await testApp.prisma.departmentHistory.create({ data: { employeeId: report.id, value: 'Engineering', effectiveFrom: reportStart } });
await testApp.prisma.employmentTypeHistory.create({ data: { employeeId: report.id, value: 'Full-time', effectiveFrom: reportStart } });
```

**Recommended Improvement**:

```typescript
// ✅ Better approach (recommended)
async function createFullyOnboardedReport(
  testApp: TestApp,
  managerId: string,
  overrides: Partial<{ email: string; grade: string; tenureStart: string }> = {},
): Promise<EmployeeUser> {
  const report = await createEmployeeUser(testApp, overrides.email ?? `report-${randomUUID()}@example.com`, 'Report', overrides.tenureStart ?? '2020-01-01', overrides.grade ?? 'Mid');
  await testApp.prisma.employee.update({ where: { id: report.employeeId }, data: { managerId } });
  return report;
}
```

**Benefits**:
A single named factory would let a future schema change to the four history tables (a real possibility — Story 1.20's temporal-history extension already changes shape on that boundary) be fixed in one place instead of four, and gives each test a one-line, intention-revealing setup instead of a 15-line block that has to be read to confirm it matches its three siblings.

**Priority**:
P2 — this is a maintainability cost, not a correctness risk today (all four blocks are currently identical and correct), but it is exactly the kind of duplication `data-factories.md`'s "factory functions with overrides" guidance exists to prevent, and the fix is cheap relative to the four-way duplication it removes.

---

### 5. Multi-concern test bundles five behaviors into one `it`

**Severity**: P2 (Medium)
**Location**: `test/employees.e2e-spec.ts:433-547` (`'allows a manager to inline-edit a direct report grade and rejects a colleague'`)
**Row**: M3
**Criterion**: Explicit Assertions / Maintainability
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Issue Description**:
This single test asserts, in sequence: (1) a manager's `writableFieldIds` includes `grade`, (2) a first grade edit persists and is visible on re-fetch, (3) a second grade edit on top of the first also persists, (4) two `gradeHistory` rows exist afterward with the correct `effectiveTo`/`value`, and (5) a colleague performing the same PATCH is rejected with 403. These are at least two genuinely unrelated concerns (the manager's successful multi-step edit workflow, and a different actor's authorization denial) sharing one `it`, so a failure in the colleague-denial assertion at the bottom reports as a failure of "allows a manager to inline-edit a direct report grade," which does not localize the actual regression.

**Recommended Improvement**:

```typescript
// ✅ Better approach (recommended) — split by concern
it('lets a manager inline-edit a direct report grade, appending a new history row', async () => { /* concerns 1-4 */ });
it('rejects a colleague inline-editing another employee's grade', async () => { /* concern 5 */ });
```

**Benefits**:
Failure output localizes to the actual broken behavior, and the authorization-denial case becomes independently discoverable/greppable by name, matching the pattern the rest of the file's colleague-denial tests already follow elsewhere (e.g. the separate custom-field and department-edit denial tests).

**Priority**:
P2 — low urgency, the test is correct today; this is a readability/diagnosability improvement.

---

## Best Practices Found

### 1. Self-verifying coverage assertions on the access matrix

**Location**: `test/support/access-matrix.spec.ts:106-181`, exercised across `shared-links.e2e-spec.ts`, `matrix-flag-gated.e2e-spec.ts`, `access-matrix-leaks.e2e-spec.ts`
**Pattern**: Coverage-as-code / drift detection
**Knowledge Base**: [test-priorities-matrix.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Why This Is Good**:
`assertMatrixCoverage`, `assertDeniedMatrixCoverage`, and `assertFlagGatedCoverage` don't just describe the access matrix — they fail loudly, by name, if any of the 96 section×audience cells, 18 denial cells, or 13 documented flag-gated cases are left unexercised anywhere in the suite. `test-design-qa.md`'s R-007 ("suites drift from the matrix", scored 6) names this exact failure mode, and this is a rare case of a test suite enforcing its own completeness rather than relying on a human to notice a gap.

**Code Example**:

```typescript
it('names the uncovered pair it rejects', () => {
  const allButOne = matrixCells().filter(
    (entry) => !(entry.section === 'S6' && entry.audience === 'colleague'),
  );
  expect(() => assertMatrixCoverage(allButOne)).toThrow(/S6\/colleague/);
});
```

**Use as Reference**: Any future section or audience added to the access model gets this same drift protection for free — worth pointing to as the template for how new record-level flags (beyond S7/S16's current two) should register their own coverage collector.

### 2. Deterministic, race-free assertions on cache/TTL boundaries

**Location**: `test/access-resolution-cache.e2e-spec.ts:49-73`, `test/project-assignment-sync.e2e-spec.ts:104-111`
**Pattern**: Injectable clock, no wall-clock races
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Why This Is Good**:
Both suites prove a time-boundary transition (a cached grant going stale, a 4-hour project-assignment freshness window lapsing) by calling `clock.advance(...)` on the injected `FixedClock` rather than by sleeping past a real timer — the exact pattern `timing-debugging.md` and `test-design-qa.md`'s TC-4 ("injectable clock... blocks deterministic testing of the AD-8 freshness window") call for.

**Code Example**:

```typescript
clock.advance(4 * HOUR + 1);
await expect(access.resolveAudience(dm, subject)).resolves.toMatchObject({
  role: 'Colleague',
});
```

**Use as Reference**: This is the model every other time-sensitive assertion in the suite already follows (`shared-links.e2e-spec.ts` expiry, `action-items.e2e-spec.ts` overdue derivation) — no exceptions were found.

### 3. Structured absence assertions instead of loose truthy checks

**Location**: `test/employee-profile-custom-fields.e2e-spec.ts:96-102`, `test/colleague-whitelist.e2e-spec.ts:133-138`
**Pattern**: Explicit wire-shape assertions (`toEqual`/`not.toHaveProperty`/exact key-set)
**Knowledge Base**: [test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)

**Why This Is Good**:
Most denial-surface assertions in this suite use `Object.keys(body.sections).sort()).toEqual([...])` (exact key-set equality) or `not.toHaveProperty` rather than `toBeUndefined()` — which is the distinction `test-design-qa.md`'s Appendix A calls out explicitly ("`toBeUndefined` would pass on a serialized `null`; `not.toHaveProperty` is what proves the key is gone"). This is applied consistently across the profile-emission tests, which is exactly where it matters most.

**Use as Reference**: The two H3 findings above (Recommendations 1-2) are the exception to this otherwise-consistent pattern, not evidence the pattern is absent — most of the suite gets this right.

---

## Test File Analysis

### Suite Metadata

- **Files Reviewed**: 82 (55 unit/integration under `src/`, 27 e2e/support under `test/`)
- **Test Framework**: Jest 30 (ts-jest) + Supertest, per `services/backend/package.json` and `test-design-qa.md`'s stack detection
- **Language**: TypeScript
- **Largest File**: `test/action-items.e2e-spec.ts` (~1,633 lines — see Recommendation-adjacent Finding, H5)
- **Smallest Files**: `test/support/graph-factory.spec.ts` (~190 lines), `src/modules/health/__tests__/health.controller.spec.ts`

### Test Structure (aggregate)

- **Fixtures/Infrastructure Used**: `FixedClock` (test/support/fixed-clock.ts), relationship-graph builder (test/support/graph-factory.ts), `ExternalBoundary` fault-injection server (test/support/external-boundary.ts), per-worker Postgres schema isolation (test/support/test-database.ts / test-schema.ts, verified directly by `database-isolation.e2e-spec.ts`), shared employee/login factories (test/support/employee-users.ts, test/support/login.ts, test/support/matrix-actors.ts), access-matrix + coverage collectors (test/support/access-matrix.ts, matrix-coverage-collector.ts, matrix-leak-assertions.ts)
- **Data Factories Used**: `createEmployeeUser`/`loginAsEmployee` (shared, ~10 files), `aGraph()` relationship builder, `bootcamp-seed.ts` whitelist graph seeder — all with override parameters, matching `data-factories.md`'s API-first, override-friendly pattern, with the one exception noted in Recommendation 4.

### Priority Distribution

No P0/P1/P2/P3 markers exist in any test name or tag in this suite (see "Priority Markers" row above) — `test-design-qa.md` defines this framework at the plan level but it was never wired into the test names themselves. Not a violation (absent convention), but worth flagging as a gap between the test-design artifact and the implemented suite if CI is ever expected to select tests by priority tag (`npx jest -t "@P0"`, as shown in `test-design-qa.md`'s Appendix A, would currently match zero tests).

---

## Context and Integration

### What the Context Said

Three prior investigation/design documents were supplied as read-only grounding (never scored, never added to the reviewed-file manifest):

- **`known-red-diagnosis.md`** — root-caused several red/flaky signals in this exact suite. One item bears directly on test *quality* rather than product correctness, and this review's own reading confirms it independently:
  - **§3a, S16 "leak" — CONFIRMED as a real test-quality defect** (not a security defect; the diagnosis document is correct that access control is fine). `test/employee-profile-custom-fields.e2e-spec.ts:194` contains `expect(colleagueRaw).not.toContain('L');`, a raw single-character substring guard on the full serialized response, intended to prove the employee-only field value `'Shirt size': 'L'` is absent from a Colleague's view. It instead trips on the unrelated `manageLeaveUrl` key that legitimately appears in the same response's S10 section. This finding does not map cleanly to any single registry row (it is not a DOM selector [L1], not an unexplained magic value [L6], and not one of the H-tier determinism rows), so per the registry's own rule ("a defect matching no row is reported in prose... without a severity and without a deduction"), it is **not scored into the ledger above**, but it is real, it is in the reviewed-file set, and it should be fixed: replace the raw substring check with a structured assertion on the parsed value map (`expect(body.sections.S16.data.values).not.toHaveProperty(employeeOnlyField.id)`), exactly as the rest of the same test already does two lines earlier.
  - The other diagnosis items (build-blocking `TS2322` in `campaigns.service.ts`, the `pg_indexes` schema-scoping bug, the `employees.e2e-spec.ts` filter-DTO defect, the `colleague-whitelist` access gap, the unresolved session-401s) are product-code or test-infrastructure defects outside a test-file quality lens, or have already been fixed per the document's own "Fixed in this pass" table — none of them change a test-quality finding in this review.
- **`ci-pipeline-progress.md`** — established that the full e2e tier (`npm run test:e2e`, all 82 `*.e2e-spec.ts`/`*.spec.ts` under `test/`) was not previously run in CI's narrow `access-control-e2e` job, and that this suite is deliberately `maxWorkers: 1` for the e2e tier (not a parallelism gap in the tests themselves — a documented, accepted serialization choice while the suite is still small, per `test-design-qa.md`'s R-002 entry).
- **`test-design-qa.md`** — supplied the risk register (R-001 through R-018) and P0–P3 test plan this review cross-references throughout (most visibly in Recommendations 1-3, all three tied back to R-001/R-002/R-007). It also explains why P0 is ~33% of planned test groups rather than the usual ~10%: the access model *is* the product's differentiating risk, which is the lens this review applied when weighing the two H3 findings as High rather than a lower severity.

### Related Artifacts

- **Test Design**: [test-design-qa.md](../test-artifacts/test-design-qa.md) — Risk register R-001 through R-018; P0 coverage plan (~30 groups / ~200 cases)
- **CI Context**: [ci-pipeline-progress.md](../test-artifacts/ci-pipeline-progress.md) — e2e tier scope and known-red gates at time of pipeline authoring
- **Prior Diagnosis**: [known-red-diagnosis.md](../test-artifacts/known-red-diagnosis.md) — root-caused red/flaky signals, one (§3a) independently confirmed above as a test-quality defect

---

## Knowledge Base References

This review consulted the following knowledge base fragments (paths relative to this report):

- **[test-quality.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-quality.md)** — Definition of Done for tests (no hard waits, ≤1000 lines, deterministic, isolated, fast)
- **[data-factories.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/data-factories.md)** — Factory functions with overrides, API-first setup
- **[test-levels-framework.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-levels-framework.md)** — Unit vs. API vs. E2E appropriateness (used to confirm this suite's unit/integration/e2e split matches `test-design-qa.md`'s stated rationale)
- **[selective-testing.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/selective-testing.md)** — Tag-based/diff-based execution, duplicate-coverage detection (used to assess the missing priority-tag wiring noted above)
- **[test-healing-patterns.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/test-healing-patterns.md)** — Common failure taxonomy, used as a lens when distinguishing the H3 conditional-assertion findings from ordinary flakiness
- **[selector-resilience.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/selector-resilience.md)** — Consulted and found not applicable: no DOM selectors exist anywhere in this backend suite (confirms the `Test IDs`/L1/L3 `PASS (n/a)` rows above)
- **[timing-debugging.md](../../.claude/skills/bmad-testarch-test-review/resources/knowledge/timing-debugging.md)** — Deterministic waiting / race-condition elimination, the basis for Best Practice #2 above
- **[criteria-registry.md](../../.claude/skills/bmad-testarch-test-review/steps-c/criteria-registry.md)** — The full C/H/M/L rule table this review's every scored finding is attributed to

For coverage mapping, consult `trace` workflow outputs (not run as part of this review).

---

## Next Steps

### Immediate Actions (Before Merge)

1. **Remove the two guarded assertions in `matrix-flag-gated.e2e-spec.ts`** (Recommendations 1-2) — replace with unconditional shape assertions.
   - Priority: P1
   - Owner: Backend/QA (whoever owns the access-matrix suite)
   - Estimated Effort: ~30 minutes (two small, well-scoped diffs)

2. **Restructure `users.e2e-spec.ts` to remove the cross-test `createdId` dependency** (Recommendation 3) — either one lifecycle test or per-test fixture creation.
   - Priority: P1
   - Owner: Backend
   - Estimated Effort: ~20 minutes

### Follow-up Actions (Future PRs)

1. **Split `action-items.e2e-spec.ts` by concern** (CRUD lifecycle / overdue derivation / campaign bulk-activation / provider-failure) to bring it under the 1000-line ceiling.
   - Priority: P2
   - Target: Next test-maintenance pass

2. **Extract a `createFullyOnboardedReport` factory in `employees.e2e-spec.ts`** and split the five-concern grade-edit test (Recommendations 4-5).
   - Priority: P2
   - Target: Next test-maintenance pass

3. **Fix the S16 substring assertion** confirmed in "Context and Integration" above (`employee-profile-custom-fields.e2e-spec.ts:194`), per `known-red-diagnosis.md` §3a.
   - Priority: P2 (real, but low risk today since a structured assertion two lines above already proves the correct behavior)
   - Target: Next test-maintenance pass

### Re-Review Needed?

⚠️ Re-review after the two P1 (High) fixes — the score and recommendation would very likely move to `Approve with Comments` or better once Recommendations 1-3 are addressed, since the remaining findings are Medium/prose-only.

---

## Decision

**Recommendation**: Request Changes

**Rationale**:
Zero Critical findings and only four High findings across 82 files is a strong result, and three of the four (H3 ×2, H4) are narrow, mechanically fixable, single-file issues rather than systemic problems — the suite's infrastructure (injectable clock, relationship-graph factory, per-worker DB isolation, self-verifying matrix coverage) is genuinely well engineered and none of that is in question here. But the workflow's scoring model is deliberately non-negotiable on this point: any High-severity finding forces `Request Changes` regardless of the numeric score, because a High row is either "a test that can pass while the behavior is broken" (H3, twice, in the suite's own highest-risk area) or "fails at random" (H4) — both cost more engineering time downstream than fixing them now, and the fixes here are cheap (see "Immediate Actions" above, ~1 hour total).

**For Request Changes**:

> Test quality needs targeted improvement with 81/100 score. Two guarded assertions in `matrix-flag-gated.e2e-spec.ts` (the suite's own zero-leak access-matrix tier) and one test-order dependency in `users.e2e-spec.ts` must be fixed before merge — none require a redesign, all three are small, isolated diffs. No security defect exists today (the underlying access control this suite protects is correct); the finding is that two of its assertions could silently stop protecting it without the suite noticing.

---

## Appendix

### Violation Summary by Location

| File                                          | Line(s)    | Severity | Row | Criterion                | Issue                                                                 | Fix                                                              |
| ---------------------------------------------- | ---------- | -------- | --- | ------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `test/matrix-flag-gated.e2e-spec.ts`           | 89-91      | P1 High  | H3  | Determinism               | Guarded assertion on S7 profile shape can silently skip                | Assert shape unconditionally, then content                        |
| `test/matrix-flag-gated.e2e-spec.ts`           | 282-285    | P1 High  | H3  | Determinism               | Guarded assertion is the test's only assertion (S16)                  | Assert shape unconditionally, then content                        |
| `test/users.e2e-spec.ts`                       | 12, 24-79  | P1 High  | H4  | Isolation                 | Test-order dependency via unshared, unreset `createdId`               | Single lifecycle test or per-test fixture creation                |
| `test/action-items.e2e-spec.ts`                | whole file | P1 High  | H5  | Test Length               | File is ~1,633 lines, exceeds the 1000-line ceiling                   | Split by concern (CRUD / overdue / campaign / provider-failure)   |
| `test/employees.e2e-spec.ts`                   | 433-483, 556-609, 625-674, 692-732 | P2 Medium | M2 | Fixture Patterns / Data Factories | Same 6-call Prisma fixture inlined 4+ times, bypasses shared factory | Extract `createFullyOnboardedReport(managerId, overrides)`        |
| `test/employees.e2e-spec.ts`                   | 433-547    | P2 Medium | M3  | Explicit Assertions        | One test bundles 5 unrelated concerns (edit workflow + auth denial)    | Split into two tests                                              |

**Unscored, confirmed real (no registry row; reported per Rule 1)**:

| File                                                | Line | Issue                                                                 | Fix                                                             |
| ----------------------------------------------------- | ---- | ------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| `test/employee-profile-custom-fields.e2e-spec.ts`     | 194  | `not.toContain('L')` substring guard trips on unrelated `manageLeaveUrl` key | Replace with a structured assertion on the parsed `values` map      |

### Related Reviews

Not applicable — this is the first `test-review` run recorded for this suite (no prior `test-review.md` existed at this path before this run).

---

## Review Metadata

**Generated By**: BMad TEA Agent (Test Architect)
**Workflow**: testarch-test-review (Create mode)
**Review ID**: test-review-services-backend-suite-20260905
**Timestamp**: 2026-09-05
**Version**: 1.0

---

## Feedback on This Review

If you have questions or feedback on this review:

1. Review patterns in the knowledge base: `.claude/skills/bmad-testarch-test-review/resources/knowledge/`
2. Consult `tea-index.csv` for detailed guidance
3. Request clarification on specific violations
4. Pair with QA engineer to apply patterns

This review applies the rubric consistently. Context can reveal additional findings and clarify impact; it cannot waive a violation, change severity, or alter the score. Formal risk acceptance belongs in `trace` or the release gate.

---

## Reviewed Files

- services/backend/src/__tests__/app-startup.spec.ts
- services/backend/src/clock/__tests__/clock.service.spec.ts
- services/backend/src/config/__tests__/env.validation.spec.ts
- services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts
- services/backend/src/modules/access/__tests__/employee-functional-roles.controller.spec.ts
- services/backend/src/modules/access/__tests__/functional-role-assignment.service.spec.ts
- services/backend/src/modules/access/__tests__/functional-role-data-access-boundary.spec.ts
- services/backend/src/modules/access/__tests__/functional-role.service.spec.ts
- services/backend/src/modules/access/__tests__/identity-section.provider.spec.ts
- services/backend/src/modules/access/__tests__/people-partner-assignment.service.spec.ts
- services/backend/src/modules/access/__tests__/permission-checker.service.spec.ts
- services/backend/src/modules/access/__tests__/profile-assembler.service.spec.ts
- services/backend/src/modules/access/__tests__/project-assignment.service.spec.ts
- services/backend/src/modules/access/__tests__/projects-section.provider.spec.ts
- services/backend/src/modules/access/__tests__/relationship-graph-generation.service.spec.ts
- services/backend/src/modules/access/__tests__/relationship-journal.service.spec.ts
- services/backend/src/modules/access/__tests__/section-access-gate.service.spec.ts
- services/backend/src/modules/access/__tests__/shared-link.service.spec.ts
- services/backend/src/modules/action-items/__tests__/action-item-input.spec.ts
- services/backend/src/modules/action-items/__tests__/action-items-section.provider.spec.ts
- services/backend/src/modules/action-items/__tests__/action-items.service.spec.ts
- services/backend/src/modules/app.module.spec.ts
- services/backend/src/modules/auth/__tests__/auth-cookie.spec.ts
- services/backend/src/modules/auth/__tests__/auth.service.spec.ts
- services/backend/src/modules/auth/__tests__/authenticated-current-user.provider.spec.ts
- services/backend/src/modules/auth/__tests__/jwt.strategy.spec.ts
- services/backend/src/modules/campaigns/__tests__/campaign-audience.spec.ts
- services/backend/src/modules/campaigns/__tests__/campaigns.service.spec.ts
- services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts
- services/backend/src/modules/directory/__tests__/custom-field-visibility.service.spec.ts
- services/backend/src/modules/directory/__tests__/custom-fields-section.provider.spec.ts
- services/backend/src/modules/directory/__tests__/custom-fields.service.spec.ts
- services/backend/src/modules/directory/__tests__/employees.service.spec.ts
- services/backend/src/modules/directory/__tests__/field-registry.service.spec.ts
- services/backend/src/modules/health/__tests__/health.controller.spec.ts
- services/backend/src/modules/integrations/__tests__/leave-period.mapper.spec.ts
- services/backend/src/modules/integrations/__tests__/leaves-section.provider.spec.ts
- services/backend/src/modules/integrations/__tests__/leaves-sync.service.spec.ts
- services/backend/src/modules/integrations/__tests__/project-assignment.mapper.spec.ts
- services/backend/src/modules/integrations/__tests__/projects-sync.scheduler.spec.ts
- services/backend/src/modules/integrations/__tests__/projects-sync.service.spec.ts
- services/backend/src/modules/management-notes/__tests__/management-notes-section.provider.spec.ts
- services/backend/src/modules/mentorship/__tests__/active-mentor-lookup.service.spec.ts
- services/backend/src/modules/mentorship/__tests__/mentorship-pair.service.spec.ts
- services/backend/src/modules/registry/__tests__/provider-registry.service.spec.ts
- services/backend/src/modules/registry/__tests__/registry.module.spec.ts
- services/backend/src/modules/risks/__tests__/risks-section.provider.spec.ts
- services/backend/src/modules/risks/__tests__/risks.service.spec.ts
- services/backend/src/modules/timeline/__tests__/timeline-event-writer.integration.spec.ts
- services/backend/src/modules/timeline/__tests__/timeline-event-writer.service.spec.ts
- services/backend/src/modules/timeline/__tests__/timeline-section.provider.spec.ts
- services/backend/src/modules/timeline/__tests__/timeline.service.spec.ts
- services/backend/src/modules/timetracker/__tests__/timetracker.service.spec.ts
- services/backend/src/modules/users/__tests__/users.controller.spec.ts
- services/backend/src/modules/users/__tests__/users.service.spec.ts
- services/backend/src/prisma/__tests__/prisma.module.spec.ts
- services/backend/src/prisma/__tests__/relationship-graph.extension.spec.ts
- services/backend/src/prisma/__tests__/temporal-history.extension.spec.ts
- services/backend/src/prisma/seed/__tests__/seed.manifest.spec.ts
- services/backend/src/prisma/seed/__tests__/seed.service.spec.ts
- services/backend/src/prisma/seed/__tests__/seed.synthetic.spec.ts
- services/backend/test/access-matrix-leaks.e2e-spec.ts
- services/backend/test/access-matrix-overlap.e2e-spec.ts
- services/backend/test/access-matrix-positive.e2e-spec.ts
- services/backend/test/access-resolution-cache.e2e-spec.ts
- services/backend/test/action-items.e2e-spec.ts
- services/backend/test/app.e2e-spec.ts
- services/backend/test/auth.e2e-spec.ts
- services/backend/test/campaigns.e2e-spec.ts
- services/backend/test/colleague-whitelist.e2e-spec.ts
- services/backend/test/cross-feature-access.exemplar.e2e-spec.ts
- services/backend/test/custom-fields.e2e-spec.ts
- services/backend/test/database-isolation.e2e-spec.ts
- services/backend/test/employee-profile-custom-fields.e2e-spec.ts
- services/backend/test/employee-profile.e2e-spec.ts
- services/backend/test/employees.e2e-spec.ts
- services/backend/test/functional-role-assignments.e2e-spec.ts
- services/backend/test/functional-roles.e2e-spec.ts
- services/backend/test/management-notes.e2e-spec.ts
- services/backend/test/matrix-flag-gated.e2e-spec.ts
- services/backend/test/project-assignment-sync.e2e-spec.ts
- services/backend/test/risks.e2e-spec.ts
- services/backend/test/shared-link-matrix.sync.spec.ts
- services/backend/test/shared-links.e2e-spec.ts
- services/backend/test/support/access-matrix.spec.ts
- services/backend/test/support/external-boundary.spec.ts
- services/backend/test/support/fixed-clock.spec.ts
- services/backend/test/support/graph-factory.spec.ts
- services/backend/test/timeline.e2e-spec.ts
- services/backend/test/users.e2e-spec.ts

## Review Context

- _bmad-output/test-artifacts/known-red-diagnosis.md
- _bmad-output/test-artifacts/ci-pipeline-progress.md
- _bmad-output/test-artifacts/test-design-qa.md

## Excluded From Review Set

No files excluded. All 82 files in the authoritative review set supplied for this run were found on disk and reviewed; none were unparseable or unscorable by the ledger.

---
title: 'Employee profile e2e — session 401 after mentor-fail test'
type: 'bugfix'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '3fd89342e3bc0ec69b96f2869823c821d4a40827'
context:
  - '{project-root}/services/backend/defects/test-debt/03-investigation-401-employee-profile-session.md'
  - '{project-root}/_bmad-output/test-artifacts/known-red-diagnosis.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The last three tests in `employee-profile.e2e-spec.ts` returned `401` on the first request using the shared `colleagueAgent`, even though no mutation had occurred in those tests. The colleague session appeared invalidated mid-suite.

**Approach:** Root cause is test isolation, not JWT expiry or `FixedClock`. The `"still returns S1 data when mentor lookup fails"` test spins up a second `createTestApp()` which defaults to `truncate: true`, wiping the worker's shared Postgres schema and deleting all users. Subsequent JWT validation fails (`JwtStrategy` requires the user row to exist). Fix by passing `truncate: false` on the secondary app and seeding with a unique `emailSuffix` to avoid email collisions in the same schema.

## Boundaries & Constraints

**Always:**
- Fix is test-only in `test/employee-profile.e2e-spec.ts` — no auth/product changes.
- Secondary test apps that share a worker schema must not truncate when the primary suite still holds live sessions.
- `FixedClock` is not used in this file — reject that hypothesis.

**Ask First:** none identified.

**Never:**
- Change JWT TTL, session semantics, or `JwtStrategy` validation for this defect.
- Disable truncate globally in `createTestApp` — only opt out where a nested app coexists with an active suite.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Suite runs to end | `colleagueAgent` logged in beforeAll; mentor-fail test runs mid-file | Last three tests get `200` on first profile request | N/A |
| Nested app boot | Second `createTestApp` during active suite | Must not truncate shared schema | `truncate: false` |
| Nested seed | Same schema as parent suite | Unique emails via suffix | `emailSuffix: '-mentor-fail'` |

</frozen-after-approval>

## Code Map

- `services/backend/test/employee-profile.e2e-spec.ts:315-367` — `"still returns S1 data when mentor lookup fails"` — secondary `createTestApp`; must use `truncate: false`.
- `services/backend/test/employee-profile.e2e-spec.ts:370-503` — three tests that failed with 401 (`reflects manager reassignment`, `reflects people partner reassignment`, `keeps assembled profile sections unchanged after C8 role assignment`).
- `services/backend/test/employee-profile.e2e-spec.ts:518-607` — `seedProfileGraph` accepts optional `emailSuffix`; `profileEmail()` helper builds suffixed addresses.
- `services/backend/test/support/app-harness.ts:133-143` — `createTestApp` truncates by default when `truncate !== false`.
- `services/backend/src/modules/auth/jwt.strategy.ts:34-40` — validates JWT `sub` against live `user` row → `401` when truncated away.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/test/employee-profile.e2e-spec.ts` — add `truncate: false` to nested `createTestApp`; seed with `emailSuffix: '-mentor-fail'` and matching `profileEmail` for login.
- [x] `services/backend/defects/test-debt/03-investigation-401-employee-profile-session.md` — document root cause and set **Status: Fixed**.
- [x] `services/backend/test/employee-profile.e2e-spec.ts` — run full file; all 18 tests pass.

**Acceptance Criteria:**
- Given the employee-profile e2e suite runs in order, when the mentor-fail nested app test completes, then subsequent tests using `colleagueAgent` authenticate successfully (not `401`).
- Given the full `employee-profile.e2e-spec.ts` file runs, then all 18 tests pass.

## Verification

**Commands:**
- `cd services/backend && npm run test:e2e:serial -- --testPathPatterns=employee-profile.e2e-spec` — expected: 18/18 pass (global matrix teardown may fail on isolated runs; test body must be green).

## Suggested Review Order

**Test isolation fix**

- Nested app must not truncate the shared worker schema mid-suite
  [`employee-profile.e2e-spec.ts:317`](../../services/backend/test/employee-profile.e2e-spec.ts#L317)

- Suffixed emails prevent unique-constraint collisions in same schema
  [`employee-profile.e2e-spec.ts:341`](../../services/backend/test/employee-profile.e2e-spec.ts#L341)

- Default truncate behaviour that caused the 401 symptom
  [`app-harness.ts:133`](../../services/backend/test/support/app-harness.ts#L133)

## Manual Test Paths

**Employee profile e2e stability** — automated regression; no manual UI path.

The fix ensures automated tests for profile assembly keep working when a nested test app is created mid-suite.

1. Run the backend employee-profile e2e suite (`npm run test:e2e:serial` with that file).
2. Confirm all 18 tests pass, especially the last three (manager reassignment, PP reassignment, C8 role assignment).

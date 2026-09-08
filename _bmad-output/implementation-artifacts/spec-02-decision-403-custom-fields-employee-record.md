---
title: '403 on routes without Employee record — fixture alignment (Story 1.8)'
type: 'bugfix'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '3fd89342e3bc0ec69b96f2869823c821d4a40827'
context:
  - '{project-root}/services/backend/defects/test-debt/02-decision-403-custom-fields-employee-record.md'
  - '{project-root}/_bmad-output/test-artifacts/known-red-diagnosis.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Three e2e tests expected `200`/`404` but got `403` because `loginAsOperator` created a bare `User` without an `Employee` row. Story 1.8 requires a resolvable viewer employee on protected routes (`CustomFieldsController`, `UsersController`).

**Approach:** **Decision option 1** — keep product behavior (403 without employee record); align test fixtures and expectations. `loginAsOperator` seeds an `Employee` alongside the user; `auth.e2e-spec` expects `403` on `GET /users` for a user without employee and asserts own-user read via `GET /users/:id` instead.

## Boundaries & Constraints

**Always:**
- Story 1.8 colleague-whitelist hardening is correct — `colleague-whitelist.e2e-spec.ts` and `access-matrix-leaks` assert 403 for no-employee users.
- No API exemptions for definition-list routes without employee record.
- Test-only changes in `test/support/login.ts` and `test/auth.e2e-spec.ts`.

**Ask First:** none — option 1 matches existing access-matrix tests.

**Never:**
- Weaken `resolveViewerEmployeeId` or remove employee requirement from controllers.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior |
|----------|--------------|---------------------------|
| Operator with employee | `loginAsOperator` after fix | `GET /custom-fields` → `200` |
| Operator values, unknown employee | authenticated operator | `GET /custom-fields/values/:id` → `404` |
| Auth user without employee | login cookie, no employee row | `GET /users` → `403` |
| Auth user without employee | login cookie | `GET /users/:ownId` → `200` (self read) |

</frozen-after-approval>

## Code Map

- `services/backend/test/support/login.ts:10-15` — `employee: { create: {} }` on operator user seed.
- `services/backend/test/auth.e2e-spec.ts:86-96` — `GET /users` expects `403`; own-user `GET /users/:id` expects `200`.
- `services/backend/test/custom-fields.e2e-spec.ts:22-42` — passes once operator has employee.
- `services/backend/test/colleague-whitelist.e2e-spec.ts:304-310` — documents intended 403 for no-employee on custom-fields (unchanged).

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/test/support/login.ts` — add `employee: { create: {} }` to `loginAsOperator`.
- [x] `services/backend/test/auth.e2e-spec.ts` — expect `403` on list users; assert self read on `GET /users/:id`.
- [x] `services/backend/defects/test-debt/02-decision-403-custom-fields-employee-record.md` — record decision (option 1) and set **Fixed**.

**Acceptance Criteria:**
- Given `loginAsOperator`, when calling `GET /api/v1/custom-fields`, then `200`.
- Given authenticated user with no employee record, when calling `GET /api/v1/users`, then `403`.

## Verification

**Commands:**
- `cd services/backend && npm run test:e2e:serial -- --testPathPatterns="custom-fields.e2e-spec|auth.e2e-spec|users.e2e-spec"` — all tests in those files pass.

## Suggested Review Order

- Operator fixture now includes employee row for Story 1.8 compliance
  [`login.ts:14`](../../services/backend/test/support/login.ts#L14)

- Auth e2e matches no-employee 403 on user list; self read unchanged
  [`auth.e2e-spec.ts:86`](../../services/backend/test/auth.e2e-spec.ts#L86)

## Manual Test Paths

**Authenticated API access without employee profile** — regression covered by e2e.

1. Run custom-fields, auth, and users e2e suites.
2. Confirm operator-backed tests return success where expected and auth asserts 403 on user list without employee.

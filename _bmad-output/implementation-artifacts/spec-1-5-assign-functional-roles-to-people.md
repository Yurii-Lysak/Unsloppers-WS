---
title: 'Assign Functional Roles to People'
type: 'feature'
created: '2026-09-01'
status: 'in-progress'
review_loop_iteration: 1
baseline_commit: 'd7f1a6890f2652cc816a1baec72798a6b56afe6d'
frontend_baseline_commit: '4fb973bd5f4fab7b0d68a6204171167cddb6f1ed'
story_key: '1-5-assign-functional-roles-to-people'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-4-define-functional-roles-and-permissions-via-ui.md'
  - '{project-root}/_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
  - '{project-root}/services/backend/src/prisma/seed/data/bootcamp-identities.json'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.4 defines roles and permissions and persists `FunctionalRoleAssignment` rows via an internal helper only. HR Admin cannot assign roles to people through the UI, so permission grants never reach real users and UJ-3 step 3 is blocked.

**Approach:** Expose authenticated assignment APIs (read + **PUT replace** — transactional diff of `roleIds[]`) gated by `manage_functional_roles`, add a minimal All Employees → Employee Profile shell with an Employment-section **Functional roles** multi-select (UJ-3 step 3), and prove assignments take effect on the next C8 call without changing C1 data-access resolution. Add `GET /permissions/me` and migrate SPA nav gating (Admin → Roles and Campaigns) to that endpoint so UJ-3's climax is demonstrable.

## Boundaries & Constraints

**Always:**
- Assignment **read and write** APIs require `manage_functional_roles` via in-process C8 (same pattern as Story 1.4 admin routes). Resolve viewer via C7; 401 unauthenticated, 403 without permission. Non-holders do not see the profile assignment control **and** receive 403 on assignment GET/PUT — no read-only assignment field before Story 1.6 defines Employment-section visibility.
- An employee may hold **multiple** functional roles; `@@unique([employeeId, roleId])` enforced. **PUT replace:** client sends the full desired `roleIds[]`; server deduplicates IDs, diffs against existing rows, and applies add/remove idempotently inside one `$transaction`. Concurrent PUT on the same employee: **last write wins** (no optimistic concurrency in this story).
- Reject PUT that would leave **zero** employees holding a role that grants `manage_functional_roles` (mirror Story 1.4's built-in HR Admin PATCH guard at the assignment layer). Reject self-assignment paths that grant `manage_functional_roles` to the caller unless they already hold it.
- C8 `hasPermission(userId, key)` must reflect new assignments and revocations on the **very next call** — no cache (Story 1.4 invariant). Employee row with assignments but no linked `User` (or `userId` not resolvable): assignment rows persist; `hasPermission` → false; `GET /permissions/me` → `[]`.
- Functional roles **never widen C1 data access**. Assigning a role changes feature permissions only; `AccessResolver.resolveAudience(viewerId, subjectId)` for the same pair is unchanged by assignment. No C1 code changes in this story.
- Assignment UI lives on the employee profile **Employment section** per EXPERIENCE.md UJ-3 step 3 — not on Admin → Roles (that remains definition-only from 1.4).
- Minimal employee surfaces only: enough to navigate All Employees → one profile and edit functional roles. Do not assemble S1–S16 by access matrix (Story 1.6).
- **All Employees (pre-1.8):** `GET /employees` and `GET /employees/:id` require authentication only — no C1 access-matrix filtering yet (Story 1.8). All authenticated users may see the **All Employees** nav entry and browse id + displayName for every seeded employee; document this intentional gap in code comments. Bootcamp seed is 24 accounts — full list without pagination is acceptable; defer pagination to Story 1.8.
- `GET /permissions/me` returns the caller's union of **catalog-valid** permission keys from their assignments (orphan DB grant rows whose keys were removed from `contracts/permission-keys.ts` are **ignored**, per Story 1.4). Used for SPA nav gating; keys validated against the catalog only.
- SPA nav gating uses **`useMyPermissions`** (`GET /permissions/me`) for **both** Campaigns (`create_form_campaigns`) and Admin → Roles (`manage_functional_roles`). Do not retain the Story 1.4 list-fetch probe (`useFunctionalRolesAccess`) for nav — assignment changes must reflect on the next permissions fetch without requiring a roles-list round-trip. Invalidate/refetch permissions on window focus and after assignment mutations affecting the current user.

**Never:**
- Do not build campaign audience / All Employees filter engine (Epic 11) — epic AC about colleague-only filter fields is proven here via C8+C1 contract tests, not a live filter UI.
- Do not implement full profile assembly, section providers, or colleague whitelist enforcement on every surface (Stories 1.6, 1.8).
- Do not allow assignment writes that lock out all functional-role administrators — only holders of `manage_functional_roles` may write assignments; seed/bootstrap paths unchanged.
- Do not conflate functional PP role with C1 PP access — independent dimensions.
- Do not add Department admin, org-relationship writes, or full-access grant UI.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| List assignments | HR Admin `GET` for employee E | 200 + assigned roles (id, name, permissionKeys) | 401; 403 without `manage_functional_roles`; 404 unknown employee |
| Non-admin read | User without `manage_functional_roles` calls assignment GET | 403 | N/A |
| Replace assignments | HR Admin `PUT { roleIds: [R1,R2] }` on E | Transactional diff; 200 + same body as GET; C8 true for union of R1∪R2 permissions on E's user | 400 unknown/duplicate-handled role id; 404 unknown employee or role deleted before save; 403/401; 403 if last admin lockout |
| Duplicate roleIds in PUT | `PUT { roleIds: [R1,R1,R2] }` | Server dedupes before diff; same result as `[R1,R2]` | 400 optional if strict validation preferred — prefer dedupe |
| Idempotent replace | Same `roleIds` submitted twice | One row per pair; no duplicates | N/A |
| Clear all roles | `PUT { roleIds: [] }` | All assignment rows for E removed (unless last-admin lockout applies to caller's intent) | Same auth/404/403 lockout rules |
| Last admin lockout | PUT removes sole remaining `manage_functional_roles` holder | Rejected | 403 with clear message |
| Revoke effect | Remove role R from E; check `hasPermission(E.userId, key)` | false on next call for keys R granted | N/A |
| Multi-role union | E holds roles A and B with distinct permissions | C8 true if either role grants the key | N/A |
| Non-admin write | User without `manage_functional_roles` calls PUT | 403 | N/A |
| Data-access boundary | X assigned `create_edit_risks`; X has no ReportingLine/PP to B | `hasPermission(X, key)` true; `resolveAudience(X,B).sections.S6` still `none` (Colleague) | N/A |
| Unknown employee | Assign to missing `employeeId` | 404 | N/A |
| Malformed employeeId | Non-UUID path param | 400 | N/A |
| Unknown role in PUT | `roleIds` contains valid UUID not in DB | 404 with clear message | N/A |
| Invalid role UUID | `roleIds` contains malformed UUID | 400 | N/A |
| Employee without User | Assignments on employee with no linked user | Rows persist; C8 false; `/permissions/me` → `[]` for that user if they cannot log in | N/A |
| Employee delete | Employee with assignments | Blocked by FK Restrict (Story 1.4) | Prisma/deletion error until cleared |
| Permissions me | Authenticated user with assignments | 200 + catalog-valid permission key array only | 401; `[]` when no employee row |
| Permissions me orphans | DB grant row with key removed from catalog | Key omitted from response | N/A |
| List employees | Authenticated `GET /employees` | 200 + all employees (id, displayName) — no C1 filter yet | 401 |
| Employee detail | Authenticated `GET /employees/:id` | 200 + header shell fields | 401; 404; 400 malformed UUID |
| Nav gate | User gains `create_form_campaigns` via assignment | Campaigns nav visible on next `/permissions/me` fetch | Hidden when permission revoked; refetch on focus |
| Concurrent PUT | Two HR Admins PUT different roleIds on same E | Last write wins | N/A |
| Unauthenticated | No session on any new route | 401 | N/A |

</frozen-after-approval>

## Dependencies

- **Story 1.4** — `FunctionalRole`, `FunctionalRoleAssignment`, C8 impl, Admin → Roles UI, internal `FunctionalRoleAssignmentService`.
- **Story 1.18** — authenticated sessions; credentialed SPA requests; e2e login utilities.
- **Story 1.16** — seeded employees (`bootcamp-identities.json`, 24 accounts per `bootcamp-scope-overrides.md`) for All Employees list and assignment demos.

## Code Map

- `services/backend/src/modules/access/functional-role-assignment.service.ts` — extend with `setAssignments(employeeId, roleIds[])` transactional replace; keep `assign`/`unassign` for tests/seed.
- `services/backend/src/modules/access/permission-checker.service.ts` — already unions assignments; read-only for this story.
- `services/backend/src/modules/access/access-resolver.service.ts` — unchanged; use in boundary tests.
- `services/backend/src/modules/access/functional-roles.controller.ts` — precedent for C7+C8 gating and Swagger.
- `services/backend/src/modules/access/permissions-me.controller.ts` — new; `GET /permissions/me`.
- `services/backend/prisma/schema.prisma:323-335` — `FunctionalRoleAssignment` model ready; no schema change expected.
- `services/backend/src/modules/directory/` — add `employees.controller.ts`, `employees.swagger.ts`; register in `directory.module.ts` (or `app.module.ts` if no directory module yet).
- `services/backend/src/prisma/seed/data/bootcamp-identities.json` — e2e actors: id `1` Site Administrator (HR Admin assigner); id `2` Anton Savchenko (UJ-3 assignee — no seed `create_form_campaigns` assignment).
- `services/frontend/src/components/SideMenu/SideMenu.tsx` — replace list-fetch nav probe with `useMyPermissions` for Admin → Roles and Campaigns.
- `services/frontend/src/router/index.tsx` — add `/employees`, `/employees/:employeeId`; route guard on profile assignment (forbidden/redirect without `manage_functional_roles`, mirroring 1.4 `/admin/roles` guard).

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/access/functional-role-assignment.service.ts` — `setAssignments(employeeId, roleIds[])`: dedupe IDs, validate all roles exist, enforce last-admin lockout, transactional diff; keep `assign`/`unassign` for tests/seed
- [x] `services/backend/src/modules/access/dto/` + `entities/` — `SetEmployeeFunctionalRolesDto`, response entity matching GET/PUT body (id, name, permissionKeys)
- [x] `services/backend/src/modules/access/employee-functional-roles.controller.ts` (new) — `GET/PUT /employees/:employeeId/functional-roles`; `ParseUUIDPipe` on params; C7+C8 gate on both verbs; `@ApiTags` + Swagger
- [x] `services/backend/src/modules/access/permissions-me.controller.ts` (new) — `GET /permissions/me`; catalog-valid keys only; register in `access.module.ts`
- [x] `services/backend/src/modules/directory/employees.controller.ts` + `employees.swagger.ts` (new) — `GET /employees`, `GET /employees/:id`; auth required; comment pre-1.8 unfiltered list
- [x] `services/backend/src/modules/directory/directory.module.ts` (or `app.module.ts`) — register `EmployeesController`
- [x] `services/backend/src/modules/access/access.module.ts` — register new access controllers
- [x] Backend tests (cover all matrix rows): `functional-role-assignment.service.spec.ts`, `employee-functional-roles.controller.spec.ts` or unit equivalent, `functional-role-data-access-boundary.spec.ts`, `test/functional-role-assignments.e2e-spec.ts`
- [x] `services/frontend/src/api/hooks/useEmployeeFunctionalRoles.ts` + `useMyPermissions.ts` — TanStack Query; permissions query refetches on window focus
- [x] `services/frontend/src/pages/EmployeesPage/` (new) — minimal All Employees list linking to profile
- [x] `services/frontend/src/pages/EmployeeProfilePage/` (new) — Employment section card; functional roles multi-select with explicit Save (react-hook-form + zod, same stack as AdminRolesPage); options from `useFunctionalRoles` list hook; visible/editable only when `manage_functional_roles` in `useMyPermissions`
- [x] `services/frontend/src/router/index.tsx` — routes + assignment route guard; `SideMenu.tsx` — All Employees for all authenticated users; Admin + Campaigns via `useMyPermissions`
- [x] `services/frontend/src/locales/en/translation.json` — i18n keys for new surfaces
- [x] `services/frontend/e2e/functional-role-assignment.spec.ts` (new) — real credentialed sessions (Story 1.18 utilities, **no API mocks**): Site Administrator (manifest id `1`) assigns "IT Campaign Sender" on Anton Savchenko (manifest id `2`) profile → log in as assignee → assert `create_form_campaigns` appeared in permissions and Campaigns nav visible; assert no false-positive if assignee already held permission before assignment

**Acceptance Criteria:**
- Given HR Admin opens All Employees, selects an employee, and assigns "IT Campaign Sender" (grants `create_form_campaigns`) on the Employment section, when they save, then the assignment persists and appears on reload *(UJ-3 step 3; matrix: Replace assignments)*
- Given the IT lead from the scenario (bootcamp identity id `2`, Anton Savchenko) logs in after assignment, when the SPA loads `/permissions/me`, then the Campaigns nav item is visible and no other access-sensitive nav items appear without their permissions *(UJ-3 climax; matrix: Nav gate)*
- Given any authenticated user, when they open the app, then the All Employees nav entry is visible and lists all seeded employees by displayName *(pre-1.8 intentional; matrix: List employees)*
- Given Person X holds a role granting `create_edit_risks` and no ReportingLine or PP relationship to employee B, when C8 and C1 are evaluated for X→B, then `hasPermission` is true for the risk permission and S6 remains absent/`none` for Colleague access *(epic AC data-access boundary; matrix: Data-access boundary)*
- Given HR Admin removes a role assignment from Person X, when X's permissions are checked on the next request, then previously granted keys from that role are denied immediately *(matrix: Revoke effect)*
- Given Person X holds two functional roles with different permissions, when C8 checks any key granted by either role, then the check succeeds — permissions union across roles *(matrix: Multi-role union)*
- Given a user without `manage_functional_roles`, when they call assignment GET/PUT APIs, deep-link to the profile assignment control, or attempt to edit roles, then the API returns 403 and the UI does not offer the control *(matrix: Non-admin read/write)*
- Given HR Admin attempts a PUT that would remove the last employee holding `manage_functional_roles`, when they save, then the API rejects with 403 and at least one admin retains role-management access *(matrix: Last admin lockout)*

### Review Findings

**2026-09-01 — bmad-review (iteration 1, spec update)**

- [x] [Review][Spec] Resolved Ask First: assignment GET is admin-only; non-holders get 403 and hidden UI
- [x] [Review][Spec] Added All Employees nav/list policy for pre-1.8 authenticated access
- [x] [Review][Spec] Added matrix rows: duplicate roleIds, last-admin lockout, malformed UUID, PUT response body, non-admin GET, employees list/detail, orphans, concurrent PUT
- [x] [Review][Spec] Added tasks for directory module wiring, route guard, unified `useMyPermissions` nav gating
- [x] [Review][Spec] Pinned e2e actors to bootcamp manifest ids 1 and 2; require real sessions not mocks
- [x] [Review][Spec] Added `/permissions/me` catalog-orphan rule, employee-without-user behavior, form save UX note
- [x] [Review][Spec] Added Review Findings section; condensed Verification; merged Epic 11 deferral to single Never bullet

**2026-09-01 — bmad-code-review (iteration 2, implementation)**

- [x] [Review][Patch] Remove `@ArrayUnique()` from `SetEmployeeFunctionalRolesDto` so server dedupes duplicate `roleIds` per spec [`services/backend/src/modules/access/dto/set-employee-functional-roles.dto.ts:11`]
- [x] [Review][Patch] All Employees nav hidden until `/permissions/me` succeeds [`services/frontend/src/components/SideMenu/SideMenu.tsx:54`]
- [x] [Review][Patch] Frontend e2e mocks assignment/permissions APIs; missing real two-session UJ-3 flow with bootcamp ids 1 and 2 [`services/frontend/e2e/integration/functional-role-assignment.integration.spec.ts`]
- [x] [Review][Patch] No profile assignment route guard mirroring `FunctionalRolesRoute` [`services/frontend/src/router/index.tsx:47`]
- [x] [Review][Patch] Assignment API `permissionKeys` omit `filterCatalogValidKeys` (inconsistent with `/permissions/me`) [`services/backend/src/modules/access/functional-role-assignment.service.ts:222`]
- [x] [Review][Patch] `assign`/`unassign` bypass last-admin lockout guards [`services/backend/src/modules/access/functional-role-assignment.service.ts:89`]
- [x] [Review][Patch] Role existence validated outside `$transaction` [`services/backend/src/modules/access/functional-role-assignment.service.ts:42`]
- [x] [Review][Patch] Remove dead `useFunctionalRolesAccess` export [`services/frontend/src/api/hooks/useFunctionalRoles.ts:17`]
- [x] [Review][Patch] Missing `employee-functional-roles.controller.spec.ts` [`services/backend/src/modules/access/employee-functional-roles.controller.ts`]
- [x] [Review][Patch] Restore unknown-employee tests for `setAssignments`/`assign` [`services/backend/src/modules/access/__tests__/functional-role-assignment.service.spec.ts`]
- [x] [Review][Patch] Add self-assignment lockout unit test [`services/backend/src/modules/access/functional-role-assignment.service.ts:128`]
- [x] [Review][Patch] Add outsider PUT 403 backend e2e [`services/backend/test/functional-role-assignments.e2e-spec.ts`]
- [x] [Review][Patch] Add frontend e2e: non-admin cannot see employment-section [`services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx:33`]
- [x] [Review][Patch] Add frontend e2e: `sidebar-employees` visible for authenticated users [`services/frontend/src/components/SideMenu/SideMenu.tsx:62`]
- [x] [Review][Patch] Add pre-1.8 unfiltered-list comment on controller [`services/backend/src/modules/directory/employees.controller.ts`]
- [x] [Review][Patch] Add `EmployeesService` unit tests [`services/backend/src/modules/directory/employees.service.ts`]
- [x] [Review][Patch] Add idempotent `setAssignments` unit test [`services/backend/src/modules/access/__tests__/functional-role-assignment.service.spec.ts`]
- [x] [Review][Patch] Add malformed UUID 400 tests for assignment and employees routes [`services/backend/test/functional-role-assignments.e2e-spec.ts`]
- [x] [Review][Patch] Form omits assigned roles missing from catalog query [`services/frontend/src/pages/EmployeeProfilePage/components/FunctionalRolesAssignmentForm/FunctionalRolesAssignmentForm.tsx:80`]
- [x] [Review][Patch] Add empty-state when employees list returns `[]` [`services/frontend/src/pages/EmployeesPage/`]

## Design Notes

**PUT replace:** Multi-select UI maps to `PUT { roleIds }`. Dedupe with `Set`, validate role IDs exist inside the transaction, diff create/delete within `$transaction`. PUT returns 200 + same shape as GET for cache invalidation.

**Minimal employees API:** Return `displayName` from `User.name` (fallback email). No manager/department columns yet. Colleague-scoped directory filtering is Story 1.8 — note in code comments.

**Profile form UX:** Explicit Save button (not auto-save on each toggle). Reuse `react-hook-form` + `zod` from Story 1.4; invalidate assignment + permissions queries on success.

**Nav gating migration:** Replace `useFunctionalRolesAccess` (200 from roles list) with `useMyPermissions` for both Admin → Roles and Campaigns so assignment-driven permission changes do not require fetching the full roles list.

**Campaigns nav placeholder:** Route may 404 or show a stub page — this story only proves permission-gated visibility, not campaign functionality. E2e asserts nav visibility only, not navigation to a campaign page.

**E2e two-session pattern:** Use Story 1.18 credentialed login helpers. Session 1: Site Administrator assigns role. Session 2: assignee (id `2`) verifies `/permissions/me` and Campaigns nav. Assert permission delta — assignee must not already hold `create_form_campaigns` from seed before assignment (or create a fresh custom role in-test).

## Verification

Run from each service directory (see **Tasks** for test file names):

```bash
cd services/backend && npm run build && npm run lint && npm run depcruise
cd services/backend && npm test -- functional-role-assignment
cd services/backend && npm test -- functional-role-data-access-boundary
cd services/backend && npm run test:e2e -- functional-role-assignments

cd services/frontend && npm run typecheck && npm run lint && npm run build
cd services/frontend && npm run test -- functional-role-assignment
```

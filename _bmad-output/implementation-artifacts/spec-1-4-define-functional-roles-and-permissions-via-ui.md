---
title: 'Define Functional Roles and Permissions via UI'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 1
baseline_commit: '1ab3a66f08365c85c763685675bcd76f00f49ccd' # services/backend HEAD on main
frontend_baseline_commit: '22438fdd2c33967b0c38104313691b17024ba870' # services/frontend HEAD on main
story_key: '1-4-define-functional-roles-and-permissions-via-ui'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-18-authentication.md'
  - '{project-root}/_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C8 `PermissionChecker` is still a deny-by-default stub (`ContractsModule` → `PermissionCheckerStub`). There is no `FunctionalRole` / `FunctionalRolePermission` persistence, so feature gates like `manage_custom_fields` (Story 3.2) always deny, and HR Admin cannot define roles or permission grants at runtime.

**Approach:** Add the functional-role schema in `access`, implement a real `PermissionCheckerService` (unbinding C8 from `ContractsModule`, mirroring Story 1.2's C3 move), seed the five built-in roles from `access-model.md` §2.2 with a bootstrap HR Admin assignment, expose authenticated REST endpoints for role CRUD + permission toggles, and build the Admin → Roles SPA surface per EXPERIENCE.md UJ-3 **steps 1–2 only** (create/edit role and permission grants). UJ-3 step 3 — assigning the role on an employee profile — is Story 1.5.

## Boundaries & Constraints

**Always:**
- Permission keys are **strings stored as data** on `FunctionalRolePermission.permissionKey` — validated against a code-side catalog in `contracts/permission-keys.ts` (AD-1: both `access` and `directory` import from `contracts`, never from each other), never a Prisma enum, so new keys can ship without schema migration. **Split:** role→permission **grant rows** are data (editable in the UI); the set of valid permission **key strings** is code — adding a wholly new key type requires a deploy that updates `permission-keys.ts`, not a schema migration.
- C8's contract takes **`userId`** (not `employeeId`). `PermissionChecker.hasPermission(userId, key)` resolves live on every call: map `User` → `Employee`, union permissions across all assigned roles; **no cache** — removing a permission from a role denies on the very next call. No `Employee` row for the `userId` → deny (`false` / admin API 403).
- The schema stores **feature permissions only** — no column, flag, or implied grant of C1 access roles (ReportingLine / ProjectLine / PP / Colleague / FullAccess). Functional roles never widen data visibility (`access-model.md` §2.3).
- All functional-role admin APIs (`GET` list/catalog, `POST`/`PATCH`/`DELETE`) require `manage_functional_roles` via C8 **database lookup** (inject `PermissionChecker` in-process — not a recursive HTTP call). Seed the built-in **HR Admin** role with that permission and assign it to the bootcamp **Site Administrator** (`services/backend/src/prisma/seed/data/bootcamp-identities.json` manifest `id: 1`; use a named seed constant keyed off that manifest entry, not a hardcoded email in multiple places) so the UI is reachable after `db:seed`.
- Built-in roles (`Unit Manager`, `Delivery Manager`, `Project Manager`, `People Partner`, `HR Admin`) are idempotent seed upserts marked `isBuiltIn: true` — **permissions editable**, **name and deletion blocked**. The built-in **HR Admin** role must always retain `manage_functional_roles` — reject any PATCH that would remove it.
- Role names: reject empty/whitespace-only names; enforce **case-insensitive uniqueness** on `FunctionalRole.name`. Custom roles may hold **zero permissions** (valid shell role for Story 1.5 assignment).
- Grantable-permission catalog: centralize keys + human labels in `contracts/permission-keys.ts` (minimum set in Design Notes); update `directory/directory.constants.ts` to re-export `manage_custom_fields` from there. `PermissionChecker` and seed validation **ignore** DB rows whose `permissionKey` is no longer in the catalog (orphans from a later deploy do not grant access).
- Default permission grants on built-in roles follow D11 pending defaults in `decisions.md` (concrete seed mapping in Design Notes). **Grant rows are data-only** — PO may revise seed defaults later without code changes; new permission *key types* still require a catalog deploy.
- An employee may hold **multiple** functional roles; C8 unions permissions across all assignments. `FunctionalRoleAssignment` uses `@@unique([employeeId, roleId])`; internal `assign()` is an idempotent add, `unassign()` removes one pair.
- `FunctionalRoleAssignment` uses `onDelete: Restrict` on `Employee` — an `Employee` with assignment rows cannot be deleted until assignments are cleared.

**Ask First:**
- PO sign-off on D11 launch defaults before treating seed permission grants as production-frozen (safe to implement with documented defaults now).

**Never:**
- Do not build the employee-profile **assignment UI** — Story 1.5. This story may persist `FunctionalRoleAssignment` rows and expose an internal `assign()` / `unassign()` helper (mirrors Story 1.3's `PeoplePartnerAssignmentService`) plus seed/bootstrap assignments only.
- Do not gate feature-module nav items (Campaigns, Resourcing, etc.) or implement feature endpoints — consumers already inject C8; they start working once assignments exist in 1.5.
- Do not conflate functional PP role with C1 PP access resolution — they are independent dimensions.
- Do not add Department admin, org-relationship write screens, or full-access grant UI.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Create custom role | HR Admin; name + permission subset | Role persisted; permissions stored as rows; listed immediately | 403 without `manage_functional_roles`; 409 duplicate name (case-insensitive); 400 unknown permission key or empty/whitespace name |
| Create zero-permission role | HR Admin; name only, no permissions checked | Role persisted with zero permission rows | Same name/key validation as create |
| Toggle permissions | HR Admin removes `create_form_campaigns` from role R; holder's `userId` U assigned R (seed/helper) | Next `hasPermission(U, 'create_form_campaigns')` → false | N/A |
| PATCH permissions | HR Admin PATCHes existing role permission set | Permission rows replaced atomically; holders see change on next `hasPermission` | 400 unknown key; built-in HR Admin cannot lose `manage_functional_roles` |
| PATCH custom name | HR Admin PATCHes `name` on custom role | Name updated | 409 duplicate name; 400 empty name |
| PATCH built-in name | HR Admin PATCHes `name` on built-in role | Rejected — names immutable | 400 or 409 with clear message |
| Non-admin API | User without `manage_functional_roles` calls any admin endpoint | 403 | N/A |
| Unauthenticated API | No session cookie | 401 on all admin routes | N/A |
| No employee row | Authenticated `userId` with no `Employee` row | `hasPermission` → false; admin API → 403 | N/A |
| Built-in delete | DELETE on built-in role | Rejected | 409 or 400 with clear message |
| Custom role delete | DELETE on non-built-in role with no assignments | Role removed | 409 if assignments remain (message identifies blocking assignments) |
| Permission catalog | GET catalog with `manage_functional_roles` | Stable key + human label for every grantable permission | 403 without permission; 401 if unauthenticated |
| Empty role list | GET list before/with zero custom roles | 200 + array (built-ins present after seed) | N/A |
| C8 union | User holds roles A and B with distinct permissions | `hasPermission` true if **either** role grants the key | N/A |
| Data-access boundary | Role grants `create_edit_risks` only | `hasPermission` true; C1 `resolveAudience` unchanged — no automatic Manager/PP sections | N/A |
| Internal assign | Seed/test calls `assign(employeeId, roleId)` twice | Second call idempotent — one assignment row | 404 for unknown employee or role id |
| Internal unassign | `unassign(employeeId, roleId)` | Assignment row removed | 404 for unknown pair |
| Concurrent PATCH | Two HR Admins PATCH same role simultaneously | Last write wins (no optimistic concurrency in this story) | N/A |
| Employee delete | DELETE employee with assignment rows | Blocked by FK Restrict | Prisma/deletion error until assignments cleared |

</frozen-after-approval>

## Dependencies

- **Story 1.18 (auth / C7)** — done; all admin routes and SPA surfaces require authenticated sessions.
- **Story 1.16 (seed manifest)** — `services/backend/src/prisma/seed/data/bootcamp-identities.json` (see `bootcamp-scope-overrides.md`).
- **Story 1.19 (contracts / registry)** — C8 token exists; this story binds the real implementation in `access`.

## Code Map

See **Tasks** for the actionable checklist. Key surfaces:

- `services/backend/prisma/schema.prisma` — add `FunctionalRole`, `FunctionalRolePermission`, `FunctionalRoleAssignment`
- `services/backend/src/modules/contracts/` — `permission-keys.ts` (new catalog); remove C8 stub binding from `contracts.module.ts`
- `services/backend/src/modules/access/` — C8 impl, functional-role services, first controller (DTOs/entities/swagger per `nest-modules.md`)
- `services/backend/src/modules/directory/custom-fields.service.ts` — existing C8 consumer; must resolve real checker after rebind
- `services/backend/src/prisma/seed/` — built-in roles, D11 defaults, HR Admin bootstrap assignment
- `services/frontend/` — `AdminRolesPage`, hooks, `/admin/roles` route, SideMenu gating

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/permission-keys.ts` (new) — canonical catalog (keys, labels, optional description) + type-safe exports — single source for validation, seed, API, and tests (AD-1 safe for all consumers)
- [x] `services/backend/prisma/schema.prisma` + migration — `FunctionalRole` (`name`, `isBuiltIn`), `FunctionalRolePermission` (`roleId`, `permissionKey`), `FunctionalRoleAssignment` (`employeeId`, `roleId`, `@@unique([employeeId, roleId])`, `onDelete: Restrict` on `Employee` FK)
- [x] `services/backend/src/modules/access/permission-checker.service.ts` (new) — implement C8: resolve `userId` → `Employee`, load role assignments → union permission keys from catalog-valid rows only; deny when no employee row or no matching grant
- [x] `services/backend/src/modules/access/functional-role.service.ts` (new) — list/create/update/delete roles; PATCH atomically replaces permission set (custom roles: name + permissions; built-in: permissions only); block built-in delete and built-in rename; block built-in HR Admin losing `manage_functional_roles`; block delete when assignments remain; case-insensitive name uniqueness; reject empty names and unknown keys
- [x] `services/backend/src/modules/access/functional-role-assignment.service.ts` (new) — internal idempotent `assign(employeeId, roleId)` / `unassign(employeeId, roleId)` for seed + unit tests only (404 on unknown ids; not in public controller)
- [x] `services/backend/src/modules/access/dto/` + `entities/` (new) — request/response DTOs with class-validator; response entities for OpenAPI
- [x] `services/backend/src/modules/access/functional-roles.controller.ts` (new) — `GET/POST /functional-roles`, `PATCH/DELETE /functional-roles/:id`, `GET /permissions/catalog`; resolve viewer via C7; enforce `manage_functional_roles` via injected C8; `@ApiTags` + Swagger decorators
- [x] `services/backend/src/modules/access/access.module.ts` — `{ provide: PermissionChecker, useClass: PermissionCheckerService }`; register new services/controller; export C8; update module JSDoc (recognized exception: first controller in this module)
- [x] `services/backend/src/modules/contracts/contracts.module.ts` — remove C8 stub provider/export; update `contracts.module.spec.ts` to assert C8 left unbound (mirror C3 pattern)
- [x] `services/backend/src/modules/directory/directory.constants.ts` — re-export `manage_custom_fields` from `contracts/permission-keys.ts`
- [x] `services/backend/src/prisma/seed/` — upsert built-in roles, D11 default permissions (Design Notes table), assign HR Admin to Site Administrator via manifest `id: 1` constant
- [x] `services/backend/src/modules/access/__tests__/permission-checker.service.spec.ts` (new) — union, immediate revocation, deny-default, no-employee-row
- [x] `services/backend/src/modules/access/__tests__/functional-role.service.spec.ts` (new) — CRUD matrix + built-in delete/rename guards + unknown-key rejection + HR Admin lockout guard + empty-permission role
- [x] `services/backend/test/functional-roles.e2e-spec.ts` (new) — HTTP 401/403 on admin routes; happy-path CRUD for seeded HR Admin session
- [x] `services/backend/src/modules/directory/__tests__/custom-fields.service.spec.ts` — extend or add case confirming real C8 rebind (not stub) when `AccessModule` is loaded
- [x] `services/frontend/src/api/hooks/useFunctionalRoles.ts` (new) + types — TanStack Query hooks for list/create/update + permission catalog (credentialed requests per Story 1.18)
- [x] `services/frontend/src/pages/AdminRolesPage/` (new) — list roles, create/edit dialog with permission checkboxes (UJ-3 steps 1–2); i18n keys in `locales/en/translation.json`; `react-hook-form` + `zod`
- [x] `services/frontend/src/router/index.tsx` + `SideMenu.tsx` — `/admin/roles` route; show Admin → Roles when roles list fetch returns **200**; hide on **403/401**; hide on **5xx/network** without treating as unauthorized (no nav flicker for authorized users on retry)
- [x] `services/frontend/e2e/admin-roles.spec.ts` (new) — Playwright smoke: log in as seeded Site Administrator (HR Admin role) → Admin → Roles → create role with one permission → visible on reload

**Acceptance Criteria:**
- Given HR Admin opens Admin → Roles, creates "Security Champion" granting only `create_form_campaigns`, when they save, then the role and permission rows persist and appear on reload with no deploy beyond migration/seed *(UJ-3 step 1–2; matrix: Create custom role)*
- Given "Security Champion" grants `create_form_campaigns` and user X holds that role (via seed or internal assign), when HR Admin removes that permission and X's access is checked, then `hasPermission(X, 'create_form_campaigns')` is false on the very next call (X is `userId`, per C8) *(matrix: Toggle permissions)*
- Given the minimum grantable permission set from the epic AC, when HR Admin edits any built-in or custom role, then each listed permission is independently selectable and persistable *(matrix: PATCH permissions)*
- Given a user without `manage_functional_roles`, when they call a functional-role admin API (including `GET /functional-roles`), then the API returns 403 and the SPA does not expose the Admin → Roles nav entry *(matrix: Non-admin API)*
- Given HR Admin attempts to remove `manage_functional_roles` from the built-in HR Admin role, when they save, then the API rejects the change and at least one seeded admin retains role-management access *(matrix: PATCH permissions / HR Admin lockout guard)*

### Review Findings

**2026-08-31 — code review (iteration 1)**

- [x] [Review][Patch] SideMenu uses flat "Roles" label instead of Admin → Roles IA — add Admin section header and nest Roles under it per EXPERIENCE.md [services/frontend/src/components/SideMenu/SideMenu.tsx]
- [x] [Review][Patch] Deep-link `/admin/roles` reachable by authenticated non–HR Admin — add route-level permission guard (redirect or forbidden page) [services/frontend/src/router/index.tsx]

- [x] [Review][Patch] Playwright smoke fully mocks functional-role APIs [services/frontend/e2e/admin-roles.spec.ts:22-62]
- [x] [Review][Patch] Missing `custom-fields.service.spec.ts` C8 rebind test despite task marked complete [services/backend/src/modules/directory/__tests__/]
- [x] [Review][Patch] Case-insensitive role name uniqueness not enforced at DB layer [services/backend/prisma/schema.prisma:23]
- [x] [Review][Patch] Prisma unique violations on create/rename return 500 instead of 409 [services/backend/src/modules/access/functional-role.service.ts:45-56]
- [x] [Review][Patch] Role delete check-then-delete is not transactional [services/backend/src/modules/access/functional-role.service.ts:141-157]
- [x] [Review][Patch] GET `/permissions/catalog` 401/403 not covered in backend e2e [services/backend/test/functional-roles.e2e-spec.ts]
- [x] [Review][Patch] SideMenu nav visibility (200 vs 403) has no automated test [services/frontend/src/components/SideMenu/SideMenu.tsx:17-18]
- [x] [Review][Patch] Seed tests do not assert D11 per-role permission key sets [services/backend/src/prisma/seed/__tests__/seed.service.spec.ts:305-307]
- [x] [Review][Patch] `FunctionalRoleAssignmentService.assign` idempotency untested [services/backend/src/modules/access/functional-role-assignment.service.ts:16-22]
- [x] [Review][Patch] `FunctionalRoleAssignmentService` has no unit tests [services/backend/src/modules/access/functional-role-assignment.service.ts]
- [x] [Review][Patch] Permission catalog fetch failure shows silent empty checkbox list [services/frontend/src/pages/AdminRolesPage/hooks/useRoleForm.ts:67]
- [x] [Review][Patch] Delete-with-assignments error message lacks assignment-identifying detail [services/backend/src/modules/access/functional-role.service.ts:152-155]

- [x] [Review][Defer] Seed upsert may overwrite custom role that shares a built-in name [services/backend/src/prisma/seed/seed.functional-roles.ts:87-99] — deferred, bootcamp-only re-seed edge case unlikely in practice

## Design Notes

**Permission-checker query shape:** resolve `Employee` by `userId` → `FunctionalRoleAssignment` rows → join `FunctionalRolePermission` → collect all catalog-valid `permissionKey` values across roles (union), then test membership. Do not short-circuit after the first role match.

**D11 seed defaults (pending PO sign-off — safe to implement now):**

Seed upserts **exactly** these permission-key sets per built-in role (sorted; no other keys on built-ins at first launch). Derived from `access-model.md` §2.2 feature narratives + D11 pending defaults in `decisions.md`.

| Built-in role | Default permission keys (exhaustive) |
| --- | --- |
| HR Admin | `change_organisational_relationships`, `manage_custom_fields`, `manage_departments`, `manage_functional_roles` |
| Unit Manager | `assign_end_mentorships`, `create_action_items`, `create_edit_risks`, `create_feedback`, `edit_career_timeline`, `fulfil_resourcing_requests`, `maintain_cds_records`, `manage_custom_fields`, `view_dashboard` |
| Delivery Manager | `approve_reject_candidates`, `assign_end_mentorships`, `close_resourcing_requests`, `create_action_items`, `create_edit_risks`, `create_feedback`, `create_resourcing_requests`, `maintain_cds_records`, `manage_custom_fields`, `view_dashboard` |
| Project Manager | `assign_end_mentorships`, `create_action_items`, `create_edit_risks`, `create_feedback`, `create_resourcing_requests`, `maintain_cds_records`, `manage_custom_fields`, `view_dashboard` |
| People Partner | `assign_end_mentorships`, `create_edit_risks`, `create_feedback`, `create_form_campaigns`, `edit_career_timeline`, `maintain_cds_records`, `record_departure`, `view_dashboard` |

**D11 five-gap mapping (for PO review):**

| D11 open question | Resolved default (roles) |
| --- | --- |
| Who holds *manage custom fields*? | HR Admin, Unit Manager, Delivery Manager, Project Manager |
| Who holds *assign and end mentorships*? | Unit Manager, Delivery Manager, Project Manager, People Partner |
| Launch default for *approve or reject proposed candidates*? | Delivery Manager only |
| Launch default for *edit the career timeline*? | Unit Manager, People Partner |
| Launch default for *create feedback*? | Unit Manager, Delivery Manager, Project Manager, People Partner |

**Explicit non-grants (do not seed on built-ins):**

| Permission key | Absent from (built-in roles) | Rationale |
| --- | --- | --- |
| `create_resourcing_requests` | UM, PP, HR Admin | UM fulfils; PP has no resourcing; HR Admin is config-only |
| `fulfil_resourcing_requests` | DM, PM, PP, HR Admin | UM-only fulfil path per §2.2 |
| `approve_reject_candidates` | UM, PM, PP, HR Admin | DM-only approve/reject per §2.2 + D11 |
| `close_resourcing_requests` | UM, PM, PP, HR Admin | DM closes requests per §2.2 |
| `create_form_campaigns` | UM, DM, PM, HR Admin | PP default; extensible via custom roles (UJ-3) |
| `record_departure` | UM, DM, PM, HR Admin | PP default per §2.2 |
| `manage_departments` | UM, DM, PM, PP | HR Admin platform admin only |
| `change_organisational_relationships` | UM, DM, PM, PP | HR Admin platform admin only |
| `manage_functional_roles` | UM, DM, PM, PP | HR Admin platform admin only; built-in HR Admin row must always retain it |

PO may revise grant rows via seed or the roles UI without code changes.

**Dashboard permission:** epic AC lists one grantable permission — "view a given dashboard". Use single key `view_dashboard`; which dashboard variant (UM/DM/PM/PP) a holder sees is resolved later from their functional-role assignments (Epic 12), not from four separate permission keys in this story.

**Built-in role names:** Store display names exactly as in `access-model.md` §2.2 (`Unit Manager`, etc.). Custom roles use HR Admin-provided names.

**Minimum grantable permission keys** (`contracts/permission-keys.ts`): `create_form_campaigns`, `create_action_items`, `create_edit_risks`, `create_resourcing_requests`, `fulfil_resourcing_requests`, `approve_reject_candidates`, `close_resourcing_requests`, `assign_end_mentorships`, `maintain_cds_records`, `edit_career_timeline`, `create_feedback`, `record_departure`, `manage_custom_fields`, `manage_departments`, `change_organisational_relationships`, `view_dashboard`, `manage_functional_roles`.

## Spec Change Log

**2026-08-31 — bmad-review resolution:**
- Intent: scoped UJ-3 to steps 1–2; step 3 (profile assignment) explicitly deferred to Story 1.5.
- Boundaries: clarified code catalog vs data grant rows; canonical seed manifest path; HR Admin lockout guard; built-in rename block; case-insensitive name uniqueness; zero-permission roles allowed; multi-role union + unique assignment pairs; orphan key handling; C8 in-process gating.
- I/O matrix: added rows for empty name, zero-permission role, PATCH name rules, 401, no-employee-row, empty list, assign/unassign, concurrent PATCH (last-write-wins), employee delete with assignments; expanded error handling on existing rows.
- Dependencies section added (1.18, 1.16, 1.19).
- Code Map condensed to pointer; detail lives in Tasks.
- Tasks: DTOs/entities/Swagger; schema constraints; service guards; e2e + directory C8 consumer test; mandatory Playwright smoke; clarified SideMenu gating (200 vs 403/401 vs 5xx).
- Design Notes: D11 seed-default table; permission key list moved here from Boundaries; bootstrap assignment merged into Boundaries.
- Acceptance: added HR Admin lockout AC; cross-refs to matrix rows.
- Verification: Playwright smoke required (Story 1.18 login fixture); removed redundant manual checks.
- Context: added `spec-1-18-authentication.md` and `bootcamp-scope-overrides.md`.

**2026-08-31 — D11 table exhaustification:**
- Design Notes: replaced partial D11 seed table with exhaustive per-role key lists (all 17 catalog keys allocated or explicitly non-granted); added D11 five-gap mapping table and explicit non-grants rationale table.
- Removed `record_departure` from HR Admin seed (PP-only per §2.2).

## Verification

**Commands:**
- `cd services/backend && npm run build` — expected: compiles clean
- `cd services/backend && npm run lint` — expected: no new lint errors
- `cd services/backend && npm test -- permission-checker` — expected: union + revocation + no-employee tests pass
- `cd services/backend && npm test -- functional-role` — expected: CRUD matrix + guard tests pass
- `cd services/backend && npm run test:e2e -- functional-roles` — expected: 401/403 + HR Admin CRUD pass
- `cd services/backend && npm run depcruise` — expected: no boundary violations
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` — expected: Admin Roles page compiles
- `cd services/frontend && npm run test` — expected: Playwright `admin-roles.spec.ts` nav/form smoke passes
- `cd services/frontend && npm run test:integration -- admin-roles` — expected: Site Administrator full-stack create-role smoke passes against seeded backend

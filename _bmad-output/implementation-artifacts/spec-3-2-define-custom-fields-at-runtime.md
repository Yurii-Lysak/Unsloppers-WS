---
title: 'Define Custom Fields at Runtime (FieldRegistry C2)'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '9c376b3e6ab029f08276064418f8b33d437a56a8'
story_key: '3-2-define-custom-fields-at-runtime'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-3-context.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C2 `FieldRegistry` is still a Wave-0 no-op stub, so custom fields cannot be defined or queried at runtime — blocking Story 3.1 and all downstream field-visibility enforcement.

**Approach:** Stand up the `directory` module with AD-6 schema (`CustomFieldDefinition` + lazy `CustomFieldValue`), a real `FieldRegistryService` that replaces the stub via DI, and authenticated REST endpoints gated by `manage_custom_fields` (definitions) plus S16-aware visibility filtering (reads/writes).

## Boundaries & Constraints

**Always:**
- AD-6 typed columns with lazy row creation; one populated value column per row, others SQL `NULL`.
- `FieldRegistry.query` is the only type-branching point for custom field values.
- Field definition writes require C8 `PermissionChecker.hasPermission(userId, 'manage_custom_fields')`.
- Per-field visibility (`management` / `employee` / `colleague`) enforced server-side on definitions and values via C1 `AccessResolver` + S16 section access.
- Directory module depends only on `contracts` (and Nest core) — no feature-to-feature imports (AD-1).
- Real `FieldRegistry` binds via `@Global() DirectoryModule` `{ provide: FieldRegistry, useExisting: FieldRegistryService }`, overriding the Wave-0 stub without editing `ContractsModule`.

**Ask First:** none identified.

**Never:** no frontend admin UI in this story (API substrate only); no All Employees list/filter engine (Story 3.1); no ALTER TABLE per custom field; no real `PermissionChecker` / `AccessResolver` implementations (stubs remain until Track A lands).

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Define select field | HR Admin + valid name/type/options/visibility | Definition persisted; field id returned; immediately listable | 403 without permission; 409 duplicate name; 400 missing/invalid options |
| Set typed value | Valid employee + field + typed value | Upsert row with one populated column; others NULL | 404 missing employee/field; 400 type mismatch |
| Clear value | value = null | Existing row deleted (lazy unset) | 404 missing field |
| Colleague reads management field | Colleague role + management visibility field | Field absent from definitions list and value responses | 403 on direct get if attempted |
| Colleague reads colleague field | Colleague role + S16 read + colleague visibility | Field present in filtered responses | N/A |

</frozen-after-approval>

## Code Map

- [`services/backend/src/modules/contracts/field-registry.contract.ts`](../../services/backend/src/modules/contracts/field-registry.contract.ts) — C2 token extended with `multi_select` and optional `options` on `defineField`
- [`services/backend/src/modules/directory/field-registry.service.ts`](../../services/backend/src/modules/directory/field-registry.service.ts) — real C2 implementation (Prisma access, type encode/decode)
- [`services/backend/src/modules/directory/custom-field-visibility.service.ts`](../../services/backend/src/modules/directory/custom-field-visibility.service.ts) — visibility × S16 enforcement using C1
- [`services/backend/src/modules/directory/custom-fields.service.ts`](../../services/backend/src/modules/directory/custom-fields.service.ts) — HTTP orchestration + permission gates
- [`services/backend/src/modules/directory/custom-fields.controller.ts`](../../services/backend/src/modules/directory/custom-fields.controller.ts) — `/api/v1/custom-fields` REST surface
- [`services/backend/prisma/schema.prisma`](../../services/backend/prisma/schema.prisma) — `CustomFieldDefinition`, `CustomFieldValue` models (AD-6)
- [`services/backend/src/app.module.ts`](../../services/backend/src/app.module.ts) — registers `DirectoryModule` after `ContractsModule`

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/prisma/schema.prisma` + migration — AD-6 models — persistence substrate
- [x] `services/backend/src/modules/directory/field-registry.service.ts` — C2 real impl — replaces stub
- [x] `services/backend/src/modules/directory/custom-field-visibility.service.ts` — visibility rules — leak prevention
- [x] `services/backend/src/modules/directory/custom-fields.controller.ts` — REST API — UI-ready endpoints
- [x] `services/backend/src/modules/directory/directory.module.ts` — DI override — global FieldRegistry swap
- [x] `services/backend/src/modules/directory/__tests__/*.spec.ts` — unit coverage — matrix rows + core paths

**Acceptance Criteria:**
- Given an HR Admin with `manage_custom_fields` creates a single-select field "Preferred office" with visibility "employee", when they save the definition, then it is immediately returned by `GET /custom-fields` with no deploy or migration step beyond the included Prisma migration
- Given a custom field "Performance flag" with visibility "management", when a Colleague-level viewer lists definitions or values, then the field is absent from the API response

## Design Notes

Multi-select values encode as JSON string arrays in `valueText` (AD-6 allows type branching only inside FieldRegistry). Select options live on `CustomFieldDefinition.options` as JSON.

## Verification

**Commands:**
- `cd services/backend && npx nest build` — PASS (compile clean; postbuild seed needs JWT env)
- `cd services/backend && npm run lint` — PASS
- `cd services/backend && npm test -- --testPathPatterns=directory` — PASS, 14 tests
- `cd services/backend && npm run depcruise` — PASS, no violations

**Manual checks:**
- With Postgres + JWT env, `POST /api/v1/custom-fields` after login with a permission-granted user creates a field; Colleague session omits management-visible fields from `GET /api/v1/custom-fields`

## Suggested Review Order

**FieldRegistry core**

- Typed storage + lazy unset semantics
  [`field-registry.service.ts:67`](../../services/backend/src/modules/directory/field-registry.service.ts#L67)

**Visibility enforcement**

- Management fields hidden from Colleague role
  [`custom-field-visibility.service.ts:13`](../../services/backend/src/modules/directory/custom-field-visibility.service.ts#L13)

**DI wiring**

- Global stub override
  [`directory.module.ts:18`](../../services/backend/src/modules/directory/directory.module.ts#L18)

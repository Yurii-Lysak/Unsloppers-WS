---
title: 'Assemble Employee Profile by Section Access'
type: 'feature'
created: '2026-09-01'
status: 'done'
review_loop_iteration: 1
baseline_commit: '17da8804a34ce2a4bd48d431d1bb5694a19db27a'
frontend_baseline_commit: 'b07302ccb459f564e34767bd5eed81f99b267a8f'
story_key: '1-6-assemble-employee-profile-by-section-access'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-5-assign-functional-roles-to-people.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** C1 resolves section grants (Stories 1.1–1.3) and two section providers exist (S9, S10), but no `ProfileAssemblerService` wires them together. `GET /employees/:id` returns only `{ id, displayName }`, so the profile page cannot respect the access matrix — ungranted sections cannot be absent server-side.

**Approach:** Add `ProfileAssemblerService` (assembler leak risk per AD-3) in `access`: resolve C1 once per request, call `@RegisterProvider('section', Sx)` only for granted sections, return a wire-safe profile DTO with an `audience` grant map and section payloads, or explicit `unavailable` markers. Update C1 `COLLEAGUE_SECTIONS` to the Rule 4 whitelist so Colleague profile reads match the matrix. Add minimal S1 and S11 section providers and a profile read endpoint; extend the SPA profile page to render only sections present in the API response.

## Boundaries & Constraints

**Always:**
- Resolve C1 **once** per profile request in `ProfileAssemblerService`; pass the resulting `ResolvedAudience` into each `SectionProvider.getSection(viewerId, subjectId, audience)` — providers must not re-call C1 when invoked from the assembler (dev-only direct routes may still resolve locally; see Design Notes).
- **Profile response shape (normative):**
  ```typescript
  {
    employeeId: string;
    displayName: string; // wire-safe; sourced in assembler from S1 data or Employee/User join — single fetch, no second HTTP call
    audience: { role: AccessRole; sections: Record<SectionId, SectionAccessLevel> };
    sections: {
      S1?: { accessLevel: 'R' | 'RW'; data: IdentitySectionDto } | { accessLevel: 'R' | 'RW'; status: 'unavailable' };
      S10?: { accessLevel: 'R'; data: LeavesSectionDto } | { accessLevel: 'R'; status: 'unavailable' };
      // only granted section ids appear
    };
  }
  ```
  `audience.sections` is a plain `Record` (AD-2). Section **payload keys exist only where grant ≠ `none`** — denied sections are **omitted entirely** (not `null`, not empty objects). Each included section carries `accessLevel: 'R' | 'RW'` matching the grant map.
- **Assembler normalization:** When a provider returns integration-level unavailability (e.g. S10 `LeavesSectionEntity.availability === 'unavailable'`), empty/`null`/`undefined` payload, or throws (`ForbiddenException` or any `Error`), the assembler maps that section to `{ accessLevel, status: 'unavailable' }` — never nested `data.availability`, never a 403/500 for the whole profile, never silent omission when grant ≠ `none`.
- When grant ≠ `none` but registry lookup returns `unavailable`, include `{ accessLevel, status: 'unavailable' }` for that section.
- Update `COLLEAGUE_SECTIONS` in `access-resolver.service.ts` to `S1: 'R'`, `S10: 'R'`, `S11: 'R'`, all others `'none'` (`access-model.md` Rule 4). Providers apply field narrowing: S10 dates-only and type-hidden; S11 project-name-only for Colleague. (`LeavesSectionProvider` already masks S10.)
- S1 provider returns wire-safe identity fields available today (`displayName`, linked `manager`/`peoplePartner` id+displayName when resolvable). **D5:** omit `mentor` for Colleague viewers. Self carries S1 `accessLevel: 'RW'` but the payload remains a read-only subset until schema adds photo/position/department — do not render write affordances for deferred fields.
- S11 minimal stub: `@RegisterProvider('section', 'S11')` returns project **names only** from C3 `ProjectAssignment.listByEmployee` when rows exist; omit PM/DM/period until the epic provider lands. Empty array is valid `data` when no assignments — not `unavailable`.
- New route `GET /employees/:employeeId/profile` (auth via C7 → map `userId` to `Employee.id` for C1; **403** when authenticated user has no linked `Employee`, mirroring `leaves.controller.ts`). Keep existing `GET /employees/:employeeId` summary unchanged for list navigation (Story 1.5).
- Do not invoke C8 `PermissionChecker` from the assembler or section providers. C8 governs functional features only.
- Frontend uses **only** `GET /employees/:employeeId/profile` for the profile page (header `displayName` and section cards) — no parallel `useEmployeeDetail` fetch (AD-5). Render section cards **only** for keys returned in `sections`; no client-side filtering, hiding, or post-hoc trimming. Show Access Scope Chip when viewer ≠ subject, labeled from `audience.role` (single display role per current C1 shape; AD-15 multi-audience provenance strings deferred).
- Employment functional-roles block (Story 1.5): show when caller holds `manage_functional_roles` via C8 **regardless of C1 audience** to the subject — it is not part of S4 assembly and stays C8-gated separately until S4 provider exists.
- **Full profile access:** C1 `FullAccess` is not resolved in `AccessResolverService` yet — holders fall through to Colleague-equivalent assembly until a later story implements C13 grant resolution. Document this gap in `profile.controller` / assembler comments; do not invent FullAccess logic here.
- Self profile with many granted-but-unbuilt sections: return `status: 'unavailable'` per section — no special-case empty profile; frontend may show unavailable placeholders for granted keys.

**Ask First:**
- If implementing S1 requires new Prisma columns beyond what exists today (photo, position, department, mentor), HALT — stub with available fields only unless human approves schema expansion in this story.

**Never:**
- Do not build full providers for S2–S8, S12–S16 beyond registry wiring (return `unavailable` until epic providers land). S11 is excepted — minimal name-only stub only.
- Do not implement S7 visibility flags (1.9), S16 per-field visibility (1.10), Shared Link audience (1.11), profile-header link UX / mentor header rules beyond S1 field omission (1.7), or colleague enforcement on directory export/search/list columns (1.8).
- Do not filter `GET /employees` list by C1 — pre-1.8 authenticated full list remains intentional.
- Do not cache C1 results (AD-4).

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| ReportingLine profile | Manager M requests B's profile | 200; `audience.role` ReportingLine; all ReportingLine-granted section keys at matrix levels; S9/S10 populated when providers ok | 401 unauthenticated; 404 unknown employee |
| ProjectLine profile | DM D (project line only) requests B's profile | 200; `audience.role` ProjectLine; section keys per Rule 2 (no S2/S3; S5/S7 narrowed at provider layer when built) | 401; 404 |
| PP profile | Assigned PP P requests B's profile | 200; `audience.role` PP; PP grant-map section keys present | 401; 404 |
| Multi-audience union | Viewer matches ReportingLine and PP for B | 200; `sections` reflects per-section union (least-restrictive across matched audiences); `audience.role` is highest-rank role only (display) | N/A |
| Self profile | X requests own profile | 200; `audience.role` Self; Self grant map; S1 present; unbuilt granted sections as `unavailable` | 401; 404 |
| Colleague profile | Unrelated A requests B | 200; `audience.role` Colleague; **only** `S1`, `S10`, `S11` keys in `sections` | 401; 404 |
| Denied section absent | A Colleague requests B | Response has no `S2`…`S9`, `S12`…`S16` keys at all | N/A |
| Viewer without Employee | Authenticated user has no `Employee` row | — | 403 before C1 |
| Unregistered granted section | ReportingLine viewer; S6 no provider yet | `sections.S6: { accessLevel: 'RW', status: 'unavailable' }` | N/A |
| Provider throw | S10 provider throws `ForbiddenException` or `Error` | `sections.S10: { accessLevel, status: 'unavailable' }`; other sections still returned | Log server-side; do not 500 whole profile unless assembler itself fails |
| S10 integration down | S10 granted; provider returns `availability: 'unavailable'` | Assembler emits `sections.S10: { accessLevel: 'R', status: 'unavailable' }` — not nested `data.availability` | N/A |
| Provider empty payload | Provider resolves to `null`/`undefined` | Section `status: 'unavailable'` | N/A |
| S10 Colleague masking | Colleague viewer; S10 granted | Leaves returned with dates; `type` and `approvalState` null | N/A |
| S1 Colleague D5 | Colleague viewer | S1 payload has no `mentor` field | N/A |
| S11 Colleague narrowing | Colleague viewer; S11 granted | Project entries expose **name** only (no PM/DM/period) | N/A |
| Malformed id | Non-UUID `employeeId` | 400 | N/A |
| Unknown employee | Valid UUID not in DB | 404 | N/A |
| Wire safety | Any success response | JSON only; ISO date strings; no `Map`, `undefined`, or class instances; top-level `displayName` present | N/A |
| C1 boundary | Functional role widens C8 only | Assembler output unchanged vs before assignment for same viewer/subject pair | N/A |
| Employment block | Colleague A with `manage_functional_roles` views B's profile | Profile sections Colleague-trimmed; Employment functional-roles block still visible and editable | 403 on assignment APIs without C8 |

</frozen-after-approval>

## Code Map

**Contracts**
- `services/backend/src/modules/contracts/access-resolver.contract.ts` — `ResolvedAudience`, `SectionId`, `SectionAccessLevel`; reuse in profile DTOs.
- `services/backend/src/modules/contracts/section-provider.contract.ts` (new) — `SectionProvider` abstract class: `getSection(viewerId, subjectId, audience): Promise<unknown>`.
- `services/backend/src/modules/contracts/stubs/access-resolver.stub.ts` — deny-all Colleague stub; no change required unless tests assert Colleague whitelist via stub (prefer real C1 in profile tests).

**Access core**
- `services/backend/src/modules/access/access-resolver.service.ts:117-134` — `COLLEAGUE_SECTIONS` all `'none'` today; replace with Rule 4 whitelist (S1/S10/S11 `R`).
- `services/backend/src/modules/access/profile-assembler.service.ts` (new) — C1 once → iterate granted sections → registry lookup → provider call → normalize to section envelope.
- `services/backend/src/modules/access/identity-section.provider.ts` (new) — `@RegisterProvider('section', 'S1')`; D5 mentor omission for Colleague.
- `services/backend/src/modules/access/projects-section.provider.ts` (new) — `@RegisterProvider('section', 'S11')`; minimal name-only stub via C3.
- `services/backend/src/modules/access/profile.controller.ts` + `profile.swagger.ts` (new) — `GET /employees/:employeeId/profile`; C7 auth; `resolveEmployeeId` → 403 when missing.
- `services/backend/src/modules/access/access.module.ts` — register assembler, profile controller, S1/S11 providers; export assembler if needed by tests.
- `services/backend/src/modules/access/entities/` + `dto/` — `EmployeeProfileEntity`, per-section union types.

**Section providers (existing)**
- `services/backend/src/modules/registry/provider-registry.service.ts:119-128` — `get('section', Sx)` lookup; `{ status: 'unavailable' }` when unregistered.
- `services/backend/src/modules/timeline/timeline-section.provider.ts` — adopt `audience` arg; S9 already registered.
- `services/backend/src/modules/integrations/leaves-section.provider.ts:15-50` — use passed `audience` for S10 gate + Colleague masking; stop redundant C1 when called from assembler.
- `services/backend/src/modules/integrations/leaves.controller.ts` — dev route keeps **local** `resolveEmployeeId` + optional local C1 resolve when `audience` not passed; update call signature after contract change.

**Directory (unchanged)**
- `services/backend/src/modules/directory/employees.controller.ts:29-37` — summary endpoint; leave unchanged.

**Tests**
- `services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts:230` — Colleague assertion currently all `'none'`; update to S1/S10/S11 `R`.
- `services/backend/src/modules/access/__tests__/profile-assembler.service.spec.ts` (new) — matrix rows with mocked registry/providers; include multi-audience union and S10 normalization cases.
- `services/backend/test/support/access-matrix.ts` — update Colleague expected grants if shared helpers assert section maps.
- `services/backend/test/employee-profile.e2e-spec.ts` (new) — credentialed HTTP: ReportingLine vs Colleague key sets on `/profile`.

**Frontend**
- `services/frontend/src/api/hooks/useEmployeeProfile.ts` (new) — TanStack Query for `GET /employees/:id/profile`.
- `services/frontend/src/pages/EmployeeProfilePage/EmployeeProfilePage.tsx` — single profile hook for title + sections; remove `useEmployeeDetail`; Access Scope Chip; section cards; keep C8-gated Employment block.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/section-provider.contract.ts` — define `SectionProvider` with `getSection(viewerId, subjectId, audience)` — AD-3 wire contract
- [x] `services/backend/src/modules/access/access-resolver.service.ts` — update `COLLEAGUE_SECTIONS` to Rule 4 whitelist; adjust unit tests
- [x] `services/backend/src/modules/access/identity-section.provider.ts` — S1 provider with D5 Colleague mentor omission
- [x] `services/backend/src/modules/access/projects-section.provider.ts` — S11 minimal name-only stub via `ProjectAssignment`
- [x] `services/backend/src/modules/access/profile-assembler.service.ts` — assemble flow; catch provider throws; normalize S10 `availability` and falsy payloads to `status: 'unavailable'`
- [x] `services/backend/src/modules/access/entities/` + `dto/` — `EmployeeProfileEntity` with top-level `displayName` and per-section union types
- [x] `services/backend/src/modules/access/profile.controller.ts` + Swagger — `GET /employees/:employeeId/profile`; 403 when viewer has no Employee row
- [x] `services/backend/src/modules/access/access.module.ts` — wire assembler, profile controller, S1/S11 providers
- [x] `services/backend/src/modules/timeline/timeline-section.provider.ts` + `integrations/leaves-section.provider.ts` + `integrations/leaves.controller.ts` — adopt `audience` signature; assembler path skips redundant C1; dev route documents dual-path
- [x] Backend tests: `profile-assembler.service.spec.ts`, update `access-resolver.service.spec.ts`, `test/employee-profile.e2e-spec.ts`
- [x] `services/frontend/src/api/hooks/useEmployeeProfile.ts` + types — fetch assembled profile
- [x] `services/frontend/src/pages/EmployeeProfilePage/` — drop `useEmployeeDetail`; Access Scope Chip; section cards (S1, S9, S10, S11 when present); unavailable placeholders; C8-gated Employment block
- [x] `services/frontend/e2e/employee-profile-assembly.spec.ts` — Colleague-trimmed sections vs manager additional sections (seed chain actors)

**Acceptance Criteria:**
- Given any credentialed profile request, when the API responds 200, then `displayName`, `audience.sections`, and `sections` satisfy the normative response shape and wire-safety rules in Boundaries *(matrix: Wire safety)*
- Given Manager M holds ReportingLine access to B, when M requests B's profile, then every ReportingLine-granted section appears at correct R/RW levels, with unbuilt sections marked `unavailable` *(matrix: ReportingLine profile)*
- Given Viewer V is a Colleague of B, when V requests B's profile, then `sections` contains **only** `S1`, `S10`, and `S11` keys *(matrix: Colleague profile / Denied section absent)*
- Given V is a Colleague and S10/S11 are returned, when inspecting payloads, then leave type is withheld and S11 exposes project names only *(matrix: S10 Colleague masking / S11 Colleague narrowing)*
- Given V is a Colleague viewing S1, when the payload is returned, then the mentor field is absent *(D5; matrix: S1 Colleague D5)*
- Given Person X gains a C8 permission without C1 audience change to B, when X requests B's profile before and after assignment, then assembled sections are identical *(matrix: C1 boundary)*

### Review Findings

- [x] [Review][Patch] Backend build fails (TS2352 unsafe cast) [`profile-assembler.service.ts:95`]
- [x] [Review][Patch] Frontend hardcodes S1/S9/S10/S11 cards — granted unavailable sections (e.g. S6) never render [`EmployeeProfilePage.tsx:88-185`]
- [x] [Review][Patch] `functional-role-assignment.service.ts` has CR-only line endings (file corruption in working tree) [`functional-role-assignment.service.ts:1`]
- [x] [Review][Patch] S1 D5 Colleague mentor omission not implemented — provider ignores `audience` [`identity-section.provider.ts:14`]
- [x] [Review][Patch] S11 provider ignores `audience` — Colleague narrowing not enforced at provider layer [`projects-section.provider.ts:19`]
- [x] [Review][Patch] Successful S10 sections leak nested `data.availability` instead of wire-safe leaves DTO [`profile-assembler.service.ts:136`]
- [x] [Review][Patch] Empty object `{}` provider payload not normalized to `status: 'unavailable'` [`profile-assembler.service.ts:148`]
- [x] [Review][Patch] Missing e2e: authenticated user without Employee row → 403 [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Missing e2e: S10 Colleague leave-type masking at `/profile` boundary [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Missing test: D5 mentor field absent for Colleague S1 payloads [`identity-section.provider.ts`]
- [x] [Review][Patch] Missing e2e: malformed UUID → 400 and unknown employee → 404 [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Missing test: C8 permission change does not alter assembled profile (C1 boundary AC) [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Missing unit test: null/undefined provider payload → unavailable [`profile-assembler.service.spec.ts`]
- [x] [Review][Patch] Missing Playwright assertion: Colleague S10 unavailable card visible [`employee-profile-assembly.spec.ts:76`]
- [x] [Review][Patch] `EmployeeProfile.sections` type omits granted section IDs beyond S1/S9/S10/S11 [`employee-profile.ts:92`]
- [x] [Review][Patch] `IdentitySectionProvider` missing deferred-field scope comments per Design Notes [`identity-section.provider.ts:1`]
- [x] [Review][Patch] Missing assembler unit test for multi-audience union section keys [`profile-assembler.service.spec.ts`]
- [x] [Review][Defer] Core Story 1-6 files still untracked in git (assembler, controller, providers, e2e) — deferred, pre-existing
- [x] [Review][Defer] Duplicate Prisma fetch for subject employee (assembler + S1 provider) — deferred, pre-existing

## Design Notes

**Viewer id mapping:** Profile controller resolves authenticated user → `Employee.id` via `employee.findUnique({ where: { userId } })` (same pattern as `leaves.controller.ts`). C1 always receives employee ids, not user ids.

**Leaves dev route dual-path:** `GET /employees/:employeeId/leaves` remains a standalone test surface. After the contract change, the provider accepts optional `audience`; when omitted (dev route), the provider resolves C1 locally as today. The assembler always passes `audience`.

**S1 stub scope:** Bootcamp schema has `User.name/email`, `Employee.managerId`, `peoplePartnerId`. Return those plus relation display names. Defer photo, position, department, mentor until owning stories/schema land — document gaps in provider code comments.

**Epic AC override:** `epics.md` Story 1.6 Colleague AC mentions S10 "incl. leave type" — superseded by `access-model.md` Rule 4 (dates only). Follow access-model.

## Verification

```bash
cd services/backend && npm run build && npm run lint && npm run depcruise
cd services/backend && npm test -- access-resolver.service.spec profile-assembler
cd services/backend && npm run test:e2e -- employee-profile

cd services/frontend && npm run typecheck && npm run lint && npm run build
cd services/frontend && npm run test -- employee-profile-assembly
```

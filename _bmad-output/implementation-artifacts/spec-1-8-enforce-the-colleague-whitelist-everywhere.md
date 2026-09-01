---
title: 'Enforce the Colleague Whitelist Everywhere'
type: 'feature'
created: '2026-09-01'
status: 'done'
review_loop_iteration: 1
baseline_commit: '2baf55d433e6e0cb10169879dc7dd5b9c877b3a2'
story_key: '1-8-enforce-the-colleague-whitelist-everywhere'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-1-6-assemble-employee-profile-by-section-access.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/test/support/access-matrix.ts'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 1.6 enforces the Colleague whitelist on `GET /employees/:id/profile`, but parallel API routes and metadata endpoints can still leak non-whitelist data (user emails via `/users`, custom-field definitions via Self over-permission, un-audited section bypass routes). Epic AC requires the whitelist at the data-access layer on every **API** surface — not only the profile assembler.

**Approach:** Add a shared `SectionAccessGate` in the `access` module that wraps AccessResolver (C1) section checks for any module. Harden every **existing** employee-data route with Colleague-negative behavior and e2e coverage. Export the gate for Epic 3 list (Story 3.5) and export (Story 3.6) without building those surfaces here. Align all AC text to `access-model.md` Rule 4: S1, S10 **dates only** (type hidden), S11 project name only.

**Scope note:** `epics.md` Story 1.8 also mentions export `.xlsx` column whitelist — that AC is deferred to Story 3.6. This story is API-route enforcement only.

## Boundaries & Constraints

**Always:**
- Whitelist is exactly **S1, S10 (dates only — `type`/`approvalState` null), S11 (project name only)** per `access-model.md` Rule 4. Override stale `epics.md` wording ("incl. leave type"). Story 3.6 AC mentioning "S10 leave type" column must be corrected when that spec is written.
- `SectionAccessGate` (`access` module):
  - `requireSection(viewerEmployeeId, subjectEmployeeId, sectionId, minLevel?)` — resolves C1 once; throws `ForbiddenException` when grant is `none` or below `minLevel` (default `'R'`; pass `'RW'` for write-gated surfaces). Provider field narrowing (e.g. S10 dates-only) happens **after** the gate passes.
  - `listGrantedSections(audience: ResolvedAudience): SectionId[]` — returns section ids where `audience.sections[id] !== 'none'`. For Colleague today this is `['S1','S10','S11']` only (not S16 — deferred to Story 1.10). Epic 3 list/export stories import this; do not duplicate C1 iteration logic.
  - Export via `access.module.ts`. Document campaign-sender exception deferral (access-model Rule 7 / Epic 10) in gate service JSDoc — no runtime hook in 1.8.
- **Route inventory:** The I/O matrix below is the sole behavioral source for every route in scope. Routes **out of scope** (already C8-gated in Stories 1.4–1.5; no C1 colleague leak risk): `GET/POST/PATCH/DELETE /functional-roles`, `GET /permissions/catalog`, `GET /permissions/me`, `GET/PUT /employees/:id/functional-roles`. User **write** routes (`POST/PATCH/DELETE /users`) are unchanged — pre-auth/admin scope; not part of this story.
- Colleague `S16` stays **`none`** in `COLLEAGUE_SECTIONS` — colleague-tier custom fields deferred to Story 1.10. `COLLEAGUE_WHITELIST` in `access-matrix.ts` lists S16 for Story 1.10 planning; `listGrantedSections` and production C1 must return only S1/S10/S11 until then.
- Negative e2e file `colleague-whitelist.e2e-spec.ts` covers every matrix row with credentialed Colleague actor from seed chain.
- Enforcement at query/provider layer — never fetch-then-strip in serializers. Parallel section routes (`/leaves`, `/timeline`) must call `requireSection` at the controller entry; providers handle field narrowing only.

**Ask First:**
- If bootcamp self-only `/users` reads need widening (e.g. `manage_functional_roles` holders listing all users), HALT — requires a new product decision beyond this story.

**Never:**
- Do not build list column catalog, filter engine, export, or search (Stories 3.1, 3.5, 3.6) — only export `SectionAccessGate` for them.
- Do not implement campaign-sender exception (access-model Rule 7 / Epic 10).
- Do not change `COLLEAGUE_SECTIONS` to grant S16 read or widen S10 to include leave type.
- Do not build full cross-audience leak harness (Story 1.14).
- Do not add frontend colleague-mode UI (Story 3.6).
- Do not refactor `profile-assembler.service.ts` to use the gate in 1.8 — assembler path is already correct from Story 1.6; avoid dual enforcement paths in one story.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Profile bypass | Colleague V → B's `/profile` | Only `S1`,`S10`,`S11` keys; S10 dates only | 401; 403 no Employee; 404 |
| Leaves parallel route | Colleague V → B's `/leaves` | Dates-only leaves; no `type` | 403 via gate S10; 404 |
| Leaves self | V → own `/leaves` | Dates-only leaves per Self S10 grant | 401; 403 no Employee; 404 |
| Timeline denied | Colleague V → B's `/timeline` | — | 403 via gate S9 |
| Timeline self | V → own `/timeline` | 200; career events per Self S9 grant | 401; 403 no Employee; 404 |
| CF values denied | Colleague V → B's `/custom-fields/values/:id` | — | 403 via gate S16; 401; 404 unknown employee |
| CF definitions leak fix | Colleague V → `GET /custom-fields` | `[]` — no management/employee/colleague-tier names while S16 is `none` | 401 |
| Users list denied | Colleague V → `GET /users` | — | 403 (bootcamp: no `findAll` for Colleague regardless of C8 permissions) |
| Users self read | Colleague V → `GET /users/:ownUserId` | Own `PublicUser` row | 403 other id; 404 |
| Directory list safe | Colleague V → `GET /employees` | All rows; each `{id,displayName}` only | 401 |
| Directory detail safe | Colleague V → `GET /employees/:id` (B) | `{id,displayName}` only — no extra header fields | 401; 404; 400 malformed UUID |
| Viewer without Employee | Authenticated user, no linked `Employee` row → any gated route above | — | 403 before C1 (mirror profile.controller) |
| Manager profile regression | ReportingLine M → B's `/profile` | Full manager section keys unchanged vs pre-1.8 | N/A |
| Manager leaves regression | ReportingLine M → B's `/leaves` | Full leave payloads (incl. `type`) | 404 |
| Manager timeline regression | ReportingLine M → B's `/timeline` | 200; career events | 404 |
| Manager CF definitions regression | ReportingLine M → `GET /custom-fields` | Management/employee-tier definitions visible per role | 401 |
| Gate unit — deny | Unit: `requireSection` Colleague + S9 | Throws `ForbiddenException` | N/A |
| Gate unit — minLevel | Unit: `requireSection` Colleague + S1, `minLevel: 'RW'` | Throws `ForbiddenException` (Colleague S1 is `R`) | N/A |
| Gate unit — grant list | Unit: `listGrantedSections` Colleague audience | `['S1','S10','S11']` | N/A |
| Malformed id | Non-UUID on `/leaves`, `/timeline`, `/custom-fields/values/:id` | — | 400 |

</frozen-after-approval>

## Code Map

Files not duplicated in Tasks (reference only):

- `services/backend/src/modules/access/access-resolver.service.ts:117-135` — `COLLEAGUE_SECTIONS` (no S16 change).
- `services/backend/src/modules/integrations/leaves-section.provider.ts:38-57` — S10 Colleague field masking (after gate passes).
- `services/backend/src/modules/timeline/timeline.service.ts:149-159` — S9 data path (after gate passes).
- `services/backend/src/modules/directory/custom-fields.service.ts:41-66` — `listDefinitions` consumer of visibility fix.
- `services/backend/test/support/access-matrix.ts:224-228` — `COLLEAGUE_WHITELIST` includes S16 for Story 1.10 planning; runtime gate returns S1/S10/S11 only.
- `services/backend/test/employee-profile.e2e-spec.ts:81-122` — existing profile cases; keep as regression.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/access/section-access-gate.service.ts` — implement `requireSection` (with `minLevel` default `'R'`) + `listGrantedSections(audience)` per Boundaries API contract
- [x] `services/backend/src/modules/access/access.module.ts` — export gate service
- [x] `services/backend/src/modules/access/__tests__/section-access-gate.service.spec.ts` — unit tests: S9 deny, S1 RW minLevel deny, `listGrantedSections` Colleague returns S1/S10/S11
- [x] `services/backend/src/modules/integrations/leaves.controller.ts` — call `requireSection(S10)` at route entry before provider; Colleague field masking stays in provider
- [x] `services/backend/src/modules/timeline/timeline.controller.ts` (or equivalent route handler) — call `requireSection(S9)` at route entry
- [x] `services/backend/src/modules/directory/custom-field-visibility.service.ts` — fix `canViewFieldDefinition` to block Colleague metadata leak (no Self S16 RW for catalog)
- [x] `services/backend/src/modules/directory/custom-fields.controller.ts` — `GET /custom-fields/values/:employeeId` returns 403 via gate S16 for Colleague viewers
- [x] `services/backend/src/modules/users/users.controller.ts` + `users.service.ts` — bootcamp self-scope: 403 on `findAll`; `findOne` allowed only for caller's own user id
- [x] `services/backend/src/modules/directory/employees.controller.ts` — document S1-safe summary; verify DTO exposes only `{id,displayName}` on list and detail
- [x] `services/backend/test/colleague-whitelist.e2e-spec.ts` — e2e for all matrix rows (profile regression + leaves + timeline + custom-fields + users + directory + self rows + manager regressions)

**Acceptance Criteria:**
- Given Colleague V and subject B, when V calls any in-scope employee-data route in the matrix, then only whitelist section data is returned or the route returns 403 per matrix *(matrix: profile, leaves, timeline, CF values, directory detail)*
- Given Colleague V, when V calls `GET /users` or another user's `GET /users/:id`, then the response is 403 *(matrix: Users list denied / Users self read)*
- Given Colleague V, when V calls `GET /custom-fields`, then no field definition metadata is returned *(matrix: CF definitions leak fix)*
- Given ReportingLine M and subject B, when M calls `/profile`, `/leaves`, `/timeline`, or `GET /custom-fields`, then behavior is unchanged vs pre-1.8 *(matrix: Manager regression rows)*
- Given V requests own `/leaves` or `/timeline`, when V is Self to the subject, then Self S10/S9 grants apply *(matrix: Leaves self / Timeline self)*
- Given `SectionAccessGate` is exported from `access`, when Epic 3 list/export stories call `listGrantedSections(audience)`, then they receive the correct granted section ids without duplicating C1 logic

## Design Notes

**Directory list:** Full employee browse with `displayName` only is intentional Colleague behavior (Story 3.6 adds column projection, not row filtering). Per-row mixed audience (manager for X, Colleague for Y) is Epic 3 scope.

## Verification

```bash
cd services/backend && npm run build && npm run lint && npm run depcruise
cd services/backend && npm test -- section-access-gate custom-field-visibility leaves.controller timeline
cd services/backend && npm run test:e2e -- colleague-whitelist
```

### Review Findings

- [x] [Review][Patch] Restore `functional-role-assignment.service.ts` formatting [services/backend/src/modules/access/functional-role-assignment.service.ts:1] — file collapsed to a single line with embedded `\r`; blocks lint/review and is out-of-scope noise
- [x] [Review][Patch] Fix non-deterministic custom-field catalog peer selection [services/backend/src/modules/directory/custom-field-visibility.service.ts:82] — `findFirst` with no `orderBy` can pick a non-report peer; managers may get Colleague/S16-none and an empty `GET /custom-fields` list
- [x] [Review][Patch] Return 404 before gate for unknown employees on timeline and CF values [services/backend/src/modules/timeline/timeline.controller.ts:47] [services/backend/src/modules/directory/custom-fields.controller.ts:61] — `requireSection` runs first; unknown UUID yields 403 instead of matrix-expected 404 (leaves controller already checks existence first)
- [x] [Review][Patch] Gate `PUT /custom-fields/:fieldId/values/:employeeId` at controller entry [services/backend/src/modules/directory/custom-fields.controller.ts:80] — spec requires controller-level `requireSection`; only GET values is gated today
- [x] [Review][Patch] Update stale `users.e2e-spec.ts` for bootcamp self-only listing [services/backend/test/users.e2e-spec.ts:47] — still expects `GET /users` → 200; controller now always returns 403
- [x] [Review][Patch] Extend `colleague-whitelist.e2e-spec.ts` matrix coverage [services/backend/test/colleague-whitelist.e2e-spec.ts] — missing 401 cases, 404 for unknown employees, no-employee 403 on profile/timeline/custom-fields, malformed UUID on directory detail
- [x] [Review][Patch] Add e2e for colleague custom-field write denial [services/backend/test/colleague-whitelist.e2e-spec.ts] — no `PUT` case; visibility write path for Colleague+S16-none is untested at HTTP boundary
- [x] [Review][Patch] Reuse `COLLEAGUE_SECTIONS` in catalog fallback [services/backend/src/modules/directory/custom-field-visibility.service.ts:92] — `colleagueAudience()` duplicates C1 colleague grants and can drift from `access-resolver.service.ts`
- [x] [Review][Patch] Align e2e fixtures with bootcamp seed chain [services/backend/test/colleague-whitelist.e2e-spec.ts:215] — spec requires credentialed Colleague from seed chain; test uses ad-hoc `seedGraph()`
- [x] [Review][Patch] Update Swagger for self-only users API [services/backend/src/modules/users/users.controller.ts:44] — `SwaggerFindAllUsers` still documents 200 list response
- [x] [Review][Patch] Strengthen manager profile regression assertion [services/backend/test/colleague-whitelist.e2e-spec.ts:185] — only checks S1/S6 defined; spec expects full manager section keys unchanged vs pre-1.8
- [x] [Review][Patch] Fix verification command reference to missing e2e file [spec-1-8-enforce-the-colleague-whitelist-everywhere.md:121] — `employee-profile.e2e-spec.ts` does not exist under `services/backend/test/`
- [x] [Review][Defer] Timeline double-resolves C1 after controller gate [services/backend/src/modules/timeline/timeline.service.ts:149] — deferred, pre-existing service-layer checks; gate added at controller but service still re-resolves
- [x] [Review][Defer] Duplicated viewer-employee resolution across controllers [services/backend/src/modules/timeline/timeline.controller.ts:113] — deferred, pre-existing pattern before shared helper extraction
- [x] [Review][Defer] Directory queries fetch user email before S1-safe mapping [services/backend/src/modules/directory/employees.service.ts:18] — deferred, pre-existing; DTO surface is S1-safe, no email leak

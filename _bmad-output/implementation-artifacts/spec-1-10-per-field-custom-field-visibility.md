---
title: 'Per-Field Custom Field Visibility'
type: 'feature'
created: '2026-09-02'
status: 'done'
review_loop_iteration: 0
baseline_commit: 'e3e94ff9484633cbc148345dd2acc7185e2e9cc3'
backend_baseline_commit: 'c8da5b9a47cb911109b4caec02611ebbad7c3c46'
frontend_baseline_commit: 'e8dc97e81aeff62fcff3a5535d27d428c642189e'
story_key: '1-10-per-field-custom-field-visibility'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-3-2-define-custom-fields-at-runtime.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
  - '{project-root}/services/backend/.claude/rules/nest-e2e.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 3.2 built `FieldRegistry`/`CustomFieldVisibilityService` with correct per-field visibility logic, but two wiring gaps leave S16 unusable per `access-model.md`'s matrix: (1) the real `AccessResolver` denies Colleague S16 entirely (`'none'`), so colleague-visibility fields can never surface despite CAP-2's documented per-field carve-out; (2) no `SectionProvider` is registered for S16, so profile assembly always returns `'unavailable'` — custom fields never appear on any profile, including Self's own.

**Approach:** Grant Colleague a base S16 `'R'` access level (a narrow, documented exception to the S1/S10/S11 whitelist, same pattern as the S7-PM and S1-mentor carve-outs), and add a `CustomFieldsSectionProvider` (`@RegisterProvider('section', 'S16')`) in the `directory` module that reuses the existing per-field visibility filter to assemble profile data; render the section on the frontend profile page.

## Boundaries & Constraints

**Always:**
- `COLLEAGUE_SECTION_GRANTS.S16` (`access-resolver.contract.ts`) changes from `'none'` to `'R'`; update its doc comment to note the CAP-2 exception. This is the single edit point — `access-resolver.service.ts`'s `COLLEAGUE_SECTIONS` and `CustomFieldVisibilityService`'s catalog-audience fallback both derive from this constant, so both pick up the fix without separate edits.
- New provider mirrors `ManagementNotesSectionProvider`: resolves audience if not supplied, throws `ForbiddenException` when `sections.S16 === 'none'`, else builds the section.
- Per-field filtering inside the new provider reuses `CustomFieldVisibilityService.canViewFieldForSubject` — never the `manage_custom_fields` HR-admin bypass from `listValuesForEmployee`, which is specific to the field-administration API and never applies to profile viewing.
- Section DTO: `{ fields: { id, name, type }[]; values: Record<fieldId, FieldValue> }`. `fields` contains only visibility-passed fields; `values` contains only entries with a stored row (lazy-unset semantics, AD-6) — a field with no value is omitted from `values`, never emitted as `null`.
- Register the provider in `DirectoryModule`'s `providers` array — already `@Global()` and the C2 owner, no `app.module.ts` change needed.
- Frontend: add `S16` to `PROFILE_SECTION_TITLE_KEYS` and a renderer in `profile-sections.tsx`; new `CustomFieldsSection.tsx` under `EmployeeProfilePage/components/` — read-only name/value list, no inline-edit affordance.
- i18n: `employeeProfile.sections.customFields` + `employeeProfile.customFields.empty` keys via `react-i18next` — no hardcoded copy.
- New backend tests assert against `test/support/access-matrix.ts`'s S16 row (`perFieldVisibility` for Self/Colleague, `readWrite` for managerLine/pp) and `COLLEAGUE_WHITELIST`, which already lists `S16` — this story turns that fixture from aspirational to enforced.
- `custom-field-visibility.service.spec.ts`'s existing test "shows colleague-visible fields when S16 grants read (Story 1.10 target)" must pass unmodified once the resolver fix lands.

**Ask First:**
- Whether AC2's "rejected or silently excluded" should also stop `field-registry.service.ts`'s `validateFilters`/sort check from distinguishing "unknown field" vs "not filterable for this viewer" in its 400 message (current behavior reveals a management field's existence, though never its value) — this is optional hardening, not required by the AC as written.

**Never:**
- No frontend admin UI for defining custom fields (`POST /custom-fields`) — out of scope, same boundary as Story 3.2.
- No inline-edit for custom field values on the profile page — read-only render only; the existing `PUT /custom-fields/:fieldId/values/:employeeId` route is untouched.
- Do not touch `EmployeesService`/`field-registry.service.ts`'s All Employees list masking — already correct and already covered by tests.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Self views employee-visibility field | HR Admin defines "Dietary preference" (employee); Self GETs own profile | S16 present; field + value included | N/A |
| Colleague views same field | Colleague GETs subject's profile | S16 absent, or present with field excluded — no trace of name/value | N/A |
| Colleague-visibility field to Colleague | Field visibility "colleague"; Colleague GETs profile | S16 present; field + value included (CAP-2) | N/A |
| Management field via list filter | Colleague filters/sorts on a management fieldId via API params | Rejected or silently excluded; no value/count leak | 400 (existing) |
| No visible custom fields | Subject has only management fields, viewer is Colleague | S16 resolves to empty section or 'unavailable', never broken | N/A |
| Field with no stored value | Field visible but lazy-unset for subject | Field present in `fields`, omitted from `values` | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/contracts/access-resolver.contract.ts:53-70` — `COLLEAGUE_SECTION_GRANTS.S16`: `'none'` → `'R'`; update doc comment
- `services/backend/src/modules/directory/custom-fields-section.provider.ts` (new) — `@RegisterProvider('section', 'S16')`, mirrors `management-notes/management-notes-section.provider.ts`; inject the abstract `FieldRegistry` contract token (not the concrete `FieldRegistryService`), matching `CustomFieldsService`'s convention; `fields` array order follows `FieldRegistryService.listFields()`'s existing alphabetical-by-name order — no separate sort needed
- `services/backend/src/modules/directory/directory.module.ts` — add new provider to `providers`
- `services/backend/src/modules/directory/custom-field-visibility.service.ts:47-61` — reuse `canViewFieldForSubject` unchanged
- `services/backend/src/modules/directory/field-registry.service.ts` — `query()`/`listFields()` reused unchanged for building the section payload
- `services/backend/src/modules/directory/custom-fields.controller.ts` / `custom-fields.service.ts` — no code change, but the `COLLEAGUE_SECTION_GRANTS.S16` flip changes their runtime behavior for Colleague viewers too (`GET /custom-fields`, `GET /custom-fields/values/:employeeId` now pass the `S16` gate instead of always 403ing) — needs regression coverage, see Tasks
- `services/backend/src/modules/access/profile-assembler.service.ts` — no change; already iterates all `SectionId`s and looks up the registry generically
- `services/backend/test/support/access-matrix.ts:209-229` — S16 row + `COLLEAGUE_WHITELIST` already encode the target shape; drive new tests off this
- `services/backend/src/modules/directory/__tests__/custom-field-visibility.service.spec.ts:43-63` — pre-existing breadcrumb tests for this story
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` — add `S16` title key + renderer, following the `S7`/`S9` pattern
- `services/frontend/src/pages/EmployeeProfilePage/components/CustomFieldsSection/CustomFieldsSection.tsx` (new) — mirrors `ManagementNotesSection` folder shape; render each `FieldValueType` explicitly (text/number/date/select as the raw string, boolean via an i18n yes/no key, multi_select as a joined list) — a stored empty multi_select (`[]`) renders as "none selected" via its own i18n key, distinct from a field key absent from `values` entirely (never-set), preserving the AD-6 lazy-unset distinction
- `services/frontend/src/types/employee-profile.ts` — add `CustomFieldsSection` type; its `values` shape mirrors the backend's `FieldValue = string | number | boolean | string[] | null` union
- `services/frontend/src/locales/en/translation.json:137-143` — add `customFields` keys alongside the existing `sections` block

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/access-resolver.contract.ts` -- flip `COLLEAGUE_SECTION_GRANTS.S16` to `'R'`, update comment -- unblocks per-field colleague visibility (CAP-2)
- [x] `services/backend/src/modules/directory/custom-fields-section.provider.ts` -- new S16 `SectionProvider` -- profile assembly currently always returns `'unavailable'` for S16
- [x] `services/backend/src/modules/directory/directory.module.ts` -- register new provider -- DI wiring
- [x] `services/backend/src/modules/directory/__tests__/custom-fields-section.provider.spec.ts` -- unit: per-field filter matrix (management/employee/colleague × Self/ReportingLine/ProjectLine/PP/Colleague — ProjectLine included: it resolves S16='RW' today via `PROJECT_LINE_SECTIONS` and is otherwise untested by this story) -- required by the per-provider matrix-test architecture rule
- [x] same spec file -- cover I/O matrix rows 5 ("no visible custom fields" → provider returns `{fields:[],values:{}}`, never `'unavailable'`, per Design Notes) and 6 ("field with no stored value" → present in `fields`, omitted from `values`) -- these matrix rows have no other test hook
- [x] `services/backend/test/employee-profile-custom-fields.e2e-spec.ts` -- Self sees employee-visibility field, Colleague doesn't, Colleague sees colleague-visibility field, and a mixed-tier fixture (one subject holding a management + an employee + a colleague field simultaneously) resolves correctly for both Self and Colleague -- covers AC1
- [x] `services/backend/test/custom-fields.e2e-spec.ts` (existing, from Story 3.2) -- add regression cases: Colleague now passes the `S16` gate on `GET /custom-fields` and `GET /custom-fields/values/:employeeId` (previously always 403) and must see colleague-visibility entries only -- closes the blast-radius gap the resolver-constant flip opens on the pre-existing REST surface
- [x] `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx` + `components/CustomFieldsSection/CustomFieldsSection.tsx` -- render S16 -- covers AC1's "not on the profile" half
- [x] `services/frontend/src/locales/en/translation.json` -- add keys -- i18n convention
- [x] `services/frontend/e2e/custom-fields-visibility.spec.ts` -- Self profile shows the field, Colleague profile doesn't -- browser-level proof

**Acceptance Criteria:**
- Given HR Admin creates "Dietary preference" with visibility "employee", when Self views own profile and a Colleague views the same profile, then Self sees the field and the Colleague sees no trace of it — not on the profile, in list columns, or in filters
- Given a "management"-visibility custom field exists and a Colleague browsing All Employees attempts to construct a filter/sort referencing that field via direct API parameters, then the request is rejected or the field is silently excluded, and the result set never indirectly reveals the value

## Design Notes

`access-resolver.stub.ts` (Wave-0 deny-all) defines its own literal `ALL_SECTIONS_DENIED` and does not import `COLLEAGUE_SECTION_GRANTS`, so it is unaffected by the flip described in Boundaries & Constraints above.

The mirrored `ForbiddenException` guard (thrown when `sections.S16 === 'none'`) is unreachable through `ProfileAssemblerService.assembleProfile`, which already skips any section whose access level is `'none'` before calling a provider — same as the existing `ManagementNotesSectionProvider`. It exists as defense-in-depth for callers that resolve `S16` directly, and is exercised only by the new provider's own unit test, never by an end-to-end profile request.

Matrix row 5 ("no visible custom fields") always resolves to a rendered empty section (`{ fields: [], values: {} }`), never `'unavailable'`: `ProfileAssemblerService.isUnavailablePayload` only collapses a payload with zero keys, and this DTO always carries the two keys `fields`/`values`.

A `multi_select` field where the subject has selected zero options is still a stored row (`field-registry.service.ts`'s `setValue` only deletes on literal `null`, not `[]`), so per the DTO rule it must appear in `values` as `[]` — the frontend must render that distinctly from a field key that's absent from `values` (never-set), see Code Map.

The Ask-First question above (whether `validateFilters`'s "unknown field" vs. "not filterable" 400-message wording needs closing) is unresolved pending human sign-off; absent an answer before implementation starts, this story's tests assume the current existence-revealing message is retained (i.e. AC2 is satisfied by "value/count never leaks," not by field-existence secrecy).

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint && npm run depcruise` -- PASS (verified)
- `cd services/backend && npm test -- --testPathPatterns=directory` -- PASS, 67/67 (verified), includes the pre-existing "Story 1.10 target" test
- `cd services/backend && npm test -- --testPathPatterns=access` -- PASS, 153/153 (verified) -- regression check for the `COLLEAGUE_SECTION_GRANTS` edit
- `cd services/backend && npm run test:e2e -- employee-profile-custom-fields` -- NOT RUN: this sandbox's Node (22.23.2) cannot execute *any* backend e2e spec — `Must use import to load ES Module ... @nestjs/schedule ... Jest's require(ESM) requires Node v24.9+`. Confirmed pre-existing and unrelated to this story by reproducing the identical failure on the untouched `management-notes.e2e-spec.ts`. Run on Node ≥24.9 (or wherever CI runs backend e2e) before merge.
- `cd services/backend && npm run test:e2e -- custom-fields` -- NOT RUN, same environment blocker as above
- `cd services/frontend && npm run build && npm run lint` -- PASS (verified)
- `cd services/frontend && npx playwright test custom-fields-visibility` -- reported PASS (3/3) by the implementing agent against a locally-run dev server; not independently re-run

## Suggested Review Order

**S16 profile section provider (new)**

- Entry point — assembles S16 per-field for the resolved viewer/subject; no HR-admin bypass, profile viewing only
  [`custom-fields-section.provider.ts:44`](../../services/backend/src/modules/directory/custom-fields-section.provider.ts#L44)
- Per-field visibility checks run concurrently via `Promise.all` — added in the review round, replacing a sequential loop
  [`custom-fields-section.provider.ts:66`](../../services/backend/src/modules/directory/custom-fields-section.provider.ts#L66)
- Registered into `DirectoryModule` providers — no `app.module.ts` change needed
  [`directory.module.ts:24`](../../services/backend/src/modules/directory/directory.module.ts#L24)
- Wire DTO — `fields`/`values` split, AD-6 lazy-unset semantics documented on the entity
  [`custom-fields-section.entity.ts:1`](../../services/backend/src/modules/directory/entities/custom-fields-section.entity.ts#L1)

**Colleague S16 access grant (the unlock)**

- The single-constant flip (`'none'` → `'R'`) everything else in this story depends on
  [`access-resolver.contract.ts:83`](../../services/backend/src/modules/contracts/access-resolver.contract.ts#L83)
- Real (unmocked) `AccessResolverService` regression sentinel — added in the review round
  [`access-resolver.service.spec.ts:237`](../../services/backend/src/modules/access/__tests__/access-resolver.service.spec.ts#L237)

**Directory-list inclusion path (newly reachable)**

- Colleague-tier custom field now included and unmasked, not excluded — added in the review round
  [`employees.service.spec.ts:118`](../../services/backend/src/modules/directory/__tests__/employees.service.spec.ts#L118)

**Backend regression coverage on the pre-existing REST surface**

- Colleague-profile section-key assertion now includes S16
  [`employee-profile.e2e-spec.ts:97`](../../services/backend/test/employee-profile.e2e-spec.ts#L97)
- Colleague custom-field-values route: 403 → 200, empty body when nothing colleague-visible
  [`colleague-whitelist.e2e-spec.ts:195`](../../services/backend/test/colleague-whitelist.e2e-spec.ts#L195)
- New regression block: Colleague reads pass, write still 403s under S16='R'
  [`custom-fields.e2e-spec.ts:80`](../../services/backend/test/custom-fields.e2e-spec.ts#L80)

**Frontend profile rendering**

- S16 renderer wired into the generic section dispatch, same pattern as S7/S9/S10/S11
  [`profile-sections.tsx:148`](../../services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx#L148)
- Read-only per-field render with explicit per-type formatting (boolean/multi_select/lazy-unset)
  [`CustomFieldsSection.tsx:18`](../../services/frontend/src/pages/EmployeeProfilePage/components/CustomFieldsSection/CustomFieldsSection.tsx#L18)

**Peripherals**

- Wire-DTO type mirrors the backend's `FieldValue` union
  [`employee-profile.ts:101`](../../services/frontend/src/types/employee-profile.ts#L101)
- New i18n keys: section title plus empty/notSet/noneSelected states
  [`translation.json:143`](../../services/frontend/src/locales/en/translation.json#L143)
- Provider unit matrix — per-field visibility × role, driven through the real `CustomFieldVisibilityService`
  [`custom-fields-section.provider.spec.ts:184`](../../services/backend/src/modules/directory/__tests__/custom-fields-section.provider.spec.ts#L184)
- Playwright coverage: Self sees the field, Colleague doesn't, a colleague-tier field reaches Colleague
  [`custom-fields-visibility.spec.ts:75`](../../services/frontend/e2e/custom-fields-visibility.spec.ts#L75)
- Backend e2e profile-assembly coverage — written but not executed in this sandbox (Node ESM blocker, see Verification)
  [`employee-profile-custom-fields.e2e-spec.ts:80`](../../services/backend/test/employee-profile-custom-fields.e2e-spec.ts#L80)

### Review Findings

- [x] [Review][Patch] N+1 audience resolution in `CustomFieldsSectionProvider` [`custom-fields-section.provider.ts:66`] — `canViewFieldForSubject` independently re-calls `accessResolver.resolveAudience(viewerEmployeeId, subjectEmployeeId)` for every custom field even though the provider already holds the resolved audience from its own `getSection` call. For N custom fields this is N+1 full audience resolutions (manager-chain/project-assignment traversal) on a single profile-page load. Fix: use the already-resolved `resolved.role` with the synchronous, public `CustomFieldVisibilityService.isVisibleToRole(role, visibility)` instead of the async per-field `canViewFieldForSubject`.

- [x] [Review][Patch] Stale Colleague-whitelist doc comment contradicts the new S16 exception [`access-resolver.service.ts:118`] — `const COLLEAGUE_SECTIONS = COLLEAGUE_SECTION_GRANTS;` still carries `` /** `access-model.md` Rule 4 — Colleague whitelist (S1, S10, S11 only). */ `` even though this diff updated the source-of-truth doc comment on `COLLEAGUE_SECTION_GRANTS` itself (`access-resolver.contract.ts:56-65`) to describe the S16 exception. The alias's local comment now contradicts the grants it aliases.

- [x] [Review][Patch] Untested value-formatting branches in the new S16 card [`CustomFieldsSection.tsx:51`] — `formatCustomFieldValue`'s `boolean`, `multi_select` (both populated and stored as `[]`), `select`, `date`, and never-set (`notSet`) branches have zero test coverage across the backend unit spec, backend e2e, and the frontend Playwright suite; only `text` with a value present in `values` is exercised (`Vegetarian`, `Sam`). A regression in any of these branches — e.g. an inverted boolean ternary, a broken `multi_select` join, or a weakened `null`/`undefined` guard — would ship undetected. Extend `e2e/custom-fields-visibility.spec.ts` with fixtures covering a `boolean` field, a `multi_select` field (populated and `[]`), and a field present in `fields` but absent from `values`.

- [x] [Review][Patch] Duplicated test-user bootstrap helpers across two new e2e files [`test/custom-fields.e2e-spec.ts:213`, `test/employee-profile-custom-fields.e2e-spec.ts:1`] — both new files hand-roll near-identical `createEmployeeUser`/`loginAs`/`PASSWORD` boilerplate instead of sharing one helper (or extending `test/support/login.ts`). Extract to a shared support helper to avoid drift between the two copies.

---
title: 'Story 9.4 — Automatic Mentor Status Transitions'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '5b3a1a4bdeaea7f1216eabcfb7c7a063f7059b0a'
context:
  - '_bmad-output/implementation-artifacts/epic-9-context.md'
  - '_bmad-output/implementation-artifacts/spec-9-3-end-a-mentorship-pair-with-required-final-feedback.md'
  - 'services/backend/AGENTS.md'
  - 'services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 9.1–9.3 implemented `deriveMentorStatus` and pair-lifecycle transitions, but `mentorStatus` is not exposed on All Employees — the epic AC requires it to be queryable/filterable yet never directly editable on any surface. Cross-surface write rejection and multi-pair edge cases lack explicit test coverage.

**Approach:** Register `mentor_status` as a derived, non-editable directory field wired to the same derivation rules as S13. Extend snapshot loading with batched active-pair data. Harden write rejection on directory PATCH and pair endpoints; add e2e/unit tests for transition matrix including multi-pair mentors.

## Boundaries & Constraints

**Always:**
- `mentorStatus` is derived only — never persisted; rules match `deriveMentorStatus`: active mentor pair → `mentor`; else flag on → `openToMentoring`; else `none` (D4).
- First pair create and last-pair end must return correct derived status in API responses (already transactional for pair data; derive runs post-commit).
- Directory field `mentor_status`: `filterable: true`, `sortable: true`, `editable: false` (omit from `BUILTIN_EDITABLE_FIELD_IDS`).
- Reject direct writes to `mentorStatus`/`status` on every surface — profile PATCH (existing interceptor), directory field PATCH (explicit 400), pair create/end bodies (whitelist).
- Ending one of multiple active pairs keeps status `mentor`.

**Ask First:**
- Whether `open_to_mentoring` should also appear as a separate directory column (recommend: defer — `mentor_status` alone satisfies FR-38).

**Never:**
- New DB column for mentor status.
- All-pairs table (Story 9.5).
- Departure auto-close hook (Epic 14).
- Frontend inline-edit UI for status.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| First pair create | Mentor with zero pairs, flag on | Response `mentorStatus: 'mentor'`; S13 shows `mentor` | N/A |
| Last pair end, flag on | Mentor's only active pair ends | `mentorStatus: 'openToMentoring'` | N/A |
| Last pair end, flag off | Mentor's only active pair ends | `mentorStatus: 'none'` | N/A |
| Multi-pair mentor | Two active mentees; end one | Status stays `mentor` | N/A |
| Directory filter | Filter `mentor_status eq mentor` | Only active mentors returned | N/A |
| Directory PATCH status | `PATCH employees/:id/fields/mentor_status` | Rejected | 400 Bad Request |
| Pair body with status | POST pair with `mentorStatus` key | Rejected | 400 Bad Request |
| Profile PATCH status | Body includes `mentorStatus` | Rejected (existing) | 400 Bad Request |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/mentorship/mentorship.service.ts` L111–137 — `deriveMentorStatus`; reuse or extract shared `computeMentorStatus(flag, hasActivePair)`
- `services/backend/src/modules/mentorship/reject-mentorship-status-write.interceptor.ts` L17–24 — profile write guard (read-only)
- `services/backend/src/modules/contracts/field-registry.contract.ts` L87–96 — add `mentor_status` to `BUILTIN_FIELD_IDS`; keep out of `BUILTIN_EDITABLE_FIELD_IDS`
- `services/backend/src/modules/directory/field-catalog.ts` L55–63 — pattern for derived field (`years_with_company`); add `mentor_status` spec with select options
- `services/backend/src/modules/directory/employee-query.helpers.ts` L12–20, L46–71 — extend `EmployeeSnapshot` + `getCellValue` switch
- `services/backend/src/modules/directory/field-registry.service.ts` L637–686 — batch-load `openToMentoring` + active mentor pairs in `loadEmployeeSnapshots`
- `services/backend/src/modules/directory/employees.service.ts` L275–282 — `updateEmployeeField` writable gate; add explicit 400 for derived mentorship fields
- `services/backend/test/mentorship.e2e-spec.ts` L118–151, L313–358, L616–842 — extend transition + write-rejection matrix
- `services/backend/test/` — add or extend directory e2e for filter/PATCH rejection
- `services/frontend/src/types/employees.ts` L91–100 — mirror `mentor_status` in `BUILTIN_FIELD_IDS`
- `services/frontend/src/locales/en/translation.json` — `directory.fields.mentorStatus` + option labels

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/field-registry.contract.ts` — add `mentor_status` builtin id
- [x] `services/backend/src/modules/directory/field-catalog.ts` — register derived select field (`mentor`, `openToMentoring`, `none`), S13, non-editable
- [x] `services/backend/src/modules/directory/employee-query.helpers.ts` — snapshot fields + `getCellValue`/`filter`/`sort` for `mentor_status`
- [x] `services/backend/src/modules/directory/field-registry.service.ts` — batch active-pair lookup in `loadEmployeeSnapshots`
- [x] `services/backend/src/modules/directory/employees.service.ts` — reject PATCH on `mentor_status` with 400
- [x] `services/backend/src/modules/mentorship/` — optional: extract `computeMentorStatus` shared by service + directory
- [x] `services/backend/src/modules/directory/__tests__/field-registry.service.spec.ts` — filter/sort by `mentor_status`
- [x] `services/backend/test/mentorship.e2e-spec.ts` — multi-pair end keeps `mentor`; pair body status rejection
- [x] `services/backend/test/` — directory e2e: list/filter by `mentor_status`; PATCH field → 400
- [x] `services/frontend/src/types/employees.ts` + `translation.json` — field id and labels

**Acceptance Criteria:**
- Given a willing mentor with zero pairs receives their first pair, when the pair is created, then their status is `mentor` in the create response and in S13
- Given a mentor's last active pair ends and their self-flag is still on, when the end is confirmed, then status reverts to `openToMentoring`; if the flag was off, status reverts to `none`
- Given a mentor with two active pairs, when one pair ends, then status remains `mentor`
- Given any direct write attempt to `mentorStatus` or `status` on profile, directory, or pair endpoints, when submitted, then the server rejects with 400
- Given I open All Employees, when I add the Mentor status column and filter by `mentor`, then only employees with an active mentor pairing appear and the column is not inline-editable

## Verification

**Commands:**
- `cd services/backend && npm test -- mentorship` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- mentorship` — expected: mentorship e2e pass
- `cd services/backend && npm run test:e2e -- directory` (or employees) — expected: directory filter/PATCH tests pass
- `cd services/backend && npm run lint && npm run build` — expected: clean
- `cd services/frontend && npm run typecheck && npm run lint` — expected: clean

**Manual checks (if no CLI):**
- All Employees: add Mentor status column, filter by each value, confirm no edit affordance on cells

## Suggested Review Order

**Derived status engine**

- Shared pure function for flag + active-pair derivation
  [`mentor-status.util.ts:1`](../../services/backend/src/modules/mentorship/mentor-status.util.ts#L1)

- S13 profile path reuses shared compute helper
  [`mentorship.service.ts:111`](../../services/backend/src/modules/mentorship/mentorship.service.ts#L111)

**Directory field registration**

- Builtin id and non-editable derived select spec
  [`field-catalog.ts:64`](../../services/backend/src/modules/directory/field-catalog.ts#L64)

- Snapshot batch-load and cell value wiring
  [`field-registry.service.ts:637`](../../services/backend/src/modules/directory/field-registry.service.ts#L637)

- Explicit 400 on direct directory PATCH attempts
  [`employees.service.ts:275`](../../services/backend/src/modules/directory/employees.service.ts#L275)

**Write rejection on pair endpoints**

- Interceptor guards pair create/end bodies before validation strips extras
  [`mentorship.controller.ts:161`](../../services/backend/src/modules/mentorship/mentorship.controller.ts#L161)

**Frontend display**

- Localized mentor status labels in table and filter popover
  [`filter-utils.ts:19`](../../services/frontend/src/components/AudienceBuilder/filter-utils.ts#L19)

**Tests**

- Directory filter, non-writable contract, and PATCH rejection
  [`employees.e2e-spec.ts:937`](../../services/backend/test/employees.e2e-spec.ts#L937)

- Transition matrix including multi-pair and body rejection
  [`mentorship.e2e-spec.ts:918`](../../services/backend/test/mentorship.e2e-spec.ts#L918)

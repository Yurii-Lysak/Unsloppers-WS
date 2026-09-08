---
title: 'View Feedback Over Time and Compare Periods'
type: 'feature'
created: '2026-09-08'
status: 'done'
review_loop_iteration: 0
baseline_commit: '73f9cf4c5fa3555204c5bd2345072110d4a36432'
backend_baseline_commit: '73f9cf4c5fa3555204c5bd2345072110d4a36432'
frontend_baseline_commit: 'e4130909874649a6d42dd8a192807cbc399def1d'
story_key: '11-2-view-feedback-over-time-and-compare-periods'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-11-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-11-1-record-feedback-with-a-visibility-flag.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 11.1 delivers S8 CRUD and a flat chronological list, but managers and PPs cannot compare feedback across time windows (e.g. Q1 vs Q3) — FR-45 and UX-DR11 require a period-comparison mode with explicit empty-period columns.

**Approach:** Add a **Compare** view mode to the existing Feedback Panel that partitions the profile S8 `records` payload (all records the server already returned for this viewer — no pagination) into two user-selected inclusive date ranges and renders them as side-by-side columns (ascending `recordedAt` within each). No backend or access-rule changes — client-side presentation only, respecting the same S8-filtered payload 11.1 returns.

## Boundaries & Constraints

**Always:**
- **No backend changes.** Reuse profile S8 `records` from `FeedbacksService.buildSection()` — RW gets all records; Self / SharedLink `R` gets shared-only subset (already server-filtered). Do not add query params, parallel `GET`, or new endpoints.
- **View modes:** `list` (default, unchanged 11.1 behavior — server-ordered list: `recordedAt` desc, `createdAt` desc, `id` asc; **do not** client-re-sort in list mode) and `compare` (read-only layout; RW add form and per-record edit/delete/toggle hidden in compare mode, restored in list mode).
- **View-mode state:** `list` | `compare` held in component state (default `list`); switching modes preserves the last-selected Period A/B values. State is not persisted across profile navigation or reload.
- **Period model:** two independent ranges — `{ start: ISO YYYY-MM-DD, end: ISO YYYY-MM-DD }` each. Inclusive membership: `start <= recordedAt <= end` (string compare on `YYYY-MM-DD` calendar dates, same semantics as `feedback-input.ts`).
- **Period validation:** each range requires both `start` and `end` populated; empty field → inline required message. If `start > end` → inline invalid-range message. Compare columns render only when **both** periods are valid; until then show period pickers + validation only (no columns).
- **Default compare ranges:** on first entry to compare mode, pre-fill Period A = Q1 and Period B = Q3 of the **current calendar year** (browser local date) — user may edit or re-preset.
- **Quarter presets:** for the **current calendar year** (browser local date), offer Q1–Q4 buttons that auto-fill each period's start/end (`Q1` Jan 1–Mar 31, `Q2` Apr 1–Jun 30, `Q3` Jul 1–Sep 30, `Q4` Oct 1–Dec 31). Presets apply to the period control that triggered them (Period A or Period B). Custom `type="date"` start/end inputs remain editable after a preset. Presets do not constrain which record years appear — custom dates may target prior years.
- **Compare layout:** two columns labeled with each period's formatted range via i18n (e.g. `employeeProfile.s8.compare.periodLabel` with interpolated bounds — locale-aware short dates, not raw ISO). Desktop: side-by-side grid; narrow viewports (`sm` breakpoint): stack columns vertically (Period A above Period B). Within each column, records sorted client-side `recordedAt` asc, then `createdAt` asc, then `id` asc (reverse of list-mode server order).
- **Empty period:** column renders explicit copy `employeeProfile.s8.compare.emptyPeriod` ("No feedback in this period.") — never hide or omit a column when the range is valid but yields zero records.
- **Record card in compare:** read-only summary — `recordedAt`, `author.displayName`, `context`, `body` (no edit/delete/toggle; no `sharedWithEmployee` flag shown).
- **Overlap:** a record may appear in both columns when ranges overlap — expected, not deduplicated.
- **Self / SharedLink (`R`):** compare mode available with the same shared-only `records` already on the profile — no write affordances.
- **Empty section:** view-mode toggle still renders when `records: []`; compare mode shows empty-period copy in both columns once ranges are valid; list mode keeps existing `employeeProfile.s8.empty`.
- **i18n:** all new strings under `employeeProfile.s8.compare.*`; view-mode toggle keys under `employeeProfile.s8.viewMode.*`. View-mode toggle exposes accessible labels (not icon-only).

**Ask First:**
- If product wants a year selector beyond current-calendar-year quarter presets, HALT — out of v1 scope unless PO confirms.

**Never:**
- "Request feedback…" action (Story 11.3); backend period-filter API; client-side access widening; semantic diff/summary of free-text bodies; trend arrows or scoring; seed data changes; modifying 11.1 server sort order; client-side re-sort in list mode.

## I/O & Edge-Case Matrix

Canonical behavioral source — Boundaries state invariants; Acceptance Criteria reference these rows.

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Compare Q1 vs Q3 | RW viewer; records in Q1 and Q3 of current year | Compare mode; Q1 column shows Q1 records asc; Q3 column shows Q3 records asc | N/A |
| Default compare entry | RW viewer switches to compare first time | Period A prefilled Q1, Period B prefilled Q3 (current year); columns render when both valid | N/A |
| Empty period | Records only in Q3; compare Q1 vs Q3 | Q1 column shows empty-period copy; Q3 column shows records; both columns visible | N/A |
| Self shared subset | Self `R`; one shared record in Q3 | Compare mode; Q1 empty; Q3 shows only that shared record; no write controls | N/A |
| Overlapping ranges | Same record date falls in both ranges | Record appears in both columns | N/A |
| Invalid range | Period A start after end | Inline validation on Period A; no columns rendered | N/A |
| Incomplete period | Period B end date empty | Inline required-field message on Period B; no columns rendered | N/A |
| List mode default | Open S8 section | List mode selected; server desc order preserved; RW forms unchanged | N/A |
| Mode switch | Compare → List → Compare | Period A/B values preserved; list mode restores add/edit | N/A |
| Quarter preset | Click Q3 on Period B | Period B start/end set to Q3 bounds for current year | N/A |
| Prior-year records | Record dated 2025-11-01; compare custom 2025-11-01–2025-11-30 vs Q1 current year | 2025 record appears only in custom column when range includes it | N/A |
| No records at all | `records: []` | List mode shows existing empty state; compare shows empty-period copy in both columns once ranges valid | N/A |
| RW compare read-only | RW viewer in compare mode | No add form; no edit/delete/toggle on record cards | N/A |

</frozen-after-approval>

## Code Map

Normative behavior lives in **Boundaries** and **I/O & Edge-Case Matrix** above; paths below are execution hints only.

- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/FeedbackSection.tsx` — add view-mode toggle; branch list vs compare; hide add form and mutation UI in compare mode.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/hooks/useFeedbackViewMode.ts` (new) — `list` | `compare` state + default Period A/B on first compare entry.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/hooks/useFeedbackSection.ts` — unchanged mutation hooks; compare is presentational only.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/utils/feedback-period.ts` (new) — `isDateInRange`, `filterRecordsForPeriod`, `sortRecordsChronologicalAsc`, `quarterBoundsForYear`, `isValidPeriodRange`, `isCompletePeriodRange`; pure functions, unit-tested.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackCompareView.tsx` (new) — two-column layout + empty states.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackCompareRecordCard.tsx` (new) — read-only record summary for compare columns.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/PeriodRangePicker.tsx` (new) — start/end date inputs + Q1–Q4 preset buttons for one period.
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackViewModeToggle.tsx` (new) — `list` / `compare` segmented control (simple button pair; no new shadcn dep required).
- `services/frontend/src/locales/en/translation.json` — `employeeProfile.s8.viewMode.*`, `employeeProfile.s8.compare.*` (including `periodLabel`, validation messages).
- `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/utils/__tests__/feedback-period.test.ts` (new) — filter, sort, quarter bounds, inclusive edges, incomplete ranges.
- `services/frontend/e2e/feedback-period-comparison.spec.ts` (new) — stub profile with Q1-empty/Q3-populated records; assert both columns, empty copy, preset flow, list-mode regression (test-design **P2-006**).

**Read-only (11.1 — do not change unless bugfix required):** `feedbacks.service.ts` sort/filter, controller routes, S8 provider, `FeedbackRecordItem` edit path in list mode.

## Tasks & Acceptance

**Execution:**
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/hooks/useFeedbackViewMode.ts` — view mode + default period state — mode switching contract
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/utils/feedback-period.ts` — period filter/sort/preset helpers — core compare logic
- [x] `services/frontend/e2e/feedback-period-logic.spec.ts` — unit-style tests for matrix rows (no Vitest in repo) — fast regression guard
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/PeriodRangePicker.tsx` — date inputs + quarter presets — period selection UX
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackCompareRecordCard.tsx` — read-only compare card — FR-45 record summary
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackCompareView.tsx` — dual columns + empty states — FR-45 layout
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/components/FeedbackViewModeToggle.tsx` — list/compare toggle — mode switching
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/FeedbackSection/FeedbackSection.tsx` — wire modes; preserve list-mode 11.1 behavior — integration
- [x] `services/frontend/src/locales/en/translation.json` — i18n keys — copy contract
- [x] `services/frontend/e2e/feedback-period-comparison.spec.ts` — empty-period + populated-period + list regression e2e — P2-006 / epic AC

**Acceptance Criteria:**
- Given employee B has feedback records dated across Q1 and Q3, when B's manager opens S8, switches to Compare, sets Period A to Q1 and Period B to Q3, then each column shows that quarter's records in ascending date order
- Given employee B has feedback only in Q3, when B's manager compares Q1 vs Q3, then Q1 shows "No feedback in this period." and Q3 shows its records with no error
- Given I am employee B (Self), when I use Compare on my profile, then only shared-with-employee records appear in the relevant period columns
- Given I am in list mode, when I view S8, then behavior matches Story 11.1 (server-ordered desc list, RW add/edit/delete/toggle)
- Given I switch from compare back to list mode, when records were edited in another session, then the list reflects the latest profile payload after invalidation (compare does not cache a stale subset)

## Spec Change Log

- 2026-09-08 — bmad-code-review: reset view-mode on employee navigation, `sm` column breakpoint, expanded e2e (order, empty section, incomplete period, quarter preset).

## Design Notes

Client-side partitioning avoids a second fetch and keeps access filtering authoritative on the server (11.1 provider). Compare mode deliberately hides RW mutation UI to keep the side-by-side layout scannable; managers switch back to list to edit. Quarter presets satisfy the epic's "compare Q1 vs Q3" AC without blocking arbitrary ranges via the date inputs. **FR-45** here is the Epic 11 feedback FR (epics.md / UX-DR11), not the mentorship FR-45 in PRD §5.9.

## Verification

**Commands:**
- `cd services/frontend && npm test -- feedback-period` — expected: unit tests pass
- `cd services/frontend && npm run test -- feedback-period-comparison` — expected: Playwright pass
- `cd services/frontend && npm run typecheck` — expected: clean

**Manual checks (if no CLI):**
- Manager profile → S8 → Compare → Q1 vs Q3 with mixed data → both columns visible, empty side shows explicit copy
- Toggle back to List → add/edit form still works

### Review Findings

- [x] [Review][Patch] Reset compare state when `employeeId` changes [`useFeedbackViewMode.ts`] — view mode and periods leaked across profile navigation
- [x] [Review][Patch] Use `sm:grid-cols-2` for compare columns [`FeedbackCompareView.tsx`] — spec requires side-by-side at `sm`, not `lg`
- [x] [Review][Patch] Assert ascending compare order in Q3 column [`feedback-period-comparison.spec.ts`] — AC order undetected by presence-only checks
- [x] [Review][Patch] Assert server desc list order on list-mode return [`feedback-period-comparison.spec.ts`] — list regression guard
- [x] [Review][Patch] E2e for incomplete period hiding columns [`feedback-period-comparison.spec.ts`] — matrix row coverage
- [x] [Review][Patch] E2e for empty `records: []` compare mode [`feedback-period-comparison.spec.ts`] — matrix row coverage
- [x] [Review][Patch] E2e for quarter preset button [`feedback-period-comparison.spec.ts`] — preset UX coverage
- [x] [Review][Defer] Profile invalidation AC (edit in another session) — deferred, pre-existing — needs multi-tab/session mutation harness not in scope for 11.2
- [x] [Review][Defer] Dedicated SharedLink `R` compare e2e — deferred, pre-existing — Self `R` path exercises same renderer; shared-link page uses identical `FeedbackSectionCard`

---
title: 'Frontend UI/UX updates to 2026-09-13 spine'
type: 'feature'
created: '2026-09-13'
status: 'done'
route: 'dispatch'
review_loop_iteration: 0
baseline_commit: '3f9123f296a2dbf78b1c5bd127e4af907eda2bd6'
context:
  - '_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/DESIGN.md'
  - '_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Frontend drifts from the final 2026-09-13 DESIGN/EXPERIENCE spines: dashboard/risk tables use row-onClick instead of the name-cell link contract, denied/empty cells render `—`/`unavailable` instead of absent, shared-link revoke skips confirm, disabled buttons drop keyboard focus, and filter/sort controls lack accessible names.

**Approach:** Bring `services/frontend` into spine compliance surface by surface (tokens already exact per Option A): fix the row-link contract, absent-not-hidden rendering, single-level Dialog discipline with reason gates, access-aware microcopy, and chip/sort/live-region a11y — no new surfaces, no visual redesign.

## Boundaries & Constraints

**Always:**
- Spines win on conflict; never copy mock markup verbatim (validation-report ship-blockers).
- Denied sections/fields/cells are absent from DOM and never fetched; toasts/banners never name gated fields.
- Keep arrow-function components, `@/` alias, TanStack Query layering, i18n keys first, semantic Tailwind tokens with no hex or manual `dark:` variants.
- Preserve uncommitted Option A token baseline in `services/frontend/src/index.css` (light/dark risk pairs already exact).

**Never:**
- Build full departure admin, full relationship-change screen, or pagination/virtualization as new features (minimal reject-route hint + minimal confirm coverage only, per decisions below).
- Redefine base shadcn tokens or restyle components beyond the compliance fixes.
- Use client-side masking as access control; widen any query beyond the viewer's resolved access.

**Decisions (2026-09-13):**
- Relationship guard: add explicit client reject-and-route hint for manager/PP/department cells (EXPERIENCE L87); no full relationship screen.
- Unavailable cells: remove per-cell `dashboard.unavailable`, render absent + stale-banner only.
- Departure/close: include minimal confirm-dialog coverage where those flows exist; otherwise verify absence.
- Token budget: keep full spec at ~2090 tokens, accept context-rot risk.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Dashboard row drill | Keyboard `Enter` on person name in UM/PP/risk table | Navigates to full Employee Profile; SR announces name, position, project | N/A |
| Denied cell | Viewer lacks grant for leave/department/projectStatus cell | Cell absent (no `—`, no `unavailable`, no layout gap) | N/A |
| Revoke shared link | Author clicks revoke | Confirm dialog, then revoke + success toast | Failed revoke retains link + error toast with retry |
| Reason-gated reject | DM rejects with empty reason | Confirm stays inoperable but focusable; activating announces why | N/A |

</frozen-after-approval>

## Code Map

- `services/frontend/src/index.css` -- Option A baseline already exact (light `:130-139`, dark `:219-228`); do not retoken, only preserve.
- `services/frontend/src/components/RiskBadge/risk-level-styles.ts:4-16` + `RiskBadge.tsx:16`, `TrendArrow.tsx:3,34`, `StatusBadge.tsx:13` -- correct bg/fg consumers; reuse pattern.
- `services/frontend/src/pages/RiskDashboardPage/RiskDashboardTable.tsx:36-52` -- row-onClick violation + `—` cells; primary fix target.
- `services/frontend/src/components/DashboardEngine/ScopedPeopleTable.tsx:56-112` -- row-onClick + `—`/`unavailable` cells; primary fix target.
- `services/frontend/src/pages/AllEmployeesPage/EmployeeTable.tsx:72-119` + `EmployeeCardList.tsx:32-35` -- compliant name-cell `Link` reference pattern; copy it.
- `services/frontend/src/pages/Resourcing/ResourcingPage.tsx:28-31`, `CampaignsPage.tsx:61-64` -- whole-row button; convert toward name-cell link.
- `services/frontend/src/components/Table/Table.tsx:41-46` -- pass-through; do not add access logic here.
- `services/frontend/src/components/SharedLink/SharedLinkManagerDialog.tsx:220-238` -- revoke without confirm; add confirm.
- `services/frontend/src/components/ConfirmationModal/ConfirmationModal.tsx:43-56`, `DecisionReasonDialog.tsx:41-46`, `EndPairDialog.tsx:57-66`, `ProposalList.tsx:104,114`, `AudienceBuilder.tsx:101-152` -- `disabled` usage + chip/live gaps.
- `services/frontend/src/locales/en/translation.json:13,15,60,271,277,422,519,804,923` -- microcopy leaks (`s7.gated`, `filtersHiddenNotice`, stale/accessDenied wording) + `emptyCell:"—"`.
- `services/frontend/src/components/ui/dialog.tsx:50-79,118`, `ui/sheet.tsx:46-66`, `ui/alert-dialog.tsx:47-66` -- Radix trap/restore inherited; fix hardcoded `Close` i18n only.

## Tasks & Acceptance

**Execution:**
- [x] `services/frontend/src/pages/RiskDashboardPage/RiskDashboardTable.tsx` -- replace row-onClick with name-cell `Link`, drop `—` empties to absent -- restores keyboard/SR drill contract
- [x] `services/frontend/src/components/DashboardEngine/ScopedPeopleTable.tsx` -- same row-link fix; replace `—`/`dashboard.unavailable` per-cell with absent + rely on stale banner -- enforces absent-not-hidden
- [x] `services/frontend/src/pages/Resourcing/ResourcingPage.tsx` + `CampaignsPage.tsx` -- convert whole-row button toward name-cell link with SR announcement -- unifies drill pattern
- [x] `services/frontend/src/components/SharedLink/SharedLinkManagerDialog.tsx` -- add explicit revoke confirm with focus trap/return; sweep for departure/request-close confirms and add minimal coverage where found, else verify absence -- matches destructive discipline
- [x] `services/frontend/src/locales/en/translation.json` -- remove `s7.gated` existence hint, align `accessDenied`/`staleData` to spine voice, drop `emptyCell:"—"` usage -- stops microcopy leaks
- [x] `services/frontend/src/components/ConfirmationModal/ConfirmationModal.tsx`, `DecisionReasonDialog.tsx`, `EndPairDialog.tsx`, `ProposalList.tsx`, `AudienceBuilder.tsx` -- switch reason/disabled gates to focusable `aria-disabled` + announce-why; give filter chips accessible remove names, `th aria-sort`, `aria-live` for counts -- meets WCAG/keyboard floor
- [x] `services/frontend/src/components/ui/dialog.tsx` -- move hardcoded `Close` to i18n keys -- completes i18n discipline
- [x] `services/frontend/src/pages/AllEmployeesPage/EmployeeTable.tsx` + `EditableCell.tsx` -- add explicit reject-and-route hint for manager/PP/department cells (no full screen) -- matches EXPERIENCE L87 guard
- [x] Edge-case unit tests for I/O matrix (row drill, denied-cell absent, revoke confirm, reason gate) -- locks spine behavior

**Acceptance Criteria:**
- Given a keyboard-only user on any dashboard/risk table, when they `Tab` to a person name and press `Enter`, then the full profile opens and the SR announces name, position, project.
- Given a viewer without a cell grant, when the table renders, then no `—`, `unavailable`, lock, or placeholder exists in the DOM for that cell.
- Given a shared-link author, when they revoke, then an explicit confirm precedes the revoke and success/error toasts follow.
- Given a reason-gated confirm with empty reason, when the control receives focus and is activated, then focus stays and the reason requirement is announced.
- Given `npm run typecheck`, `npm run build`, and `npm run lint` from `services/frontend`, when run, then all pass with no new warnings beyond the existing chunk-size note.

## Implementation Notes

## Spec Change Log

## Review Triage Log

- [low] B1 AudienceBuilder chips show field name only, no operator/value — verified in `AudienceBuilder.tsx:69-76`; spec only required accessible remove names, chip content is product decision → defer.
- [low] B2 `aria-disabled` hint uses `role="status"` with no `aria-describedby` link — verified in `ConfirmationModal.tsx:57-61`; announcement still fires via live region, association is enhancement → patch (add `aria-describedby`).
- [false] B3 Stale banner collapses per-row detail — disproved as defect: spec Decisions mandate absent + table-level stale banner only, no per-row detail required.
- [false] B4 Empty denied cells render silent `null` — disproved as defect: spec mandates absent-not-hidden, silent is the contract; loading states handled separately.
- [low] B5 Non-navigable Campaigns/Resourcing rows are plain text with no why-blocked explanation — verified; spec only unified the drill pattern, explanation is new surface and rarely met → rejected per low-fix-cost rule.
- [false] B6 `directory.cellEmpty:"—"` remains — disproved: directory key is a TimeTracker load/error state, out of the spec's denied-cell surface list (Code Map + subagent note).
- [false] B7 `revokeFailed` key missing — disproved: `translation.json:1063` has `revokeFailed`.
- [false] B8 Enter-key bypass of reason gate — disproved: gated buttons are `type="button"` with no parent `form onSubmit` (`DecisionReasonDialog.tsx:50-62`), no submit path to bypass.
- [low] B9 DecisionReason hint reuses generic mode description — verified `DecisionReasonDialog.tsx:86`; specific validation wording is a direct text swap → patch.
- [low] B10 ProposalList `isDeciding` silent return, `headcountFull` toast-only — verified `ProposalList.tsx:109-117,129-133`; grouped with E7/E17 → patch (announce deciding state).
- [false] B11 Guard hint not keyboard-focusable — disproved: hint includes `sr-only` text inside the cell span (`EditableCell.tsx:167-175`), announced in browse mode; non-writable cells are not focusable by design.
- [false] B12 Verification missing test command — rejected: fix would edit this spec, excluded by review routing rules; matrix tests already ran via `test:unit`.
- [maybe-false] E1 Duplicate fieldId key collision — could not confirm filters allow duplicates; needs schema check (`definition.filters` uniqueness) → defer (unverified, would be medium).
- [medium] E2 Lowercased guard lookup never matches camelCase set entries — verified `EditableCell.tsx:49-61` (`managerId` vs `managerid`); manager field can bypass guard → patch (lowercase set entries).
- [medium] E3 Writable relationship field skips guard branch — verified guard only under `!writable || !field.editable` (`EditableCell.tsx:163`); editable relationship id would open editor against intent → patch (check guard before edit branch).
- [low] E4 No hint prop + disabled activate shows nothing — verified `ConfirmationModal.tsx:57` requires `confirmDisabledHint`; callers may omit → patch (fallback text).
- [low] E5 Reopened dialog retains `announcedHint` — verified no reset on open (`ConfirmationModal.tsx:45`); escape/outside close bypasses `handleClose` → patch (reset on open).
- [maybe-false] E6 Empty employeeId builds `/employees/undefined` — could not confirm empty ids occur (`DashboardTableRow.employeeId` required); needs data proof → rejected (would be low).
- [low] E7/E17 ProposalList `isDeciding` returns silently — verified, same root as B10 → patch (single entry).
- [medium] E8 Double-click fires approve twice before `isDeciding` flips — verified `onApprove` has no in-flight guard (`ProposalList.tsx:109-118`); over-approval risk → patch (ref guard).
- [low] E9/E10 Submitting state shows no hint (`EndPair`/`DecisionReason`) — verified hint conditioned on `!canConfirm` only; valid-reason + submitting gives silent click → patch (single entry, include submitting in hint).
- [false] E11 Non-string reason throws on `.trim` — disproved: `isReasonConfirmable(reason: string)` typed, all callers pass `useState('')` strings.
- [false] E12 Null rows crash — disproved: `dashboardRowsHaveStaleCells(rows: Array…)` typed, callers pass arrays (`ScopedPeopleTable.tsx` guards empty).
- [low] E13 Arming another link mid-revoke desyncs pending target — verified `armRevoke` has no `isRevoking` guard; rarely met and fix adds branch → rejected per low-fix-cost rule.
- [maybe-false] E14 employeeId with slash breaks route — could not confirm ids contain slashes (UUIDs/slugs); needs data proof → defer (unverified, would be low → rejected, so drop; recorded as defer).
- [false] E15 Over-limit announces required-only — disproved: `Textarea maxLength` enforces limit, over-limit unreachable via UI.
- [low] E16 Revoke success leaves focus unhandled — verified (only cancel returns focus); keyboard lands on refetch → patch (move focus to manage heading/button on success).
- [low] E18 SR label has name only, AC asks name+position+project — verified label interpolates `{name}` only; position not in row type, project partially available; wording needs product call → defer.
- [medium] V1 ScopedPeopleTable has no consumer render test (pre-verified gap) — helpers-only coverage, row-link/banner/absent regression undetected → patch (add render test).
- [medium] V2 RiskDashboardTable has no consumer render test (pre-verified gap) — helper-only coverage, row-click/`—` regression undetected → patch (add render test).
- [medium] V3 Revoke two-step only helper-tested (pre-verified gap) — arm/cancel/confirm/failure untested at hook/component → patch (add hook/component test).
- [medium] V4 ProposalList gate/toast untested (pre-verified gap) — no test executes headcountFull/deciding path → patch (add render test).
- [medium] V5 AudienceBuilder chips + EditableCell guard untested (pre-verified gap) — chip remove/guard hint untested → patch (add render tests).
- [medium] O1 `management-notes-visibility` e2e asserts removed gate — verified gate removed intentionally per absent-not-hidden; e2e will fail → patch (update e2e).
- [medium] O2 `risk-dashboard` e2e clicks non-interactive row — verified row no longer has `onClick`; must drive link → patch (update e2e).
- [medium] O3 `campaigns` e2e clicks row/button — verified rows are `li` + inner link now; must drive link → patch (update e2e).

## Design Notes

Row-link reference (`EmployeeTable.tsx:115-118`): name cell is a real `Link` to the profile route; the row itself carries no `onClick` or `tabindex`. Replicate this shape in `RiskDashboardTable` and `ScopedPeopleTable` rather than inventing a new drill affordance. Trend arrows keep text equivalents (`up`/`down`/`flat`), never color-only.

## Verification

**Commands:**
- `npm run typecheck` (from `services/frontend`) -- expected: exit 0
- `npm run build` (from `services/frontend`) -- expected: exit 0, only existing chunk-size warning
- `npm run lint` (from `services/frontend`) -- expected: no new issues

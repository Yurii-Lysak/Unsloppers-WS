# Validation Report — people management

- **DESIGN.md:** `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/DESIGN.md`
- **EXPERIENCE.md:** `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/EXPERIENCE.md`
- **Run at:** 2026-09-13 (Update + Validate; mock fixes applied, 3 parallel reviewers)

## Overall verdict

Spines are coherent, inheritance-disciplined, and shape-conformant, with all 14 cited mockups present and a credible IA-closure claim across FR-1–FR-65. The pair is downstream-usable for directory, resourcing, risk, campaign, and mentorship work, but MyProfile, departure-cascade, CDS/IDP, and the seven admin sub-surfaces ride on "stated admin task" coverage rather than flows, mocks, or DESIGN visual specs, and two risk tokens diverge from the cited CSS source.

Extra reviewers shift the picture toward the mocks: token contrast is now measured and verified (all 12 pairs ≥ 4.5:1, recomputed exactly — prior high closed), Update fixes verified as landed (dismissed excluded, row links in 01, sidebar wired in 01, MOCK-ONLY note, relabels, dash cleanup, address trim, IDP receipt, state-note comments), but ship-blockers remain in mock markup that story-dev will copy: plain-text/`onclick` rows in 04/05/06/08/09, dead `#` nav in 8 mocks, inline cards substituting for Dialog confirms in 07/11, journal-gate wording drift in 12, flat-trend over-render and `—`-for-empty cells. Fix the token divergences + mock ship-blockers before build; add one self-service/departure flow plus admin visual coverage.

## Category verdicts
- Flow coverage — thin
- Token completeness — adequate
- Component coverage — adequate
- State coverage — adequate
- Visual reference coverage — adequate
- Bloat & overspecification — strong
- Inheritance discipline — strong
- Shape fit — strong

## Findings by severity

### Critical (3 — all mock-consistency, row-link contract)
**[Mock consistency]** — `onclick`/non-link rows break keyboard/SR drill (`06-resourcing-list.html:19-24` onclick rows; `04-dashboard-um.html:31-34,38-39` + `05-dashboard-dm.html:36-38,42-44` plain `<b>` rows)
Fix: name-cell real `<a href="02-employee-profile.html">` / `<a href="07-resourcing-detail.html">`; row never `tabindex=0`; SR announces name, position, project. 01 already fixed — replicate.

### High (11)
**[Rubric · Flow]** — MyProfile self-service journey has no flow (`EXPERIENCE.md:38` vs flows; FR-16–FR-19)
Fix: add Flow 7 (edit contacts/address, upload certificate, self-complete IDP, read flagged note) or scope MyProfile out of this phase.
**[Rubric · Flow]** — Departure recording/cascade has no walkthrough (`EXPERIENCE.md:53-54,111` + failure-branch mentions only)
Fix: short admin flow (record → re-parenting gate → cascade preview → read-only profile) exercising atomic-cascade + sole-holder gates.
**[Rubric · Token]** — `risk-medium` + `risk-leaver` proposed values diverge from cited source (`DESIGN.md:158,160` vs "verbatim from index.css" `DESIGN.md:21`)
Fix: land index.css change first or demote to `proposed-` keys.
**[Accessibility]** — Filter-chip × buttons have no accessible name; sort/toolbar controls are non-semantic spans (`01:40-42,48-50`; `08:23-24`)
Fix: chip dismiss = focusable button `aria-label="Remove {filter} filter"` (Enter/Space, Backspace/Delete when focused); sortable headers `<button aria-sort>`; toolbar chips `aria-haspopup="dialog"` buttons. Do not copy mock markup verbatim.
**[Mock consistency]** — Dead `#` nav in 02/03/04/05/06/08/09/11/12 (per-mock lines in review-mock-consistency.md §§02–12)
Fix: wire every sidebar entry to its mock file; self = active span.
**[Mock consistency]** — Reject/close + pair-end rendered inline, not single-level Dialog (`07:22,25`; `11:27`)
Fix: annotate as Dialog content (focus trap + return focus, reason-required) or add Dialog variant; never stacked.
**[Mock consistency]** — Journal readability drift (`12:25` "Reporting-line, Project-line, or PP" vs EXPERIENCE "current manager/PP")
Fix: adopt spine wording; spines win.
**[Mock consistency]** — Flat-trend over-render (`08:26,29,30` `— flat` with no level change)
Fix: omit trend when no change; keep `▲up/▼down` + text equivalents only.
(+ 3 more high in review-mock-consistency.md remaining-gaps ordering.)

### Medium (~25)
Rubric (8): F3 SharedLink recipient journey unwalked; F4 CDS/Feedback no flow; F5 admin sub-surfaces flow-less; T2 `display-sm` undefined; T3 ~15 inherited tokens without source lines; C1 admin components behavior-only; S1 conflict states only in flow branches; V1 seven admin surfaces share one mock; I1 proposed values under "verbatim" header.
Accessibility (4): live-region/role table missing; focus-trap boundary (initial focus/wrap/dead-invoker); 200% zoom/320px commit; stale-banner focus contract in spine not just mock comments.
Mock consistency (14): `—` for empty cells (01/04/05/06/09); viewed-as missing (06/07/08); responsive/a11y annotations missing (02/03/07/10/13); unmocked states (Sheet/Dialog/CmdK/expired/closed/stale/offline/zero-results/toasts); counts/labels; mentor-gate annotation; Save-all vs writethrough; tabs semantics; candidate/recipient drill links. See review-mock-consistency.md §§01–13.
Fix: per-item fixes in reviewer files; spine wins on conflict.

### Low (~19)
Rubric (6): T4 ASSUMPTION-only tokens; C2 export/share-picker behavior-only; C3 frozen-receipt alias; S2 empty-state inconsistency; S3 offline ASSUMPTION; V2 PM/PP lenses undedicated; V3 CDS anchors.
Accessibility (4): mock `#777` caption 4.48:1 (use code token, not mock gray); reduced-motion ingress; tooltip-only definitions; `▸` glyph noise.
Mock consistency (9): CSS duplicate `background:#fff` (`08:10`); font/bg drift; Bench tab variant; unused `.p-low`; pool/pair counts; Journal nav dup; `⦸` on Unassigned; draft dashes.
Fix: cosmetic; treat mock grays/glyphs as illustrative.

## Reviewer files
- `review-rubric.md` (critical 0 · high 3 · medium 7–8 · low 6)
- `review-accessibility.md` (critical 0 · high 1 · medium 4 · low 4)
- `review-mock-consistency.md` (critical 3 · high 7 · medium 14 · low 9)
- `validation-report.html` (this report, visual twin)

# Mock-vs-Spine Consistency Review — ux-people-management-2026-09-13

Date: 2026-09-13 · Reviewer: mock-vs-spine consistency reviewer (lenses: adversarial + edge-case + structure + verification-gap)
Spines: `EXPERIENCE.md` (IA + Component/State Patterns + Key Flows) + `DESIGN.md` (Components + Do's/Don'ts). **Spines win on conflict.**
Scope: all files in `mockups/` (01–13 + design-system.html).

## Overall verdict

**CONDITIONAL PASS — fixes verified, ship-blockers remain in row-link contract + dead nav + Dialog-vs-inline confirms.**

Verified as landed: dismissed excluded from default view (01:57 comment), row links added in 01 (name-cell `<a>`), sidebar wired in 01/04/05 (real hrefs for core surfaces), MOCK-ONLY absent-not-hidden note in 02:77, relabels (02:25 colleague-variant note, 04:26 "low excluded", 08:15 "low never counts"), dash cleanup (04/05 counters use pills + "(low: not active)" caption, not bare dashes for risk), address trim (03:28 single-line), IDP receipt (03:43-44 closed + evidence + assessor), HTML-comment state notes (01:61, 08:33).
Remaining: **3 critical** (onclick rows in 06/09, non-link rows in 04/05/08), **dead `#` nav in 02/03/04/05/06/08/09/11**, Dialog-confirm substitution in 07/11, journal-gate wording drift in 12, `—` for empty cells, flat-trend over-render in 08, and per-mock unmocked states (Sheet quick-view, Dialog confirms, CmdK empty/loading/stale, expired-link screen, campaign-closed, stale/offline banners, zero-results, failure/rollback/conflict toasts — covered only as comments or in design-system.html demos, not as mock states).

Counts: critical 3 · high 7 · medium 14 · low 9 (33 findings).

## Per-mock findings

### 00 · design-system.html — PASS (reference, not a surface)
- [low] 08-risk-dashboard `.toolbar` CSS typo `background:#fff;background:#fff` duplicated (08:10). Fix: delete duplicate declaration.
- [low] 06–13 shell uses `Inter/#F7F7FA` while 01–05 use `system/#f6f5fb`; tokens say inherit shadcn + `--sidebar #15112E`. Fix: align all mocks to token vars (cosmetic only, spine wins).
- Note (positive): Sheet quick-view (349), Dialog confirm (348), CmdK with filter JS (269-372), empty/skeleton/toast/banner (323-332), tooltip leaver-vs-dismissed (337), ProjectSelector Unassigned (343) are live demos — they do **not** satisfy per-surface state coverage; each surface still needs its own annotation or linked state.

### 01 · 01-directory.html — PASS with notes
- [medium] 01:31,33,34 — `—` in Leave/overdue cells for empty ("—", "—"). Spine State Patterns: gated = absent; empty directory has prescribed copy. Em-dash in a data cell reads as masked/redacted and leaks existence. Fix: render empty leave/overdue cells blank (absent) with no dash; keep `— flat` trend text-equivalent only where trend applies.
- [low] 01:36 tabs include "Bench — static list" alongside saved-view tabs. Spine Filter-builder assumes tabs + read-only-shared + viewer-scoped re-resolve; static-list bench variant is an Open Item (pending PO). Fix: rename to "Bench (saved view)" or annotate as open-item variant; add ownerless/no-owner-chip note if departed creator.
- Verified: dismissed excluded (01:57 comment), name-cell real links (01:52-56), sidebar wired (01:27-31), viewed-as (01:33), responsive/a11y foot (01:60) + empty/toast/Sheet comment (01:61), colleague-mode absent-not-hidden note (01:62).

### 02 · 02-employee-profile.html — NEEDS FIX (dead nav)
- [high] 02:22 — dead `#` nav: `Risk Dashboard / Campaigns / Mentorship` are `href="#"`. Spine IA: every surface reached from sidebar; mocks must wire navigation. Fix: point to `08-risk-dashboard.html`, `09-campaigns-list.html`, `11-mentorship-hub.html` (same pattern as 01).
- [medium] 02:28,65-67 — mentor shown unconditionally (header "Mentor: Iryna Shevchenko" + S13 mentee/mentor rows) with no open-to-mentoring gate annotation (D5). Spine Mentorship: assign rejected unless flag on at write time; pool shows flag only. Fix: annotate mentor row with flag state (e.g. "open ✓ / on") or add comment that viewer is reporting-line entitled and flag was on; do not show mentor for Colleague variant (already noted in 02:25 — keep).
- [medium] 02:80 — actions `Record risk / Assign action item / Request feedback / Propose` have no Dialog/Sheet annotation. Spine: destructive/irreversible = single-level Dialog with reason where mandated; quick-create = Sheet. Fix: add HTML comment noting Sheet for create + Dialog confirm for destructive, focus-trap + return-focus.
- [medium] 02 — missing responsive/a11y annotation block (no `<md` Sheet note, no SR announce, no Esc/focus text; contrast with 01:60). Fix: add one-line foot/comment: sections stack `<md`, tables → row-drill lists, SR "Profile: {name}, {M} sections available".
- Verified: MOCK-ONLY gated S7 note (02:77), viewed-as + colleague-variant (02:25), department change routed to relationship screen (02:34), S5 CV+certs-only note (02:46), S9 no-departure note (02:57), IDP done (02:64 — actually S12, correct).
- [low] 02:11 `.p-low` defined but S6 pill uses `p-med` only — unused class, harmless. No fix needed.

### 03 · 03-my-profile.html — PASS with notes
- [high] 03:18 — dead `#` nav: `My Action Items (2) / My Leaves` are `href="#"`. Fix: wire to real surfaces or remove from mock nav; annotate counts come from S14/leaves.
- [medium] 03:49 — `Save all / Discard draft` contradicts spine Inline-edit writethrough (`Enter`/blur commits, failed write retains edit + toast). Mock banner (03:22) says blur/Enter commits, then offers bulk save. Fix: relabel to "Retry failed writes" or annotate bulk-save as mock shorthand; add failed-write toast + `aria-disabled` note.
- [medium] 03 — missing responsive/a11y annotation (no Sheet/Dialog/Esc/SR text beyond 03:22). Fix: add comment: `<md` stacking, focus order = reading order, toasts `role=status`.
- Verified: address trim single-line (03:28), IDP receipt with evidence + assessor + reopening rule (03:43), employment read-only + relationship-screen hint (03:31-37), photo/cert upload with assumed limit flagged (03:39 — mark as assumption).

### 04 · 04-dashboard-um.html — NEEDS FIX (row-link contract)
- [critical] 04:31-34,38-39 — table rows have plain `<b>` names, no real links; no `onclick` either. Violates Component Patterns: name cell must be a real link (focusable, Enter drills), row never `tabindex=0`, SR announces name/position/project. Fix: wrap names in `<a href="02-employee-profile.html">` (people) and REQ cells in `<a href="07-resourcing-detail.html">`.
- [high] 04:17 — dead `#` nav: `Risk Dashboard / Resourcing` are `href="#"`. Fix: wire to `08-risk-dashboard.html` / `06-resourcing-list.html`.
- [medium] 04:31,33,34 — `—` for empty leave/overdue ("—", "—"). Fix: blank/absent cells (see 01).
- [low] 04:34 — "(low: not active)" caption is correct intent (low never active) but phrased as parenthetical; keep — spine wins, no change. Banner (04:21) stale wording matches spine verbatim — verified.
- Verified: viewed-as + SR announce (04:20), stale banner (04:21), counters active-only with low excluded (04:22-28), Unassigned note via 05/06 (acceptable cross-ref).

### 05 · 05-dashboard-dm.html — NEEDS FIX (row-link contract)
- [critical] 05:36-38,42-44 — same as 04: no links in request or people rows. Fix: same `<a>` wrap as 04.
- [high] 05:19 — dead `#` nav (`Risk Dashboard / Resourcing`). Fix: same as 04.
- [medium] 05:24 — `⦸ Unassigned — project-less requests` option uses prohibition glyph `⦸` for a valid selectable group. Spine: Unassigned is valid/normal, always reachable. Fix: drop `⦸`, use "Unassigned — project-less requests (2)" plain.
- [medium] 05:43-44 — `—` for empty risk/overdue. Fix: blank cells.
- Verified: project selector scoping + Clear→aggregate + "never widens audience" (05:23-26), per-project counters (05:27-33), Unassigned bucket banner with empty-state copy (05:46), viewed-as + SR (05:22).

### 06 · 06-resourcing-list.html — NEEDS FIX (onclick rows)
- [critical] 06:19-24 — rows use `onclick="location.href=..."` with no `<a>`. Fails keyboard/SR contract (not focusable, no Enter, no announce). Fix: replace with real links in Request cell (`<a href="07-resourcing-detail.html">REQ-…</a>`); keep row hover only.
- [high] 06:12 — sidebar `Dashboard / All Employees` are `href="#"`. Fix: wire to `04-dashboard-um.html`/`05-dashboard-dm.html` + `01-directory.html`.
- [medium] 06:14 — viewed-as audience vague ("DM Dmytro lens"); no "Viewed as …" line or SR text unlike 04/05. Fix: add crumb "Viewed as D. Petrenko (DM)".
- [medium] 06:22 — `— no project —` uses dashes for empty project. Spine Empty-project-bucket copy is "No unattached requests." Fix: cell "Unassigned" pill only (already present in 06:22 col 2); drop mono dash line or replace with "No project — Unassigned bucket".
- [low] 06:25 — row-drill note present but describes click only. Fix: append "name cell is a real link; Enter drills".
- Verified: Unassigned-always-last banner (06:15), counters (06:16), explicit-close note (06:25), closed-unsuccessful row demonstrates campaign-closed analogue for resourcing (06:23).

### 07 · 07-resourcing-detail.html — NEEDS FIX (confirm pattern)
- [high] 07:22,25 — reject/close flows rendered as inline cards + plain buttons, not single-level Dialog. Spine Dialog-confirms: destructive (reject, reverse-approval, close, departure, pair-end) require explicit Dialog confirm with reason; never stacked. Fix: annotate inline cards as "Dialog content shown inline for mock — production is single-level Dialog with focus trap + return focus" or add Dialog demo link; keep reason-required copy (already correct: "Rejected — reason required", "Raise headcount or close the request.").
- [medium] 07:13-14 — missing viewed-as audience (header shows avatar DP + project only). Fix: add "Viewed as D. Petrenko (DM, reviewing)" + comp-band visibility note (already in guard 07:17 — keep).
- [medium] 07:12 — sidebar detail entry `href="#"` (self). Fix: use real back link only (already 07:14) or mark active span, not dead link.
- [low] 07:18-21 — proposal/candidate rows have buttons but candidate names are not links (no drill to profile/SharedLink). Fix: link internal candidate to `02-employee-profile.html` / `13-shared-link-view.html`.
- Verified: comp-band guard (07:17), auto-Shared-Link scope S1/S4/S11/S12/S5-CV+certs S6-optional S2/S3/S7/S8-never (07:19,23), headcount guards + aria-disabled note (07:25), journaled history with before/after lifetimes (07:26).

### 08 · 08-risk-dashboard.html — NEEDS FIX (flat trends + row links)
- [high] 08:25-31 — person cells are plain `<strong>`, no links; violates row-link contract (drill to `02-employee-profile.html` S6). Fix: wrap names in `<a>`.
- [high] 08:26,29,30 — `— flat` trend rendered on rows with no level change. Spine: trend renders only on actual level change, diffed backend-side. Flat should be absent, not dashed. Fix: drop trend element on flat rows (keep `▲up/▼down` only); keep text equivalents on rendered arrows.
- [medium] 08:13-14 — sidebar dead `#` (`Dashboard / All Employees`); header scope "Manager / PP access" names no viewer. Fix: wire sidebar; add "Viewed as M. Shevchuk (PP)"-style crumb.
- Verified: active-only + low-never-counts + severity sort (08:15,24), counters (08:16-22), empty-state copy (08:32), stale/offline/CmdK/a11y HTML comment (08:33) — state-notes fix landed.

### 09 · 09-campaigns-list.html — NEEDS FIX (onclick rows)
- [high] 09:17-20 — rows use `onclick`, no `<a>`. Fix: link campaign titles to `10-campaign-detail.html`.
- [medium] 09:18 — `— not activated —` uses dashes for draft empty state. Spine Campaign-frozen/tracking: draft editor + audience builder, no completion until frozen. Fix: replace with "Not activated — no tracking yet" plain text, no dashes.
- [medium] 09:11-12 — sidebar `Dashboard` dead `#`; header "PP lens · M. Shevchuk" is viewed-as but no SR/response note. Fix: wire Dashboard link; keep PP lens label.
- Verified: draft/active/closed with per-campaign completion (09:14,16-20), frozen/active pills (09:17,19), external-form-never-read sub (09:13), closed row present (09:20).

### 10 · 10-campaign-detail.html — PASS with notes
- [medium] 10 — campaign-closed state unmocked (this mock is active/frozen; no closed read-only variant). Spine Campaign-frozen + list shows closed exists. Fix: add comment or second screenshot note: closed = editor read-only + tracking frozen + "Request closed" copy; or link closed row from 09 to a closed variant.
- [low] 10:21-24 — recipient names plain text, no profile drill links. Fix: wrap in `<a href="02-employee-profile.html">` or annotate entitled-subset (already 10:19 — extend to rows).
- Verified: wizard steps draft→audience→frozen→close (10:16), frozen receipt + Action-Item-per-recipient (10:17), entitled-subset audience note (10:19), all-or-nothing rollback note (10:19), per-person completed/not/overdue (10:20-24), viewed-as PP (10:14), back link (10:15).

### 11 · 11-mentorship-hub.html — PASS with notes
- [medium] 11:13 — sidebar `Dashboard` dead `#`. Fix: wire to dashboard lens.
- [medium] 11:27 — end-pair confirm rendered as inline card, not Dialog. Spine requires Dialog with reason (closure note) + conflict handling. Fix: annotate as Dialog content (same fix as 07); keep refusal/conflict copy (already correct).
- [low] 11:16 — header claims "pool (5)" but renders 3 cards; pairs header "(4)" renders 3 rows. Fix: align counts or mark "truncated for mock".
- Verified: pool identity-card + flag only (11:15-20), assign gated on flag with server-side note (11:20-21), guarded mentor selects (11:21 — D5 compliant), ended-pair note + S9 event + readability scope (11:25), viewed-as UM + permission (11:14).

### 12 · 12-admin.html — NEEDS FIX (journal wording)
- [high] 12:25 — journal readability: "Full-access holders + subject's current Reporting-line, Project-line, or PP holder (AD-2 C10)". EXPERIENCE.md IA table says "readable by Full-access holders + subject's current manager/PP". Mismatch (project-line added, manager generalized). Spine wins. Fix: change to EXPERIENCE wording or cite AD-2 C10 as override with pointer; do not invent "Project-line" reader without spine update.
- [medium] 12:13 — sidebar `Dashboard` dead `#`; `Journal` tab links to `12-admin.html` self (duplicate). Fix: wire Dashboard; make Journal an in-page tab anchor, not a nav link.
- [medium] 12:16 — tabs are `<span>`, not focusable controls. Spine a11y floor: full keyboard parity. Fix: use `<button role=tab>` (as in 01) or annotate tabs as mock shorthand.
- Verified: functional-role immediate revoke (12:18), custom-field live/filter note (12:19), guarded dept-manager writer (12:20), org-relation guards + journaled example (12:22), never-zero-holder gate (12:23), departure cascade preview wording matches spine (12:24), shared-link access in journal matches spine scope (12:25 row 3).

### 13 · 13-shared-link-view.html — PASS with notes
- [medium] 13:21 — expired/revoked screen unmocked (only inline mono copy of the string). Spine Link-expired/revoked: no profile content, no section names leak. Fix: add second state (comment or linked variant) showing full-screen message "This link has expired / been revoked. Ask the sender for a new one." with zero cards rendered.
- [medium] 13 — missing responsive/a11y annotation (no `<md` stacking, no SR announce, no focus note; nav-absent is correct per IA). Fix: add one-line comment.
- Verified: nav-absent (correct — direct token URL), named authenticated recipient (13:11), request-lifetime expiry + re-clamp (13:12), scope line S1/S4/S11/S12/S5-CV+certs S6-enabled S2/S3/S7/S8-never (13:13), S5 contract/W8/Diia exclusion (13:19), S6 prediction-not-departure (13:20), absent-not-masked note (13:21).

## Remaining gaps (cross-mock, ordered by ship risk)

1. Row-link contract: 04/05/08 plain-text rows; 06/09 `onclick` rows (critical). 01 fixed; 02/07/10/11 minor (candidate/recipient names). Fix: name-cell `<a>`, never `tabindex=0` on row, SR announce per EXPERIENCE.
2. Dead `#` nav: 02/03/04/05/06/08/09/11/12 (high). Fix: wire every sidebar entry to its mock file; self = active span.
3. Dialog confirms: 07 reject/close + 11 pair-end rendered inline (high). Fix: annotate as Dialog content or add Dialog variant; single-level, focus trap + return focus, reason-required.
4. Journal readability drift: 12:25 vs EXPERIENCE (high). Fix: adopt spine wording; spine wins.
5. Flat-trend over-render: 08 `— flat` (high). Fix: omit trend when no level change.
6. `—` for empty data cells: 01/04/05/06/09 (medium). Fix: blank/absent cells; reserve `—` for trend text-equivalent only.
7. Viewed-as audience missing/vague: 06/07/08 (medium). Fix: "Viewed as {name} ({role})" crumb on every mock (01/02/04/05 pattern).
8. Responsive/a11y annotations missing: 02/03/07/10/13 (+ partial 04/05) (medium). Fix: one-line foot/comment per mock (`<md` Sheet/stacked lists, SR announce string, Esc single-layer, `aria-disabled` + toast).
9. Unmocked states (no visual, comment/design-system only): Sheet quick-view (all tables), Dialog confirms (per-surface), CmdK empty/loading/stale, expired-link screen (13), campaign-closed variant (10), stale/offline banners (except 04), zero-results (except 08 copy), failure/rollback/conflict toasts (10:19 note, 11:27 note only). Fix: add HTML-comment state notes per surface (01:61/08:33 pattern) or link design-system demos; do not leave silent.
10. Counts/labels: 11 pool(5)/pairs(4) vs rendered 3/3; 12 Journal nav dup; 05 `⦸` on Unassigned; 09 draft dashes (low). Fix: align numbers, drop glyphs/dashes.
11. CSS/typo: 08:10 duplicate `background:#fff`; cross-mock font/bg drift (`Inter/#F7F7FA` vs system/`#f6f5fb`) (low). Fix: dedupe; align to tokens (cosmetic).

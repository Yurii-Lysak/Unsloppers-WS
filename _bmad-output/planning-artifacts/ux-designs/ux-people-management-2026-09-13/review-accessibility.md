# Accessibility Review — people management UX spines (WCAG 2.2 AA)

- **DESIGN.md:** `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/DESIGN.md`
- **EXPERIENCE.md:** `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/EXPERIENCE.md`
- **Mockups sampled:** `mockups/01-directory.html`, `mockups/02-employee-profile.html`, `mockups/07-resourcing-detail.html`, `mockups/08-risk-dashboard.html`, `mockups/10-campaign-detail.html`, `mockups/11-mentorship-hub.html`, `mockups/13-shared-link-view.html`
- **Standard:** WCAG 2.2 AA, responsive web only (no native client). Radix/shadcn primitives assumed; this lens reviews spine behavior + token risk + mock copy-paste risk, not library internals.
- **Run at:** 2026-09-13 (re-run; prior review's unmeasured-contrast high is now resolved — see F1)

## Overall verdict

**Pass with fixes — no release-blockers in the token layer.** The headline risk from the prior review is closed: DESIGN.md L162 now states per-pair measured ratios for all five risk levels + success in both modes, and every claimed figure independently recomputes exactly (light 5.33 / 4.82 / 4.72 / 5.26 / 6.25 / 6.22; dark 6.23 / 6.40 / 5.55 / 6.09 / 5.59 / 7.72 — all ≥ 4.5:1 including 12px pill text). The row-drill keyboard/SR contract and the `aria-disabled` (never `disabled`) rule are now explicit in EXPERIENCE.md and demonstrated in the mocks. Remaining work is one **high**: the mocks — which story-dev will copy — render filter chips with bare `×` buttons (no accessible name) and sort indicators / toolbar chips as non-interactive spans, which undercuts the otherwise good keyboard/SR contracts. Four mediums (live-region role table, focus-trap boundary detail, zoom/reflow commit, banner focus contract in spine rather than mock comments only) and four lows follow. Do not copy mock chip/sort markup verbatim into code without F2's fix.

## Findings

- **[high] Filter-chip × buttons have no accessible name; sort + toolbar controls are non-semantic spans (mock copy-paste risk).** `01-directory.html` L40–42: chips render `<button>×</button>` with no `aria-label` (SR hears "times"); L48–50: sort indicators are `<span class="sort">▲/⇅</span>` inside `<th>` with no button, no `aria-sort`; `08-risk-dashboard.html` L23 toolbar chips and L24 `<th>Employee ▼` are plain spans. EXPERIENCE.md Filter-builder row + L131 parity claim never name the chip-dismiss key or the sort control pattern, so devs copying the mock ship WCAG 4.1.2 / 2.1.1 failures on the two most-used surfaces. *Fix:* spine — chips dismiss via focusable button with `aria-label="Remove {filter} filter"` (keys: `Enter`/`Space`, plus `Backspace`/`Delete` when chip focused); sortable headers are `<button aria-sort>` or `th[aria-sort]` + inner button; toolbar chips that open popovers are `aria-haspopup="dialog"` buttons. Mocks — add the labels/buttons before they become code templates.

- **[medium] aria-live / role mapping table still missing for the full toast/banner set.** EXPERIENCE.md L115 (CmdK `aria-live`) + L130 ("palette, toasts, timeline annotations announce") + L99–111 state inventory cover the principle, and mock comments (`01` L61, `08` L33) cite `role=status` / `role=alert` — but no normative table assigns politeness per message: export-ready, stale/offline banner, campaign-frozen lock, link-expired, headcount-block ("Raise headcount…"), conflict-error ("Updated — refreshed view"), offline toast. *Fix:* add a table — success/info toasts + stale/offline banners → `role="status"` / `aria-live="polite"`; destructive/conflict/link-expired/headcount-block refusals → `role="alert"`; CmdK results → polite count announcement ("{N} results"); banners never move focus (see F5).

- **[medium] Focus-trap boundary contract incomplete (initial focus, wrap, dead-invoker fallback).** EXPERIENCE.md L117 (`Esc` single-layer) + L128 (trap + return-focus) and DESIGN.md Dialog/Sheet rows state the shape and the never-stack guardrail, but not: initial focus target (WAI-ARIA dialog pattern — heading vs close button), `Tab`/`Shift+Tab` wrap containment, or where focus lands when the invoker unmounts (row deleted, request closed, pair ended). Relies on Radix defaults silently. *Fix:* commit — initial focus = first focusable control (or dialog heading with `tabindex="-1"`); `Tab` wraps; on close return to invoker, fallback to surface heading when invoker is gone; keep the single-level guard.

- **[medium] 200% zoom / 400% reflow (WCAG 1.4.10) lacks a testable commit.** DESIGN.md L180 requires zoom to preserve table→profile navigation and EXPERIENCE.md L140–141 reflows tables to stacked row-drill lists with no h-scroll (mock `01` L60 repeats the claim), but neither states the sticky-header behavior at zoom, the 320px-width acceptance case, or that sort + drill + filter controls all stay reachable without two-dimensional scroll. *Fix:* commit — at 200%+ / 320px CSS width: tables reflow to stacked row-drill lists (no h-scroll), sticky header becomes a static section header, drill link + sort + filter + pagination remain reachable by keyboard; add 200% + 320px to the test matrix.

- **[medium] Non-dismissible stale banner has a focus contract only in mock comments, not in the spine.** DESIGN.md L227 (dismiss-forbidden while outage holds) + EXPERIENCE.md L105 define the banner correctly, and `08` L33 adds "`role=status` … never auto-focuses, never dismissible during outage" — but the spine never says: no auto-focus on appearance, no `tabindex`, dismiss control absent (not `disabled`), heading announces staleness on navigation. A dev implementing from the spine alone could auto-focus the banner or render a disabled dismiss button. *Fix:* move the `08` L33 contract into EXPERIENCE.md State Patterns: `role="status"`, never auto-focuses, no dismiss control while the outage holds, page heading carries staleness on navigation.

- **[low] Mock caption gray `#777` on white computes to 4.48:1 — just under 4.5.** Used for timestamps/helper text in `01` L25, `02` L13/L46/L57 (12px, so normal-text 4.5:1 applies). Spine caption token inherits shadcn defaults (unaffected); table headers (`#6B6B80` on `#FAFAFC`, verified 4.99:1) and all risk pills pass. *Fix:* in code use the inherited muted-foreground token (verified ≥ 4.5:1), not the mock's literal `#777`; treat mock grays as illustrative.

- **[low] Reduced-motion leaves Sheet/Dialog/banner ingress and trend-arrow behavior unstated.** EXPERIENCE.md L132 covers skeleton shimmer, toasts, transitions ("render instantly"). Open animation (scale/fade), banner slide-in, and any trend-arrow motion are not named. *Fix:* append — under `prefers-reduced-motion`, Sheet/Dialog open instantly (no scale/fade), banner appears without slide, trend arrows never animate.

- **[low] Risk/leaver definitions risk tooltip-only delivery on touch/SR paths.** DESIGN.md Tooltip row + L211 guardrail ("tap equivalent") and EXPERIENCE.md L119 (tap-to-reveal) cover row quick-actions, but risk-definition and leaver-vs-dismissed text should not live only behind hover. Mocks already do the right thing inline (`01` L55 "leaver · prediction", `13` L20 "Prediction, not departure"). *Fix:* spine — pill carries `aria-label`/`aria-describedby` with the definition; leaver-vs-dismissed disambiguation is body text on first use per surface, tooltip repeats it, never the sole carrier.

- **[low] Shared-link timeline (`13` L17) uses `▸` glyphs as list separators.** Screen readers may announce them as noise; the spine Timeline row already requires label+date pairing (which the mock satisfies in text). *Fix:* render the timeline as a real `<ul>` with `aria-hidden="true"` on the glyphs, or drop glyphs in code — mock text content is otherwise a good text equivalent.

## Positive notes (no action)

- Contrast claims now measured and independently verified — all 12 risk/success pairs ≥ 4.5:1 both modes; recomputation matched every stated figure to the decimal.
- Trend arrows always share the pill fg with text equivalents (`up`/`down`/`flat`) — never color-alone; counter deltas (`08` L17–21) pair numerals with labeled pills.
- Timeline dots always label+date paired with manual-correction badge; no departure event ever in the timeline (CAP-14 honored in `02` L57, `13` L17).
- Leaver plum breaks the heat ramp (never red, never conflated with `dismissed`); prediction disclaimer inline in `01` L55 and `13` L20.
- Absent-not-hidden honored end-to-end (colleague note `01` L62, gated-section handling `02` L77 mock-only annotation, expired-link silence `13` L21).
- CmdK results server-side pre-filtered per audience; risk/leave types never in colleague results (DESIGN.md CmdK guardrail).
- No presence dots anywhere; avatars always photo + adjacent name + initials fallback.
- Row-drill contract done right in spine (EXPERIENCE.md L83: name cell is a real link, row never `tabindex=0`, SR announcement shape) and in mocks (real `<a>` per row, `01` L52–56).
- `aria-disabled="true"` + reason-toast contract explicit (EXPERIENCE.md L120, `07` L25) — no `disabled`-attr focus loss.
- Non-dismissible stale banner + Unassigned-bucket hint use correct persistent-informative semantics; pre-filtered CmdK + tap-equivalence rule + focus-order-equals-reading-order + visible ring token all stated.

## Finding counts

- Critical: 0 · High: 1 · Medium: 4 · Low: 4

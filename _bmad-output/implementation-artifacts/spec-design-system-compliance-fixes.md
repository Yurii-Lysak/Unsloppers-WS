---
title: 'Design system compliance fixes'
type: 'bugfix'
created: '2026-09-13'
status: 'done'
route: 'oneshot'
review_loop_iteration: 0
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Mockups follow DESIGN.md Option A but `services/frontend/src/index.css` still ships stale risk tokens, and four mockups retain empty-cell dashes, span tabs, and a CSS duplicate.

**Approach:** Align index.css risk-medium/leaver/high tokens to DESIGN.md, blank empty data cells, fix admin tabs to buttons, dedupe CSS, and unify mock shell font/background.

</frozen-after-approval>

## Implementation Notes

- index.css: light risk-medium `#FCF1DC/#92620E`→`#FDE8D3/#9A4A0A`, leaver grey→plum `#EDE7F6/#5B4A8A`; dark low-fg→`#6BD193`, medium→`#42260F/#E89B4C`, high-fg→`#F4798A`, leaver→`#2B2145/#C0B3E8` per DESIGN.md frontmatter (spine wins over code+html). `npm run typecheck` passes.
- Mockups: blanked `—` empty cells in 04/05/11, dropped `—` flat trend in 01 pill + 08 counters, renamed Bench tab to `Bench (saved view)`, reworded 06 unassigned mono, deduped 08 `.chip` background, converted 12 tabs spans→buttons with reset CSS, unified 01–05 shells to `Inter/#F7F7FA`.
- Left uncommitted per workspace policy (no `git add` on own initiative; changes span workspace + frontend submodule).

## Review Triage Log

- 08 counter flat `— 0` vs omitted row trends: medium, patched (removed counter spans).
- index.css accent-warning/chart overlap claim: false — banner/chart tokens are separate roles, hex sharing intentional.
- All other 18 blind-hunter findings (pagination, chip a11y, section skips, save-model, banner timestamps, row-count continuity, pool counts, tab panels, date input): pre-existing, deferred as out of scope.

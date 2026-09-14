---
status: final
name: people management
description: Calm internal HR platform on shadcn/ui radix-nova — this DESIGN.md specifies the brand layer and risk-severity delta only.
created: 2026-09-13
updated: 2026-09-13
sources:
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - _bmad-output/specs/spec-people-management-platform/SPEC.md
  - _bmad-output/specs/spec-people-management-platform/access-model.md
  - _bmad-output/specs/spec-people-management-platform/interface-contracts.md
  - _bmad-output/specs/spec-people-management-platform/decisions.md
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - docs/project-requirements.md
colors:
  # Base system inherited VERBATIM from services/frontend/src/index.css (Tailwind v4
  # @theme inline + :root / .dark vars). Do not restate here: background, foreground,
  # card, popover, primary (#5B4FE0 light / #7D95F7 dark), secondary, muted, accent,
  # destructive (#DC3545 light / #E95B6C dark), border, input, ring, chart-1..5,
  # sidebar* tokens, shadow-*, radius. This frontmatter holds ONLY the delta.
  # Hex values below are verbatim from index.css; OKLCH equivalents live in Colors body.
  success: '#157A52'
  success-foreground: '#FFFFFF'
  success-dark: '#4CAF79'
  success-foreground-dark: '#1C1B29'
  risk-low: '#E8F7EF'
  risk-low-foreground: '#157A52'
  risk-low-dark: '#1A3D2A'
  risk-low-foreground-dark: '#6BD193'
  risk-attention: '#FCF1DC'
  risk-attention-foreground: '#92620E'
  risk-attention-dark: '#3D3420'
  risk-attention-foreground-dark: '#D4A845'
  risk-medium: '#FDE8D3'
  risk-medium-foreground: '#9A4A0A'
  risk-medium-dark: '#42260F'
  risk-medium-foreground-dark: '#E89B4C'
  risk-high: '#FBE7E9'
  risk-high-foreground: '#A61D2B'
  risk-high-dark: '#3D1F25'
  risk-high-foreground-dark: '#F4798A'
  risk-leaver: '#EDE7F6'
  risk-leaver-foreground: '#5B4A8A'
  risk-leaver-dark: '#2B2145'
  risk-leaver-foreground-dark: '#C0B3E8'
typography:
  # All roles inherit shadcn/ui + Tailwind defaults (sans, mono). No brand face.
  display:
    note: 'Inherit shadcn — no display face; largest heading is page title'
  page-title:
    fontSize: 24px
    fontWeight: '600'
    lineHeight: '1.25'
  section-title:
    fontSize: 18px
    fontWeight: '600'
    lineHeight: '1.35'
  body:
    note: 'Inherit shadcn body (sans 14px comfortable density)'
  label:
    fontSize: 14px
    fontWeight: '500'
    lineHeight: '1.4'
  caption:
    fontSize: 12px
    fontWeight: '400'
    lineHeight: '1.4'
  mono:
    note: 'Inherit shadcn mono — IDs, PeopleForce links, journal timestamps'
rounded:
  # Inherited verbatim from --radius: 0.625rem in services/frontend/src/index.css.
  sm: 4px
  md: 6px
  lg: 10px
  xl: 14px
  full: 9999px
spacing:
  # Tailwind v4 4-based scale inherited; named layout tokens verbatim from index.css.
  '1': 4px
  '2': 8px
  '3': 12px
  '4': 16px
  '5': 24px
  '6': 32px
  sidebar-width: 15rem
  sidebar-collapsed-width: 3.75rem
  header-height: 3rem
  table-row-height: 56px
  sheet-width: 480px
components:
  button-primary:
    background: '{colors.primary}'
    foreground: '{colors.primary-foreground}'
    note: 'Inherited system primary; success fill reserved for confirm-complete only'
  button-destructive:
    background: '{colors.destructive}'
    foreground: '{colors.destructive-foreground}'
  risk-pill-low:
    background: '{colors.risk-low}'
    foreground: '{colors.risk-low-foreground}'
    radius: '{rounded.full}'
  risk-pill-attention:
    background: '{colors.risk-attention}'
    foreground: '{colors.risk-attention-foreground}'
    radius: '{rounded.full}'
  risk-pill-medium:
    background: '{colors.risk-medium}'
    foreground: '{colors.risk-medium-foreground}'
    radius: '{rounded.full}'
  risk-pill-high:
    background: '{colors.risk-high}'
    foreground: '{colors.risk-high-foreground}'
    radius: '{rounded.full}'
  risk-pill-leaver:
    background: '{colors.risk-leaver}'
    foreground: '{colors.risk-leaver-foreground}'
    radius: '{rounded.full}'
  table-row:
    height: '{spacing.table-row-height}'
  sheet:
    width: '{spacing.sheet-width}'
    radius: '{rounded.lg}'
  dialog:
    radius: '{rounded.lg}'
  card:
    radius: '{rounded.lg}'
  input:
    radius: '{rounded.sm}'
---

## Brand & Style

People Management is a calm internal HR tool for an engineering organisation of 500+. The posture is **trust through access-correctness**: the interface must never imply data exists where it must not (per `access-model.md` Rule 1 — a `—` cell is absent from the DOM, never a masked placeholder). Visual restraint serves that posture — quiet surfaces, one blue accent that means *actionable*, semantic risk color that means *state*, never decoration.

Density is **comfortable pro-tool** (coaching-path decision, `.memlog.md` 2026-09-13): 56px table rows, 16/24px section rhythm, readable at 500-row scale without compaction anxiety. [ASSUMPTION] Comfortable density holds across directory, dashboards, and profile; no per-surface density switch in this spine.

Shell is a **sidebar layout**: dark Ink sidebar (`--sidebar: #15112E` light / `#0D0A1F` dark, verbatim from `services/frontend/src/index.css`) on desktop ≥1024px, collapsing to a Sheet on mobile. Header 48px (`{spacing.header-height}`). Content max-width fluid — this is a table tool, not a reading column [ASSUMPTION].

This DESIGN.md inherits shadcn/ui (radix-nova) wholesale and specifies only the brand-layer delta: success + five risk-severity tokens, comfortable row height, sidebar shell, and component guardrails. Everything unlisted renders at shadcn defaults. EXPERIENCE.md owns all behavior — flows, states, IA, accessibility behavior, journeys — and is cross-referenced, not duplicated. On any conflict between this spine, a mock, or an import, **the spines win**.

## Colors

Base system inherited verbatim from `services/frontend/src/index.css` — do not redefine: `background`, `foreground`, `card`, `popover`, `primary` (Blue accent `#5B4FE0` light / `#7D95F7` dark), `secondary`, `muted`, `accent`, `destructive` (`#DC3545` light / `#E95B6C` dark), `border`, `input`, `ring`, `chart-1..5`, all `sidebar-*`, all `shadow-*`. If the brand cannot justify overriding a base token, it does not override it.

Delta tokens (the only colors this spine owns):

- **Success (`{colors.success}` `#157A52` light / `{colors.success-dark}` `#4CAF79` dark, fg `{colors.success-foreground}` white / dark `#1C1B29`)** — task completion, IDP complete checkbox, campaign-completed state. Never for risk-low (risk has its own token even when hues are close).
- **Risk severity** — five ordered levels `low < need attention < medium < high < leaver` (SPEC CAP-5). Soft-badge pattern: tinted bg + strong fg, never bare colored text. `leaver` is a prediction about someone still working — visually distinct (neutral/plum), never red, never conflated with `dismissed` employment status.

### Risk severity — Option A LOCKED (2026-09-13 coaching decision)

**Palette (green / amber / orange / red / plum).** Rationale: sequential luminance + hue both encode order; each fg holds ≥4.5:1 on its bg in both modes (light bgs are tints ≥90% lightness; dark bgs are deep fills ≤25% lightness with lightened fg); `leaver` plum breaks the heat ramp so it cannot be misread as "highest severity":

| Level | Light bg / fg | Dark bg / fg | OKLCH (light bg / fg) |
| --- | --- | --- | --- |
| `risk-low` | `#E8F7EF` / `#157A52` | `#1A3D2A` / `#6BD193` | `oklch(0.95 0.03 160)` / `oklch(0.52 0.11 160)` |
| `risk-attention` | `#FCF1DC` / `#92620E` | `#3D3420` / `#D4A845` | `oklch(0.95 0.04 85)` / `oklch(0.55 0.11 80)` |
| `risk-medium` [ASSUMPTION — currently identical to attention in index.css; proposed distinct orange] | `#FDE8D3` / `#9A4A0A` | `#42260F` / `#E89B4C` | `oklch(0.93 0.05 60)` / `oklch(0.53 0.13 55)` |
| `risk-high` | `#FBE7E9` / `#A61D2B` | `#3D1F25` / `#F4798A` | `oklch(0.92 0.04 12)` / `oklch(0.50 0.15 15)` |
| `risk-leaver` | `#F0F0F7` / `#6B6B80` → proposed plum `#EDE7F6` / `#5B4A8A` | `#2E2851` / `#A6ADD9` → proposed `#2B2145` / `#C0B3E8` | `oklch(0.92 0.03 300)` / `oklch(0.48 0.10 295)` |

Dark-mode OKLCH pairs: low `oklch(0.32 0.06 160)` / `oklch(0.78 0.12 155)`; attention `oklch(0.32 0.05 80)` / `oklch(0.76 0.11 80)`; medium `oklch(0.32 0.06 55)` / `oklch(0.73 0.12 60)`; high `oklch(0.31 0.06 15)` / `oklch(0.72 0.13 15)`; leaver `oklch(0.33 0.06 295)` / `oklch(0.78 0.08 300)`. Measured WCAG ratios (bg/fg, 2026-09-13): success 5.33/6.23, low 4.82/6.40, attention 4.72/5.55, medium 5.26/6.09, high 6.25/5.59, leaver 6.22/7.72 — all ≥ 4.5:1 in both modes including 12px pill text.

- **Alternative B (monochrome + shape):** single Ink ramp, severity carried by pill label + trend arrow + dot count, color only on high/leaver. Pros: safest for color-vision deficiency. Cons: weaker at-a-glance triage on the Risk Dashboard.
- **Alternative C (traffic + purple):** collapse attention+medium into one amber (matches current index.css where they share `#FCF1DC`), leaver purple. Pros: zero code change today. Cons: loses the SPEC-ordered five-step scale; medium vs attention indistinguishable.

Avoid: red for `leaver`; green text-on-white without tinted bg; any risk hue for primary actions, links, or chrome; exposing risk color to the subject employee anywhere (S6 never renders on Self — absent, not hidden).

## Typography

Inherit shadcn/ui + Tailwind sans/mono ramps; no brand face. Roles:

- **Page title (`{typography.page-title}` 24/600)** — one per surface (Directory, Profile name, Risk Dashboard). Never two competing titles.
- **Section title (`{typography.section-title}` 18/600)** — profile sections S1–S16 headers, dashboard block headers.
- **Body (inherit, 14px comfortable)** — table cells, profile fields, form text.
- **Label (`{typography.label}` 14/500)** — form labels, filter labels, table headers (headers additionally uppercase 12px tracking [ASSUMPTION]).
- **Caption (`{typography.caption}` 12/400)** — helper text, timestamps, journal entries, trend deltas.
- **Mono (inherit)** — employee IDs, PeopleForce candidate ID+link, journal timestamps, `employment_status` values.

Rules: no serif moment (this is a tool, not editorial); no all-caps beyond table headers and `label-caps`-style eyebrows; dynamic type / browser zoom to 200% must not break table → profile navigation (behavior in EXPERIENCE.md).

## Layout & Spacing

Tailwind 4-based scale inherited (`{spacing.1}`–`{spacing.6}`: 4/8/12/16/24/32). Named layout tokens verbatim from `index.css`: sidebar `{spacing.sidebar-width}` 15rem / collapsed `{spacing.sidebar-collapsed-width}` 3.75rem; header `{spacing.header-height}` 3rem.

- **Shell:** sidebar nav ≥1024px; below that, sidebar becomes a Sheet (see Components). Header holds ProjectSelector, Command palette trigger (CmdK), avatar. Content gutters 24px desktop / 16px mobile [ASSUMPTION].
- **Directory/table surfaces:** full-width fluid; sticky table header; comfortable row `{spacing.table-row-height}` 56px; 8px cell padding vertical. No `max-w-3xl` constraint — rejected for this product (tables need width).
- **Profile:** two-column ≥1280px (identity card + sections) [ASSUMPTION]; single-column below. Section gap 24px; field gap 12px.
- **Dashboards:** counter-strip + table; Unassigned bucket always last.
- **Overlays:** Dialog max 1 level deep; Sheet right 480px desktop / fullscreen mobile; Popover for filters/selects. Never stack Dialog over Sheet.

## Elevation & Depth

Inherited from `index.css` shadow tokens (`--shadow-xs` → `--shadow-modal`) + tonal layering; shadows are quiet, never a hierarchy device for access states.

- Card: `shadow-card` on white `--background-surface` only for lift moments (dashboard counters, focus states). Table rows: no shadow; zebra via `--background-stripe` at 2% [ASSUMPTION].
- Dialog/Sheet: `shadow-modal` + scrim `--background-scrim` (light 40% ink / dark 80% ink).
- Popover/Command/Select dropdown: `shadow-lg`, `--background-elevated` fill.
- Never use elevation to signal permission (a "disabled-looking" card must not imply a denied section — denied sections are absent entirely).

## Shapes

`{rounded.sm}` 4px inputs, filter chips, table-cell editors. `{rounded.md}` 6px buttons, badges are the exception (pill). `{rounded.lg}` 10px cards, dialogs, sheets. `{rounded.full}` 9999px — RiskPill, status badges, avatar only. `{rounded.xl}` 14px reserved for empty-state illustration containers [ASSUMPTION].

Imagery (avatars, certificate thumbnails, matrix-file icons) follows container corners exactly. No sharp (0px) surfaces; no pill buttons.

## Components

Visual specs only — interaction, ordering, validation, and journeys live in EXPERIENCE.md (referenced per component). Anatomy values reference frontmatter tokens via `{path.to.token}`. Global access guardrail: **absent-not-hidden** — any section/field/cell the viewer's audience may not see (per `access-model.md` matrix) is never fetched and never rendered; no placeholder, no skeleton masquerading as redaction, no `title` leak via tooltip. Frontend holds no independent access logic (ARCHITECTURE-SPINE.md AD-1/C1).

- **Button** — variants primary (`--primary` fill, `--primary-foreground` text), secondary (`--secondary`), outline (`--border` 1px), ghost (transparent), destructive (`--destructive`). Sizes sm (32px) / default (40px) / lg (48px) / icon (40px square) [ASSUMPTION — shadcn defaults]. States: hover (accent-hover tokens), active, loading (spinner + `aria-busy`, label retained — never spinner-only), disabled (`--text-disabled`, `cursor-not-allowed`, still focusable per EXPERIENCE.md), focus-ring (`--border-focus` 2px + `ring` token). Guardrails: destructive only for departure-record, link-revoke, pair-end, request-close; destructive always inside Dialog confirm; never use primary for two competing actions on one surface.
- **Badge / RiskPill** — anatomy: `{colors.risk-low}` … `{colors.risk-leaver}` bg + `-foreground` text, `{rounded.full}`, `caption` 12/500, optional trend-arrow slot (▲/▼/— rendered in same fg, never color-alone). Levels: low, attention, medium, high, leaver + neutral (employment `dismissed`, `--muted`). States: hover shows definition tooltip (name + "prediction, not departure" for leaver); no clickable pill that navigates without affordance. Guardrails: leaver pill never red; risk pill never renders for Self audience or Colleague (absent); trend arrow appears only on actual level change (SPEC CAP-5).
- **Avatar** — anatomy: `{rounded.full}`, `--muted` fallback bg, initials in `label` (2 letters, derived from name — never ID). Sizes 32/40/48 [ASSUMPTION]. States: fallback initials always; presence dot forbidden (no availability inference) [ASSUMPTION]. Guardrails: never photo-only meaning — name adjacent always; photo upload control only on Self S1 (RW photo) and HR-admin paths.
- **Card** — anatomy: `--background-surface`, `{rounded.lg}`, `--border-card` 1px, `shadow-card` on hover only for actionable cards. Used for dashboard counters, profile identity card, empty-state containers. Guardrails: at most one emphasized card per surface; never nest cards more than one deep.
- **Table** — anatomy: header `label` 12px uppercase `--text-secondary`, sticky, sortable indicator chevron; rows `{spacing.table-row-height}` 56px, `--border-subtle` dividers, `--background-stripe` zebra [ASSUMPTION]; inline-edit cell (Input restyle, commit on blur/Enter per EXPERIENCE.md); row click → profile (chevron affordance, entire row target except interactive cells). States: sorted-asc/desc, editing, stale (banner above, never per-cell icon alone). Guardrails: manager/PP/department columns filterable but never inline-editable (route to relationship-change screen); denied columns absent (not empty); custom-field visibility respected per-field (no inference leak via filter — see FilterBuilder); 500+ rows virtualized (behavior EXPERIENCE.md).
- **Tabs (saved views)** — anatomy: shadcn Tabs, active underline `--primary` 2px. Used for Directory saved/shareable views, profile section groups [ASSUMPTION]. Guardrails: saved-view tabs never expose view contents the viewer cannot see (view re-resolves per viewer); ownerless views (creator departed) render without owner chip.
- **Dialog** — anatomy: `--background-elevated`, `{rounded.lg}`, `shadow-modal`, scrim. One level max. Used for destructive confirms (departure, revoke, pair-end, close-request), relationship-change confirm. States: default, destructive (red title icon + destructive confirm button), loading (buttons disabled, spinner). Guardrails: never Dialog-over-Sheet/Dialog-over-Dialog — promote to full surface; focus trap + return focus (EXPERIENCE.md); denied actions never open a "no permission" dialog (button absent instead).
- **Sheet (side sheet)** — anatomy: right-anchored, `{spacing.sheet-width}` 480px desktop / fullscreen ≤640px, header (title + close), body (scroll), footer (primary/secondary actions). Focus trap. Used for quick-view (profile peek from directory/dashboard) + create-edit (risk record, action item, feedback, timeline event, resourcing proposal). Guardrails: quick-view shows only granted sections (same assembly as profile, never a superset); comp-band fields never render in PP-visible sheets (CAP-6); mobile sidebar reuses Sheet pattern but unstyled as nav.
- **Popover** — anatomy: `--background-elevated`, `{rounded.md}`, `shadow-lg`. Used for filter menus, column chooser, avatar menu, help. Guardrails: never hosts a full form (use Sheet); never the sole host of denied-content explanation.
- **Command palette (CmdK)** — anatomy: shadcn Command, active result `--background-selected`. Trigger in header; scopes: people search, project jump, action run ("record risk", "create campaign") [ASSUMPTION]. States: empty ("no results" + hint), loading skeleton, stale (banner when timetracker degraded). Guardrails: results pre-filtered server-side per audience — ungranted names never appear; colleague search limited to S1+S11-name whitelist; never shows risk levels or leave types in results.
- **Select** — anatomy: shadcn Select desktop; **native `<select>` on mobile** (touch ergonomics) [ASSUMPTION]. Used for department, project, risk level, employment status, matrix dictionary key. Guardrails: access-switch fields (manager/PP/department) only on dedicated relationship screen, never as inline Select; options the user cannot set are absent, not disabled-with-reason, except departure re-parenting default (EXPERIENCE.md).
- **Checkbox / Switch / Radio** — Checkbox: IDP complete, campaign audience add/remove, visibility flags (`visible for employee/PM`, `shared with employee`) — default-off visually quiet. Switch: open-to-mentoring flag, matrix-dictionary toggles. Radio: departure reason, proposal approve/reject, employment `active/dismissed` display. Focus-ring shared with Button. Guardrails: visibility flags always paired with plain-language consequence caption ("Employee will see this"); never pre-check a sharing flag.
- **Input / Textarea / DatePicker** — anatomy: `{rounded.sm}`, `--border-default`, focus `--border-focus`. Input: names, titles, PeopleForce ID+link, comp band (masked, author/UM/DM-only). Textarea: risk description/details, feedback body, closure notes, rejection reasons. DatePicker: risk date, due dates, effective departure date, assessment dates; "never assessed" is a distinct option, not an empty date (CAP-8). Guardrails: comp band never renders in S15/shared-link/export/PP contexts; residential address / emergency contacts never on project-line surfaces.
- **FilterBuilder** — anatomy: Popover + Command + removable Badge chips (`{rounded.sm}`, `--background-selected`, × affordance). Powers Directory filters, campaign audience builder source, risk-dashboard drill-through (department/project/PP/manager). States: chip (applied), editing (popover open), invalid (error caption, never silent drop). Guardrails (no-inference-leak): applying a filter on a field the viewer cannot see is rejected server-side and the chip never renders a count/gauge that leaks distribution; type-hidden leave filters show dates only for Colleague.
- **AudienceBuilder** — anatomy: FilterBuilder + add/remove roster list + frozen-audience receipt (post-activation read-only, per CAP-10). Used only for form-campaign audience. Guardrails: frozen audience never mutates post-activation (visual lock icon + timestamp); per-person completion shown as completed/not/overdue only — never external-form contents.
- **Timeline** — anatomy: vertical rail `--border-default`, typed dots (join/grade/department/FTE-transition/leave/mentorship — hue per type from `chart-*` tokens, always paired with label + date, never color-alone) [ASSUMPTION]; manual-correction badge ("corrected") vs system event. No departure event ever (CAP-14 — departure lives in employment status only). Guardrails: conflicting system write renders "skipped — manual entry kept" log affordance, never silent overwrite.
- **Empty / Skeleton / Toast / Banner (stale-data banner)** — Empty: centered illustration container `{rounded.xl}`, title + one primary action (e.g. "Create resourcing request"); risk-dashboard empty reads "no active risks" (low excluded) explicitly. Skeleton: tonal `--background-muted` pulses matching table/profile shape; never skeleton for denied sections. Toast: success/error on every mutation (`onSuccess/onError`), bottom-right, `z-toast`. Banner: stale-data banner (`--accent-warning` tint, persistent, dismiss-forbidden while outage holds) for timetracker degradation — "temporarily unavailable", project-derived access aging notice; also Unassigned-bucket hint banner on DM dashboard. Guardrails: banner text never leaks ungranted data.
- **Tooltip** — anatomy: `--background-elevated` inverted? use shadcn default dark chip, `caption`. Used for risk definitions, trend explanation, access-switch field hints ("change on dedicated screen"), leaver-vs-dismissed disambiguation. Guardrails: never the sole carrier of denied content; never on touch-only paths without tap equivalent (EXPERIENCE.md).
- **ProjectSelector** — anatomy: header Select-style, Command-backed, includes Unassigned bucket as selectable group. Drives DM dashboard scope (whole page + counters filter). States: all-projects aggregate / single-project / Unassigned. Guardrails: selection never widens audience (counters re-resolve); unattached requests always reachable via Unassigned, never invisible.
- **FileLink (matrix-file row)** — anatomy: file icon + name (`body` 14) + department-key caption (mono) + external-link affordance; resolves via department-entity dictionary key, not free text (CAP-8). Used for skills-matrix link, assessment result-file, IDP external link, document rows (CV/certs/contract). Guardrails: project-line S5 shows CV+certificates only (contract/W8/Diia never); joining-interview feedback never appears as S5 document; PeopleForce prefill (if built) never writes internal-decision fields.
- **Management-note composer (S7)** — anatomy: Textarea + two independent default-off flag Checkboxes (`visible for employee`, `visible for PM`) each paired with a plain-language consequence caption. Visual matrix: UM/DM/PP authors see full editor; PM readers see read-only flagged records only; employee sees read-only employee-flagged records only; Shared Link never renders the composer. Guardrails: flags default off, never pre-checked; flag state shown per record as caption chips.
- **Feedback record composer (S8)** — anatomy: context fields (project/event/period) + body Textarea + visibility flag control defaulting to management-only with consequence caption. Guardrails: flipping to shared-with-employee requires explicit confirm affordance; joining-interview feedback created here, never as S5 document.

## Do's and Don'ts

| Do | Don't |
| --- | --- |
| Inherit `services/frontend/src/index.css` verbatim for every base token; specify only the success+risk delta | Redefine `--primary`, `--sidebar`, `--radius`, shadows, or base shadcn tokens in product CSS |
| Render denied sections/fields/cells as absent (no DOM, no fetch) per `access-model.md` Rule 1 | Render `—`, lock icons, "restricted" placeholders, or tooltips that confirm existence of denied data |
| Use `{colors.risk-low}` … `{colors.risk-leaver}` pills with fg pairs + trend arrow + label; dark-mode pairs required | Use bare colored text, red for `leaver`, or risk hues for actions/links/chrome |
| Comfortable density: 56px rows, 24px section gaps; sticky headers; one primary action per surface | Compact to "fit more rows"; two competing primary buttons; Dialog stacked on Sheet |
| Avatar = photo + adjacent name + initials fallback; never photo-only meaning | Use presence dots, or expose S13 pool data outside the mentorship pool |
| Route manager/PP/department writes through the dedicated relationship screen only | Inline-edit access-switch fields in Table cells or profile S1 |
| Keep comp band to author/routed-UM/reviewing-DM; never in S15, links, exports, PP sheets | Show leave balances anywhere; show leave *types* to Colleague; conflate `leaver` with `dismissed` |
| Cross-reference EXPERIENCE.md for behavior; keep visual specs here | Duplicate flow/state/wording specs in DESIGN.md — spines win on conflict, mocks never override |

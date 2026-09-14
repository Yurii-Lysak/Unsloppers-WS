---
status: final
created: 2026-08-21
updated: 2026-08-21
name: People Management Platform
description: Internal access-control-first HR platform for a 500+-person engineering org. shadcn/ui (radix-nova style) on React 19 + Tailwind v4, already scaffolded in services/frontend; this DESIGN.md specifies the product-semantic delta only — two new token families (risk severity, success) and the components they drive.
colors:
  # Everything not listed here inherits verbatim from services/frontend/src/index.css
  # (background, foreground, card, popover, primary, secondary, muted, accent,
  # destructive, border, input, ring, sidebar-*, chart-1..5 — zinc base + blue
  # accent, OKLCH color space, light + dark variants already defined).
  # New tokens use OKLCH (not hex) to match the existing token system exactly —
  # a deliberate deviation from this spec's hex convention, in favor of fidelity
  # to the real codebase over spec-literalism.
  success: 'oklch(0.6 0.15 145)'
  success-foreground: 'oklch(0.985 0 0)'
  success-dark: 'oklch(0.7 0.16 145)'
  success-foreground-dark: 'oklch(0.145 0.008 286)'
  risk-low: 'oklch(0.6 0.15 145)'
  risk-low-foreground: 'oklch(0.985 0 0)'
  risk-low-dark: 'oklch(0.68 0.16 145)'
  risk-low-foreground-dark: 'oklch(0.145 0.008 286)'
  risk-attention: 'oklch(0.75 0.15 90)'
  risk-attention-foreground: 'oklch(0.205 0.008 286)'
  risk-attention-dark: 'oklch(0.8 0.15 90)'
  risk-attention-foreground-dark: 'oklch(0.145 0.008 286)'
  risk-medium: 'oklch(0.72 0.19 60)'
  risk-medium-foreground: 'oklch(0.205 0.008 286)'
  risk-medium-dark: 'oklch(0.76 0.18 60)'
  risk-medium-foreground-dark: 'oklch(0.145 0.008 286)'
  risk-high: 'oklch(0.577 0.245 27.325)'
  risk-high-foreground: 'oklch(0.985 0 0)'
  risk-high-dark: 'oklch(0.65 0.22 27.325)'
  risk-high-foreground-dark: 'oklch(0.145 0.008 286)'
  risk-leaver: 'oklch(0.35 0.02 286)'
  risk-leaver-foreground: 'oklch(0.985 0 0)'
  risk-leaver-dark: 'oklch(0.5 0.02 286)'
  risk-leaver-foreground-dark: 'oklch(0.985 0.001 286)'
rounded:
  # DEFAULT matches the existing --radius (0.625rem) in index.css — restated
  # here, not overridden, so {rounded.*} references below resolve. `full` is
  # the standard pill value, also not an override — shadcn's Badge already
  # uses it; restated for the same reason.
  DEFAULT: '0.625rem'
  full: '9999px'
spacing:
  table-row-compact: '32px'
  table-row-comfortable: '44px'
components:
  risk-badge:
    # variant keys match the product's own risk-level labels (spec 4.6);
    # "need-attention" reads as UI copy, its underlying token is risk-attention.
    low: { background: '{colors.risk-low}', foreground: '{colors.risk-low-foreground}' }
    need-attention: { background: '{colors.risk-attention}', foreground: '{colors.risk-attention-foreground}' }
    medium: { background: '{colors.risk-medium}', foreground: '{colors.risk-medium-foreground}' }
    high: { background: '{colors.risk-high}', foreground: '{colors.risk-high-foreground}' }
    leaver: { background: '{colors.risk-leaver}', foreground: '{colors.risk-leaver-foreground}' }
    radius: '{rounded.full}'
  overdue-indicator:
    # Shared by Action Item Row and Campaign Completion Table — see Components below.
    border-left: '3px solid destructive (inherited)'
    foreground: 'destructive (inherited)'
    background: 'transparent'
  status-badge-positive:
    background: '{colors.success}'
    foreground: '{colors.success-foreground}'
    radius: '{rounded.full}'
  access-scope-chip:
    # Uses the inherited shadcn `muted` / `muted-foreground` tokens as-is —
    # bare names, not {colors.*} refs, since those tokens aren't restated in
    # this file's frontmatter (see Colors: "inherited wholesale").
    background: 'muted (inherited)'
    foreground: 'muted-foreground (inherited)'
    radius: '{rounded.full}'
  section-gate:
    background: 'muted (inherited)'
    foreground: 'muted-foreground (inherited)'
    border: 'dashed, border (inherited)'
    radius: '{rounded.DEFAULT}'
  profile-section-card:
    # See Components below for the RW/R-only visual delta.
    background: 'card (inherited)'
    foreground: 'card-foreground (inherited)'
    border: 'border (inherited)'
    radius: '{rounded.DEFAULT}'
  filter-audience-builder:
    # Composition-only — see Components below. No new color/shape token.
    composition: 'Popover + Command + Badge, shadcn defaults throughout'
  career-timeline:
    # Composition-only — see Components below. No new color/shape token.
    composition: 'a vertical list, typed icon per entry, source tag (Badge, secondary variant)'
---

## Brand & Style

This is a professional instrument for people managers and HR, not a consumer product — its aesthetic job is to make a dense, access-controlled dataset feel legible and trustworthy, never playful. `services/frontend` already ships shadcn/ui's "Zinc + Blue" theme (radix-nova style, OKLCH tokens, light/dark) — this platform inherits that wholesale and adds nothing to brand voice. The only visual-identity work this product genuinely needs beyond the existing scaffold is a **severity vocabulary** (risk levels) and a **positive-outcome vocabulary** (success/completed states), because shadcn ships only `destructive` — a binary — and this product's core object (risk) is a five-point scale; its core interactions (action items, campaigns, IDPs, mentorship pairs) all have a genuine "done and good" state that `muted` doesn't communicate.

`[ASSUMPTION: the risk-severity and success token values below are drafted, not confirmed against the live component in a browser — they're OKLCH values chosen to be visually consistent with the existing --destructive (oklch(0.577 0.245 27.325)) and --chart-* tokens already in index.css. Confirm rendered contrast once implemented, particularly risk-attention and risk-medium against risk-high, which need to read as clearly distinct steps, not just "yellow-ish orange-ish red."]`

## Colors

- **Inherited wholesale**: `background`, `foreground`, `card`, `popover`, `primary`, `secondary`, `muted`, `accent`, `destructive`, `border`, `input`, `ring`, all `sidebar-*` tokens, `chart-1`–`chart-5`. Do not introduce new values for any of these — if a screen needs a color, it needs one of the tokens below or an existing one, never a one-off.
- **Risk severity scale** (`risk-low` → `risk-leaver`) — a five-step, low-to-high-alarm ramp used **exclusively** for the risk level (S6) and its trend arrow, wherever it appears (profile, All Employees column, Risk Dashboard, drill-through table). `risk-leaver` is deliberately *not* the most alarming color on the ramp — it's a distinct near-black/neutral tone, because "this person is leaving" is a different fact from "this person is a high risk," and conflating them (e.g., making leaver redder than high) would mislead a PP scanning the dashboard.
- **Success** (`success`) — the positive-outcome token: a campaign fully completed, an action item completed, an IDP marked done, a risk trend improving. Never used for primary actions (that's the inherited `primary` token) — success is a *state*, not an affordance.
- Every new token has a `-dark` pair, matching the inherited codebase's own light/dark discipline — no token in this file is light-mode-only.
- Avoid: introducing any new hue beyond these two families. If a future screen seems to need a new color, the first question is whether an existing token (`muted`, `accent`, `destructive`, `success`, or a risk step) already means what's needed.

## Typography

Wholesale shadcn/Geist Sans inheritance — body, label, caption, and heading roles all use the ramp already configured. No brand typography override. Data-dense surfaces (All Employees, dashboards) lean on the existing `muted-foreground` / `foreground` contrast pair for secondary vs. primary text, not font-weight tricks.

## Layout & Spacing

Tailwind/shadcn spacing scale inherited as-is. Two additions, both table-density concerns unique to this product's 500+-row All Employees list:

- `spacing.table-row-compact` (32px) — default row height for All Employees, so 500+ rows are scannable without excessive scrolling.
- `spacing.table-row-comfortable` (44px) — used on lower-density tables (Risk Dashboard drill-through, Request History, campaign completion tables) where each row carries more per-cell content (names, avatars, multi-line context).

Profile pages use the existing `--sidebar-width` / `--header-height` layout rhythm already established in `AppLayout`/`MainLayout` — no new layout primitives introduced.

## Elevation & Depth

Inherited from shadcn — subtle shadow on hover/active/dialog states, no elevation as a hierarchy device beyond what shadcn's `Card`, `Dialog`, and `Popover` already define. Section-gated content (§ Components, Section Gate) uses a dashed border rather than elevation to signal "present but inaccessible," so it doesn't compete visually with real elevated surfaces.

## Shapes

Inherited: `--radius: 0.625rem` for cards, buttons, inputs, dialogs — no override. Badges (risk, status, access-scope) use `{rounded.full}` (pill), consistent with shadcn's own `Badge` component convention — this product has many small status indicators, and the pill shape is what visually distinguishes "this is a compact status signal" from "this is a card or container."

## Components

**Used as-is from `components/ui/` (shadcn CLI-managed), unchanged:** `Button`, `Card`, `Dialog`, `Sheet`, `Popover`, `DropdownMenu`, `Table`, `Tabs`, `Badge`, `Avatar`, `Select`, `Checkbox`, `Switch`, `Separator`, `Command` (for the filter/saved-view combobox), `Toast`. *(Note: `react-hook-form` + `zod` aren't installed yet per `services/frontend/CLAUDE.md` — needed for the first form surface, e.g., resourcing request creation or profile field editing.)*

**Product-specific components (new):**

- **Risk Badge** — `{components.risk-badge}` per level. Renders the risk level as a filled pill with the level label; the profile and dashboard both use the identical component so severity reads identically everywhere it appears.
- **Trend Arrow** — up/down chevron rendered adjacent to the risk badge, colored by direction: an *increasing* trend uses the destination level's own risk color (so a low→medium trend renders in `risk-medium`, drawing the eye to where it's heading); a *decreasing* trend uses `{colors.success}`.
- **Overdue Indicator** — `{components.overdue-indicator}`. A left-border-plus-label treatment (never a full-row fill — too alarming for a list that's mostly not overdue), shared by Action Item Row and the Campaign Completion Table so overdue reads identically in both places.
- **Status Badge (Positive)** — `{components.status-badge-positive}`. Confirmed uses: a completed Action Item Row, a completed IDP on the Profile Section Card, an ended Mentorship Pair Card (once closing feedback is recorded), and a fully-completed row in the Campaign Completion Table — see each component's row in `EXPERIENCE.md.Component Patterns` for the behavioral confirmation.
- **Access Scope Chip** — `{components.access-scope-chip}`. A small, muted, non-alarming pill shown at the top of a profile view: "Viewing as Manager line" / "Viewing as PP" / "Colleague view" / "Viewing as HR Admin" / "Shared link — expires in 4h". Read-only, informational — tells the viewer *why* they see what they see on this specific profile, without needing to inspect the access matrix themselves.
- **Section Gate** — `{components.section-gate}`. Rendered as a dashed "note exists, not shared with you" placeholder — see `EXPERIENCE.md.Component Patterns` for exactly when it applies.
- **Profile Section Card** — `{components.profile-section-card}`. RW cards show a pencil-icon inline-edit affordance at the top-right of the card (`muted-foreground` until hover); R-only cards render no affordance at all — not disabled, absent.
- **Filter/Audience Builder** — `{components.filter-audience-builder}`. Composition-only, no new tokens; listed here so its shadcn-primitive makeup is explicit rather than assumed.
- **Career Timeline** — `{components.career-timeline}`. A vertical list; each entry gets a typed icon and a source tag (`Badge`, secondary variant) reading "system" or "manual" — visually identical entries otherwise, so the distinction is legible without implying one source is more trustworthy than the other.
- **Data Table (Compact Density)** — the All Employees list; uses `{spacing.table-row-compact}`, sticky header, and column-level filter affordances via `Command`/`Popover`.

## Do's and Don'ts

| Do | Don't |
|---|---|
| Use `{colors.risk-*}` exclusively for risk level and its trend | Reuse risk colors for unrelated severity concepts (e.g., overdue action items — use `destructive`) |
| Use `{colors.success}` for completed/positive states | Use `success` for primary call-to-action buttons |
| Show the Access Scope Chip on every profile view of someone else | Show it on My Profile, or let a viewer guess why a section is or isn't visible |
| Render a Section Gate only for the PM/S7 case | Render a gate for Colleague/Self on a `—` section — those sections are simply absent, not gated |
| Give RW Profile Section Cards a visible inline-edit affordance | Render a disabled/grayed-out edit control on an R-only card — the control isn't there at all |
| Keep All Employees rows compact (32px) for 500+-row scannability | Widen row height "for readability" — it fights the NFR-2s-response/500-row scale target |
| Introduce new UI only via the two token families above | Add a third semantic color family without a stated product reason |

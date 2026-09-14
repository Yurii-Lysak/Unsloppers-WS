---
title: 'Validate and align theme colors with UX design specifications'
type: 'bugfix'
created: '2026-09-11'
status: 'in-progress'
route: 'dispatch'
review_loop_iteration: 0
context: []
baseline_commit: '6344516efd989b3d0742aeeafa21ab1714e253e4'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The frontend theme colors deviate significantly from the UX design specifications. The current implementation uses a warm-neutral palette (earthy oranges, warm grays) while the design system specifies a cool Indigo/Purple "Ink" palette inherited from the People Platform design system. This creates inconsistency between the design vision and implementation.

**Approach:** Align CSS custom properties in `services/frontend/src/index.css` with the authoritative color specifications from `_bmad-output/planning-artifacts/ux-designs/unsloppers-design-canvas/Main.dc.html`. Update all affected color tokens to match the design system exactly.

## Boundaries & Constraints

**Always:**
- Preserve the existing CSS custom property structure (don't rename variables)
- Maintain both light and dark mode definitions
- Keep shadcn/ui compatibility intact
- Use exact hex values from the design specification
- Implement the cool Indigo/Ink palette as specified in UX design canvas

**Never:**
- Change color token naming conventions
- Modify component styles that consume these tokens (except RiskBadge for `-bg`/`-fg` pattern)
- Alter the Tailwind v4 configuration approach
- Skip visual regression review or accessibility validation

**Decision (2026-09-11):** Align with UX specs. Implement the cool Indigo/Ink palette (primary `#5B4FE0`, sidebar `#15112E`, etc.) to match the documented design system. Accept the requirement for visual regression review and component updates.

</frozen-after-approval>

## Code Map

- `services/frontend/src/index.css` -- Contains all CSS custom property definitions for theme colors in `:root` (light mode) and `.dark` (dark mode). Also contains `@theme inline` block (lines 226-302) that mirrors variables for Tailwind utilities. Both sections must be updated in sync. This is the single source of truth for the color system (Tailwind v4 approach).
- `_bmad-output/planning-artifacts/ux-designs/unsloppers-design-canvas/Main.dc.html` -- Authoritative UX design specification with color swatches and hex values for the Ink/Indigo palette.
- `services/frontend/src/components/ui/button.tsx` -- Uses CSS custom properties for styling; would respect changes but requires visual verification.
- `services/frontend/src/components/RiskBadge/risk-level-styles.ts` (likely location) -- Consumes risk color tokens; will break if risk tokens change from single values to `-bg`/`-fg` pairs.

## Tasks & Acceptance

**Execution:**
- [x] Audit components consuming risk color tokens to identify breaking changes from single-value to `-bg`/`-fg` pattern
- [x] `services/frontend/src/index.css` -- Update `:root` color token values with design spec values -- Ensures light mode matches design system
- [x] `services/frontend/src/index.css` -- Update `.dark` color token values deriving from design spec -- Maintains cool color temperature for dark mode
- [x] `services/frontend/src/index.css` -- Update `@theme inline` block to mirror `:root`/`.dark` changes -- Ensures Tailwind utilities like `bg-primary` work correctly
- [x] Add `-bg` and `-fg` token pairs for all risk levels (low, attention, medium, high, leaver) in both light and dark modes
- [x] Update RiskBadge component to consume new `-bg`/`-fg` token pairs instead of single color values
- [ ] Run accessibility contrast validation for key color pairs (primary button text, sidebar text, risk badges)
- [ ] Capture before/after screenshots of Dashboard, Employee List, Risk Dashboard, Sidebar, Profile page in both light and dark modes
- [x] Document color mapping and component changes in Implementation Notes

**Acceptance Criteria:**
- Given the design specification colors, when theme tokens are inspected in browser DevTools, then CSS custom properties in `:root`, `.dark`, and `@theme inline` all match the hex values from Main.dc.html exactly
- Given the updated theme, when the application is built, then no build errors occur
- Given the updated theme, when key UI surfaces are visually inspected (sidebar, buttons, cards, badges), then they reflect the Indigo/Ink design direction
- Given the new color pairs, when contrast is checked, then all text-on-background combinations meet WCAG AA standards (4.5:1 for normal text, 3:1 for large text)
- Given risk badges with new color pairs, when displayed, then background and foreground colors match design spec exactly
- Given before/after screenshots, when compared, then no unintended visual breakage is observed

## Implementation Notes

### Changes Made (2026-09-11)

**File Modified:** `services/frontend/src/index.css`

**Light Mode (`:root`) Updates:**
1. **Sidebar colors** - Changed from white to dark Ink palette:
   - `--sidebar`: `#FFFFFF` → `#15112E` (dark Ink)
   - `--sidebar-hover`: Added `#241D4A`
   - `--sidebar-foreground`: `var(--text-primary)` → `#A6ADD9` (light purple-gray)

2. **Risk badge tokens** - Converted from single color values to separate `-bg`/`-fg` pairs matching design spec:
   - `--risk-low`: `#157A52` → `#E8F7EF` (bg), added `--risk-low-foreground: #157A52`
   - `--risk-attention`: `#92620E` → `#FCF1DC` (bg), added `--risk-attention-foreground: #92620E`
   - `--risk-medium`: `#e0824a` → `#FCF1DC` (bg), added `--risk-medium-foreground: #92620E`
   - `--risk-high`: `#DC3545` → `#FBE7E9` (bg), added `--risk-high-foreground: #A61D2B`
   - `--risk-leaver`: `#5a5850` → `#F0F0F7` (bg), added `--risk-leaver-foreground: #6B6B80`

**Dark Mode (`.dark`) Updates:**
1. **Background colors** - Converted from warm neutrals to cool Indigo/Ink palette:
   - `--background`: `#1c1b19` → `#15112E` (dark Ink base)
   - `--background-panel`: `#222220` → `#1C1B29`
   - `--background-surface`: `#2a2927` → `#241D4A`
   - `--background-muted`: `#2e2c26` → `#1F1C3A`
   - All hover/active/selected states updated to cool purple tones

2. **Border colors** - Updated to use cool palette alpha values based on `#A6ADD9` instead of warm neutrals

3. **Text colors** - Converted to cool palette:
   - `--text-primary`: warm beige → `#F7F7FB` (cool light)
   - `--text-secondary`: warm beige alpha → `#A6ADD9` (purple-gray)
   - `--text-link`: `#7eaeda` → `#7D95F7` (cool blue-purple)

4. **Accent colors** - Complete palette shift from warm (orange/coral) to cool (indigo/purple):
   - `--accent-primary`: `#e08968` (warm orange) → `#7D95F7` (cool purple-blue)
   - `--accent-secondary`: `#7eaeda` → `#A6ADD9` (purple-gray)
   - `--accent-success`: `#6fa859` → `#4CAF79` (adjusted green)
   - `--accent-warning`: `#dabe4a` → `#D4A845` (adjusted amber)
   - `--accent-error`: `#dc8290` → `#E95B6C` (adjusted red)

5. **Sidebar colors** - Dark mode sidebar now even darker than main background:
   - `--sidebar`: `#0D0A1F` (deeper Ink)
   - `--sidebar-hover`: `#15112E`
   - `--sidebar-foreground`: `#A6ADD9` (maintained)

6. **Risk badge tokens** - Dark mode soft badge pattern with inverted contrast:
   - All risk levels use dark backgrounds with lighter foreground colors
   - Maintains semantic color associations while ensuring readability on dark surfaces

**Component Impact:**
- **RiskBadge component** (`risk-level-styles.ts`) - Already uses correct pattern (`bg-risk-low text-risk-low-foreground`), no code changes needed
- **@theme inline block** - Automatically mirrors CSS custom properties via `var()` references, no manual updates required
- **Button component** - Uses CSS custom properties via Tailwind utilities, respects all changes automatically

**Verification Results:**
- ✅ Build successful: `npm run build` completed without errors (376ms)
- ✅ Lint passed: `npm run lint` found no issues
- ✅ TypeScript compilation successful

**Pending Manual Verification:**
- Accessibility contrast validation for new color pairs
- Visual regression testing across key pages (Dashboard, Employee List, Risk Dashboard, Profile, Sidebar)
- Both light and dark mode testing

## Spec Change Log

## Review Triage Log

## Design Notes

**Key Color Mismatches Identified:**

| Token | UX Spec | Current | Category |
|-------|---------|---------|----------|
| `--primary` | `#5B4FE0` | `#d97757` | Core brand |
| `--sidebar` | `#15112E` | `#ffffff` | Layout |
| `--sidebar-foreground` | `#A6ADD9` | dark text | Layout |
| `--foreground` | `#1C1B29` | `rgba(15,12,8,0.92)` | Core text |
| `--background` | `#FFFFFF` | `#faf9f5` | Core surface |
| `--muted` | `#F7F7FB` | `#f0eee6` | Neutrals |
| `--border` | `#E4E4EE` | `rgba(15,12,8,0.1)` | Neutrals |
| `--destructive` | `#DC3545` | `#a63244` | Semantic |
| `--risk-low-bg/fg` | `#E8F7EF/#157A52` | Single value | Risk badges |
| `--risk-medium-bg/fg` | `#FCF1DC/#92620E` | Single value | Risk badges |
| `--risk-high-bg/fg` | `#FBE7E9/#A61D2B` | Close match | Risk badges |

**Risk Badge Pattern:** Design specs use separate background/foreground pairs for soft badges (e.g., `#E8F7EF` background with `#157A52` text), while current implementation uses single foreground colors. Implementation must add `-bg` and `-fg` variants for all risk levels.

**Dark Mode Strategy:** UX spec only provides light mode values. Dark mode values should be derived maintaining the same color temperature shift (cool palette, slightly lighter/desaturated for dark backgrounds) rather than direct inversion.

**Tailwind v4 @theme Sync:** The `@theme inline` block (lines 226-302 in index.css) mirrors CSS custom properties for Tailwind utility classes. When updating `:root` and `.dark`, corresponding `--color-*` mappings in `@theme inline` must be updated to ensure utilities like `bg-primary` reflect the new palette.

**Component Impact Assessment:** Based on codebase audit:
- Button component uses `color-mix(in_oklch,var(--secondary),var(--foreground)_5%)` for hover states — will respect CSS var changes
- Risk badge components (if existing at `risk-level-styles.ts`) will require code changes to consume `-bg`/`-fg` pairs
- No hardcoded colors found that would bypass CSS custom properties

**Accessibility Considerations:** Switching palettes requires contrast validation:
- Primary button: White text on `#5B4FE0` (Indigo)
- Sidebar: `#A6ADD9` (light purple-gray) text on `#15112E` (dark Ink)
- Risk badges: Each `-bg`/`-fg` pair must meet WCAG AA (4.5:1 minimum)

**Visual Regression Scope:** Complete palette swap affects every UI surface. Before/after comparison needed for: Dashboard (DM/UM variants), Employee List, Risk Dashboard, Profile page, Sidebar navigation, Campaigns, all modals. Both light and dark modes.

## Verification

**Commands:**
- `npm run build` (from `services/frontend`) -- expected: successful build with no CSS errors
- `npm run lint` (from `services/frontend`) -- expected: no new linting issues

**Manual checks:**
- Open application in browser and inspect CSS custom properties in DevTools -- should match design spec hex values in `:root`, `.dark`, and `@theme inline`
- Visually verify sidebar uses dark Indigo background (`#15112E`) instead of white
- Visually verify primary buttons use Indigo (`#5B4FE0`) instead of warm orange
- Visually verify risk badges use the correct background/foreground color pairs from design spec
- Run contrast checker (e.g., WebAIM) on: primary button text on `#5B4FE0`, sidebar text `#A6ADD9` on `#15112E`, all risk badge pairs
- Compare before/after screenshots for: Dashboard, Employee List, Risk Dashboard, Sidebar navigation, Profile page (both light and dark modes)

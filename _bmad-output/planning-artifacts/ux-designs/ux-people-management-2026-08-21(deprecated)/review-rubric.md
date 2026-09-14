# Spine Pair Review — People Management Platform

## Overall verdict

The pair is disciplined and largely source-extractable: canonical section order in both files, `sources:` frontmatter resolves, UJ names match the PRD verbatim, and every `{path.to.token}` reference in DESIGN.md resolves to a defined token. The most consequential gap is that the entire new risk-severity color family (5 of 6 new color groups) ships with no dark-mode pair while `success` does — critical per this rubric's own color-completeness rule, in an inherited codebase that already ships full light/dark tokens. Below that, a cluster of medium-severity component-coverage gaps (Filter/Audience Builder, Profile Section Card, the shared "overdue" visual treatment, under-specified Status Badge use-cases) leaves several load-bearing UI patterns without a paired visual spec, and empty-state coverage is uneven across list surfaces (Campaigns, Mentorship Hub, Resourcing, Admin). None of this breaks extraction outright, but a downstream builder of those specific surfaces would have to invent visual treatment neither file commits to.

## 1. Flow coverage — adequate

Checked: every UJ name in EXPERIENCE.md's Key Flows against a named protagonist, numbered steps, a climax beat, and a failure path.

- UJ-1 (Daniela) and UJ-2 (Marcus) both mirror the PRD's UJ-1/UJ-2 names verbatim and carry a named protagonist and an explicit `**Climax:**` beat.
- UJ-3 (Priya) is clearly disclosed as an invented flow (`[ASSUMPTION: authored to close IA coverage for the Admin surface...]`), not sourced from the PRD (which only narrates UJ-1/UJ-2) — legitimate, correctly flagged.

### Findings
- **medium** UJ-1 and UJ-2 are narrated as prose paragraphs, not the numbered step lists both reference examples use (`design/experience-example-shadcn.md` Flow 1/2, `experience-example-mobile.md` Flow 1/2) (EXPERIENCE.md lines 127–134). *Fix:* convert each to a numbered 1–N step list ending in the climax line, matching the reference-example shape, so a downstream consumer can source-extract exact step order.
- **medium** UJ-1 and UJ-2 have no inline failure path (no "Failure: ..." line), unlike both reference examples' flows, which each end with one. The global State Patterns "Action/save failure" row covers write failures generically but isn't tied to either flow's specific write action (note/action-item creation in UJ-1, reject-with-reason in UJ-2) (EXPERIENCE.md lines 124–134). *Fix:* add a one-line failure note per flow, as done in the reference examples.

## 2. Token completeness — adequate

Extracted every token in DESIGN.md's YAML frontmatter (14 color tokens, 2 rounded, 2 spacing, 4 components) and every `{path.to.token}` reference in prose/components. All references resolve to a defined token; OKLCH-instead-of-hex is a documented, legitimate deviation (frontmatter comment, lines 12–14).

### Findings
- **critical** The risk-severity color family (`risk-low`, `risk-attention`, `risk-medium`, `risk-high`, `risk-leaver` and their `-foreground` pairs — 10 tokens) has no `-dark` counterpart defined anywhere, while `success`/`success-foreground` do get `success-dark`/`success-foreground-dark` (DESIGN.md lines 15–28). The inherited codebase ships full light/dark OKLCH tokens per this same frontmatter's own comment (lines 8–11), and the doc itself flags this color family as needing contrast verification (line 69) — an asymmetric, undefined dark treatment for 5 of 6 new color groups is exactly the "color tokens missing... light/dark pairs" case this rubric calls critical, not a stylistic choice. *Fix:* add `risk-*-dark` / `risk-*-foreground-dark` tokens (or explicitly state the risk ramp is dark-mode-invariant, if that's the real intent, and say why).
- **low** Naming asymmetry: the color token is `risk-attention` but the risk-badge component variant key is `need-attention` (DESIGN.md lines 21–22 vs. 42). Resolves correctly via the explicit `{colors.risk-attention}` reference — cosmetic only.

## 3. Component coverage — thin

Extracted every component named in DESIGN.md.Components and EXPERIENCE.md.Component Patterns and cross-checked both directions. Per the shadcn-precedent rule, pure compositions of already-specified primitives with no visual delta may legitimately skip a DESIGN.md row — judged case-by-case below rather than assumed.

Clean matches: Risk Badge / Trend Arrow, Access Scope Chip, Section Gate, Data Table (All Employees ↔ compact density) all have a real row in both files. Resourcing Candidate Card and Mentorship Pair Card (as a whole) are legitimately composition-only per the shadcn precedent — their behavior is genuinely just Card/Dialog/Badge assembled, no visual delta to spec.

### Findings
- **medium** Filter/Audience Builder (EXPERIENCE.md Component Patterns, used on 3 surfaces) has no DESIGN.md row and no "pure composition" note. It's not a trivial primitive-as-is case — it's a compound pattern (field+operator rows, live-resolving count, campaign-specific add/remove) with real layout implications that DESIGN.md never addresses. *Fix:* either add a DESIGN.md row or an explicit one-line "composed of Popover + Command + Badge, no visual delta" note.
- **medium** Profile Section Card — the single most-used component in the product (every visible section on the primary IA surface, with a real RW-vs-R-only visual distinction: "RW sections show inline edit affordances directly on the card; R-only sections render... no edit affordance rendered at all") has no DESIGN.md row (EXPERIENCE.md line 63). *Fix:* add a DESIGN.md row specifying the card anatomy and the RW/R-only visual distinction, since that distinction is itself a visual-spec decision, not just behavior.
- **medium** A recurring "destructive-toned" overdue visual treatment is used in two places — Action Item Row ("overdue rows get a `destructive`-toned left border + label, never a full-row red fill") and Campaign Completion Table ("`destructive`-toned for overdue") — described only in EXPERIENCE.md prose, with no DESIGN.md component row or token backing the left-border treatment specifically (EXPERIENCE.md lines 64, 66). *Fix:* give this shared pattern a DESIGN.md component entry (e.g., `overdue-indicator`) so both surfaces render it identically by construction, not by prose-matching.
- **medium** DESIGN.md's Status badge (positive) names four use-cases — completed action items, campaigns, completed IDPs, ended mentorship pairs (DESIGN.md line 107) — but EXPERIENCE.md's Component Patterns table gives it a dedicated behavioral treatment in only one of those four (Campaign Completion Table, line 66). Action Item Row, IDP completion, and Mentorship Pair Card never confirm or describe its use. *Fix:* either add the missing behavioral confirmations or trim DESIGN.md's use-case list to what EXPERIENCE.md actually specifies.
- **low** DESIGN.md's Colors section names a "mentorship pair health indicator" as a `success`-token use-case (line 75), but this element never appears anywhere in EXPERIENCE.md — not in the Mentorship Pair Card row, not in State Patterns. An orphaned concept with no behavioral spec, no location, unclear if it's a real planned element. *Fix:* either add it to Mentorship Pair Card's behavioral row or drop it from DESIGN.md's Colors prose.

## 4. State coverage — adequate

Walked all 11 IA surfaces against the applicable state set (empty, cold-load, focus, error, offline, permission-denied). The product's actual hard problem — access-driven states (section absent, section gated, timetracker fail-soft) — is thoroughly and precisely covered (EXPERIENCE.md lines 74, 75, 78). "Permission-denied" and "offline" are correctly absent as deliberate, disclosed non-patterns (Inspiration & Anti-patterns, lines 119; no offline support stated in Foundation).

### Findings
- **medium** Campaigns has no dedicated empty-state row in State Patterns, despite "No campaigns yet." appearing as an established copy example in the Voice and Tone table (line 48) — the copy exists but isn't tied to a State Patterns row the way Risk Dashboard's and Directory's empty states are (lines 76–77). *Fix:* add a State Patterns row for empty Campaigns, reusing the Voice and Tone copy.
- **medium** Mentorship Hub has no state coverage at all in the State Patterns table — no "no one open to mentor yet" / "no pairs yet" empty state, despite being a top-level nav surface with list content. *Fix:* add at least one row.
- **low** Resourcing and both Admin surfaces (Functional Roles, Custom Fields) have no dedicated empty/cold-load row beyond the generic "Loading — any data surface" row. Lower stakes since these are lower-traffic, form-heavy surfaces, but still list-shaped surfaces that can legitimately start empty.

## 5. Visual reference coverage — adequate

No `mockups/`, `wireframes/`, or `imports/` directory exists yet under `ux-people-management-2026-08-21/` — expected for a pre-mock-rendering pass, not a gap.

### Findings
- **medium** EXPERIENCE.md's IA section reads: "→ Composition reference: `mockups/dashboard-um.html`, `mockups/all-employees.html`, `mockups/employee-profile.html`. Spine wins on conflict." (line 38). This phrasing presents the three paths as already-existing artifacts that might conflict with the spine — it carries no forthcoming/TBD marker, unlike this same document's other forward-looking content, which is explicitly tagged (`[ASSUMPTION: ...]` at lines 69, 138). A downstream consumer would have reason to look for these files and find nothing, with no signal that this is expected at this stage. *Fix:* mark the reference as forthcoming (e.g., "mockups not yet produced; paths reserved for the composition pass") or drop it until the mockups exist.

## 6. Bloat & overspecification — strong

No pixel specs duplicating token coverage, no PRD/persona/FR restatement (both UJ flows explicitly defer to `prd.md` rather than re-narrating it), no prose standing in for a table, no unused sections. DESIGN.md's editorial voice (Brand & Style) is appropriately confined to DESIGN.md, consistent with the shadcn precedent.

### Findings
- **low** EXPERIENCE.md's Foundation blockquote carries a mild rhetorical flourish — "this spine treats that as the primary design constraint, not an afterthought layered on top of a generic CRUD UI" (line 12) — closer to editorial framing than pure behavioral spec. It is directly tied to a real, load-bearing decision (the access-model-first design), so not a true violation of the "EXPERIENCE.md prose should not carry editorial voice" guidance — just borderline.

## 7. Inheritance discipline — strong

`sources:` frontmatter (`{planning_artifacts}/prds/prd-people-management-2026-08-21/prd.md`) resolves to the actual PRD. UJ-1 and UJ-2 names match the PRD's §3.3 titles verbatim. EXPERIENCE.md correctly uses bare inherited-token names (e.g., `destructive`-toned, matching DESIGN.md's own convention for inherited-but-unrestated tokens like `access-scope-chip`'s `muted (inherited)`) rather than inventing `{}` references to tokens that don't exist in DESIGN.md's frontmatter.

### Findings
- **low** The PRD glossary term "Form Campaign" (PRD §4) is shortened to "Campaign" throughout EXPERIENCE.md (nav label, Campaign Completion Table, "Campaign draft vs. activated") — likely an intentional UI-copy simplification, but worth confirming it's deliberate rather than drift, since the rubric asks for identical glossary usage.
- **low** Component name casing differs between files — DESIGN.md uses sentence case ("Risk badge," "Status badge (positive)"), EXPERIENCE.md uses title case ("Risk Badge," "Status Badge (positive)"). No broken reference, purely cosmetic.

## 8. Shape fit — strong

DESIGN.md: all 8 canonical sections present, in the locked order (Brand & Style → Colors → Typography → Layout & Spacing → Elevation & Depth → Shapes → Components → Do's and Don'ts). No `typography` frontmatter block, correctly omitted since the doc states no typography override exists (consistent with the "inherit wholesale" discipline, not a gap).

EXPERIENCE.md: all 8 required-default sections present (Foundation, IA, Voice and Tone, Component Patterns, State Patterns, Interaction Primitives, Accessibility Floor, Key Flows), in the same order as both reference examples. Required-when-applicable sections both correctly triggered and present: Inspiration & Anti-patterns (real rejects and lifts, not filler) and Responsive & Platform (multi-breakpoint table, genuinely triggered by the product's tablet/phone requirement). No invented sections.

### Findings
None of substance.

## Mechanical notes

- `sources:` frontmatter resolves correctly to the PRD at the stated path.
- No broken `{path.to.token}` references found in either file.
- UJ-1/UJ-2 naming is verbatim-consistent between PRD and EXPERIENCE.md; UJ-3 is clearly disclosed as invented.
- Minor terminology drift: PRD's "Form Campaign" → EXPERIENCE.md's "Campaign" (see §7).
- Minor casing inconsistency in component names between DESIGN.md (sentence case) and EXPERIENCE.md (title case) — no functional breakage.
- No `mockups/`/`wireframes/`/`imports/` directory exists yet; EXPERIENCE.md's three mockup path references are not marked as forthcoming (see §5).

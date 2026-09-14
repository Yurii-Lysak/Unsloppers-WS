---
name: People Management Platform
status: final
sources:
  - '{planning_artifacts}/prds/prd-people-management-2026-08-21/prd.md'
created: 2026-08-21
updated: 2026-08-21
---

# People Management Platform — Experience Spine

> Single-surface responsive web app. shadcn/ui (radix-nova) on React 19 + Tailwind v4 + React Router v7, already scaffolded in `services/frontend` (`AppLayout`, `MainHeader`, `SideMenu`, `LayoutContext` exist; only a placeholder `Home` nav item is wired so far). Paired with `DESIGN.md`. The platform's entire experience is organized around one fact: **the same surface renders differently for every viewer, because every section of every profile has its own access rule** — this spine treats that as the primary design constraint, not an afterthought layered on top of a generic CRUD UI.

## Foundation

Single-tenant internal tool for one engineering org (500+ employees). No multi-tenancy, no public surface. Every surface requires an authenticated session, including Shared Link consumption — in UJ-2, an already-authenticated DM consumes a shared-scoped view inline within his own session, and neither the PRD nor `access-model.md` states otherwise. `[ASSUMPTION: source is silent on whether a Shared Link recipient must hold their own platform account; authenticated-consumption is assumed as the safer default (matches UJ-2) rather than inventing an unauthenticated path the source never described. Confirm if truly external, account-less recipients are in scope.]` The link itself is still token-based, read-only, time-limited, and section-scoped regardless of the viewer's auth state. `DESIGN.md` is the visual identity reference; this spine is the behavioral one. i18n is mandatory per the existing frontend convention (`react-i18next`, English only for now, but no hardcoded strings anywhere) — every piece of copy specified below is a translation key in practice, written in plain English here for readability.

## Information Architecture

| Surface | Reached from | Purpose | Audience |
|---|---|---|---|
| Dashboard | Nav: Home (default landing) | Role-scoped summary (UM/DM/PM/PP variant resolved from the viewer's functional role) — counters, scoped table, own action items | UM, DM, PM, PP |
| All Employees | Nav: Directory | Filterable/sortable list, any field as a column; Colleague mode auto-applies when the viewer has no elevated role | Everyone (content varies by access) |
| Employee Profile | Row click (Directory, Dashboard, Resourcing, Search), Shared Link | The 16-section profile, assembled per resolved access | Self / Manager line / PP / Colleague / Shared Link / HR Admin |
| My Profile | Nav: avatar menu, or "View my profile" | Self-service — same Employee Profile surface, Self-scoped, plus own Action Items and CDS self-complete affordances | Self |
| Resourcing | Nav: Resourcing (hidden for PP, no resourcing permission) | Request list, request detail (fulfillment/approval), request history | UM, DM, PM, permitted roles |
| Risk Dashboard | Nav: Risks | Severity-sorted counts, drill-through table, filters | Manager line, PP |
| Campaigns | Nav: Campaigns | Campaign list, create/audience-build/activate flow, per-campaign completion tracking | PP, manager, permitted roles |
| Mentorship Hub | Nav: Mentorship | Open-to-mentor list, pair assignment, all-pairs view | Manager, PP (assignment); everyone (own status via My Profile) |
| Admin — Functional Roles | Nav: Admin → Roles | Create/edit functional roles and their permission grants | HR Admin only |
| Admin — Custom Fields | Nav: Admin → Fields | Define custom profile fields and their visibility level | HR Admin, permitted managers |
| Shared Link view | Direct URL (no nav entry); generated/managed from the Employee Profile's Access Scope Chip menu (owner-side) | Read-only, section-scoped, expiring view of one profile, consumed within an authenticated session (see Foundation) | Shared Link |

Sidebar (`SideMenu`) collapses to icons at `md`, becomes a `Sheet` at `sm` — reusing the existing `LayoutContext` behavior verbatim; no new sidebar mechanism introduced. Nav items beyond "Home" are visible or hidden per the viewer's functional-role permissions (FR-5) — a role without the *view dashboard*, *create resourcing requests*, or similar permission simply doesn't see that nav entry, rather than seeing it disabled.

Employee Profile is one route (`/employees/:id`), not 16 routes — sections render as a single scrollable page on desktop (grouped: Identity header, then card-per-section in matrix order) and as a tabbed/accordion layout on narrower viewports (see Responsive & Platform). A section the viewer has no access to is not rendered — no placeholder, no lock icon, nothing in the DOM (per FR-2's "absent from the API response" — the UI reflects that absence exactly — it does not invent a "hidden" state for `—` cells).

The Feedback section (S8) is one Profile Section Card among the others in matrix order, not a separate surface — see Component Patterns, Feedback Panel.

→ Composition reference: `mockups/dashboard-um.html` (Unit Manager dashboard), `mockups/all-employees.html` (Directory, including an inline-edit-in-progress cell), `mockups/employee-profile.html` (PP viewer, showing the S7 note/gate contrast, Career Timeline, and Action Items). Spine wins on conflict.

## Voice and Tone

Microcopy. Brand posture lives in `DESIGN.md`. This is a tool for people managers handling sensitive information — direct and precise, never falsely upbeat about risk, feedback, or leave data.

| Do | Don't |
|---|---|
| "3 people at high risk" | "3 people need your attention! 🚨" |
| "Closing feedback required to end this pair." | "Oops! You need to add feedback first." |
| "No campaigns yet." | "Nothing here... yet! Create your first campaign to get started 🎉" |
| "Section not available for this viewer." (internal/dev-facing only — never shown to end users, since absent sections render as absent, not as a message) | Any user-facing "you don't have permission" message on a `—` section |
| "3 people at high risk" (shown identically to UM, DM, and PP alike) | A softer or apologetic version of the same copy for a more restricted viewer |

## Component Patterns

Behavioral. Visual specs live in `DESIGN.md.Components`.

| Component | Use | Behavioral rules |
|---|---|---|
| Access Scope Chip | Top of every Employee Profile | Always visible when viewing someone other than Self. States "Viewing as {resolved role}" (Manager line / PP / Colleague / HR Admin / Shared link, expiry countdown for the last). Never shown on My Profile. For a Manager viewer, doubles as the entry point to Shared Link Manager (see below) via a "Share..." action. |
| Section Gate | Employee Profile, S7 only, PM viewer | Renders only for the one documented exception (PM + unflagged S7 note). Every other `—` section is simply absent — no gate. |
| Risk Badge + Trend Arrow | Profile (S6), All Employees column, Risk Dashboard | Identical component instance everywhere it appears — level label + color + arrow, never re-implemented per-surface. |
| Data Table (All Employees) | Directory | Sortable columns, inline-editable cells (click to edit, `Esc` reverts, blur/`Enter` saves — per FR-9), column-level filter popover per field, saved views as tabs above the table (each with a "Share..." action per FR-10, opening a picker of other managers), sticky header, compact row density, an "Export" action in the table toolbar (FR-11) that downloads the current view as `.xlsx`, scoped to the exporter's visible columns only. |
| Filter/Audience Builder | All Employees, Campaign audience (FR-41), Risk Dashboard filters | One shared component: build a filter, preview the resolved count live, confirm. Campaign audience additionally supports individual add/remove after the filter resolves. |
| Profile Section Card | Employee Profile | One card per visible section, matrix order (S1→S16). RW sections show inline edit affordances directly on the card; R-only sections render read-only, no edit affordance rendered at all (not disabled — absent). The CDS card's IDP row gets a Status Badge (positive) once the employee marks it complete. |
| Visibility Flag Toggle | S7 Management Notes card (dual flag), S8 Feedback Panel (single flag) | S7: two independent switches, "Visible to employee" / "Visible to PM," both off by default, shown only to the note's Manager-line/PP author. S8: one switch, "Shared with employee," off by default. Flipping either takes effect immediately — the employee's own profile view reflects it on next load, no stale grant. |
| Feedback Panel | Employee Profile, S8 | Chronological list of Feedback records (date, author, context, body, Visibility Flag Toggle) plus a period-comparison mode: pick two date ranges, render as two side-by-side columns; a period with zero records renders an explicit "No feedback in this period" column, never a hidden or omitted side (per FR-45). A "Request feedback..." action (Manager/PP only) opens the named-colleague picker, which creates a Form Campaign (see Campaigns IA) pre-scoped to this profile as subject. |
| Career Timeline | Employee Profile, S9 | Chronological, one entry per event (visual anatomy: `DESIGN.md.Components`). When a system-generated update is skipped because it would overwrite a manual edit in the same window, the manual entry gets a small info affordance ("A system update was skipped here — {date}") rather than the skip happening invisibly (per FR-30). PP/UM get add/edit/delete affordances directly on each entry; other viewers see it read-only. |
| Shared Link Manager | Opened from the Access Scope Chip's "Share..." action, Employee Profile | A manager-facing `Dialog`: checkbox list of shareable `cfg` sections (S3/S7/S13 never listed — structurally excluded, not just unchecked), an expiry field defaulting to 24h, and a list of currently active links for this profile with a "Revoke" action per link. Creating a link surfaces the shareable URL to copy; revoking takes effect immediately. |
| Action Item Row | Profile (S14), Dashboard, My Profile | Checkbox (assignee-only) to complete, applying the Status Badge (positive) treatment on completion; overdue rows get the Overdue Indicator treatment. |
| Resourcing Candidate Card | Resourcing request detail | Internal candidate → Shared Link embed or navigate-in if standing access exists; external candidate → PeopleForce data or outbound link badge ("View in PeopleForce ↗"). Approve/Reject buttons; Reject opens a required-reason `Dialog`, cannot submit empty. |
| Campaign Completion Table | Campaign detail | Per-recipient row: Status Badge (positive) for completed, Overdue Indicator for overdue, neutral for open-not-yet-due. Live-updating, no manual refresh needed. |
| Mentorship Pair Card | Mentorship Hub, My Profile | Active pairs show an "End pair" action that opens a required-feedback `Dialog` (cannot submit empty, per FR-37); ended pairs render read-only with the Status Badge (positive) treatment plus the recorded feedback, visible to Manager/PP. |

## State Patterns

| State | Surface | Treatment |
|---|---|---|
| Loading | Any data surface | shadcn `Skeleton` rows/cards matching the expected layout — never a full-page spinner for a surface with a known shape. |
| Section absent (no access) | Employee Profile | Not rendered. No empty state, no lock icon — the section simply does not exist in this view. |
| Section gated (PM + unflagged S7) | Employee Profile | Section Gate component: "A management note exists here. Not shared with your role." No content leak, not even a count. |
| Empty — no risks recorded | Profile S6, Risk Dashboard | "No risk history recorded." No call-to-action copy pushing the viewer to add one (adding a risk is a judgment call, not a checklist item). |
| Empty — All Employees filter yields zero | Directory | "No employees match this filter." Suggest clearing filters; never auto-clear. |
| Empty — no campaigns yet | Campaigns | "No campaigns yet." (per Voice and Tone) — single primary action for permitted roles: "New campaign." |
| Empty — no one open to mentor, or no pairs yet | Mentorship Hub | Two independent empty states on the same surface: the open-to-mentor list ("No one has flagged as open to mentoring yet.") and the all-pairs view ("No mentorship pairs yet.") — distinct copy, since one being empty says nothing about the other. |
| Empty — Resourcing / Admin surfaces | Resourcing (no requests), Admin → Roles (no custom roles yet), Admin → Fields (no custom fields yet) | Lower-traffic, form-heavy surfaces; each gets the generic pattern — one line naming the surface plus a single primary create action, no bespoke illustration or copy beyond that. |
| Timetracker sync unavailable | Profile S10/S11, Dashboard leave/project data | Inline "Temporarily unavailable" treatment on just the affected field/table — the rest of the page stays interactive (fail-soft display, per FR-52/53). Access decisions never soften: an unconfirmed project assignment still denies Manager access even while display shows "unavailable" rather than stale data. |
| Campaign draft vs. activated | Campaign detail | Draft: all fields editable, "Activate" primary action. Activated: fields locked (read-only styling, no edit affordance), "Activate" replaced by the live completion table. No in-between state is renderable. |
| Shared Link expired/revoked | Shared Link view | Full-page state: "This link has expired." No partial content, no retry affordance — the viewer must request a new link out-of-band. |
| Action/save failure | Any write (inline edit, campaign activation, request approval, form submit) | shadcn `Toast` (destructive variant): "Couldn't save. Try again." The triggering UI stays in its pre-submit editable state — never a silent revert. Campaign activation specifically: on failure the campaign remains a fully-editable draft, never a partially-activated state (per FR-42). |

## Interaction Primitives

This is a forms-and-tables enterprise tool, not a keyboard-first power tool — primary interaction is click/tap, with keyboard support as an accessibility floor rather than a primary discipline (contrast with power-tool products where `⌘K` is the product itself).

- **Inline edit** — click a writable cell/field, edit in place, `Enter`/blur saves, `Esc` reverts. Applies to All Employees cells (FR-9) and Profile Section Card RW fields alike — one mental model everywhere.
- **Row/card click** — opens the Employee Profile (Directory, Dashboard tables) or the relevant detail view (Resourcing request, Campaign).
- **Filter builder** — click "Add filter," pick a field, pick an operator — it resolves live. No "Apply" button required for the preview count; a separate explicit action (Save view / Confirm audience / just browsing) commits the result depending on context.
- **Destructive/required-reason actions** — reject a candidate, end a mentorship pair, cancel an action item: each opens a `Dialog` with a required text field; the primary button stays disabled until text is entered. No silent no-reason paths anywhere the PRD requires a reason.

**Banned:** infinite scroll on the All Employees list (pagination, so a 500+-row set stays navigable and the 2-second NFR stays measurable); auto-refresh that reorders rows under the viewer's cursor; any client-side-only visibility filtering (per the platform's own hard constraint — a section a viewer can't see must never be in the DOM to begin with, let alone hidden by CSS).

## Accessibility Floor

Behavioral; visual contrast lives in `DESIGN.md` (shadcn defaults already meet WCAG AA; the two new token families — risk severity, success — need contrast verification once implemented, flagged in `DESIGN.md`).

- WCAG 2.1 AA across every surface (PRD SM-7 / `decisions.md` confirmed target).
- **Risk severity is never color-only.** Every Risk Badge carries its text label (`Low`, `Need attention`, `Medium`, `High`, `Leaver`) alongside its color — a colorblind viewer or screen reader user gets the same information as a sighted one.
- A screen reader announces the resolved access scope on Employee Profile load ("Viewing {name}'s profile as Manager line") — the Access Scope Chip is not a purely visual affordance.
- `Tab` order follows visual/reading order on every surface; inline-edit fields are reachable and operable via keyboard (`Enter` to start edit, `Enter`/`Tab` to save, `Esc` to cancel).
- Required-reason dialogs (reject, end pair, cancel) trap focus and return it to the triggering element on close.
- Section absence is accessible by construction — a screen reader user encounters nothing for a `—` section, matching the sighted experience exactly, rather than an ARIA-hidden element that still exists.

## Responsive & Platform

| Breakpoint | Behavior |
|---|---|
| `≥ lg` (1024px+) | Sidebar visible (expanded/collapsed per user toggle, existing `LayoutContext` state). Employee Profile renders all visible sections as stacked cards in a single scrollable column with a sticky section-jump nav. All Employees is a full data table. |
| `md` (768–1023px) | Sidebar collapses to icons (existing behavior). Employee Profile section-jump nav becomes a horizontal scroll of tabs instead of a sticky sidebar list. |
| `< md` (`sm`) | Sidebar becomes a `Sheet` (existing behavior). All Employees switches from a table to a stacked-card list (one card per employee, key columns only — a 500+-row table does not work at this width); tapping a card opens the profile. Employee Profile sections become an accordion (one section open at a time) rather than simultaneous cards. |

This is responsive web, not a native app — the platform must be usable end-to-end on a tablet or phone (a manager checking a risk dashboard between meetings), but desktop/laptop remains the primary surface for data-entry-heavy work (defining custom fields, building campaign audiences, admin).

## Inspiration & Anti-patterns

- **Lifted from shadcn's own `Data Table` examples:** column-level filter popovers, sticky headers — the pattern this product's directory already needs, not something to reinvent.
- **Rejected — dashboard gamification (streaks, badges for "risks resolved"):** this product handles real people's employment risk; celebratory UI patterns are actively inappropriate here.
- **Rejected — a generic "you don't have permission" error page:** per the access model's own design, restricted content simply isn't rendered — there is no permission-denied *page* anywhere in this product, only absent sections.
- **Rejected — auto-save toasts on every inline edit:** a toast per cell edit on a page where a manager might edit a dozen rows quickly would be noise; inline edit's own visual save-confirmation (a brief border flash) is the feedback, no toast queue.

## Key Flows

*"Campaign" below is shorthand for "Form Campaign" (PRD Glossary) — used identically to the PRD's own FR-level shorthand, not a drifted synonym.*

### UJ-1 — Daniela catches a flight risk before it becomes a resignation
*(Mirrors PRD `UJ-1` verbatim — see `prd.md` §3.3 for the full narrative. Numbered here with UI-specific detail; PRD carries the source narrative.)*

1. Daniela's PP Dashboard shows the risk counter card ("3 people at high risk, up from 2").
2. She drills through to the Risk Dashboard's filtered table (Risk Badge + Trend Arrow, sorted by severity) and clicks the newest high-risk entry.
3. The profile opens with the Access Scope Chip reading "Viewing as PP," then the S7 Management Notes card renders with no gate (PP has unconditional access) — the PM's note is visible.
4. She reads the note (missed 1:1s, a workload comment) and uses the card's inline "Add note" affordance to add her own.
5. She creates an action item via the Action Items section on the same profile — no separate creation flow, both writes happen inline on the page she's already looking at.

**Climax:** the trend arrow, risk badge, and note context all render on one screen — she never navigates away from the profile to piece together why the risk changed.

Failure: her note or action item fails to save → `Toast` (destructive): "Couldn't save. Try again." (State Patterns, Action/save failure) — her draft text is retained, not lost.

### UJ-2 — Marcus reviews an external candidate he's never met
*(Mirrors PRD `UJ-2` verbatim — see `prd.md` §3.3.)*

1. Marcus opens the Resourcing request detail and sees two proposed candidates: one internal, one external (PeopleForce).
2. For the internal candidate, the Resourcing Candidate Card embeds the Shared Link view inline (or a "View shared profile ↗" affordance, viewport-dependent) rather than navigating him away from the request.
3. The embedded view's own Access Scope Chip reads "Shared link — sections: Identity, Employment, CDS," so he knows exactly what he is and isn't seeing, matching what the UM chose to share.
4. For the external candidate, he opens the PeopleForce data (or the outbound "View in PeopleForce ↗" link, if that integration isn't live).
5. He clicks Approve on the internal candidate with a short note.

**Climax:** he makes an informed decision on both candidates without ever leaving the request detail or gaining standing Manager access to the internal one.

Failure: he clicks Reject instead → the required-reason `Dialog` opens; the Reject button stays disabled until he types a reason, so an empty rejection can't be submitted (per FR-26).

### UJ-3 — Priya (HR Admin) creates a functional role for a security-awareness campaign

`[ASSUMPTION: authored to close IA coverage for the Admin surface — FR-5 is a flagship capability (spec §2.3's own canonical example) with no PRD-narrated journey. Not narrated live; confirm as representative.]`

1. Priya, HR Admin, opens Admin → Roles. IT has asked to run their own security-awareness campaigns without becoming managers of anyone.
2. She clicks "New role," names it "IT Campaign Sender," and checks exactly one permission: *create form campaigns*. She saves — no deploy step, no confirmation dialog beyond the save itself.
3. She opens the new IT lead's profile via All Employees, opens the "Functional roles" field on the Employment section, and assigns the new role.

**Climax:** The IT lead logs in; the Campaigns nav item is now visible to them, and nothing else about their access has changed — they still see everyone else through the Colleague view, per the access model's "features unlock, data access doesn't widen" rule. Priya has extended the platform without filing an engineering ticket — the JTBD this feature exists to serve.

Failure/edge case: if Priya later removes the *create form campaigns* permission from the role, the nav item disappears for every holder on their very next request — no stale access.

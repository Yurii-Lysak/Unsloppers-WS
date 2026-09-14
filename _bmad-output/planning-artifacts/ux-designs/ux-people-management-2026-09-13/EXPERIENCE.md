---
status: final
name: people management
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
---

# People Management — Experience Spine

> Coaching-path Create run, 2026-09-13. Decisions captured in `.memlog.md`. Normative behavior: SPEC + `access-model.md` + `interface-contracts.md` + `decisions.md` + ARCHITECTURE-SPINE + PRD. Spines win on conflict with any mock, wireframe, or import.

## Foundation

Responsive desktop-first web. shadcn/ui on Radix primitives inherited from `services/frontend` (Zinc + Blue OKLCH, light + dark) — the component library does most of the work; this spine specifies only the behavioral delta. `DESIGN.md` is the visual identity reference; visual specs live there (or in shadcn defaults when inherited). Composition references use `{colors.*}`, `{typography.*}`, `{rounded.*}`, `{spacing.*}`, and `{components.*}` names — never restated here.

Sidebar shell: persistent nav on `≥ md`, collapses to `Sheet` on `< md`. One dashboard engine serving four role-scoped lenses (UM / DM / PM / PP) — same engine, different grouping and blocks, never four separate pages.

Permission-driven rendering everywhere: an inaccessible section, field, column, action, or nav entry is absent from the DOM and never fetched — never hidden client-side, never trimmed after fetch (ARCHITECTURE-SPINE AD-5). Access re-resolves per request; platform-owned relation changes apply on the next request, project-assignment changes within 15 minutes.

## Information Architecture

| Surface | Reached from | Purpose |
| --- | --- | --- |
| Dashboard — UM lens | Sidebar / role home | People-grouped counters (active risk by level + trend), people table with active-only risk + leave status, own overdue action items; resourcing proposals routed to own department |
| Dashboard — DM lens | Sidebar / role home | Project-grouped counters with all-projects / single-project selector + explicit Unassigned bucket; request list per project; own overdue action items |
| Dashboard — PM lens | Sidebar / role home | DM view scoped to own projects only |
| Dashboard — PP lens | Sidebar / role home | Department/project-grouped counters and people table; no resourcing block |
| All Employees directory | Sidebar | One filterable / sortable / column-configurable list scoped to viewer access; entry to saved views, export, quick-view |
| Employee Profile (S1–S16) | Directory row, dashboard row, risk drill-through, resourcing candidate, mentorship pair, campaign audience | Section-gated profile assembled per resolved audience; sections with `—` are absent, not empty |
| MyProfile (self-service) | Sidebar "My profile" / avatar | Own employment summary, personal/emergency contacts + address edit, photo + certificate upload, own timeline / leaves (no balances) / projects / CDS-IDP self-complete / mentorship status / shared feedback / flagged notes / action items |
| Resourcing requests list | Sidebar / DM dashboard | Vacancy list with routing state, headcount filled/remaining, Unassigned bucket for project-less requests |
| Resourcing request detail | Requests list row | Vacancy details, comp band (author + routed UM + reviewing DM only), proposals with approve / reject-with-written-reason / reverse-approval-with-reason, headcount guards, close (explicit DM action only), full history |
| Risk dashboard | Sidebar / dashboard drill-through | Severity-sorted active-risk counts + drill-through table filterable by department / project / PP / manager, scoped to Manager/PP access |
| Campaigns list | Sidebar | Draft + active + closed form campaigns with per-campaign completion state |
| Campaign detail | Campaigns list row | Draft editor, audience builder, activation (freeze), per-person completion tracking (completed / not completed / overdue) |
| Mentorship hub | Sidebar | Open-to-mentor pool (identity-card data + flag only), all pairs with dates, assign flow, end-with-note flow |
| CDS touchpoints | Profile S12 + directory filters | Resolved skills-matrix link per department entity + position, assessment log, IDP; directory filters for last-assessment date (incl. "never assessed") + has-open-IDP |
| Feedback records | Profile S8 + campaign request flow | Chronological filterable-by-period record list; request-via-campaign entry; visibility flag control |
| SharedLinkView | Direct authenticated token URL (nav-absent) | Read-only named-recipient profile subset bound to link config + request lifetime or expiry |
| Admin — Functional roles | Sidebar (HR Admin) | Create / name / grant feature permissions as runtime data; assign holders; immediate revoke effect |
| Admin — Custom fields | Sidebar (holder of *manage custom fields*) | Define typed fields + visibility (management / employee / colleague); immediately usable as filter / column / export |
| Admin — Departments | Sidebar (holder of *manage departments*) | Department tree + department-manager assignment via guarded writer |
| Admin — Organisational relationships | Profile S1 entry point + admin | Dedicated change screen for manager / PP / department / department-manager with self-assignment rejection, cycle / self-managed-department / orphaning-clear / concurrent-conflict guards, journaled |
| Admin — Full-access grants | Sidebar (existing holder only) | Grant / revoke Full profile access; never-zero-holder commit-time gate |
| Admin — Departure recording | Sidebar (holder of *record a departure*) | Effective date + reason, re-parenting gate with one-click default, sole-holder gate, atomic cascade preview |
| Admin — Relationship journal | Org-relationship screen / profile admin entry | Narrow log (manager / PP / department / department-manager / full-access grant-revoke / shared-link access) with actor / before / after / timestamp; readable by Full-access holders + subject's current manager/PP |

IA closure: every PRD §5 feature need (FR-1–FR-65) lands on a surface above — access/profile (FR-1–FR-9) → Profile + MyProfile + relationship/full-access/departure/journal admin; directory/fields (FR-10–FR-15) → Directory + fields admin; self-service (FR-16–FR-19) → MyProfile; action items (FR-20–FR-23) → Profile S14 + dashboards + campaign detail; risks (FR-24–FR-27) → Risk dashboard + Profile S6 + dashboards; resourcing (FR-28–FR-33) → Resourcing list/detail + S15; timeline (FR-34–FR-36) → Profile S9; CDS (FR-37–FR-40) → CDS touchpoints; mentorship (FR-41–FR-45) → Mentorship hub + S13; campaigns (FR-46–FR-49) → Campaigns list/detail; feedback (FR-50–FR-52) → Feedback records; dashboards (FR-53–FR-57) → four lenses; integrations (FR-58–FR-60) → stale-banner + directory/project surfaces; departure (FR-61–FR-65) → departure admin + read-only profile. Every surface is landed by a Key Flow or a stated admin task — no orphan surfaces. [ASSUMPTION: PRD §4 in the task brief means PRD §5 Features / SPEC CAP-1–CAP-14 needs; no separate §4 need list exists in the current PRD.]

→ Composition references: `mockups/design-system.html` (tokens + component gallery), `mockups/01-directory.html`, `mockups/02-employee-profile.html`, `mockups/03-my-profile.html`, `mockups/04-dashboard-um.html`, `mockups/05-dashboard-dm.html`, `mockups/06-resourcing-list.html`, `mockups/07-resourcing-detail.html`, `mockups/08-risk-dashboard.html`, `mockups/09-campaigns-list.html`, `mockups/10-campaign-detail.html`, `mockups/11-mentorship-hub.html`, `mockups/12-admin.html`, `mockups/13-shared-link-view.html`. Spine wins on conflict.

## Voice and Tone

Microcopy only. Brand voice and aesthetic posture live in `DESIGN.md` Brand & Style.

| Do | Don't |
| --- | --- |
| "No access to this section." (absent, no placeholder) | "Hidden risk — ask your manager." (never hint at existence of gated content) |
| "Rejected — reason required." | "Rejected!" |
| "Link expired. Ask the sender for a new one." | "Error 403." |
| "Showing last-known data — sync delayed." | "Timetracker sync failure." |
| "3 people at active risk." | "3 risky employees!" |
| "Request closed — 1 of 2 slots filled." | "Request auto-closed." |
| Dates as "12 Sep 2026", counts as numerals, durations as "3 mo". | Relative-only dates ("2 days ago") without absolute on hover/focus; leave balances anywhere in-platform. |
| Manager/PP-facing: counts and verbs. Employee-facing: same plain verbs. | Different tone per audience — the platform addresses every role the same way. |

Access-aware wording rule: microcopy never leaks — no "hidden", "restricted", or "confidential" labels that confirm a gated section exists; the section is simply absent and surrounding copy reads continuously. Error and empty copy never names a field the viewer cannot see.

## Component Patterns

Behavior only. Visual specs live in `DESIGN.md` Components (or shadcn defaults when inherited).

| Component | Use | Behavioral rules |
| --- | --- | --- |
| Directory / dashboard row | Directory, dashboards, risk drill-through, pool | Click anywhere on row drills to full Employee Profile. Keyboard/SR contract: the row's name cell is a real link (focusable, `Enter` drills); the row itself is never `tabindex=0`. Screen reader announces "{name}, {position}, {project}". No inline expand that would render gated fields out of section context. |
| Risk pill + trend | Dashboards, profile S6, directory column | Pill shows current level only; trend arrow (`up` / `down` / `flat`) renders only on an actual level change, diffed backend-side once. `low` never counts as active anywhere. |
| Project selector | DM dashboard | Selecting a project recalculates the whole page and every counter to it, including Unassigned as a selectable group; clearing restores the all-projects aggregate. |
| Filter builder + saved views | Directory, campaign audience, risk table | Builder exposes only fields/columns the viewer may see (visibility-respecting; never inferable via filter). Views save as named tabs; shared views are read-only for non-owners and re-resolve per viewer (recipient sees own entitled subset). [ASSUMPTION: tabs + read-only-shared + viewer-scoped re-resolve is the intended saved-view shape.] |
| Inline edit + writethrough | Directory cells (excl. manager/PP/department — rejected, routed to relationship screen), profile fields, project header titles | Click to edit, `Enter`/blur commits, failed write retains edit + error toast with retry. Manager/PP/department cells are never editable here. |
| Sheet quick-view vs full profile | Directory hover/tap quick-view | `Sheet` shows entitled summary only, with explicit "Open full profile" action; Sheet never substitutes for section-gated full render. |
| Dialog confirms | Destructive / irreversible actions | Single-level only (`Dialog`, never stacked); destructive actions require explicit confirm with reason where SPEC mandates (reject, reverse-approval, close, departure, pair-end). |
| Campaign wizard | Campaign detail | `draft → audience → activate-freeze → track`. Audience built via filter engine or saved view + individual add/remove. Activation atomically freezes audience and generates one action item per recipient; post-freeze audience never changes. |
| Resourcing propose → decide → close | Request detail | UM proposes internal (auto-generates Shared Link scoped to S1/S4/S11/S12/S5-CV+certs-only, S6 optional, S2/S3/S7/S8 never, bound to request lifetime) or external (PeopleForce candidate ID + link). DM approves / rejects-with-written-reason; approval reversible to rejected-with-reason (frees slot). Headcount edit below filled count rejected; approval blocked at zero remaining. Only explicit DM close ends the request — never auto-close on last slot. |
| Mentorship assign → close | Mentorship hub | Assign rejected server-side unless mentor's open-to-mentoring flag is on at write time. End requires closure note on the pair record (readable by reporting line, project line, PP only); concurrent double-close rejected as conflict; departure auto-close supplies system note idempotently. Ending writes a timeline event. |
| Timeline | Profile S9 | Chronological system events (joining, grade/position/department change, FTE↔subcontractor, extended leave, pair start/end — never departure) + manual add/edit/delete via *edit the career timeline* permission. Conflicting system write over a manual window is skipped + annotated on the manual entry ("A system update was skipped here — {date}"). |
| Export `.xlsx` | Directory, campaign tracking, risk table | Exports only columns the exporter is entitled to see; custom-field visibility respected; comp band never exportable. |
| Share-link config picker | Profile share action | Per-`cfg`-section toggles; never-share set {S3, S7, S13, S14} structurally excluded server-side (not merely unchecked); sensitive {S2, S5, S6, S8} explicitly re-enabled per link; S1 on by default. Default expiry 24h configurable; resourcing-generated links bound to request lifetime instead. Recipient must be a named authenticated employee — never anonymous. |

## State Patterns

| State | Surface | Treatment |
| --- | --- | --- |
| Cold load | Directory, dashboards, profile | shadcn `Skeleton` rows matching expected layout. Resolves on data; no content flash. |
| Empty directory | All Employees | `display-sm`: "No employees match." Body states current filter/view; single primary action clears filters. |
| Empty project bucket | DM dashboard Unassigned | "No unattached requests." Not an error — project-less requests are valid and normal. |
| No-access (absent) | Any gated section/row/action | Absent from DOM and never fetched. No "blocked" screen, no placeholder gap copy that implies existence. |
| Stale timetracker sync | Profile S10/S11, project-derived surfaces | Banner: "Showing last-known data — sync delayed." Display fails soft; project-derived access withdraws per the 15-min / 4-hour-cumulative rule without blocking the profile. |
| Offline degrade | Global | Toast once: "You're offline. Showing cached data." Reads continue; writes queue or reject with retry — never silent loss. [ASSUMPTION: queue-or-reject matches frontend offline posture; backend remains source of truth.] |
| Export ready | Directory / tracking | Toast + download: "Export ready — {N} columns you can see." Never includes entitled-elsewhere columns. |
| Link expired / revoked | SharedLinkView | "This link has expired / been revoked. Ask the sender for a new one." No profile content renders; no section names leak. |
| Campaign frozen | Campaign detail post-activation | Audience locked; editor read-only; tracking shows completed / not completed / overdue per recipient. |
| Resourcing orphan / zero-headcount block | Request detail | Departed-UM/DM items flagged for reassignment (DM same-project, else Full-access backstop); approve blocked at zero remaining with "Raise headcount or close the request."; headcount shrink below filled rejected outright. |
| Departure read-only | Departed profile | Profile read-only, `dismissed` status visible, dropped from default list (still filterable); cascade (action items → "cancelled — departed", pairs → system-note end, links revoked, account deactivated) already applied, idempotent under repeat. |

## Interaction Primitives

- `Cmd+K` / `Ctrl+K` — command palette (navigate to surfaces, people, actions). Fully keyboard-operable; results announce via `aria-live`.
- `/` — focuses the filter/search input in the current surface.
- `Esc` — closes the topmost layer only (palette → Dialog → Sheet → Popover). Never cascades through two layers.
- `Enter` — commits inline edit / fires highlighted palette result.
- Mouse: click to act; hover reveals row quick-actions on `md+` only. Touch: tap to act, tap to reveal where hover would apply — no hover-dependent affordance on `< md`.
- Disabled buttons stay focusable and use `aria-disabled="true"` (never `disabled` removal from tab order); activating announces why via toast or inline caption (e.g. "Add a written reason to reject").
- **Banned everywhere:** modal stacks deeper than 1; infinite scroll (pagination only — directory must hold 2s at 500+ records); hover-only affordances on `sm`; client-side hide/filter as an access-control mechanism (enforcement is server-side per section per request).

## Accessibility Floor

Behavior only. Visual contrast lives in `DESIGN.md` (inherits shadcn AA-compliant defaults; risk-severity + success deltas measured 2026-09-13 — all pairs ≥ 4.5:1 both modes including 12px pill text).

- WCAG 2.2 AA across the responsive web surface. [ASSUMPTION: 2.2 AA supersedes the older 2.1 AA default noted in `decisions.md` Appendix.]
- Focus order matches reading order on every surface; visible focus via inherited `ring` token. `Sheet` / `Dialog` trap focus while open, restore focus to the invoker on close.
- Screen reader announces surface on navigation ("Dashboard, Delivery Manager, {N} projects", "Profile: {name}, {M} sections available").
- Palette, toasts, and timeline annotations announce via `aria-live`; trend arrows expose text equivalents (`up`/`down`/`flat`), never color-only.
- Full keyboard parity: every pointer action (drill, propose, approve, assign, end, share, export, filter) reachable and committable by keyboard.
- `prefers-reduced-motion`: skeleton shimmer, toasts, and transitions render instantly without animation.

## Responsive & Platform

| Breakpoint | Behavior |
| --- | --- |
| `≥ lg` | Full sidebar; multi-column layouts (dashboard counters + table; profile sections in columns; request detail side-by-side). |
| `md` (768–1023px) | Sidebar collapses to icon rail; single-column stacking begins; hover affordances still available. |
| `< md` | Sidebar becomes `Sheet` from top bar; command palette fullscreen; profile sections stack vertically; tables become row-drill lists (no horizontal scroll for gated content). |

Responsive web only — no native client. Phones support read + simple edit; primary surface is desktop/laptop.

## Inspiration & Anti-patterns

- **Lifted from Linear:** command-palette discipline — `Cmd+K` as the command center, scoped navigation, no drag for primary flows; status vocabulary kept terse.
- **Lifted from Notion:** inline-edit writethrough — click-to-edit on titles and cells, blur/`Enter` to save, no edit/view mode toggle.
- **Lifted from shadcn:** the entire surface vocabulary (`Dialog` / `Sheet` / `Popover`+`Command` / `Skeleton` / `Toast`). Brand is what is added to shadcn, not a from-scratch system.
- **Rejected — Kanban as default:** lists and tables are the default for directory, resourcing, and risk; boards hide severity and headcount math behind columns.
- **Rejected — Badges / streaks / celebratory closure:** task completion and pair closure are their own reward; no gamified toasts.
- **Rejected — AI-suggested next task / assignee:** the platform surfaces entitled state, never tells the viewer whom to pick; assignment is a human decision with a written reason where mandated.
- **Rejected — Client-side masking as access control:** stripping fields in the frontend is never a substitute for the server-side section grant; a `—` cell must be absent from the API response.
- **Rejected — Period-over-period comparison for feedback:** feedback bodies are free text with nothing structural to compare; records list chronologically and filter by period only.

## Key Flows

SPEC glossary terms used verbatim (Shared Link, ProjectAssignment, Form Campaign, Feedback record, Mentorship pair, IDP, Employment status, Departure cascade, Access switch, Colleague).

### Flow 1 — UM Olena fulfils a resourcing request with an auto Shared Link (weekday morning)

1. Olena (Unit Manager) opens her UM Dashboard lens; a resourcing request routed to her department awaits proposals.
2. She opens the Resourcing request detail: vacancy details, duration, workload, headcount 1 of 1 remaining, required department matching hers.
3. She proposes an internal employee from her access scope; submitting the proposal automatically generates a Shared Link naming the reviewing DM as sole recipient, scoped to S1/S4/S11/S12/S5 (CV+certs only) with S6 optionally enabled, bound to the request's lifetime.
4. She optionally adds an external candidate via PeopleForce candidate ID + link (no full data pull).
5. **Climax:** the proposal lands with its access-safe evidence attached — the DM can evaluate the internal candidate through the auto Shared Link without Olena ever widening standing access, and the comp band stays visible only to author, routed UM, and reviewing DM.
6. Olena tracks proposed → approved/rejected with feedback on the request and on the candidate's S15 Request history.

Failure: Olena's access to the candidate narrows before decision → the Shared Link re-clamps on the next view and affected sections stop rendering without a separate revoke step; the proposal itself remains for the DM to decide.

### Flow 2 — DM Dmytro scopes the dashboard and decides, with a written rejection path (same week)

1. Dmytro (Delivery Manager) opens the DM Dashboard lens on all-projects aggregate.
2. He uses the project selector to scope the whole page — counters, tables, Unassigned bucket — to a single project; clearing it restores the aggregate.
3. He opens the Resourcing request detail Olena fulfilled; reviews the internal candidate via the auto Shared Link and the external candidate via the stored PeopleForce link.
4. He approves the internal candidate with a short note — one headcount slot fills; the decision records in request history and S15.
5. He rejects the external candidate; the write is refused until he supplies a written reason, then lands.
6. **Climax:** the scoped dashboard now shows the slot filled and remaining headcount zero — further approvals block until he raises headcount or closes; only his explicit permission-gated close ends the request, successfully or unsuccessfully, never auto-close.
7. Later, discovering ineligibility, he reverses the approval to rejected with a written reason; the slot frees.

Failure: remaining headcount is zero and Dmytro tries to approve anyway → blocked with "Raise headcount or close the request."; a headcount edit below the filled count is rejected outright. If the awaiting DM had departed, the request auto-routes to a live DM on the same project, else flags to a Full-access holder — never silently orphaned.

### Flow 3 — PP Marta builds a campaign audience from a saved view and tracks completion (mid-sprint)

1. Marta (People Partner) opens Campaigns list and creates a draft Form Campaign (title, description, purpose, external form link, due date).
2. She builds the audience via the filter engine, starting from a saved Directory view, with individual add/remove; recipients outside her access resolve through the entitled subset (campaign-sender exception applies only post-activation, for that campaign only).
3. She activates: the audience atomically freezes and the platform generates one Action Item per recipient (source `campaign`).
4. She tracks per-person completion — completed / not completed / overdue — on the Campaign detail; the system never reads the external form itself, trusting only each recipient's own Action Item completion.
5. **Climax:** the frozen audience holds while people join, move, and leave — activation-time membership is the truth, and every distributed form traces back to exactly this campaign; Marta sees each recipient's name + that campaign's Action Item status and nothing else beyond her standing access.

Failure: activation partially fails → all-or-nothing rollback, no partial audience state; campaign stays draft with an error toast and retry.

### Flow 4 — UM Olena ends a mentorship requiring a closing note, writing a timeline event (end of month)

1. Olena opens the Mentorship hub; the pair she oversees (within her access scope) is ready to end.
2. She initiates end-of-pair; the write is refused until she supplies the mandatory closure note stored on the Mentorship pair record.
3. She submits the note; the pair ends, the mentor's status reverts per the self-flag rule, and a Career Timeline event records the pair end on each member's S9.
4. The closure note is readable by the reporting line, project line, and PP only — never by mentor, mentee, or Colleague viewers; the pool already excluded S13 data throughout.
5. **Climax:** the ended pair with its note and dates stays visible in the hub and on S13/S9 — the ending is documented institutional memory, not a silent unlink.

Failure: a concurrent closer already ended the pair → Olena's write is rejected as a conflict (not silently overwritten); departure-triggered auto-close instead applies idempotently with a system-generated note bypassing the gate.

### Flow 5 — PP Daniela triages risk from the dashboard drill-through (Friday review)

1. Daniela (People Partner) opens the Risk dashboard: severity-sorted counts with `medium`, `high`, and `leaver` emphasised, scoped to the people she holds Manager or People Partner access over.
2. She drills from the `high` count into the filtered table — severity descending, then date — reading the trend arrow that renders only on an actual level change.
3. She filters by department, then opens a row's Employee Profile at S6 to read level, description, details, date, and full history.
4. **Climax:** the count, the sorted table, and the profile section tell one continuous story — Daniela moves from portfolio signal to a single person's risk record without re-filtering or losing scope, and every surface she touched respected her access without ever hinting at records outside it.
5. She records follow-up as an Action Item from the profile; the dashboard's open-item counters pick it up.

Failure: the risk record changes under her (newer record lands) → the table row re-sorts on refresh with a toast "Updated — refreshed view."; her open profile section shows the current level with its trend intact.

### Flow 6 — HR Admin Priya creates a functional role without widening data access (Tuesday afternoon)

1. Priya (HR Admin) opens Admin → Functional roles and creates a new role (e.g. "Security-awareness sender"), granting only *create form campaigns* from the granular permission set — no deploy, no schema change.
2. She assigns holders through the UI; the permission takes effect immediately for everyone holding it.
3. A holder opens Campaigns and builds an audience — the audience resolves strictly within their existing access scope (Colleague view unless they hold a Manager or People Partner relationship); the new role widened features, never data.
4. **Climax:** by Wednesday the IT department runs its own campaign through the new role while seeing exactly the audience their relationships earn them — extensibility delivered as data, with the access matrix untouched and the Full-access grant nowhere in the picture.

Failure: Priya removes the permission mid-campaign-draft → holders lose the create action immediately with "Permission changed — draft kept, creation disabled."; in-flight drafts never send.

---

## Open items

- Saved-view shape (filter-only vs static-list bench variant) and departed-creator ownership (ownerless vs archive) — pending PO confirm.
- Launch defaults for five functional-role permissions (*manage custom fields*, *assign and end mentorships*, *approve or reject proposed candidates*, *edit the career timeline*, *create feedback*) — pending PO confirm (D11).
- Photo/certificate limits, IDP reopening, assessor field shape, timeline deletion mode, post-access-loss author cancel — Appendix defaults assumed, pending confirm.
- Offline write posture (queue vs reject-with-retry) assumed; confirm against frontend NFR pass.

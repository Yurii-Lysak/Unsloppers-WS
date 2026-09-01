---
stepsCompleted: [step-01, step-02, step-03]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - _bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md
  - _bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md
  - docs/backlog_review_draft.md
  - docs/PRD_parallel_delivery_plan.md (§7 only — Epic 10/11 full specs)
---

# People Management Platform - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the People Management Platform, decomposing the requirements from the PRD, UX design (DESIGN.md/EXPERIENCE.md), and Architecture (ARCHITECTURE-SPINE.md) into implementable stories. Per user direction, the epic structure preserves the 13 capabilities already established across SPEC.md, the PRD, and the architecture spine's Capability → Architecture Map (each epic = one backend module) — this step does not re-derive a new cut. Story-level acceptance criteria draw on the pre-existing, previously-drafted `docs/backlog_review_draft.md` (61 stories) and `PRD_parallel_delivery_plan.md` §7 (Epic 10/11), reconciled against the PRD's 55 FRs and the architecture's ratified contracts (C1–C8) rather than redrafted from scratch.

## Requirements Inventory

### Functional Requirements

FR-1: Resolve access role per subject (Self/Manager line/PP/Colleague/Shared Link/HR Admin), re-resolved every request
FR-2: Assemble profile by resolved section access (S1-S16), absent not hidden for ungranted sections
FR-3: Enforce the Colleague whitelist server-side (S1, S10, S11 only)
FR-4: Management notes (S7) with dual visibility flags (visible-for-employee, visible-for-PM)
FR-5: Define and assign functional roles at runtime via UI, no deploy
FR-6: Generate and manage a shareable profile link (section-scoped, expiring, revocable, logged)
FR-7: Filter, sort, and column any field (built-in/derived/custom) on All Employees
FR-8: Define custom fields at runtime (text/number/date/select/boolean), no schema migration
FR-9: Inline-edit list cells subject to the access matrix
FR-10: Save and share filter/column views as named tabs
FR-11: Export the current view to .xlsx, scoped to visible columns
FR-12: Colleague mode of the All Employees list (whitelist columns only)
FR-13: View own employment summary (Self-Service)
FR-14: Edit own contact information (S2/S3, address)
FR-15: Upload own photo and certificates
FR-16: View own timeline/leaves/projects/CDS/mentorship/feedback/notes/action items; mark IDP and action items complete
FR-17: Create an action item manually (manager/PP/permitted role)
FR-18: Auto-generate action items from campaign activation
FR-19: Complete or cancel an action item
FR-20: Derive and display overdue state for action items, consistent everywhere
FR-21: Record a risk (5-level severity) with retained history
FR-22: Show risk trend vs. previous record
FR-23: Risk Dashboard — severity-sorted counts, drill-through, filters
FR-24: Create a resourcing request (optionally unattached to a project)
FR-25: Fulfil a request with internal or external (PeopleForce) candidates
FR-26: Approve or reject a proposed candidate with a required reason
FR-27: Retain resourcing request history (on request and candidate profile, S15)
FR-28: Auto-generate career timeline events on tracked field changes
FR-29: Manually add/edit/delete timeline events (PP/UM), soft-delete
FR-30: Resolve manual-vs-system timeline conflicts (manual wins, system write skipped+logged)
FR-31: Resolve the current skills-matrix link via a maintained dictionary
FR-32: Maintain the CDS assessment log
FR-33: Maintain and self-complete the IDP
FR-34: Filter All Employees by assessment recency (incl. "never assessed") and open-IDP
FR-35: Self-flag open to mentoring
FR-36: Assign a mentor-mentee pair (manager/PP)
FR-37: End a mentorship pair with mandatory closing feedback
FR-38: Auto-transition mentor status per the self-flag rule
FR-39: View all mentor-mentee pairs
FR-40: Create a draft form campaign
FR-41: Build the campaign audience via filter engine or saved view, with manual add/remove
FR-42: Activate a campaign — atomic audience freeze + action-item generation
FR-43: Track per-person campaign completion
FR-44: Record feedback with a visibility flag (management-only default)
FR-45: View feedback chronologically with period comparison
FR-46: Request feedback from named colleagues via a targeted campaign
FR-47: Shared dashboard engine (counters, scoped table, own action items)
FR-48: Unit Manager dashboard
FR-49: Delivery Manager dashboard with project selector
FR-50: Project Manager dashboard
FR-51: People Partner dashboard (no resourcing block)
FR-52: Integrate timetracker Leaves API
FR-53: Integrate timetracker Projects & People API (feeds access model)
FR-54: Integrate PeopleForce for candidates/vacancies, with outbound-link fallback
FR-55: Resolve cross-system identity via explicit mapping table

### NonFunctional Requirements

NFR-1: Access-matrix test coverage — every audience x relationship-path x section combination, every negative case, is covered by a passing test (SPEC SM-1; architecture AD-13 names the parameterized-suite mechanism)
NFR-2: All Employees list responds within 2s at 500+ records under arbitrary filters, including permission resolution
NFR-3: WCAG 2.1 AA across List, Profile, and Dashboard surfaces
NFR-4: External integration failures (timetracker, PeopleForce) degrade gracefully, never take the app down
NFR-5: Non-production environments use pseudonymized data only — never real personal data in agent contexts, logs, screenshots, or the repository
NFR-6: Zero cross-developer blocking during delivery — cross-feature dependencies resolve via frozen interface contracts (C1-C8), stubs, sanctioned fallbacks, or same-developer sequencing, never a wait
NFR-7: The intelligent repository (specs, decisions, architecture, this epics doc) stays current with shipped behavior
NFR-8: Access is evaluated server-side, per section, per request — no client-side caching, no session-stable assumption; any future cache invalidates on a generation-bump only (never bare TTL)

### Additional Requirements

- Architecture is brownfield-adjacent, not a fresh starter: `services/backend` (NestJS 11 + Prisma 7 + Postgres 18) and `services/frontend` (React 19 + shadcn/ui + Tailwind v4) already have real, committed scaffolds and conventions to ratify — Epic 1's first stories are Wave-0 substrate work, not "apply a starter template."
- Wave-0 substrate (architecture spine AD-1/AD-2/AD-3): stand up `contracts` (C1-C8 abstract tokens) and `registry` (DiscoveryService-based Provider Registry) modules before any feature module depends on them.
- `dependency-cruiser` CI rule enforcing AD-1's module-boundary rule from commit one.
- Auth module (JWT local auth, AD-9/D9) — net-new, no source document addressed this before architecture.
- Four temporal history tables (GradeHistory/PositionHistory/DepartmentHistory/EmploymentTypeHistory) + the Prisma Client Extension coupling them to the Timeline Event Writer (AD-7).
- Deployment: single containerized deploy, docker-compose across backend+frontend+Postgres (AD-12/D10).
- openapi-typescript-generated frontend types from the backend's Swagger surface, CI-diffed (AD-10).
- Parameterized access-matrix test scaffold driven off `access-model.md`'s table (AD-13), stood up before the first section provider ships.
- Pseudonymized seed-data tool (Story 1.16, **done**) — **24-account** manifest for dev/demo; see `bootcamp-scope-overrides.md`. **500+-record scale** is a separate perf target (Story 3.7 / NFR-2), not the seed manifest size.

### UX Design Requirements

UX-DR1: Risk severity color scale (5 steps, light+dark) and success token — new DESIGN.md tokens, not in the existing shadcn palette
UX-DR2: Risk Badge + Trend Arrow component — identical instance on Profile, All Employees, Risk Dashboard
UX-DR3: Overdue Indicator component — shared by Action Item Row and Campaign Completion Table
UX-DR4: Status Badge (Positive) component — completed action items, IDPs, ended mentorship pairs, completed campaign rows
UX-DR5: Access Scope Chip — shows resolved role on every profile view of someone else; entry point to Shared Link Manager
UX-DR6: Section Gate component — the one documented PM/S7 exception only
UX-DR7: Profile Section Card — RW vs. R-only visual distinction (edit affordance present/absent)
UX-DR8: Data Table (compact density) for All Employees — sortable, inline-editable, saved-view tabs, column filters, export
UX-DR9: Filter/Audience Builder — shared component across All Employees, Campaign audience, Risk Dashboard filters
UX-DR10: Career Timeline component — system/manual source tag, "system update skipped" indicator
UX-DR11: Feedback Panel — chronological + side-by-side period comparison, explicit empty-period columns
UX-DR12: Shared Link Manager — section picker (S3/S7/S13 structurally excluded), expiry, active-link revocation
UX-DR13: Visibility Flag Toggle — dual-flag (S7) and single-flag (S8) variants
UX-DR14: 11-surface IA exactly as EXPERIENCE.md's table (Dashboard, All Employees, Employee Profile, My Profile, Resourcing, Risk Dashboard, Campaigns, Mentorship Hub, Admin Roles, Admin Fields, Shared Link view)
UX-DR15: Responsive behavior — table-to-card-list collapse on All Employees at `sm`, profile accordion collapse, sidebar Sheet (all per existing `LayoutContext` behavior)
UX-DR16: Accessibility floor — risk severity never color-only, resolved access scope announced to screen readers, full keyboard operability on inline edit and required-reason dialogs
UX-DR17: i18n discipline — every new string a translation key (existing `react-i18next` convention), no hardcoded copy

### FR Coverage Map

FR-1 – FR-6: Epic 1 - Access Control, Roles & Employee Profile
FR-7 – FR-12: Epic 3 - All Employees Directory & Custom Fields
FR-13 – FR-16: Epic 2 - Self-Service
FR-17 – FR-20: Epic 4 - Action Items and Tasks
FR-21 – FR-23: Epic 5 - Risks and Risk Dashboard
FR-24 – FR-27: Epic 6 - Resourcing
FR-28 – FR-30: Epic 7 - Career Timeline
FR-31 – FR-34: Epic 8 - CDS: Career Development System
FR-35 – FR-39: Epic 9 - Mentorship Hub
FR-40 – FR-43: Epic 10 - Forms & Survey Campaigns
FR-44 – FR-46: Epic 11 - Feedback
FR-47 – FR-51: Epic 12 - Dashboards
FR-52 – FR-55: Epic 13 - External Integrations: Timetracker & PeopleForce

NFR-1 (access-matrix tests), NFR-8 (per-request evaluation): cross-cutting, primarily Epic 1, verified per-epic via each epic's own SectionProvider tests (architecture AD-13)
NFR-2 (2s/500-record perf): Epic 3, Story 3.7
NFR-3 (WCAG 2.1 AA), UX-DR15, UX-DR16: Epic 12, Story 12.6 (deliberately scheduled last, after the surfaces it audits exist)
NFR-4 (graceful integration degradation): Epic 13
NFR-5 (pseudonymized data), NFR-6 (zero-blocking delivery), NFR-7 (repo currency): process-level, Epic 1's Wave-0 stories + delivery practice, not a single epic's output
Additional Requirements (auth, contracts/registry substrate, CI rule, temporal tables, deployment, generated types, test scaffold, seed-data tool): Epic 1's opening stories (see below) — this is where `backlog_review_draft.md`'s own Epic 1 numbering already starts (access-resolution substrate), so folding auth/infra in here rather than inventing a separate non-user-value "Epic 0" keeps the existing story numbering intact.
UX-DR1 – UX-DR13 (components/tokens): distributed to the epic owning each — e.g. UX-DR2 (Risk Badge) → Epic 5, UX-DR11 (Feedback Panel) → Epic 11, UX-DR12 (Shared Link Manager) → Epic 1
UX-DR14 (IA), UX-DR15 (responsive), UX-DR17 (i18n): cross-cutting, enforced per-epic as each surface ships

## Epic List

### Epic 1: Access Control, Roles & Employee Profile
Every profile-scoped read/write in the system passes through this epic's access resolution — nothing else can ship correctly before it exists, and it is itself the epic that makes "who sees what" a first-class, testable guarantee rather than an assumption. Covers: viewer access-role resolution across the reporting/project/PP graphs; section-by-section profile assembly (S1–S16); the Colleague whitelist; management notes with dual visibility flags; HR-Admin-defined functional roles and permissions with zero-deploy extensibility; shareable, expiring, revocable profile links. Its opening stories also stand up the Wave-0 substrate every other epic depends on: the `contracts` and `registry` modules, the CI dependency-boundary rule, JWT auth, the four temporal history tables, single-container deployment, and the parameterized access-matrix test scaffold.
**FRs covered:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-6 (+ Additional Requirements: auth, Wave-0 substrate)

### Epic 2: Self-Service
An employee's own view of themselves — read access to most of their profile, write access to a defined subset, with risk level and unflagged notes permanently invisible regardless of role.
**FRs covered:** FR-13, FR-14, FR-15, FR-16

### Epic 3: All Employees Directory & Custom Fields
One list serving every audience, where any field — built-in, derived, or defined at runtime by HR Admin — is filterable, sortable, and a column, with inline edit, saved views, export, and a Colleague-scoped mode.
**FRs covered:** FR-7, FR-8, FR-9, FR-10, FR-11, FR-12

### Epic 4: Action Items and Tasks
The platform's single task entity — created manually or by campaign activation, completed by the assignee, cancelled by the author, overdue everywhere consistently.
**FRs covered:** FR-17, FR-18, FR-19, FR-20

### Epic 5: Risks and Risk Dashboard
A retained-history, five-level risk record per employee with trend, plus a dedicated dashboard — scoped to Manager/PP, never visible to the employee.
**FRs covered:** FR-21, FR-22, FR-23

### Epic 6: Resourcing
Request → fulfilment → approval for staffing, spanning internal candidates and external PeopleForce candidates, with full history retained.
**FRs covered:** FR-24, FR-25, FR-26, FR-27

### Epic 7: Career Timeline
A system-generated event log for tracked changes, with manual override for backfill/correction — never a separately maintained record, never silently overwritten.
**FRs covered:** FR-28, FR-29, FR-30

### Epic 8: CDS: Career Development System
A registry and hub — not an assessment engine — linking to an externally maintained skills matrix, logging assessments, and tracking IDP completion.
**FRs covered:** FR-31, FR-32, FR-33, FR-34

### Epic 9: Mentorship Hub
Self-flagged willingness, manager/PP-driven pairing, and mandatory closing feedback on every ended pair.
**FRs covered:** FR-35, FR-36, FR-37, FR-38, FR-39

### Epic 10: Forms & Survey Campaigns
A campaign that targets a frozen audience with a link to an externally hosted form, tracked entirely through action-item completion.
**FRs covered:** FR-40, FR-41, FR-42, FR-43

### Epic 11: Feedback
Manager/PP-authored feedback with a visibility flag, viewable over time, plus a named-colleague feedback-request flow built on Form Campaigns.
**FRs covered:** FR-44, FR-45, FR-46

### Epic 12: Dashboards
One shared engine, four role-scoped views — grouping differs (by person vs. by project), the underlying components don't.
**FRs covered:** FR-47, FR-48, FR-49, FR-50, FR-51

### Epic 13: External Integrations: Timetracker & PeopleForce
Two integrations, one of which (timetracker projects/people) is load-bearing for the access model itself, not just for display.
**FRs covered:** FR-52, FR-53, FR-54, FR-55

---

## Epic 1: Access Control, Roles & Employee Profile

Every profile-scoped read/write in the system passes through this epic's access resolution. Stories 1.1–1.14 are reconciled from `docs/backlog_review_draft.md` (already reviewed, AC/GWT preserved, condensed to template shape); 1.15–1.20 cover Wave-0 substrate the architecture spine requires that the original backlog draft either drafted as cross-cutting NFR stories or never anticipated at all.

### Story 1.1: Derive Manager Access from Reporting Hierarchy

As the system,
I want to compute a person's Manager access role via the transitive closure of "reports to" relationships,
So that anyone above someone in the reporting chain is recognized as their manager without a manual grant.

**FRs:** FR-1. **Architecture:** AD-1 (owned by `access`), interface contract C1.

**Acceptance Criteria:**

**Given** employee B reports to Manager M, and M reports to Director D
**When** D's access with respect to B is resolved
**Then** D holds Manager access with respect to B, without any explicit grant, re-evaluated on every request — never cached across a session

**Given** employee A and employee B share no reporting relationship, direct or transitive
**When** A's access with respect to B is resolved
**Then** A does not hold Manager access with respect to B via this path
**And** the closure rejects cycles and never grants Self access to Self

### Story 1.2: Extend Manager Access via Project Assignment

As the system,
I want Manager access to also derive from "assigned to a project managed by," backed by an internally-owned `ProjectAssignment` record,
So that PMs/DMs see the people on their projects, seeded internally until Epic 13 supplies it live.

**FRs:** FR-1. **Architecture:** interface contract C3 — ratified shape now includes `confirmed`/`confirmedAt` (AD-8); this story owns the seed/internal-write path, Epic 13 becomes the real writer.

**Acceptance Criteria:**

**Given** employee B is assigned to Project X, whose PM is P and whose DM is D
**When** D's access with respect to B is resolved
**Then** D holds Manager access with respect to B at the same strength as P's, and this leg is unioned with Story 1.1's reports-to leg

**Given** PM P holds Manager access to B solely via B's assignment to Project X
**When** B's assignment ends and P's access to B is resolved again
**Then** P no longer holds Manager access via that project, with no grace period
**And** `ProjectAssignment` is queryable domain data independent of any live external integration

### Story 1.3: Assign People Partner Relationships

As an HR Admin,
I want to assign a People Partner to an employee,
So that the assigned PP (and their HR line) holds People Partner access for that employee.

**FRs:** FR-1.

**Acceptance Criteria:**

**Given** HR Admin assigns People Partner X to employee B
**When** X's access with respect to B is resolved
**Then** X holds PP access to B, and X's HR line above them also resolves as holding PP access to B
**And** the assignment is surfaced in B's profile header (Story 1.7)

**Given** People Partner X currently holds PP access to employee B
**When** HR Admin reassigns B's PP to Y and X's access to B is resolved again
**Then** X no longer holds PP access to B, and Y does, with no delay

### Story 1.4: Define Functional Roles and Permissions via UI

As an HR Admin,
I want to create a functional role, name it, and grant it a set of feature permissions through the UI,
So that a new part of the org can use platform features without a code deploy.

**FRs:** FR-5. **Architecture:** interface contract C8 `PermissionChecker` (AD-2) is the single enforcement point this data model feeds — no module queries the permission tables directly.

**Acceptance Criteria:**

**Given** HR Admin is on the functional roles admin screen
**When** they create a role named "Security Champion" and grant it only "create form campaigns"
**Then** the role is persisted as data and anyone later assigned it can create form campaigns with no code deploy
**And** the minimum grantable set (create form campaigns, create action items, create/edit risks, create resourcing requests, fulfil resourcing requests, assign mentors, maintain CDS records, manage custom fields, view a given dashboard) is independently grantable

**Given** the "Security Champion" role grants "create form campaigns" and Person X holds it
**When** HR Admin removes that permission from the role
**Then** X's next attempt is denied immediately, without logout or delay
**And** the role/permission schema never stores or implies a data-access grant — access roles stay derived from relationships only

### Story 1.5: Assign Functional Roles to People

As an HR Admin,
I want to assign an existing functional role to a person through the UI,
So that they gain the role's granted features without gaining new data access.

**FRs:** FR-5. **Architecture:** enforced via C8 `PermissionChecker`, scoped by C1 `AccessResolver` — a functional role never widens what C1 already resolved.

**Acceptance Criteria:**

**Given** Person X (IT) is assigned the "Security Champion" role, which grants "create form campaigns," and X holds no Manager/PP relationship over anyone
**When** X builds a campaign audience via the All Employees filter engine
**Then** X can only see/filter Colleague-visible fields (S1, S10, S11) for candidate recipients

**Given** Person X is assigned a role granting "create/edit risks"
**When** X attempts to view or edit S6 for a person X holds no Manager/PP access over
**Then** the action is denied — the functional role grants the feature, never the underlying data access
**And** a person may hold multiple functional roles simultaneously, each granting features independently

### Story 1.6: Assemble Employee Profile by Section Access

As any authenticated user,
I want to open a profile and see only the sections I'm entitled to,
So that the profile always respects the access matrix.

**FRs:** FR-2. **Architecture:** this is `ProfileAssemblerService` (AD-3) — resolves C1 once, then calls a `SectionProvider` only for granted sections via the `registry` module (AD-1). C1's `sections` map converts to a plain `Record` before crossing any HTTP boundary (AD-2 wire-safety rule).

**Acceptance Criteria:**

**Given** Viewer V holds Manager access with respect to profile owner B
**When** V requests B's profile
**Then** the response includes every section at the Manager-line R/RW level `access-model.md` specifies, with none silently missing

**Given** Viewer V holds no Manager, PP, or Self relationship to profile owner B
**When** V requests B's profile via the API directly
**Then** the response contains only S1, S10 (incl. leave type), and S11 (project name only) keys — no keys at all for any other section
**And** field-level exceptions within a section (e.g. S1's photo RW for Self) are honored, not just section-level R/RW/none

### Story 1.7: Profile Header Shows Manager, PP and Mentor

As any viewer with profile access,
I want to see the person's manager, people partner and mentor at the top of their profile,
So that I understand their reporting context at a glance.

**FRs:** FR-2. **Architecture/Decision:** D5 (confirmed) — the mentor field follows 4.11's "visible to manager line and PP" rule, overriding the general Colleague-R grant on identity-card content; `access-model.md` rule 6 and the Access Scope Chip (UX-DR5) both encode this.

**Acceptance Criteria:**

**Given** Viewer V holds Manager access with respect to profile owner B, and B has an assigned manager, PP and mentor
**When** V opens B's profile
**Then** the header shows B's current manager, PP, and mentor, each linking through only where V has access to the target profile

**Given** Viewer V is a Colleague of profile owner B, and B has an assigned mentor
**When** V opens B's profile
**Then** the header shows manager and PP (per S1) but withholds the mentor field, per D5

### Story 1.8: Enforce the Colleague Whitelist Everywhere

As a Colleague of another employee,
I want every surface — profile, API, export, search — to never reveal sections outside the whitelist,
So that access can't leak through any path other than the rendered page.

**FRs:** FR-3. **Architecture:** AD-5 — the whitelist is a fixed section-grant map C1 returns, never a post-hoc filter; enforcement lives at the data-access layer, not the frontend or a serializer.

**Acceptance Criteria:**

**Given** Viewer V is a Colleague of profile owner B
**When** V calls the profile API for B directly, bypassing the UI
**Then** the response contains only S1, S10 (incl. leave type), and S11 (project name only) keys, with no other section data present at any level

**Given** Viewer V is browsing All Employees in Colleague mode
**When** V exports the current view to `.xlsx`
**Then** the exported file's actual columns are limited to the whitelist, even if V attempted to add a non-whitelist column beforehand

### Story 1.9: Management Notes with Visibility Flags

As a UM, DM or PP responsible for a person,
I want to create, read and edit free-form management notes about them, each carrying independent "visible for employee" and "visible for PM" flags defaulting off,
So that sensitive notes stay private unless explicitly shared.

**FRs:** FR-4. **UX:** Visibility Flag Toggle (UX-DR13, dual-flag variant).

**Acceptance Criteria:**

**Given** PP creates a management note about employee B with both flags left off
**When** B views their own profile, and separately B's PM views S7
**Then** neither B nor the PM can see that note, in the UI or via direct API request

**Given** B's PM is not B's UM/DM/PP, and one S7 note about B is flagged "visible for PM" while a second is not
**When** the PM opens B's S7 section
**Then** the PM sees exactly the one flagged note, read-only, and cannot see or edit the unflagged one
**And** UM/DM/PP retain full RW on all notes regardless of flags — the one documented exception to "PM is a Manager for every other section"

### Story 1.10: Per-Field Custom Field Visibility

As an HR Admin,
I want each custom field's visibility (management/employee/colleague) to gate its appearance in profile views, filters and list columns,
So that a value I haven't been granted can't be inferred.

**FRs:** FR-8 (jointly with Epic 3's `FieldRegistry`, C2). **Architecture:** AD-6 — the visibility gate is a generic, field-agnostic check at query time, the same discipline as the typed-value-table rule.

**Acceptance Criteria:**

**Given** HR Admin creates a custom field "Dietary preference" with visibility "employee"
**When** Self views their own profile, and separately a Colleague views the same profile
**Then** Self sees the field, and the Colleague sees no trace of it — not on the profile, in list columns, or in filters

**Given** a "management"-visibility custom field exists and a Colleague is browsing All Employees
**When** the Colleague attempts to construct a filter/sort referencing that field via direct API parameters
**Then** the request is rejected or the field is silently excluded, and the result set never indirectly reveals the value (e.g. via a count change)

### Story 1.11: Generate a Shareable Profile Link

As a manager,
I want to generate a shareable view of an employee's profile for someone without Manager/PP access,
So that, for example, a DM can evaluate an internal candidate they don't yet have access to.

**FRs:** FR-6. **Architecture:** AD-5 — S3/S7/S13 exclusion is backend-validated (the `access` module's link-creation endpoint rejects them outright), never a frontend checkbox omission. **UX:** Shared Link Manager (UX-DR12).

**Acceptance Criteria:**

**Given** a UM holds Manager access to employee B and opens the share-link creation flow
**When** the UM enables S1 and S9 only, leaving S2/S5/S6/S8 at their default
**Then** the link exposes S1 and S9 read-only, and S2/S5/S6/S8 are excluded because they were never explicitly enabled

**Given** a UM is configuring a share link for employee B
**When** the UM (or a direct API call) attempts to include S3, S7, or S13
**Then** the request is rejected server-side — these sections are never offered nor acceptable under any configuration

### Story 1.12: Shared Link Expiry, Logging and Revocation

As a manager who shared a profile link,
I want it to expire, be logged on every access, and be revocable,
So that exposure is time-bound and auditable.

**FRs:** FR-6. **Decision:** default expiry 24h, configurable at creation (`decisions.md` appendix).

**Acceptance Criteria:**

**Given** a share link is created with no custom expiry
**When** 24 hours elapse and someone opens the link
**Then** the link no longer grants access
**And** every access attempt, successful or not, writes a log entry with when and where it originated

**Given** a share link is active and not yet expired
**When** the creating manager revokes it, and someone then attempts to open it
**Then** access is denied and the attempt is logged, with expired/revoked/invalid responses uniform enough not to leak which case applies

### Story 1.13: Cache Access Resolution Safely and Revoke Immediately on Project-Assignment End

As the system,
I want any future access-resolution cache to be invalidated by a relationship-graph generation counter, never a bare TTL,
So that the 2-second performance bar (NFR-2) is achievable without "managerial access is not sticky" ever becoming a stale-grant leak.

**FRs:** NFR-8. **Decision/Architecture:** D1 + AD-4 (settled, not open for team debate as the original draft framed it) — no caching by default in v1; if built, invalidates synchronously on any reports-to/project-assignment/PP-assignment write, including a bare `confirmed`/`confirmedAt` flip (AD-8).

**Acceptance Criteria:**

**Given** the All Employees list is loaded with 500+ records and several arbitrary filters
**When** the viewer's request completes
**Then** response time including full permission resolution is under 2 seconds

**Given** PM P holds Manager access to employee B solely via B's assignment to Project X, and any cache is warm for this pair
**When** B's assignment ends and P immediately re-requests B's profile
**Then** P is denied Manager-level access, with no stale cached grant served

### Story 1.14: Prevent Section Leaks Across All Surfaces for Every Denied Audience

As the platform,
I want a matrix-driven negative-test harness covering every `—` cell and flag-gated case in the access matrix,
So that a leak in any section, for any audience, on any surface is a build-breaking defect, not a discovered-in-production one.

**FRs:** NFR-1. **Architecture:** AD-13 — this harness is the concrete implementation of the parameterized access-matrix test convention, living next to each `SectionProvider` it verifies.

**Acceptance Criteria:**

**Given** Employee E requests their own profile
**When** the response is inspected
**Then** S6 (risks) and S15 (request history) are entirely absent — no keys, no empty placeholders

**Given** a DM opens a shared link created with only S1 and S9 enabled
**When** the DM inspects the rendered page and calls the underlying API directly, attempting to force S3/S7/S13
**Then** those sections are absent from both surfaces under all circumstances
**And** the harness is data-driven off `access-model.md`'s matrix itself, so a future matrix change automatically extends or flags outdated coverage

### Story 1.15: Access-Control Test Suite Architecture

As the delivery team,
I want a documented, CI-enforced test strategy for access control — by audience, by relationship path, and by section/feature — extending beyond the profile matrix into Risks, Resourcing, and Action Items,
So that "access-control correctness" (the platform's primary quality attribute) has a repeatable strategy, not ad hoc per-feature coverage.

**FRs:** NFR-1. **Architecture:** AD-13.

**Acceptance Criteria:**

**Given** the automated access-control suite runs in CI
**When** a code change accidentally grants Self read access to S6
**Then** the corresponding negative test fails and blocks the merge

**Given** the test strategy is applied to Resourcing (Epic 6)
**When** a PM without access to a candidate's profile attempts to view it directly instead of through profile-sharing
**Then** an automated test asserts this is rejected with the same rigor as the profile-matrix tests

### Story 1.16: Pseudonymized Seed Data Tool

As the delivery team,
I want a seed tool loading a fixed bootcamp test population with substituted identities,
So that every track has realistic dev data from day one and no real personal data enters an agent context, log, or the repository.

**FRs:** NFR-5. **Shipped:** 24 accounts from `docs/bootcamp-seed-accounts-source.csv` → `bootcamp-identities.json`. **Not shipped:** 500+ manifest, TimeTracker Accounting as seed source.

**Acceptance Criteria (shipped):**

**Given** a new non-production environment is provisioned
**When** `npm run db:seed` runs (no TimeTracker keys required)
**Then** the database contains **24** employee records from the bundled manifest with pseudonymized names/emails and synthetic history rows

**Given** a developer uses an AI coding agent against a seeded environment
**When** the agent inspects employee data
**Then** everything it encounters is test/pseudonymized data from the delivered list — not real PII

**Deferred to Story 3.7:** All Employees list performance validation at **500+** records (NFR-2) — separate from seed manifest size.

### Story 1.17: Intelligent Repository, Process Setup, and Single-Container Deployment

As the delivery team,
I want the intelligent repository, BMad configuration, and deployment topology stood up before feature work starts,
So that the foundation-phase requirement is real and the platform is deployable and demonstrable from Wave 0 onward.

**FRs:** none directly — the only real PRD source requirement is "deployed and demonstrable, not on a laptop" (`project-requirements-v2.md`, bootcamp grading criteria); the original "FRs: NFR-7" citation here was a miscite (NFR-7 is repo/doc currency, not deployment mechanism). **Architecture:** AD-12, amended 2026-09-01 — managed-PaaS deploy (frontend on Vercel, backend on Render, Postgres on Neon), superseding the original single docker-compose stack; still one environment, no staging tier.

**Acceptance Criteria:**

**Given** the foundation phase starts
**When** the team sets up their workspace
**Then** the intelligent repository exists with specs, decisions, architecture, and agent rules/skills, BMad is configured, and hard sequential dependencies across epics/stories are identified and documented rather than discovered mid-sprint

**Given** the architecture's deployment target (AD-12, amended 2026-09-01)
**When** the team stands up their first deploy
**Then** frontend runs on Vercel and backend runs on Render, both auto-deploying on push to `main`, against a managed Postgres instance on Neon, satisfying "deployed and demonstrable, not on a laptop" as one environment with no staging tier — see `docs/deployment.md`

### Story 1.18: Authentication

As any user,
I want to log in with a session that persists across requests,
So that every controller can identify me before any access decision is made.

**FRs:** none directly (cross-cutting; no source document addressed auth before architecture coaching). **Decision/Architecture:** D9 + AD-9 — simple local JWT auth, decoupled from access-role resolution; C7 `CurrentUserProvider` (AD-2) is the only sanctioned way another module learns the current user.

**Acceptance Criteria:**

**Given** a registered user submits valid credentials
**When** they log in
**Then** a JWT session is issued, and every subsequent authenticated request resolves a `userId` via C7 without any controller importing the `auth` module directly

**Given** an unauthenticated request hits any protected endpoint, including Shared Link consumption
**When** the request is evaluated
**Then** it is rejected — every surface requires a session, per `EXPERIENCE.md`'s Foundation

### Story 1.19: Backend Substrate — Contracts and Provider Registry Modules

As the delivery team,
I want the `contracts` module (C1–C8 abstract tokens) and the `registry` module (DiscoveryService-based Provider Registry) stood up before any feature module depends on them,
So that Wave-1 tracks can build against frozen interfaces and stub providers from day one, with zero cross-developer blocking.

**FRs:** NFR-6. **Architecture:** AD-1, AD-2, AD-3.

**Acceptance Criteria:**

**Given** the `contracts` module is stood up
**When** any feature module is scaffolded
**Then** C1–C8 exist as abstract classes/injection tokens with zero business logic or Prisma imports, each with a Wave-0 stub provider available

**Given** the `registry` module is stood up
**When** a class is decorated `@RegisterProvider(family, id)`
**Then** it is discovered and indexed at bootstrap; a registration collision fails at bootstrap, and a missing registration for a granted section/field/summary surfaces as an explicit "unavailable" state, never a silent omission
**And** the `dependency-cruiser` CI rule forbidding direct feature-to-feature imports is wired and enforced on every PR

### Story 1.20: Temporal Employment History Tables and Timeline Coupling

As the system,
I want grade/position/department/employment-type changes stored as effective-dated history tables, structurally coupled to the Career Timeline,
So that "current value" is never a denormalized field that can drift, and every tracked change is auditable without a bolted-on log.

**FRs:** FR-2 (S4 Employment). **Architecture:** AD-7 — one table per dimension, current = row with `effectiveTo IS NULL`; a Prisma Client Extension fires `TimelineEventWriter` (C4, stubbed until Epic 7 lands the real implementation) automatically on every write.

**Acceptance Criteria:**

**Given** an employee's grade changes
**When** the write lands in `GradeHistory`
**Then** the previous row's `effectiveTo` is set and a new row is created with `effectiveTo IS NULL`, and the Prisma extension calls C4 automatically — no service method writes to a history table through any other path

**Given** a system-sourced write would overwrite a manual timeline correction in the same effective window
**When** the extension detects the conflict
**Then** the history write is suppressed and C4's `markSystemWriteSkipped(manualEventId, skippedAt)` attaches skip metadata to the existing manual entry — never a silent no-op and never a separate timeline row

---

## Epic 2: Self-Service

An employee's own view of themselves — read access to most of their profile, write access to a defined subset, with S6 (risk) and unflagged S7 notes permanently invisible regardless of role. All 5 stories reconciled from `docs/backlog_review_draft.md`.

### Story 2.1: View Own Employment Summary

As an employee,
I want to see my own grade, position, seniority, employment type and English level,
So that I have visibility into my own employment record.

**FRs:** FR-13.

**Acceptance Criteria:**

**Given** I open my own profile
**When** the S4 Employment section renders
**Then** I see grade, position, seniority, employment type, English level, probation status, and contract type as read-only text — S4 is `R` for Self even though I may hold `RW` on it for other people as a manager/PP elsewhere

**Given** a field in S4 has no value set
**When** the section renders
**Then** it displays a clear empty state rather than omitting the whole section, and no write endpoint for S4 is exposed to me

### Story 2.2: Edit Own Personal and Emergency Contacts

As an employee,
I want to view and edit my personal contacts, residential address, place of stay and emergency contacts myself,
So that I don't have to ask HR for routine updates.

**FRs:** FR-14.

**Acceptance Criteria:**

**Given** I edit my personal phone number and save
**When** the change is persisted
**Then** S2 reflects it immediately for me (RW), read-only for my manager line, and RW for my PP

**Given** a colleague with no manager/PP relationship to me requests my profile via any API surface
**When** the response is inspected
**Then** S2 and S3 are entirely absent — and S3 can never be enabled on a shared link under any configuration (AD-5)

### Story 2.3: Upload Photo and Certificates

As an employee,
I want to upload my own profile photo and certificates,
So that my profile stays current.

**FRs:** FR-15.

**Acceptance Criteria:**

**Given** I upload a new photo
**When** it saves
**Then** it replaces my prior photo immediately, visible wherever S1 is rendered for any entitled audience — this is the one RW exception on an otherwise-R-for-Self section

**Given** I attempt to upload or edit a non-certificate S5 document type (contract, CV, etc.), or edit any S1 field other than photo
**When** I submit the request
**Then** it is rejected — my write access is scoped to photo (S1) and certificate upload only (S5)

### Story 2.4: View Own Timeline, Leaves, Projects, CDS and Mentorship

As an employee,
I want to see my own career timeline, leave balances (with a timetracker link), current projects, and CDS section, and manage my mentorship status,
So that I can track my own progress.

**FRs:** FR-16 (part 1). **Architecture:** S10 display degrades per AD-8's fail-soft rule if the timetracker sync lags.

**Acceptance Criteria:**

**Given** I open self-service
**When** each section loads
**Then** I see my timeline (S9), leaves with a timetracker link (S10), projects (S11), and CDS (S12) as read-only, and my open-to-mentoring flag (S13) as the one writable field in that section

**Given** I view S12
**When** I tick "complete" on my own IDP
**Then** a completion date is recorded and displayed, and I cannot edit an assessment conclusion or IDP deadline — only the checkbox is writable for Self

### Story 2.5: View Shared Feedback, Flagged Notes and Own Action Items

As an employee,
I want to see feedback explicitly shared with me, management notes flagged visible-for-employee, and my own action items which I can mark complete,
So that I only see what's meant for me and nothing more.

**FRs:** FR-16 (part 2). **Architecture:** consumes the same S7 (CAP-1) and S8 (CAP-11) Visibility Flag Toggle state (UX-DR13) those epics write.

**Acceptance Criteria:**

**Given** a feedback record about me has "shared with employee" set, and a management note about me has "visible for employee" set, and a second note has no flags
**When** I view S7 and S8
**Then** I see the shared feedback and the flagged note; the unflagged note is entirely absent — record-level filtering, not a section toggle

**Given** I have a risk record at any level
**When** I view any part of my own profile, including dashboards and notifications
**Then** no risk level, trend, or history is visible or inferable anywhere — S6 is `—` for Self with no exception
**And** marking my own action item complete records a completion date, but I cannot edit its title/due date or cancel it

---

## Epic 3: All Employees Directory & Custom Fields

One list serving every audience, differing only in the data each is entitled to. All 6 stories reconciled from `docs/backlog_review_draft.md`.

### Story 3.1: Sortable, Filterable Employee List

As a manager or PP,
I want a tabular list of employees I can sort and filter by any profile field, including derived fields,
So that I can find the people I need.

**FRs:** FR-7. **Architecture:** query layer treats fields (built-in, derived, custom) as uniform queryable metadata (AD-6), never hardcoded columns — this is C2 `FieldRegistry`'s whole reason to exist.

**Acceptance Criteria:**

**Given** the employee list contains people with only a stored join date
**When** a manager filters by "years with company > 3"
**Then** only employees whose computed tenure exceeds 3 years are returned, computed live from join date — no different in shape from filtering a stored field

**Given** a Colleague-level user with no Manager/PP relationship to anyone in the list
**When** they open the field picker
**Then** only whitelist fields are offered, and the list responds within 2 seconds at 500+ records with arbitrary filters (NFR-2)

### Story 3.2: Define Custom Fields at Runtime

As an HR Admin or manager,
I want to define a new custom field and set values on profiles,
So that new data needs don't require a deploy or migration.

**FRs:** FR-8. **Architecture:** AD-6 — one `CustomFieldValue` table, typed columns, `NULL` for unused (never a sentinel), rows created lazily. This story owns C2 `FieldRegistry`'s real implementation.

**Acceptance Criteria:**

**Given** an HR Admin creates a single-select custom field "Preferred office" with visibility "employee"
**When** they save the definition
**Then** it is immediately usable as a filter and column on All Employees, with no deploy, migration, or developer step

**Given** a custom field "Performance flag" is created with visibility "management"
**When** a Colleague-level user opens All Employees or a profile with that field set
**Then** the field is absent as a column/filter option and from any API response for that user, with no way to infer its value via result-count changes

### Story 3.3: Inline Editing on the List

As a user with write access to a field,
I want to edit it inline from the list,
So that I don't need to open the full profile for small updates.

**FRs:** FR-9.

**Acceptance Criteria:**

**Given** a Unit Manager viewing All Employees has write access to "grade" for one of their direct reports
**When** they edit the grade cell inline and save
**Then** the change persists to the same underlying field shown on the full profile, reflected in both places immediately

**Given** a Colleague-level user views a column configured as inline-editable
**When** they attempt to edit a cell for someone they hold no Manager/PP relationship with
**Then** the cell renders non-editable, and a direct API write attempt is rejected server-side regardless of UI state

### Story 3.4: Saved and Shared Views

As a manager,
I want to save a filter/column configuration as a named view and share it with other managers,
So that useful lists don't need rebuilding.

**FRs:** FR-10.

**Acceptance Criteria:**

**Given** a manager configures filters and columns on All Employees
**When** they save the configuration as a named view "Needs a conversation"
**Then** it appears as a tab, persists across sessions, and coexists with any other saved views they have — filter-based only for v1, no static membership-list variant (`decisions.md` appendix)

**Given** Manager A saves a view including a management-visible custom field and shares it with Manager B
**When** Manager B opens the shared view
**Then** B sees only the rows and columns B is personally entitled to see — the view is a re-executed query definition, never a cached data snapshot

### Story 3.5: Export to Excel

As a user viewing the list,
I want to export the current view to `.xlsx`,
So that I can work with it outside the app.

**FRs:** FR-11.

**Acceptance Criteria:**

**Given** a manager's current view includes a custom field visible only to "management," which the manager holds
**When** they export to `.xlsx`
**Then** the file contains that column and matching filtered rows, generated server-side from the same access-resolved query the list itself uses — never from what happens to be rendered in the DOM

**Given** a user's column selection somehow includes a field they aren't entitled to see for one or more rows
**When** they export
**Then** the resulting file omits that column/value entirely for those rows, with no trace anywhere in the file, and export doesn't time out or truncate at 500+ rows

### Story 3.6: Colleague Mode of the List

As a Colleague-level user,
I want the same list page scoped to the whitelist,
So that I can still browse people without seeing restricted data.

**FRs:** FR-12. **Architecture:** AD-5 — the whitelist is the resolved-role grant map C1 returns, enforced identically to profile assembly, never a frontend hide.

**Acceptance Criteria:**

**Given** a user holds no Manager, PP, or HR Admin relationship with respect to anyone currently in the list
**When** they open All Employees
**Then** only S1 identity fields, S10 leave type, and S11 project name are available as columns/filters, verified absent from the API response on direct inspection — not merely hidden in the UI

**Given** a user is the manager of Employee X but a plain colleague to Employee Y
**When** they open All Employees showing both rows
**Then** X's row shows manager-line fields while Y's row is whitelist-only, correctly scoped per row within the same render

### Story 3.7: All Employees List Performance at 500+ Records Under 2 Seconds

As the delivery team,
I want a repeatable load test validating the list's filter/sort engine and access-resolution together at scale,
So that NFR-2 is verified under realistic conditions, not assumed from unit tests alone.

**FRs:** NFR-2. **Architecture:** exercises Story 1.13's caching rule (AD-4) under load — if a cache exists, this is where its correctness-under-load is actually checked.

**Acceptance Criteria:**

**Given** a workspace seeded with the bootcamp manifest (Story 1.16, 24 accounts) **and** a load fixture or scale-up to 500+ records for perf testing
**When** the DM loads All Employees with three filters applied, including a derived field and a custom field
**Then** the response, including full permission resolution, returns within 2 seconds

**Given** the CI performance benchmark for the list is part of the pipeline
**When** a change increases response time beyond the 2-second budget at 500+ records
**Then** the pipeline flags the regression before merge, and any case that can't meet the bar is documented explicitly rather than shipped as "close enough"

---

## Epic 4: Action Items and Tasks

The platform's single task entity, sourced manually or via campaign activation. 3 stories reconciled + 1 new (renumbered from the backlog's unnumbered "NEW").

### Story 4.1: Manually Create an Action Item

As a manager or PP,
I want to create an action item for someone in my access scope,
So that I can assign them a task with a due date.

**FRs:** FR-17.

**Acceptance Criteria:**

**Given** I am UM for employee B (B reports to me)
**When** I create an item with title "Submit Q3 self-review," a due date, and no link
**Then** it is created with status `open`, author me, assignee B, source `manual`, and appears in B's S14 section and my dashboard's own-action-items widget

**Given** I am a PM with no Manager/PP relationship to employee C and no permitted functional role over C
**When** I attempt to create an action item for C via the API
**Then** the request is rejected server-side, evaluated against the same transitive access resolution as everywhere else — not cached or inferred client-side

### Story 4.2: Complete and Cancel Action Items

As the assignee,
I want to mark my action item complete; as the author, I want to cancel one with a reason,
So that the lifecycle reflects reality.

**FRs:** FR-19.

**Acceptance Criteria:**

**Given** I am the assignee of an open item
**When** I mark it complete
**Then** status becomes `completed`, completion date is set to now, and only I — not the author, manager, or PP — can do this

**Given** I am the author of an open item
**When** I cancel it without a reason
**Then** the cancellation is rejected; providing a reason succeeds, sets status to `cancelled`, stores the reason, and this works even if I've since lost live Manager/PP access to the assignee (authorship is a historical fact, per `decisions.md` appendix) — both terminal states are final, never reopenable

### Story 4.3: Overdue Highlighting

As any viewer of an action item,
I want overdue items flagged wherever they appear,
So that nothing slips through unnoticed.

**FRs:** FR-20.

**Acceptance Criteria:**

**Given** an open item with a due date in the past
**When** it renders on the assignee's profile, self-service list, or any manager/PP/dashboard view
**Then** it is visibly marked overdue in every one of those places, using one consistent derivation — never a stored flag needing a batch job

**Given** an overdue item is completed or cancelled
**When** it next renders anywhere
**Then** it is never shown as overdue again, including in a campaign's per-person completion table for a campaign-sourced item

### Story 4.4: Auto-Generate Action Item on Form Campaign Activation

As the system,
I want campaign activation to generate exactly one action item per frozen recipient,
So that Epic 10's campaigns and this epic's task entity are the same underlying mechanism, never two.

**FRs:** FR-18. **Architecture:** interface contract C6 `ActionItemCreation` (AD-2) — Epic 10 calls this directly; assigning Epic 4 and Epic 10 to the same developer avoids a cross-track wait on this contract.

**Acceptance Criteria:**

**Given** a PP has activated a form campaign "Annual Engagement Survey" with a frozen audience of 50 employees
**When** activation completes
**Then** exactly 50 action items are created, one per recipient, each with the campaign's title, sender, due date, link, source `campaign`, status `open`
**And** activation is atomic — a partial failure never leaves some recipients with an item and others without

**Given** a campaign-sourced action item's due date has passed and it's still open
**When** the campaign's sender views the per-person completion table
**Then** the recipient shows as overdue, using the exact same derivation as Story 4.3 — no separate logic

---

## Epic 5: Risks and Risk Dashboard

A retained-history, five-level risk record per employee, never visible to the employee themself. All 3 stories reconciled.

### Story 5.1: Record a Risk

As a manager or PP,
I want to record a risk level with description, details and date for a person I'm responsible for,
So that risk history is tracked over time.

**FRs:** FR-21.

**Acceptance Criteria:**

**Given** I hold Manager or PP access over employee E
**When** I submit a new risk record with level `high`, a description, details, and a date
**Then** it is appended to E's risk history (append-only, never overwritten) and becomes E's current level

**Given** I am employee E viewing my own profile
**When** I request my own profile data, including a direct API call to the risk section
**Then** S6 is absent from the response entirely — no create/view control is offered to me under any circumstance, this is the one section with zero Self access

### Story 5.2: Show Risk Trend

As a viewer of a risk record,
I want to see a trend arrow versus the previous record,
So that I know whether things are improving or worsening.

**FRs:** FR-22. **Architecture:** trend is a backend-computed field on the risk provider's DTO (fixed severity ordering `low < need attention < medium < high < leaver`), never re-derived independently by three different frontend surfaces (per `EXPERIENCE.md`'s Trend Arrow component, DESIGN.md).

**Acceptance Criteria:**

**Given** employee E has a previous record at "low" and I hold Manager/PP access over E
**When** a new record is saved at "medium"
**Then** an "up" (worsening) arrow shows alongside the new current level

**Given** employee E has no prior risk records
**When** the first-ever record is saved
**Then** no arrow shows — there is no previous record to compare against, and the arrow is never visible to E themself

### Story 5.3: Risk Dashboard

As a manager or PP,
I want a Risk Dashboard scoped to my access, with counts by level and a sortable, filterable table with drill-through,
So that I can act on risk across my people.

**FRs:** FR-23.

**Acceptance Criteria:**

**Given** I am a People Partner assigned to a set of employees, some at "high" risk
**When** I open the Risk Dashboard
**Then** I see counts by level (medium/high/leaver visually emphasised) scoped exclusively to my assigned people, and clicking the "high" count filters the table to those people, sorted by severity then date; clicking a row opens the profile

**Given** I am an ordinary employee with no Manager/PP relationship over anyone
**When** I attempt to navigate directly to the Risk Dashboard URL or call its API
**Then** I am denied access or the response contains no risk data at all, for myself or anyone else

---

## Epic 6: Resourcing

Request → fulfilment → approval, spanning internal and external candidates, with full history retained. All 4 stories reconciled.

### Story 6.1: Create a Resourcing Request

As a DM or PM,
I want to create a resourcing request with vacancy details, comp level, duration and workload, optionally linked to a project,
So that I can start filling a role even before the project exists in the system.

**FRs:** FR-24.

**Acceptance Criteria:**

**Given** I am a DM and have not selected a project for this request
**When** I fill in vacancy details, comp level, duration and workload and save
**Then** the request is created and stored as a valid, fully-functional unattached request — no degraded state, no blocking validation on the project field

**Given** I am a DM responsible for Project X, and a PM of Project X has created a request for it
**When** I open my Resourcing → Requests view
**Then** I see both my own requests and the PM's, per the same manager-access chain as everywhere else (Story 1.2) — a PM outside my chain's requests stay invisible to me

### Story 6.2: Fulfil a Request with Internal or External Candidates

As a UM,
I want to see requests assigned to me and propose internal specialists or an external candidate,
So that I can submit candidates for approval.

**FRs:** FR-25. **Architecture:** interface contract C5 `ExternalIdentityMapping` (AD-2) resolves an attached PeopleForce candidate to a platform identity where the integration is live; the outbound-link fallback (spec-sanctioned) needs no contract at all.

**Acceptance Criteria:**

**Given** a resourcing request is assigned to me as UM
**When** I select a specialist from my unit and attach them as a candidate, then submit
**Then** the candidate is recorded as proposed and the request moves to DM review status

**Given** the PeopleForce integration is not yet live
**When** I attach an external candidate using an outbound PeopleForce link instead of a live record
**Then** the candidate is recorded with that link and the request can still be submitted for DM review — this fallback is accepted, permanent behavior for this iteration, not a placeholder

### Story 6.3: DM Reviews and Approves/Rejects Candidates

As a DM,
I want to review proposed candidates and approve or reject each with a written reason,
So that resourcing decisions are recorded.

**FRs:** FR-26. **Architecture:** internal-candidate review reuses Story 1.11's Shared Link Manager (AD-5) when the DM lacks standing access — never a separate ad hoc sharing mechanism.

**Acceptance Criteria:**

**Given** I am a DM reviewing a request with a proposed internal candidate I don't yet hold Manager/PP access to
**When** I open the candidate's profile link
**Then** I'm offered the Shared Link flow instead of a direct profile, and can approve after reviewing the shared content

**Given** I am a DM reviewing a proposed candidate
**When** I choose to reject without entering a reason
**Then** the rejection is blocked; providing a reason succeeds and the reason is stored — each candidate's decision on a multi-candidate request is recorded independently

### Story 6.4: Request History

As anyone involved in a resourcing request,
I want every proposal attempt recorded,
So that the full history is visible in Resourcing and on the candidate's profile.

**FRs:** FR-27. **Architecture:** hard constraint — approval never writes a `ProjectAssignment` (C3) record; `integrations`' timetracker sync (Epic 13) is the sole writer, per AD-1/AD-2's single-writer rule for that table.

**Acceptance Criteria:**

**Given** a DM approves an internal candidate proposed on a request
**When** the decision is saved
**Then** it appears immediately in Resourcing → Requests and in the candidate's profile S15 (Manager line/PP read-only, never visible to the candidate themself) — and no project record is created locally; S11 changes only after the next timetracker sync

**Given** an employee was proposed and rejected with a written reason
**When** that employee views their own profile
**Then** S15 is not rendered and not returned by the API for them, while their manager line and PP can still see the rejection and its reason

---

## Epic 7: Career Timeline

A system-generated event log with manual override — never a separately maintained record. 2 stories reconciled + 1 new (renumbered from the backlog's unnumbered "NEW"), all now resolved by D2/AD-7's settled conflict rule rather than the open design question the original backlog framed.

### Story 7.1: Auto-Generate Timeline Events

As the system,
I want to write a timeline event automatically whenever a tracked change occurs,
So that the timeline stays current without manual upkeep.

**FRs:** FR-28. **Architecture:** AD-7 — a Prisma Client Extension on the four temporal history tables (Story 1.20) fires this write automatically; no service method writes to history any other way.

**Acceptance Criteria:**

**Given** an employee currently has grade "Middle" and a UM with edit access updates it to "Senior" effective 2026-09-01
**When** the change is saved
**Then** a timeline event of type "grade change" is written automatically, recording old value, new value, effective date, and flagged system-generated

**Given** the timetracker sync reports an extended leave and the career-timeline write fails due to a transient error
**When** the sync completes
**Then** S10 still reflects the leave, the missing timeline write is logged for retry, and the application does not crash or block other profile functionality

### Story 7.2: Manual Timeline Edits

As a PP or UM,
I want to manually add, edit or delete timeline events,
So that I can backfill historical data or correct wrongly inferred events.

**FRs:** FR-29. **Decision:** soft-delete with audit trail (`decisions.md` appendix, confirmed default).

**Acceptance Criteria:**

**Given** a PP is viewing an employee's timeline and the Excel headcount record shows a grade change predating the system
**When** the PP manually adds a timeline event dated 2019-03-15 with old/new values
**Then** it saves, is tagged manually entered, and inserts into its correct chronological position — not merely appended

**Given** a DM has Manager-line access to an employee via project assignment but is not the assigned UM or PP
**When** the DM attempts to edit or delete a timeline event
**Then** the request is rejected server-side — only UM and PP have write access to S9 events, the one documented narrowing within Manager line's section-level RW

### Story 7.3: Resolve Conflicts Between System-Generated and Manually-Edited Timeline Events

As the system,
I want a later system-generated write to skip, never overwrite, an existing manual correction covering the same change window,
So that Story 7.2's entire premise — correcting a wrongly-inferred event — isn't defeated by the next sync.

**FRs:** FR-30. **Decision:** D2 (settled, not an open team decision as the original backlog framed it) — skip metadata is attached to the manual event via C4's `markSystemWriteSkipped`; the manual entry shows the EXPERIENCE.md affordance ("A system update was skipped here — {date}"), never a separate skip row.

**Acceptance Criteria:**

**Given** a system-generated grade-change event exists, a PP edits it to correct the effective date, and the edit is tagged manually edited
**When** a later sync re-derives the same underlying change for the same window
**Then** the system does not overwrite or duplicate the manual edit — it skips the write, calls `markSystemWriteSkipped` on the manual event, and the timeline still shows exactly one event at the corrected date with the skip affordance visible on it

**Given** a PP has manually backfilled an unrelated department change, and the employee later has a genuinely new, unrelated department change
**When** the system's auto-write logic processes the new change
**Then** a new system-generated event is written normally — the earlier manual entry never suppresses an unrelated future event

---

## Epic 8: CDS: Career Development System

A registry and hub, not an assessment engine. All 4 stories reconciled.

### Story 8.1: Skills Matrix Link and Assessment Log

As anyone viewing S12,
I want to see a link to the current skills matrix for the person's department+position and a log of past assessments,
So that I have the full CDS picture.

**FRs:** FR-31 (+ read side of FR-32).

**Acceptance Criteria:**

**Given** "Engineering" + "Backend Developer" is mapped to "matrix-backend-v3.pdf" in the dictionary, and the employee has a completed assessment on file
**When** the employee's manager opens the CDS section
**Then** the manager sees the current matrix link and the assessment log entry with its date, assessor, result link, and conclusion

**Given** 40 employees share that department+position and the dictionary entry is updated to "matrix-backend-v4.pdf"
**When** any of those profiles is viewed afterward
**Then** all 40 show the new file with no individual employee record edited — and a Colleague viewer sees no trace of S12 at all

### Story 8.2: IDP Records

As a manager/PP, I want to create and update IDP records; as the employee, I want to mark my own IDP complete,
So that development plans are tracked to completion.

**FRs:** FR-33. **Decision:** reopening a completed IDP is disallowed by default (`decisions.md` appendix, confirmed).

**Acceptance Criteria:**

**Given** I am the Unit Manager of employee Jane
**When** I create an IDP with description, deadline 2026-12-01, and an external file link
**Then** it appears on Jane's CDS section with no completion date — shown as open

**Given** Jane has that open IDP
**When** Jane opens self-service and checks "complete"
**Then** the completion date is recorded as today and shown alongside the deadline; Jane has no control to create, edit, or delete IDP records — only the checkbox — and a shared-link viewer with S12 enabled never sees the checkbox at all

### Story 8.3: Manager/PP Maintain Assessments and Conclusions

As a manager or PP,
I want to create assessment records and edit conclusions,
So that the CDS log stays accurate.

**FRs:** FR-32. **Decision:** assessor field is free text or a person reference, implementer's choice (`decisions.md` appendix).

**Acceptance Criteria:**

**Given** I am the PP assigned to employee John, whose log has one prior entry
**When** I add a new record with date, assessor, result link, and a conclusion
**Then** it is appended — the prior entry stays unchanged, and no numeric score/rating field exists anywhere in the model, per the registry-only scope boundary

**Given** John's assessment record has an existing conclusion
**When** his Unit Manager edits the conclusion text
**Then** the date/assessor/result-link stay unchanged, no new log entry is created, and a Colleague or unrelated Manager/PP cannot write to this at all — rejected server-side even if the UI never exposes the control

### Story 8.4: Filter by Assessment Recency and Open IDP

As a manager or PP,
I want to filter All Employees by last-assessment-date and by has-open-IDP,
So that I can find people who need attention.

**FRs:** FR-34. **Architecture:** exposed via CAP-2's `FieldRegistry`/`FieldProvider` (Epic 3, AD-3) — CDS is the data owner, Directory is the query surface.

**Acceptance Criteria:**

**Given** employee A's last assessment is 2024-01-10, employee B has zero assessment records, and employee C's is 2026-07-01
**When** a PP filters "assessed before 2025-01-01"
**Then** A appears, C doesn't, and B appears in neither — "never assessed" is its own distinct, separately selectable filter option, not an empty value that silently drops out

**Given** employee D has one open IDP, E has one closed IDP, and F has none
**When** their manager filters "has an open IDP: yes"
**Then** only D appears — and none of this is exposed to a viewer without Manager/PP access to these employees, consistent with S12 being absent for Colleague

---

## Epic 9: Mentorship Hub

Self-flagged willingness, manager/PP-driven pairing, mandatory closing feedback. 2 stories reconciled + 3 new (renumbered from the backlog's "NEW A/B/C").

### Story 9.1: Self-Flag Open to Mentoring

As an employee,
I want to flag myself as open to mentoring and see my assigned mentor/mentees,
So that I can participate in the mentorship program.

**FRs:** FR-35.

**Acceptance Criteria:**

**Given** I am viewing my own profile's Mentorship section
**When** I set "open to mentoring" on and save
**Then** the flag persists and I immediately appear in the manager/PP-facing willing-mentors list

**Given** I have an assigned mentor and/or mentees
**When** I view my own profile
**Then** I see them read-only, in the section and the header — and any attempt to set my own status directly to "mentor" is rejected, since status is derived only from flag + pair lifecycle (Story 9.4)

### Story 9.2: Assign a Mentor-Mentee Pair

As a manager or PP,
I want to see everyone open to mentoring and pair a willing mentor with a mentee,
So that mentorship relationships form.

**FRs:** FR-36.

**Acceptance Criteria:**

**Given** a willing mentor with zero existing pairs, and a mentee within my Manager/PP access scope
**When** I select the mentor, pick the mentee, and create the pair
**Then** the pair is recorded active with a start date, the mentor's status changes to "mentor," and a "mentorship pair start" event is written to both people's timelines (Story 7.1)

**Given** I am a UM whose access covers only my own subordinates and their reports
**When** I open the assignment flow for a willing mentor
**Then** the mentee picker lists only employees within my resolved access scope — the mentor list itself is global, only mentee selection is scoped

### Story 9.3: End a Mentorship Pair with Required Final Feedback

As a manager or PP,
I want to explicitly end an active mentorship pair and be required to provide final feedback before it can close,
So that the pairing is properly concluded and its outcome is captured.

**FRs:** FR-37. **Decision:** D6 — closing feedback is stored on the pair record itself, not routed through the general Feedback entity, deliberately decoupling this epic from Epic 11.

**Acceptance Criteria:**

**Given** an active mentorship pair
**When** a manager or PP attempts to end it without entering final feedback
**Then** the action is blocked — the pair stays active with no end date recorded

**Given** the manager/PP provides final feedback and confirms
**When** the pair ends
**Then** an end date is recorded, the feedback is persisted on the pair, an end event writes to both timelines, the ended pair stays visible in history on both profiles, and if this was the mentor's last active pairing their status reverts per Story 9.4's rule

### Story 9.4: Automatic Mentor Status Transitions

As the platform,
I want mentorship status to be a derived field, never directly settable by any user through any surface,
So that it stays a function of the self-flag and active pairs alone.

**FRs:** FR-38. **Decision:** D4 — reverts to "open to mentoring" on last-pair-end only if the self-flag is still on; otherwise reverts to no status at all.

**Acceptance Criteria:**

**Given** a willing mentor with zero pairs receives their first pair assignment (Story 9.2)
**When** the pair is created
**Then** their status transitions to "mentor" atomically with pair creation — no intermediate inconsistent state

**Given** a mentor with exactly one active pairing has it ended (Story 9.3), and their self-flag is still on
**When** the end is confirmed
**Then** status reverts to "open to mentoring"; if the flag had been turned off, status reverts to no status at all
**And** any direct write attempt to the status field, through any surface, is rejected — it's queryable/filterable on All Employees but never directly editable

### Story 9.5: View All Mentor-Mentee Pairs (Active and Ended)

As a manager or PP,
I want a dedicated view listing every mentor-mentee pair in the organization, active and ended, with dates and status,
So that I can see the overall state of the mentorship program at a glance.

**FRs:** FR-39.

**Acceptance Criteria:**

**Given** several pairs exist, some within and some outside my Manager/PP access
**When** I open the "All pairs" view
**Then** I see only pairs where I hold Manager or PP access to the mentor or the mentee, each row showing mentor, mentee, start date, end date, and status

**Given** the list contains both active and ended pairs
**When** I filter by status
**Then** the table updates accordingly, and clicking a name navigates to that profile subject to my normal access rights — no new access is granted by this view

---

## Epic 10: Forms & Survey Campaigns

A campaign that targets a frozen audience with a link to an externally hosted form, tracked entirely through action-item completion — the platform never reads the form itself. All 4 stories reconciled from `PRD_parallel_delivery_plan.md` §7 (this epic had zero prior stories in the backlog draft).

### Story 10.1: Create a Form Campaign

As a PP, manager, or holder of the "create form campaigns" functional permission,
I want to create a campaign record with title, description, purpose, an external form link, and a due date,
So that I can start a survey/form task without the system ever hosting the form itself.

**FRs:** FR-40.

**Acceptance Criteria:**

**Given** I am a People Partner
**When** I create a campaign titled "Annual Engagement Survey" with a description, purpose, an external Google Forms link, and a due date
**Then** it saves in draft state, visible to me, with no action items generated and no employees notified

**Given** I hold a functional role granted "create form campaigns" but no Manager/PP relationship to anyone
**When** I create a campaign record
**Then** it is created successfully — the permission unlocks the feature; my ability to *target* anyone is still bounded by my own access scope when I build the audience (Story 10.2)

### Story 10.2: Build and Freeze Campaign Audience

As a campaign creator,
I want to build the audience using the All Employees filter engine, preview it, and adjust it individually,
So that the recipient list is exactly who I intend before I commit to sending anything.

**FRs:** FR-41. **Architecture:** reuses Epic 3's `FieldRegistry`/filter engine directly (C2) — no separate audience-resolution logic.

**Acceptance Criteria:**

**Given** I filter for "department = Engineering AND country = Poland" and preview the results
**When** I then remove one person and add another who didn't match the filter
**Then** my adjusted list is the current audience preview, scoped to my own resolved access — I cannot preview or target anyone my access wouldn't show me on All Employees

**Given** a campaign's audience preview currently resolves to 50 people
**When** the campaign is activated (Story 10.3) and afterward a new employee joins who would also match
**Then** the new employee is never added — the list is frozen at activation, not a live query

### Story 10.3: Activate a Campaign

As a campaign creator,
I want activation to atomically freeze the audience and generate one action item per recipient,
So that a campaign can never be half-activated and no recipient is silently skipped.

**FRs:** FR-42. **Architecture:** interface contract C6 `ActionItemCreation` (Epic 4, AD-2) — assign this story and Epic 4's action-item creation to the same developer to turn the dependency into same-dev sequencing, never a cross-track wait.

**Acceptance Criteria:**

**Given** a draft campaign has a resolved audience of 30 people and complete fields
**When** the creator activates it
**Then** the audience freezes at exactly those 30, and 30 action items are created — one per recipient, each with the campaign's title, sender, due date, and link

**Given** an activation is in progress and an error occurs partway through creating action items
**When** the activation fails
**Then** the campaign is not left partially active — it either stays a fully-editable draft or the activation retries to completion; recipients never end up split between having an item and not
**And** once activated, title/description/link/due date lock, and the campaign cannot be activated twice or deactivated back to draft

### Story 10.4: Track Campaign Completion

As the campaign creator,
I want a per-person table showing who has completed, who hasn't, and who is overdue,
So that I can see response progress without the system ever reading the external form.

**FRs:** FR-43. **Architecture:** reuses Epic 4's overdue derivation (Story 4.3) exactly — no separate logic.

**Acceptance Criteria:**

**Given** a campaign was activated for 30 recipients, 10 of whom marked their action item complete, 5 of whom are past due and still open
**When** the creator opens the tracking table
**Then** they see 10 completed, 5 overdue, and 15 not-yet-completed-but-not-overdue, live-updating with no manual refresh

**Given** a recipient has not actually filled in the external form but marks their action item complete anyway
**When** the sender views the tracking table
**Then** that recipient shows as completed — the system has no way to detect otherwise, by design, and this table is visible only to the campaign's creator, not to other recipients

---

## Epic 11: Feedback

Manager/PP-authored feedback with a visibility flag, viewable over time, plus a named-colleague feedback-request flow built on Form Campaigns. All 3 stories reconciled from `PRD_parallel_delivery_plan.md` §7.

### Story 11.1: Record Feedback with a Visibility Flag

As a manager or PP,
I want to add a feedback record on an employee's profile with subject, author, date, context, and body, defaulting to management-only visibility,
So that sensitive feedback stays private unless I explicitly choose to share it.

**FRs:** FR-44. **UX:** Visibility Flag Toggle (UX-DR13, single-flag variant), Feedback Panel (UX-DR11).

**Acceptance Criteria:**

**Given** I am the manager of employee B
**When** I create a feedback record about B with context "Q3 project retrospective" and a body, leaving the visibility flag unset
**Then** it saves as "management only" by default and is not visible to B on their own profile

**Given** a management-only feedback record about B already exists
**When** B's PP edits the record's visibility flag to "shared with employee"
**Then** B can now see that specific record on their own profile immediately (Story 2.5) — no stale grant, consistent with the same rule everywhere else in the system

### Story 11.2: View Feedback Over Time and Compare Periods

As a manager or PP,
I want feedback laid out chronologically with a period-comparison view,
So that I can see patterns or shifts in feedback over time rather than reading disconnected records.

**FRs:** FR-45.

**Acceptance Criteria:**

**Given** employee B has feedback records dated across Q1 and Q3
**When** B's manager opens the feedback view and selects "compare Q1 vs Q3"
**Then** records from each quarter are shown grouped/delineated by period, chronological within each

**Given** employee B has feedback records only in Q3, none in Q1
**When** B's manager compares Q1 vs Q3
**Then** Q1 renders an explicit empty state and Q3 shows its records — no crash, no hidden panel, no silently omitted side of the comparison

### Story 11.3: Request Feedback from Named Colleagues via a Form Campaign

As a manager or PP,
I want to request feedback about a person from specific named colleagues, implemented as a targeted Form Campaign,
So that I reuse the campaign mechanism instead of building a second data-collection system.

**FRs:** FR-46. **Architecture:** thin reuse of Epic 10's campaign creation/audience/activation/tracking flow — no separate "feedback campaign" data model. Frontend-orchestrated per AD-11 (this epic never calls into `campaigns` directly; the page sequences the calls).

**Acceptance Criteria:**

**Given** I am a PP and want feedback about employee B from three specific colleagues
**When** I initiate "request feedback" for B and select those three by name as the audience
**Then** a form campaign is created targeted exactly at those three, associated with B as subject, ready to activate through Epic 10's normal flow — named individually, not via a filter

**Given** the feedback-request campaign has been activated and all three colleagues mark their action items complete
**When** the PP checks the campaign's completion table (Story 10.4)
**Then** all three show completed, but no Feedback record is auto-created — the PP manually creates it (Story 11.1) after reviewing the external form's actual responses

---

## Epic 12: Dashboards

One shared engine, four role-scoped views — grouping differs, the underlying components don't. All 5 stories reconciled.

### Story 12.1: Build Shared Dashboard Engine

As the delivery team,
I want one reusable dashboard engine (counter tiles, scoped table, own-action-items widget, quick nav) configured per role rather than four separately coded pages,
So that adding a future role dashboard is a configuration change, not a new build.

**FRs:** FR-47. **Architecture:** every widget resolves data through C1 `AccessResolver`/the Provider Registry (AD-3) — a dashboard is not a separate access surface.

**Acceptance Criteria:**

**Given** the engine is configured for both UM and DM
**When** each dashboard renders
**Then** both use the same counter-tile, table, and action-item-widget components, differing only in grouping and which blocks are configured to appear

**Given** a UM opens their dashboard
**When** the subordinates table and counters render
**Then** only people the UM holds Manager access over appear, with no dashboard-specific bypass of the standard access-resolution layer

### Story 12.2: Unit Manager Dashboard

As a Unit Manager,
I want a dashboard grouped by people, with counters, a subordinates table, my own action items, and quick navigation,
So that I can act on my team without navigating away.

**FRs:** FR-48.

**Acceptance Criteria:**

**Given** a UM has 12 subordinates via the reporting hierarchy, three with active risk records
**When** the UM opens their dashboard
**Then** the headcount counter reads 12, risk-by-level counters reflect exactly those three, and the subordinates table lists all 12 with risk, project, and leave status

**Given** the UM authored an action item for a subordinate, now overdue
**When** the UM opens their dashboard
**Then** it appears in their own-action-items widget, sorted by due date, visibly marked overdue via the same derivation as Story 4.3

### Story 12.3: Delivery Manager Dashboard with Project Selector

As a Delivery Manager,
I want a dashboard grouped by project, with a selector that filters the whole page when I pick one project,
So that I can see either my full portfolio or drill into a single project.

**FRs:** FR-49.

**Acceptance Criteria:**

**Given** a DM is responsible for three projects with a combined 18 people and 2 open risk records
**When** the DM opens their dashboard with no project selected
**Then** three project tables render and the top counters show headcount 18 and both risk records, aggregated across all three

**Given** the DM's dashboard is showing all three projects
**When** the DM selects "Project Atlas"
**Then** only Atlas's table remains and every counter recalculates to Atlas alone; clearing the selection restores the all-projects view, and the resourcing block shows both the DM's own requests and those created by PMs of their projects (Epic 6)

### Story 12.4: Project Manager Dashboard

As a Project Manager,
I want the same dashboard shape as the DM, scoped to my own projects,
So that I see exactly my portfolio, nothing more.

**FRs:** FR-50.

**Acceptance Criteria:**

**Given** a person is PM on two projects and an ordinary team member on a third
**When** they open their PM dashboard
**Then** only the two PM projects appear as project tables — reusing Story 12.3's engine, not a separate page implementation

**Given** the PM's dashboard renders its resourcing block
**When** requests are shown
**Then** only requests the PM personally created appear (Story 6.1's PM scoping), not the broader DM-level visibility

### Story 12.5: People Partner Dashboard

As a People Partner,
I want the same building blocks scoped to my assigned people, groupable by department or project, with no resourcing block,
So that I see exactly the HR-relevant picture for my portfolio.

**FRs:** FR-51.

**Acceptance Criteria:**

**Given** a PP is assigned to 40 employees across two departments
**When** the PP opens their dashboard
**Then** all counters and tables reflect only those 40, groupable by department or project, and no resourcing block appears anywhere — even if the PP separately holds another functional role, that role's UI lives elsewhere, not folded into this dashboard's default composition

**Given** several of the PP's assigned people have IDPs due within 30 days
**When** the PP opens their dashboard
**Then** an "IDPs approaching deadline" widget (first cut of the design-freedom HR widgets) lists those people, scoped to the PP's assigned population only

### Story 12.6: Accessibility and Responsive Layout Pass

As the delivery team,
I want a dedicated audit-and-fix pass across the three highest-traffic surfaces — All Employees, Employee Profile, and the four dashboards,
So that NFR-3 (WCAG 2.1 AA) is verified and fixed, not assumed from component-level intent alone.

**FRs:** NFR-3, UX-DR16, UX-DR15. **Note:** deliberately scheduled last — it needs Epic 1 (profile), Epic 3 (list), and this epic's own dashboards all substantially built first.

**Acceptance Criteria:**

**Given** a user relies on keyboard-only navigation
**When** they use All Employees to apply a filter, sort a column, and open a profile
**Then** every action is achievable via keyboard alone, with visible focus indicators throughout

**Given** a screen-reader user views a risk dashboard row or an overdue action item
**When** the assistive technology announces it
**Then** the risk level or overdue status is conveyed through text/label, never through color or icon alone — and all three surfaces are usable, not just "not broken," at desktop/tablet/narrow-mobile breakpoints, with self-service specifically checked on mobile

---

## Epic 13: External Integrations: Timetracker & PeopleForce

Two integrations, one of which is load-bearing for the access model itself, not just for display. All 4 stories reconciled — 13.2's "unresolved tension" and 13.4's "open decision," as the original backlog draft called them, are now both settled by this session's architecture and decision work.

### Story 13.1: Integrate Timetracker Leaves API

As the platform,
I want vacation/sick/parental/other leave data pulled from the timetracker into S10,
So that leave status is accurate everywhere it's shown without the platform owning leave management itself.

**FRs:** FR-52. **Architecture:** AD-8's fail-soft display rule (S10/S11 show "temporarily unavailable" on sync lag) covers this feed too, not just the projects/people one.

**Acceptance Criteria:**

**Given** the timetracker reports an employee on approved vacation 2026-08-25 to 2026-08-29
**When** the profile (S10) or All Employees is rendered for an entitled viewer
**Then** the dates and type are shown, sourced from the feed, respecting S10's existing access matrix unchanged (this story supplies data, never alters who can see it)

**Given** the timetracker leaves API is unreachable
**When** a viewer opens a profile or All Employees
**Then** S10 shows a clear "temporarily unavailable" state and the rest of the page renders normally — no crash, no block

### Story 13.2: Integrate Timetracker Projects & People API (Permission-Critical)

As the platform,
I want the real projects/people feed to become `ProjectAssignment`'s live writer, with confidence-state gating,
So that Manager access derived from project assignment is accurate and never stale during a sync outage.

**FRs:** FR-53. **Decision/Architecture:** D3 + AD-8 (settled — the original backlog flagged fail-safe-vs-fail-open as an open decision this story "cannot skip"; it's resolved now, not a prerequisite this story still owes). C3's ratified shape carries `confirmed`/`confirmedAt`; this story is the writer.

**Acceptance Criteria:**

**Given** the timetracker reports employee B newly assigned to Project X, whose PM is P
**When** the next sync runs successfully
**Then** the `ProjectAssignment` row is written with `confirmed: true` and a fresh `confirmedAt`, and P's resolved access to B includes Manager access via the project leg, sourced live rather than from seed data

**Given** the timetracker projects/people API is unreachable for a prolonged period
**When** access is resolved for a subject/object pair whose grant depended on this feed
**Then** the row's `confirmedAt` ages out of the freshness window and Manager access is not granted from it — fail-safe by construction, not by a per-incident judgment call — while sync failures are logged/alertable, never silent

### Story 13.3: Investigate & Integrate PeopleForce Candidate/Vacancy API with Fallback Link

As the platform,
I want a time-boxed investigation of PeopleForce's API followed by whatever integration is feasible, with the outbound-link fallback always available,
So that Resourcing never blocks on this integration being fully live.

**FRs:** FR-54.

**Acceptance Criteria:**

**Given** the team begins this story
**When** the time-boxed investigation concludes
**Then** a decision record covering auth, candidate/vacancy endpoints, custom fields, rate limits, and webhooks is committed to the intelligent repository, informed by `developer.peopleforce.io`'s `llms.txt` index — and implementation scope is chosen based on it

**Given** the live integration is not completed within this iteration
**When** a UM attaches an external candidate in Resourcing (Story 6.2)
**Then** the candidate is recorded via an outbound PeopleForce link, treated as a normal supported path, not a broken feature

### Story 13.4: Resolve Cross-System Identity (PeopleForce ↔ Platform ↔ Timetracker)

As the platform,
I want every PeopleForce and timetracker record to resolve to the correct platform employee via an explicit mapping table, never email inference,
So that re-hires, email changes, and contractor transitions never produce a misattributed record.

**FRs:** FR-55. **Decision/Architecture:** D8, interface contract C5 (already ratified as a Wave-0 starting schema in `interface-contracts.md`/AD-2 — this story validates/refines it against real investigation findings, it does not design it from a blank page as the original backlog framed it).

**Acceptance Criteria:**

**Given** the foundation-phase investigation into PeopleForce/timetracker identity concludes
**When** the team reviews C5's `external_identities` schema (`system`, `externalId`, `employeeId`, `supersededBy`) against real findings
**Then** the schema is confirmed or refined, and the decision is recorded as a dated entry in the intelligent repository, not left implicit in code

**Given** a candidate sourced from PeopleForce is approved and later onboarded as an employee
**When** their platform employee profile and original PeopleForce candidate record are compared
**Then** the mapping table correctly associates the two via `(system, externalId) → employeeId`, with `supersededBy` handling the re-hire case — never an email-only match

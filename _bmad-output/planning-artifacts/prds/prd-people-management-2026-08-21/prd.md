---
title: People Management Platform
created: 2026-08-21
updated: 2026-08-21
status: final
---

# PRD: People Management Platform

## 0. Document Purpose

This PRD is for the delivery team (three developers building in parallel), the downstream BMad workflows (`bmad-ux`, `bmad-architecture`, `bmad-create-epics-and-stories`, `bmad-sprint-planning`), and whoever evaluates the finished platform. It is derived from, and stays consistent with, the project's canonical SPEC at `_bmad-output/specs/spec-people-management-platform/SPEC.md` (plus its companions `access-model.md`, `interface-contracts.md`, `decisions.md`) — that SPEC is the reviewed, preservation-validated contract; this PRD narrates it into product terms (vision, journeys, feature-level requirements, metrics) without re-deciding anything it already settled. Where this document notes a confirmed default, no stakeholder was available to confirm it directly at the time it was inferred (the bootcamp brief's PM/DM/other roles are fictional, not people to consult) — each is indexed in §10. Features are grouped by the SPEC's 13 capabilities; FRs are nested under each and numbered globally (FR-1–FR-55) for stable downstream reference. Glossary terms are used verbatim throughout — no synonyms.

## 1. Process and Data Guardrails

*Constraints from `SPEC.md` that govern how this platform is built and operated, not what it does — surfaced first so they aren't rediscovered late, after Vision and Features are already being read.*

- **BMad is the mandated framework.** BMad governs planning and process for at least the start of this project. Migrating away from it must be a recorded, deliberate decision — never silent drift.
- **A foundation phase gates implementation.** Before feature work starts, the team aligns on and writes down: prototyping/design approach, best-practice research, test architecture, and named owners per topic — run in parallel across the team, not sequentially. Feature implementation does not start until this is done.
- **The intelligent repository is mandatory documentation**, not optional artifact hygiene — specs, decisions, call transcripts, external API docs, and agent rules/skills all live in it, and it is expected to stay current (SM-6).
- **Inter-team communication and status are captured** for analysis, not left in ephemeral channels.
- **Non-production environments use pseudonymized data only** — real structure and volume, substituted names and contacts. Real personal data must never appear in agent contexts, logs, screenshots, or the repository.
- **Grade, position, department, and employment type are temporal.** They must be modeled as time-bounded records, not scalar fields with a bolted-on audit log. This is a data-modeling requirement for `bmad-architecture` to satisfy; it's named here so it isn't rediscovered late.

## 2. Vision

An engineering organization of 500+ employees, spread across units and delivery projects, currently manages people data through a patchwork that doesn't encode who's allowed to see what about whom. The People Management Platform replaces that with a single system built around one governing idea: every piece of information about a person lives in a **section**, and every section has an explicit, enforced access rule per audience — never a blanket "manager sees everything" assumption, never a client-side hide, never a stale grant. A unit manager opens a profile and sees exactly what their role and relationship to that person earn them; a colleague sees a whitelist; an HR Admin can invent a whole new functional role from the UI on a Tuesday afternoon and have it enforced everywhere by Wednesday.

*Confirmed: this is a web application, responsive across desktop/tablet/mobile browsers — no native client is in scope.*

This is also, deliberately, the vehicle for an AI-native SDLC bootcamp. The platform's functional scope matters, but the source spec is explicit that *how* it gets built — spec-driven, genuinely parallelized across three developers with zero cross-person blocking, documented in an intelligent repository — is graded above the shipped feature count. This PRD, and everything built from it, is judged as much on process fidelity as on working software.

## 3. Target User

### 3.1 Jobs To Be Done

- **As a Unit Manager**, I need to see my people's risk, leave, and project status in one place, and act on it (assign mentors, raise risks, fulfill resourcing requests) without waiting on HR to pull data for me.
- **As a Delivery/Project Manager**, I need visibility into everyone on my projects — even people I don't formally manage — the moment they're assigned, and that visibility to disappear the moment they roll off.
- **As a People Partner**, I need full HR-line visibility into my assigned people's sensitive sections (risk, notes, feedback, CDS) without holding any resourcing power I shouldn't have.
- **As an HR Admin**, I need to extend the platform to new organizational needs (a new functional role, a new custom field) myself, without filing a ticket and waiting on engineering.
- **As any employee**, I need self-service over my own data (contacts, photo, IDP, mentorship flag) and a guarantee that sensitive things about me (risk level, unflagged notes) stay invisible to me and to my colleagues.
- **As a Colleague of someone I don't manage**, I need enough visibility (name, role, project, leave status) to work with them day-to-day, and nothing more.

### 3.2 Non-Users (v1)

- External recruiters or candidates — they exist only as data pulled *into* the platform from PeopleForce, never as platform users themselves.
- Anyone outside the org's employee/contractor population — there is no public-facing surface.
- Compensation/payroll administrators — the platform carries no compensation data (§6).

### 3.3 Key User Journeys

*Both journeys below were authored from the source spec's described relationships and rules rather than narrated live (no PM/DM/other role-holder was available to consult) — confirmed as representative by the user.*

- **UJ-1. Daniela catches a flight risk before it becomes a resignation.**
  - **Persona + context:** Daniela is a People Partner responsible for ~40 people across two units; she checks in on her portfolio most mornings.
  - **Entry state:** Authenticated, viewing the PP Dashboard.
  - **Path:** She sees a counter showing 3 people at "high" risk, up from 2 last week. She drills through to the filtered Risk Dashboard table, sorted by severity, and clicks the newest high-risk entry — someone she doesn't directly manage but is PP for. Her PP access opens their full profile, including S7 management notes flagged visible-for-PM from that person's PM. She reads the note, sees a pattern (missed 1:1s, a comment about workload), and creates a management note of her own plus an action item for herself to schedule a conversation.
  - **Climax:** The trend arrow and the note context together tell her *why* the risk escalated, not just that it did — she doesn't have to ask the PM separately.
  - **Resolution:** She has a concrete next step logged (the action item) and a documented note trail for her next 1:1.
  - **Edge case:** If the note had no PM-visibility flag set, she'd still see it (PP has unconditional RW on S7) — but a PM checking the same profile without the flag would see nothing.

- **UJ-2. Marcus reviews an external candidate he's never met.**
  - **Persona + context:** Marcus is a Delivery Manager who requested a resourcing need two weeks ago; a Unit Manager has just proposed candidates.
  - **Entry state:** Authenticated, notified (or checking) that his resourcing request has proposals.
  - **Path:** He opens the request and sees two candidates: an internal employee he has no Manager/PP relationship to, and an external PeopleForce candidate. For the internal one, the link resolves through the Shared Link the Unit Manager generated — Marcus sees exactly the sections the UM chose to share (say, S1, S4, S12), not the person's full profile. For the external one, the link opens the pulled PeopleForce data (or, if that integration isn't live yet, an outbound link to PeopleForce itself).
  - **Climax:** Marcus can make an informed approve/reject decision on both candidates without ever gaining standing Manager access to the internal one.
  - **Resolution:** He approves one candidate with a short note; the decision and reason are recorded in request history, visible later on the request and on the internal candidate's own profile (S15).
  - **Edge case:** If Marcus rejects, he must supply a written reason — an empty rejection isn't a valid state.

## 4. Glossary

- **Access role** — One of Self, Manager, People Partner (PP), or Colleague; derived from relationships (reporting hierarchy, project assignment, PP assignment), never assigned. Shared Link and HR Admin are additionally resolvable viewer roles for profile access, distinct from this relationship-derived set.
- **Functional role** — An assigned, extensible role (Unit Manager, Delivery Manager, Project Manager, People Partner, HR Admin, or any HR-Admin-created role) that unlocks features but never widens data access on its own.
- **Section** — One of 16 named regions of an employee profile (S1–S16, e.g., Identity Card, Risks, Management Notes), each with its own per-audience access level. See `access-model.md`.
- **Manager line** — Everyone holding Manager access to a given employee: their Unit Manager, the PM/DM of their current projects, and everyone above any of those in the reporting or project chain.
- **Colleague** — Any authenticated employee holding none of Self/Manager line/PP with respect to a given profile. Sees a strict whitelist (S1, S10, S11 project name only) — **except** the Identity Card's mentor field, which is visible only to Manager line and PP, never Colleague, per the header-visibility override (FR-1).
- **Shared Link** — A time-limited, revocable, per-section-configurable, read-only view of a profile generated by a manager for someone without standing access (e.g., a DM reviewing a candidate).
- **ProjectAssignment** — The record linking an employee to a project and that project's PM/DM; the input that extends Manager access beyond the reporting hierarchy.
- **Timeline event** — A dated, typed entry on an employee's Career Timeline (S9), either system-generated (on a tracked field change) or manually added/edited by PP/UM.
- **Action item** — The platform's single task entity; created manually or generated automatically by a Form Campaign, assigned to one employee, with open/completed lifecycle and overdue derivation.
- **Form Campaign** — A record (title, description, purpose, external form link, due date) targeting a frozen audience, whose activation generates one Action Item per recipient.
- **Feedback record** — A structured note on an employee's profile (S8) with a management-only/shared-with-employee visibility flag.
- **CDS (Career Development System)** — The registry of a person's skills-matrix link, assessment log, and IDP; never the assessment mechanism itself.
- **IDP (Individual Development Plan)** — One record with description, deadline, external file link, and a self-completable checkbox.
- **Mentorship pair** — A mentor-mentee relationship with a start date, optional end date, and (on ending) mandatory closing feedback.
- **External identity mapping** — The `(system, externalId) → employeeId` table resolving a person's PeopleForce and timetracker records to their platform identity, with an explicit `supersededBy` pointer for re-hires.

## 5. Features

| # | Feature | FRs |
|---|---|---|
| 5.1 | Access Control, Roles & Employee Profile | FR-1–FR-6 |
| 5.2 | All Employees Directory & Custom Fields | FR-7–FR-12 |
| 5.3 | Self-Service | FR-13–FR-16 |
| 5.4 | Action Items and Tasks | FR-17–FR-20 |
| 5.5 | Risks and Risk Dashboard | FR-21–FR-23 |
| 5.6 | Resourcing | FR-24–FR-27 |
| 5.7 | Career Timeline | FR-28–FR-30 |
| 5.8 | CDS — Career Development System | FR-31–FR-34 |
| 5.9 | Mentorship Hub | FR-35–FR-39 |
| 5.10 | Forms & Survey Campaigns | FR-40–FR-43 |
| 5.11 | Feedback | FR-44–FR-46 |
| 5.12 | Dashboards | FR-47–FR-51 |
| 5.13 | External Integrations: Timetracker & PeopleForce | FR-52–FR-55 |

### 5.1 Access Control, Roles & Employee Profile
**Description:** The platform's core substrate — every other feature depends on it. The system resolves a viewer's access role per subject from the transitive closure of reporting and project-assignment relationships, assembles a profile from exactly the sections that role earns, and lets HR Admin extend the functional-role system without a deploy. Realizes UJ-1, UJ-2.

**Functional Requirements:**

#### FR-1: Resolve access role per subject
The system can resolve a viewer's access role (Self / Manager line / PP / Colleague / Shared Link / HR Admin) with respect to any given employee, on every request.
**Consequences (testable):**
- A manager two or more levels up the reporting chain resolves as Manager line for a report nested at any depth, with no explicit grant.
- A PM/DM resolves as Manager line for everyone on their project(s); ending a project assignment removes that access on the very next request.
- The same viewer can resolve differently against three different subjects within one session.

#### FR-2: Assemble profile by resolved section access
An authenticated viewer can open any employee's profile and see exactly the sections (S1–S16) their resolved role grants, at the granted level (R/RW), and no others.
**Consequences (testable):**
- A section the viewer has no access to is absent from the API response, not merely hidden in the UI.
- Every `—` cell in the access matrix (`access-model.md`) has a corresponding negative test.
- The profile header's mentor field (part of S1) follows the stricter rule — visible only to Manager line and PP — overriding S1's general Colleague-R grant for that one field; a Colleague viewer's response never includes it.

#### FR-3: Enforce the Colleague whitelist
A Colleague viewer sees exactly S1, S10 (including leave type), and S11 (project name only) — enforced at the API.
**Consequences (testable):**
- No export, search result, or notification surfaces a non-whitelisted field to a Colleague viewer.

#### FR-4: Management notes with dual visibility flags
A manager or PP can create a free-form management note (S7) with two independent, off-by-default flags: visible-for-employee and visible-for-PM.
**Consequences (testable):**
- UM/DM/PP see all notes regardless of flags; a PM sees only notes flagged visible-for-PM, read-only.
- An employee sees only notes flagged visible-for-employee.

#### FR-5: Define and assign functional roles at runtime
HR Admin can create a new functional role, name it, and grant it a set of feature permissions (from the minimum grantable set in `access-model.md`) entirely through the UI.
**Consequences (testable):**
- A newly created role is assignable to a person and enforced immediately, with no deploy or schema change.
- Removing a permission from a role takes effect immediately for everyone holding it.
- A role with a feature permission but no Manager/PP relationship to anyone operates within the Colleague view for that feature's data.

#### FR-6: Generate and manage a shareable profile link
A manager can generate a Shared Link to a specific employee's profile, choosing which `cfg` sections to include (S3, S7, S13 can never be shared; S2/S5/S6/S8 excluded by default).
**Consequences (testable):**
- The link defaults to 24-hour expiry, configurable at creation. *(Confirmed default — no maximum/minimum bound; treated as unbounded unless told otherwise.)*
- The link is read-only under every enabled section, with no code path returning `RW`.
- Every access via the link is logged (when, from where) and the link is revocable before expiry.

### 5.2 All Employees Directory & Custom Fields
**Description:** One list serving every audience, differing only in the data each is entitled to. Any field — built-in, derived, or defined at runtime — is filterable, sortable, and usable as a column.

**Functional Requirements:**

#### FR-7: Filter, sort, and column any field
A viewer can filter, sort by, and add as a column any built-in field, derived field (e.g., years-with-company), or custom field, scoped to their own access.
**Consequences (testable):**
- The list responds within 2 seconds at 500+ records under arbitrary filters, including permission resolution.
- A filter never allows inferring a value the viewer can't see (custom-field visibility respected in filter results).

#### FR-8: Define custom fields at runtime
HR Admin or a manager with the *manage custom fields* permission can define a new field (text/number/date/single-select/multi-select/boolean) with a visibility level (management/employee/colleague) and start using it immediately.
**Consequences (testable):**
- No schema migration or developer involvement is required between defining a field and using it as a filter/column/export field.

#### FR-9: Inline-edit list cells
A viewer with write access to a field can edit it directly from the list, writing through to the profile.
**Consequences (testable):**
- An edit attempt against a field the viewer has only `R` on is rejected server-side.

#### FR-10: Save and share views
A viewer can save a filter+column configuration as a named view, appearing as a tab; views are owned by their creator and can be shared with other managers; multiple views coexist. *(Confirmed default — see `decisions.md` appendix: filter-based views only for v1; a static manually maintained membership-list variant is out of scope unless revisited.)*

#### FR-11: Export to `.xlsx`
A viewer can export the current view to `.xlsx`, containing only the columns they're entitled to see.

#### FR-12: Colleague mode of the list
The same list, restricted to whitelist columns; a row click opens the limited (Colleague) profile view.

### 5.3 Self-Service
**Description:** An employee's own view of themselves — read access to most of their profile, write access to a defined subset, with the risk section and unflagged notes never visible regardless of role. Realizes the "As any employee" JTBD.

**Functional Requirements:**

#### FR-13: View own employment summary
An employee can view their own grade, position, seniority, employment type, and English level (read-only).

#### FR-14: Edit own contact information
An employee can edit their own personal contacts (S2), emergency contacts (S3), and residential address/place of stay, without manager or HR involvement.

#### FR-15: Upload photo and certificates
An employee can upload their own profile photo and certificates (S5). *(Confirmed default — see `decisions.md` appendix: format/size limits unspecified by source; sane conventional defaults apply unless the org states a specific requirement.)*

#### FR-16: View own timeline, leaves, projects, CDS, mentorship, feedback, notes, and action items
An employee can view their own career timeline, leaves/projects (read-only, linking out to the timetracker to manage leave), CDS section (marking their own IDP complete), mentorship status, feedback explicitly shared with them, management notes flagged visible-for-employee, and their own action items (marking them complete).
**Consequences (testable):**
- The employee never sees their own risk level (S6) or a management note without the visible-for-employee flag, under any surface.
- Leave requests/edits are never performed in-app — only displayed, with a link to the timetracker.

### 5.4 Action Items and Tasks
**Description:** The single task entity, sourced either manually or via campaign activation, visible on the profile, in self-service, and on dashboards. Feeds into UJ-1's action-item creation step.

**Functional Requirements:**

#### FR-17: Create an action item manually
A manager, PP, or permitted functional role can create an action item (title, description, assignee, due date, optional link) for anyone in their access scope.

#### FR-18: Auto-generate action items from campaign activation
Activating a Form Campaign (§5.10) generates one action item per frozen recipient, carrying the campaign's title, sender, due date, and link.

#### FR-19: Complete or cancel an action item
The assignee can mark their own item complete (recording completion date); the author can cancel it with a reason, even after losing live Manager/PP access to the assignee. *(Confirmed default — see `decisions.md` appendix: authorship is treated as a historical fact, not a live permission.)*

#### FR-20: Derive and display overdue state
An open action item past its due date renders as overdue everywhere it appears (profile, self-service, dashboards, campaign tracking table), using one consistent derivation.

### 5.5 Risks and Risk Dashboard
**Description:** A simple, retained-history risk record per employee with trend and a dedicated dashboard, scoped to Manager/PP access and never visible to the employee themself.

**Functional Requirements:**

#### FR-21: Record a risk
A manager or PP can record a risk (level: low/need attention/medium/high/leaver, description, details, date) for anyone in their access scope; history is retained, current level is the most recent record.

#### FR-22: Show trend
The profile and dashboard show a trend arrow versus the previous record, present only when the level actually changed.

#### FR-23: Risk Dashboard
A dedicated dashboard shows severity-sorted counts (medium/high/leaver emphasised), a drill-through table filterable by unit/department/project/PP/manager, scoped to the viewer's Manager/PP access, and never visible to Self.

### 5.6 Resourcing
**Description:** Request → fulfillment → approval flow for staffing, spanning internal candidates and external PeopleForce candidates, with full history retained. Realizes UJ-2.

**Functional Requirements:**

#### FR-24: Create a resourcing request
A DM, PM, or permitted role can create a request (vacancy details, expected comp level, duration, workload), optionally without a linked project.

#### FR-25: Fulfil a request with internal or external candidates
A UM can propose internal employees or attach external PeopleForce candidates, submitting one or more for DM approval.

#### FR-26: Approve or reject a proposed candidate
A DM sees proposed candidates — an internal candidate via a Shared Link (FR-6) if they lack standing access, an external candidate via pulled PeopleForce data or an outbound link — and approves or rejects each with a written reason.
**Consequences (testable):**
- A rejection without a reason is not a valid state.
- Approval never itself writes a ProjectAssignment record — assignment happens in the timetracker, reaching the profile only on the next sync (§5.13).

#### FR-27: Retain request history
Every proposal (proposed → approved/rejected, with feedback) is recorded on the request and on the candidate's own profile (S15).

### 5.7 Career Timeline
**Description:** A system-generated event log for tracked changes, with manual override for historical backfill and correction — never a separately maintained record.

**Functional Requirements:**

#### FR-28: Auto-generate timeline events
The system generates a timeline event whenever a tracked field changes: joining, grade/position/department change, FTE↔subcontractor transition, extended leave, mentorship pair start/end.

#### FR-29: Manually add, edit, or delete events
PP and UM can manually add, edit, or delete timeline events, for backfill or correction.
**Consequences (testable):**
- Deletion is soft-delete with an audit trail. *(Confirmed default — see `decisions.md` appendix; soft chosen as the safer default given access-control correctness is the platform's primary quality attribute.)*

#### FR-30: Resolve manual-vs-system conflicts
A manually edited event is never silently overwritten by a later system-generated event covering the same change window — the system write is skipped and logged.

### 5.8 CDS — Career Development System
**Description:** A registry and hub, not an assessment engine — links to an externally maintained skills matrix, logs assessments, and tracks IDP completion.

**Functional Requirements:**

#### FR-31: Resolve the current skills-matrix link
The profile shows a link to the current skills-matrix file for the person's department+position, resolved via a centrally maintained dictionary.
**Consequences (testable):**
- Updating the central matrix file changes what every affected profile's link resolves to, with no per-profile update.

#### FR-32: Maintain the assessment log
A manager or PP can add assessment log entries (date, assessor, result-file link, text conclusion). *(Confirmed default — see `decisions.md` appendix: the assessor field is a simple identifying field, free text or a person reference — implementer's choice, source doesn't specify further.)*

#### FR-33: Maintain and self-complete the IDP
A manager or PP creates/updates an IDP (description, deadline, external file link); the employee can mark their own IDP complete, recording the completion date.
**Consequences (testable):**
- A completed IDP cannot be reopened by manager/PP by default. *(Confirmed default — see `decisions.md` appendix.)*

#### FR-34: Filter by assessment recency and open IDP
All Employees supports filtering by last-assessment date (before/after/between, plus "never assessed" as a distinct selectable option) and has-open-IDP (yes/no).

### 5.9 Mentorship Hub
**Description:** Self-flagged willingness, manager/PP-driven pairing, and mandatory closing feedback on every ended pair. Status transitions follow the self-flag rule (D4).

**Functional Requirements:**

#### FR-35: Self-flag open to mentoring
An employee can flag themselves open-to-mentoring and see their own assigned mentor/mentees.

#### FR-36: Assign a mentor-mentee pair
A manager or PP browsing everyone flagged open-to-mentor can pick a mentee and create a pair; on the mentor's first pair, their status changes to "mentor."

#### FR-37: End a pair with mandatory closing feedback
A manager or PP ends a pair only by providing required closing feedback, stored on the pair record itself rather than the general Feedback entity (see D6 for the trade-off rationale); the end date is recorded, and an end event is written to the Career Timeline.

#### FR-38: Auto-transition mentor status
When a mentor's last active pair ends, their status reverts to "open to mentoring" only if their self-flag is still on; otherwise it reverts to no status.

#### FR-39: View all pairs
Manager/PP can view all mentor-mentee pairs (active and ended) with start/end dates and status.

### 5.10 Forms & Survey Campaigns
**Description:** A campaign that targets a frozen audience with a link to an externally hosted form, tracked entirely through action-item completion — the platform never reads the form itself.

**Functional Requirements:**

#### FR-40: Create a draft campaign
A PP, manager, or permitted role can create a campaign (title, description, purpose, external form link, due date) in draft state.

#### FR-41: Build the campaign audience
The creator builds the audience via the All Employees filter engine (FR-7) or a saved view (FR-10), with individual add/remove after the filter resolves, scoped to their own access.

#### FR-42: Activate a campaign
Activation atomically freezes the audience and generates one action item per recipient (FR-18); a campaign cannot be activated twice or reverted to draft, and its fields lock on activation.
**Consequences (testable):**
- A person newly matching the filter after activation is never added; someone no longer matching is never removed.
- Activation either fully succeeds for every recipient or leaves the campaign as a fully editable draft — no partial state.

#### FR-43: Track per-person completion
The creator sees a live per-recipient table: completed / not completed / overdue, using the same overdue derivation as FR-20, with completion signaled only by the recipient's own action-item completion.

### 5.11 Feedback
**Description:** Manager/PP-authored feedback records with a visibility flag, viewable over time, plus a named-colleague feedback-request flow built on Form Campaigns.

**Functional Requirements:**

#### FR-44: Record feedback with a visibility flag
A manager or PP can record feedback (subject, author, date, context, body) on a profile, defaulting to management-only visibility, flippable to shared-with-employee.
**Consequences (testable):**
- Flipping the flag makes the record immediately visible on the employee's own profile view (FR-16), with no stale grant.

#### FR-45: View feedback chronologically with period comparison
Manager/PP can view an employee's feedback records in chronological order and compare two selected periods side by side.
**Consequences (testable):**
- A period with zero records renders as an explicit zero-count comparison column — never a hidden panel, an error, or a silently omitted side of the comparison.

#### FR-46: Request feedback from named colleagues
A manager or PP can launch a "request feedback about [employee]" flow that creates a Form Campaign (FR-40) targeted at specific named colleagues rather than a filter; on completion, no Feedback record is auto-created — the requester manually authors one (FR-44) after reviewing the external form's actual responses.

### 5.12 Dashboards
**Description:** One shared engine, four role-scoped views. Grouping differs (by person vs. by project); the underlying components don't.

**Functional Requirements:**

#### FR-47: Shared dashboard engine
A single engine renders summary counters, a scoped people/project table with risk and leave status, and the viewer's own action items (sorted by due date, overdue highlighted), reused across all four dashboards.

#### FR-48: Unit Manager dashboard
Grouped by people: headcount, risk counts by level, open/overdue action items, active resourcing requests, open campaigns; table of subordinates.

#### FR-49: Delivery Manager dashboard
Grouped by project: one table per project, with a project selector (default: all projects) that filters the whole page — table and every counter — to the selected project; clearing it restores the all-projects aggregate. Includes the DM's own and their PMs' resourcing requests.

#### FR-50: Project Manager dashboard
The DM dashboard's shape, scoped to the PM's own projects.

#### FR-51: People Partner dashboard
Grouped by department or project, scoped to the PP's assigned people; never renders a resourcing block.

### 5.13 External Integrations: Timetracker & PeopleForce
**Description:** Two integrations, one of which (timetracker projects/people) is load-bearing for the access model itself, not just for display.

**Functional Requirements:**

#### FR-52: Integrate timetracker Leaves API
Leave data (S10) is pulled from the timetracker's real API.
**Consequences (testable):**
- S10 is covered by the same fail-soft display rule as FR-53: a sync outage shows "temporarily unavailable" rather than blocking the profile.

#### FR-53: Integrate timetracker Projects & People API
Project/PM/DM assignment data is pulled from the real API and feeds the ProjectAssignment record (FR-1's project-assignment leg).
**Consequences (testable):**
- During a sync outage, an unconfirmed project assignment never grants Manager access (fails safe); S10/S11 display shows "temporarily unavailable" rather than blocking the profile (fails soft) — the two axes are independent.

#### FR-54: Integrate PeopleForce for candidates and vacancies
External-candidate data (FR-25/FR-26) and vacancy source-of-truth are pulled from PeopleForce where the API allows.
**Consequences (testable):**
- Where the integration isn't completed in time, an outbound link to the candidate in PeopleForce is an accepted, sanctioned fallback — never a silent gap in the resourcing flow.

#### FR-55: Resolve cross-system identity
Every PeopleForce and timetracker record resolves to the correct platform employee via an explicit `(system, externalId) → employeeId` mapping table, never inferred from email alone, with a `supersededBy` pointer for re-hires.

**Feature-specific NFRs:**
- External integration failures degrade gracefully and never take the application down.

## 6. Non-Goals (Explicit)

- **Compensation and salary data** — no compensation section on the profile, this or any planned iteration.
- **Pre-onboarding** — creating a person before their first working day and pulling ATS data on offer acceptance is deferred.
- **Email template management** (eSender replacement) — deferred.
- **Performing competency assessments inside the system** — links to matrices and records outcomes only; assessment happens outside (§5.8).
- **Learning management (LMS) functionality** — a separate track, not duplicated here.
- **Mentorship goals, session logs, and progress tracking** — pair formation/ending/visibility only, this iteration.
- **Project allocation percentages / workload distribution modeling** — an employee's ongoing % allocation across projects is not tracked. (A resourcing request's one-time *requested* workload value, FR-24, is a distinct field.)
- **In-app/email notifications** — good-to-have, not required this iteration.
- **Analytics and reports** (beyond the Risk and campaign-completion views already specified) — good-to-have, not required this iteration.
- **Native mobile apps** — the platform is a responsive web application (confirmed).

## 7. MVP Scope

### 7.1 In Scope
All 13 features in §5 (FR-1–FR-55) constitute v1/MVP — this platform has no "post-launch" feature tier beyond the two explicitly good-to-have items below. The source spec treats this as a single delivery, not a phased rollout.

### 7.2 Out of Scope for MVP
- Notifications (§4.13 of source spec) and Analytics/Reports (§4.14 of source spec) — build only if the 13 core features are otherwise complete. `[NOTE FOR PM: these are the first candidates to reintroduce if the team has capacity after core scope.]`
- Everything listed in §6 Non-Goals.

## 8. Success Metrics

*Every metric below is validated against a deployed, demonstrable build — not a local demo on a developer's laptop.*

**Primary**
- **SM-1**: Access-matrix test coverage — every audience × relationship-path × section combination, every `—` cell, and every flag-gated record (against both gated audiences) has a passing negative test. Validates FR-1, FR-2, FR-3, FR-4.
- **SM-2**: Functional-role extensibility — a new functional role, created and granted permissions entirely through the UI, is enforced with zero code changes, demonstrated live. Validates FR-5.
- **SM-3**: Timetracker integration runs against its real API (not seed/mock data) in the demonstrated build. Validates FR-52, FR-53.
- **SM-4**: List performance — All Employees responds within 2 seconds at 500+ records under arbitrary filters, including permission resolution. Validates FR-7.
- **SM-5**: Parallel-delivery evidence — the delivery record (sprint status, commit/PR history) shows no developer-waits-on-developer defect against the wave/track plan `bmad-sprint-planning` produces.

**Secondary**
- **SM-6**: Intelligent-repository fidelity — the specs in this repository match shipped behavior at demo time (spot-checked, not just self-attested).
- **SM-7**: Accessibility — List, Profile, and Dashboard pages meet WCAG 2.1 AA (confirmed default, `decisions.md`).

**Counter-metrics (do not optimize)**
- **SM-C1**: Feature-count velocity — shipping more of §5 faster must never come at the cost of SM-1's test coverage or SM-5's parallel-delivery discipline. Per the source spec, a team that ships less but demonstrates clean process scores higher than one that ships more by abandoning it. Counterbalances SM-3/SM-4 being read as "ship fast."
- **SM-C2**: Cache-driven staleness — SM-4's performance work (any access-resolution caching, per D1) must never be optimized by weakening cache invalidation; a stale permission cache is a data leak, not a performance win. Counterbalances SM-4.

## 9. Open Questions

None blocking. All items the source material flagged for confirmation (D3, D5) were resolved this session — see `decisions.md`. Eight further low-stakes defaults (upload limits, IDP reopening, saved-view shape, CDS assessor field, timeline-deletion mode, post-access-loss cancellation, link expiry, accessibility target) ship with stated defaults, safe to revisit later if the org has a specific requirement — see `decisions.md`'s appendix.

## 10. Assumptions Index

All entries below were inferred without a source-stated answer and have since been **confirmed by the user (2026-08-21)** — none remain open.

- §2 — Platform is a responsive web application; no native mobile app.
- §3.3 (UJ-1, UJ-2) — Both journeys authored from source material rather than narrated live; confirmed representative.
- §5.1 FR-6 — Shared Link expiry has no stated maximum/minimum bound beyond the 24h default.
- §5.2 FR-10 — Saved views are filter-based only for v1; no static membership-list variant.
- §5.3 FR-15 — Photo/certificate upload limits use sane conventional defaults; source states none.
- §5.4 FR-19 — Action-item cancellation survives the author losing live access; authorship is a historical fact.
- §5.7 FR-29 — Timeline event deletion is soft-delete with audit trail.
- §5.8 FR-32 — CDS assessor field is free text or a person reference, implementer's choice.
- §5.8 FR-33 — Reopening a completed IDP is disallowed by default.
- §6 — Native mobile apps out of scope (follows from the §2 web-application decision).
- §8 SM-7 — WCAG 2.1 AA is the accessibility target.

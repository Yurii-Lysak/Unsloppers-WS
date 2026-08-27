---
title: People Management Platform
created: 2026-08-21
updated: 2026-08-26
status: final
---

# PRD: People Management Platform

## 0. Document Purpose

This PRD is for the delivery team (three developers building in parallel), the downstream BMad workflows (`bmad-ux`, `bmad-architecture`, `bmad-create-epics-and-stories`, `bmad-sprint-planning`), and whoever evaluates the finished platform. It is derived from, and stays consistent with, the project's canonical SPEC at `_bmad-output/specs/spec-people-management-platform/SPEC.md` (plus its companions `access-model.md`, `interface-contracts.md`, `decisions.md`) — that SPEC is the reviewed, preservation-validated contract, last re-derived from `docs/project-requirements-v2.md` (v1.5, 2026-08-26); this PRD narrates it into product terms (vision, journeys, feature-level requirements, metrics) without re-deciding anything it already settled. Where this document notes a confirmed default, no stakeholder was available to confirm it directly at the time it was inferred (the bootcamp brief's PM/DM/other roles are fictional, not people to consult) — each is indexed in §10. Features are grouped by the SPEC's 14 capabilities; FRs are nested under each and numbered globally (FR-1–FR-65) for stable downstream reference. Glossary terms are used verbatim throughout — no synonyms.

## 1. Process and Data Guardrails

*Constraints from `SPEC.md` that govern how this platform is built and operated, not what it does — surfaced first so they aren't rediscovered late, after Vision and Features are already being read.*

- **BMad is the mandated framework.** BMad governs planning and process for at least the start of this project. Migrating away from it must be a recorded, deliberate decision — never silent drift.
- **A foundation phase gates implementation.** Before feature work starts, the team aligns on and writes down: prototyping/design approach, best-practice research, test architecture, and named owners per topic — run in parallel across the team, not sequentially. Feature implementation does not start until this is done.
- **The intelligent repository is mandatory documentation**, not optional artifact hygiene — specs, decisions, call transcripts, external API docs, and agent rules/skills all live in it, and it is expected to stay current (SM-8).
- **Inter-team communication and status are captured** for analysis, not left in ephemeral channels.
- **Non-production environments use the seeded test population only** (v2 §4.17) — never real employee data, never real personal data in agent contexts, logs, screenshots, or the repository.
- **Grade, position, department, employment type, and employment status are temporal.** They must be modeled as time-bounded records, not scalar fields with a bolted-on audit log. This is a data-modeling requirement for `bmad-architecture` to satisfy; it's named here so it isn't rediscovered late.
- **Three independent access dimensions.** Access roles (derived), functional roles (assigned, never widen data access on their own), and full profile access (a separate, non-self-assignable grant) must never collapse into one role list. HR Admin grants no data access on its own.
- **Both dimensions must permit a write.** A section's `RW` for an audience does not itself grant every permission-gated action inside that section — the role×permission matrix must also allow the action.
- **Zero cross-person blocking.** One developer waiting on another is a process defect; cross-feature dependencies resolve only via a frozen interface contract, a stub, a spec-sanctioned fallback, or same-developer sequencing.

## 2. Vision

An engineering organization of 500+ employees, spread across departments and delivery projects, currently manages people data through a patchwork that doesn't encode who's allowed to see what about whom. The People Management Platform replaces that with a single system built around one governing idea: every piece of information about a person lives in a **section**, and every section has an explicit, enforced access rule per audience — never a blanket "manager sees everything" assumption, never a client-side hide, never a stale grant. Manager access now comes in two tiers: the **reporting line** (reports-to and department-management relations) sees the full manager section-set; the **project line** (project-assignment only) sees a deliberately narrower set — no personal contacts, no emergency contacts, documents limited to CV and certificates. A unit manager opens a profile and sees exactly what their role and relationship to that person earn them; a colleague sees a strict whitelist; an HR Admin can invent a whole new functional role from the UI on a Tuesday afternoon and have it enforced everywhere by Wednesday — without thereby gaining read access to anyone's data. Full profile access exists as its own grant, separate from every functional role.

*Confirmed: this is a web application, responsive across desktop/tablet/mobile browsers — no native client is in scope. Authentication is the platform's own implementation over the seeded population; no SSO this iteration.*

This is also, deliberately, the vehicle for an AI-native SDLC bootcamp. The platform's functional scope matters, but the source spec is explicit that *how* it gets built — spec-driven, genuinely parallelized across three developers with zero cross-person blocking, documented in an intelligent repository — is graded above the shipped feature count. This PRD, and everything built from it, is judged as much on process fidelity as on working software.

## 3. Target User

### 3.1 Jobs To Be Done

- **As a Unit Manager**, I need to see my people's risk, leave, and project status in one place, and act on it (assign mentors, raise risks, fulfill resourcing requests routed to my department) without waiting on HR to pull data for me.
- **As a Delivery/Project Manager**, I need visibility into everyone on my projects — even people I don't formally manage — the moment they're assigned, and that visibility to disappear when they roll off (within the platform's stated access-control windows).
- **As a People Partner**, I need full HR-line visibility into my assigned people's sensitive sections (risk, notes, feedback, CDS) without holding any resourcing power I shouldn't have.
- **As an HR Admin**, I need to extend the platform to new organizational needs (a new functional role, a new custom field, a new department) myself, without filing a ticket and waiting on engineering — and without automatically gaining read access to employee data I wasn't already entitled to see.
- **As any employee**, I need self-service over my own data (contacts, photo, IDP, mentorship flag) and a guarantee that sensitive things about me (risk level, unflagged notes, leave balances) stay invisible to me and to my colleagues.
- **As a Colleague of someone I don't manage**, I need enough visibility (name, role, project, leave dates — but not leave type) to work with them day-to-day, and nothing more.

### 3.2 Non-Users (v1)

- External recruiters or candidates — they exist only as data referenced *into* the platform (PeopleForce candidate ID + link on resourcing proposals), never as platform users themselves.
- Anyone outside the org's employee/contractor population — there is no public-facing surface.
- Compensation/payroll administrators — the platform carries no compensation data on profiles (§6); a resourcing request's expected-comp band is a separate, scoped object (FR-28).

### 3.3 Key User Journeys

*UJ-1 and UJ-2 were authored from the source spec's described relationships and rules rather than narrated live (no PM/DM/other role-holder was available to consult) — confirmed as representative by the user (2026-08-21). Resourcing mechanics in UJ-2 reflect the v2 auto-generated Shared Link flow. UJ-3 was added during this update's finalize review — same authored-not-narrated caveat — to dramatize the Vision's headline functional-role-creation claim and give `bmad-ux` a concrete scenario for the roles-admin screen.*

- **UJ-1. Daniela catches a flight risk before it becomes a resignation.**
  - **Persona + context:** Daniela is a People Partner responsible for ~40 people across two departments; she checks in on her portfolio most mornings.
  - **Entry state:** Authenticated, viewing the PP Dashboard.
  - **Path:** She sees a counter showing 3 people at active risk (any level above low), up from 2 last week. She drills through to the filtered Risk Dashboard table, sorted by severity, and clicks the newest high-risk entry — someone she doesn't directly manage but is PP for. Her PP access opens their full profile, including S7 management notes flagged visible-for-PM from that person's PM. She reads the note, sees a pattern (missed 1:1s, a comment about workload), and creates a management note of her own plus an action item for herself to schedule a conversation.
  - **Climax:** The trend arrow and the note context together tell her *why* the risk escalated, not just that it did — she doesn't have to ask the PM separately.
  - **Resolution:** She has a concrete next step logged (the action item) and a documented note trail for her next 1:1.
  - **Edge case:** If the note had no PM-visibility flag set, she'd still see it (PP has unconditional RW on S7) — but a PM checking the same profile without the flag would see nothing.

- **UJ-2. Marcus reviews an external candidate he's never met.**
  - **Persona + context:** Marcus is a Delivery Manager who requested a resourcing need two weeks ago; a Unit Manager has just proposed candidates.
  - **Entry state:** Authenticated, notified (or checking) that his resourcing request has proposals.
  - **Path:** He opens the request and sees two candidates: an internal employee he has no Manager/PP relationship to, and an external candidate identified by PeopleForce ID + link. He also sees the expected comp band the routed UM and DM can see (FR-28) — never the PP, never on the candidate's own profile. For the internal one, submitting the proposal automatically generated a Shared Link naming Marcus as sole recipient — scoped to S1/S4/S11/S12 and S5 (CV+certs only), bound to the request's lifetime. For the external one, he follows the stored PeopleForce link (full data pull is not required).
  - **Climax:** Marcus can make an informed approve/reject decision on both candidates without ever gaining standing Manager access to the internal one.
  - **Resolution:** He approves one candidate with a short note; the decision fills one headcount slot and is recorded in request history, visible later on the request and on the internal candidate's own profile (S15).
  - **Edge case:** If Marcus rejects, he must supply a written reason — an empty rejection isn't a valid state. The auto-generated shared link dies when the request is decided.

- **UJ-3. Priya stands up a new functional role without filing a ticket.**
  - **Persona + context:** Priya is an HR Admin; a new "Learning Champion" initiative needs someone per department who can create CDS assessment-log entries but nothing else new.
  - **Entry state:** Authenticated, on the Functional Roles admin screen.
  - **Path:** She creates a role named "Learning Champion," and from the minimum grantable permission set she checks exactly one box: *maintain CDS records*. She saves — no deploy, no ticket to engineering. She assigns the role to five people, one per department.
  - **Climax:** A Learning Champion who has no Manager/PP relationship to someone outside their own department opens that person's profile and sees only the Colleague view — the new role unlocked the CDS-editing *feature* but granted no additional *data* access, exactly as FR-5 requires.
  - **Resolution:** Within the same session, all five assignees can add assessment-log entries for people already in their access scope; Priya later removes the permission from the role, and it stops working for all five immediately, with no re-login.
  - **Edge case:** If Priya tries to grant the role *manage custom fields* and later revokes it, anyone still mid-edit on a custom-field definition is rejected on their next write, not merely blocked from opening a new one.

## 4. Glossary

- **Access role** — One of Self, Reporting line, Project line, People Partner (PP), or Colleague; derived from relationships (reporting hierarchy, department management, project assignment, PP assignment), never assigned. Shared Link and Full profile access are additionally resolvable viewer roles for profile access, distinct from this relationship-derived set.
- **Reporting line** — Manager access via *reports-to* or *manages-the-department* (including nested departments), and everyone above through those paths. Grants the full manager section-set.
- **Project line** — Manager access via *manages-the-project* only (PM/DM of the person's projects, and everyone above through the project path only). Strictly narrower: no S2, no S3, S5 limited to CV and certificates.
- **Functional role** — An assigned, extensible role (Unit Manager, Delivery Manager, Project Manager, People Partner, HR Admin, or any HR-Admin-created role) that unlocks features but never widens data access on its own. HR Admin grants platform administration only — not data access.
- **Full profile access** — A separate grant (not a functional role) that sees every section of every profile. Granted only by an existing holder, never self-assignable, seeded at deployment, never revocable down to zero holders.
- **Section** — One of 16 named regions of an employee profile (S1–S16, e.g., Identity Card, Risks, Management Notes), each with its own per-audience access level. See `access-model.md`.
- **Access switch** — One of four organizational-relationship fields (employee's manager, employee's PP, employee's department, department's manager) that change access immediately when edited. Writable only through the dedicated *change organisational relationships* permission on a dedicated screen — never inline, never self-assignable, always journaled. *(The permission name keeps `access-model.md`'s own spelling verbatim as a system identifier; it is not prose and is intentionally not Americanized like the rest of this document.)*
- **Colleague** — Any authenticated employee holding none of Self/Reporting line/Project line/PP with respect to a given profile. Sees a strict whitelist (S1, S10 dates only with leave type hidden, S11 project name only) — **except** the Identity Card's mentor field (visible only to reporting/project line and PP, per D5) and the **campaign-sender exception** (a campaign creator sees that campaign's recipients and action-item statuses only).
- **Shared Link** — A time-limited (default 24h, except resourcing-generated links bound to request lifetime), revocable, named-recipient-only, read-only view of a profile. Works only for an authenticated, explicitly-named recipient — never anonymous. Its exposed sections continuously re-clamp to the creator's *currently held* access on every view, and it is explicitly revoked as part of the creator's departure cascade (FR-6, FR-63).
- **ProjectAssignment** — The record linking an employee to a project and that project's PM/DM; the input that extends Project line access beyond the reporting hierarchy. Written only by timetracker sync (§5.13), never by resourcing approval.
- **Timeline event** — A dated, typed entry on an employee's Career Timeline (S9), either system-generated (on a tracked field change) or manually added/edited by whoever holds *edit the career timeline* permission. Leaving the company is never a timeline event — it lives in employment status (§5.14).
- **Action item** — The platform's single task entity; created manually or generated automatically by a Form Campaign, assigned to one employee, with open/completed lifecycle and overdue derivation.
- **Form Campaign** — A record (title, description, purpose, external form link, due date) targeting a frozen audience, whose activation generates one Action Item per recipient. The sole mechanism for distributing any external form.
- **Feedback record** — A structured note on an employee's profile (S8) with a management-only/shared-with-employee visibility flag. Joining-interview feedback is a feedback record, not a document.
- **CDS (Career Development System)** — The registry of a person's skills-matrix link, assessment log, and IDP; never the assessment mechanism itself. Matrix dictionary keys off the department entity, not a free-text string.
- **IDP (Individual Development Plan)** — One record with description, deadline, external file link, and a self-completable checkbox.
- **Mentorship pair** — A mentor-mentee relationship with a start date, optional end date, and (on ending) mandatory closing note stored on the pair record — readable by reporting/project line and PP only, never by mentor, mentee, or colleagues.
- **Employment status** — A time-bounded fact per employee (`active` | `dismissed`); the single source every consumer reads for "has this person left." Distinct from the `leaver` risk-level prediction.
- **Departure (departure cascade)** — The permission-gated act of recording an employee's `dismissed` status with an effective date and reason (FR-62), and the atomic, idempotent set of consequences it triggers on that date: read-only profile, default-list exclusion, auto-cancelled open action items, auto-ended mentorship pairs, revoked shared links, account deactivation, and full access revocation (FR-63). Blocked while the departing person still manages/partners anyone or is the sole full-profile-access holder, until re-parented/transferred (FR-62).
- **External identity mapping** — The `(system, externalId) → employeeId` table resolving a person's timetracker and optional PeopleForce records to their platform identity, with an explicit `supersededBy` pointer for re-hires. The seeded platform record is the primary identity anchor.

## 5. Features

| # | Feature | FRs |
|---|---|---|
| 5.1 | Access Control, Roles & Employee Profile | FR-1–FR-9 |
| 5.2 | All Employees Directory & Custom Fields | FR-10–FR-15 |
| 5.3 | Self-Service | FR-16–FR-19 |
| 5.4 | Action Items and Tasks | FR-20–FR-23 |
| 5.5 | Risks and Risk Dashboard | FR-24–FR-27 |
| 5.6 | Resourcing | FR-28–FR-33 |
| 5.7 | Career Timeline | FR-34–FR-36 |
| 5.8 | CDS — Career Development System | FR-37–FR-40 |
| 5.9 | Mentorship Hub | FR-41–FR-45 |
| 5.10 | Forms & Survey Campaigns | FR-46–FR-49 |
| 5.11 | Feedback | FR-50–FR-52 |
| 5.12 | Dashboards | FR-53–FR-57 |
| 5.13 | External Integrations: Timetracker & PeopleForce | FR-58–FR-60 |
| 5.14 | Employment Status & Departure Lifecycle | FR-61–FR-65 |

### 5.1 Access Control, Roles & Employee Profile
**Description:** The platform's core substrate — every other feature depends on it. The system resolves a viewer's access role per subject from the transitive closure of reporting, department-management, and project-assignment relationships (as two tiers: reporting line vs. project line), assembles a profile from exactly the sections that role earns, and lets HR Admin extend the functional-role system without a deploy. Full profile access operates as its own grant. Realizes UJ-1, UJ-2, UJ-3.

**Functional Requirements:**

#### FR-1: Resolve access role per subject
The system can resolve a viewer's access role (Self / Reporting line / Project line / PP / Colleague / Shared Link / Full profile access) with respect to any given employee, on every request.
**Consequences (testable):**
- A manager two or more levels up the reporting chain resolves as Reporting line for a report nested at any depth, with no explicit grant.
- A department manager sees everyone in their department and nested sub-departments.
- A PM/DM resolves as Project line for everyone on their project(s); the project line is narrower than the reporting line (FR-2).
- Platform-owned relationship changes take effect on the next request; project-assignment changes take effect within 15 minutes (FR-58).
- The same viewer can resolve differently against three different subjects within one session.
- When a viewer resolves to more than one access role for the same subject (e.g., both Project line and PP), effective access for each section is the **union** — the least-restrictive access among every resolved role's grant for that section, never a single ranked precedence. *(D13, `decisions.md`)*

#### FR-2: Assemble profile by resolved section access
An authenticated viewer can open any employee's profile and see exactly the sections (S1–S16) their resolved role grants, at the granted level (R/RW), and no others.
**Consequences (testable):**
- A section the viewer has no access to is absent from the API response, not merely hidden in the UI.
- Every `—` cell and every narrowed project-line cell in the access matrix (`access-model.md`) has a corresponding negative test.
- The profile header's mentor field (part of S1) follows the stricter rule — visible only to reporting/project line and PP — overriding S1's general Colleague-R grant for that one field.
- Project line viewers never receive S2, S3, or full S5 — only CV and certificates within S5.

#### FR-3: Enforce the Colleague whitelist and campaign-sender exception
A Colleague viewer sees exactly S1, S10 (dates only — leave type hidden), and S11 (project name only) — enforced at the API.
**Consequences (testable):**
- No export, search result, or notification surfaces a non-whitelisted field to a Colleague viewer.
- A form-campaign creator sees, for that campaign only, each recipient by name plus that campaign's action-item status — nothing else from S14, nothing from any other section. The exception ends when the campaign closes.

#### FR-4: Management notes with dual visibility flags
A manager or PP can create a free-form management note (S7) with two independent, off-by-default flags: visible-for-employee and visible-for-PM.
**Consequences (testable):**
- Reporting line, DMs, and PP see all notes regardless of flags; a PM sees only notes flagged visible-for-PM, read-only.
- An employee sees only notes flagged visible-for-employee.

#### FR-5: Define and assign functional roles at runtime
HR Admin can create a new functional role, name it, and grant it a set of feature permissions (from the minimum grantable set in `access-model.md`) entirely through the UI.
**Consequences (testable):**
- A newly created role is assignable to a person and enforced immediately, with no deploy or schema change.
- HR Admin itself grants no data access — only platform administration.
- A write requires both section access and the relevant feature permission.
- Removing a permission from a role takes effect immediately for everyone holding it.
- A role with a feature permission but no Manager/PP relationship to anyone operates within the Colleague view for that feature's data (subject to the campaign-sender exception).

#### FR-6: Generate and manage a Shared Link
A manager can generate a Shared Link to a specific employee's profile, choosing which `cfg` sections to include. The never-share set is {S3, S7, S13, S14}; sensitive sections {S2, S5, S6, S8} are excluded by default and must be explicitly re-enabled; only S1 is on by default.
**Consequences (testable):**
- The link works only for an authenticated, explicitly-named recipient — never anonymous.
- Manual links default to 24-hour expiry, configurable at creation. Resourcing-generated links (FR-31) live until the request is decided instead.
- The link is read-only under every enabled section, with no code path returning `RW`.
- The creator's access is re-checked on **every** view — a continuous, per-section re-clamp to whatever access the creator currently holds, not a one-time or alive/dead check. If the creator's access narrows (e.g., Reporting line drops to Project line) short of ending outright, any section the narrower tier no longer grants stops rendering for the recipient on the very next view. *(D14, `decisions.md`)*
- Revocation rights follow whoever currently holds Manager/PP access over the subject (full-access holders as backstop) — never the original creator, so a link is never left with nobody able to revoke it.
- Every access via the link is logged and the link is revocable before expiry.
- Every shared link a departing employee created is explicitly revoked as its own step of the departure cascade (FR-63) — not left to the re-clamp rule above alone. *(D17)*

#### FR-7: Change organisational relationships
Whoever holds the *change organisational relationships* permission can change an employee's manager, PP, department, or a department's manager on a dedicated screen.
**Consequences (testable):**
- These fields display in S1 but are never writable through S1's general edit path or the All Employees inline grid (FR-11).
- No self-assignment is possible regardless of held permissions.
- Every change is recorded in the relationship and grant journal (FR-9).
- A department's manager may never be a current member of that department or any nested sub-department — rejected server-side, closing off self-managed Reporting-line access as a backdoor around "risk never visible to self" (FR-19/FR-27). *(D15, `decisions.md`)*
- A manager change is rejected if the proposed new manager is already a descendant of the employee in the reporting chain — no reporting cycles, since a cycle would break the transitive-closure walk FR-1 depends on. *(D15)*
- A concurrent conflicting write to the same access-switch field is rejected with a conflict error, never silently overwritten — the losing write never reaches the journal (FR-9) as if it had applied. *(D15)*
- Clearing a department's manager through this direct-edit path (as distinct from a departure, FR-62) is blocked unless a replacement manager is designated in the same change — a department is never left without Reporting-line coverage. *(D15)*

#### FR-8: Grant and manage full profile access
An existing full-profile-access holder can grant or revoke full profile access to another person.
**Consequences (testable):**
- No self-assignment. The first holder is seeded at deployment. Removing the last holder is blocked — the holder count is re-checked **at commit time**, not from an earlier read, so two holders revoking each other at the same instant can never race the count to zero. *(D16, `decisions.md`)*
- Every grant and revocation is journaled (FR-9).
- Full profile access is independent of every functional role, including HR Admin.
- Recording a departure for the sole remaining holder is blocked by the same never-zero gate — see FR-62.
- A departing non-sole holder's grant is explicitly revoked and journaled as its own named step of the departure cascade (FR-63) — the same treatment as shared links, never left implicit in a generic "access ends" clause.

#### FR-9: Maintain the relationship and grant journal
The system records manager changes, PP changes, department changes, department-manager changes, full-access grants/revocations, and shared-link accesses in a dedicated journal (not the general audit log).
**Consequences (testable):**
- Each entry holds actor, subject, before/after values, and timestamp.
- Readable by full-access holders and by the subject's current manager and PP.

### 5.2 All Employees Directory & Custom Fields
**Description:** One list serving every audience, differing only in the data each is entitled to. Any field — built-in (including employment status), derived, or defined at runtime — is filterable, sortable, and usable as a column.

**Functional Requirements:**

#### FR-10: Filter, sort, and column any field
A viewer can filter, sort by, and add as a column any built-in field, derived field (e.g., years-with-company), or custom field, scoped to their own access.
**Consequences (testable):**
- The list responds within 2 seconds at 500+ records under arbitrary filters, including permission resolution.
- A filter never allows inferring a value the viewer can't see (custom-field visibility respected in filter results).
- Dismissed employees are excluded from the default view but remain filterable (FR-64).

#### FR-11: Define custom fields at runtime
Whoever holds the *manage custom fields* permission can define a new field (text/number/date/single-select/multi-select/boolean) with a visibility level (management/employee/colleague) and start using it immediately.
**Consequences (testable):**
- No schema migration or developer involvement is required between defining a field and using it as a filter/column/export field.

#### FR-12: Inline-edit list cells
A viewer with write access to a field can edit it directly from the list, writing through to the profile.
**Consequences (testable):**
- An edit attempt against a field the viewer has only `R` on is rejected server-side.
- Manager, PP, and department fields are never inline-editable — they route only through FR-7.

#### FR-13: Save and share views
A viewer can save a filter+column configuration as a named view, appearing as a tab; views are owned by their creator and can be shared with other managers; multiple views coexist. *(Confirmed default — see `decisions.md` appendix: filter-based views only for v1; a static manually maintained membership-list variant is out of scope unless revisited.)*
**Consequences (testable):**
- A shared view is read-only for non-creators (only the creator can edit or delete it) and always re-resolves its results to the *viewer's own* access scope, never the creator's — the same saved filter can return different rows to different recipients. *(Confirmed default — source doesn't specify; the alternative, leaking the creator's row set to a narrower-access recipient, would be a data leak.)*
- On the creator's departure (FR-62), a saved view becomes **ownerless**: it stays visible/usable by anyone it was shared with, but can't be edited or deleted until an HR Admin (or whoever holds *manage custom fields*) explicitly adopts it as the new owner. *(Confirmed default — see `decisions.md` appendix; a saved view has no profile-scoped manager/PP to transfer to, unlike a subject's own records.)*

#### FR-14: Export to `.xlsx`
A viewer can export the current view to `.xlsx`, containing only the columns they're entitled to see.

#### FR-15: Colleague mode of the list
The same list, restricted to whitelist columns; a row click opens the limited (Colleague) profile view.

### 5.3 Self-Service
**Description:** An employee's own view of themselves — read access to most of their profile, write access to a defined subset, with risk section, unflagged notes, and leave balances never visible regardless of role.

**Functional Requirements:**

#### FR-16: View own employment summary
An employee can view their own grade, position, seniority, employment type, and English level (read-only).

#### FR-17: Edit own contact information
An employee can edit their own personal contacts (S2), emergency contacts (S3), and residential address/place of stay, without manager or HR involvement.

#### FR-18: Upload photo and certificates
An employee can upload their own profile photo and certificates (S5). *(Confirmed default — see `decisions.md` appendix: format/size limits unspecified by source; sane conventional defaults apply unless the org states a specific requirement.)*

#### FR-19: View own timeline, leaves, projects, CDS, mentorship, feedback, notes, and action items
An employee can view their own career timeline, leaves/projects (full own leave detail per S10, read-only — linking out to the timetracker to manage leave and balances), CDS section (marking their own IDP complete), mentorship status, feedback explicitly shared with them, management notes flagged visible-for-employee, and their own action items (marking them complete).
**Consequences (testable):**
- The employee never sees their own risk level (S6) or a management note without the visible-for-employee flag, under any surface.
- Leave balances are never rendered in-platform — the timetracker is the only place balances live or change.

### 5.4 Action Items and Tasks
**Description:** The single task entity, sourced either manually or via campaign activation, visible on the profile, in self-service, and on dashboards. Feeds into UJ-1's action-item creation step.

**Functional Requirements:**

#### FR-20: Create an action item manually
Whoever holds the relevant permission can create an action item (title, description, assignee, due date, optional link) for anyone in their access scope.
**Consequences (testable):**
- An attempt to create an action item for someone outside the creator's access scope is rejected server-side, regardless of which functional permission the creator holds.

#### FR-21: Auto-generate action items from campaign activation
Activating a Form Campaign (§5.10) generates one action item per frozen recipient, carrying the campaign's title, sender, due date, and link.

#### FR-22: Complete or cancel an action item
The assignee can mark their own item complete (recording completion date); the author can cancel it with a reason, even after losing live Manager/PP access to the assignee. *(Confirmed default — see `decisions.md` appendix: authorship is treated as a historical fact, not a live permission.)* On departure (FR-63), open items auto-cancel as "cancelled — departed."
**Consequences (testable):**
- Visibility follows S14, with the campaign-sender exception (FR-3) as the sole widening.
- A cancellation write against an item already in a cancelled state is a no-op, regardless of whether the author's manual cancellation or FR-63's departure-triggered auto-cancellation arrives second — neither ever overwrites the other's reason/cancelled-by field. *(D17, `decisions.md`)*

#### FR-23: Derive and display overdue state
An open action item past its due date renders as overdue everywhere it appears (profile, self-service, dashboards, campaign tracking table), using one consistent derivation.

### 5.5 Risks and Risk Dashboard
**Description:** A simple, retained-history risk record per employee with trend and a dedicated dashboard, scoped to Manager/PP access and never visible to the employee themself. Realizes UJ-1.

**Functional Requirements:**

#### FR-24: Record a risk
A manager or PP can record a risk (level: low/need attention/medium/high/leaver, description, details, date) for anyone in their access scope; history is retained, current level is the most recent record. A risk can never be closed — it only moves between levels, including back down to low.
**Consequences (testable):**
- `leaver` is a prediction about someone still working and must never be conflated with the `dismissed` employment-status fact (FR-61).

#### FR-25: Show trend
The profile and dashboard show a trend arrow versus the previous record, present only when the level actually changed.

#### FR-26: Define active risk
"Active" means any level above low. Every dashboard counter and active-risk figure uses this definition exclusively.
**Consequences (testable):**
- A person at `low` is never counted as active risk anywhere.

#### FR-27: Risk Dashboard
A dedicated dashboard shows severity-sorted active-risk counts (medium/high/leaver emphasized), a drill-through table filterable by department/project/PP/manager, scoped to the viewer's Manager/PP access, and never visible to Self.

### 5.6 Resourcing
**Description:** Request → fulfillment → approval flow for staffing, spanning internal candidates and external PeopleForce-referenced candidates, with full history retained. Realizes UJ-2. The resourcing request is a platform-owned entity — never synchronized with PeopleForce.

**Functional Requirements:**

#### FR-28: Create a resourcing request
A DM, PM, or permitted role can create a request (vacancy details, expected comp band, duration, workload, headcount defaulting to 1, required department for UM routing). The project reference is optional.
**Consequences (testable):**
- The comp band is visible only to the author, the routed UM, and the reviewing DM — never PP, never in S15, a shared link, or an export.
- An unattached request with no project is valid and appears in the Unassigned bucket (FR-55).

#### FR-29: Fulfill a request with internal or external candidates
A UM can propose internal employees or attach external candidates by PeopleForce candidate ID + link (full PeopleForce data pull not required), submitting one or more for DM approval.
**Consequences (testable):**
- Submitting an internal employee automatically generates a Shared Link (FR-6) naming the reviewing DM as sole recipient, scoped to S1/S4/S11/S12 and S5 (CV+certs only), with S6 optional at UM's choice, S2/S3/S7/S8 never included, bound to the request's lifetime.

#### FR-30: Approve or reject a proposed candidate
A DM sees proposed candidates via the auto-generated Shared Link (internal) or stored PeopleForce link (external) and approves or rejects each with a written reason.
**Consequences (testable):**
- A rejection without a reason is not a valid state.
- Approval fills one headcount slot; it never writes a ProjectAssignment — assignment happens in the timetracker, reaching the profile only on the next sync (§5.13).
- An approval can later be reversed to rejected with a written reason, freeing the headcount slot it had filled — otherwise a candidate found ineligible after approval leaves the slot permanently marked filled with no correction route. *(D18, `decisions.md`)*
- A request awaiting a decision from a now-departed DM auto-routes to a live DM on the same project if one is resolvable, or is flagged for reassignment to a full-access holder as backstop. *(D18)*

#### FR-31: Close a resourcing request
Only the DM's explicit close (permission-gated) ends the request, successfully or unsuccessfully — it never auto-closes when the last headcount slot fills.
**Consequences (testable):**
- Headcount filled/remaining is always accurate. Auto-generated shared links die when the request is decided.
- A headcount edit that would drop the total below the already-approved/filled count is rejected outright — the DM must reverse enough approvals (FR-30) first. *(D18)*
- Approving a new candidate is blocked once remaining headcount is zero, until the DM raises headcount or closes the request — a request never over-fulfills beyond its stated headcount. *(D18)*

#### FR-32: Retain request history
Every proposal (proposed → approved/rejected, with feedback) is recorded on the request and on the candidate's own profile (S15).

#### FR-33: Route by department
Every request requires a department that determines which UM fulfills it. "Unit" means department only — there is no separate unit entity.
**Consequences (testable):**
- Routing re-resolves to the department's **current** UM on every view, never pinned to whoever held that role at request-creation time. *(D18, `decisions.md`)*
- If re-resolution finds no successor UM for the department (e.g., temporarily headless), a pending proposal authored by the now-departed UM is flagged for reassignment rather than left silently orphaned. *(D18)*

### 5.7 Career Timeline
**Description:** A system-generated event log for tracked changes, with manual override for historical backfill and correction — never a separately maintained record. Leaving the company is never a timeline event.

**Functional Requirements:**

#### FR-34: Auto-generate timeline events
The system generates a timeline event whenever a tracked field changes: joining, grade/position/department change, FTE↔subcontractor transition, extended leave, mentorship pair start/end.
**Consequences (testable):**
- Every one of the six tracked change types produces exactly one system-generated event, via a single event-writer shared across the codebase, never a per-feature bespoke write.

#### FR-35: Manually add, edit, or delete events
Whoever holds the *edit the career timeline* permission can manually add, edit, or delete timeline events, for backfill or correction.
**Consequences (testable):**
- Deletion is soft-delete with an audit trail. *(Confirmed default — see `decisions.md` appendix.)*
- No departure ever appears as a timeline event.

#### FR-36: Resolve manual-vs-system conflicts
A manually edited event is never silently overwritten by a later system-generated event covering the same change window — the system write is skipped and logged.

### 5.8 CDS — Career Development System
**Description:** A registry and hub, not an assessment engine — links to an externally maintained skills matrix, logs assessments, and tracks IDP completion.

**Functional Requirements:**

#### FR-37: Resolve the current skills-matrix link
The profile shows a link to the current skills-matrix file for the person's department entity + position, resolved via a centrally maintained dictionary keyed off the department entity (not a free-text string).
**Consequences (testable):**
- Updating the central matrix file or renaming its owning department changes what every affected profile's link resolves to, with no per-profile update.

#### FR-38: Maintain the assessment log
Whoever holds the *maintain CDS records* permission can add assessment log entries (date, assessor, result-file link, text conclusion). *(Confirmed default — see `decisions.md` appendix: the assessor field is a simple identifying field, free text or a person reference — implementer's choice.)*

#### FR-39: Maintain and self-complete the IDP
A manager or PP creates/updates an IDP (description, deadline, external file link); the employee can mark their own IDP complete, recording the completion date.
**Consequences (testable):**
- A completed IDP cannot be reopened by manager/PP by default. *(Confirmed default — see `decisions.md` appendix.)*

#### FR-40: Filter by assessment recency and open IDP
All Employees supports filtering by last-assessment date (before/after/between, plus "never assessed" as a distinct selectable option) and has-open-IDP (yes/no).

### 5.9 Mentorship Hub
**Description:** Self-flagged willingness, permission-gated pairing, and mandatory closing notes on every ended pair. Status transitions follow the self-flag rule (D4).

**Functional Requirements:**

#### FR-41: Self-flag open to mentoring
An employee can flag themselves open-to-mentoring and see their own assigned mentor/mentees.
**Consequences (testable):**
- Flagging or unflagging is entirely self-service — no manager, PP, or permission gate stands between an employee and their own open-to-mentor flag.

#### FR-42: Assign a mentor-mentee pair
Whoever holds the *assign and end mentorships* permission, browsing a company-wide pool of everyone flagged open-to-mentor (identity-card data + flag only, never S13), can pick a mentee scoped to their own access and create a pair; on the mentor's first pair, their status changes to "mentor."
**Consequences (testable):**
- Creating a pair is rejected server-side, at write time, unless the prospective mentor's open-to-mentoring flag is currently on — never relying on the pool UI alone to have filtered them out, since a stale UI or a direct API call must not be able to assign a non-consenting employee as a mentor. *(D20, `decisions.md`)*

#### FR-43: End a pair with mandatory closing note
Whoever holds the *assign and end mentorships* permission ends a pair only by providing a required closing note, stored on the pair record — readable by reporting/project line and PP only, never by mentor, mentee, or colleagues; the end date is recorded, and an end event is written to the Career Timeline. *(See D6 in `decisions.md` for the decoupling rationale.)* Departure-triggered auto-close (FR-63) supplies a system-generated note and bypasses this gate.
**Consequences (testable):**
- Ending a pair is rejected — a conflict, not a silent overwrite of the closing note — if the pair's status has already transitioned to ended by a concurrent request. This is a *rejection*, deliberately unlike FR-63's departure-triggered auto-close, which is an idempotent *no-op*: an automated cascade must never fail a transaction on a race it caused, but two humans racing to close the same pair should each learn what happened. *(D20, `decisions.md`)*

#### FR-44: Auto-transition mentor status
When a mentor's last active pair ends, their status reverts to "open to mentoring" only if their self-flag is still on; otherwise it reverts to no status. Un-flagging mid-pair removes the person from the pool without touching the active pair.
**Consequences (testable):**
- The self-flag's value is always read fresh at the moment the last pair ends, never cached from when the pair was created — flipping the flag on and off multiple times while a pair is active never changes the pair itself, only the reversion outcome once it ends.

#### FR-45: View all pairs and show mentor in header
Manager/PP can view all mentor-mentee pairs (active and ended) with start/end dates and status. The mentor appears in every profile header alongside manager and PP, visible only per D5's stricter rule.

### 5.10 Forms & Survey Campaigns
**Description:** A campaign that targets a frozen audience with a link to an externally hosted form, tracked entirely through action-item completion — the platform never reads the form itself. The sole mechanism for distributing any external form.

**Functional Requirements:**

#### FR-46: Create a draft campaign
A PP, manager, or permitted role can create a campaign (title, description, purpose, external form link, due date) in draft state.
**Consequences (testable):**
- A draft's fields remain fully editable, and the campaign generates no action items, until activation (FR-48).

#### FR-47: Build the campaign audience
The creator builds the audience via the All Employees filter engine (FR-10) or a saved view (FR-13), with individual add/remove after the filter resolves, scoped to their own access.
**Consequences (testable):**
- The audience-building step can never add a person the creator's own access scope wouldn't otherwise let them see, even via individual add.

#### FR-48: Activate a campaign
Activation atomically freezes the audience and generates one action item per recipient (FR-21); a campaign cannot be activated twice or reverted to draft, and its fields lock on activation.
**Consequences (testable):**
- A person newly matching the filter after activation is never added; someone no longer matching is never removed.
- Activation either fully succeeds for every recipient or leaves the campaign as a fully editable draft — no partial state.

#### FR-49: Track per-person completion
The creator sees a live per-recipient table: completed / not completed / overdue, using the same overdue derivation as FR-23, with completion signaled only by the recipient's own action-item completion.

### 5.11 Feedback
**Description:** Permission-gated feedback records with a visibility flag, viewable chronologically and filterable by period, plus a named-colleague feedback-request flow built on Form Campaigns.

**Functional Requirements:**

#### FR-50: Record feedback with a visibility flag
Whoever holds the *create feedback* permission can record feedback (subject, author, date, context, body) on a profile, defaulting to management-only visibility, flippable to shared-with-employee — including joining-interview feedback (a feedback record, not a document).
**Consequences (testable):**
- Flipping the flag makes the record immediately visible on the employee's own profile view (FR-19), with no stale grant.
- Joining-interview feedback never appears in S5.

#### FR-51: View feedback chronologically with period filtering
Manager/PP can view an employee's feedback records in chronological order and filter by period. There is no period-over-period comparison — the body is free text with nothing structural to compare.

#### FR-52: Request feedback from named colleagues
A manager or PP can launch a "request feedback about [employee]" flow that creates a Form Campaign (FR-46) targeted at specific named colleagues rather than a filter; on completion, no Feedback record is auto-created — the requester manually authors one (FR-50) after reviewing the external form's actual responses.

### 5.12 Dashboards
**Description:** One shared engine, four role-scoped views. Grouping differs (by person vs. by project); the underlying components don't. Realizes UJ-1 (PP dashboard as entry point).

**Functional Requirements:**

#### FR-53: Shared dashboard engine
A single engine renders summary counters, a scoped people/project table with active risk (FR-26) and leave status, and the viewer's own action items (sorted by due date, overdue highlighted), reused across all four dashboards.
**Consequences (testable):**
- All four role-scoped views (FR-54–57) are thin configurations of this one engine — a fix or a new counter type applied to the engine appears identically across all four, never patched per-dashboard.

#### FR-54: Unit Manager dashboard
Grouped by people: headcount, active risk counts by level (with trend), open/overdue action items, active resourcing requests, open campaigns; table of subordinates.
**Consequences (testable):**
- The table and counters are scoped to the UM's reporting-line reach (FR-1) — a person outside it never appears, even transiently.

#### FR-55: Delivery Manager dashboard
Grouped by project: one table per project, with a project selector (default: all projects) that filters the whole page — table and every counter — to the selected project; includes an explicit **Unassigned** bucket for project-less resourcing requests; clearing the selection restores the all-projects aggregate. Includes the DM's own and their PMs' resourcing requests.
**Consequences (testable):**
- Selecting a single project (or Unassigned) recalculates every counter on the page, not just the table; the all-projects aggregate always equals the sum of every project's counters plus Unassigned.

#### FR-56: Project Manager dashboard
The DM dashboard's shape, scoped to the PM's own projects.
**Consequences (testable):**
- A PM never sees a project selector option, counter, or table row for a project they don't manage.

#### FR-57: People Partner dashboard
Grouped by department or project, scoped to the PP's assigned people; never renders a resourcing block.
**Consequences (testable):**
- No resourcing counter, table, or link appears anywhere on the PP dashboard, even if the viewer separately holds a resourcing permission.

### 5.13 External Integrations: Timetracker & PeopleForce
**Description:** Timetracker integration (leaves and projects/people) is load-bearing for the access model and required against the provided test environment and seeded population. PeopleForce integration is required only to the extent of storing candidate ID + link on resourcing proposals; full profile prefill is good-to-have.

**Functional Requirements:**

#### FR-58: Integrate timetracker Leaves and Projects & People APIs
Leave data (S10) and project/PM/DM assignment data (S11, feeding the ProjectAssignment record and FR-1's project-assignment leg) are pulled from the timetracker's test environment against the seeded population.
**Consequences (testable):**
- A confirmed project assignment takes effect as an access grant within 15 minutes — confirmed the moment a **single** successful sync observes it, never requiring 15 continuous minutes of sustained success (so a flapping sync can still confirm a genuinely new assignment). *(D19, `decisions.md`, refining D3)*
- On sync failure, S10/S11 display fails soft: last-known data is served behind a visible "not fresh"/"temporarily unavailable" indicator rather than blocking the profile. Project-derived Manager access fails safe: it is withdrawn once **4 cumulative hours** of failed sync accrue within a rolling window — a flapping sync that never sustains 4 *continuous* failed hours, but whose total failed time crosses the bound, still withdraws access; the clock is never reset to zero by a brief recovery. *(D3, PO-confirmed 2026-08-21; concrete numbers from v2 source, 2026-08-26; cumulative-window semantics added 2026-08-26, D19.)* This is a bounded, named residual risk, not a silent gap: a project-line grant may survive up to 4 cumulative hours past the relationship's real end during a prolonged or flapping outage.

#### FR-59: Store PeopleForce candidate references on resourcing proposals
External candidates on resourcing requests always carry a stored PeopleForce candidate ID + link, regardless of whether the full PeopleForce prefill integration is ever built.
**Consequences (testable):**
- If the optional prefill button is built (good-to-have), it is one-way, read-only, per-field-confirmed, never writes internal-decision fields (grade, seniority, employee type, department, manager, PP, contract, employment status, risk), and never silently overwrites an existing value.

#### FR-60: Resolve cross-system identity
Every timetracker and optional PeopleForce record resolves to the correct platform employee via an explicit `(system, externalId) → employeeId` mapping table, never inferred from email alone, with a `supersededBy` pointer for re-hires. The seeded platform record is the primary identity anchor — a PeopleForce link is optional per employee.
**Consequences (testable):**
- A lookup against a re-hired person's old external ID resolves through the `supersededBy` pointer to their current mapping row, never to a stale or orphaned employee record.

**Feature-specific NFRs:**
- External integration failures degrade gracefully and never take the application down, within the explicit bounds stated for the timetracker — the 15-minute/4-hour-cumulative numbers in FR-58 (D3, D19), not a vaguer general promise.

### 5.14 Employment Status & Departure Lifecycle
**Description:** A time-bounded employment-status fact per employee, with a permission-gated departure flow that cascades atomically across profile, list, tasks, mentorship, account, and access.

**Functional Requirements:**

#### FR-61: Maintain employment status as a temporal fact
The system maintains a time-bounded employment-status record per employee (`active` | `dismissed`) as the single source every consumer reads.
**Consequences (testable):**
- The departures figure in Analytics, the default All Employees exclusion, and dashboard counters all read the same record — never each computing their own definition.
- `dismissed` must never be conflated with the `leaver` risk-level prediction (FR-24).
- Reactivating a `dismissed` record back to `active` (a re-hire onto the same platform identity, distinct from FR-60's cross-system re-hire mapping) is out of scope for v1 — a re-hired person is provisioned as a new record. *(Confirmed default — source doesn't address in-place reactivation; treating it as out of scope avoids undefined behavior for history/access on an un-dismissed record. Flagged for `bmad-architecture`/SPEC follow-up if genuinely needed.)*

#### FR-62: Record a departure
Whoever holds the *record a departure* permission records a departure with an effective date and a reason.
**Consequences (testable):**
- Recording a departure for someone who still manages or partners anybody is blocked until they're re-parented, with re-parenting to their own manager offered as a one-click default on the same screen.
- Recording a departure for the sole remaining full-profile-access holder (FR-8) is blocked by the same gate until that grant is transferred to another holder first. *(D16, `decisions.md`)*

#### FR-63: Execute departure cascade
On the effective date, the departure cascade runs atomically and immediately: the profile becomes read-only and drops from the default employee list (staying filterable), open action items auto-cancel as "cancelled — departed," active mentorship pairs auto-end with a system-generated closure note (bypassing FR-43's mandatory-note gate), every shared link (FR-6) the departing employee created is explicitly revoked, any full profile access grant (FR-8) the person held is explicitly revoked and journaled (FR-9), the person's account deactivates, and every other access grant they held ends per §5.1's standard access-revocation rule (FR-1/FR-2).
**Consequences (testable):**
- The mentorship auto-close step is a no-op if the pair is already ended — so when both members of a pair depart on overlapping effective dates, the second departure's cascade trigger never re-writes or double-logs the closure note. *(D17, `decisions.md`)*
- The action-item auto-cancel step is a no-op against an item already cancelled — see FR-22's concurrent-cancellation rule.
- The shared-link revocation is its own named step, not left as an implicit side-effect of FR-6's continuous re-clamp rule — this is what makes SM-6's "stops working on departure" promise a direct guarantee. *(D17)*
- A departing non-sole full-access holder's grant gets the same explicit, journaled named-step treatment as shared links above — never merely folded into the generic "every other access grant ends" clause. The sole-holder case never reaches this step: FR-62 blocks recording the departure until the grant is transferred first.

#### FR-64: Exclude dismissed employees from default list view
Dismissed employees are excluded from the default All Employees view but remain reachable via explicit filter (including employment-status filter).

#### FR-65: Display employment status on profile
Employment status appears in S4 (Employment) and is readable by reporting line, project line, and PP — never by Self or Colleague.

**Feature-specific NFRs:**
- The departure cascade must never be a lingering background job — it is atomic and immediate on the effective date.

## 6. Non-Goals (Explicit)

- **Compensation and salary data on profiles** — no compensation section on the profile, not in this or any planned iteration. A resourcing request's expected-comp band (FR-28) is a distinct, in-scope object — not this.
- **Employee provisioning** — no creation flow, no Active Directory integration; the population is seeded only (v2 §4.17).
- **SSO** — no Entra ID this iteration; authentication is the platform's own implementation over the seeded population.
- **Leave balances shown in-platform** — self-service links out to the timetracker for both viewing leaves and managing balances.
- **Vacancy synchronization with PeopleForce** — the resourcing request has no counterpart in the recruiting system, in either direction.
- **Pre-onboarding** — creating a person before their first working day and pulling ATS data on offer acceptance is deferred.
- **Email template management** (eSender replacement) — deferred.
- **Performing competency assessments inside the system** — links to matrices and records outcomes only; assessment happens outside (§5.8).
- **Learning management (LMS) functionality** — a separate track, not duplicated here.
- **Mentorship goals, session logs, and progress tracking** — pair formation/ending/visibility only, this iteration.
- **Project allocation percentages / workload distribution modeling** — an employee's ongoing % allocation across projects is not tracked. (A resourcing request's one-time *requested* workload value, FR-28, is a distinct field.)
- **In-app/email notifications** — good-to-have, not required this iteration.
- **Analytics and reports** (beyond the Risk and campaign-completion views already specified) — good-to-have, not required this iteration.
- **PeopleForce profile prefill** — good-to-have, built only after required scope is complete (FR-59).
- **Native mobile apps** — the platform is a responsive web application (confirmed).

## 7. MVP Scope

### 7.1 In Scope
All 14 features in §5 (FR-1–FR-65) constitute v1/MVP — this platform has no "post-launch" feature tier beyond the explicitly good-to-have items below. The source spec treats this as a single delivery, not a phased rollout.

### 7.2 Out of Scope for MVP
- Notifications, Analytics/Reports, and PeopleForce profile prefill — build only if the 14 core features are otherwise complete. `[NOTE FOR PM: these are the first candidates to reintroduce if the team has capacity after core scope.]`
- Everything listed in §6 Non-Goals.

## 8. Success Metrics

*Every metric below is validated against a deployed, demonstrable build — not a local demo on a developer's laptop.*

**Primary**
- **SM-1**: Access-matrix test coverage — every audience × relationship-path × section combination, every `—` cell, every narrowed project-line cell, every flag-gated record (against both gated audiences), the campaign-sender exception, and every self-assignment/journal guarantee has a passing negative test. Validates FR-1, FR-2, FR-3, FR-4, FR-7, FR-8, FR-9.
- **SM-2**: Functional-role extensibility — a new functional role, created and granted permissions entirely through the UI, is enforced with zero code changes and never widens data access on its own, demonstrated live. Validates FR-5; realizes UJ-3.
- **SM-3**: Timetracker integration runs against its provided test environment over the seeded population (not mock/seed data) in the demonstrated build. Validates FR-58.
- **SM-4**: List performance — All Employees responds within 2 seconds at 500+ records under arbitrary filters, including permission resolution. Validates FR-10.
- **SM-5**: Parallel-delivery evidence — the delivery record (sprint status, commit/PR history) shows no developer-waits-on-developer defect against the wave/track plan `bmad-sprint-planning` produces.
- **SM-6**: Shared-link correctness — a shared link works only for its named authenticated recipient, continuously re-clamps to the creator's current access (not just alive/dead), is explicitly revoked the instant its creator departs, and is always revocable by whoever currently holds the relationship. Validates FR-6, FR-29, FR-63.
- **SM-7**: Departure cascade — recording a departure produces an atomic, immediate, idempotent cascade (read-only profile, list exclusion, action items, mentorship, shared links, account, access) with no orphaning of managed reports or of the sole full-access holder. Validates FR-8, FR-62, FR-63.

**Secondary**
- **SM-8**: Intelligent-repository fidelity — the specs in this repository match shipped behavior at demo time (spot-checked, not just self-attested).
- **SM-9**: Accessibility — List, Profile, and Dashboard pages meet WCAG 2.1 AA (confirmed default, `decisions.md`).

**Counter-metrics (do not optimize)**
- **SM-C1**: Feature-count velocity — shipping more of §5 faster must never come at the cost of SM-1's test coverage or SM-5's parallel-delivery discipline. Per the source spec, a team that ships less but demonstrates clean process scores higher than one that ships more by abandoning it. Counterbalances SM-3/SM-4.
- **SM-C2**: Cache-driven staleness — SM-4's performance work (any access-resolution caching, per D1) must never be optimized by weakening cache invalidation; a stale permission cache is a data leak, not a performance win. Counterbalances SM-4.

## 9. Open Questions

- **D11 — Default functional-role permission assignments (§5.1).** Five launch defaults need PO confirmation before the roles admin screen ships: who holds *manage custom fields*, who holds *assign and end mentorships*, and the launch defaults for *approve or reject proposed candidates*, *edit the career timeline*, and *create feedback*. Current default pending sign-off: seed the pre-v2 hardcoded assignments, then let HR Admin adjust via the roles UI. See `decisions.md` D11. Not blocking the permission mechanism itself.
- **D12 — Timetracker sync model: events vs. state snapshots (§5.13).** To be settled during the §5.13 investigation — determines whether the relationship journal can answer "why did this person have access on date X." See `decisions.md` D12. Technical investigation outcome, not a product decision.

Nine further low-stakes defaults (upload limits, IDP reopening, saved-view shape, CDS assessor field, timeline-deletion mode, post-access-loss cancellation, link expiry with the resourcing exception, accessibility target, and saved-view ownership on the creator's departure) ship with stated defaults — see `decisions.md`'s appendix.

Eight edge-case safety guards, surfaced by a `bmad-review` edge-case-hunter pass and resolved as engineering defaults (not PO-confirmed — none change stated behavior, they close unhandled paths within it): multi-role access resolves as union (D13, FR-1); shared-link exposure re-clamps continuously (D14, FR-6); organisational-relationship writes reject self-managed departments, reporting cycles, silent concurrent overwrites, and orphaning clears (D15, FR-7); full-access holder-count checks happen at commit time and block a sole holder's departure (D16, FR-8/FR-62); the departure cascade is idempotent and explicitly revokes shared links (D17, FR-22/FR-63); the resourcing lifecycle supports reversible approval, headcount-floor/zero-remaining guards, and live UM/DM re-resolution (D18, FR-30/FR-31/FR-33); the timetracker sync clock is cumulative-window rather than continuous-streak, and confirms on first success (D19, FR-58); mentorship pairs enforce consent server-side and reject concurrent double-closes (D20, FR-42/FR-43). Full detail in `decisions.md` D13–D20; source findings in `review-edge-case-hunter.md` and `validation-report.md`.

## 10. Assumptions Index

Entries confirmed 2026-08-21 unless noted.

- §2 — Platform is a responsive web application; no native mobile app; no SSO; local auth over seeded population (D9, reaffirmed by v2 §4.17).
- §3.3 (UJ-1, UJ-2, UJ-3) — UJ-1 and UJ-2 authored from source material rather than narrated live; confirmed representative. UJ-3 was added during this update's finalize review (not user-narrated) to dramatize the functional-role-creation scenario — its representativeness has not yet been separately confirmed by the user.
- §5.1 FR-6 — Shared Link default expiry (24h, configurable at creation; resourcing-generated links bound to request lifetime instead, v2 §4.8) is a fact directly stated by the source, not an inferred assumption — indexed here only for traceability alongside the genuine assumptions below.
- §5.2 FR-13 — Saved views are filter-based only for v1; no static membership-list variant. A shared view re-resolves to the viewer's own access and is read-only for non-creators (source doesn't specify; chosen to avoid a data leak). On the creator's departure, a view becomes ownerless rather than transferring to a manager/PP — a view has no profile-scoped owner to transfer to (added 2026-08-26, D17/appendix).
- §5.3 FR-18 — Photo/certificate upload limits use sane conventional defaults; source states none.
- §5.4 FR-22 — Action-item cancellation survives the author losing live access; authorship is a historical fact.
- §5.7 FR-35 — Timeline event deletion is soft-delete with audit trail.
- §5.8 FR-38 — CDS assessor field is free text or a person reference, implementer's choice.
- §5.8 FR-39 — Reopening a completed IDP is disallowed by default.
- §5.13 FR-58 — Timetracker sync failure: 15-minute grant window, 4-hour access withdrawal, fail-soft display (D3, PO-confirmed 2026-08-21; concrete numbers from v2, 2026-08-26). Named residual risk: a project-line grant may survive up to 4 cumulative hours past the relationship's real end during a prolonged or flapping outage. The 4-hour clock is a rolling cumulative window, never reset by brief recovery, and a single successful sync confirms an assignment (added 2026-08-26, D19).
- §5.14 FR-61 — In-place reactivation of a `dismissed` record back to `active` is out of scope for v1; a re-hire is provisioned as a new record. Source doesn't address this; flagged for SPEC/architecture follow-up if genuinely needed.
- §5.1 FR-1/FR-6/FR-7/FR-8, §5.4 FR-22, §5.6 FR-30/FR-31/FR-33, §5.9 FR-42/FR-43, §5.13 FR-58, §5.14 FR-62/FR-63 — eight edge-case safety guards (multi-role union, shared-link continuous re-clamp, organisational-relationship write safety, full-access commit-time check, cascade idempotency, resourcing lifecycle robustness, timetracker sync-clock semantics, mentorship pair write-time safety) added 2026-08-26 from a `bmad-review` edge-case-hunter pass; see the §9 pointer and `decisions.md` D13–D20 for full detail. None were PO-confirmed individually — all are safe-default enforcement of guarantees the PRD already stated elsewhere.
- §6 — Native mobile apps, employee provisioning, SSO, and in-platform leave balances out of scope.
- §8 SM-9 — WCAG 2.1 AA is the accessibility target.

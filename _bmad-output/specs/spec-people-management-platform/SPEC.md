---
id: SPEC-people-management-platform
companions: [access-model.md, interface-contracts.md, decisions.md, ../../planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md]
sources: [../../../docs/project-requirements.md, ../../../docs/PRD_parallel_delivery_plan.md, ../../../docs/backlog_review_draft.md]
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Source documents listed in frontmatter are for traceability — consult them only if you need narrative rationale or prose color this contract intentionally omits. Note: `PRD_parallel_delivery_plan.md` and `backlog_review_draft.md` also carry delivery-planning and story-level content that this SPEC deliberately does not absorb (see `decisions.md`) — that content remains their own direct input for the `bmad-sprint-planning` and `bmad-create-epics-and-stories` phases respectively, not something downstream should expect to find here.
>
> Bare `§N` / `N.N` citations below refer to `project-requirements.md`'s own numbering (its §1–10); citations into the other two source files name them explicitly.

# People Management Platform

## Why

An engineering organisation of 500+ employees, distributed across units and projects, needs a platform to replace/upgrade its current internal people-management system — decomposed into a strict per-section access model so that who-sees-what-about-whom is never a footnote. This is also the vehicle for an AI-native SDLC bootcamp: learning a clean, spec-driven, genuinely parallel delivery process is the primary objective, and a working product is the deliberate side effect — where the two conflict, process wins (spec §1). That priority is why this SPEC, and the BMad artifacts built on top of it, matter as much as the code.

## Capabilities

- **CAP-1 — Access Control, Roles & Employee Profile** *(Epic 1, incl. Profile Sharing 4.8)*
  - **intent:** Resolve every viewer's access role per subject (Self / Manager / PP / Colleague / Shared Link / HR Admin) from the transitive closure of reports-to and project-assignment relationships, assemble each profile from exactly the S1–S16 sections that role is entitled to, enforce the colleague whitelist and the S7 manager/PM note-visibility exception, let HR Admin define and grant functional roles/permissions as runtime data with no deploy, and let a manager generate a time-limited, revocable, logged, per-section shareable profile link.
  - **success:** Every audience × relationship-path × section combination is covered by a negative test asserting the API/UI/export/search never returns a `—` cell, an unflagged S7 record, or a non-whitelisted colleague field; a new functional role created via UI is assignable and enforced with zero code change; access is re-resolved per request, with no session-cached grant surviving a relationship change.

- **CAP-2 — All Employees Directory & Custom Fields** *(Epic 3)*
  - **intent:** One filterable/sortable employee list, scoped per viewer's access, where any built-in, derived, or custom field is usable as a column/filter/sort key; custom fields are defined at runtime with no schema migration; list cells are inline-editable subject to the access matrix; filter+column configurations save as named shareable views; the current view exports to `.xlsx` containing only columns the exporter can see.
  - **success:** A newly created custom field is immediately usable as filter/column/export with no deploy; a saved view is reusable and shareable; the list responds within 2s at 500+ records under arbitrary filters, including permission resolution.

- **CAP-3 — Self-Service** *(Epic 2)*
  - **intent:** An employee can view their own employment summary, edit their own personal/emergency contacts and residential address, upload a photo and certificates, and view their own timeline, leaves, projects, CDS/IDP, mentorship status, shared feedback, flagged management notes, and action items — marking IDP items and action items complete themselves.
  - **success:** An employee completes every listed self-service action without manager/PP involvement, and never sees their own risk level or an unflagged management note.

- **CAP-4 — Action Items and Tasks** *(Epic 4)*
  - **intent:** One task entity (title, description, assignee, author, due date, optional link, status, completion date, source) creatable manually by a manager/PP/permitted role for anyone in their access scope, creatable automatically by form-campaign activation (one per recipient), completable by the assignee, cancellable by the author with a reason, rendered overdue everywhere it appears once its due date passes.
  - **success:** Lifecycle open→completed is correct for both manual and campaign-sourced items; overdue derivation is identical everywhere it's displayed; cancellation is scoped by historical authorship, not live access.

- **CAP-5 — Risks and Risk Dashboard** *(Epic 5)*
  - **intent:** Record a per-employee risk (level low/need attention/medium/high/leaver, description, details, date) with retained history, show a trend arrow versus the previous record, and provide a Risk Dashboard with severity-sorted counts and a drill-through table filterable by unit/department/project/PP/manager, scoped to the viewer's Manager/PP access.
  - **success:** The current level is always the most recent record; the trend arrow appears only on an actual level change; the section and dashboard are never visible to the employee themself.

- **CAP-6 — Resourcing** *(Epic 6)*
  - **intent:** A DM/PM/permitted role creates a resourcing request (vacancy details, comp level, duration, workload, optional project link); a UM fulfils it by proposing internal employees or external PeopleForce candidates; the DM approves/rejects each proposal with a written reason (using profile-sharing for internal candidates the DM lacks access to); full request history is retained on both the request and the employee's profile (S15).
  - **success:** An unattached request with no project is a valid, normal state; every proposal records proposed→approved/rejected with feedback; external-candidate review works via the live PeopleForce integration or, as an explicit sanctioned fallback, a link out to PeopleForce.

- **CAP-7 — Career Timeline** *(Epic 7)*
  - **intent:** Auto-generate a timeline event via one system-wide event-writer whenever a tracked change occurs (joining, grade/position/department change, FTE↔subcontractor transition, extended leave, mentorship pair start/end); PP/UM can manually add/edit/delete events for historical backfill or correction. Presented as a readable chronological timeline on the profile.
  - **success:** Every tracked change type produces exactly one system-generated event with no separately maintained record; a manual correction is never silently overwritten by a later system-generated event covering the same window — the conflicting write is skipped and logged.

- **CAP-8 — CDS: Career Development System** *(Epic 8)*
  - **intent:** Each profile shows a resolved link to the current skills-matrix file for that person's department+position via a maintained dictionary (not encoded structure), an assessment log (date, assessor, result-file link, text conclusion) editable by manager/PP, and an IDP (description, deadline, external file link, complete checkbox) the employee can self-complete — plus All-Employees filters for last-assessment date (with "never assessed" as a distinct selectable option) and has-open-IDP.
  - **success:** Updating the centrally maintained matrix file changes what every affected profile's link resolves to, with no per-profile update needed; "never assessed" is selectable and distinct from any date-range result.

- **CAP-9 — Mentorship Hub** *(Epic 9)*
  - **intent:** An employee self-flags open-to-mentoring and sees their own mentor/mentees; a manager/PP browses everyone flagged open-to-mentor, assigns a mentor-mentee pair (flipping the mentor's status to "mentor" on their first pair), ends a pair only with required closing feedback recorded (reverting status per the self-flag rule — see `decisions.md` D4), and views all pairs with dates. The mentor is shown in every profile header alongside manager and PP.
  - **success:** A pair cannot be ended without closing feedback; mentor-status transitions follow the self-flag rule exactly, even when the flag was turned off mid-pair; every pair end writes a CAP-7 timeline event.

- **CAP-10 — Forms & Survey Campaigns** *(Epic 10)*
  - **intent:** A PP/manager/permitted role creates a draft form campaign (title, description, purpose, external form link, due date), builds its audience via the CAP-2 filter engine or a saved view with individual add/remove, activates it to atomically freeze the audience and generate one CAP-4 action item per recipient, and tracks per-person completion (completed / not completed / overdue) — without ever reading or verifying the external form itself.
  - **success:** The frozen audience never changes post-activation; activation is all-or-nothing, with no partial state; a recipient's own action-item completion is the only completion signal the system trusts.

- **CAP-11 — Feedback** *(Epic 11)*
  - **intent:** A manager/PP records a feedback record on a profile (subject, author, date, context, body) with a visibility flag defaulting to management-only and flippable to shared-with-employee; records are viewable chronologically with period-over-period comparison; a manager/PP can request feedback from specific named colleagues via a CAP-10 campaign targeted individually (not by filter), then manually authors the resulting Feedback record after reading the external form's actual responses.
  - **success:** Flipping visibility makes a record immediately visible on the employee's own profile with no stale grant; a colleague never gets read access to feedback about someone else; completing a feedback-request campaign never auto-creates a Feedback record.

- **CAP-12 — Dashboards** *(Epic 12)*
  - **intent:** One shared dashboard engine serving four role-scoped views — Unit Manager (grouped by people), Delivery Manager (grouped by project, with an all-projects/single-project selector that recalculates every counter), Project Manager (the DM view scoped to own projects), People Partner (grouped by department/project, no resourcing block) — each showing summary counters, a people/project table with risk and leave status, and the viewer's own overdue-highlighted action items.
  - **success:** Selecting a project on the DM dashboard filters the whole page and every counter to it; clearing the selection restores the all-projects aggregate; the PP dashboard never renders a resourcing block.

- **CAP-13 — External Integrations: Timetracker & PeopleForce** *(Epic 13)*
  - **intent:** Integrate the timetracker's Leaves API (S10) and Projects & People API (S11 — which also feeds the CAP-1 project-assignment access leg) against real endpoints; integrate PeopleForce for external-candidate data (CAP-6) and vacancy source-of-truth; reliably map every PeopleForce/timetracker record to the right employee, including through re-hires (see Constraints for the required mapping strategy).
  - **success:** A timetracker sync outage fails safe for access (an unconfirmed project assignment never grants Manager access) and fails soft for display ("temporarily unavailable" instead of blocking the profile); where the PeopleForce integration isn't completed in time, a link out to PeopleForce is an accepted, sanctioned fallback — never a silent gap.

## Constraints

- Access roles (derived from relationships, never assigned) and functional roles (assigned, extensible, data not code, never widen data access) are two independent dimensions and must never collapse into one role list.
- Access is evaluated server-side, per section, per request — never cached client-side, never assumed stable across a session; any permission cache invalidates synchronously off a generation-bump on the relationship graph, never relying on TTL for correctness. *(D1, `decisions.md`)*
- The colleague view is a strict server/API-level whitelist (S1, S10 incl. leave type, S11 project name only) — never implemented by hiding fields client-side.
- Custom fields carry their own visibility (management default / employee / colleague); filters and list columns must respect it — a user must never infer a value they can't see via a filter.
- Any profile field, including runtime-defined custom fields, must be filterable/sortable/column-usable with no schema migration — a column-per-field data model is rejected.
- Grade, position, department, and employment type are temporal — modeled as time-bounded records, not scalar fields plus a bolted-on audit log.
- CDS never encodes the competency matrix in schema — it's a registry/hub only; assessment and the matrix itself happen outside the system.
- The system never reads or verifies an external form or its responses, for campaigns or feedback requests alike — the assignee's own action-item completion is the only trusted completion signal.
- A resourcing request is valid and normal with no linked project — project attachment must not be required.
- Cross-system identity resolves via an explicit `(system, externalId) → employeeId` mapping table, never inferred from email alone, with an explicit `supersededBy` pointer for re-hires.
- A resourcing approval never itself writes a `ProjectAssignment` (C3) record — the person is assigned to the project in the timetracker, and CAP-13's sync is the only writer; CAP-6's approval action only records the approval decision.
- Leave requests and edits are never performed in-app — CAP-3 displays leave data (read-only, via S10) and links out to the timetracker to manage it; the timetracker is the only place leave balances change.
- Team decomposition and delivery sequencing must guarantee zero cross-person blocking — one developer waiting on another is a process defect regardless of shipped output. Cross-feature dependencies resolve only via a frozen interface contract, a stub, a spec-sanctioned fallback, or same-developer sequencing.
- BMad is the mandated planning/process framework for at least the start of the project; migrating away from it must be a recorded deliberate decision, never silent drift.
- Non-production environments use pseudonymized data only (real structure/volume, substituted names/contacts) — never real personal data in agent contexts, logs, screenshots, or the repository.
- The All Employees list responds within 2 seconds at 500+ records under arbitrary filters, including permission-resolution cost.
- External integration failures (timetracker, PeopleForce) degrade gracefully and never take the application down.
- List, profile, and dashboard pages meet accessibility (WCAG 2.1 AA baseline, see `decisions.md` appendix) and responsive-layout requirements.
- Access-control correctness is the system's primary quality attribute — every audience/relationship-path/section combination, every `—` cell, and every flag-gated record against both gated audiences must be covered by an explicit test.
- The intelligent repository (this workspace: specs, decisions, call transcripts, external API docs, agent rules/skills) is mandatory, not optional documentation — and a foundation phase (prototyping/design approach, best-practice research, test architecture, named owners per topic) runs before feature implementation starts, with findings aligned and written down first. Inter-team communication and status are captured for analysis.

## Non-goals

- Compensation and salary data — no compensation section on the profile, this or any planned iteration.
- Pre-onboarding — creating a person before their first working day and pulling ATS data on offer acceptance — deferred.
- Email template management (eSender replacement) — deferred.
- Performing competency assessments inside the system — links to matrices and records outcomes only; assessment happens outside.
- Learning management (LMS) functionality — a separate track, not duplicated here.
- Mentorship goals, session logs, and progress tracking — pair formation/ending/visibility only, this iteration.
- Project allocation percentages / workload distribution modeling — an employee's ongoing % allocation across projects is not tracked or computed. (A resourcing request's one-time *requested* workload value, CAP-6, is a distinct, in-scope field — not the same concept.)
- In-app/email notifications — not required this iteration (good-to-have, build only once core scope is complete).
- Analytics and reports — not required this iteration (good-to-have, build only once core scope is complete).

## Success signal

The platform is deployed and demonstrable — not running on someone's laptop — with every access-matrix cell and negative case covered by passing tests, a new functional role provably creatable and grantable through the UI with no deploy, and the timetracker integration running against its real API. The intelligent repository's specs match shipped behavior, and the delivery shows no developer-waits-on-developer defect (per Constraints), verified against whatever wave/track plan the `bmad-sprint-planning` phase produces — that plan is out-of-contract input to this success signal, not something this SPEC defines itself.

## Assumptions

- A manually edited timeline event is never silently overwritten by a later system-generated event covering the same change window — the system write is skipped and logged (documented as an ADR). *(D2, `decisions.md`)*
- Mentor status reverts to "open to mentoring" when the last active pair ends only if the self-flag is still on; otherwise it reverts to no status. *(D4)*
- Mentorship-end closing feedback is stored as its own field on the mentorship pair record for this iteration, not routed through the general Feedback (S8) entity. *(D6)*
- IDP maintenance (manager/PP side) and the employee's self-complete checkbox are owned by the same story/developer rather than split across epics. *(D7)*
- The `external_identities` mapping table shape is an early-ratified starting point, refinable once the real PeopleForce/timetracker investigation (Epic 13) completes. *(D8/C5)*
- Eight further low-stakes defaults ship as-is unless told otherwise — upload limits, IDP reopening, saved-view shape, CDS assessor field, timeline-deletion mode, post-access-loss cancellation, link expiry, and the accessibility conformance target. Full defaults and rationale in `decisions.md`'s Appendix.
- Confirmed: during a timetracker sync outage, access fails safe (an unconfirmed project assignment never grants Manager access) while display fails soft (S10/S11 show "temporarily unavailable" instead of blocking the profile). *(D3, PO-confirmed)*
- Confirmed: the profile header's mentor field follows the stricter "visible to manager line and PP" rule from 4.11, overriding the general Colleague-R grant on identity-card content. *(D5, PO-confirmed)*

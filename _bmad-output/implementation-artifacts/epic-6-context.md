# Epic 6 Context: Resourcing

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver the end-to-end staffing workflow: a Delivery or Project Manager raises a resourcing need, a Unit Manager proposes internal or external candidates, and a Delivery Manager approves or rejects each proposal with recorded rationale. Every proposal attempt is retained as permanent history. The resourcing request is a platform-owned entity — it is never synchronized with PeopleForce, and approving a candidate does not assign them to a project locally; actual assignment happens in the timetracker and surfaces on profiles only after the next sync.

## Stories

- Story 6.1: Create a Resourcing Request
- Story 6.2: Fulfil a Request with Internal or External Candidates
- Story 6.3: DM Reviews and Approves/Rejects Candidates
- Story 6.4: Request History

## Requirements & Constraints

- DMs, PMs, or roles holding the *create resourcing requests* permission can create a request capturing vacancy details, expected compensation band, duration, workload, headcount (defaulting to 1), and a required department for Unit Manager routing. An optional project reference may be omitted — an unattached request is a normal, fully functional state, not a draft or degraded record.
- The expected compensation band is visible only to the request author, the routed Unit Manager, and the reviewing Delivery Manager. It must never appear to People Partners, on the candidate's profile, in shared links, or in exports.
- Request visibility follows the manager-access chain: a DM sees their own requests plus those created by PMs on projects they manage; a PM sees only requests they personally created. Functional-role permissions unlock feature actions but never widen underlying data access.
- Every request routes to the Unit Manager of the selected department. Routing re-resolves to the department's current manager on every view — it is never pinned to whoever held the role at creation time. If no successor Unit Manager exists, pending work is flagged for reassignment rather than left orphaned.
- Unit Managers propose one or more internal employees from their unit and/or external candidates identified by PeopleForce reference. A live PeopleForce data pull is optional; storing an outbound PeopleForce link is an accepted, permanent fallback — not a placeholder that blocks the workflow.
- Submitting an internal candidate auto-generates a read-only shared profile link for the reviewing DM, scoped to identity, employment, projects, CDS, and CV/certs, bound to the request's lifetime. Sensitive profile sections are never included by default.
- The reviewing DM approves or rejects each candidate independently. Rejection without a written reason is invalid. Approval consumes one headcount slot but never writes a local project-assignment record. An approval may later be reversed to rejected with a reason, freeing the slot. Requests do not auto-close when headcount fills — only an explicit DM close ends them.
- Full proposal history (proposed → approved/rejected, with feedback) appears on the request detail and, for internal candidates only, in profile section S15. S15 is readable by the candidate's manager line and People Partner, never by the candidate themselves.
- People Partners have no resourcing surface on their dashboard. Project-less requests appear in an Unassigned bucket on Delivery Manager dashboards.

## Technical Decisions

- Backend module: `resourcing`, depending on `contracts` and `registry` only (no cross-feature module imports).
- Feature gates use the shared permission checker; no module queries functional-role tables directly.
- Department directory contract supplies live Unit Manager routing. Access-control state reader backs reassignment when a responsible DM or Unit Manager has departed — routing to a live successor on the same project, or flagging a full-access holder as backstop.
- External candidates resolve through the external-identity mapping contract where PeopleForce integration is live; the outbound-link fallback requires no mapping.
- The resourcing module registers a dashboard summary provider (open-request counts) and a section provider for S15 (request history on profiles).
- Project references on requests come from the timetracker integration feed, not a locally owned project entity. The integrations module remains the sole writer of project-assignment records.
- Frontend surface: `pages/Resourcing/` per the information architecture — request list, request detail (fulfillment and approval), and history. Where a flow spans capabilities with no backend contract between them (e.g., candidate review triggering a shared link), the frontend orchestrates sequential API calls from the initiating page.

## UX & Interaction Patterns

- **Resourcing** nav entry is visible to Unit Managers, Delivery Managers, Project Managers, and permitted roles; hidden for People Partners and anyone without resourcing permissions.
- **Resourcing Candidate Card** on request detail: internal candidates embed or link to the auto-generated shared profile view; external candidates show PeopleForce data or an outbound "View in PeopleForce" link. Approve and Reject actions sit on each card.
- **Required-reason dialog** for rejections — primary button disabled until text is entered; no silent no-reason paths.
- **Empty state** on the request list follows the generic pattern: one line naming the surface plus a single primary create action.
- The Marcus review journey (UJ-2) is the reference flow: a DM reviews mixed internal/external candidates, makes informed decisions without gaining standing manager access to internal candidates, and sees decisions recorded immediately.

## Cross-Story Dependencies

- **Within epic:** 6.1 → 6.2 → 6.3 → 6.4 form a sequential pipeline (create → propose → decide → persist history).
- **Epic 1 (Access Control):** manager-access chain for request visibility (Story 1.2), shared-link generation and consumption for internal candidate review (Stories 1.11/1.12), S15 access matrix enforcement.
- **Epic 13 (External Integrations):** timetracker supplies optional project references and is the sole writer of project assignments post-approval; PeopleForce integration (Story 13.3) enhances external-candidate attachment but does not block this epic — the fallback link path ships independently.
- **Epic 12 (Dashboards):** DM and PM dashboards include a resourcing block with role-appropriate visibility scoping; UM dashboard shows active resourcing requests assigned to them.

# People Management Platform — Backlog Review & Story Drafts

Draft for review. Nothing in this document has been written back to ClickUp yet — it's staged here so you can review, cut, or adjust before anything gets pushed.

Legend: each existing story shows its current one-line description (quoted from ClickUp), a proposed extended description grounded in the spec (with section-number citations), and a Given/When/Then block. New stories (flagged `NEW`) close gaps identified in the earlier review pass.

---

# Epic 1: Roles, Access Control & Employee Profile

## Story 1.1: Derive Manager Access from Reporting Hierarchy (ClickUp task: 869em1vva)
**Current description:** **As** the system, **I want** to compute a person's Manager access role via the transitive closure of "reports to" relationships, **so that** anyone above someone in the reporting chain is recognized as their manager without a manual grant.

**FRs:** FR1, FR2 (reports-to leg)

**Acceptance Criteria:**
*   Given employee B reports to A directly or transitively, when any endpoint resolves A's access with respect to B, then A is recognized as holding Manager access with respect to B, re-evaluated on every request (never cached across a session).
*   Given no reporting relationship exists between A and B, when A's access to B is resolved via this path, then A is not granted Manager access this way.

**Proposed description:**
Implement the "reports to" leg of Manager access as the transitive closure of the reporting-hierarchy graph (2.1). If B reports to A, directly or through any number of intermediate managers, A must be recognized as holding Manager access with respect to B — a manager two or more levels up sees everything about every person nested beneath them, without any explicit grant (2.1, consequences bullet 1). This is one of two relations (the other is project assignment, Story 1.2) whose union produces the full Manager access role for a subject/object pair (2.1). Access must be evaluated relationship-by-relationship and per request: the same person can simultaneously be Manager for one profile, PP for another, and a plain Colleague for a third within a single session (2.1) — so this resolution cannot be computed once at login and reused. Resolution is server-side, per section, per request (3.3.4), and combines with reports-to closure feeding the caching/invalidation design in NEW A. Guard against and reject reporting-graph cycles (A cannot transitively report to itself) even though the org chart should not normally contain them.

- Transitive closure must work at arbitrary depth, not just direct reports (2.1).
- Reorganizations (a person's manager changing) must be reflected on the very next request, not after a cache TTL or deploy (2.1, 3.3.4, S6).
- The reports-to leg is combined (unioned) with the project-assignment leg (Story 1.2) to produce final Manager access — do not implement this story as a standalone permission source that ignores project assignment.
- A person with no reporting-chain relationship to another gets no Manager access via this path (negative case must be tested, S9 DoD).
- Self is never its own manager; the closure must not incorrectly grant Manager access to Self.

**Given/When/Then:**
```
Scenario: Skip-level manager sees a deeply nested report
Given employee B reports to Manager M, and M reports to Director D
When D's access with respect to B is resolved
Then D holds Manager access with respect to B, without any explicit grant having been made for D over B

Scenario: No reporting relationship yields no Manager access via this path
Given employee A and employee B share no reporting relationship, direct or transitive
When A's access with respect to B is resolved
Then A does not hold Manager access with respect to B via the reports-to leg
```

## Story 1.2: Extend Manager Access via Project Assignment (ClickUp task: 869em1vwr)
**Current description:** **As** the system, **I want** Manager access to also derive from "assigned to a project managed by," backed by an internally-owned `ProjectAssignment` record, **so that** PMs/DMs see the people on their projects, seeded internally until Epic 13 supplies it live.

**FRs:** FR2 (project leg)

**Acceptance Criteria:**
*   Given employee B is assigned to a project whose PM or DM is A, when A's access to B is resolved, then A holds Manager access with respect to B.
*   Given B's project assignment ends, when access is next resolved, then A no longer holds Manager access via that project.
*   `ProjectAssignment` must be queryable domain data independent of any external integration being live.

**Note:** This is the foundational data model consumed later by Epic 13 (Integrations) — see epics.md implementation note under Epic 1.

**Proposed description:**
Implement the "assigned to a project managed by" leg of Manager access (2.1). If B is assigned to a project whose PM or DM is A, A holds Manager access with respect to B, at full strength — a DM sees full information about every person on their projects, at the same level as that person's own unit manager (2.1, consequences). Because the PM and the DM sit in the same project chain, and the DM is above the PM in that chain, the DM must transitively see everything the PM sees on shared projects, plus every person on the DM's other projects (2.1). Back this with an internally-owned `ProjectAssignment` record so the permission model does not depend on the timetracker integration being live (5.1 notes the projects/people API is "load-bearing beyond display" for permissions, but this story must work with internally-seeded data until Epic 13 wires the real feed). Critically, when a project assignment ends, the derived Manager access must end with it immediately — "managerial access is not sticky" (2.1, last consequence bullet). This immediate-revocation requirement is a direct input to the caching design in NEW A: whatever caching layer exists must not let ended project assignments continue to grant access.

- B assigned to a project managed by PM/DM A ⇒ A resolves as holding Manager access to B (2.1).
- DM inherits the PM's visibility on shared projects, plus all people on the DM's other projects, via the same-chain relationship (2.1).
- Ending an assignment revokes the derived access on the very next resolution — no grace period, no sticky access (2.1).
- `ProjectAssignment` is queryable, internally-owned domain data, independent of any live external integration (5.1; explicit story note).
- This leg is unioned with the reports-to leg (Story 1.1) to compute final Manager access for a subject/object pair.

**Given/When/Then:**
```
Scenario: DM inherits access via a PM's project chain
Given employee B is assigned to Project X, whose PM is P and whose DM is D
When D's access with respect to B is resolved
Then D holds Manager access with respect to B, at the same strength as P's access to B

Scenario: Ending a project assignment revokes Manager access immediately
Given PM P holds Manager access to employee B solely via B's assignment to Project X
When B's assignment to Project X ends and P's access to B is resolved again
Then P no longer holds Manager access with respect to B via that project
```

## Story 1.3: Assign People Partner Relationships (ClickUp task: 869em1vz3)
**Current description:** **As** an HR Admin, **I want** to assign a People Partner to an employee, **so that** the assigned PP (and their HR line) holds People Partner access for that employee.

**FRs:** FR1 (PP leg)

**Acceptance Criteria:**
*   Given HR Admin assigns PP X to employee B, when X's access to B is resolved, then X holds People Partner access with respect to B.
*   The assignment is visible on B's profile header.

**Proposed description:**
Implement the People Partner access role, which — unlike Manager — arises purely from assignment rather than hierarchy (2.1): if A is the assigned people partner of B, A holds PP access with respect to B. The "PP" audience in the section access matrix (3.2) is defined as "the assigned people partner and the HR line above them" (3.1), so this story must resolve not just the directly-assigned PP but also whoever sits above that PP in the HR reporting line — mirroring the transitive nature of Manager access but scoped to the PP/HR hierarchy rather than the general org chart. The assignment feeds the profile header (4.2, Story 1.7) and must be changeable: reassigning or clearing a person's PP must end the previous PP's (and their HR line's) access immediately, consistent with the "resolved server-side per request" rule (3.3.4). PP access is independent of Manager access — a person can be a Manager for one profile and a PP for another within the same session (2.1).

- HR Admin (or another role holding the appropriate functional permission) assigns PP X to employee B; resolving X's access to B afterward returns People Partner (2.1).
- The HR line above X — X's own management chain — also resolves as holding PP-equivalent access to B (3.1's definition of the "PP" audience).
- The current PP assignment is surfaced in B's profile header (4.2, S1 identity card content list).
- Reassigning B's PP ends the previous PP's access to B on the next resolution, not after a delay.
- Only one active PP assignment exists per employee at a time (the matrix and header both refer to "the" people partner, singular).

**Given/When/Then:**
```
Scenario: Assigning a People Partner grants PP access
Given HR Admin assigns People Partner X to employee B
When X's access with respect to B is resolved
Then X holds People Partner access with respect to B, and X's HR line above them also resolves as holding PP access to B

Scenario: Reassigning the People Partner revokes the previous PP's access
Given People Partner X currently holds PP access to employee B
When HR Admin reassigns B's People Partner to Y and X's access to B is resolved again
Then X no longer holds PP access with respect to B, and Y does
```

## Story 1.4: Define Functional Roles and Permissions via UI (ClickUp task: 869em1w0x)
**Current description:** **As** an HR Admin, **I want** to create a functional role, name it, and grant it a set of feature permissions through the UI, **so that** a new part of the org can use platform features without a code deploy.

**FRs:** FR3

**Acceptance Criteria:**
*   Given I create a role and grant it "create form campaigns", when I save it, then the permission is stored as data and applies immediately to anyone holding the role.
*   Given a permission is removed from a role, when a holder next attempts that action, then it is denied immediately.
*   Minimum grantable permissions: create form campaigns, create action items, create/edit risks, create resourcing requests, fulfil resourcing requests, assign mentors, maintain CDS records, manage custom fields, view a given dashboard.

**Proposed description:**
Build the admin UI and data model that let an HR Admin create a new functional role, name it, and grant it a set of feature permissions — entirely as data, with no deploy and no schema change (2.3). This is the mechanism that lets other parts of the org (the spec's example: IT running its own security-awareness campaigns) use platform features without becoming managers or people partners and without a code change (2.3). Functional roles and their permissions must be modeled as data, not code — the data-model notes explicitly warn against storing "is a DM" as a permission (S6). Feature permissions must be granular and independently grantable; the spec lists a minimum set: create form campaigns, create action items, create and edit risks, create resourcing requests, fulfil resourcing requests, assign mentors, maintain CDS records, manage custom fields, and view a given dashboard (2.3). This story covers only the definition side (role + permission set); assigning a defined role to a person is Story 1.5. Critically, a functional role permission never grants or widens data access on its own (2.1, 2.3) — that boundary is enforced in Story 1.5/1.6, but this story's data model must not conflate "feature permission" with "access role" in its schema, or the boundary becomes unenforceable downstream.

- HR Admin can create a role, name it, and select from the minimum permission set (and any future ones) entirely through the UI (2.3).
- Saving a new role/permission grant persists it as data and it applies immediately to all current and future holders — no deploy (2.3).
- Removing a permission from a role takes effect immediately for everyone currently holding that role, mid-session (2.3, explicit rule).
- At minimum, these permissions are independently grantable: create form campaigns, create action items, create/edit risks, create resourcing requests, fulfil resourcing requests, assign mentors, maintain CDS records, manage custom fields, view a given dashboard (2.3).
- The role/permission schema does not itself store or imply any data-access (Manager/PP/Colleague) grant — access roles remain derived from relationships only (2.1, S6).

**Given/When/Then:**
```
Scenario: HR Admin creates a role with a granular permission
Given HR Admin is on the functional roles admin screen
When they create a role named "Security Champion" and grant it only "create form campaigns"
Then the role is persisted as data, and anyone later assigned this role can create form campaigns without any code deploy

Scenario: Revoking a permission takes effect immediately
Given the "Security Champion" role currently grants "create form campaigns" and Person X holds that role
When HR Admin removes the "create form campaigns" permission from the role
Then X's next attempt to create a form campaign is denied, without needing to log out or wait
```

## Story 1.5: Assign Functional Roles to People (ClickUp task: 869em1w2e)
**Current description:** **As** an HR Admin, **I want** to assign an existing functional role to a person through the UI, **so that** they gain the role's granted features without gaining new data access.

**FRs:** FR3 (cont.)

**Acceptance Criteria:**
*   Given a functional role with granted permissions exists, when I assign it to employee X, then X can perform the granted actions, scoped to X's existing access role only.
*   The assignment never widens what data X can see about any person.

**Proposed description:**
Build the UI and data model to assign an existing functional role (defined per Story 1.4) to a specific person. This is the enforcement point for the single most important rule in Section 2: "Access roles (2.1) are not extensible this way. A new functional role never widens what data its holders can see about a person; it only unlocks features. Where a feature needs data, it operates within the holder's existing access role. A newly created role that can send campaigns sees the audience through the colleague view unless it also holds a Manager or People Partner relationship" (2.3). Concretely, when a person exercises a permission granted by an assigned functional role (e.g., creating a form campaign, creating an action item), any data that feature touches (audience selection, target list, profile links) must be filtered through that person's already-resolved access role (2.1) exactly as if they held no functional role at all — Colleague-level visibility by default, wider only if they independently hold Manager or PP access to the people in question. A person may hold multiple functional roles simultaneously (the model is additive on features, not on access).

- HR Admin assigns an existing functional role to person X entirely through the UI, and X gains the role's granted feature actions immediately.
- While exercising a role-granted feature, X's visibility into any person's data is bounded by X's existing resolved access role (2.1) — never widened by the functional-role assignment (2.3).
- Example: a person assigned the "create form campaigns" permission but holding no Manager/PP relationship to anyone builds an audience using only Colleague-visible fields (2.3, 4.12).
- A person can hold more than one functional role at once; each grants its own features independently.
- Removing a role assignment removes the granted feature actions for that person immediately.

**Given/When/Then:**
```
Scenario: A functional-role holder with no managerial access is scoped to the colleague view
Given Person X (IT department) is assigned the "Security Champion" role, which grants "create form campaigns", and X holds no Manager or PP relationship over anyone
When X builds a campaign audience using the All Employees filter engine
Then X can only filter and see the colleague-visible whitelist fields (S1, S10, S11) for candidate recipients, not management-only sections

Scenario: Assigning a functional role never grants data access
Given Person X is assigned a role granting "create/edit risks"
When X attempts to view or edit the risk section (S6) of a person X holds no Manager or PP access over
Then the action is denied, because the functional role grants the feature but not the underlying data access
```

## Story 1.6: Assemble Employee Profile by Section Access (ClickUp task: 869em1w46)
**Current description:** **As** any authenticated user, **I want** to open a profile and see only the sections I'm entitled to, **so that** the profile always respects the access matrix.

**FRs:** FR4, FR15

**Acceptance Criteria:**
*   Given I hold Manager access with respect to the profile owner, when I request the profile, then sections marked for Manager line are returned/rendered, and sections marked "—" are absent from both API and UI.
*   Given I hold no relationship (Colleague), when I request the profile, then only S1, S10 (with leave type) and S11 (project name only) are returned.
*   Access is resolved server-side, per section, on every request.

**Proposed description:**
Build the profile-assembly engine that, for a given viewer and a given profile owner, resolves the viewer's audience (Self, Manager line, PP, Colleague, Shared link, or HR Admin — 3.1) and returns exactly the sections and read/write levels that audience is entitled to per the section access matrix (3.2). "The employee profile is decomposed into sections. Access is granted per section, per audience. There is no 'profile-level' permission" (3.1) — this story is the implementation of that principle end-to-end (4.2). A section marked `—` for the resolved audience is not rendered and not returned by the API — this must hold for every audience/section combination in the 16-row matrix (3.2), not only the Colleague case (which Story 1.8 stress-tests specifically). Access must be evaluated relationship-by-relationship and per request, after resolving the requester's access role per 2.1 (3.3.4) — a manager for one profile section-set can simultaneously be a colleague elsewhere in the same session. Field-level granularity within a section also matters: e.g., S1 gives Self R overall but RW on the photo specifically, and S16 (custom fields) has its own per-field visibility layer (Story 1.10) nested inside the section-level rule.

- The profile endpoint/service assembles its response purely from the sections the resolved audience is entitled to, per row of the matrix (3.2) — this includes correctly applying R vs RW per section and per audience.
- Sections marked `—` for the resolved audience are absent from the API payload entirely (not empty/null), and not rendered in the UI (3.3.1).
- The engine correctly resolves all five non-admin audiences (Self, Manager line, PP, Colleague, Shared link) plus HR Admin (full access) for the same profile from different viewers (3.1).
- Access is resolved server-side, per section, on every single request — no client-cached role, no session-level role assumption (3.3.4).
- Field-level exceptions within a section (e.g., S1 photo RW for Self despite the section being R) are honored, not just section-level R/RW/— (3.2).

**Given/When/Then:**
```
Scenario: Manager-line viewer receives all Manager-line sections
Given Viewer V holds Manager access with respect to profile owner B
When V requests B's profile
Then the response includes S1–S16 at the Manager-line R/RW levels specified in 3.2, with no sections silently missing that the matrix grants

Scenario: Colleague viewer receives only whitelisted sections, with denied sections truly absent
Given Viewer V holds no Manager, PP, or Self relationship to profile owner B
When V requests B's profile via the API directly
Then the response contains only S1, S10 (including leave type) and S11 (project name only), and no keys for S2–S9, S12–S16 are present in the payload at all
```

## Story 1.7: Profile Header Shows Manager, PP and Mentor (ClickUp task: 869em1w6d)
**Current description:** **As** any viewer with profile access, **I want** to see the person's manager, people partner and mentor at the top of their profile, **so that** I understand their reporting context at a glance.

**FRs:** FR16

**Acceptance Criteria:**
*   Given a profile is rendered, when the header loads, then it shows the current manager, people partner, and mentor if assigned.

**Proposed description:**
Render, at the top of every profile, the person's current manager, people partner, and mentor — "the profile header shows the manager, the people partner and the mentor" (4.2). Manager and PP are identity-card content per S1 (3.2), and the mentor is described separately in the mentorship section: "On any profile: the mentor is displayed alongside the manager and the people partner in the profile header, visible to manager line and PP" (4.11). **Spec nuance to flag for the team:** S1's audience row grants Colleague `R` on the identity card, which nominally includes "manager, people partner, mentor" as listed content (3.2), yet 4.11 explicitly narrows the mentor field's header visibility to "manager line and PP" only. These two clauses are not obviously reconcilable as written. Implement 4.11 as authoritative for the mentor field specifically (it is the more specific, later rule), so Colleague and Self viewers may see manager/PP in the header per S1 but the mentor may need to be withheld from them per 4.11 — but raise this with the product owner for an explicit ruling rather than silently picking an interpretation, since a wrong guess is either an over-share (3.3.1 violation) or a missing feature. Each of the three header fields should link through to the relevant profile only when the viewer has access to view it (consistent with the sharing/whitelist rules elsewhere in the spec).

- The header renders current manager, current PP, and current mentor (if one is assigned) for any viewer with access to the profile (4.2).
- If no mentor is currently assigned, the mentor field is omitted or shown as blank — not an error state (4.11).
- Manager and PP fields are shown per S1's audience rules (3.2); the mentor field's visibility is implemented per 4.11's explicit "visible to manager line and PP" restriction, flagged to the PO as needing clarification against S1's Colleague `R` on identity-card content.
- The header reflects the live/current values of manager, PP and mentor and updates immediately after any reassignment (Stories 1.1–1.3, 4.11's mentor pairing flow).
- Links from the header to the manager's/PP's/mentor's own profile only work for a viewer who has access to that target profile — clicking through must not bypass the target's own access resolution (Story 1.6).

**Given/When/Then:**
```
Scenario: Manager-line viewer sees manager, PP and mentor in the header
Given Viewer V holds Manager access with respect to profile owner B, and B has an assigned manager, PP and mentor
When V opens B's profile
Then the header displays B's current manager, current PP, and current mentor

Scenario: Mentor field is withheld from an audience 4.11 does not name
Given Viewer V is a Colleague of profile owner B (no Manager, PP, or Self relationship), and B has an assigned mentor
When V opens B's profile
Then the header does not display B's mentor, per the 4.11 restriction to manager line and PP (pending PO confirmation against S1's Colleague identity-card access)
```

## Story 1.8: Enforce the Colleague Whitelist Everywhere (ClickUp task: 869em1w88)
**Current description:** **As** a Colleague of another employee, **I want** every surface — profile, API, export, search — to never reveal sections outside the whitelist, **so that** access can't leak through any path other than the rendered page.

**FRs:** FR5

**Acceptance Criteria:**
*   Given I am a Colleague, when I call the profile API directly, then only S1, S10 and S11 fields are present in the response payload.
*   Given I export or search as a Colleague, when results are returned, then the same whitelist applies — verified on the API response, not just the UI.

**Proposed description:**
Enforce the Colleague whitelist as a hard, server-side rule everywhere Colleague-audience data can be surfaced. "Colleague view is a whitelist, not a blacklist. A colleague sees exactly S1, S10 including leave type, and the S11 project name. Everything else is absent. Do not implement this by hiding fields in the frontend — the API must not return them" (3.3.3). This story is the Colleague-specific instance of the general leak rule (3.3.1) — every "—" cell for the Colleague column in the matrix (3.2: S2–S9, S12–S16) must not reach a Colleague viewer through the profile UI, the profile API, the All Employees list/export (4.1 "Colleague mode: same page, whitelist columns only"), search/filter results, or error messages. The enforcement must live in the data-access layer (query/service level), not in a serializer that strips fields after fetching real data, and not in the frontend component tree — a Colleague-mode API call must never even fetch, let alone transmit, out-of-scope fields. This story's negative-test suite is the Colleague-specific slice of the broader cross-audience suite in NEW B.

- The profile API response for a Colleague-resolved viewer contains only S1, S10 (with leave type) and S11 (project name only) keys — no other section keys present, empty, or null-valued (3.3.3).
- The All Employees list in Colleague mode exposes only whitelist columns as available columns/filters — attempting to request a non-whitelist column via API parameters (not just UI) is rejected or ignored, not partially honored (4.1, 3.3.3).
- `.xlsx` export performed as a Colleague contains only whitelist columns, verified on the exported file's actual columns, not just the export UI's checkbox list (4.1, 3.3.1).
- Search/typeahead results returned to a Colleague never include or allow filtering by out-of-scope fields, including as a means of indirect inference (3.3.1, 3.3.5's inference rule applied to sections generally).
- Enforcement is implemented at the data/query layer so no code path can accidentally serialize a restricted field to a Colleague-resolved response (3.3.3).

**Given/When/Then:**
```
Scenario: Colleague API call returns only whitelist fields
Given Viewer V is a Colleague of profile owner B
When V calls the profile API for B directly (bypassing the UI)
Then the response body contains only S1, S10 (with leave type), and S11 (project name only) keys, with no other section data present at any level

Scenario: Colleague-mode export never includes restricted columns
Given Viewer V is browsing All Employees in Colleague mode
When V exports the current view to .xlsx
Then the exported file's columns are limited to the whitelist fields, even if V attempts to add a non-whitelist column to the view configuration beforehand
```

## Story 1.9: Management Notes with Visibility Flags (ClickUp task: 869em1wb0)
**Current description:** **As** a UM, DM or PP responsible for a person, **I want** to create, read and edit free-form management notes about them, each carrying independent "visible for employee" and "visible for PM" flags defaulting off, **so that** sensitive notes stay private unless explicitly shared.

**FRs:** FR6, FR60

**Acceptance Criteria:**
*   Given I am the UM/DM/PP for employee B, when I create a note with both flags off, then B cannot see it and B's PM cannot see it.
*   Given I set "visible for PM" on a note, when B's PM (not the UM/DM/PP) views S7, then they see that note read-only and no others.
*   Given I set "visible for employee" on a note, when B views their own profile, then B sees that specific note only.
*   A PM's access to S7 remains read-only and flag-gated even though the PM is a Manager for every other section.

**Proposed description:**
Implement Section S7 (Management notes) as free-form notes authored by managers and PP, each carrying two independent flags, both off by default: *visible for employee* and *visible for PM* (3.2 row S7, 3.3.2). This is "the one documented exception to 'Manager sees everything' in 2.1: a PM is a Manager for every other section but a flag-gated reader for S7" (3.3.2) — so this story must deliberately special-case S7's access rule against the general Manager-line RW default that every other section uses. Full read/write on S7 (create, read, edit) belongs only to UM, DM and PP "responsible for the people they are responsible for," regardless of flags (3.3.2) — a PM never gets write access to S7, only a flag-gated, read-only view of records explicitly marked *visible for PM* (3.2, 3.3.2). The employee gets an even narrower read-only view: only records flagged *visible for employee*, and only that specific record — not the section generally (3.2, 4.3: "An employee cannot see... any management note without the visible for employee flag"). Both flags are per-record and independent of each other (a note can be visible to PM only, employee only, both, or neither). Unflagged records must be completely absent (not shown-but-locked) from both the employee's and the PM's views — this is a specific instance of the general no-leak rule (3.3.1) and should be covered by the negative-test matrix in NEW B.

- UM, DM and PP responsible for employee B can create, read and edit S7 notes about B regardless of flag state (3.3.2).
- A newly created note defaults both flags to off; in that state neither B nor B's PM can see it in any form (3.3.2).
- Setting *visible for PM* makes that specific record readable (read-only) to PMs in B's project chain; other unflagged records remain invisible to those PMs (3.3.2).
- Setting *visible for employee* makes that specific record readable (read-only) to B on their own profile; other unflagged records remain invisible to B (3.3.2, 4.3).
- A PM never has write access to S7, even for records flagged visible for PM — this is the one documented exception to "PM is a Manager for every other section" (3.3.2).
- Unflagged notes are absent (not present-but-hidden) from the API response for the employee and for the PM, consistent with the section's general no-leak rule (3.3.1).

**Given/When/Then:**
```
Scenario: Flag-off note is invisible to both employee and PM
Given PP creates a management note about employee B with both "visible for employee" and "visible for PM" left off
When B views their own profile, and separately B's PM views S7
Then neither B nor the PM can see that note, in the UI or via direct API request

Scenario: PM sees only PM-flagged notes, read-only, no write access
Given B's PM is not B's UM, DM, or PP, and one S7 note about B is flagged "visible for PM" while a second is not
When the PM opens B's S7 section
Then the PM sees exactly the one flagged note in read-only form, cannot edit it, and does not see the unflagged note at all
```

## Story 1.10: Per-Field Custom Field Visibility (ClickUp task: 869em1wda)
**Current description:** **As** an HR Admin, **I want** each custom field's visibility (management/employee/colleague) to gate its appearance in profile views, filters and list columns, **so that** a value I haven't been granted can't be inferred.

**FRs:** FR7

**Acceptance Criteria:**
*   Given a custom field is set to "management" visibility, when a colleague views or filters the All Employees list, then that field never appears as a column, filter option, or filterable value — including as a means to infer the value indirectly.

**Proposed description:**
Implement per-field visibility for custom fields (S16, 4.1), independent of, and layered inside, the section-level access rules. "Custom fields carry their own visibility. When a custom field is created, its visibility level is set: management (default), employee (also visible to Self), or colleague (also visible to everyone). Filters and list columns respect it: a user filtering the All Employees list must not be able to infer a value they cannot see" (3.3.5). This applies everywhere custom fields surface — the profile's S16 section, the All Employees list columns, saved views, filters, and export (4.1) — and must be enforced server-side like every other access rule (3.3.1, 3.3.4). The inference concern in 3.3.5 is the hard part of this story: it is not enough to hide a `management`-visibility field from a colleague's column list — the filter engine must also prevent the colleague from indirectly deriving the value, e.g., by filtering the list down to a count and inferring a boolean/select value from how the result set changes, or by sorting on a proxy the field influences. Since custom fields and their visibility are themselves data (S6: "Any field, including fields that do not exist yet, must be filterable and sortable... A column-per-field schema will not survive this"), the visibility check must be a generic, field-agnostic gate applied at query time, not a per-field hardcoded rule.

- New custom fields default to `management` visibility unless explicitly set otherwise at creation (3.3.5).
- Three visibility levels are supported: `management` (management audiences only), `employee` (adds Self), `colleague` (adds everyone) (3.3.5).
- A field below a viewer's visibility level never appears as a profile field, list column, filter option, or export column for that viewer (3.3.5, 3.3.1).
- The filter/list engine prevents indirect inference of a hidden field's value (e.g., via count changes when toggling a filter the viewer cannot see) — this needs explicit test coverage beyond simple field hiding (3.3.5).
- Changing a field's visibility takes effect immediately across all surfaces (profile, list, filter, export), consistent with the "no stale grant" principle applied elsewhere (2.3, 3.3.4).

**Given/When/Then:**
```
Scenario: Employee-visibility field is shown to Self but not to a Colleague
Given HR Admin creates a custom field "Dietary preference" with visibility set to "employee"
When the field's owner (Self) views their own profile, and separately a Colleague views the same profile
Then Self sees the field and its value, and the Colleague does not see the field at all — not on the profile, in list columns, or in filters

Scenario: A management-only field cannot be inferred via the filter engine
Given a custom field "Performance flag" has visibility "management" and a Colleague is browsing All Employees
When the Colleague attempts to construct a filter or sort referencing that field via direct API parameters
Then the request is rejected or the field is silently excluded from filterable/sortable options, and the resulting list is unaffected in any way that would reveal the field's values
```

## Story 1.11: Generate a Shareable Profile Link (ClickUp task: 869em1wfu)
**Current description:** **As** a manager, **I want** to generate a shareable view of an employee's profile for someone without Manager/PP access, **so that**, for example, a DM can evaluate an internal candidate they don't yet have access to.

**FRs:** FR37

**Acceptance Criteria:**
*   Given I am creating a share link, when I select sections to include, then S2, S5, S6 and S8 default to excluded and must be explicitly enabled, and S3, S7 and S13 are never offered.
*   The resulting link never grants write access.

**Proposed description:**
Build profile sharing (4.8): "A manager generates a shareable view of an employee's profile for someone who does not hold Manager or People Partner access over that person — typically a DM evaluating a proposed candidate" (4.8), which is the mechanism resourcing request review relies on when "the link leads to their profile — which the DM may not yet have access to" (4.7). Only sections marked `cfg` in the matrix (3.2) can ever be offered as share options; the creator picks which of those to include on this specific link (4.8). Among the `cfg` sections, the sensitive ones — S2 (personal contacts), S5 (documents), S6 (risks), S8 (feedbacks) — must default to excluded on every new link and require explicit, per-link enabling; there is no "remember my last configuration" that pre-enables them (4.8). S3 (emergency contacts), S7 (management notes) and S13 (mentorship) are never offered as options at all — the matrix marks their Shared-link column as `—`, meaning they cannot be shared under any configuration (3.2, 4.8: "S3, S7 and S13 can never be shared"). A generated link is read-only regardless of which sections are enabled — "a shared link never grants write access" (4.8) — and whatever sections are enabled are still bounded by the Shared-link column's own R/`cfg` semantics in 3.2 (e.g., S1 is `R` on a shared link even though the creating manager may see S1 as RW). Only someone who already holds Manager or PP access to the profile can create a link for it (4.8).

- The share-link creation UI/API only ever offers sections marked `cfg` in the matrix (3.2); S3, S7 and S13 are not present as selectable options under any circumstances (4.8, 3.2).
- S2, S5, S6 and S8 are unchecked/excluded by default on every new link and must be explicitly toggled on each time — no persisted default that pre-enables them (4.8).
- The resulting shared-link view is strictly read-only for every enabled section, with no write affordances rendered or accepted by the API (4.8).
- Enabled sections are rendered/returned at the Shared-link column's access level from 3.2, not at the creating manager's own (possibly higher) access level.
- Only a viewer already holding Manager or PP access over the target profile can generate a link for it (4.8).

**Given/When/Then:**
```
Scenario: Manager creates a link with sensitive sections left at their default
Given a UM holds Manager access to employee B and opens the share-link creation flow for B
When the UM enables S1 and S9 only, leaving S2/S5/S6/S8 at their default
Then the created link exposes S1 and S9 in read-only form, and S2, S5, S6, S8 are excluded because they were not explicitly enabled

Scenario: Never-shareable sections cannot be selected at all
Given a UM is configuring a share link for employee B
When the UM looks for S3 (emergency contacts), S7 (management notes), or S13 (mentorship) in the section picker
Then none of those sections appear as selectable options, and no API request can force them onto the link
```

## Story 1.12: Shared Link Expiry, Logging and Revocation (ClickUp task: 869em1wj1)
**Current description:** **As** a manager who shared a profile link, **I want** it to expire, be logged on every access, and be revocable, **so that** exposure is time-bound and auditable.

**FRs:** FR38

**Acceptance Criteria:**
*   Given a link is created with no custom expiry, when 24 hours pass, then the link stops working.
*   Given someone opens the link, when the access occurs, then a log entry records when and from where.
*   Given I revoke the link before expiry, when anyone next opens it, then access is denied.

**Proposed description:**
Implement the lifecycle controls that make profile sharing (Story 1.11) time-bound and auditable, per 4.8: "The link expires. Default 24 hours, configurable at creation. Every access via the link is logged: when, from where. Revocable before expiry." Expiry defaults to 24 hours from creation but must be configurable per link at creation time (4.8). Every single access via the link — not just the first — must be logged with a timestamp and origin ("from where," e.g. IP address or comparable identifying info) (4.8), giving the creating manager an audit trail of who viewed the shared profile and when. The link's creator can revoke it explicitly before its natural expiry, and revocation must take effect immediately — any subsequent access attempt, even one already in flight, must be denied (4.8). An expired or revoked link should fail closed and should not leak information through its failure mode (e.g., should not distinguish "expired" vs "revoked" vs "never existed" in a way that helps an attacker enumerate valid links) — consistent with the general no-leak-via-any-surface principle (3.3.1) applied to error responses.

- A link created without a custom expiry stops granting access exactly at 24 hours after creation (4.8).
- Expiry duration is configurable per link at creation time, overriding the 24-hour default (4.8).
- Every access attempt through the link — successful or not — writes a log entry capturing at minimum when it occurred and where it originated from (4.8).
- The link creator can revoke an active link before its expiry; the very next access attempt after revocation is denied (4.8).
- The creator can view the access log for a link they created (supports the "auditable" half of the story's rationale).
- Expired/revoked/invalid link access attempts return a uniform, non-information-leaking response (3.3.1 principle applied to this surface).

**Given/When/Then:**
```
Scenario: Link expires automatically after the default window
Given a share link is created with no custom expiry specified
When 24 hours elapse from creation and someone opens the link
Then the link no longer grants access to the profile

Scenario: Revoking a link before expiry blocks all further access
Given a share link is active and has not yet expired
When the creating manager revokes the link, and someone then attempts to open it
Then access is denied, and the attempt is logged as a denied access
```

## Story NEW A: Cache Access Resolution Safely and Revoke Immediately on Project-Assignment End (ClickUp task: NEW)
**Proposed description:**
This story exists to reconcile two requirements that pull in opposite directions and currently have no settled resolution in the spec: the performance requirement that "the All Employees list with 500+ records, arbitrary filters and derived fields responds within 2 seconds, including permission resolution" (Section 7), and the correctness requirement that managerial access ends the instant a project assignment ends — "managerial access is not sticky" (2.1, last consequence bullet) — under the explicit warning that "a stale permission cache is a data leak" (Section 6, data model notes). Naive per-request recomputation of the full transitive closure over both the reporting graph and the project-assignment graph (2.1, Section 6) for every row of a 500+-row list, on every request, is unlikely to hit the 2-second bar without some form of caching or precomputation — but any cache risks serving stale "Manager" access to someone whose project assignment just ended, which the spec calls a data leak, not a performance tradeoff. **This story does not hand down a final answer.** It proposes a concrete starting approach for the team to evaluate and sign off on before implementation: a short-TTL, per-subject access cache keyed by `(subject_id)`, tagged with a monotonically increasing generation/version counter maintained on the relationship graph (reporting hierarchy + project assignments + PP assignments). Any mutation to that graph — a manager reassignment, a project assignment created or ended, a PP reassignment — bumps the generation counter, which invalidates every cached entry derived from an older generation on next read, giving effectively-immediate invalidation without a full synchronous recompute of every cached entry. This should be treated as a design decision requiring explicit team/architecture sign-off, not as a spec requirement to implement as described — alternative approaches (e.g., precomputed materialized access-edge tables refreshed synchronously on graph writes, or per-request computation with heavy indexing and no caching at all) should be evaluated against the same two constraints before committing.

- The All Employees list responds within 2 seconds for 500+ records with arbitrary filters, including full permission resolution for the requesting viewer (Section 7).
- Ending a project assignment (or a manager/PP reassignment) is reflected in access resolution on the very next request with no observable staleness window, regardless of whatever caching layer exists (2.1, Section 6).
- A concrete caching proposal (short-TTL, per-subject cache + generation/version counter invalidated on any relationship-graph write) is documented as a starting point and explicitly flagged in the story/ticket as requiring team and architecture sign-off — it is not to be treated as a settled design (Section 6, Section 7).
- Test coverage includes a scenario that specifically proves immediate revocation holds under whatever caching strategy is chosen: end a project assignment and immediately re-query access for the same subject/object pair in the same test, asserting no stale grant (2.1, Section 6, DoD).
- Bulk access resolution for list/dashboard views avoids N+1 per-row relationship-graph queries (batched resolution, precomputed edge sets, or equivalent).
- The chosen caching/invalidation approach exposes basic observability (cache hit rate, invalidation event counts) so the team can verify in production that it is neither stale nor a bottleneck.

**Given/When/Then:**
```
Scenario: All Employees list meets the performance bar at scale
Given the workspace has 500+ employee records and the viewer applies several arbitrary filters and derived-field sorts
When the viewer loads the All Employees list
Then the response, including full permission resolution for every visible row, returns within 2 seconds

Scenario: Access is revoked immediately after a project assignment ends, even with caching active
Given PM P holds Manager access to employee B solely via B's assignment to Project X, and the access-resolution cache is warm for this pair
When B's assignment to Project X ends and P immediately re-requests B's profile
Then P's request is denied Manager-level access to B, with no stale cached grant served
```

## Story NEW B: Prevent Section Leaks Across All Surfaces for Every Denied Audience (ClickUp task: NEW)
**Proposed description:**
This story generalizes the no-leak rule beyond the Colleague-specific case already covered by Story 1.8. Rule 3.3.1 states: "Every cell in the matrix is strict. A section marked — for an audience must not reach that audience through any surface: not the UI, not the API, not an export, not a notification, not a search result, not an error message. The same applies to flag-gated records. A leak is a critical defect, whichever section it happens in" (3.3.1). This applies to *every* `—` cell and every flag-gated case in the full matrix (3.2), across *every* audience, not only Colleague-vs-restricted-sections. Concrete examples the spec and matrix imply: **Self and S6/S15** — an employee must never see their own risk record (S6 is `—` for Self, and explicitly reinforced in 4.3: "An employee cannot see their risk level...") or their own request history (S15 is `—` for Self); **a PM and an unflagged S7 record** — a PM must never see a management note that isn't flagged *visible for PM* (3.3.2, covered functionally in Story 1.9 but needing cross-surface verification here); **everyone and S3/S7/S13 via a shared link** — these three sections must never appear through a shared link under any configuration, for any link viewer (3.2, 4.8). This story's job is to build a systematic, matrix-driven negative-test harness — not a fresh implementation of access rules (those live in Stories 1.6–1.11) — that walks every `—` cell and every flag-gated case in 3.2/3.3.2/3.3.5 and asserts absence across every surface the spec names: API response, UI render, `.xlsx` export, notifications (4.13, if implemented), and search/filter results. This is also the story that operationalizes the Definition-of-Done clause: "access control is covered by tests per audience, per relationship path and per section, including negative tests for every — cell, for unflagged S7 records against both the employee and a PM, and for the colleague whitelist" (Section 9).

- A test matrix is derived directly from the section access matrix (3.2): every `—` cell (e.g., Self×S6, Self×S15, Colleague×S2–S9/S12–S16, Shared-link×S3/S7/S13) and every flag-gated case (S7's employee/PM flags per 3.3.2, S8's shared-with-employee flag) is enumerated as a distinct test case, not hand-picked ad hoc (3.2, 3.3.1, Section 9).
- For each enumerated case, the test asserts the section/record's literal absence — not a null, not an empty array, not a masked value — from the API response, so a denied audience cannot infer existence from payload shape (3.3.1).
- The same absence assertion is repeated, per case, against the profile UI render, `.xlsx` export output, search/filter results, and notification content where 4.13 is implemented (3.3.1).
- The shared-link surface is specifically exercised: S3, S7 and S13 never appear in a shared-link response or UI under any link configuration, including attempts to force them via direct API manipulation of the link's section list (4.8, 3.2).
- The harness is data-driven off the matrix definition itself (rather than one hardcoded test per cell written by hand), so that a future change to the matrix (3.2) automatically extends or flags outdated coverage.
- Any leak surfaced by this suite fails the build — per Section 9's Definition of Done, "a leak is a critical defect, whichever section it happens in" (3.3.1) is treated as build-breaking, not a warning.

**Given/When/Then:**
```
Scenario: Self never sees their own risk or request-history sections
Given Employee E requests their own profile
When the profile response is inspected
Then S6 (risks) and S15 (request history) are entirely absent from the payload — no keys, no empty placeholders — consistent with the — cells for Self in those rows

Scenario: A shared link never exposes never-shareable sections, even via direct manipulation
Given a DM opens a shared link created with only S1 and S9 enabled
When the DM inspects the rendered page and also calls the underlying API directly, attempting to request S3, S7, or S13 by section id
Then S3, S7 and S13 are absent from both the UI and the API response under all circumstances, regardless of the requested parameters
```

---

# Epic 2: Self-Service

## Story 2.1: View Own Employment Summary (ClickUp task: 869em1wp6)
**Current description:** "As an employee, I want to see my own grade, position, seniority, employment type and English level, so that I have visibility into my own employment record.

FRs: FR17

Acceptance Criteria:
Given I open my own profile, when S4 renders, then I see these fields read-only."

**Proposed description:**
Implements the Self column of the S4 Employment section (3.2, row S4) as part of self-service (4.3). On their own profile, the employee sees their employee type (FTE / Subcontractor), grade, seniority, English level, and employment status/contract type, all rendered read-only — S4 is `R` for Self, never `RW` (3.2). Position and position history are also part of S4; per the matrix the whole section is read-only for Self, so position history should be displayed as read-only, not editable.

Business rules and edge cases:
- Access is resolved server-side per section (3.3 rule 4): the API response for "self" must include S4 fields regardless of any other role the viewer holds elsewhere, but always as read-only for Self — do not accidentally grant RW because the same user is also a UM/PP for other profiles.
- Probation status is also part of S4's contents per the matrix and should be included in the read-only rendering even though not explicitly named in the story text.
- Temporal fields (grade, position, department, employment type) are time-bounded records under the hood (6, "Temporal data") — the profile shows the *current* value; history is surfaced via the career timeline (S9 / 4.9), not duplicated here.
- Out of scope: editing any S4 field as Self (S4 is RW only for Manager line and PP, 3.2); compensation/salary data does not exist on the profile at all (10, Out of scope).
- This section must not silently disappear or error if a field is null (e.g., no English level recorded yet) — render the section with an empty/placeholder value, not omit the whole section.

Acceptance criteria:
- Given I open my own profile, when S4 renders, then I see grade, position, seniority, employment type, English level, probation status and employment/contract type as read-only text (no edit affordance).
- Given I have no edit permission on S4 as Self, when I inspect the API response for my own profile, then no write endpoint/action for S4 fields is exposed to me.
- Given a field in S4 has no value set, when the section renders, then it displays a clear empty state rather than omitting the section.
- Given I am also a UM/PP for a different profile, when I view my *own* profile, then S4 is still read-only for me (role scoping is per-relationship, 2.1).

**Given/When/Then:**
```
Scenario: Employee views their own read-only employment summary
Given I am logged in as an employee viewing my own profile
When the S4 Employment section renders
Then I see my grade, position, seniority, employment type, English level and contract type
And none of these fields are editable

Scenario: Employee cannot edit S4 even via a direct API call
Given I am logged in as an employee
When I attempt to submit a write request to update my own grade or position via the API
Then the request is rejected because S4 is read-only for the Self audience (3.2)
```

## Story 2.2: Edit Own Personal and Emergency Contacts (ClickUp task: 869em1wqr)
**Current description:** "As an employee, I want to view and edit my personal contacts, residential address, place of stay and emergency contacts myself, so that I don't have to ask HR for routine updates.

FRs: FR18

Acceptance Criteria:
Given I edit my personal phone number, when I save, then S2 reflects the change immediately, visible to me (RW), my manager line (R) and my PP (RW).
Given I edit my emergency contacts, when I save, then S3 updates the same way, visible only to me (RW) and my PP (RW) — never to a colleague or beyond read for manager line."

**Proposed description:**
Implements Self = `RW` on S2 Personal contacts and S3 Emergency contacts (3.2), explicitly called out in 4.3 ("see and edit personal contacts, residential address, place of stay, emergency contacts — without asking HR"). This is the only self-service section pair with full write access for the employee (contrast with S1 photo-only RW, and read-only S4/S9/S10/S11/S12).

Business rules and edge cases:
- S2 fields: personal phone, personal email, messengers, residential address, current place of stay. Access after edit: Self `RW`, Manager line `R`, PP `RW`, Colleague `—` (not rendered, not returned by API — 3.3 rule 1 and rule 3, colleague whitelist only covers S1/S10/S11).
- S3 fields: contact person, relationship, phone. Access after edit: Self `RW`, Manager line `R` (per matrix, not "beyond read"), PP `RW`, Colleague `—`, and S3 is never shareable via a profile-sharing link (4.8: "S3, S7 and S13 can never be shared").
- Edits must take effect immediately and be visible to entitled audiences on their next fetch — no caching staleness (6, "stale permission cache is a data leak" applies broadly; also apply to data caching for these sections).
- Multiple contact methods (e.g., several messengers) and potentially multiple emergency contacts should be supported as add/edit/remove, not a single fixed field — confirm data shape with the team's data model, but the story should not assume exactly one of each.
- Out of scope: this story does not grant Self any access to S1 core identity fields beyond photo, nor to S4/S5/S6 etc. Only S2 and S3 are affected here.
- Validation: basic format validation (phone, email) should occur client- and server-side; do not block on a save if optional fields (e.g., "place of stay") are empty, since not all fields are mandatory per the spec.

Acceptance criteria:
- Given I edit my personal phone number and save, then S2 reflects the change immediately for me (RW), is visible read-only to my manager line, and RW to my PP.
- Given I edit my emergency contacts and save, then S3 updates the same way and is visible only to me (RW) and my PP (RW); it is never visible, in any form, to a colleague, and is read-only (not editable) for manager line.
- Given a colleague (no manager/PP relationship to me) requests my profile via any API surface, then S2 and S3 are entirely absent from the response, not merely hidden in the UI (3.3 rule 1 and 3).
- Given a manager attempts to edit my S3 emergency contact via the API, when the request is submitted, then it is rejected because manager line only holds `R` on S3.
- Given my profile is shared via a shared-link (4.8), when the link is generated, then S3 cannot be enabled/included under any configuration.

**Given/When/Then:**
```
Scenario: Employee edits their own emergency contact
Given I am logged in as an employee viewing my own profile
When I update my emergency contact's phone number and save
Then S3 reflects the new phone number immediately
And my assigned people partner can see the same updated value with RW access
And my manager line can see the updated value read-only

Scenario: Colleague cannot see or infer emergency contact data
Given I am logged in as a colleague with no manager or people-partner relationship to this employee
When I request this employee's profile through the API
Then the S3 emergency contacts section is entirely absent from the response
And no field, filter, export or error message reveals its existence or values
```

## Story 2.3: Upload Photo and Certificates (ClickUp task: 869em1wtd)
**Current description:** "As an employee, I want to upload my own profile photo and certificates, so that my profile stays current.

FRs: FR19

Acceptance Criteria:
Given I upload a new photo, when it saves, then it replaces the prior photo (S1, photo is RW for Self).
Given I upload a certificate, when it saves, then it appears under my Documents (S5), where I otherwise have read-only access."

**Proposed description:**
Implements the two narrow write exceptions carved out of otherwise read-only sections for Self, per 3.2 and 4.3 ("upload a photo" / "upload certificates"):
- S1 Identity card is `R` for Self overall, but photo specifically is `RW` — the employee can replace their own photo but cannot edit any other S1 field (name, position, department, manager, etc. remain read-only/system-managed for Self).
- S5 Documents is `R (own)` for Self overall, with an explicit carve-out to upload certificates — the employee can add certificate files but cannot upload or edit contract, W8, cooperation form, Diia City, CV, or joining interview feedback, and cannot edit/delete existing documents of any type, including certificates they previously uploaded (spec grants "upload," not "manage").

Business rules and edge cases:
- Photo upload replaces the prior photo (single current photo per employee); consider whether prior photos need retention for audit — the spec only requires replacement behavior, not history.
- Certificate upload appends to the Documents list under S5; it does not replace or modify any other document type in that section.
- File type/size constraints aren't specified — apply sane defaults (image formats for photo; common document formats for certificates) and flag as a decision to confirm with design, but do not block delivery on this being unspecified in the source spec.
- Access after upload: photo is visible per S1's normal matrix (Self R, Manager line RW, PP RW, Colleague R, shared-link on by default). Certificates are visible per S5 (Self R, Manager line R, PP RW, Colleague —, shared-link cfg).
- Out of scope: editing/deleting an uploaded certificate or photo history, uploading any other S5 document type as Self, editing any other S1 field as Self.

Acceptance criteria:
- Given I upload a new photo, when it saves, then it replaces my prior photo and the new photo is immediately visible wherever S1 is rendered for any entitled audience.
- Given I upload a certificate, when it saves, then it appears under my Documents (S5) alongside any existing documents, without altering other document entries.
- Given I attempt to upload or edit a non-certificate document type (e.g., contract, CV) as Self, when I submit the request, then it is rejected — S5 write access for Self is scoped to certificates only.
- Given I attempt to edit any S1 field other than photo (e.g., my department or position) as Self, when I submit the request, then it is rejected — S1 write access for Self is scoped to photo only.
- Given I upload a certificate, when a colleague (no manager/PP relationship) views my profile, then S5 including the new certificate remains entirely absent for them (3.2, S5 = "—" for Colleague).

**Given/When/Then:**
```
Scenario: Employee replaces their own profile photo
Given I am logged in as an employee viewing my own profile
When I upload a new photo
Then it replaces my previous photo immediately
And my manager line, PP and colleagues (per S1 access) see the updated photo

Scenario: Employee cannot upload a non-certificate document type
Given I am logged in as an employee viewing my own Documents (S5) section
When I attempt to upload a contract or CV file instead of a certificate
Then the upload is rejected
And only certificate uploads succeed for the Self audience on S5
```

## Story 2.4: View Own Timeline, Leaves, Projects, CDS and Mentorship (ClickUp task: 869em1wug)
**Current description:** "As an employee, I want to see my own career timeline, leave balances (with a timetracker link), current projects, and CDS section, and manage my mentorship status, so that I can track my own progress.

FRs: FR20 (part 1)

Acceptance Criteria:
Given I open self-service, when each section loads, then I see my timeline (S9), leaves with a timetracker link (S10), projects (S11) and CDS (S12) as read access.
I can mark my own IDP complete and manage my own open-to-mentoring flag (S13)."

**Proposed description:**
Implements Self read access to S9, S10, S11, S12 plus the two narrow write exceptions on S12 (IDP complete) and S13 (open-to-mentoring flag), per 3.2 and 4.3.

Per-section rules:
- **S9 Career timeline** — system-generated event log (4.9): joining, grade/position/department changes, FTE↔subcontractor transitions, extended leave, mentorship pair start/end. Self is `R` only; the employee cannot edit or add events (manual override is PP/UM-only, 4.9). Must render as a chronological, typed timeline.
- **S10 Leaves and absences** — vacation, sick, parental, extended leave, dates and types. Self is `R`, sourced from the timetracker integration (5.1). Include a link out to the timetracker page so the employee can *manage* (request/modify) leave there — the platform itself does not let them create or edit leave records (4.3: "see their own leaves and balances, with a link to the timetracker page to manage them"). Degrade gracefully if the timetracker integration is unavailable (7, "external integration failures degrade gracefully and never take down the application").
- **S11 Projects** — project, PM, DM, period. Self is `R`. Project assignment and workload-percentage are not modelled this iteration (10, Out of scope) — do not add allocation fields.
- **S12 CDS** — skills matrix link (resolved from the department+position dictionary, 4.10), assessment log (date, assessor, result link, final conclusion), and IDP. Self is `R` on the whole section **except** the IDP completion checkbox, which the employee can tick themselves (4.10: "the employee can read the section and mark their own IDP complete"). Ticking it records and displays a completion date; an IDP with no completion date remains "open." The employee cannot create/edit assessment records or conclusions, cannot edit the IDP description/deadline/file link — only toggle complete.
- **S13 Mentorship** — Self has `RW` on their own open-to-mentoring flag, and `R` on their pairs (assigned mentor, assigned mentee(s)) (3.2; 4.11 "On the employee's own profile"). The employee cannot self-assign a mentor or mentee — pairing is a manager/PP action (4.11). S13 can never be shared via a profile-sharing link (4.8).

Business rules and edge cases:
- CDS skills matrix link must resolve to the employee's *actual/current* department and position — if department/position changes, the link should reflect the new mapping, not a stale one.
- Toggling open-to-mentoring off should not affect an already-active mentor/mentee pairing; ending a pair is a manager/PP action (4.11) that the employee cannot trigger themselves.
- If the employee has no assigned mentor and no mentees, S13 still renders with an empty/appropriate state and the toggle control.
- Out of scope for this story: editing CDS assessment records/conclusions, editing IDP fields other than the complete checkbox, creating/ending mentorship pairs, seeing mentoring goals/session logs/progress tracking (10, explicitly out of scope for the whole platform).

Acceptance criteria:
- Given I open self-service, when each section loads, then I see my career timeline (S9), leaves with a timetracker link (S10), projects (S11) and CDS (S12) as read-only.
- Given I view S12, when I tick the "complete" checkbox on my own IDP, then a completion date is recorded and displayed, and the IDP is no longer shown as "open."
- Given I view S13, when I toggle "open to mentoring," then my flag updates and is reflected wherever S13 is shown to my manager line/PP; my assigned mentor and mentees (if any) are shown read-only.
- Given the timetracker integration is temporarily unavailable, when I open S10, then the section degrades gracefully (clear message, no crash) rather than breaking the whole profile page.
- Given I attempt to edit an assessment conclusion or IDP deadline on S12 as Self, when I submit the request, then it is rejected — only the IDP complete checkbox is writable for Self.

**Given/When/Then:**
```
Scenario: Employee marks their own IDP complete
Given I am logged in as an employee viewing my own CDS section (S12)
When I check the "complete" box on my IDP
Then the completion date is recorded and displayed alongside the deadline
And the IDP is no longer shown as open

Scenario: Employee cannot edit CDS assessment data
Given I am logged in as an employee viewing my own CDS section (S12)
When I attempt to edit an assessment's final conclusion
Then the edit is rejected
And the conclusion remains only editable by my manager line or PP
```

## Story 2.5: View Shared Feedback, Flagged Notes and Own Action Items (ClickUp task: 869em1wx2)
**Current description:** "As an employee, I want to see feedback explicitly shared with me, management notes flagged visible-for-employee, and my own action items which I can mark complete, so that I only see what's meant for me and nothing more.

FRs: FR20 (part 2), FR21

Acceptance Criteria:
Given a feedback record has "shared with employee" set, when I view S8, then I see it, and unshared records are absent.
Given I have an action item assigned to me, when I mark it complete, then its status updates and the completion date is recorded.
Given I have a risk record or an unflagged management note, when I view my own profile, then neither is visible anywhere on it (negative test for FR21)."

**Proposed description:**
Implements Self access to S7 (flag-gated), S8 (flag-gated) and S14 (own items, with completion), and enforces the two hard negative boundaries called out at the close of 4.3: an employee can never see their own risk level (S6), and can never see a management note (S7) unless it is individually flagged *visible for employee*.

Per-section rules:
- **S7 Management notes** — Self is `R`, and *only* for records where the *visible for employee* flag is explicitly set (default off, 3.2, 3.3 rule 2). This is record-level, not section-level: some notes about the employee may be visible, others not, in the same list. The employee never sees the *visible for PM* flag or any note lacking the employee flag, regardless of who authored it.
- **S8 Feedback** — Self is `R`, only for records flagged *shared with employee* (default is *management only*, 4.15). Unshared records must be entirely absent from the response, not shown redacted. Records should support viewing over time (4.15, "viewable over time, with comparison between periods") to the extent that applies to the employee's own shared subset.
- **S14 Action items and tasks** — Self has `R` on their own items plus the ability to mark complete. Marking complete records and displays a completion date (4.5 lifecycle: `open` → `completed`). The employee cannot edit title/description/due date/assignee, and cannot cancel an item (cancellation with reason is author-only, 4.5). Items generated by form campaigns (4.12) appear the same way and are completed by the employee following the external link then marking the item done — the system does not verify the external form was actually filled in.

Business rules and edge cases (hard negatives, cite 4.3 closing paragraph and 3.3):
- **Risk (S6) is never visible to Self under any circumstance** — 3.2 marks S6 Self as `—`, and 4.3 explicitly states "An employee cannot see their risk level." This must hold across UI, API, exports, notifications, and search/filter results (3.3 rule 1); it is a critical-defect-class leak if violated.
- **Unflagged S7 notes are invisible to Self** — this is record-level filtering, not a section toggle: if 9 of 10 notes about the employee are unflagged and 1 is flagged, the employee sees exactly 1.
- **Unshared S8 feedback is invisible to Self** — same record-level filtering pattern as S7.
- Per 3.3 rule 1, none of the withheld data may leak through indirect surfaces either: no notification content, no error message revealing a note/feedback/risk record exists, no aggregate count on a dashboard that implies a hidden record's presence.
- Out of scope: employee editing or authoring feedback/management notes about themselves (S7/S8 are `RW` only for manager line/PP); employee cancelling their own action items; employee seeing S15 request history (marked `—` for Self in 3.2, not part of this story or self-service at all).

Acceptance criteria:
- Given a feedback record about me has "shared with employee" set, when I view S8, then I see it; given another record lacks that flag, then it is absent from my view entirely (not just hidden in the UI).
- Given a management note about me has "visible for employee" set, when I view S7, then I see that specific record read-only; notes without the flag are absent, even if other notes about me are visible.
- Given I have an action item assigned to me, when I mark it complete, then its status changes to `completed` and a completion date is recorded and displayed; I cannot edit its title, due date, or cancel it.
- Given I have a risk record, when I view any part of my own profile (including dashboards, notifications, exports), then no risk level, trend, or history is visible or inferable anywhere.
- Given a management note about me exists with only the "visible for PM" flag set (not "visible for employee"), when I view S7, then that note is absent from my view.
- Given I query or filter my own action items, when the API responds, then only my own items are returned, with no fields belonging to S6/S7(unflagged)/S8(unshared)/S15 attached or inferable.

**Given/When/Then:**
```
Scenario: Employee sees only explicitly shared feedback and flagged notes
Given a feedback record about me is flagged "shared with employee"
And a separate management note about me is flagged "visible for employee"
And another management note about me has no visibility flags set
When I open my own profile and view S7 and S8
Then I see the shared feedback record and the flagged management note
And the unflagged management note does not appear anywhere on my profile

Scenario: Employee can never see their own risk level
Given a risk record exists for me at level "medium"
When I view any section of my own profile, including dashboards and notifications
Then no risk level, trend arrow, or risk history is displayed or otherwise inferable
And this holds even though S6 exists and is populated in the system
```

---

# Epic 3: All Employees Directory & Custom Fields

## Story 3.1: Sortable, Filterable Employee List (ClickUp task: 869em1wz7)
**Current description:** "**As** a manager or PP, **I want** a tabular list of employees I can sort and filter by any profile field, including derived fields, **so that** I can find the people I need.

**FRs:** FR8, FR9

**Acceptance Criteria:**
*   Given the list loads, when I sort by a column, then rows reorder accordingly.
*   Given I filter by "years with company > 3", when results render, then only matching people appear, computed from join date.
*   Every built-in field I have access to is available as both filter and column (custom fields become available once Story 3.2 lands)."

**Proposed description:**
Build the All Employees / Our Team list page (4.1): a single tabular page serving managers, people partners and ordinary employees, where the data returned differs by audience per the Section 3 access matrix, not the page itself. The list must support sorting on any column and filtering on any field the requester is entitled to see, including built-in profile fields (country, join year, gender, department, grade, position, employee type, English level, risk level, mentorship status, project) and, once Story 3.2 lands, every custom field (4.1).

Derived fields must be first-class: "years with company" is not stored, only join date is, but it must be filterable and sortable as a number exactly as if it were a stored field (4.1). The same principle applies to any other computed field added later (e.g., "has an open IDP" from 4.10) — the filter engine must not special-case derived vs. stored fields in a way that limits operators available to derived ones.

Because a column-per-field schema cannot support arbitrary/future filtering (Section 6), the underlying query layer must be built on a model that treats fields (including custom ones) as queryable metadata, not as hardcoded table columns — this is a foundational decision this story depends on architecturally, even though schema design itself is not this story's deliverable.

All filtering and sorting is evaluated server-side after resolving the requester's access role (3.3.5): a field marked as `—` or gated by custom-field visibility for this viewer must not be filterable, sortable, or inferable from filter results (3.3.3). Performance target: 500+ records, arbitrary filters and derived fields, response within 2 seconds including permission resolution (Section 7).

- Every built-in field the current user has access to (per 3.2) is usable as both a filter and a column.
- Sorting any column reorders rows correctly, including derived and (later) custom fields.
- A numeric/comparison filter (e.g., "years with company > 3") returns only matching people, computed live from join date.
- A field the viewer has no access to cannot be selected as a filter or column, and cannot be inferred from result sets (e.g., via range-filter probing).
- List responds within 2 seconds for 500+ employees with arbitrary filters applied (Section 7).
- Filtering/sorting behavior is identical in shape whether the field is stored, derived, or (in future) a custom field.

**Given/When/Then:**
```
Scenario: Filter by a derived field
Given the employee list contains people with only a stored join date
When a manager filters by "years with company > 3"
Then only employees whose computed tenure exceeds 3 years are returned, calculated from join date

Scenario: Filter excludes fields outside the viewer's access
Given a colleague-level user with no Manager or PP relationship to anyone in the list
When they open the field picker for filters and columns
Then only whitelist fields (S1, S10 type, S11 project name) are offered, and no restricted field appears or can be queried via the API
```

## Story 3.2: Define Custom Fields at Runtime (ClickUp task: 869em1x0h)
**Current description:** "**As** an HR Admin or manager, **I want** to define a new custom field and set values on profiles, **so that** new data needs don't require a deploy or migration.

**FRs:** FR10

**Acceptance Criteria:**
*   Given I create a new custom field (text/number/date/single-select/multi-select/boolean), when I save it, then it's immediately usable as a filter and column with no developer involvement."

**Proposed description:**
Allow HR Admin and managers to define new custom fields on the employee profile at runtime, entirely through the UI, with no deploy, no schema migration and no developer involvement (4.1, Section 6). Supported field types: text, number, date, single-select, multi-select, and boolean (4.1). Once defined, a field must be immediately settable on individual profiles and immediately usable as a filter and a column in the All Employees list (Story 3.1) — no publish step, no delay, no engineering ticket.

Every custom field must carry a visibility level, chosen at creation, defaulting to *management*: `management` (managers/PP/HR Admin only), `employee` (also visible to Self), or `colleague` (also visible to everyone) (3.3.5). This visibility must be enforced consistently everywhere the field can appear — profile section S16, list column, list filter, export (Story 3.5) — such that a user filtering or sorting by the field can never infer a value they are not entitled to see (3.3.5). This is the same constraint driving Story 3.1's access-aware filtering, now extended to arbitrary user-defined fields.

Architecturally, this confirms the data-model note in Section 6: a column-per-field relational schema cannot support fields that don't exist yet, so custom field definitions and values must be modeled as metadata + typed value storage (e.g., an EAV-style or JSON/typed-value table keyed by field definition), not as ALTER TABLE operations.

Who can create fields: HR Admin always; "manager" here should be read together with 2.3 — in practice this is gated by the `manage custom fields` functional permission, which HR Admin holds by default and can grant to other functional roles.

- A user with the `manage custom fields` permission can create a field of each supported type (text, number, date, single-select, multi-select, boolean) via the UI only.
- A newly created field is immediately available as both a filter and a column on All Employees, with zero deploy/migration/developer step.
- A field's visibility level (management default / employee / colleague) can be set at creation and governs read access on the profile, in list columns, in filters, and in exports.
- Single-select and multi-select fields support defining and later editing their option lists through the UI.
- A user without visibility into a given custom field cannot see it as a column, cannot filter on it, and cannot detect its existence via filter behavior (e.g., result-count changes).
- Setting a value on a profile for a custom field respects the same write-access rules as other profile fields (per the access matrix, RW audiences per section).

**Given/When/Then:**
```
Scenario: HR Admin defines and immediately uses a new field
Given an HR Admin creates a new single-select custom field "Preferred office" with visibility "employee"
When they save the field definition
Then the field is immediately available as a filter and a column on the All Employees list, with no deploy or developer action, and values can be set on individual profiles through the UI

Scenario: Field visibility hides it from unauthorized viewers
Given a custom field "Performance flag" is created with visibility "management"
When a colleague-level user opens All Employees or requests the profile of someone with that field set
Then the field does not appear as a column/filter option and is not present in any API response for that user
```

## Story 3.3: Inline Editing on the List (ClickUp task: 869em1x2c)
**Current description:** "**As** a user with write access to a field, **I want** to edit it inline from the list, **so that** I don't need to open the full profile for small updates.

**FRs:** FR11

**Acceptance Criteria:**
*   Given a column is inline-editable and I have write access to that field for that row, when I edit the cell, then it writes through to the underlying profile field.
*   Given I lack write access to that field for that row, when I view the list, then the cell is not editable."

**Proposed description:**
Allow selected columns on the All Employees list to be edited inline, writing through to the same underlying profile field shown on the full profile page (4.1). Editability is per cell (field × row), not per column globally: a column may be inline-editable in principle, but a given cell is only actually editable if the current viewer holds write (`RW`) access to that field for that specific employee, as resolved from the Section 3 access matrix for the viewer's relationship to that row's person (manager line, PP, Self, HR Admin, etc.) (3.2, 3.3.4). Access is evaluated server-side, per section, per request (3.3.4) — the frontend must not merely grey out a cell; the API must reject a write to a field/row the user is not authorized for.

This applies to both built-in fields and custom fields (Story 3.2); for custom fields, write access follows the same section (S16) rules as other RW audiences in 3.2 (Manager line and PP get RW; Self/Colleague depend on the field's configured visibility level, which governs read, not automatically write — write eligibility should default to management/PP unless product decides otherwise, since S16's Self/Colleague rows are "per field visibility" for read).

Inline edits must respect field type/validation matching the field's definition (text, number, date, single-select, multi-select, boolean per Story 3.2), and must not bypass any business rule that exists on the full profile edit form for that same field (e.g., required option lists for select fields).

- A cell in an inline-editable column, for a row/field the user has write access to, is rendered as editable and saving it updates the same underlying value shown on the employee's profile.
- A cell for a row/field the user lacks write access to is rendered read-only (non-editable) in the UI.
- Even if a client is manipulated to attempt a write to a non-authorized cell, the API rejects it (server-side enforcement, not just UI hiding) (3.3.1, 3.3.4).
- Edits validate against the field's type (e.g., select fields restrict to defined options, number/date fields reject malformed input).
- An inline edit is reflected immediately in the list (optimistic or confirmed update) and on the full profile page.
- Inline editing works uniformly for built-in and custom fields, subject to their respective access rules.

**Given/When/Then:**
```
Scenario: Authorized inline edit writes through to the profile
Given a Unit Manager viewing All Employees has write access to the "grade" field for one of their direct reports
When they edit the grade cell inline and save
Then the change is persisted to the employee's profile and is visible both in the list and on the full profile page

Scenario: Unauthorized user cannot edit or bypass restriction
Given a colleague-level user views All Employees and a column is configured as inline-editable
When they attempt to edit a cell for a person they have no Manager/PP relationship with
Then the cell is rendered non-editable in the UI, and a direct API write attempt to that field/row is rejected server-side
```

## Story 3.4: Saved and Shared Views (ClickUp task: 869em1x3g)
**Current description:** "**As** a manager, **I want** to save a filter/column configuration as a named view and share it with other managers, **so that** useful lists don't need rebuilding.

**FRs:** FR12

**Acceptance Criteria:**
*   Given I save a view, when I reopen the list, then it appears as a tab, and multiple views coexist.
*   Given I share the view with another manager, when they open the list, then they see the view as an available tab, filtered to their own access — not mine."

**Proposed description:**
Let a user save the current All Employees filter + column configuration as a named view, which then appears as a tab alongside other views; multiple saved views coexist per user (4.1). Views are owned by their creator. A view's owner can share it with other managers so it appears as a tab in the recipient's list too (4.1).

Critically, a saved view stores the filter/column *configuration* (which fields, which operators/values, which columns are shown), not a materialized/snapshotted result set or a cached column-visibility override. When any user — owner or a recipient the view was shared with — opens a saved view, the query is re-executed against that user's own resolved access (Section 3): the same view definition can legitimately return different rows and hide/show different columns for different viewers, because access is evaluated server-side per request (3.3.4), never baked into the saved view itself. A view must never act as a way to leak data the current viewer wouldn't otherwise see.

Reference use cases from the spec (not to be built as fixed features, just illustrative): a manually-maintained "bench" list, "needs a conversation", "everyone who joined this year in Poland", "all people open to mentoring" (4.1). Note "manually-maintained" implies at least one view type may involve an explicit list of people in addition to/instead of a pure filter — clarify with product whether Story 3.4 needs to support a static member list variant or whether that's future scope; default to filter-based views for this story unless directed otherwise.

Saved views are also the mechanism referenced by form campaign audience selection (4.12) — a saved view can be used directly as a campaign audience — so the view's filter definition must be resolvable/reusable outside the All Employees page itself.

- A user can save the current filter + column configuration under a name; it appears as a new tab on the list.
- Multiple saved views coexist for the same user without overwriting each other.
- The view owner can share a view with specific other managers (or, at minimum, make it visible to them as a tab).
- When a recipient opens a shared view, results and visible columns reflect the recipient's own access role, not the owner's — the view is a query definition, not a data snapshot.
- Only the owner (or someone with appropriate permission) can edit/delete/rename the view; recipients can use it but shared-view edit rights should be explicitly defined (default: view-only for recipients unless product says otherwise).
- Deleting or unsharing a view removes it from recipients' tab lists without affecting the owner's copy.

**Given/When/Then:**
```
Scenario: Save and reload a view
Given a manager configures filters and columns on All Employees
When they save this configuration as a named view "Needs a conversation"
Then the view appears as a tab, persists across sessions, and coexists with any other saved views they have

Scenario: Shared view respects the recipient's own access, not the owner's
Given Manager A saves a view including a custom field visible only to management and shares it with Manager B
When Manager B opens the shared view
Then Manager B sees only the rows and columns they are personally entitled to see, which may differ from what Manager A sees, even though the underlying filter/column definition is identical
```

## Story 3.5: Export to Excel (ClickUp task: 869em1x4v)
**Current description:** "**As** a user viewing the list, **I want** to export the current view to .xlsx, **so that** I can work with it outside the app.

**FRs:** FR13

**Acceptance Criteria:**
*   Given I export the current filtered/columned view, when the file is generated, then it contains only the columns and rows I'm entitled to see."

**Proposed description:**
Allow the current All Employees view (current filters + current column selection, whether ad hoc or a saved view) to be exported to an `.xlsx` file (4.1). The export must be generated server-side from the same access-resolved query used to render the list, not from whatever happens to be rendered in the browser DOM — this matters because a client-side "export what's on screen" approach is exactly the kind of leak surface Rule 3.3.1 calls out ("not the UI, not the API, not an export...").

The export is entitled-columns-only: it must contain only the columns the exporting user is currently authorized to see (built-in fields per 3.2, custom fields per their configured visibility level per 3.3.5), and only the rows the export resolves to given the current filters and the exporter's row-level access. A restricted field must never appear in the file even as an extra/hidden column, and must not be reconstructable from other exported columns. This applies identically to colleague-mode exports (Story 3.6): a colleague exporting the list gets only whitelist columns.

Because export reuses the list's query/access layer, this story should not re-implement filtering/access logic — it should call the same server-side resolution as Story 3.1's list rendering and Story 3.3's write checks conceptually mirror for reads, then stream results into an `.xlsx` writer.

- Exporting the current view produces a valid `.xlsx` file containing exactly the columns currently selected in the view AND that the exporter is entitled to see (intersection, not just "whatever is selected").
- Exported rows match exactly what the current filters return for this exporter's access-resolved query (no more, no fewer).
- A field the exporter cannot see is absent from the file entirely — not blanked, not present as an empty column, not present in a hidden sheet/metadata.
- Column headers in the export match the list's column labels (built-in and custom field names).
- Export works correctly for large result sets (up to 500+ employees per Section 7 scale) without timing out or truncating silently.
- Colleague-mode export (Story 3.6) only ever contains whitelist columns (S1, S10 type, S11 project name).

**Given/When/Then:**
```
Scenario: Export respects column-level entitlement
Given a manager's current view includes a custom field visible only to "management" and the manager holds that access
When they export the view to .xlsx
Then the file contains that column with correct values, alongside all other selected columns and matching filtered rows

Scenario: Export never includes a restricted field even if selected
Given a user's current column selection somehow includes a field they are not entitled to see for one or more rows (e.g., a management-only field for a colleague relationship)
When they export the view
Then the resulting .xlsx omits that column/value entirely for those rows, with no trace of the restricted data anywhere in the file
```

## Story 3.6: Colleague Mode of the List (ClickUp task: 869em1x69)
**Current description:** "**As** a colleague-level user, **I want** the same list page scoped to the whitelist, **so that** I can still browse people without seeing restricted data.

**FRs:** FR14

**Acceptance Criteria:**
*   Given I am a colleague to everyone in the list, when I open All Employees, then only whitelist columns (S1, S10 type, S11 project name) are shown.
*   Given I click a row, when the profile opens, then it renders the limited colleague profile view."

**Proposed description:**
Serve the same All Employees page to a user in the Colleague audience (someone holding none of Manager line, PP, or HR Admin with respect to a given employee), but scoped to a strict whitelist of visible data (3.2 §3.3.3, 4.1). The colleague view is not a design choice among many options — the spec is explicit that "colleague view is a whitelist, not a blacklist" (3.3.3): a colleague sees exactly S1 (Identity card), S10 including leave type (Leaves and absences), and S11 project name only (Projects) — everything else is absent, no exceptions, no partial leakage of other sections' data through columns, filters, or derived fields.

This must be enforced server-side, not by hiding columns/fields in the frontend (3.3.3, explicit instruction: "Do not implement this by hiding fields in the frontend — the API must not return them"). The same server-side, per-request, per-section access resolution used elsewhere (3.3.4) applies here: the API response for a colleague-mode request must simply not contain non-whitelisted fields, regardless of what the client requests or renders.

Because access is relationship-specific and evaluated per row (2.1, 3.2), a single logged-in user may be in Colleague mode for some rows of the list and in Manager/PP mode for others within the same list render — the list must apply the correct per-row access, not a single global "am I a manager of anyone" toggle. (Colleague mode as a distinct "mode" mainly matters for a user who has zero Manager/PP relationships to anyone, but the underlying enforcement is per-row/per-person regardless.)

Clicking a row for someone the viewer only holds Colleague access to opens the profile filtered to the same whitelist (S1, S10 type, S11 project name), rendering only those sections, consistent with 4.2 ("a section the viewer has no access to is not rendered and not returned by the API").

- All Employees, for a colleague relationship, exposes only S1 fields, S10 (including leave type), and S11 project name (no PM/DM, no period) as columns and filters — every other built-in and custom field is absent from the API response, not merely hidden in the UI.
- The whitelist is enforced server-side: inspecting network responses/API calls for a colleague-mode request shows no restricted fields present anywhere in the payload.
- Clicking a row for a colleague-only relationship opens the limited colleague profile view, rendering only the whitelisted sections (per 4.2), not the full profile with sections hidden by CSS/JS.
- Custom fields with visibility `colleague` are the only custom fields shown to colleagues; `management`/`employee`-only custom fields are absent for this audience (3.3.5).
- A user who is a Manager/PP for some employees and a Colleague for others sees the correct per-row scoping within a single list view/session (2.1).
- Sort/filter options offered to a colleague-mode user are limited to whitelist fields only — they cannot filter or sort by a non-whitelisted field even if they somehow know it exists.

**Given/When/Then:**
```
Scenario: Colleague sees only whitelist columns, enforced server-side
Given a user holds no Manager, PP, or HR Admin relationship with respect to anyone currently in the list
When they open All Employees
Then only S1 identity fields, S10 leave type, and S11 project name are available as columns/filters, and the underlying API response contains no other fields for those rows, even on direct inspection

Scenario: Mixed access within one list render
Given a user is the manager of Employee X but a plain colleague to Employee Y
When they open All Employees showing both rows
Then Employee X's row shows manager-line fields per 3.2 while Employee Y's row is restricted to whitelist-only fields, within the same list render
```

---

# Epic 4: Action Items & Tasks

## Story 4.1: Manually Create an Action Item (ClickUp task: 869em1x9t)
**Current description:** "As a manager or PP, I want to create an action item for someone in my access scope, so that I can assign them a task with a due date.

FRs: FR27 (part 1), FR28

Acceptance Criteria:
Given I am the UM/DM/PP (or hold a permitted functional role) for employee B, when I create an item with title, description, due date and optional link, then it appears on B's profile (S14).
Given I attempt to create an item for someone outside my access scope, when I submit, then it is rejected."

**Proposed description:**
Implements manual creation of action items per spec 4.5 point 1 and 2.3. Action items are the single task entity in the system (4.5) and are shown on the employee's profile under S14 (3.2), on self-service (4.3), and on manager/PP dashboards (4.4).

Who can create: a Unit Manager, Delivery Manager, Project Manager, or People Partner can create an action item for any employee they hold Manager or People Partner access over, per the hierarchy resolution in 2.1 (transitive closure of "reports to" and "assigned to a project managed by"). Additionally, any functional role granted the *create action items* permission (2.3) can create action items, but only within its own access scope — a functional role never widens data access, so a role without a Manager/PP relationship to a person sees them only via the colleague view and cannot target them. Creation can be initiated either from the employee's profile (S14 section) or from the creator's own dashboard.

Fields captured on creation: title, short description, assignee (the target employee), author (creator, set automatically), due date, optional link, status (defaults to `open`), completion date (null at creation), and source (set to `manual` for this flow, distinct from `campaign` — see 4.5 point 2 and the NEW story below).

Acceptance criteria:
- A UM/DM/PP (or holder of the *create action items* functional permission) can create an item with title, short description, due date, and optional link for any person within their current access scope; on save it appears on that person's profile S14 section with status `open`, author, and source `manual`.
- The access scope check is evaluated server-side at creation time using the same transitive Manager/PP resolution as 2.1 — not cached or inferred client-side.
- An attempt to create an item for a person outside the creator's access scope (no Manager, PP, or permitted functional-role relationship) is rejected by the API, not merely hidden in the UI.
- Title and due date are required; short description and link are optional.
- The created item is immediately visible on the assignee's own action-item list (S14, self-service, 4.3) and on the author's dashboard "own action items" widget (4.4.1).
- The action item record stores author and assignee as distinct identities even when the same functional-role holder later loses access to the assignee (so historical author attribution is not lost when access changes, per 2.1's "not sticky" rule for access itself, though the record persists).

**Given/When/Then:**
```
Scenario: Manager creates an action item for a direct report
Given I am UM for employee B (B reports to me)
And I am on B's profile page
When I create an action item with title "Submit Q3 self-review", due date 2026-09-01, and no link
Then a new action item is created with status "open", author me, assignee B, source "manual"
And the item appears in B's S14 section and on my dashboard's own-action-items widget

Scenario: Creation rejected for a person outside access scope
Given I am a PM with no Manager or PP relationship to employee C, and I hold no functional role with the "create action items" permission over C
When I attempt to create an action item for C via the API
Then the request is rejected with an authorization error
And no action item is created or visible on C's profile
```

## Story 4.2: Complete and Cancel Action Items (ClickUp task: 869em1xbf)
**Current description:** "As the assignee, I want to mark my action item complete; as the author, I want to cancel one with a reason, so that the lifecycle reflects reality.

FRs: FR29

Acceptance Criteria:
Given I am the assignee, when I mark an item complete, then its status becomes completed and the completion date is recorded.
Given I am the author, when I cancel an item and provide a reason, then its status reflects cancellation with the reason stored."

**Proposed description:**
Implements the action item lifecycle described in 4.5: "Lifecycle: `open` → `completed`. The assignee marks their own items complete, and the completion date is recorded and displayed. The author can cancel an item with a reason." This applies uniformly regardless of source (`manual` or `campaign` — see 4.12 point 4, where completing a campaign-sourced item is the completion signal for that recipient's form task).

Who can do what:
- **Complete**: only the assignee of the item can mark it complete, from their own profile view (S14, RW for self per 3.2) or self-service task list (4.3). No one else — not even the author, manager, or PP — can complete an item on the assignee's behalf, since S14 grants the employee "R (own) + mark complete" while Manager line/PP get RW but completion is explicitly the assignee's action.
- **Cancel**: only the author of the item can cancel it, and must supply a reason at cancellation time. Cancellation is available to the author regardless of whether they still hold current Manager/PP access over the assignee (the authoring relationship is a historical fact, not a live permission check) — though this should be confirmed against product intent if ambiguous.

Fields involved: status (`open` → `completed` or `open` → `cancelled`), completion date (set automatically when status becomes `completed`; remains null for `cancelled`), cancellation reason (free text, required when cancelling, stored on the item).

Acceptance criteria:
- Given I am the assignee of an open item, when I mark it complete, then status becomes `completed`, completion date is set to now, and this is displayed alongside the item wherever it renders.
- Given I am not the assignee, when I attempt to mark an item complete (via API), then the request is rejected.
- Given I am the author of an open item, when I cancel it and provide a reason, then status becomes `cancelled`, the reason is stored and displayed, and the item no longer counts toward the assignee's "open" or "overdue" totals.
- Given I am not the author, when I attempt to cancel an item, then the request is rejected.
- An attempt to cancel without providing a reason is rejected — the reason field is mandatory for cancellation.
- A `completed` or `cancelled` item is terminal: it cannot be reopened, re-completed, or re-cancelled through this flow.

**Given/When/Then:**
```
Scenario: Assignee completes their own action item
Given I am the assignee of an open action item "Submit Q3 self-review"
When I mark the item complete
Then the item's status becomes "completed"
And the completion date is recorded and displayed on the item

Scenario: Author cancels an item but must supply a reason
Given I am the author of an open action item assigned to employee B
When I attempt to cancel the item without entering a reason
Then the cancellation is rejected and the item remains "open"
When I cancel the item again and provide the reason "No longer applicable"
Then the item's status becomes "cancelled"
And the reason "No longer applicable" is stored and shown on the item
```

## Story 4.3: Overdue Highlighting (ClickUp task: 869em1xd3)
**Current description:** "As any viewer of an action item, I want overdue items flagged wherever they appear, so that nothing slips through unnoticed.

FRs: FR29 (cont.)

Acceptance Criteria:
Given an item's due date has passed and it's still open, when it renders on a profile, dashboard, or list, then it is visibly marked overdue."

**Proposed description:**
Implements the overdue rule from 4.5: "An item past its due date is shown as overdue wherever it appears." "Overdue" is a derived, not stored, state: an item is overdue when its status is `open` (never `completed` or `cancelled`) and its due date is strictly before the current date/time. This derivation must be computed consistently everywhere an action item renders, not duplicated with divergent logic per surface.

Surfaces that must show the overdue flag, all sourced from the same derivation:
- Employee profile S14 section (own items, and manager-line/PP view of someone else's items).
- Self-service action item list (4.3).
- UM dashboard: "the manager's own action items sorted by due date, with overdue highlighted" and the summary counter "overdue action items" (4.4.1).
- DM/PM dashboard equivalents, and PP dashboard.
- Any campaign per-person completion table (4.12 point 5), where an incomplete recipient past the campaign due date shows overdue.
- Any future list/export of action items.

Because overdue is derived at read time (or via a scheduled recompute for counters), it must always reflect "now" — a manual "overdue" field that must be separately updated is out of scope and would violate the "wherever it appears" requirement.

Acceptance criteria:
- Given an open item with due date in the past, when it renders on the assignee's profile (S14), the assignee's self-service list, or any manager/PP/dashboard view of it, then it is visibly marked overdue (e.g. distinct styling/badge) in every one of those places.
- Given an item is `completed` or `cancelled`, when its due date has passed, then it is never shown as overdue, regardless of where it renders.
- Given an open item whose due date is today or in the future, then it is never shown as overdue.
- The UM/DM/PM/PP dashboard "overdue action items" counters count exactly the set of items meeting the overdue definition, scoped to the items visible to that viewer's access scope.
- Overdue status updates automatically as soon as the due date passes, without requiring any write to the item (no manual flag toggling, no batch job dependency for on-demand views).
- A campaign-sourced action item (source `campaign`) uses the same overdue logic and the same visual treatment as a manually created item, including in the campaign's per-person completion table (4.12 point 5).

**Given/When/Then:**
```
Scenario: An open item past its due date is highlighted on the profile and dashboard
Given an open action item assigned to employee B with due date 2026-08-01 (in the past relative to today, 2026-08-20)
When B views their own profile S14 section
And B's manager views their dashboard's own-action-items widget for an item they authored for B
Then the item is visibly marked overdue in both places

Scenario: Completing an item removes the overdue mark everywhere it appears
Given an open, overdue action item assigned to employee B, currently shown as overdue on B's profile, B's self-service list, and the campaign completion table (if source is "campaign")
When B marks the item complete
Then the item's status becomes "completed" and its completion date is recorded
And the item is no longer displayed as overdue on any surface, including the profile, self-service list, dashboard counters, and campaign table
```

## Story NEW: Auto-Generate Action Item on Form Campaign Activation (ClickUp task: NEW)
**Proposed description:**
Implements the automatic action-item creation path required by 4.5 point 2 ("Generated automatically by a form campaign (4.12) — one action item per recipient when the campaign is activated, carrying the campaign's link and due date") and 4.12 point 3 ("Each recipient gets an action item (4.5) on their profile with the title, the sender, the due date and the link. Following the link opens the external form.").

Trigger: activation of a form campaign. Per 4.12 point 2, the campaign's audience is resolved via the All Employees filter engine (with optional individual add/remove after resolution) and is **frozen at the moment of activation** — people who join the audience's underlying filter criteria later are not added retroactively. This story covers the action-item side of that activation event only; it does not cover campaign/audience creation or the campaign entity itself.

Behavior on activation: for each recipient in the campaign's frozen audience, the system creates exactly one action item with:
- title: the campaign's title
- short description: derived from the campaign (e.g. campaign's short description/purpose, per whatever the campaign entity exposes)
- assignee: the recipient
- author / sender: the campaign creator (PP, manager, or holder of the *create form campaigns* functional permission per 2.3)
- due date: the campaign's due date
- optional link: the campaign's link to the external form
- status: `open`
- completion date: null
- source: `campaign` (explicitly distinct from `manual`, per 4.5's "source (manual or campaign)" field)

This item then behaves identically to a manually created one for all downstream stories in this epic: it appears on the recipient's profile S14 section and self-service task list (4.3), the recipient marks it complete when they've filled in the external form — which is the sole completion signal, since the system does not read or verify the external form (4.12 point 4) — and it is subject to the same overdue highlighting rules (Story 4.3) wherever it renders, including the campaign's own per-person completion table (4.12 point 5: "who has completed, who has not, who is overdue").

**Dependency note:** This story depends on the Forms & Campaigns epic, which is not yet scoped as a ClickUp epic/list at the time of writing. This story only covers the action-item creation path invoked at activation time; it does not include building the campaign entity, audience resolution/freezing, or the campaign creator UI. Whoever implements campaign activation (in the Forms & Campaigns epic, once scoped) must call this action-item-creation path — passing campaign title, sender/author, due date, and link — as part of the activation transaction, ideally atomically with freezing the audience, so that every frozen recipient reliably gets exactly one action item and no recipient is silently skipped.

Acceptance criteria:
- Given a campaign with a frozen audience of N recipients, when the campaign is activated, then exactly N action items are created, one per recipient, each with source `campaign`, status `open`, and completion date null.
- Given a campaign activation, each created action item carries the campaign's title, the due date, and the link to the external form; following the link from the action item opens the external form.
- Given a campaign activation, the author/sender field on each created action item identifies the campaign's creator (sender), so the recipient can see who sent it.
- A recipient marking their campaign-sourced action item complete does not verify or read the external form; completion is purely the recipient's self-reported signal, exactly as for any manually completed item (Story 4.2).
- The campaign's per-person table (4.12 point 5) reflects the real-time status (completed / not completed / overdue) of each recipient's generated action item, using the same overdue derivation as Story 4.3.
- If campaign activation fails partway (e.g. after some but not all action items are created), the system does not leave a partially-created, inconsistent set of action items silently — activation should be atomic or clearly recoverable/idempotent per recipient.

**Given/When/Then:**
```
Scenario: Activating a form campaign generates one action item per frozen recipient
Given a PP has created a form campaign "Annual Engagement Survey" with a due date and an external form link
And the PP has resolved and frozen an audience of 50 employees
When the PP activates the campaign
Then 50 action items are created, one per recipient
And each item has title "Annual Engagement Survey", source "campaign", status "open", the campaign's due date, the campaign's link, and the PP as author/sender

Scenario: A campaign-sourced item shows as overdue in the campaign's own tracking table
Given a campaign-sourced action item assigned to employee B with a due date that has passed and status still "open"
When the campaign's sender views the campaign's per-person completion table
Then B is shown as "overdue" in that table
And when B later marks the action item complete, B is shown as "completed" instead, with the completion date recorded
```

---

# Epic 5: Risks & Risk Dashboard

## Story 5.1: Record a Risk (ClickUp task: 869em1xfr)
**Current description:** "As a manager or PP, I want to record a risk level with description, details and date for a person I'm responsible for, so that risk history is tracked over time.

FRs: FR30

Acceptance Criteria:
Given I create a risk record, when I save it, then it's appended to that person's risk history and becomes the current level, with full history retained."

**Proposed description:**
Implements S6 (Risks) of the access matrix, §3.2, and the "Per employee" recording rules of §4.6. A person holding Manager or People Partner access (§2.1) over an employee — or a functional role granted the *create and edit risks* permission (§2.3), scoped to their own access — can add a new risk record to that employee's risk history. Each record has: `level` (enum: `low`, `need attention`, `medium`, `high`, `leaver`), `description` (free text summary of the situation), `details` (free text elaboration), and `date`. Risk history is append-only and retained in full (§4.6); a new record never overwrites an old one. The employee's "current level" is always the most recent record by date. Per S6, the section is `RW` for Manager line and PP, and **`—` (no access at all) for Self** — a risk record must never be readable by the employee it is about, through the UI, the API, an export, a notification, or a search result (§3.3 rule 1, §4.3 "An employee cannot see their risk level"). Write access is server-side gated per §3.3 rule 4: re-check the requester's resolved access role for this specific employee on every write, not just on page load.

Acceptance criteria:
- A user with Manager or PP access (or an equivalent granted functional-role permission) over employee E can create a risk record with level, description, details, and date; on save it is appended to E's risk history and becomes the current level.
- Level accepts only the five defined values (`low`, `need attention`, `medium`, `high`, `leaver`); no other values are permitted.
- Risk history is never destructively edited or deleted by this story — new records only append; prior records remain visible in the full history to authorized viewers.
- A user attempting to create/view a risk record for a person they hold no Manager or PP relationship over (e.g. a colleague, or a functional-role holder outside their access scope) is rejected server-side, not just hidden client-side.
- The employee themselves cannot create, read, or in any way discover a risk record about themselves, via any surface (UI, API, export, notification, search).
- Saving a risk record does not require a project or profile-section dependency beyond the employee existing in the system.

**Given/When/Then:**
```
Scenario: Manager records a new risk for a direct report
Given I hold Manager access over employee E (per §2.1 hierarchy resolution)
And I have permission to create and edit risks
When I submit a new risk record for E with level "high", a description, details, and a date
Then the record is appended to E's risk history
And E's current level becomes "high"
And the full risk history for E remains retained and unmodified for prior records

Scenario: Employee cannot see or create a risk record about themselves
Given I am employee E, viewing my own self-service profile
When I request my own profile data, including via direct API call to the risk section
Then the risk section (S6) is absent from the response entirely, not merely hidden in the UI
And I am not offered any control to create or view a risk record for myself
```

## Story 5.2: Show Risk Trend (ClickUp task: 869em1xhm)
**Current description:** "As a viewer of a risk record, I want to see a trend arrow versus the previous record, so that I know whether things are improving or worsening.

FRs: FR31

Acceptance Criteria:
Given the new record's severity is higher or lower than the previous one, when it renders, then an up/down arrow shows accordingly.
Given it's the first record or the level is unchanged, when it renders, then no arrow is shown."

**Proposed description:**
Implements the "Trend" rule of §4.6: "alongside the current level, show an arrow indicating whether the risk has gone up or down compared with the previous record. No arrow when the level is unchanged or when this is the first record." This is a display/derivation feature layered on top of Story 5.1's risk history — no new fields are stored; the trend arrow is computed at read time (or cached and recomputed on each new record) by comparing the current record's level to the immediately preceding record's level for the same employee, ordered by date. The five levels have a fixed severity ordering for comparison purposes: `low` < `need attention` < `medium` < `high` < `leaver` (increasing severity/concern, consistent with §4.6's "severity descending" sort used on the dashboard, Story 5.3). Comparison rules:
- If the current record's level is more severe than the previous record's level → show an "up" (worsening) arrow.
- If the current record's level is less severe than the previous record's level → show a "down" (improving) arrow.
- If the level is unchanged, or there is no previous record (this is the first record ever for that person) → show no arrow.
The trend arrow appears everywhere the current risk level is surfaced to an authorized viewer: the employee's profile S6 section and the Risk Dashboard table (Story 5.3). Per S6/§4.6, this is subject to the same access scoping as Story 5.1 — the arrow, like the level itself, is never visible to the employee about themselves.

Acceptance criteria:
- Given a second-or-later risk record whose level is more severe than the immediately preceding record, an "up" arrow renders next to the current level.
- Given a second-or-later risk record whose level is less severe than the immediately preceding record, a "down" arrow renders.
- Given a record whose level is identical to the immediately preceding record, no arrow renders.
- Given the first-ever risk record for a person, no arrow renders (there is no "previous" to compare against).
- The severity ordering used for comparison is fixed and consistent across this feature and the Risk Dashboard sort (Story 5.3): `low` < `need attention` < `medium` < `high` < `leaver`.
- The trend arrow is subject to the same S6 access scoping as the underlying risk level — never rendered or returned to the employee themselves.

**Given/When/Then:**
```
Scenario: Worsening risk shows an up arrow
Given employee E has a previous risk record at level "low"
And I hold Manager or PP access over E
When a new risk record is saved for E at level "medium"
Then the current level displayed for E is "medium"
And an "up" (worsening) trend arrow is shown alongside it

Scenario: First-ever risk record shows no trend arrow
Given employee E has no prior risk records
And I hold Manager or PP access over E
When a new risk record is saved for E at level "need attention"
Then the current level displayed for E is "need attention"
And no trend arrow is shown, because there is no previous record to compare against
```

## Story 5.3: Risk Dashboard (ClickUp task: 869em1xm5)
**Current description:** "As a manager or PP, I want a Risk Dashboard scoped to my access, with counts by level and a sortable, filterable table with drill-through, so that I can act on risk across my people.

FRs: FR32

Acceptance Criteria:
Given I open the Risk Dashboard, when it loads, then I see counts by level (medium/high/leaver emphasized) and a table sorted by severity then date, scoped only to people I hold Manager or PP access over.
Given I click a count, when the table filters, then it shows only matching records, and clicking a row opens the profile.
Given I am an employee, when I attempt to access the Risk Dashboard, then it is not visible to me at all."

**Proposed description:**
Implements the "Risk Dashboard" separate page described in §4.6. This is distinct from Stories 5.1/5.2 (which cover recording a risk and rendering its trend arrow); this story is the aggregate view. Requirements per §4.6:
- **Counts by level:** one count per level (`low`, `need attention`, `medium`, `high`, `leaver`), with `medium`, `high` and `leaver` visually emphasised (e.g. color/weight) since these are the levels requiring action.
- **Table:** all people with a risk record, sorted by severity descending (using the same fixed ordering as Story 5.2: `leaver`/`high` most severe down to `low`), then by date within equal severity. Each row shows the person, current risk level, and the trend arrow from Story 5.2.
- **Filters:** by unit, department, project, PP, manager (per §4.6 explicitly).
- **Drill-through:** clicking a count filters the table to just that level (and clears/combines with other active filters); clicking a table row opens that person's profile.
- **Scoping:** the dashboard shows only people the viewer holds Manager or People Partner access over per §2.1 (the same relationship-derived scoping used everywhere else — a UM sees their nested subordinates, a DM/PM sees people on their projects, a PP sees their assigned people and the HR line above). This must be enforced server-side, not just client-side filtered (§3.3 rule 4).
- **Never visible to the employee:** per §4.6's closing line and S6's `—` for Self, an employee must never be able to open, link to, or otherwise reach the Risk Dashboard or any data from it about themselves — not the page, not an API response, not a notification. If a functional role without Manager/PP access attempts to reach it, they see no data (empty/zero-scoped), consistent with §2.3 ("a newly created role... sees the audience through the colleague view unless it also holds a Manager or People Partner relationship").
- Access to the dashboard feature itself should also respect the *view dashboard* / risk-related functional permission where applicable (§2.3), on top of the underlying per-person Manager/PP data scoping — a user must satisfy both the feature permission and the data-access scoping.

Acceptance criteria:
- Given a Manager or PP opens the Risk Dashboard, the counts by level and the table are scoped exclusively to people they hold Manager or PP access over per §2.1 — no other employees' risk data appears anywhere on the page.
- The `medium`, `high`, and `leaver` counts are visually emphasised relative to `low` and `need attention`.
- The table is sorted by severity descending, then by date, and displays the trend arrow (per Story 5.2) for each row.
- The dashboard supports filtering by unit, department, project, PP, and manager.
- Clicking a count drills through to the table filtered to that level; clicking a table row navigates to that person's profile.
- An employee (Self, with no Manager/PP relationship over anyone) cannot access the Risk Dashboard through any route — no menu entry, no direct URL, no API response containing risk data about themselves or others.

**Given/When/Then:**
```
Scenario: PP drills through from a severity count to a filtered table and into a profile
Given I am a People Partner assigned to a set of employees, some of whom have "high" risk records
When I open the Risk Dashboard
Then I see counts by level scoped only to my assigned people, with "high" visually emphasised
When I click the "high" count
Then the table filters to show only my assigned people currently at "high" risk, sorted by date
When I click a row in the filtered table
Then I am taken to that person's profile

Scenario: Employee cannot access the Risk Dashboard at all
Given I am logged in as an ordinary employee with no Manager or PP relationship over anyone
When I attempt to navigate directly to the Risk Dashboard URL or call its underlying API
Then I am denied access or the dashboard renders no risk data
And no risk counts, table rows, or trend arrows for myself or any other employee are returned in the response
```

---

# Epic 6: Resourcing

## Story 6.1: Create a Resourcing Request (ClickUp task: 869em1xna)
**Current description:** "As a DM or PM, I want to create a resourcing request with vacancy details, comp level, duration and workload, optionally linked to a project, so that I can start filling a role even before the project exists in the system.

FRs: FR33

Acceptance Criteria:
Given I create a request without a project reference, when I save it, then it's stored as a valid unattached request.
Given I am a DM, when I view my requests, then I also see requests created by the PMs of my projects."

**Proposed description:**
Implements request creation for Resourcing, per spec Section 4.7. A resourcing request captures a vacancy: details/requirements, expected compensation level, duration, and workload. Creation is available to DM and PM (Section 2.2), and to any functional role granted the *create resourcing requests* permission (Section 2.3) — functional-role grants unlock the feature but never widen data access on their own.

A request may optionally reference a project, but does not have to: requests are frequently created before the project exists in the system, and an **unattached request is a normal, valid state**, not an error or a "draft" — it must be listable, fulfillable, and reviewable exactly like an attached request (Section 4.7). The project link, when present, is a reference to a project drawn from the timetracker integration (Section 5.1), not a locally-owned project entity.

Visibility follows the DM dashboard model (Section 4.4.2): a DM sees both the requests they created themselves and the requests created by the PMs of their projects — this is not a separate grant but flows from the DM sitting above the PM in the manager-access chain (Section 2.1). A PM sees only the requests they created for their own projects.

Acceptance criteria:
- A DM or PM (or a role holding the *create resourcing requests* permission) can create a request with: vacancy details/requirements, expected compensation level, duration, workload, and an optional project reference.
- Saving a request without a project reference succeeds and the request is stored as a normal, fully-functional unattached request (no degraded state, no blocking validation on the project field).
- A newly created request is routed for fulfilment (Story 6.2) and starts in a state consistent with the resourcing workflow (e.g. "open"/"pending fulfilment").
- A DM's request list includes their own requests plus every request created by the PMs of projects the DM is responsible for (per the Section 2.1 hierarchy).
- A PM's request list is scoped to requests they created for their own projects only.
- Someone without the *create resourcing requests* permission (and without UM/DM/PM functional role) cannot create a request via UI or API.

**Given/When/Then:**
```
Scenario: Creating an unattached resourcing request
Given I am logged in as a DM
And I have not selected a project for this request
When I fill in vacancy details, expected compensation level, duration and workload and save
Then the request is created and stored as a valid unattached request
And it appears in my Resourcing → Requests list without a project reference

Scenario: DM sees requests created by their PMs
Given I am a DM responsible for Project X
And a PM of Project X has created a resourcing request for that project
When I open my Resourcing → Requests view
Then I see both my own requests and the request created by the PM
And a PM of a different project I am not responsible for does not see that request
```

## Story 6.2: Fulfil a Request with Internal or External Candidates (ClickUp task: 869em1xpt)
**Current description:** "As a UM, I want to see requests assigned to me and propose internal specialists or an external candidate, so that I can submit candidates for approval.

FRs: FR34

Acceptance Criteria:
Given a request is assigned to me, when I attach an internal employee, then they're added as a proposed candidate.
Given the PeopleForce integration isn't live yet, when I attach an external candidate, then I can still record them via an external PeopleForce link as a fallback.
Given I've attached one or more candidates, when I submit, then the request moves to DM review."

**Proposed description:**
Implements request fulfilment per Section 4.7. Fulfilment is available to UM (Section 2.2) and to any functional role granted the *fulfil resourcing requests* permission (Section 2.3). The UM sees incoming requests assigned to them — both attached and unattached requests are valid work items (Section 4.7) — and proposes one or more candidates against each:

- **Internal candidates**: select specialists from their own unit. The UM's ability to select is scoped to people they hold Manager access over (their unit), per Section 2.1.
- **External candidates**: attach a candidate pulled from PeopleForce (Section 5.2). Where the PeopleForce API integration is not implemented in time, the fallback is an external link to the candidate's record in PeopleForce (Section 4.7, Section 5.2) — this fallback is an acceptable permanent behaviour for this iteration, not a placeholder to be blocked on.

A request can carry a mix of internal and external candidates. Once one or more candidates are attached, the UM submits the request, moving it into DM review (feeds Story 6.3). Note: this story does not implement the DM's decision — only getting candidates from "attached" to "submitted for review."

Acceptance criteria:
- A UM (or a role with the *fulfil resourcing requests* permission) sees the list of requests assigned to them, including unattached ones, and can open any of them to propose candidates.
- Attaching an internal employee from the UM's unit adds them as a proposed candidate on the request, linked to their employee profile.
- Attaching an external candidate records them either via a live PeopleForce candidate reference or, when that integration is unavailable, via a plain external link to PeopleForce — both paths are supported and selectable.
- Multiple candidates (internal, external, or a mix) can be attached to the same request before submission.
- Submitting the request with one or more candidates transitions it to "pending DM review" and it becomes visible to the DM (and to the DM's PMs where applicable, per Section 4.7/4.4.2).
- A UM without the fulfilment permission, or a request not assigned to the acting UM, cannot be modified.

**Given/When/Then:**
```
Scenario: UM fulfils a request with an internal candidate and submits for review
Given a resourcing request is assigned to me as UM
When I select a specialist from my unit and attach them as a candidate
And I submit the request
Then the candidate is recorded as proposed on the request
And the request moves to DM review status

Scenario: External candidate fallback when PeopleForce integration is unavailable
Given the PeopleForce API integration is not live
And a resourcing request is assigned to me as UM
When I attach an external candidate using an external PeopleForce link instead of a live candidate record
Then the candidate is recorded on the request with that external link
And I can still submit the request for DM review with this candidate included
```

## Story 6.3: DM Reviews and Approves/Rejects Candidates (ClickUp task: 869em1xrk)
**Current description:** "As a DM, I want to review proposed candidates and approve or reject each with a written reason, so that resourcing decisions are recorded.

FRs: FR35

Acceptance Criteria:
Given a proposed internal candidate whose profile I lack access to, when I open their link, then a profile-sharing flow (Epic 1, Stories 1.11/1.12) is offered instead of a direct profile.
Given I reject a candidate, when I submit, then a written reason is required and stored."

**Proposed description:**
Implements DM candidate review per Section 4.7. The DM sees candidates proposed against their requests (their own and those created by PMs of their projects, per Section 4.4.2/2.1) and decides on each candidate individually — approval/rejection is per-candidate, not per-request, since a request may carry multiple proposed candidates.

Candidate link behaviour differs by candidate type:
- **Internal candidate**: the link leads to the candidate's employee profile. The DM may not already hold Manager or People Partner access to that person (Section 2.1) — most commonly because the candidate isn't yet on one of the DM's projects. In that case profile sharing applies (Section 4.8): the DM is offered a profile-sharing flow rather than a direct profile view. Note this story only needs to trigger/consume that flow; the flow itself (link generation, section selection, expiry, revocation, access logging) is owned by the profile-sharing epic/stories referenced in the current description (Epic 1, 1.11/1.12) and by spec Section 4.8 (sensitive sections S2/S5/S6/S8 excluded by default; S3/S7/S13 never shareable; link expires, default 24h, configurable; access is logged; link is never write-access).
- **External candidate**: the link leads to the candidate's PeopleForce data (if that integration is live) or to an external link to the candidate in PeopleForce (fallback per Section 5.2), consistent with how the candidate was attached in Story 6.2.

For each candidate the DM either **approves** the assignment or **rejects** it. Rejection requires a written reason, which is stored with the decision (this reason is what later surfaces in request history, Story 6.4, and in profile section S15). Approval does not perform any project assignment itself — see Story 6.4 for what happens after approval.

Acceptance criteria:
- The DM can view all candidates proposed on requests they can see (own + their PMs' requests), grouped/filterable by request.
- Opening an internal candidate's link when the DM lacks Manager/PP access to that profile offers the profile-sharing flow (Section 4.8) instead of the raw profile; if the DM already has access (e.g. candidate already on one of the DM's projects), the direct profile opens normally.
- Opening an external candidate's link opens the PeopleForce record or, as fallback, the external PeopleForce link, matching how the candidate was attached.
- The DM can approve a candidate with no further input required beyond confirmation.
- The DM can reject a candidate only by supplying a non-empty written reason; the reason is persisted with the rejection record.
- Each candidate's decision (approve/reject + optional reason) is recorded independently, and the request/candidate state reflects each decision distinctly (a request with several candidates may end with some approved and some rejected).

**Given/When/Then:**
```
Scenario: DM approves an internal candidate they don't yet have profile access to
Given I am a DM reviewing a request with a proposed internal candidate
And I do not currently hold Manager or People Partner access to that candidate's profile
When I open the candidate's profile link
Then I am offered a profile-sharing flow instead of the candidate's direct profile
And I can review the shared profile content and then approve the candidate

Scenario: Rejecting a candidate requires a written reason
Given I am a DM reviewing a proposed candidate on one of my requests
When I choose to reject the candidate without entering a reason
Then the rejection is blocked and I am prompted to provide a written reason
And when I provide a reason and resubmit, the rejection is recorded with that reason stored against the candidate
```

## Story 6.4: Request History (ClickUp task: 869em1xuk)
**Current description:** "As anyone involved in a resourcing request, I want every proposal attempt recorded, so that the full history is visible in Resourcing and on the candidate's profile.

FRs: FR36

Acceptance Criteria:
Given a candidate is approved or rejected, when the decision is saved, then it appears in Resourcing → Requests and, for internal candidates, in profile section S15.
Approval does not itself create a project record — the profile reflects the assignment only after the next timetracker sync (Epic 13)."

**Proposed description:**
Implements the audit/history trail for resourcing per Section 4.7 and profile section S15 (Section 3.2, row S15). Every proposal attempt — proposed → approved/rejected, together with any written feedback/rejection reason from Story 6.3 — is recorded permanently and surfaces in two places:
1. **Resourcing → Requests**, showing the full history of candidates and decisions for a request (visible to whoever can see the request, per the DM/PM visibility rules in Section 4.7/4.4.2).
2. **The candidate's employee profile, section S15**, for internal candidates only (external/PeopleForce candidates have no employee profile in this system). Per the Section 3.2 access matrix, S15 is: `—` for Self (the employee cannot see their own request history), `R` for Manager line, `R` for PP, and not shown to Colleague; it can also be shared via a shared link (`cfg`), and a DM sees their own requests natively without needing a share. This means an employee is never shown that they were proposed/rejected for a role, but their manager line and PP can see it.

**Critical behavioural rule:** approving a candidate in this system does **not** create or write a project-assignment record locally. The actual assignment happens externally, in the timetracker (Section 5.1 integration). The project only appears on the candidate's profile (Section 3.2 S11 "Projects") after the next scheduled/triggered sync pulls the updated project-and-people data from the timetracker (see Epic 13 for the sync mechanics). Request history itself, however, must reflect the approval decision immediately — it does not wait on the sync.

Acceptance criteria:
- Every proposal attempt (a candidate being proposed, then approved or rejected) is persisted as an immutable history record including timestamp, request, candidate, decision, and (for rejections) the written reason.
- Approved/rejected decisions appear immediately in Resourcing → Requests, scoped to who is entitled to see that request (Section 4.7 visibility rules).
- For internal candidates, the same decision appears in their profile's S15 section, respecting the S15 access rule: visible to Manager line and PP (read-only), never to the employee themself (Self = `—`), not to Colleague, and optionally via a shared link.
- Approving a candidate does not create any local project/assignment record and does not change S11 (Projects) on the candidate's profile at approval time.
- After the next timetracker sync ingests the corresponding assignment, the candidate's profile reflects the new project in S11 — this story's scope is verifying the request-history record exists and is correctly gated by S15 access, not the sync mechanics themselves (owned by the timetracker integration epic).
- History records survive regardless of the current status of the request (e.g. a later re-fulfilment of an unattached/reopened request does not erase prior attempts).

**Given/When/Then:**
```
Scenario: Approval is recorded in history but does not create a local project assignment
Given a DM approves an internal candidate proposed on a resourcing request
When the approval decision is saved
Then the decision appears immediately in Resourcing → Requests
And it appears in the candidate's profile section S15 for viewers with Manager-line or PP access
And no project record is created locally, and the candidate's profile Projects section (S11) is unchanged
And only after the next timetracker sync does the new project appear on the candidate's profile

Scenario: Employee cannot see their own request history
Given an employee was proposed as a candidate on a resourcing request and was rejected with a written reason
When that employee views their own profile
Then section S15 (Request history) is not rendered and not returned by the API for them
And their manager line and people partner can still see the rejection and its reason
```

---

# Epic 7: Career Timeline

## Story 7.1: Auto-Generate Timeline Events (ClickUp task: 869em1xwh)
**Current description:** As the system, I want to write a timeline event automatically whenever a tracked change occurs, so that the timeline stays current without manual upkeep.

FRs: FR39

Acceptance Criteria:
Given an employee's grade, position, department, or FTE↔subcontractor status changes, or an extended leave or a mentorship pair starts/ends, when the change is saved, then a corresponding timeline event is written automatically, for each of these event types.

**Proposed description:**
Per Section 4.9 (Career timeline, [NORMATIVE]), the career timeline is a **system-generated event log** — never a manually maintained record by default. The system must write a timeline event automatically whenever one of the following tracked changes occurs, with no user action required to keep the timeline current:

- Joining the company
- Grade change
- Position change
- Department change
- FTE ↔ subcontractor transition
- Extended leave (start)
- Mentorship pair start (also required independently by Section 4.11: "an end event is written to the career timeline (4.9)" implies the start event follows the same automatic pattern)
- Mentorship pair end (explicitly required by Section 4.11 when a manager or PP ends a pair)

Per Section 6 ("Data model notes"), grade, position, department and employment type are temporal/time-bounded data, not scalar fields with an audit log bolted on — each tracked change should be modeled as a dated, typed record so that both the current value and the full change history are queryable, and so the timeline event can be derived reliably from the change record rather than reconstructed after the fact.

Each written event must carry, at minimum: an event type/category (one of the tracked types above), an effective date, the old value and new value where applicable (e.g. previous grade → new grade), and a marker that it was system-generated (as distinct from a manually entered event — see Story 7.2, and the open question in the new "Resolve Conflicts" story). Event writing must be triggered by the underlying data change wherever it originates — profile edits by an authorized role (per the S4/S9 access matrix in Section 3.2), and the timetracker sync for project/leave-derived data (Section 5.1) — so the log stays accurate regardless of which pathway produced the change.

Per Section 4.9 presentation requirements, events written by this story feed the visual chronological timeline on the profile (S9); each event must be typed/categorized so the timeline can group, filter, or icon-differentiate event kinds and remain readable as a standalone chronology.

Per Section 3.2 (S9), read access to the career timeline is Self: R, Manager line: RW, PP: RW, Colleague: no access, Shared link: cfg (off by default). Note that "RW" for Manager line and PP at the section level refers to the section's overall access grant; manual write actions on individual events are further restricted to PP and UM only, per Story 7.2 — this story (7.1) only needs to guarantee correct automatic *reads/writes by the system itself*, respecting the section-level read visibility in 3.2 for who can view the resulting events.

**Acceptance criteria:**
- Given a tracked change (grade, position, department, FTE↔subcontractor, extended leave start, mentorship pair start, mentorship pair end, or company joining) is saved for an employee, when the change is persisted, then a timeline event of the corresponding type is written automatically, with no manual step required.
- Each auto-generated event is typed/categorized (one of the eight tracked types) and stores an effective date plus relevant before/after values.
- Auto-generated events are visually and structurally distinguishable from manually entered events (supports Story 7.2's audit/override needs and the conflict-handling story).
- The timeline renders as a standalone chronological visual sequence on the profile (S9), readable without cross-referencing other sections, respecting the read-access rules in Section 3.2.
- No tracked change type is ever silently dropped: if an event fails to write (e.g. due to a sync error from the timetracker integration), the failure is logged/surfaced rather than the timeline silently going stale, consistent with Section 7's requirement that integration failures degrade gracefully.
- Employees never have to remember to update the timeline themselves — there is no "add event" affordance available to a non-PP/UM role in the default/system-generated flow.

**Given/When/Then:**
```
Scenario: Grade change automatically produces a timeline event
Given an employee currently has grade "Middle"
And a UM with edit access updates the employee's grade to "Senior" effective 2026-09-01
When the grade change is saved
Then a new career timeline event of type "grade change" is written automatically
And the event records old value "Middle", new value "Senior", and effective date 2026-09-01
And the event is flagged as system-generated
And the event appears in the correct chronological position on the employee's profile timeline (S9)

Scenario: Timetracker sync reports a project-derived leave, edge case of a failed write
Given the internal timetracker sync reports an employee has entered extended leave
And the career-timeline write for this event fails due to a transient integration error
When the sync process completes
Then the extended leave change is still reflected in the Leaves and absences section (S10)
But the missing career-timeline event is logged/flagged for retry or investigation
And the application does not crash or block other profile functionality (per Section 7 graceful-degradation requirement)
```

## Story 7.2: Manual Timeline Edits (ClickUp task: 869em1xyr)
**Current description:** As a PP or UM, I want to manually add, edit or delete timeline events, so that I can backfill historical data or correct wrongly inferred events.

FRs: FR40

Acceptance Criteria:
Given I add a manual event with a past date, when I save it, then it appears correctly ordered in the chronological timeline.
Given I edit or delete an existing event, when I save, then the change is reflected and the timeline remains a readable chronology.

**Proposed description:**
Per Section 4.9 ("Manual override"), the career timeline is system-generated by default (see Story 7.1), but **PP and UM only** can edit, delete, and manually add timeline events. No other role — not Self, not DM, not PM, not HR Admin acting outside a PP/UM assignment, not a Colleague — has write access to individual timeline events, even though Section 3.2 (S9) grants "RW" at the section level to both "Manager line" and "PP." This story narrows that section-level grant: within Manager line, only UM (not DM/PM) may write to S9 events. This is deliberate, not an oversight — two concrete reasons are given in the spec:

1. **Historical backfill.** Current career-history data (joining dates, prior grade/position/department changes, etc., predating this system) lives only in a separate Excel headcount-change record. Since the system-generated log (Story 7.1) can only capture changes that occur *after* go-live, someone must be able to manually enter historical events to make the timeline complete from day one. PP/UM are the roles trusted to perform this backfill accurately.
2. **Correcting wrongly-inferred events.** The automatic event-writing logic (Story 7.1), especially anything derived from the timetracker sync (Section 5.1) or profile edits, can misinterpret a change (e.g., a data-entry correction that isn't really a grade change, or a duplicate/garbled sync event). PP/UM need the ability to edit or delete an incorrect auto-generated event so the timeline stays trustworthy.

Manual entries must support the same structure as system-generated ones: a type/category (drawn from the tracked event types in 4.9, or another sensible categorization if the team decides free-text notes are also needed — flag this as a design decision if pursued beyond the tracked types), an effective date (which may be in the past, to support backfill), and a description/value. A manually entered or manually edited event should be distinguishable (e.g., tagged) from a system-generated one, both for internal audit purposes and to support the conflict-handling logic described in the "Resolve Conflicts" story below.

Deleting an event should be a deliberate, auditable action — consider whether deletions are hard-deletes or soft-deletes with an audit trail, given that Section 7 makes "access control correctness" and data integrity a primary quality attribute for this platform; this decision is left to the team but should be documented.

This story does not need to solve what happens when a manual edit and a later system-generated event collide over the same underlying change — that is explicitly out of scope here and is covered as an open decision in the new "Resolve Conflicts" story.

**Acceptance criteria:**
- Only users holding the PP or UM functional/access role for a given employee can add, edit, or delete that employee's timeline events; all other roles (including DM, PM, Self, Colleague) get read-only or no access to S9 per Section 3.2, with any write attempt rejected server-side.
- A PP or UM can manually add a new event with a past (backfill) date, a type/category, and a description; upon save it is inserted into the correct chronological position in the timeline, not merely appended.
- A PP or UM can edit an existing event's date, type, or description, and the timeline re-sorts/re-renders correctly and remains a readable chronology after the edit.
- A PP or UM can delete an existing event (whether system-generated or manually entered), and the timeline reflects the removal immediately and remains a readable chronology.
- Manually created or edited events are visibly tagged as manual/edited (as distinct from system-generated), supporting audit and future conflict-resolution logic.
- All manual add/edit/delete actions are attributable (who made the change, when) to support the "correcting wrongly-inferred events" justification and general audit requirements.

**Given/When/Then:**
```
Scenario: PP backfills a historical grade change from the Excel record
Given a PP is viewing an employee's career timeline
And the employee's Excel headcount record shows a grade change from "Junior" to "Middle" effective 2019-03-15, predating the system
When the PP manually adds a new timeline event of type "grade change" dated 2019-03-15 with old value "Junior" and new value "Middle"
Then the event is saved and tagged as manually entered
And the event appears in its correct chronological position among any other existing events
And the timeline remains a readable, ordered chronology

Scenario: DM attempts to edit a timeline event and is denied
Given a DM has Manager-line access to an employee's profile via project assignment
And the DM is not the assigned UM or PP for that employee
When the DM attempts to edit or delete a career timeline event on that employee's profile
Then the request is rejected by the server regardless of any client-side UI state
And the event remains unchanged
And no partial or inferred timeline data is exposed to the DM beyond standard read access per Section 3.2
```

## Story NEW: Resolve Conflicts Between System-Generated and Manually-Edited Timeline Events (ClickUp task: NEW)
**Proposed description:**
**This is an open design decision, not a specified behavior — the spec (Section 4.9) does not answer it, and the team must explicitly discuss and sign off on an approach before this story is implemented.**

Section 4.9 establishes two coexisting write paths into the same event log: (1) the system automatically writes an event whenever a tracked change occurs (Story 7.1), and (2) PP/UM can manually add, edit, or delete events for backfill and correction purposes (Story 7.2). The spec does not define what should happen when both paths target the *same underlying change*. Concretely, this can happen when:

- The system logs an event (e.g., a grade change) from a data sync or profile edit.
- A PP or UM manually edits that event — for example, to fix an incorrect date, correct a misread value, or annotate it based on the Excel backfill record.
- A later system sync (e.g., a subsequent timetracker sync, or a re-run of the change-detection logic) re-derives the same underlying change and attempts to write a new event covering the same change window.

Should the later system-generated event overwrite, duplicate, coexist alongside, or be suppressed relative to the earlier manual edit? Section 4.9 is silent on this. Getting it wrong risks either (a) silently discarding a correction a PP/UM deliberately made (undermining the entire premise of Story 7.2 — "correct events the system inferred wrongly" is pointless if the next sync just re-breaks it), or (b) suppressing legitimate new system events because of an unrelated past manual entry, or (c) producing duplicate/contradictory events that make the timeline unreadable as a chronology (violating the 4.9 presentation requirement that the timeline be "readable... in its own right").

**Proposed default (starting point only — requires team confirmation or replacement, not to be treated as settled):**
- Manual edits are tagged as such (per Story 7.2) and are **never silently overwritten** by a later system-generated event covering the same underlying change/time window.
- When a system re-sync or re-derivation would produce a new event for a change window that already has a manual entry covering it, the system **skips writing the new event** for that window rather than overwriting or duplicating it.
- This skip should be logged/auditable (not silent to the system, even though it is silent to the end user) so that PP/UM can review cases where the system "wanted" to write something but deferred to an existing manual entry, in case the manual entry itself needs revisiting.

This default is a reasonable starting point but leaves open questions the team must resolve: How is a "same underlying change/time window" defined precisely (exact date match? overlapping ranges? same field + approximate date)? Should PP/UM be notified when a system event is suppressed this way? Is there a mechanism to intentionally let a new system event supersede an old manual one (e.g., an explicit "re-sync this event" action)? These must be decided and documented before implementation, ideally as an ADR in the intelligent repository per Section 8.3.

**Acceptance criteria:**
- A documented decision (e.g., an ADR) exists defining conflict-resolution behavior between manual and system-generated timeline events, signed off by the team, before this story is implemented — this story cannot be estimated or built against the spec alone.
- Whatever rule is chosen is implemented consistently for all tracked event types (Section 4.9's list), not decided ad hoc per type.
- The chosen behavior is covered by automated tests reflecting the specific scenario in Section 4.9's motivating case (system infers wrongly → PP/UM corrects → system re-syncs).
- Any suppression, overwrite, or duplication decision made automatically by the system is auditable (who/what changed the event, and why), consistent with the audit expectations in Story 7.2.
- The resulting timeline remains a single, readable, non-duplicated chronology per profile after any conflict resolution runs.

**Given/When/Then:**
```
Scenario: Manual correction is preserved across a later re-sync (proposed default behavior)
Given a system-generated event exists recording a grade change from "Middle" to "Senior" effective 2026-01-10
And a PP edits that event to correct the effective date to 2026-01-15, based on the Excel backfill record
And the edited event is tagged as manually edited
When a later timetracker/profile-sync re-derives the same grade change (Middle → Senior) for the same change window
Then the system does not overwrite or duplicate the manually edited event
And the system logs that it skipped writing a new event because a manual entry already covers this window
And the employee's timeline still shows exactly one grade-change event, dated 2026-01-15

Scenario: Manual entry and a genuinely unrelated new system event coexist
Given a PP has manually added a backfilled event for a department change effective 2018-06-01
And an employee has a new, unrelated department change effective 2026-08-01
When the system's automatic event-writing logic (Story 7.1) processes the 2026-08-01 department change
Then a new system-generated event is written for the 2026-08-01 change
And the earlier manually entered 2018-06-01 event remains unmodified
And the timeline correctly orders and displays both events as distinct, non-conflicting entries
```

---

# Epic 8: CDS - Career Development System

## Story 8.1: Skills Matrix Link and Assessment Log (ClickUp task: 869em1y3k)
**Current description:** "As anyone viewing S12, I want to see a link to the current skills matrix for the person's department+position and a log of past assessments, so that I have the full CDS picture.

FRs: FR41 (part 1)

Acceptance Criteria:
Given the department+position dictionary maps to a matrix file, when the profile renders CDS, then the resolved current link is shown.
Given the central matrix file is updated, when any profile pointing at it is viewed, then it reflects the new file without a per-profile change."

**Proposed description:**
Implement the read/display side of the CDS section (S12) on the employee profile: the resolved skills-matrix link and the assessment log. Per 4.10, this system is a **registry and hub only** — it never hosts the competency matrix, never runs the assessment, and never computes a score. All it stores is a dictionary mapping and a log of externally-produced results.

- **Skills matrix dictionary**: maintain a lookup of `department + position → matrix file link`, editable by HR Admin/PP (this story covers rendering; maintenance UI for the dictionary itself may be a separate admin story if not already covered elsewhere — confirm before excluding it). The profile resolves the employee's current department + position and displays the current link — do not store a static link on the employee record, or updates won't propagate.
- **Assessment log**: render each completed assessment as a row with date, assessor, a link to the external result file, and a final conclusion (free text, entered in-system per 4.10 — this is the one piece of assessment output that lives natively in the system, not just a link). This story covers read/display of the log; creation/editing of log entries and conclusions is Story 8.3.
- **Access control (per 3.2 S12 / 2.1)**: Self = R (employee can read their own CDS section, including matrix link and past assessments, but not create entries); Manager line = RW; PP = RW; Colleague = no access (section not rendered, not returned by API); Shared link = cfg (off by default, can be enabled per-link per 4.8, and if enabled must exclude write actions).
- Section must render only for department/position combinations that have a mapped matrix file; define and confirm the fallback behavior when no mapping exists yet (e.g., "no matrix assigned" state) with product before building.
- No scores, ratings, or matrix content are ever stored or computed here — only the link and the log metadata described above.

**Acceptance Criteria:**
- Given the department+position dictionary maps to a matrix file, when a viewer with access to S12 opens the profile, then the resolved current link is shown.
- Given the central matrix file/link is updated in the dictionary, when any profile pointing at that department+position is viewed, then it reflects the new link with no per-profile edit required.
- Given a completed assessment exists, when the CDS section renders, then it appears in the log with date, assessor, result-file link, and final conclusion text.
- Given a viewer is a Colleague (no Manager/PP relationship) with respect to the profile, when they request the profile via UI or API, then S12/CDS is absent entirely — not hidden client-side.
- Given the employee is viewing their own profile (Self), when they open the CDS section, then they can read the matrix link and assessment log but see no create/edit controls for assessments.
- Given no matrix file is mapped for the person's department+position, when the CDS section renders, then it shows a clear "no matrix assigned" state rather than a broken link or error.

**Given/When/Then:**
```
Scenario: Manager views resolved skills matrix link and assessment log
Given the department "Engineering" and position "Backend Developer" are mapped to matrix file "matrix-backend-v3.pdf" in the dictionary
And the employee has one completed assessment on file with date, assessor and conclusion recorded
When the employee's manager opens the employee's profile and views the CDS section
Then the manager sees the current matrix link resolving to "matrix-backend-v3.pdf"
And sees the assessment log entry with its date, assessor, result link and final conclusion

Scenario: Central matrix file update propagates without per-profile changes
Given 40 employees have department "Engineering" and position "Backend Developer"
And the dictionary entry for that department+position is updated to point to "matrix-backend-v4.pdf"
When any of those 40 employees' profiles are viewed afterward
Then the CDS section shows "matrix-backend-v4.pdf" for all of them
And no individual employee record was edited to achieve this
```

## Story 8.2: IDP Records (ClickUp task: 869em1y60)
**Current description:** "As a manager/PP, I want to create and update IDP records; as the employee, I want to mark my own IDP complete, so that development plans are tracked to completion.

FRs: FR41 (part 2)

Acceptance Criteria:
Given I am the employee, when I check "complete" on my IDP, then the completion date is recorded and shown alongside the deadline.
Given no completion date is set, when the IDP renders, then it's shown as open."

**Proposed description:**
Implement the IDP (Individual Development Plan) record within CDS/S12, covering **both** sides of its lifecycle per 4.10 and the self-service bullet in 4.3:

**Side A — Manager/PP maintenance (4.10):**
- Manager (with respect to the employee, per 2.1) or PP can create and update an IDP record consisting of: a short description, a deadline, and a link to the external IDP file (the plan itself lives outside the system, same pattern as the matrix and assessment files — no plan content is stored in-system).
- Manager/PP can edit these fields at any time; they cannot un-complete an IDP that the employee has marked complete except by editing the record fields (deadline/description/link) — clarify with product whether Manager/PP should be able to reopen a completed IDP; if not specified, disallow it in this pass and flag for follow-up.
- Per 4.10, "one record per plan" — an employee may have IDP history (prior plans) as well as a current one; the data model must support multiple IDP records per employee over time, not a single overwritable field.

**Side B — Employee self-complete (4.3, 4.10, S12 access matrix row "R (+ complete own IDP)"):**
- In self-service, the employee sees their own IDP(s) read-only (description, deadline, link) with a single **complete** checkbox action — this is the *one* write action Self has on S12.
- When the employee ticks complete, the system records the completion date (server-side, current date) and displays it alongside the deadline from that point on.
- An IDP with no completion date is *open*. This "open" derived state is what Story 8.4's filter consumes — keep the concept consistent across both stories.
- The employee cannot create, edit, or delete IDP records — only toggle completion on an existing one.

**Access control (3.2, S12):** Self = R + complete own IDP only; Manager line = RW; PP = RW; Colleague = no access; Shared link = cfg, and even when enabled a shared link must never grant write access (4.8), so the complete-checkbox must never appear on a shared link view regardless of cfg.

**Acceptance Criteria:**
- Given I am a Manager or PP for the employee, when I create an IDP record with description, deadline and external file link, then it is saved and appears on the employee's CDS section with no completion date (open).
- Given I am a Manager or PP for the employee, when I edit the description, deadline or link of an existing IDP, then the updated values are shown to all viewers with S12 access.
- Given I am the employee, when I check "complete" on my own IDP, then the completion date is recorded and shown alongside the deadline.
- Given no completion date is set on an IDP, when it renders anywhere (profile, self-service, filters), then it is treated/labeled as **open**.
- Given I am the employee, when I view my CDS section, then I can see and complete my own IDP but have no controls to create, edit, or delete IDP records.
- Given a profile is accessed via a shared link with S12 enabled, when the link viewer views the IDP, then no complete checkbox or any write control is rendered, per 4.8's "never grants write access."

**Given/When/Then:**
```
Scenario: Manager creates an IDP for their report
Given I am the Unit Manager of employee Jane Doe
When I create an IDP record for Jane with description "Complete leadership training", deadline 2026-12-01, and a link to the external IDP file
Then the record appears on Jane's CDS section with that description, deadline and link
And it has no completion date, so it is shown as open

Scenario: Employee marks their own IDP complete
Given employee Jane Doe has an open IDP with deadline 2026-12-01 and no completion date
When Jane opens her own self-service CDS section and checks "complete" on that IDP
Then the completion date is recorded as today's date
And the IDP now displays both the deadline and the completion date
And the IDP no longer counts as "open" in any filter or view
```

## Story 8.3: Manager/PP Maintain Assessments and Conclusions (ClickUp task: 869em1y79)
**Current description:** "As a manager or PP, I want to create assessment records and edit conclusions, so that the CDS log stays accurate.

FRs: FR42

Acceptance Criteria:
Given I am the manager or PP for the employee, when I add an assessment record, then it's appended to the log with date, assessor and result link, and I can edit its final conclusion text."

**Proposed description:**
Implement the write side of the assessment log within CDS/S12 (4.10). This is the counterpart to Story 8.1 (which covers read/display): here, Manager (per 2.1's derived relationship to the employee) or PP creates new assessment log entries and edits final conclusions on existing ones.

- **Create an assessment record**: fields are date (of the assessment), assessor (who performed it — free text or a person reference, confirm with product which), a link to the external result file, and a final conclusion entered as text in-system. The assessment itself (the matrix, the scoring) happens entirely outside the system per the 4.10 scope boundary — this story only records the outcome pointer and the narrative conclusion, never a score or rating value.
- **Edit a conclusion**: Manager/PP can edit the final-conclusion text on an existing log entry after creation (e.g., to correct or expand it). Whether the date/assessor/link fields are also editable after creation, or only appendable, should be confirmed — default to allowing full edit of any field on records the editor has RW access to, consistent with S12 = RW for Manager line/PP in the access matrix.
- New entries are **appended** to the log, preserving full history — assessments are never overwritten or deleted as part of normal use per the "assessment log" wording in 4.10 (a log implies append-only history; deletion, if needed at all, should be a distinct/rare admin action, not part of this story's scope).
- This action directly feeds Story 8.4's "last assessment date" filter — creating a new record updates what counts as the employee's most recent assessment date.
- **Access control**: only Manager line and PP have write access to S12 (3.2); Self and Colleague have no create/edit capability here (Self is read-only per 8.2/8.1's scope, Colleague has no access at all).

**Acceptance Criteria:**
- Given I am the Manager or PP for the employee, when I add an assessment record with date, assessor and result link, then it is appended to the assessment log (existing entries unchanged) and immediately visible to anyone with S12 read access.
- Given I am the Manager or PP for the employee, when I enter or edit the final conclusion text on a log entry, then the updated conclusion is saved and displayed with that entry.
- Given I am not a Manager or PP with respect to the employee (e.g., a Colleague, or a Manager/PP for a different employee), when I attempt to add or edit an assessment record via the API, then the request is rejected — this must hold even if the UI never exposes the control.
- Given a new assessment record is added, when the employee's "date of last assessment" is subsequently queried (Story 8.4), then it reflects the new record's date if it is the most recent.
- Given I try to enter a numeric score or rating field for an assessment, when the record is saved, then no such field exists in the data model — the system only accepts date, assessor, a result-file link, and free-text conclusion, per the 4.10 scope boundary.

**Given/When/Then:**
```
Scenario: PP adds a new assessment record with conclusion
Given I am the People Partner assigned to employee John Smith
And John's current assessment log has one prior entry dated 2025-06-01
When I add a new assessment record dated 2026-08-15 with assessor "Maria Ivanova" and a link to "john-smith-assessment-2026.pdf"
And I enter the final conclusion "Strong progress on system design; ready for senior review"
Then the log now shows two entries, with the 2026-08-15 entry appended and not replacing the 2025-06-01 entry
And the new entry displays the date, assessor, result link and conclusion text I entered

Scenario: Manager edits an existing conclusion without altering the log's history
Given employee John Smith has an assessment record dated 2025-06-01 with conclusion "Needs improvement in code review practices"
When John's Unit Manager edits that record's conclusion to "Needs improvement in code review practices; revisit in Q3"
Then the record's date, assessor and result link remain unchanged
And the conclusion text is updated to the new value
And no new log entry is created
```

## Story 8.4: Filter by Assessment Recency and Open IDP (ClickUp task: 869em1yaa)
**Current description:** "As a manager or PP, I want to filter All Employees by last-assessment-date and by has-open-IDP, so that I can find people who need attention.

FRs: FR43

Acceptance Criteria:
Given I filter "assessed before" a date, when results render, then only matching people appear, and "never assessed" is selectable as its own distinct option.
Given I filter "has an open IDP: yes" then when results render, then only people with an open IDP appear.
Given I filter "has an open IDP: yes", when results render, then only people with an open IDP appear."

**Proposed description:**
Add two CDS-derived filters (and matching columns, per 4.1's "any field can be used as a filter and as a column") to the All Employees list, per 4.10 "Filtering from All Employees" and the general filter engine in 4.1.

- **Date of last assessment** filter: a date-comparison filter offering three modes — *assessed before* a given date, *assessed after* a given date, and *between* two dates. This is a derived field (most recent date in the employee's assessment log from Story 8.3), not a stored scalar — compute it from the log at query time or maintain it as a denormalized/indexed derived value for performance (see 7: All Employees must respond within 2 seconds with 500+ records and arbitrary filters, including permission resolution).
  - **"Never assessed" must be its own selectable, distinct filter option** — not represented as an empty/null date that silently drops out of "assessed before" results. A person with zero assessment records is a first-class, findable state, since the primary use case (per 4.10) is finding people not assessed for a long time, and someone never assessed is the most extreme case of that.
- **Has an open IDP** filter: yes/no. "Open" = an IDP record exists with no completion date (defined in Story 8.2). An employee with multiple IDPs counts as "has an open IDP: yes" if at least one of their IDP records is open; confirm this aggregation rule with product if ambiguous.
- **Access scoping**: this filter operates within the All Employees list, which is itself scoped per the viewer's access (4.1, 3.2). A user must not be able to use these filters to infer CDS data about people whose S12 they cannot see — e.g., filtering must not leak "assessed"/"has open IDP" signal for a Colleague-only relationship, since S12 is `—` for Colleague. In practice this means these filters/columns should only be usable/visible to viewers who have at least some Manager/PP access within the list (consistent with 3.3 rule 5 on custom-field visibility applying to filters too).
- These filters compose with the rest of the filter engine (AND-combinable with other fields, savable as a view per 4.1) — no separate page or special-cased UI.

**Acceptance Criteria:**
- Given I filter "assessed before" a given date, when results render, then only people whose most recent assessment date is before that date appear.
- Given I filter using "never assessed" as its own distinct filter value, when results render, then only people with zero assessment log entries appear — they are not silently included in, or excluded from, "assessed before" results.
- Given I filter "assessed after" or "between" two dates, when results render, then only people whose most recent assessment date falls in that range appear.
- Given I filter "has an open IDP: yes", when results render, then only people with at least one IDP record lacking a completion date appear.
- Given I filter "has an open IDP: no", when results render, then people with zero IDP records or with all IDP records completed appear.
- Given I am a viewer with no Manager/PP relationship to some employees in the list (Colleague scope), when I apply these filters, then no CDS-derived data about those employees is exposed through the filtered results or counts, consistent with S12 being inaccessible to Colleagues.

**Given/When/Then:**
```
Scenario: PP finds people not assessed in over a year, including those never assessed
Given employee A's most recent assessment date is 2024-01-10
And employee B has zero assessment log entries
And employee C's most recent assessment date is 2026-07-01
When a People Partner filters All Employees by "date of last assessment: assessed before 2025-01-01"
Then employee A appears in the results
And employee C does not appear in the results
And employee B does not appear in the "assessed before" results
When the same PP instead selects the distinct "never assessed" filter option
Then employee B appears in those results

Scenario: Manager finds people with an open IDP
Given employee D has one IDP with no completion date (open)
And employee E has one IDP with a completion date recorded (closed)
And employee F has no IDP records at all
When employee D and E's manager filters All Employees by "has an open IDP: yes"
Then employee D appears in the results
And employee E does not appear in the results
And employee F does not appear in the results
```

---

# Epic 9: Mentorship Hub

## Story 9.1: Self-Flag Open to Mentoring (ClickUp task: 869em1ybx)
**Current description:** As an employee, I want to flag myself as open to mentoring and see my assigned mentor/mentees, so that I can participate in the mentorship program.

FRs: FR44

Acceptance Criteria:
Given I set "open to mentoring", when I save, then my status updates and I appear in the manager/PP-facing list of willing mentors.

**Proposed description:**
As an employee, on my own profile (self-service, per Section 4.3 and Section 3.2 S13), I want to toggle an "open to mentoring" flag and see who is currently assigned to me as mentor and mentees, so that I can opt into the mentorship program and understand my own mentorship relationships without needing HR to intervene.

Per S13 (3.2), the employee has RW access to their own "open to mentoring" flag and R (read-only) access to their pairs (assigned mentor, assigned mentees, ended pairs). The mentorship *status* field itself (e.g., "open to mentoring" / "mentor" / none) is derived — see NEW B — this story only covers the employee-settable flag and the read-only display; it must not let the employee set themselves to "mentor" directly.

The mentor, if assigned, is also displayed in the profile header alongside manager and people partner (4.11, "On any profile"), visible to the manager line and PP — that header rendering is shared plumbing this story should wire into or at minimum not break.

Scope notes:
- This story does not cover the manager/PP assignment flow (see Story 9.2) or ending a pair (see NEW A).
- "Available for mentoring" here means the self-flag only; it does not itself create a pair or change status to "mentor."

**Acceptance criteria:**
- Given I am an authenticated employee viewing my own profile's Mentorship section (S13), when I set "open to mentoring" to on and save, then the flag is persisted on my profile and I immediately appear in the manager/PP-facing list of willing mentors (4.11 "For manager and PP").
- Given I toggle "open to mentoring" off before any pair has been created for me, when I save, then I am removed from the willing-mentors list and no status change or timeline event is written.
- Given I have an assigned mentor, when I view my own profile, then I see that mentor's name/link in the Mentorship section and in the profile header, and I cannot edit this field (read-only per S13).
- Given I have one or more assigned mentees (as a mentor), when I view my own profile, then I see the list of current mentees, read-only.
- Given I have ended mentorship pairs in my history (as mentor or mentee), when I view my own profile, then those ended pairs remain visible read-only (supports 4.11 "Ending a mentorship" history requirement).
- Given I attempt to set my own status directly to "mentor" (e.g., via API or inline edit), when the request is processed, then it is rejected — status is a derived field, never directly settable by a user (see NEW B).

**Given/When/Then:**
```
Scenario: Employee flags themselves as open to mentoring
Given I am logged in and viewing my own profile's Mentorship section
When I turn on "open to mentoring" and save
Then the flag is persisted, and I appear in the manager/PP-facing list of willing mentors

Scenario: Employee views read-only mentor and mentee assignments
Given I have an assigned mentor and/or one or more assigned mentees
When I open my own profile
Then I see my mentor and mentees displayed read-only, with no controls to edit them directly

Scenario: Employee cannot self-assign the "mentor" status
Given I have never been part of a mentorship pair
When I try to directly set my mentorship status to "mentor"
Then the system rejects the change, because status is computed only from pair creation/ending (NEW B)
```

## Story 9.2: Assign a Mentor-Mentee Pair (ClickUp task: 869em1ydg)
**Current description:** As a manager or PP, I want to see everyone open to mentoring and pair a willing mentor with a mentee, so that mentorship relationships form.

FRs: FR45

Acceptance Criteria:
Given I select a willing mentor and a mentee available to me, when I create the pair, then it's recorded with a start date.
Given this is the mentor's first pair, when the pair is created, then the mentor's status changes from "open to mentoring" to "mentor," a filterable field.
A view of all pairs (active/ended) with dates and status is available.

**Proposed description:**
As a manager (UM, DM, PM) or People Partner, I want to view everyone who has flagged themselves as open to mentoring and, for a chosen willing mentor, assign them a mentee, so that a new mentorship pair is formed (Section 4.11, "For manager and PP").

Per 4.11: "A list of everyone who has flagged themselves as open to mentoring. Clicking a willing mentor opens an assignment flow: pick a mentee from the list of employees available to that manager, and create the pair." The mentee candidate list must be scoped to employees the acting manager/PP holds Manager or People Partner access over per Section 2.1's hierarchy resolution (transitive "reports to" + project assignment closure) — not a global employee list. A PP's scope is the people they are assigned to as people partner. The *mentor selection* (the willing-mentor list) is global — anyone flagged open to mentoring, regardless of the acting manager's org scope — since 4.11 does not restrict that list to "available to that manager"; only mentee selection is scoped that way.

This story covers only the creation of a new pair with a start date. It explicitly excludes:
- Ending a pair (NEW A).
- The derivation mechanics of the mentor status field itself (NEW B) — this story only needs to trigger that transition on first-pair creation, per the acceptance criteria below.
- The standalone all-pairs history view (NEW C) — 4.11 lists this as a related but separate capability; this story should surface pair creation, NEW C is the dedicated listing page.

Required permission: the "assign mentors" feature permission (Section 2.2 UM row; Section 2.3 extensible permission list) gates who can open this assignment flow — UM by default, PP by default (per 4.11's "manager and PP" framing plus 2.2), and any functional role explicitly granted "assign mentors."

**Acceptance criteria:**
- Given I am a manager or PP with the "assign mentors" permission, when I open the Mentorship Hub, then I see a list of every employee currently flagged "open to mentoring" (or already "mentor" status, to add additional mentees to an existing mentor).
- Given I click a willing mentor from that list, when the assignment flow opens, then the mentee picker is restricted to employees available to me per my resolved Manager/PP access scope (2.1) — no employee outside that scope is selectable or visible in the picker.
- Given I select a mentor and a mentee and confirm, when the pair is created, then it is persisted with a start date (defaulting to today, or explicitly settable) and an "active" status.
- Given this is the mentor's first ever pair (they currently have status "open to mentoring" with zero prior pairs), when the pair is created, then the mentor's status automatically changes to "mentor," and this status is exposed as a filterable field on All Employees (4.1, 4.11).
- Given the mentor already has status "mentor" (i.e., this is not their first pair), when a new pair is created for them, then their status remains "mentor" (no redundant transition) and the new pair is simply added alongside existing ones.
- Given a pair is successfully created, when I look at Section 9 (career timeline, 4.9) for both the mentor and the mentee, then a "mentorship pair start" event has been written for each.

**Given/When/Then:**
```
Scenario: Manager creates the mentor's first pair
Given a willing mentor with status "open to mentoring" and zero existing pairs
And a mentee who is within my Manager/PP access scope (2.1)
When I select the mentor, pick the mentee, and create the pair
Then the pair is recorded as active with a start date
And the mentor's status changes from "open to mentoring" to "mentor"
And a "mentorship pair start" event is added to both the mentor's and mentee's career timelines

Scenario: Mentee picker is scoped to employees available to the acting manager
Given I am a UM whose Manager access (2.1) covers only my own subordinates and their reports
When I open the assignment flow for a willing mentor
Then the mentee picker lists only employees within my resolved access scope
And employees outside that scope do not appear, even by search

Scenario: Second pair for an existing mentor does not re-trigger the status transition
Given a mentor whose status is already "mentor" with one active mentee
When I assign them a second mentee
Then the new pair is created as active
And the mentor's status remains "mentor" (unchanged)
```

## NEW A — End a Mentorship Pair with Required Final Feedback (ClickUp task: NEW)
**Proposed description:**
As a manager or PP, I want to explicitly end an active mentorship pair and be required to provide final feedback on the mentorship before it can close, so that the pairing is properly concluded, its outcome is captured, and the historical record on both profiles stays accurate (Section 4.11, "Ending a mentorship").

Per 4.11: "a manager or PP ends a pair explicitly. The end date is recorded, and final feedback on the mentorship is required to close it — a pair cannot be ended without it. Ended pairs remain visible in history on both profiles, and an end event is written to the career timeline (4.9). If the mentor has no other active mentees, their status returns to open to mentoring."

This story owns the "end" side of the pair lifecycle; the derived status-reversion mechanics are specified in detail in NEW B (this story only needs to trigger that reversion check on end). The feedback captured here is a mentorship-specific final record — it may reuse the general Feedback mechanism (4.15: subject, author, date, context, body, visibility flag) with context = "mentorship end," but regardless of implementation, it must be mandatory and blocking, not optional.

Only a manager or PP with appropriate access/permission over the mentor or mentee (per 2.1, or holding the "assign mentors" permission per 2.3) may end a pair — this should generally mirror who could have created it.

**Acceptance criteria:**
- Given an active mentorship pair, when a manager or PP with sufficient access initiates "end mentorship" and submits without entering final feedback, then the action is blocked with a validation error — the pair is NOT ended and remains active.
- Given an active mentorship pair, when a manager or PP provides final feedback text and confirms ending it, then the pair's status changes to "ended," an end date is recorded (defaulting to today or explicitly settable, and not earlier than the pair's start date), and the feedback is persisted and associated with the pair.
- Given a pair has been ended, when I view either the mentor's or the mentee's profile (S13 Mentorship section), then the ended pair still appears in each person's mentorship history, with its start date, end date, and status "ended" visible.
- Given a pair has been ended, when I view the career timeline (S9, 4.9) of both the mentor and the mentee, then a "mentorship pair end" event has been written for each, alongside the earlier "mentorship pair start" event from Story 9.2.
- Given the mentor has no other active mentee pairs after this one ends, when the end is confirmed, then the mentor's status automatically reverts from "mentor" to "open to mentoring" (see NEW B for the full derivation rule).
- Given the mentor still has at least one other active mentee pair after this one ends, when the end is confirmed, then the mentor's status remains "mentor" (no reversion).

**Given/When/Then:**
```
Scenario: Ending a pair without final feedback is blocked
Given an active mentorship pair between a mentor and a mentee
When a manager or PP attempts to end the pair without entering final feedback
Then the system rejects the action, the pair remains active, and no end date is recorded

Scenario: Ending a pair with required final feedback succeeds
Given an active mentorship pair
When a manager or PP enters final feedback and confirms ending the pair
Then the pair is marked "ended" with an end date and the feedback is saved
And an end event is written to both the mentor's and mentee's career timelines
And the ended pair remains visible in mentorship history on both profiles

Scenario: Ending a mentor's only active pair reverts their status
Given a mentor with exactly one active mentee pair
When that pair is ended with required final feedback
Then the mentor's status automatically changes from "mentor" back to "open to mentoring"

Scenario: Ending one of several active pairs does not revert status
Given a mentor with two active mentee pairs
When one of the two pairs is ended with required final feedback
Then the mentor's status remains "mentor" because at least one active mentee pairing remains
```

## NEW B — Automatic Mentor Status Transitions (ClickUp task: NEW)
**Proposed description:**
The mentorship status shown on an employee's profile and used as an All Employees filter/column (Section 4.1, Section 4.11) — values such as "open to mentoring" and "mentor" — must be a **derived/computed field**, never directly settable by any user through any surface (UI, API, inline edit, custom field editor). It is entirely a function of the employee's self-flag (Story 9.1) and their active mentorship pairs (Stories 9.2, NEW A).

Derivation rules (per 4.11):
1. An employee with the "open to mentoring" self-flag set, and zero active mentor-side pairs, has status "open to mentoring."
2. On creation of a mentor's first ever pair (Story 9.2), their status transitions to "mentor," regardless of the state of their self-flag afterward.
3. While a person has at least one active pairing as mentor, their status remains "mentor."
4. When a mentor's last remaining active mentee pairing ends (NEW A) and they have zero active mentee pairings left, their status automatically reverts to "open to mentoring" — but only if their self-flag is still set; if they had since turned the self-flag off, they should revert to no mentorship status at all (neither "open to mentoring" nor "mentor"). [Flag this edge case explicitly for product/design confirmation before implementation, since the spec does not explicitly address the flag-off-while-mentoring case.]
5. An employee who has never flagged themselves and has no pairs has no mentorship status (absent/blank), not "open to mentoring."

This story is the "engine" behind the transitions triggered by Story 9.2 (creation) and NEW A (ending); it should be implemented as a single computed-status service/function invoked by both, rather than duplicated status-setting logic in each. It must also be exposed as a genuine filterable/sortable field on the All Employees list (4.1: "mentorship status" is explicitly named as a filterable field in the list of example fields), which implies it needs to be queryable server-side (e.g., indexed or materialized), not computed only at render time per profile.

**Acceptance criteria:**
- Given an employee sets their "open to mentoring" flag and has no pairs, when their status is computed, then it reads "open to mentoring."
- Given a willing mentor receives their first pair assignment (Story 9.2), when the pair is created, then their status transitions from "open to mentoring" to "mentor" atomically with pair creation — there is no intermediate state where the pair exists but status hasn't updated.
- Given a mentor with exactly one active pairing has that pairing ended (NEW A), when the end is confirmed, then their status reverts to "open to mentoring" (assuming their self-flag is still on).
- Given a mentor with two or more active pairings has one of them ended, when the end is confirmed, then their status remains "mentor" (no reversion) because at least one active pairing remains.
- Given any user attempts to directly set an employee's mentorship status field (not via flag toggle, pair creation, or pair ending) through the UI, inline edit, or API, when the request is processed, then it is rejected — the field is read-only/derived everywhere except through the flag and pair lifecycle actions.
- Given mentorship status values exist across the org, when a manager or PP filters or adds a column for "mentorship status" on the All Employees list (4.1), then the filter/column returns correct, up-to-date values reflecting the latest pair and flag state for every employee.

**Given/When/Then:**
```
Scenario: Status transitions to "mentor" on first pair, atomically
Given an employee with status "open to mentoring" and no prior pairs
When a manager or PP creates the first mentorship pair for them as mentor
Then their computed status becomes "mentor" in the same operation, with no inconsistent intermediate state

Scenario: Status reverts to "open to mentoring" when the last active pair ends
Given a mentor with exactly one active mentee pairing and their self-flag still on
When that pairing is ended (with required final feedback, per NEW A)
Then their computed status reverts to "open to mentoring"

Scenario: Direct status edit is rejected
Given any user with edit access to an employee's profile
When they attempt to set the mentorship status field directly to "mentor" or "open to mentoring" without going through flag toggle or pair lifecycle actions
Then the system rejects the write because the field is derived, not directly editable

Scenario: Status is filterable on All Employees
Given employees across the org have a mix of "open to mentoring," "mentor," and no mentorship status
When a manager filters the All Employees list by mentorship status = "mentor"
Then only currently active mentors are returned, matching their live computed status
```

## NEW C — View All Mentor–Mentee Pairs (Active and Ended) (ClickUp task: NEW)
**Proposed description:**
As a manager or PP, I want a dedicated view listing every mentor-mentee pair in the organization — both active and ended — with start date, end date, and status, so that I can see the overall state of the mentorship program at a glance, distinct from the single-pair assignment workflow in Story 9.2 (Section 4.11: "A view of all mentor–mentee pairs, active and ended, with start date, end date and status").

This is a listing/reporting surface, not an action flow: Story 9.2 handles creating one pair at a time via the assignment flow off the willing-mentors list; NEW A handles ending one pair at a time. NEW C is the aggregate view where all pairs — regardless of who created them or when — can be reviewed together. Given the "manager/PP-facing" framing in 4.11 and the general access model (Section 2.1), each viewer should see pairs scoped to their own Manager/PP access — i.e., a UM sees pairs where the mentor or mentee falls within their reporting/project chain, a PP sees pairs among the people they are assigned to, and HR Admin / broader roles per 2.1 see correspondingly more. This view should support at least filtering by status (active/ended) and should link through to the mentor's and mentee's profiles.

Because this is materially the same underlying data set the Mentorship Hub's "willing mentors" list and Story 9.2/NEW A operate on, it makes sense to build this as one Mentorship Hub page with tabs or sections ("Open to mentor," "All pairs") rather than a fully separate feature, but the acceptance criteria below only depend on the "all pairs" listing existing and being correct.

**Acceptance criteria:**
- Given I am a manager or PP, when I open the "All pairs" view in the Mentorship Hub, then I see a table of mentor-mentee pairs scoped to my Manager/PP access (2.1), each row showing mentor name, mentee name, start date, end date (blank if still active), and status (active/ended).
- Given pairs exist both within and outside my access scope, when I view the list, then only pairs where I hold Manager or PP access with respect to at least the mentor or the mentee are shown; others are absent, not merely hidden client-side.
- Given the list contains both active and ended pairs, when I apply a status filter for "active" or "ended," then the table updates to show only pairs matching that status.
- Given a pair was created in Story 9.2 or ended in NEW A, when I refresh or reopen this view, then it reflects the current state (including the recorded end date) without manual sync.
- Given I click a mentor or mentee name in a row, when the profile opens, then I am taken to that person's profile respecting my existing access rights (no new access is granted by this view).
- Given no pairs exist within my access scope, when I open the view, then I see an empty state rather than an error.

**Given/When/Then:**
```
Scenario: Manager views all pairs within their access scope
Given several mentorship pairs exist, some within and some outside my Manager access (2.1)
When I open the "All pairs" view in the Mentorship Hub
Then I see only the pairs where I hold Manager or PP access to the mentor or mentee
And each row shows mentor, mentee, start date, end date, and status

Scenario: Filtering the pairs list by status
Given the all-pairs view contains a mix of active and ended pairs
When I filter by status = "ended"
Then only ended pairs are shown, each with a populated end date

Scenario: Navigating from the pairs list to a profile
Given the all-pairs view is open
When I click on a mentor's or mentee's name in a row
Then I am navigated to their profile, subject to my normal access rights
```

---

# Epic 12: Role-Based Dashboards (starter stories — this epic currently has zero stories in ClickUp)

## Story 12.1 — Build Shared Dashboard Engine (Layout, Counters, Table Components) (ClickUp task: NEW)
**Proposed description:**
Section 4.4 is explicit that the four role dashboards (UM, DM, PM, PP) "share components and differ in grouping and in which functional blocks appear. Build one dashboard engine, not four pages." This story is the foundational engine the other four dashboard stories (12.2–12.5) build on: a reusable set of components — summary counter tiles, a scoped/filterable table of people or projects, an "own action items" widget sorted by due date with overdue highlighting (reusing Epic 4's overdue derivation), and quick-navigation links — plus a per-role configuration layer that decides which blocks appear and how the page is grouped (by person for UM, by project for DM/PM, by department/project for PP). All data shown must go through the same access-resolution layer as the rest of the platform (Section 3, 2.1) — a dashboard is not a separate access surface.

- A single dashboard engine renders all four role variants from configuration (block list + grouping dimension), not four independently coded pages.
- Counter tiles, the scoped table, and the action-items widget are shared components reused across all four dashboards.
- Every dashboard's data is resolved through the same Manager/PP access rules as the rest of the platform (2.1, 3.2) — no dashboard-specific bypass.
- The engine supports both "grouped by people" (UM) and "grouped by project" (DM/PM) layouts, plus PP's department/project grouping (4.4.4), from the same underlying primitives.
- Adding a new role dashboard variant in the future is a configuration change, not a new page build.

**Given/When/Then:**
```
Scenario: Shared components render consistently across role dashboards
Given the dashboard engine is configured for both the UM and DM roles
When each dashboard is rendered
Then both use the same counter-tile, table, and action-item-widget components, differing only in grouping and which blocks are configured to appear

Scenario: Dashboard data respects the same access rules as the rest of the platform
Given a UM opens their dashboard
When the subordinates table and counters render
Then only people the UM holds Manager access over (per 2.1) appear, with no bypass of the standard access-resolution layer
```

## Story 12.2 — Unit Manager Dashboard (ClickUp task: NEW)
**Proposed description:**
Implements the UM dashboard per 4.4.1, grouped by people: summary counters (headcount of subordinates, active risks by level, open action items, overdue action items, active resourcing requests, open form campaigns), a table of subordinates with risk status, project, leave status and profile links, the manager's own action items sorted by due date with overdue highlighted, and quick navigation to All Employees, saved views, resourcing, the risk dashboard, the mentorship hub, and campaigns. All counters and table rows are scoped to people the UM holds Manager access over (2.1) — subordinates via the reporting-hierarchy closure.

- The dashboard shows all six summary counters listed in 4.4.1, each computed over exactly the UM's subordinates (2.1 transitive reports-to closure).
- The subordinates table shows risk status, current project, and leave status per person, each linking to that person's profile.
- The UM's own action items are listed sorted by due date, with overdue items visually flagged (reusing Story 4.3's derivation).
- Quick navigation links to All Employees, saved views, Resourcing, Risk Dashboard, Mentorship Hub, and Campaigns are present and functional.
- Risk-related counters/table cells reflect the same access rule as the standalone Risk Dashboard (Epic 5) — never showing risk data the UM isn't entitled to (there is none outside their own scope by definition, but this should not be assumed without a test).

**Given/When/Then:**
```
Scenario: UM dashboard counters and table reflect only their subordinates
Given a UM has 12 subordinates via the reporting hierarchy, three of whom have active risk records
When the UM opens their dashboard
Then the headcount counter reads 12, the risk-by-level counters reflect exactly those three records, and the subordinates table lists all 12 people with their risk, project and leave status

Scenario: Overdue action items are highlighted on the UM's own widget
Given the UM authored an action item for a subordinate with a due date in the past and status still open
When the UM opens their dashboard
Then that item appears in their own-action-items widget, sorted by due date, and is visibly marked overdue
```

## Story 12.3 — Delivery Manager Dashboard with Project Selector (ClickUp task: NEW)
**Proposed description:**
Implements the DM dashboard per 4.4.2, organized by project rather than by person. On open, it shows one table per project the DM is responsible for, each listing the people on that project with profile links, risk status, and leave status; the top-of-page counters (headcount, risk counts by level, open resourcing requests) are calculated across **all** the DM's projects by default. A project selector at the top, defaulting to "All projects," filters the whole page to a single project when selected — the counters recalculate for that project alone and only that project's table remains visible; clearing the selection returns to the all-projects view. The page also shows the DM's own resourcing requests and their state, including requests created by the PMs of their projects (per Epic 6's visibility rule, Section 4.7/2.1).

- Without a project selected, the page shows one table per DM-responsible project, and counters aggregate across all of them.
- Selecting a specific project in the selector filters the page to that project's table only, and recalculates every counter for that project alone.
- Clearing the project selection returns to the all-projects view without a page reload losing other UI state (e.g., any other active filters).
- The resourcing block shows both the DM's own requests and those created by PMs of the DM's projects (Epic 6, 4.4.2).
- Project and people data displayed is scoped to the DM's Manager access via project assignment (2.1) — a project the DM does not manage never appears, even transiently.

**Given/When/Then:**
```
Scenario: Default view aggregates across all of a DM's projects
Given a DM is responsible for three projects with a combined 18 people and 2 open risk records
When the DM opens their dashboard with no project selected
Then three project tables render, and the top counters show headcount 18 and risk counts covering the full set of 2 records

Scenario: Selecting a project filters the whole page
Given the DM's dashboard is showing all three projects
When the DM selects "Project Atlas" in the project selector
Then only Project Atlas's table remains visible, and every counter recalculates to reflect Project Atlas alone
When the DM clears the selection
Then the page returns to showing all three projects with aggregated counters
```

## Story 12.4 — Project Manager Dashboard (ClickUp task: NEW)
**Proposed description:**
Per 4.4.3, the PM dashboard is "identical to the DM dashboard, scoped to the PM's own projects." This story reuses the same dashboard engine and layout as Story 12.3, configured for the PM role: project tables, counters, project selector, and the resourcing block all behave the same way, but scoped to only the projects where the acting user is PM (2.1) — not the full DM chain. A PM never sees a DM's other projects, and their resourcing block shows only requests they themselves created (Epic 6, Story 6.1's PM-scoping rule), not PM-created requests from other PMs on the same project chain.

- The PM dashboard reuses the DM dashboard's engine/layout (Story 12.3) with no separate page implementation.
- Project tables, counters, and the project selector are scoped exclusively to projects where the viewer is PM (2.1), even if they also happen to be a DM elsewhere in the org — those are two distinct dashboard contexts, not merged.
- The resourcing block on the PM dashboard shows only requests the PM created themselves (Story 6.1's PM scoping), not the full DM-level visibility.
- All access scoping is resolved server-side per the standard access-resolution layer (2.1, 3.3.4), identical to the DM dashboard's enforcement.

**Given/When/Then:**
```
Scenario: PM dashboard is scoped to only the PM's own projects
Given a person is PM on two projects and also happens to be an ordinary team member (not PM/DM) on a third
When they open their PM dashboard
Then only the two projects where they are PM appear as project tables, with counters and resourcing scoped to those two projects only

Scenario: PM dashboard resourcing block excludes other PMs' requests
Given the PM's project also has a request created by a different PM (not applicable in a single-PM project, but relevant if project has co-PMs or the PM later becomes DM elsewhere)
When the PM views their dashboard's resourcing block
Then only requests they personally created appear, distinct from the broader DM-level visibility in Story 12.3
```

## Story 12.5 — People Partner Dashboard (ClickUp task: NEW)
**Proposed description:**
Implements the PP dashboard per 4.4.4: the same building blocks as the other dashboards, scoped to the people the PP is assigned to (2.1's PP relationship, including the HR line above), groupable by department or project, and explicitly with **no resourcing block** — PP holds no resourcing functional role by default (2.2: "People Partner... No resourcing functionality"). Section 4.4.4 marks the specific HR-facing widgets as [DESIGN FREEDOM], suggesting starting points: incomplete profile data, upcoming CDS assessments, IDPs approaching their deadline, campaign completion rates, and joiners/leavers in the period — but explicitly invites better ideas, to be judged on usefulness to a real people partner. This story should implement the dashboard shell (counters, grouping toggle, quick navigation) plus at least a first cut of two or three of the suggested widgets, with room for the team to iterate on which HR widgets are most valuable.

- The dashboard is scoped to people the PP is assigned to and the HR line above them (2.1's PP relationship definition, 3.1).
- Data is groupable by department or by project, toggled by the viewer.
- No resourcing block appears anywhere on this dashboard, regardless of any other functional role the PP might separately hold (unless that role explicitly grants resourcing permissions per 2.3, in which case it would appear via that role, not via the PP dashboard's default composition).
- At least a first version of two or three HR-specific widgets is implemented from the suggested list (incomplete profile data, upcoming CDS assessments, IDPs approaching deadline, campaign completion rates, joiners/leavers in period), clearly labeled as candidates for iteration rather than final.
- Quick navigation to All Employees, saved views, the Risk Dashboard, Mentorship Hub, and Campaigns (the PP-relevant subset) is present.

**Given/When/Then:**
```
Scenario: PP dashboard is scoped to assigned people and excludes resourcing
Given a PP is assigned to 40 employees across two departments
When the PP opens their dashboard
Then all counters and tables reflect only those 40 people, groupable by department or project, and no resourcing block is present anywhere on the page

Scenario: HR-specific widget surfaces useful signal
Given several of the PP's assigned people have IDPs with deadlines in the next 30 days
When the PP opens their dashboard
Then an "IDPs approaching deadline" widget (or equivalent) lists those people, scoped to the PP's assigned population only
```

---

# Epic 13: Integrations — Timetracker & PeopleForce (starter stories — this epic currently has zero stories in ClickUp)

## Story 13.1 — Integrate Timetracker Leaves API (ClickUp task: NEW)
**Proposed description:**
Implements the first of the two timetracker APIs described in 5.1: "Leaves — vacation, sick leave, parental and other leave types, with dates and status." This feed populates S10 (Leaves and absences, 3.2) across the platform: the employee's own self-service leave view (Story 2.4, with a link out to the timetracker to manage leave), the colleague-visible leave type on All Employees (3.3.3's whitelist explicitly includes S10 "including type"), and any dashboard leave-status columns (Epic 12). Per Section 7, "external integration failures degrade gracefully and never take down the application" — this story owns making that concretely true for the leaves feed specifically: if the timetracker is unreachable, S10 should show a clear degraded state rather than blocking profile/list rendering.

- Vacation, sick leave, parental leave, and other leave types, with dates and status, are pulled from the timetracker and reflected in S10 on the profile and in All Employees filters/columns.
- A sync failure or timeout against the timetracker leaves endpoint does not block rendering of the employee profile or the All Employees list — S10 (or the affected rows/columns) degrades to a clear "unavailable" state instead.
- Leave data respects the existing S10 access matrix (3.2) unchanged by this story — Self/Manager line/PP get R, Colleague gets R including type — this story only supplies the data, it does not alter who can see it.
- Sync cadence/triggering (polling interval, webhook, or on-demand) is decided and documented as part of this story, balancing data freshness against the 2-second All Employees performance budget (Section 7).

**Given/When/Then:**
```
Scenario: Leave data populates the profile and list from the timetracker
Given the timetracker reports an employee is on approved vacation from 2026-08-25 to 2026-08-29
When the employee's profile (S10) or the All Employees list is rendered for an entitled viewer
Then the vacation dates and type are shown, sourced from the timetracker feed

Scenario: Timetracker outage degrades gracefully
Given the timetracker leaves API is unreachable
When a viewer opens a profile or the All Employees list
Then S10 shows a clear "leave data temporarily unavailable" state, and the rest of the profile/list renders normally without crashing or blocking
```

## Story 13.2 — Integrate Timetracker Projects & People API (Permission-Critical) (ClickUp task: NEW)
**Proposed description:**
Implements the second timetracker API from 5.1: "Projects and people — projects, the people working on them, PM and DM." Unlike the leaves feed, this one is explicitly called out as "load-bearing beyond display: project assignment is an input to the permission model (2.1), so a person's PM and DM derive their access from it." This story wires the real feed into the internally-owned `ProjectAssignment` record that Epic 1 (Story 1.2) already builds the permission model against — Story 1.2 works against internally-seeded data; this story is what makes that data real and live.

**This story inherits the unresolved tension flagged in Epic 1's "Cache Access Resolution" story:** Section 7 requires integration failures to "degrade gracefully and never take down the application," while Section 2.1 requires managerial access to end immediately when a project assignment ends, and Section 6 warns that a stale permission cache is a data leak. This story must define, concretely, what happens to derived Manager access during a sync failure or delay — fail-safe (treat unsynced/stale project data as not granting access) vs. fail-open (keep last-known assignment) has real security implications and needs explicit sign-off, not an implicit default. This is flagged as a decision this story cannot skip, not a nice-to-have.

- The timetracker's projects/people feed (project, assigned people, PM, DM) populates the internally-owned `ProjectAssignment` record (Epic 1, Story 1.2), replacing/supplementing the internally-seeded data it was built against.
- A person's derived Manager access (2.1, project leg) reflects the synced project assignment data, consistent with Story 1.2's access rules.
- The sync's failure/staleness behavior with respect to access resolution is explicitly decided and documented (fail-safe vs. fail-open), reviewed against the "not sticky" access rule (2.1) and the "stale cache is a leak" warning (Section 6) — this decision is a prerequisite for closing this story, not an implementation detail to improvise.
- Sync failures do not crash or block the application (Section 7); they surface as a monitored/alertable condition given the access-control stakes involved.
- The project name and PM/DM fields populate S11 (Projects) on the profile per the existing access matrix (3.2), unchanged by this story.

**Given/When/Then:**
```
Scenario: Timetracker project assignment grants access through the real feed
Given the timetracker reports employee B newly assigned to Project X, whose PM is P
When the next sync runs
Then P's resolved access to B includes Manager access via the project leg (2.1), sourced from the live timetracker feed rather than internally-seeded data

Scenario: Sync failure behavior for access is explicit, not accidental
Given the timetracker projects/people API is unreachable during a scheduled sync
When access is resolved for a subject/object pair whose project data would normally come from this feed
Then the system behaves according to the explicitly documented fail-safe or fail-open decision for this scenario, and the failure is logged/alertable rather than silently producing an unintended access outcome
```

## Story 13.3 — Investigate & Integrate PeopleForce Candidate/Vacancy API with Fallback Link (ClickUp task: NEW)
**Proposed description:**
Implements Section 5.2: PeopleForce is the recruiting system of record, used to pull candidate information for external candidates proposed in resourcing (Epic 6, Story 6.2) and as the source of truth for vacancies. The spec is explicit that this integration's shape is not yet known: "Investigate authentication, the candidate and vacancy endpoints, custom fields, rate limits and webhooks yourselves, and record what you find as decisions in the repository," with the API documented at `developer.peopleforce.io` (including a machine-readable index at `llms.txt` "worth putting into the intelligent repository"). This story therefore has two phases: (1) a time-boxed investigation producing a written decision record (auth method, relevant endpoints, rate limits, webhook availability) committed to the intelligent repository (Section 8.3), and (2) implementation of whatever subset is feasible in the timeline, with the explicit fallback already sanctioned by the spec: "Where this integration cannot be completed in time, an external link to the candidate in PeopleForce is an acceptable fallback for this iteration" (5.2, and reiterated in 4.7).

- A written investigation record exists in the intelligent repository covering PeopleForce authentication, the candidate and vacancy endpoints, custom fields, rate limits, and webhook availability, informed by `developer.peopleforce.io` and its `llms.txt` index.
- Wherever the live integration is implemented, Epic 6 Story 6.2's external-candidate attachment flow pulls real candidate data (name, relevant profile fields) from PeopleForce rather than requiring a link only.
- Wherever the live integration is not completed in time, the external-link fallback (already built into Story 6.2/6.3 as a first-class path, not a placeholder) is what ships — this story does not block Epic 6 on PeopleForce being fully live.
- Vacancy data sourced from PeopleForce is available as the source of truth referenced by Resourcing (Epic 6), to whatever extent the investigation determines is feasible.
- Rate limits discovered during investigation are respected by the integration's call pattern (e.g., batching, caching candidate lookups) to avoid the platform being rate-limited by PeopleForce during normal Resourcing usage.

**Given/When/Then:**
```
Scenario: Investigation output is captured before implementation begins
Given the team begins work on this story
When the time-boxed PeopleForce investigation concludes
Then a decision record covering auth, endpoints, custom fields, rate limits and webhooks is committed to the intelligent repository, and implementation scope is chosen based on it

Scenario: Fallback link path ships correctly when live integration isn't completed
Given the live PeopleForce candidate integration is not completed within this iteration
When a UM attaches an external candidate in Resourcing (Story 6.2)
Then the candidate is recorded via an external link to their PeopleForce record, and this is treated as a normal, supported path rather than a broken feature
```

## Story 13.4 — Resolve Cross-System Identity (PeopleForce ↔ Platform ↔ Timetracker) (ClickUp task: NEW)
**Proposed description:**
Section 6 flags this explicitly and leaves it unresolved: "A person exists as a PeopleForce candidate, then as an employee here, and separately as a timetracker user. Decide how identity is resolved and stored — email alone is not sufficient." This is foundational, blocking work: Resourcing (Epic 6) needs to know when a PeopleForce candidate becomes the same person as a newly created employee profile; the timetracker integrations (Stories 13.1, 13.2) need to know which timetracker user record corresponds to which platform employee, since Story 13.2's data is permission-critical; and career timeline backfill (Epic 7) may need to reconcile historical identity across systems too. This story should be treated as one of the very first foundation-phase decisions (Section 8.4) — the team should explicitly evaluate identity strategies (e.g., a stable external-ID mapping table per system, keyed by more than email) and document the chosen approach and its edge cases (name changes, email changes, re-hires, contractor-to-FTE transitions) as an ADR in the intelligent repository before Stories 13.1–13.3 and Epic 6's PeopleForce path depend on it being settled.

- A documented identity-resolution strategy exists in the intelligent repository, covering how a person is matched/linked across PeopleForce, the platform's own employee records, and the timetracker, explicitly addressing why email alone is insufficient (Section 6) and what field(s) or mapping mechanism is used instead.
- The strategy defines behavior for edge cases: a PeopleForce candidate becoming a platform employee (resourcing approval → onboarding, even though pre-onboarding itself is out of scope per Section 10), a person's email changing, a re-hire, and a contractor-to-FTE (or FTE-to-subcontractor) transition already tracked as a career-timeline event type (4.9).
- Stories 13.1 and 13.2 (timetracker sync) and Epic 6's PeopleForce candidate flow reference and depend on this resolved identity mapping rather than each inventing its own ad hoc matching logic.
- The decision is recorded as a dated ADR-style entry in the intelligent repository (Section 8.3), not left as an implicit convention in code.

**Given/When/Then:**
```
Scenario: Identity strategy is documented before dependent integrations rely on it
Given the foundation phase is underway
When the team resolves the cross-system identity question
Then a written decision exists in the intelligent repository describing the chosen mapping approach and its handling of email changes, re-hires, and contractor transitions

Scenario: A resourcing-approved external candidate is correctly linked to their new employee record
Given a candidate sourced from PeopleForce is approved and later onboarded as an employee in the platform
When their platform employee profile and their original PeopleForce candidate record are compared
Then the identity-resolution mapping correctly associates the two records, using the documented strategy rather than an email-only match
```

---

# Cross-Cutting: NFRs & Process Requirements (starter stories — this list currently has zero stories in ClickUp)

## Story NFR.1 — Access-Control Test Suite (Per Audience × Relationship Path × Section) (ClickUp task: NEW)
**Proposed description:**
Operationalizes the Definition of Done's access-control clause directly: "access control is covered by tests per audience, per relationship path and per section, including negative tests for every — cell, for unflagged S7 records against both the employee and a PM, and for the colleague whitelist" (Section 9). This is the overarching test-architecture story for the whole platform's access model — distinct from, and a superset of, Epic 1's Story NEW B, which is specifically the profile-section leak harness (3.3.1). This story's scope also covers access-control correctness *outside* the profile-section matrix: Risk (Epic 5) visibility, Resourcing (Epic 6) request/candidate visibility, action item authorship/assignment rules (Epic 4), and any other place the spec gates behavior by relationship (Manager/PP/Colleague/Self) rather than by section alone. Section 7 names "access control correctness" as "the primary quality attribute" of this platform — this story is where that quality attribute gets a concrete, repeatable, automated test strategy rather than being scattered ad hoc across feature stories.

- A documented test strategy exists (agreed in the foundation phase, Section 8.4) defining how access-control tests are structured: by audience, by relationship path (reports-to vs. project-assignment vs. PP-assignment), and by section/feature, so coverage gaps are visible rather than implicit.
- Every `—` cell and flag-gated case in the Section 3.2 matrix has automated negative-test coverage (this is Epic 1 Story NEW B's specific deliverable, referenced and relied on here).
- Access-control-sensitive features outside the profile matrix — risk visibility (Epic 5), resourcing request/candidate visibility (Epic 6), action item authorship rules (Epic 4), dashboard scoping (Epic 12) — have equivalent per-relationship-path test coverage, not just happy-path tests.
- The test suite runs as part of CI and a regression in access-control coverage (a previously-passing negative test starting to fail) blocks merge, consistent with Section 9's framing of a leak as "a critical defect."
- Test data uses pseudonymized records (NFR.3) — access-control tests must not depend on or embed real personal data.

**Given/When/Then:**
```
Scenario: A negative access test exists for every matrix cell and blocks regressions
Given the automated access-control suite runs in CI
When a code change accidentally grants Self read access to S6 (Risks)
Then the corresponding negative test fails and blocks the merge

Scenario: Access-control coverage extends beyond the profile matrix
Given the test strategy is applied to Resourcing (Epic 6)
When a PM without access to a candidate's profile attempts to view it directly instead of through the profile-sharing flow
Then an automated test asserts this is rejected, with the same rigor as the profile-section matrix tests
```

## Story NFR.2 — All Employees List Performance at 500+ Records Under 2 Seconds (ClickUp task: NEW)
**Proposed description:**
Implements the explicit performance bar from Section 7: "the All Employees list with 500+ records, arbitrary filters and derived fields responds within 2 seconds, including permission resolution." This story is the load-testing and optimization pass that validates Epic 3's filter/list engine (Stories 3.1–3.6) and Epic 1's access-resolution/caching approach (Story NEW A) actually meet this bar together, under realistic conditions — arbitrary filter combinations, derived fields (years with company, CDS recency), and custom fields (Story 3.2), for a viewer whose access requires nontrivial resolution (a DM with many projects, say), not just an HR Admin with full access.

- A repeatable load/performance test exists exercising the All Employees list with 500+ synthetic (pseudonymized) employee records, combining several filters including at least one derived field and one custom field.
- The measured response time, including full permission resolution for a non-trivial access scope, is under 2 seconds at this scale, for both a wide-access viewer (HR Admin) and a narrower one requiring hierarchy/project-assignment resolution (a DM with several projects).
- Performance is verified specifically with Epic 1's chosen caching/invalidation approach (Story NEW A) active, not only against an idealized no-cache baseline.
- Performance regressions are caught automatically (e.g., a CI performance budget or periodic benchmark), not discovered only in production.
- If the 2-second bar cannot be met for some combination, that gap is documented explicitly rather than silently shipped as "close enough."

**Given/When/Then:**
```
Scenario: List meets the performance bar under realistic filtering
Given a workspace seeded with 500+ pseudonymized employee records and a DM with access to several projects
When the DM loads All Employees with three filters applied, including a derived field and a custom field
Then the response, including full permission resolution, returns within 2 seconds

Scenario: A performance regression is caught before release
Given the CI performance benchmark for the All Employees list is part of the pipeline
When a change increases response time beyond the 2-second budget at 500+ records
Then the pipeline flags the regression before merge
```

## Story NFR.3 — Pseudonymized Data for Non-Production Environments (ClickUp task: NEW)
**Proposed description:**
Implements Section 7's data-handling requirement: "The system holds personal data of real people. Use pseudonymised data in all non-production environments: real structure and volume, substituted names and contacts. Do not paste real personal data into agent contexts, logs, screenshots, or the repository." This story builds the tooling (a seed/pseudonymization script or pipeline) that produces realistic non-production data — matching real structural characteristics (org depth, department distribution, project assignment patterns, custom field usage) and volume (500+ records, per Section 7's own scale target) — with all personally identifying values substituted. This is also a process guardrail: it should be paired with a documented rule (referenced in the intelligent repository, Section 8.3) that real personal data must never be pasted into AI agent contexts, logs, screenshots, or the repository, since this platform's whole workflow involves heavy AI-agent usage per Section 8.

- A pseudonymization/seed tool exists that generates non-production employee data matching realistic org structure (hierarchy depth, department/project distribution) and volume (500+ records) with substituted names, emails, phone numbers, and addresses.
- All non-production environments (dev, staging, demo) are seeded from this tool rather than from any copy of real production data.
- The generated data includes realistic values for custom fields, risk records, management notes, and other sensitive sections, so access-control testing (NFR.1) and performance testing (NFR.2) both exercise realistic data shapes without real PII.
- A documented policy (in the intelligent repository) states that real personal data must never be pasted into AI agent contexts, logs, screenshots, or the repository, and the team's tooling/workflow is checked against this policy during the foundation phase.
- Pseudonymized data is regenerable/refreshable so it doesn't silently drift out of sync with schema changes (e.g., new custom fields) over the project's lifetime.

**Given/When/Then:**
```
Scenario: Non-production environment is seeded with realistic, non-real data
Given a new staging environment is provisioned
When it is seeded using the pseudonymization tool
Then it contains 500+ employee records with realistic org structure and volume, and no real names, emails, or contact details anywhere

Scenario: Real personal data is never required for agent-assisted development
Given a developer uses an AI coding agent against the staging environment
When the agent inspects employee data as part of its work
Then all data it encounters is pseudonymized, and no real personal data has been pasted into its context, logs, or the repository
```

## Story NFR.4 — Accessibility & Responsive Layout Pass (ClickUp task: NEW)
**Proposed description:**
Implements Section 7's requirement: "Accessibility and responsive layout for the list, profile and dashboard pages." This story is the dedicated audit-and-fix pass across the platform's three highest-traffic surfaces — the All Employees list (Epic 3), the Employee Profile (Epic 1, Story 1.6), and the role dashboards (Epic 12) — covering keyboard navigation, screen-reader semantics (labels, roles, live-region announcements for dynamic content like filter results and inline edits), color contrast (relevant given the risk-level and overdue-highlighting visual treatments in Epics 4 and 5), and responsive behavior across common breakpoints (desktop, tablet, and at minimum a usable narrow/mobile view for self-service).

- The All Employees list, Employee Profile, and all four role dashboards are navigable via keyboard alone, with visible focus states.
- Screen-reader semantics are correct for dynamic content specifically: filter result updates, inline-edit success/failure, and overdue/risk-trend visual indicators all have a non-color-only, screen-reader-accessible equivalent (not relying on color/icon alone, per WCAG-style contrast and non-text-alone conventions).
- Color contrast for status indicators (risk level colors, overdue highlighting, trend arrows) meets a documented accessibility standard (e.g., WCAG AA), decided during the foundation phase (Section 8.4) and applied consistently.
- The three surfaces are usable (not just "not broken") at common responsive breakpoints, with self-service specifically checked on a narrow/mobile viewport since employees are the most likely to access it off a phone.
- An accessibility audit (automated tooling plus a manual pass) is run against these three surfaces and documented findings are triaged/fixed before this story is considered done.

**Given/When/Then:**
```
Scenario: All Employees list is fully keyboard-navigable
Given a user relies on keyboard-only navigation
When they use the All Employees list to apply a filter, sort a column, and open a profile
Then every action is achievable via keyboard alone, with visible focus indicators throughout

Scenario: Risk and overdue indicators are accessible beyond color alone
Given a screen-reader user views a risk dashboard row or an overdue action item
When the assistive technology announces the row/item
Then the risk level or overdue status is conveyed through text/label, not only through color or icon
```

## Story NFR.5 — Intelligent Repository & BMAD Parallel-Decomposition Setup (ClickUp task: NEW)
**Proposed description:**
Implements Section 8's engineering-process requirements, which the spec states explicitly are graded and non-optional: BMAD as the starting framework (8.1), decomposition and workspace/submodule configuration that makes genuine parallel work possible so "a situation where one person waits for another is unacceptable and will be treated as a process defect regardless of the output" (8.2), a mandatory intelligent repository holding specs, decisions, call transcripts, external API documentation, and agent rules/skills (8.3), and a foundation phase with named owners per topic whose findings are aligned and written down before implementation starts (8.4). This story is the setup work itself: standing up the intelligent repository structure, configuring BMAD (or documenting a deliberate, recorded decision to deviate — 8.1 explicitly allows migration/customization "as a deliberate decision recorded in the repository rather than a drift"), and structuring the backlog/workspace so the parallelism rule in 8.2 is actually achievable given this epic breakdown (e.g., minimizing hard sequential dependencies between epics where avoidable, and flagging the ones that remain, like Epic 4's campaign-action-item story depending on Epic 10).

- An intelligent repository is stood up containing (at minimum) the specs, architecture/foundation-phase decisions, external API documentation (including the PeopleForce `llms.txt` index per Section 5.2), and agent rules/skills referenced elsewhere in this document.
- BMAD is configured as the starting framework, or a deliberate decision to deviate is recorded in the repository with rationale, per 8.1.
- The current epic/story decomposition (this document plus the existing ClickUp board) is reviewed specifically for parallelization: hard sequential dependencies between stories/epics are identified and flagged (e.g., the campaign→action-item dependency in Epic 4, the identity-resolution dependency in Epic 13's Story 13.4 blocking 13.1–13.3 and parts of Epic 6) so the team can staff around them rather than discovering blocking chains mid-sprint.
- Foundation-phase topics (per 8.4: prototyping/design approach, best-practices/technology choices, testing architecture, and any other team-identified foundational topic) have named owners and their findings are written down in the repository before feature implementation begins.
- Inter-team communication and status are captured per 8.5, in a form usable for post-iteration analysis.

**Given/When/Then:**
```
Scenario: Repository and BMAD setup exist before feature implementation begins
Given the foundation phase starts
When the team sets up their workspace
Then an intelligent repository exists with specs, decisions, API docs and agent rules/skills, and BMAD is configured (or a recorded deviation decision exists)

Scenario: Hard sequential dependencies across epics are identified, not discovered mid-sprint
Given the current epic/story decomposition includes some stories that depend on others being scoped first (e.g., Epic 4's campaign-action-item story on Epic 10, or Epic 13's identity story blocking other integration stories)
When the foundation phase reviews the decomposition for parallelizability
Then these dependencies are explicitly documented and staffing is planned to minimize idle waiting, consistent with Section 8.2's "no one waits" rule
```

---


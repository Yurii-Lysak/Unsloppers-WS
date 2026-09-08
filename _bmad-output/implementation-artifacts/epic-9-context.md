# Epic 9 Context: Mentorship Hub

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Enable a lightweight mentorship program built on employee self-declared willingness, manager/PP-driven pairing, and mandatory closing feedback on every ended relationship. Employees control whether they are open to mentoring; assigners browse a company-wide willing-mentor pool and pair mentors with mentees within their access scope; every pair end captures a closure note managers can review later. Mentor status is always derived from the self-flag and active pairs — never directly editable. Scope is pair formation, ending, and visibility only — not mentorship goals, session logs, or progress tracking.

## Stories

- Story 9.1: Self-Flag Open to Mentoring
- Story 9.2: Assign a Mentor-Mentee Pair
- Story 9.3: End a Mentorship Pair with Required Final Feedback
- Story 9.4: Automatic Mentor Status Transitions
- Story 9.5: View All Mentor-Mentee Pairs (Active and Ended)

## Requirements & Constraints

- Employees may toggle an open-to-mentoring self-flag on their own profile and view their assigned mentor and mentees read-only. They cannot set mentor status directly.
- Holders of the *assign and end mentorships* functional permission browse a company-wide pool of everyone currently flagged open-to-mentor. The pool exposes identity-card data plus the flag only — never S13 mentorship-section content.
- Mentee selection is scoped to the assigner's resolved access; the willing-mentor list itself is global.
- Creating a pair records an active relationship with a start date, transitions the mentor's derived status to "mentor" on their first pair, and writes a pair-start event to both employees' career timelines (S9).
- Ending a pair requires mandatory closing feedback before the end date is recorded. The feedback is stored on the pair record itself — not routed through the general Feedback entity — and is readable only by the reporting line, project line, and PP; never by the mentor, mentee, or colleagues.
- After the mentor's last active pair ends, derived status reverts to "open to mentoring" only if the self-flag is still on; otherwise it reverts to no status. Turning the flag off while still mentoring removes the person from the future-assignment pool but does not alter the active pair or current "mentor" status until that pair actually ends.
- A dedicated all-pairs view lists active and ended pairs organization-wide, filtered to pairs where the viewer holds Manager or PP access to the mentor or mentee. Rows show mentor, mentee, dates, and status; clicking a name navigates under normal access rules — the view grants no new access.
- The assigned mentor appears in every profile header alongside manager and PP, but the header mentor field follows a stricter visibility rule than general S1 Colleague access: only the reporting/project line and PP may see it.
- S13 (Mentorship section) is never shareable via Shared Links. Pair creation must be rejected server-side unless the prospective mentor's open-to-mentoring flag is on at write time, regardless of what the pool UI displays. Concurrent attempts to end an already-ended pair must surface a conflict error, not silently overwrite the closure note.

## Technical Decisions

- **D4:** Mentor status on last-pair-end follows the self-flag rule exactly; un-flagging mid-pair never touches the active pair.
- **D5:** The S1 profile-header mentor field overrides the general Colleague-R grant — visible only to reporting/project line and PP.
- **D6:** Closure feedback lives on `MentorshipPair`, deliberately decoupling this epic from Epic 11 (Feedback). Departure-triggered auto-close (Epic 14) supplies a system-generated note and bypasses the mandatory-note gate.
- **D20:** Consent is enforced at pair-creation write time; concurrent human pair-ending uses optimistic-concurrency rejection (distinct from departure auto-close, which is an idempotent no-op per D17).
- The `mentorship` module owns pair lifecycle logic. It consumes **C4 `TimelineEventWriter`** for pair start/end events and **C8 `PermissionChecker`** for the *assign and end mentorships* gate — functional permissions are never conflated with section visibility (C1). It registers a **departure-hook** provider so Epic 14 can auto-end active pairs during the departure cascade.
- Mentor status is a derived, queryable field — any direct write attempt through any surface is rejected.

## UX & Interaction Patterns

- **Mentorship Hub** (nav: Mentorship) hosts the open-to-mentor list, pair-assignment flow, and all-pairs table. Managers and PP use it for assignment; everyone manages their own flag via My Profile.
- **Mentorship Pair Card** appears on the Hub and My Profile. Active pairs expose an "End pair" action opening a required-feedback Dialog — primary button disabled until text is entered. Ended pairs render read-only with positive status treatment plus the recorded feedback, visible to Manager/PP only.
- Two independent empty states on the Hub: one for an empty willing-mentor pool, one for no pairs yet — distinct copy for each.
- Microcopy stays direct: e.g. "Closing feedback required to end this pair." — no false cheerfulness on sensitive management actions.

## Cross-Story Dependencies

- **Epic 1** access resolution and functional-role permission infrastructure (`assign and end mentorships`; launch default pending PO sign-off: UM + managers/PP) gate every mentorship action.
- **Epic 7** (Career Timeline) receives pair start/end events through C4; an early C4 stub is acceptable so mentorship can ship before timeline UI is complete.
- **Epic 14** (Departure) auto-ends active pairs with a system-generated closure note; the mentorship departure-hook must be idempotent.
- Story 9.4 (status transitions) depends on pair creation (9.2) and ending (9.3) being in place; 9.3's status revert invokes 9.4's rule.

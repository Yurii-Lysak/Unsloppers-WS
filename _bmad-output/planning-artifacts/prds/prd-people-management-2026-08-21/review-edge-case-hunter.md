# Edge-Case Lens — People Management Platform PRD

```json
[
  {
    "location": "prd.md:128-144",
    "trigger_condition": "Viewer simultaneously qualifies for two+ access roles w.r.t. same subject (e.g., PP and Project line)",
    "guard_snippet": "When a viewer's relationships resolve to more than one access role for the same subject, the system grants the union of all resolved roles' section access, per section.",
    "potential_consequence": "Undefined precedence could silently deny or over-grant sections depending on implementation choice."
  },
  {
    "location": "prd.md:175-180",
    "trigger_condition": "Department's-manager set to a member of that same department, creating self-managed reporting-line access.",
    "guard_snippet": "A department's manager may never be a current member of that department or any of its nested sub-departments.",
    "potential_consequence": "Employee gains Reporting-line access to their own profile, exposing S6 risk data barred to Self."
  },
  {
    "location": "prd.md:166-174",
    "trigger_condition": "Shared-link creator's own access narrows (e.g., Reporting line to Project line) after link creation.",
    "guard_snippet": "On every view, the link's exposed sections are re-clamped to the creator's currently-held access level, not just presence of any relationship.",
    "potential_consequence": "Recipient keeps viewing sections the creator is no longer entitled to grant."
  },
  {
    "location": "prd.md:128-135;175-180",
    "trigger_condition": "Two sequential legitimate manager-field edits form a cycle (A manages B, B manages A).",
    "guard_snippet": "An employee's-manager edit is rejected if the new manager is already a descendant of the employee in the reporting chain.",
    "potential_consequence": "Reporting-line's transitive-closure walk loops indefinitely, breaking access resolution for the whole cycle."
  },
  {
    "location": "prd.md:175-194",
    "trigger_condition": "Two actors write conflicting values to the same access-switch field concurrently.",
    "guard_snippet": "A concurrent conflicting write to the same access-switch field is rejected with a conflict error, not silently overwritten.",
    "potential_consequence": "One admin's change is silently lost with no journal indication a conflict occurred."
  },
  {
    "location": "prd.md:175-180;481-485",
    "trigger_condition": "Department's-manager field cleared via FR-7 directly, without departure, leaving no replacement.",
    "guard_snippet": "Clearing a department's-manager field is blocked unless a replacement manager is designated in the same change.",
    "potential_consequence": "Department members lose all Reporting-line coverage until someone notices and reassigns a manager."
  },
  {
    "location": "prd.md:182-188",
    "trigger_condition": "The last two full-profile-access holders each revoke the other at the same instant.",
    "guard_snippet": "A revocation of full profile access is rejected at commit time if the current holder count, re-checked rather than cached, would fall to zero.",
    "potential_consequence": "Concurrent mutual revocation drives holder count to zero, violating the never-zero invariant."
  },
  {
    "location": "prd.md:182-188;481-485",
    "trigger_condition": "The sole remaining full-profile-access holder is recorded as departed (FR-62).",
    "guard_snippet": "Recording a departure is blocked for the last remaining full-profile-access holder until the grant is transferred, alongside the existing manager/PP re-parenting gate.",
    "potential_consequence": "FR-63's blanket grant revocation removes the last holder, leaving zero full-profile-access holders platform-wide."
  },
  {
    "location": "prd.md:300-304;486-487",
    "trigger_condition": "UM who proposed a candidate departs while the proposal is still pending DM decision.",
    "guard_snippet": "A pending proposal authored by a now-departed UM is flagged for reassignment to another UM in the routed department rather than left silently pending.",
    "potential_consequence": "Proposal sits indefinitely with no owner able to follow up or resubmit."
  },
  {
    "location": "prd.md:305-310;486-487",
    "trigger_condition": "DM who must approve/reject a proposed candidate departs before deciding.",
    "guard_snippet": "A resourcing request awaiting decision from a departed DM is auto-routed to that DM's successor or flagged for reassignment.",
    "potential_consequence": "Candidate proposal is stuck awaiting approval from an account that is now deactivated."
  },
  {
    "location": "prd.md:371-373;486-487",
    "trigger_condition": "Mentor and mentee of the same active pair both depart on overlapping effective dates.",
    "guard_snippet": "Departure-cascade pair auto-closure is a no-op if the pair record is already ended, so a second departure trigger never re-writes it.",
    "potential_consequence": "Same pair record is auto-ended twice, double-writing or overwriting the system-generated closing note."
  },
  {
    "location": "prd.md:486-487;535",
    "trigger_condition": "Shared-link creator's own departure ends their relationship, per SM-6 the link should stop working.",
    "guard_snippet": "The departure cascade (FR-63) explicitly revokes every shared link the departing employee created, in addition to the grants they held.",
    "potential_consequence": "Link keeps working for its recipient because no cascade step actually enforces SM-6's stated invalidation."
  },
  {
    "location": "prd.md:260-264;486-487",
    "trigger_condition": "Author manually cancels an action item (FR-22) as its assignee's departure auto-cancels it (FR-63) concurrently.",
    "guard_snippet": "A cancellation write is a no-op against an item already in a cancelled state, regardless of which cancellation path arrives second.",
    "potential_consequence": "Item's cancellation reason or cancelled-by field is overwritten depending on write ordering, corrupting history."
  },
  {
    "location": "prd.md:218-222;486-487",
    "trigger_condition": "A saved view's creator departs; only the creator can edit or delete a saved view.",
    "guard_snippet": "On the creator's departure, a saved view's edit/delete rights transfer to a current manager/PP holder, or the view is retired.",
    "potential_consequence": "Shared view becomes permanently frozen and undeleteable once its creator's account deactivates."
  },
  {
    "location": "prd.md:305-318",
    "trigger_condition": "An approved candidate is later found ineligible, or the DM wants to reverse an approval.",
    "guard_snippet": "An approved candidate can be reversed to rejected with a written reason, which frees the headcount slot it had filled.",
    "potential_consequence": "Headcount stays permanently marked filled by a candidate who never actually joined."
  },
  {
    "location": "prd.md:294-315",
    "trigger_condition": "Headcount is edited downward below the count of already-approved/filled slots.",
    "guard_snippet": "A headcount edit that would drop below the already-filled count is rejected, or requires un-approving enough candidates first.",
    "potential_consequence": "Filled/remaining counts go negative, contradicting FR-31's 'always accurate' guarantee."
  },
  {
    "location": "prd.md:311-315",
    "trigger_condition": "Remaining headcount reaches zero but the DM leaves the request open (no auto-close, per FR-31).",
    "guard_snippet": "Approving a new candidate is blocked once remaining headcount is zero, unless the DM first raises headcount or closes the request.",
    "potential_consequence": "Request silently over-fulfills beyond its stated headcount with no stated cap enforcement."
  },
  {
    "location": "prd.md:319-321",
    "trigger_condition": "The routed department's UM (FR-7 access switch) changes while a request is still pending fulfillment.",
    "guard_snippet": "A pending request's routing re-resolves to the department's current UM on every view rather than pinning to the UM at creation time.",
    "potential_consequence": "Request stays addressed to a UM who no longer manages that department, with no successor notified."
  },
  {
    "location": "prd.md:452-457",
    "trigger_condition": "Sync fails, recovers briefly, then fails again repeatedly, never reaching 4 continuous failed hours.",
    "guard_snippet": "The 4-hour withdrawal clock accumulates total failed time within a rolling window rather than resetting to zero on any brief recovery.",
    "potential_consequence": "Access-withdrawal safeguard never fires despite cumulative staleness far exceeding the PRD's stated 4-hour bound."
  },
  {
    "location": "prd.md:452-457",
    "trigger_condition": "Sync flaps within the 15-minute grant window, never sustaining success long enough to confirm assignment.",
    "guard_snippet": "A new project assignment is confirmed once a single successful sync observes it, not requiring 15 uninterrupted minutes.",
    "potential_consequence": "A genuinely new project assignment never registers as an access grant during a flapping period."
  },
  {
    "location": "prd.md:368-370",
    "trigger_condition": "An 'assign and end mentorships' holder creates a pair for a person whose open-to-mentoring flag is currently off.",
    "guard_snippet": "Creating a mentor-mentee pair is rejected server-side unless the prospective mentor's open-to-mentoring flag is currently on, not merely discoverable via the pool UI.",
    "potential_consequence": "A non-consenting, unflagged employee is assigned as a mentor without ever having opted in."
  },
  {
    "location": "prd.md:371-373",
    "trigger_condition": "Two permission holders submit closing notes to end the same pair concurrently.",
    "guard_snippet": "Ending a pair is rejected if the pair's status has already transitioned to ended by a concurrent request.",
    "potential_consequence": "Second closing note silently overwrites the first, or the pair is double-logged as ended."
  }
]
```

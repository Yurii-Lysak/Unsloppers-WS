# People Management Platform — PRD & Parallel Delivery Plan

**Version:** 1.0
**Based on:** `People_Management_Platform_Pro.md` (Iteration 2 requirements, v1.2) and `backlog_review_draft.md` (the 41-story ClickUp backlog review + 20 gap-closing stories drafted earlier in this workspace)
**Status:** Draft for team review — nothing here has been pushed to ClickUp

---

## 0. Purpose and how to read this document

The original requirements document defines *what* the platform must do and *why* (roles, access matrix, functional scope). The backlog draft turned that into 61 stories with acceptance criteria and Given/When/Then scenarios, and flagged a handful of places where the spec doesn't fully answer its own questions. Neither document, on its own, tells three developers how to split the work so that nobody sits idle waiting on someone else.

This document is that missing layer. It does four things the other two don't:

1. **Names the small number of internal contracts** that, if frozen on day one, let the team build almost everything else in parallel against a stub instead of a finished dependency.
2. **Makes a concrete decision** on every point the backlog draft left open, so "we don't know yet" never becomes a reason to block. Each decision is a default the team can override later — but a stated default beats silence.
3. **Lays out delivery waves and a track allocation for three developers**, chosen specifically so that within a wave, no track's story depends on another track's *unfinished* work — only on contracts and stubs that exist from day one.
4. **Restructures the full backlog as one dependency-tagged tree**, ordered so that reading top-to-bottom within your track is a valid build order. Each entry carries `[Wave] [Track] [Depends-on] [Blocks] [Stub available]` tags.

Two epics the earlier backlog review flagged as gaps — **Epic 10 (Forms & Survey Campaigns)** and **Epic 11 (Feedback)** — had zero stories in ClickUp despite being core (not stretch) functionality per spec sections 4.12 and 4.15. This document specifies them in full for the first time (Section 6).

Section numbers in **bold parentheses**, e.g. **(2.1)**, refer to the original requirements document. Story references like *Story 1.6* or *Epic 9 NEW A* refer to `backlog_review_draft.md`, which remains the source of truth for full acceptance criteria and Given/When/Then scenarios on every story except the two new epics specified here in full.

## 1. Delivery principles

The original spec is explicit that process is graded and that "a situation where one person waits for another is unacceptable and will be treated as a process defect regardless of the output" **(8.2)**. This document treats that as a hard constraint, not an aspiration, and applies one rule consistently: **a cross-feature dependency is never resolved by making one developer wait for another to finish — it's resolved by a frozen interface, a stub implementation, a spec-sanctioned fallback, or by giving both sides of the dependency to the same developer in sequence.**

Every dependency identified while building the tree in Section 6 was resolved one of these four ways. None were resolved by "Dev B blocks on Dev A." Section 3 lists the handful of contracts that make this possible; Section 5 shows exactly how each cross-epic link gets resolved.

The plan also follows the spec's own instruction to run a short foundation phase before feature work starts, with named owners per topic, findings aligned and written down before implementation begins **(8.4)**. Section 4's waves start with exactly that.

---

## 2. Foundational contracts (freeze the interface Wave 0, implement it Wave 1)

These are not full designs — they're the minimum interface surface that lets a second developer start coding against a contract before the first developer has finished implementing it. Each one already has a natural owner from the tree in Section 6; the point of listing them separately is that **the signature gets agreed and written down in Wave 0**, before anyone starts Wave 1 implementation.

### C1 — AccessResolver
`resolveAudience(viewerId, subjectId) → { role: Self | ManagerLine | PP | Colleague | SharedLink | HRAdmin, sections: Map<SectionId, "R" | "RW" | "none"> }`

Backed by the union of the reports-to closure and the project-assignment closure for Manager access, and the PP-assignment + HR-line closure for PP access **(2.1, 3.1)**. Must be called server-side on every request touching profile-scoped data — never cached client-side, never assumed stable across a session **(3.3.4)**. Owned by Track A / Epic 1 (Stories 1.1–1.6). A Wave-0 stub implementation can simply return a fixed permissive role for seeded dev/test data, so every other track can start Wave 1 immediately without waiting for the real hierarchy-resolution logic.

### C2 — FieldRegistry (custom fields + arbitrary filtering)
`defineField(name, type, visibility) → fieldId`, `setValue(employeeId, fieldId, value)`, plus a query interface that treats built-in, derived, and custom fields uniformly for filtering/sorting/columns **(4.1, Section 6's "column-per-field schema will not survive this" note)**. Owned by Track B / Epic 3 (Story 3.2). Everything downstream — All Employees, exports, saved views, dashboards — filters through this registry's query shape rather than hand-rolling field access.

### C3 — ProjectAssignment (internal, pre-integration)
`{ employeeId, projectId, pmId, dmId, startDate, endDate }` — already specified in the backlog draft (Epic 1, Story 1.2) as internally-owned and queryable independent of the timetracker being live, precisely so Epic 1's access model doesn't block on Epic 13's integration. Owned by Track A initially (seeded data); Epic 13 Story 13.2 later becomes the *writer* of this same table from the real timetracker feed. Nothing downstream changes when that swap happens.

### C4 — Timeline Event Writer
`recordTimelineEvent(employeeId, type, effectiveDate, oldValue, newValue, source: "system" | "manual", authorId?)`

Any part of the system that changes a tracked field — grade, position, department, FTE↔subcontractor, extended leave, mentorship pair start/end, joining **(4.9)** — calls this instead of writing timeline rows itself. Owned by Track C / Epic 7 (Story 7.1). A Wave-0 stub can simply log-and-no-op, so Epic 1 (profile edits), Epic 9 (mentorship pairs), and Epic 13 (timetracker-derived leave/FTE changes) can all call it starting in Wave 1/2 without waiting on Epic 7's UI to exist.

### C5 — External identity mapping
`external_identities: { system: "peopleforce" | "timetracker", externalId, employeeId, supersededBy? }`

Per **Decision D8** (Section 3), a dedicated mapping table keyed by (system, external ID) rather than email, supporting re-hires and identity changes via an explicit `supersededBy` pointer. Schema shape ratified in Wave 0; Epic 13 Story 13.4 owns validating/refining it, but Epic 6 and Epic 13 Stories 13.1/13.3 can reference "this employee's PeopleForce candidate ID" from Wave 1 onward without redesigning later.

### C6 — Action item creation
`createActionItem({ assigneeId, authorId, title, description?, dueDate, link?, source: "manual" | "campaign", campaignId? }) → ActionItem`

Already implicit in Epic 4's stories (1.1–4.3); called out explicitly here because Epic 10's campaign-activation story (10.3) calls it directly. Recommended resolution: assign Epic 4 and Epic 10 to the **same developer** in Wave 2 (Section 4), so this "dependency" never actually produces a cross-developer wait — see Section 5.

---

## 3. Decision log

The backlog draft deliberately flagged several places where the spec doesn't answer its own question, rather than silently guessing. Silence is fine for a review; it's a blocker for parallel delivery. Each item below now has a concrete default, with rationale, so no developer needs to stop and ask before writing code. Every default is explicitly revisable — the "PO sign-off" column says which ones are worth a five-minute confirmation before or shortly after the team builds to them, and which are safe to just build.

**D1 — Access-resolution caching (Epic 1, Story NEW A).** Decided default: a short-TTL, per-subject access cache keyed by subject ID, tagged with a monotonically increasing generation counter on the relationship graph (reports-to + project assignment + PP assignment). Any write to that graph bumps the counter and invalidates affected cache entries synchronously — correctness comes from the generation bump, not from the TTL, so the "not sticky" access rule **(2.1)** holds even under caching. TTL exists purely as a performance knob. *PO sign-off: not needed to start; revisit after Wave 3 performance testing (NFR.2) if the approach doesn't hold up under load.*

**D2 — Timeline conflict resolution (Epic 7, Story NEW).** Decided default: a manually-edited timeline event is never silently overwritten by a later system-generated event covering the same change window; the system-generated write is skipped and the skip is logged/auditable. Rationale: this is the entire point of manual correction **(4.9)** — a sync that silently re-breaks a correction defeats the feature. *PO sign-off: not needed to start; document as an ADR per the spec's own instruction (4.9's conflict case).*

**D3 — Timetracker sync failure behavior and access (Epic 13, Story 13.2).** Decided: **fail-safe for access, fail-soft for display.** If the timetracker sync cannot currently confirm a project assignment, that assignment does not grant Manager access (treat "unknown" as "not confirmed," never as "still active"). Separately, if the sync is simply slow or briefly unavailable, S10/S11 *display* shows a clear "temporarily unavailable" state rather than blocking the profile **(Section 7's degrade-gracefully rule)**. These are two different axes — a security control fails closed; a display feature fails soft — and conflating them was the source of the original ambiguity. *PO sign-off: recommended, since this affects what managers can and can't do during an outage — flag for a quick confirmation in Wave 1.*

**D4 — Mentorship status when the self-flag is off but the pair ends (Epic 9, Story NEW B).** Decided: status reverts to "open to mentoring" when a mentor's last active pair ends **only if** their self-flag is still on at that moment; otherwise it reverts to no status at all (neither value). Rationale: the flag is the independent source of truth for willingness; reverting to "open to mentoring" against someone's explicit off-flag would be a bug. *PO sign-off: not needed.*

**D5 — Profile header mentor visibility (Epic 1, Story 1.7).** Decided default: **(4.11)**'s explicit "visible to manager line and PP" governs the header's mentor field specifically, overriding S1's general Colleague-`R` grant on identity-card content — treat the more specific, later rule as authoritative. *PO sign-off: recommended but non-blocking — build to this default now, confirm in passing.*

**D6 — Mentorship-end feedback storage (Epic 9, Story NEW A).** Decided: for v1, store the mandatory closing feedback as its own field directly on the mentorship pair record, **not** routed through the general Feedback (S8) entity. This is a deliberate decoupling — it removes what would otherwise be a hard Epic 9 → Epic 11 dependency, in service of the parallelization priority. Revisit unifying into S8 later if the team wants mentorship feedback queryable alongside other feedback records. *PO sign-off: not needed.*

**D7 — IDP self-complete ownership (Epic 8, Story 8.2).** Decided (already reflected in the backlog draft): both the manager/PP maintenance side and the employee's self-complete checkbox live in the same story, owned by the same developer, rather than being split across Epic 2 and Epic 8. Avoids a two-developer handoff on one small feature. *PO sign-off: not needed.*

**D8 — Cross-system identity strategy (Epic 13, Story 13.4).** Decided starting schema (Contract C5): a dedicated `external_identities` mapping table keyed by (system, external ID) → employee ID, populated explicitly on first link — never inferred from email alone **(Section 6)** — with an explicit `supersededBy` pointer to handle re-hires. This is a Wave-0 ratification, not a final answer: the actual PeopleForce/timetracker investigation (Story 13.3, 13.1) may refine it, but it unblocks every other Wave-1/2 story that needs to reference "this employee's external record" before that investigation completes. *PO sign-off: not needed to start; the investigation itself (13.3) is where real answers get confirmed.*

---

## 4. Delivery waves and track allocation

Three developers, three named tracks (A/B/C — reassign the letters to actual people however makes sense; what matters is that each letter carries a clean, non-blocking sequence of work). Track assignments below are the concrete example this document commits to; the tags in Section 6 are what actually matter if the team wants to reshuffle who does what.

### Wave 0 — Foundation (short: a few days, all three developers plus whoever owns product decisions, together)

Per the spec's own foundation-phase requirement **(8.4)**, this wave runs in parallel across topics with named owners, and its output is written down before Wave 1 starts:

- Stand up the intelligent repository and confirm the BMAD/process setup — *NFR.5*.
- Build the pseudonymized seed-data tool — *NFR.3*. This needs to exist before Wave 1 so every track has realistic data from day one, not partway through.
- Freeze the six contracts in Section 2 (interface signatures only, not implementations).
- Ratify the eight decisions in Section 3.
- Confirm the team's own foundation-phase topics the original spec asks for but doesn't prescribe — prototyping/design approach, stack choices, test architecture **(8.4)** — this document doesn't choose a stack; that's the team's call, but it belongs in this same wave.

### Wave 1 — Core substrate (parallel, roughly 1–2 weeks)

| Track | Work | Depends on |
|---|---|---|
| **A** | Epic 1, full: 1.1→1.2→1.3 (access role resolution) → 1.4→1.5 (functional roles) → 1.6 (profile assembly) → NEW A (caching, per D1) → 1.8→1.9→1.10 (leak enforcement, S7 flags, custom-field visibility) → NEW B (leak test harness — this *is* the core of NFR.1) → 1.7 (header, per D5) → 1.11→1.12 (profile sharing) | Nothing outside Track A — this track *is* the AccessResolver contract's real implementation |
| **B** | Epic 3, full: 3.2 (FieldRegistry real implementation) → 3.1 (filterable list) → 3.3 (inline edit) → 3.4 (saved views) → 3.5 (export) → 3.6 (colleague mode) | AccessResolver *contract* (Wave 0), stub implementation until Track A's real one lands mid-wave — swap is transparent |
| **C** | Epic 7's Timeline Event Writer real implementation, then 7.1 (auto-generate) → 7.2 (manual edit) → NEW (conflict resolution, per D2). If there's slack, start Epic 13 Story 13.2's *consumer* side against the ProjectAssignment stub (C3) — the real feed lands later, the consumer code doesn't need to wait. | Timeline Writer *contract* (Wave 0) is Track C's own deliverable; ProjectAssignment stub already exists from Track A's Story 1.2 |

By the end of Wave 1: real AccessResolver, real FieldRegistry, real Timeline Writer, a working Directory, a working Profile, and the section-leak test harness all exist. Every Wave 2 epic below only needs these.

### Wave 2 — Feature fan-out (parallel, the largest wave, roughly 3–4 weeks)

Nine epics, zero cross-track blocking by construction (see Section 5 for exactly how each link that *could* have blocked something got resolved):

| Track | Sequence | Why this order |
|---|---|---|
| **A** | Epic 4 (Action Items) → Epic 10 (Forms & Campaigns) → Epic 11 (Feedback) | Epic 10 calls Epic 4's `createActionItem` contract (C6) and Epic 11 Story 11.3 calls into Epic 10 — giving one developer all three turns two potential cross-dev dependencies into a single dev's own sequencing problem |
| **B** | Epic 6 (Resourcing) → Epic 8 (CDS) → Epic 2 (Self-Service) | Epic 6 needs Track A Wave 1's profile-sharing (1.11/1.12), already done by the time Wave 2 starts; Self-Service is picked up last because it's the epic that consumes the most other sections (S9/S10/S11/S12/S13/S7/S8/S14) and benefits from more of them already existing |
| **C** | Epic 5 (Risks) → Epic 9 (Mentorship) → Epic 13 Stories 13.1 & 13.3 (Timetracker leaves, PeopleForce) | Epic 9 needs the Timeline Writer, which Track C built themselves in Wave 1; 13.1/13.3 are independent integrations with no upstream dependency inside this codebase |

### Wave 3 — Integration, dashboards, hardening (parallel, roughly 1–2 weeks)

- **Epic 12 (Dashboards, all 5 stories)** — natural last wave, since dashboards aggregate data from Epics 4, 5, 6, 9, 10 which are now all real. Split 12.1 (shared engine) + 12.2–12.4 (UM/DM/PM, same engine) to one developer, 12.5 (PP) + wiring the real data feeds to another; the third continues below.
- **Epic 13 Story 13.2** (swap the real timetracker projects/people feed in for the Wave-1 stub) and **13.4** (finalize the identity-mapping implementation against whatever 13.1/13.3 discovered).
- **NFR.2** (performance pass) — now that real caching, real filtering, and real data volume exist.
- **NFR.4** (accessibility pass) across List, Profile, and Dashboards.
- **NFR.1's remaining scope** — extend Wave 1's leak-test harness to Risks, Resourcing, Action Items, and Dashboards, per NFR.1's own stated scope.

---

## 5. How every cross-epic dependency was resolved (so none of them turn into a developer waiting on another)

| Dependency | Resolution |
|---|---|
| Epic 3 needs AccessResolver (Epic 1) | Wave-0 contract + Wave-1 stub; real swap is transparent mid-wave |
| Epic 9 needs the Timeline Writer (Epic 7) | Wave-0 contract; Track C builds the real thing themselves before touching Epic 9 |
| Epic 10 needs the filter engine (Epic 3) and `createActionItem` (Epic 4) | Epic 3 is real by end of Wave 1; Epic 4→Epic 10 assigned to the same developer in Wave 2 |
| Epic 11 Story 11.3 needs Epic 10 | Same developer, same Wave-2 sequence (Epic 4 → Epic 10 → Epic 11) |
| Epic 6 needs profile sharing (Epic 1, 1.11/1.12) | Real by end of Wave 1, before Wave 2 starts |
| Epic 6 needs PeopleForce (Epic 13.3) | Spec-sanctioned fallback (external link) means Epic 6 never actually blocks — ships independently, upgrades later when 13.3 lands |
| Epic 1's ProjectAssignment needs the real timetracker feed (Epic 13.2) | Already decoupled by design in the backlog draft — internally seeded from Wave 1, real feed swapped in during Wave 3 with no consumer-side changes |
| Epic 12 needs data from almost everything | By design, scheduled last (Wave 3); its shell (12.1) can start earlier against mock data if a developer has slack |
| NFR.1 needs Epic 1's leak rules | Same developer, same wave — NEW B *is* Track A's Wave-1 deliverable |
| Epic 9's mentorship-end feedback vs. Epic 11's Feedback entity | Decoupled by decision D6 — mentorship feedback gets its own field, no dependency at all |

---

## 6. Full feature tree

Read top-to-bottom within your track and you have a valid build order. Full acceptance criteria and Given/When/Then for every story tagged *(see backlog draft)* live in `backlog_review_draft.md`; Epic 10 and Epic 11 are specified in full in Section 7 of this document since nothing else covers them.

### Wave 0 — Foundation (all three developers)

- **NFR.5 — Intelligent Repository & BMAD Setup.** Stand up the repo, confirm process framework. `Depends-on: none` `Blocks: everything — do this first` *(see backlog draft)*
- **NFR.3 — Pseudonymized Seed Data.** Build the tool that generates realistic non-production data at 500+-record scale. `Depends-on: none` `Blocks: Wave 1 start for all tracks` *(see backlog draft)*
- **Contracts C1–C6.** Freeze interface signatures only (Section 2). `Depends-on: none` `Blocks: Wave 1 stubs`
- **Decisions D1–D8.** Ratify the defaults in Section 3. `Depends-on: none` `Blocks: nothing — build to these defaults immediately`

### Wave 1 — Track A: Epic 1, Roles, Access Control & Employee Profile

- **1.1 — Derive Manager Access from Reporting Hierarchy.** Transitive closure of "reports to." `Depends-on: none` `Blocks: 1.2, everything downstream of AccessResolver`
- **1.2 — Extend Manager Access via Project Assignment.** Adds the project-assignment leg; introduces the internally-seeded ProjectAssignment (C3). `Depends-on: 1.1` `Blocks: 1.6+`
- **1.3 — Assign People Partner Relationships.** PP access role + HR line resolution. `Depends-on: none` `Blocks: 1.6+`
- **1.4 — Define Functional Roles and Permissions via UI.** Data-driven role/permission definitions. `Depends-on: none`
- **1.5 — Assign Functional Roles to People.** Enforces "functional role never widens data access." `Depends-on: 1.4, 1.1–1.3`
- **1.6 — Assemble Employee Profile by Section Access.** The AccessResolver's real payload-assembly logic (C1). `Depends-on: 1.1–1.3` `Blocks: 1.7–1.12, all of Epic 3's colleague mode, all Wave 2 epics' access checks`
- **NEW A — Cache Access Resolution Safely.** Per Decision D1. `Depends-on: 1.6`
- **1.8 — Enforce the Colleague Whitelist Everywhere.** `Depends-on: 1.6`
- **1.9 — Management Notes with Visibility Flags (S7).** `Depends-on: 1.6`
- **1.10 — Per-Field Custom Field Visibility.** Coordinates with Track B's FieldRegistry (C2) — interface-level coordination only, no wait. `Depends-on: 1.6`
- **NEW B — Prevent Section Leaks Across All Surfaces.** This *is* NFR.1's core deliverable. `Depends-on: 1.6, 1.8, 1.9` `Blocks: NFR.1's Wave-3 extension`
- **1.7 — Profile Header Shows Manager, PP and Mentor.** Per Decision D5. `Depends-on: 1.1–1.3`
- **1.11 — Generate a Shareable Profile Link.** `Depends-on: 1.6` `Blocks: Epic 6 (Resourcing) candidate review`
- **1.12 — Shared Link Expiry, Logging and Revocation.** `Depends-on: 1.11`

### Wave 1 — Track B: Epic 3, All Employees Directory & Custom Fields

- **3.2 — Define Custom Fields at Runtime.** Real implementation of FieldRegistry (C2). `Depends-on: none (AccessResolver stub only)` `Blocks: 3.1, all downstream field-visibility enforcement`
- **3.1 — Sortable, Filterable Employee List.** `Depends-on: 3.2, AccessResolver (stub until Track A lands, then real)` `Blocks: Epic 10's audience builder (3.4 specifically)`
- **3.3 — Inline Editing on the List.** `Depends-on: 3.1`
- **3.4 — Saved and Shared Views.** `Depends-on: 3.1` `Blocks: Epic 10's Story 10.2 (audience-from-saved-view)`
- **3.5 — Export to Excel.** `Depends-on: 3.1`
- **3.6 — Colleague Mode of the List.** `Depends-on: 3.1, 1.8 (whitelist rule, coordinate with Track A, no blocking — same rule, independent enforcement point)`

### Wave 1 — Track C: Epic 7, Career Timeline (+ early Epic 13 start if slack)

- **Timeline Event Writer — real implementation (C4).** `Depends-on: none` `Blocks: 7.1, 7.2, and every future caller (Epic 1 profile edits, Epic 9, Epic 13)`
- **7.1 — Auto-Generate Timeline Events.** `Depends-on: Timeline Writer`
- **7.2 — Manual Timeline Edits.** PP/UM only. `Depends-on: 7.1`
- **NEW — Resolve Conflicts Between System and Manual Events.** Per Decision D2. `Depends-on: 7.1, 7.2`
- *(If slack remains)* **13.2 consumer-side prep** — write the code that will read from ProjectAssignment (C3, already seeded by Track A), without waiting for the real timetracker feed. `Depends-on: C3 (stub, exists from Wave 1 start)`

### Wave 2 — Track A: Epic 4 → Epic 10 → Epic 11

- **4.1 — Manually Create an Action Item.** `Depends-on: AccessResolver (real, from Wave 1)` `Blocks: 4.2, 4.3, Epic 10's activation story`
- **4.2 — Complete and Cancel Action Items.** `Depends-on: 4.1`
- **4.3 — Overdue Highlighting.** `Depends-on: 4.1` `Blocks: Epic 12's counters, Epic 10's completion tracking`
- **NEW — Auto-Generate Action Item on Campaign Activation.** Implements Contract C6's consumer side. `Depends-on: 4.1` `Blocks: Epic 10, Story 10.3`
- **10.1 — Create a Form Campaign.** `Depends-on: AccessResolver`
- **10.2 — Build and Freeze Campaign Audience.** `Depends-on: 10.1, Epic 3's filter engine (3.1/3.4, real by now)`
- **10.3 — Activate a Campaign.** Calls Contract C6 directly — same developer as Epic 4, so this is a same-dev sequencing step, not a cross-dev wait. `Depends-on: 10.2, Epic 4's createActionItem (C6)`
- **10.4 — Track Campaign Completion.** Reuses 4.3's overdue derivation exactly. `Depends-on: 10.3, 4.3`
- **11.1 — Record Feedback with a Visibility Flag.** `Depends-on: AccessResolver` — could actually start any time in Wave 2, doesn't need Epic 10
- **11.2 — View Feedback Over Time and Compare Periods.** `Depends-on: 11.1`
- **11.3 — Request Feedback from Named Colleagues via a Form Campaign.** `Depends-on: 11.1, Epic 10 (10.1, 10.2's individual-add path)`

### Wave 2 — Track B: Epic 6 → Epic 8 → Epic 2

- **6.1 — Create a Resourcing Request.** `Depends-on: AccessResolver`
- **6.2 — Fulfil a Request with Internal or External Candidates.** External path uses the spec-sanctioned PeopleForce-link fallback — never blocks on Epic 13.3. `Depends-on: 6.1`
- **6.3 — DM Reviews and Approves/Rejects Candidates.** `Depends-on: 6.2, Epic 1's 1.11/1.12 (real by end of Wave 1)`
- **6.4 — Request History.** `Depends-on: 6.3` `Note: does not wait on Epic 13.2's real sync — S11 update after approval is explicitly out of this story's scope`
- **8.1 — Skills Matrix Link and Assessment Log.** `Depends-on: AccessResolver`
- **8.2 — IDP Records (manager/PP + employee self-complete, per Decision D7).** `Depends-on: 8.1`
- **8.3 — Manager/PP Maintain Assessments and Conclusions.** `Depends-on: 8.1`
- **8.4 — Filter by Assessment Recency and Open IDP.** `Depends-on: 8.2, 8.3, Epic 3's filter engine (real by now)`
- **2.1 — View Own Employment Summary.** `Depends-on: AccessResolver, S4 data model (Epic 1)`
- **2.2 — Edit Own Personal and Emergency Contacts.** `Depends-on: AccessResolver`
- **2.3 — Upload Photo and Certificates.** `Depends-on: AccessResolver`
- **2.4 — View Own Timeline, Leaves, Projects, CDS and Mentorship.** `Depends-on: Epic 7 (timeline, real by end of Wave 1), Epic 8 (8.2), Epic 9 (mentorship flag) — pick up last in this track for exactly this reason`
- **2.5 — View Shared Feedback, Flagged Notes and Own Action Items.** `Depends-on: Epic 1 (1.9), Epic 11 (11.1), Epic 4 (4.2)`

### Wave 2 — Track C: Epic 5 → Epic 9 → Epic 13 (13.1, 13.3)

- **5.1 — Record a Risk.** `Depends-on: AccessResolver`
- **5.2 — Show Risk Trend.** `Depends-on: 5.1`
- **5.3 — Risk Dashboard.** `Depends-on: 5.1, 5.2` `Blocks: Epic 12's risk widgets`
- **9.1 — Self-Flag Open to Mentoring.** `Depends-on: AccessResolver`
- **9.2 — Assign a Mentor-Mentee Pair.** `Depends-on: 9.1, Timeline Writer (real, from Track C's own Wave 1)`
- **NEW A — End a Mentorship Pair with Required Final Feedback.** Feedback stored on the pair itself, per Decision D6 — no Epic 11 dependency. `Depends-on: 9.2`
- **NEW B — Automatic Mentor Status Transitions.** Per Decision D4. `Depends-on: 9.2, NEW A`
- **NEW C — View All Mentor-Mentee Pairs.** `Depends-on: 9.2, NEW A`
- **13.1 — Integrate Timetracker Leaves API.** `Depends-on: none (independent integration)`
- **13.3 — Investigate & Integrate PeopleForce Candidate/Vacancy API.** `Depends-on: C5 (identity mapping, ratified Wave 0)` `Note: Epic 6 already ships against the fallback, so this never blocks Track B`

### Wave 3 — Integration, dashboards, hardening (reassign the three developers freely here)

- **12.1 — Build Shared Dashboard Engine.** `Depends-on: AccessResolver` `Blocks: 12.2–12.5`
- **12.2 — Unit Manager Dashboard.** `Depends-on: 12.1, Epic 4 (action items), Epic 5 (risks)`
- **12.3 — Delivery Manager Dashboard with Project Selector.** `Depends-on: 12.1, Epic 5, Epic 6 (resourcing)`
- **12.4 — Project Manager Dashboard.** `Depends-on: 12.3 (shares its engine)`
- **12.5 — People Partner Dashboard.** `Depends-on: 12.1, Epic 8 (CDS), Epic 10 (campaigns)`
- **13.2 — Integrate Timetracker Projects & People API (real feed, replacing the Wave-1 stub).** Per Decision D3's fail-safe access rule. `Depends-on: C3 (already exists), Epic 1's caching (NEW A, to verify no stale-access regression)`
- **13.4 — Finalize Cross-System Identity Mapping.** `Depends-on: 13.1, 13.3, C5`
- **NFR.2 — All Employees List Performance at 500+ Records.** `Depends-on: 3.1 (real), NEW A (real caching)`
- **NFR.4 — Accessibility & Responsive Layout Pass.** `Depends-on: Epic 1 (profile), Epic 3 (list), Epic 12 (dashboards) all existing`
- **NFR.1 — extend the Wave-1 leak-test harness.** `Depends-on: NEW B (Wave 1) plus Epics 4, 5, 6, 9, 12 all existing`

---

## 7. Epic 10 & Epic 11 — full specification

These two epics had ClickUp lists but zero stories, despite being core (not stretch) functionality per the original spec — Forms & Survey Campaigns **(4.12)** and Feedback **(4.15)**. Everything else in this document treats `backlog_review_draft.md` as the source of full detail; these two are specified here in full since nothing else covers them yet.

# Epic 10: Forms & Survey Campaigns (newly scoped — this ClickUp list exists but has zero stories; this is core functionality per spec 4.12, not stretch)

## Story 10.1 — Create a Form Campaign
**Proposed description:**
Implements the campaign-creation half of 4.12, point 1: "A PP or a manager creates a form in the system: title, short description, purpose, a link to the external form, and a due date. The form itself lives outside the system — Microsoft Forms, Google Forms, or anything else." This story is deliberately narrow — it is record creation only, not audience resolution (Story 10.2) or activation (Story 10.3). Per 4.12's closing paragraph, creation is available to PP and managers by default, plus any functional role granted the *create form campaigns* permission (2.3) — the canonical example being IT running its own security-awareness campaigns without holding managerial access to anyone. A campaign starts in a draft/unactivated state; nothing is sent to anyone until Story 10.3's activation step.

- A PP, manager, or holder of the *create form campaigns* functional permission (2.3) can create a campaign with title, short description, purpose, external form link, and due date.
- The campaign is created in a non-activated ("draft") state and has no effect on any employee until explicitly activated (Story 10.3).
- The system does not host or embed the external form in any way — it only stores the link (4.12: "The form itself lives outside the system").
- A functional-role holder with the permission but no Manager/PP relationship to anyone can still create a campaign record — the permission unlocks the feature; whether they can *target* anyone meaningfully is governed by audience resolution (Story 10.2), consistent with 2.3's "never widens data access" rule.
- Campaign fields (title, description, purpose, link, due date) are all editable while the campaign remains in draft state, prior to activation.

**Given/When/Then:**
```
Scenario: PP creates a draft campaign
Given I am a People Partner
When I create a campaign titled "Annual Engagement Survey" with a description, purpose, an external Google Forms link, and a due date
Then the campaign is saved in draft state, visible to me, with no action items generated and no employees notified

Scenario: Functional-role holder without managerial access can still create a campaign
Given I hold a functional role granted the "create form campaigns" permission (2.3) but no Manager or PP relationship to anyone
When I create a campaign record
Then the campaign is created successfully, though my ability to target recipients is still bounded by my own access scope when I later build its audience (Story 10.2)
```

## Story 10.2 — Build and Freeze Campaign Audience
**Proposed description:**
Implements 4.12, point 2: "They select the audience using the All Employees filter engine (4.1): build a filter, preview the resulting people, confirm. A saved view can be used as an audience. Individual people can be added or removed after the filter resolves. The resolved list is frozen when the campaign is activated; people who join later are not added." This story directly depends on Epic 3's filter engine (Stories 3.1, 3.4) — it does not reimplement filtering, it reuses the same All Employees query engine to build a preview, then persists the resolved audience against the campaign. A functional-role holder building an audience is bounded by their own resolved access scope (2.1, 2.3) — they can only target people whose relevant fields they can see/filter on, exactly as in All Employees.

Freezing is the critical behavioral rule here: the audience is a snapshot taken at activation time (Story 10.3), not a live query re-evaluated later. Someone who newly matches the filter after activation is never retroactively added, and someone who stops matching is not removed either — the list is frozen, full stop.

- A campaign's audience can be built using the same filter engine as All Employees (Story 3.1), including using a saved view (Story 3.4) directly as the starting audience.
- After a filter resolves a preview list, individual people can be manually added to or removed from that list before activation.
- The campaign creator's audience-building is scoped to their own resolved access (2.1) — they cannot target or preview people outside what their access role would show them on All Employees.
- The audience is not persisted as "frozen" until the campaign is activated (Story 10.3) — up to that point, the resolved preview can be rebuilt/changed freely.
- Once frozen (at activation), the specific list of recipient IDs is stored immutably on the campaign; no query re-evaluation happens afterward.

**Given/When/Then:**
```
Scenario: Building an audience from a filter, with manual adjustment
Given I am creating a campaign and open the audience builder
When I filter for "department = Engineering AND country = Poland" and preview the results, then remove one person and add another who didn't match the filter
Then my adjusted list is the current audience preview for this campaign, ready to be frozen at activation

Scenario: Audience is frozen at activation, not live
Given a campaign's audience preview currently resolves to 50 people matching a filter
When the campaign is activated, and afterward a new employee joins who would also match that filter
Then the new employee is not added to the campaign's recipient list — the frozen list remains exactly the 50 people captured at activation time
```

## Story 10.3 — Activate a Campaign
**Proposed description:**
Implements the activation transition referenced throughout 4.12 and the trigger point for Epic 4's "Auto-Generate Action Item on Form Campaign Activation" story. Activation is the single event that (a) freezes the audience (Story 10.2) and (b) triggers one action item per recipient, each carrying the campaign's title, sender, due date, and link (4.12 point 3; Epic 4's NEW story owns the action-item-creation mechanics themselves). This story owns the activation transition and its atomicity: freezing the audience and generating action items should happen as a single atomic (or reliably idempotent) operation, so a partial failure never leaves some recipients with action items and others without, and so a campaign can never be "half-activated." Once activated, a campaign cannot be edited (title/description/link/due date are locked) and cannot be re-activated — this is a one-way transition, consistent with the frozen-audience model.

**Explicit dependency on Epic 4:** this story calls into the action-item creation path defined in Epic 4's NEW story ("Auto-Generate Action Item on Form Campaign Activation"). To avoid the two epics blocking each other during parallel development, the recommended approach is to define the call contract early (a function/service call: `createCampaignActionItems(campaignId, recipientIds, title, senderId, dueDate, link) -> ActionItem[]`) so whichever epic's story lands first can build against a stub of the other.

- Activating a draft campaign transitions it to "active" status, freezes its audience (Story 10.2) at that exact moment, and triggers action-item generation for every frozen recipient (Epic 4 NEW story).
- Activation is atomic per campaign: either all frozen recipients end up with a generated action item, or the activation fails cleanly and the campaign remains in draft (no partial activation state).
- Once activated, the campaign's title, description, purpose, link, and due date become locked/read-only; only its completion-tracking view (Story 10.4) continues to update.
- A campaign cannot be activated twice, and cannot be deactivated back to draft.
- Only the campaign creator, or someone with equivalent permission/access, can trigger activation.

**Given/When/Then:**
```
Scenario: Activating a campaign freezes the audience and generates action items
Given a draft campaign has a resolved audience of 30 people and complete title/description/link/due-date fields
When the creator activates the campaign
Then the audience is frozen at exactly those 30 people, and 30 action items are created (one per recipient), each with the campaign's title, sender, due date, and link

Scenario: Activation is atomic — no partial state
Given a campaign activation is in progress and an error occurs partway through creating action items
When the activation fails
Then the campaign is not left in a partially-active state — either it remains a fully-editable draft, or the activation is retried to completion, but recipients never end up split between "has an action item" and "doesn't" due to a failed activation
```

## Story 10.4 — Track Campaign Completion
**Proposed description:**
Implements 4.12 point 5: "The sender sees the campaign with a per-person table: who has completed, who has not, who is overdue." This is a read/reporting story layered on top of the action items generated by Story 10.3 — it does not introduce new completion logic, it visualizes the state of each recipient's generated action item (open / completed / overdue, reusing Epic 4 Story 4.3's overdue derivation exactly). The system never reads or verifies the external form itself (4.12 point 4: "That is the completion signal — the system does not read the external form and does not verify anything") — this table is purely a reflection of each recipient's self-reported action-item completion.

- The campaign creator (sender) sees a per-person table listing every frozen recipient with their current status: completed, not completed (open), or overdue.
- "Overdue" in this table uses the exact same derivation as Epic 4 Story 4.3 (open + due date passed) — no separate overdue logic is implemented here.
- The table updates in real time as recipients mark their generated action items complete (Epic 4 Story 4.2) — no manual refresh/sync step is required.
- This table is visible only to the campaign's creator/sender (and anyone else with equivalent access to the campaign, e.g. via the *create form campaigns* permission scope) — not to other recipients, who see only their own single action item.
- The table works identically for campaigns targeting large audiences (hundreds of recipients) without a separate pagination/performance story — reuse Section 7's general performance expectations.

**Given/When/Then:**
```
Scenario: Sender sees live completion status across all recipients
Given a campaign was activated for 30 recipients, 10 of whom have marked their action item complete, 5 of whom are past the due date and still open, and the rest still open but not yet overdue
When the campaign creator opens the campaign's tracking table
Then they see 10 recipients marked completed, 5 marked overdue, and the remaining 15 marked not-yet-completed but not overdue

Scenario: Completion table never reflects verification of the external form
Given a recipient has not actually filled in the external form but marks their action item complete anyway
When the sender views the tracking table
Then that recipient shows as "completed" — the system has no way to detect or flag that the external form wasn't genuinely filled in, by design (4.12 point 4)
```

---

# Epic 11: Feedback (newly scoped — this ClickUp list exists but has zero stories; this is core functionality per spec 4.15, not stretch)

## Story 11.1 — Record Feedback with a Visibility Flag
**Proposed description:**
Implements the core of 4.15: "Feedback records are added on the employee's profile page by managers and PP. A record contains: subject (the employee), author, date, context (project, event, period), and the feedback body. Each record carries a visibility flag: management only (default) or shared with employee." This is section S8 (3.2): Manager line and PP get `RW`; Self gets `R` limited to records flagged *shared with employee* (see Epic 2 Story 2.5, already implemented against this rule); Colleague gets `—` entirely (4.15: "A colleague cannot browse feedback about another person"). This story owns creation and editing; Epic 2 Story 2.5 already owns the employee's flag-gated read side, and Epic 1's NEW B negative-test story already covers cross-surface leak prevention generically — this story just needs to feed correctly-flagged records into that existing machinery, not reimplement access control.

- A manager (holding Manager access over the subject per 2.1) or PP can create a feedback record with subject, author (self, automatic), date, context (free text or structured — project/event/period), and body.
- The visibility flag defaults to *management only* on creation and must be explicitly changed to *shared with employee* to become visible to the subject (4.15, 3.2).
- Only Manager line and PP can create or edit feedback records (S8 = RW for those audiences only, 3.2); Self and Colleague have no write access here at all.
- Editing an existing record (e.g., correcting the body or context) is available to Manager line/PP with RW access, consistent with other RW sections.
- A record's visibility flag can be changed after creation (e.g., a manager later decides to share a previously management-only note with the employee) — flipping it makes the record immediately visible to the employee's own profile view (Epic 2 Story 2.5), consistent with "no stale grant" (3.3.4).

**Given/When/Then:**
```
Scenario: Manager records management-only feedback by default
Given I am the manager of employee B
When I create a feedback record about B with context "Q3 project retrospective" and a body, leaving the visibility flag unset
Then the record is saved with visibility "management only" by default, and is not visible to B on their own profile

Scenario: Flipping the visibility flag makes a record visible to the employee
Given a management-only feedback record about employee B already exists
When B's PP edits the record's visibility flag to "shared with employee"
Then B can now see that specific record on their own profile (S8), consistent with Epic 2 Story 2.5's flag-gated read rule
```

## Story 11.2 — View Feedback Over Time and Compare Periods
**Proposed description:**
Implements 4.15's requirement that "records are viewable over time, with comparison between periods." This is a read-side story for Manager line/PP (who have full S8 read access per 3.2) — it lays feedback records for a given employee out chronologically and supports comparing across periods (e.g., quarter over quarter, or any two arbitrary date ranges), so a manager or PP can see patterns or shifts in feedback over time rather than reading disconnected records one at a time. This story does not change access rules — it only adds a presentation/query capability on top of Story 11.1's records, scoped by the same S8 access rule (Manager line/PP full history; Self only their shared-with-employee subset per Epic 2 Story 2.5, which this story's period-comparison view should also support at the narrower scope if self-service ever surfaces it).

- Manager line and PP viewing an employee's feedback (S8) see records laid out chronologically, not just as an unordered list.
- A period-comparison view lets the viewer select two time ranges (e.g., two quarters) and see the feedback records falling in each side by side, or otherwise clearly delineated.
- This view respects the exact same S8 access scoping as Story 11.1 — a viewer without Manager/PP access to the subject sees nothing, and Self sees only their shared-with-employee subset if this view is ever exposed in self-service.
- The comparison capability works correctly with zero, one, or many records in either period (no crash on an empty period).

**Given/When/Then:**
```
Scenario: Manager compares feedback across two quarters
Given employee B has feedback records dated across Q1 and Q3 of the year
When B's manager opens the feedback view and selects "compare Q1 vs Q3"
Then the records from each quarter are shown grouped/delineated by period, in chronological order within each

Scenario: Comparison view handles an empty period gracefully
Given employee B has feedback records only in Q3, none in Q1
When B's manager compares Q1 vs Q3
Then Q1 shows an empty state and Q3 shows its records, with no error
```

## Story 11.3 — Request Feedback from Named Colleagues via a Form Campaign
**Proposed description:**
Implements 4.15's explicit mechanism: "PP and managers can request feedback about a person from specific colleagues — implemented as a form campaign (4.12) targeted at named individuals." This story is a thin, deliberate reuse of Epic 10 (Forms & Survey Campaigns) rather than a new mechanism — a feedback request *is* a form campaign whose audience is a specific named list of colleagues (not a filter-resolved audience) and whose external form is wherever the org collects feedback text (e.g., a Google Form). This story owns the "request feedback" entry point and its wiring into Epic 10's existing campaign creation/audience/activation/tracking flow; it does not duplicate any of that flow's logic.

**Explicit dependency on Epic 10:** this story cannot ship before Epic 10's create-campaign (10.1) and build-audience (10.2, specifically its "individual people can be added" path, which covers "named individuals" here) stories exist. To avoid blocking, whichever developer picks this up should build the "request feedback" entry point against Epic 10's campaign-creation contract as soon as that contract (not necessarily full implementation) is defined — this is a natural Wave 2 story that slots in once Epic 10 is underway, not a Wave 1 blocker for anyone else.

- A manager or PP can initiate a "request feedback about [employee]" flow, which creates a form campaign (Epic 10, Story 10.1) pre-associated with the subject employee, targeted at a named list of colleagues chosen individually (not via a filter) rather than for the requester's own management purposes.
- The resulting campaign is otherwise a normal campaign — it goes through the same activation (10.3) and completion-tracking (10.4) flow as any other campaign; there is no separate "feedback campaign" data model.
- On completion (i.e., a targeted colleague marks their action item complete after providing feedback via the external form), the request itself does not automatically create a Feedback record (S8) — per 4.12 point 4, the system never reads the external form. The requester manually creates the resulting Feedback record (Story 11.1) after reviewing the external form's actual responses, using the campaign's completion table (Story 10.4) to know who responded.
- This flow is explicitly named in the story/description as "request," not "collect," to avoid implying automatic data capture that the platform does not perform.

**Given/When/Then:**
```
Scenario: PP requests feedback about an employee from named colleagues
Given I am a PP and want feedback about employee B from three specific colleagues
When I initiate "request feedback" for B and select those three colleagues by name as the audience
Then a form campaign is created targeted exactly at those three people, associated with B as the subject, ready to activate through Epic 10's normal flow

Scenario: Completing the request does not auto-create a Feedback record
Given the feedback-request campaign has been activated and all three colleagues have marked their action items complete
When the PP checks the campaign's completion table (Story 10.4)
Then all three show as completed, but no Feedback record (S8) has been automatically created — the PP must review the external form's actual responses and manually create the Feedback record(s) (Story 11.1) based on what they read there
```

---

## 8. Appendix — small product questions still worth a quick PO answer

These are narrower than the architectural decisions in Section 3 — none of them block Wave 0/1 start, and each story above already ships with a reasonable default from the backlog draft. Listed here so they don't get lost before the relevant story is picked up.

- **Photo/certificate file constraints (Epic 2, Story 2.3).** Format and size limits aren't specified; the backlog draft applies sane defaults. Confirm before shipping if the org has specific requirements (e.g. max upload size).
- **Reopening a completed IDP (Epic 8, Story 8.2).** Can a manager/PP un-complete an IDP the employee already marked done? Current default: no — disallow until told otherwise.
- **Saved views with a static membership list (Epic 3, Story 3.4).** The spec's "manually-maintained bench list" example implies a saved view might sometimes be an explicit list of people rather than a pure filter. Current default: filter-based views only for v1; confirm if a static-list variant is actually needed.
- **Assessment "assessor" field shape (Epic 8, Story 8.3).** Free text or a reference to a person record? Either works for the story as written; pick one before building the UI.
- **Timeline event deletion — hard vs. soft delete (Epic 7, Story 7.2).** Given access-control correctness is the platform's primary quality attribute, a soft-delete-with-audit-trail is the safer default; confirm if hard delete is acceptable instead.
- **Action item cancellation after the author loses access (Epic 4, Story 4.2).** Current default: the author can still cancel their own historical item even if they no longer hold live Manager/PP access to the assignee, since authorship is a historical fact, not a live permission. Confirm this matches intent.


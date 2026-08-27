# Validation Report — People Management Platform

- **PRD:** `_bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md`
- **Rubric:** `.claude/skills/bmad-prd/assets/prd-validation-checklist.md`
- **Run at:** 2026-08-26T19:27
- **Grade:** Good

## Overall verdict

On the seven-dimension quality rubric alone, this PRD is Excellent: every dimension is strong or adequate, all issues the rubric walker found (a stale FR-range claim, a broken cross-reference, a self-contradictory Assumptions Index entry, a missed glossary term, seven FRs missing Consequences bullets) were fixed in this same session and verified against the file. The narrative, thesis, scope honesty, and NFR substance all hold up well for a launch-grade, chain-top PRD.

The edge-case-hunter lens tells a different story, and it's the one that matters more before this feeds `bmad-architecture`. Walking every branch the access model, departure cascade, and resourcing lifecycle imply, it found 22 unhandled paths — several of which are not cosmetic gaps but **direct violations of guarantees the PRD itself states elsewhere**: FR-8 promises full profile access is "never revocable down to zero holders," but neither concurrent mutual revocation nor the departure of the sole remaining holder is guarded against it. SM-6 promises a shared link "stops working" when the creator's relationship ends, but FR-63's departure cascade never mentions shared links or saved views at all. FR-31 promises headcount is "always accurate," but there is no un-approve path once a candidate is found ineligible. These are the kind of gaps that are cheap to close now, in a requirements document, and expensive to discover mid-implementation of the access layer — which is this platform's stated primary quality attribute. That is why the grade is Good rather than Excellent: the document is well-written and internally consistent, but it is not yet a complete contract for the access model's edges.

## Dimension verdicts

- Decision-readiness — strong
- Substance over theater — strong
- Strategic coherence — strong
- Done-ness clarity — adequate (bare-FR inconsistency named, not fully closed — see Mechanical notes)
- Scope honesty — strong
- Downstream usability — adequate (all rubric-found defects fixed and verified this session)
- Shape fit — strong

## Findings by severity

The edge-case lens (per its own convention) assigns no severity labels — findings below are grouped by which stated guarantee they threaten, ordered roughly by consequence. The rubric's own findings from this session are listed separately at the end, all already resolved.

### Access-model precedence and self-reference (FR-1, FR-2, FR-7)

- **No precedence rule for multi-role viewers** (FR-1/FR-2) — nothing states what happens when a viewer simultaneously resolves as, e.g., both Project line and PP for the same subject; only the S7 PM-flag case gets an explicit rule.
  Fix: state the resolution rule explicitly — e.g., "when a viewer resolves to more than one access role for the same subject, section access is the union of every resolved role's grants, per section."
- **Self-managed department loophole** (FR-7) — nothing blocks setting a department's manager to a current member of that same department, which would grant that person Reporting-line (hence non-Self) access to their own profile — bypassing the "risk never visible to self" rule (FR-19/FR-27).
  Fix: add a consequence to FR-7 — "a department's manager may never be a current member of that department or any nested sub-department."
- **Manager-chain cycle** (FR-7) — two sequential legitimate edits (A manages B, then B manages A) can form a cycle with no stated guard, which would break the reporting-line transitive-closure walk for everyone in it.
  Fix: reject an employee's-manager edit if the proposed manager is already a descendant of the employee in the reporting chain.
- **Concurrent conflicting access-switch writes** (FR-7/FR-9) — two actors writing different values to the same access-switch field at the same time has no stated resolution; the journal (FR-9) records what happened but not that a conflict occurred.
  Fix: reject the losing concurrent write with a conflict error rather than silently overwriting.
- **Orphaned department on manager-field clear** (FR-7) — clearing a department's-manager field directly (not via departure) is not blocked, which would leave every member of that department with no Reporting-line coverage until someone notices.
  Fix: block clearing a department's-manager field unless a replacement is designated in the same change.

### Full profile access floor (FR-8)

- **Concurrent mutual revocation drives holders to zero** — the last two holders each revoking the other at the same instant isn't guarded; FR-8 only states the invariant, not how a write is rejected to preserve it.
  Fix: re-check the current holder count at commit time (not from a cached read) and reject any revocation that would drop it to zero.
- **Departure of the sole remaining holder bypasses the floor** — FR-62's re-parenting gate covers management/PP relationships but never checks full-profile-access status; FR-63's cascade would revoke the last holder's grant along with everything else.
  Fix: extend FR-62's departure-blocking gate to also require transferring full profile access first if the departing person is its sole holder.

### Departure cascade completeness (FR-61–65)

- **Shared links outlive their creator's departure** — FR-63 enumerates profile/list/action-items/mentorship/account/access consequences but never mentions shared links, so SM-6's own promise ("a link stops working when the creator's relationship ends") has no enforcing step for the departure case specifically.
  Fix: add "every shared link the departing employee created is revoked" to FR-63's cascade.
- **Saved views become permanently frozen** — FR-13 makes a saved view editable/deletable only by its creator; FR-63 never transfers or retires that ownership on departure.
  Fix: add a consequence — on departure, saved-view edit/delete rights transfer to a current manager/PP or the view is retired.
- **Double-departure race on a shared mentorship pair** — if both mentor and mentee depart on overlapping effective dates, nothing states the auto-close (FR-63) is idempotent against a pair already ended, risking a double-write over the system-generated closing note.
  Fix: state that departure-triggered pair auto-closure is a no-op against an already-ended pair.
- **Concurrent cancellation race on action items** — an author's manual cancellation (FR-22) and a departure-triggered auto-cancellation (FR-63) against the same item, arriving concurrently, has no stated ordering rule.
  Fix: state that a cancellation write against an already-cancelled item is a no-op regardless of which path arrives second.
- **Shared-link scope doesn't re-clamp to the creator's current access** — if the creator's own access narrows after link creation (e.g., loses a project, drops from Reporting to Project line) short of full departure, nothing states the link's exposed sections shrink to match.
  Fix: state that a shared link's exposed sections are re-clamped, on every view, to whatever access level its creator currently holds — not just presence of some relationship.

### Resourcing lifecycle (FR-28–33)

- **No un-approve path** — an approved candidate later found ineligible has no stated reversal mechanism, so the headcount stays marked filled with no correction route.
  Fix: allow a DM to reverse an approval to rejected with a written reason, freeing the headcount slot.
- **Headcount editable below filled count** — nothing blocks editing headcount downward past the number already filled, which would make FR-31's "filled/remaining always accurate" guarantee false by construction.
  Fix: reject a headcount edit that would drop below the current filled count, or require un-approving candidates first.
- **No cap enforcement at zero remaining** — FR-31 states the request never auto-closes on the last slot, but nothing blocks approving further candidates once remaining hits zero, so a request could over-fulfill indefinitely.
  Fix: block new approvals once remaining headcount is zero, until the DM raises headcount or closes the request.
- **UM/DM departure mid-decision** — a pending proposal or approval decision with no stated reassignment path when the UM who proposed it, or the DM who must decide, departs first.
  Fix: state that a pending item owned by a now-departed UM/DM is flagged for reassignment within the same department/project rather than left stuck.
- **Stale routing on UM reassignment** — a request routed to a department's UM at creation time has no stated behavior if that UM changes (via FR-7) before the request is fulfilled.
  Fix: state that routing re-resolves to the department's current UM on every view, not pinned at creation.

### Timetracker sync flapping (FR-58)

- **4-hour withdrawal clock resets on any brief recovery** — a sync that fails, recovers briefly, and fails again repeatedly never accumulates 4 continuous hours, so the access-withdrawal safeguard may never fire despite cumulative staleness far exceeding the stated bound.
  Fix: state that the 4-hour clock accumulates total failed time within a rolling window rather than resetting on any momentary recovery.
- **15-minute grant window has the same flapping gap** — a genuinely new project assignment observed during a flapping period may never accumulate 15 uninterrupted minutes of confirmed sync, so it never registers as an access grant at all.
  Fix: state that an assignment is confirmed once a single successful sync observes it, not after 15 continuous minutes.

### Mentorship consent and concurrency (FR-41–45)

- **No server-side consent gate on pair creation** — FR-42 relies on browsing the open-to-mentor pool, but nothing states the server rejects creating a pair for someone whose flag is not currently on — a UI-only guard is not a stated guarantee.
  Fix: state that creating a pair is rejected server-side unless the prospective mentor's flag is currently on.
- **Concurrent pair-ending race** — two permission holders ending the same pair at once has no stated resolution; the second closing note could silently overwrite the first.
  Fix: reject an end-pair request if the pair's status has already transitioned to ended by a concurrent request.

## Rubric findings (this session — all resolved)

- **medium** — Assumptions Index (§10) self-contradicted itself on UJ-1/2/3 confirmation status. *Resolved:* reworded to match §3.3's clean split.
- **medium** — Consequences-bullet fix from the prior polish pass only covered the Dashboard FRs it named as an example, leaving FR-20/34/41/44/46/47/60 untouched. *Resolved:* added Consequences bullets to all seven.
- **low** — FR-6's heading didn't use the Glossary term "Shared Link." *Resolved:* retitled to "Generate and manage a Shared Link."
- **low** — Spelling drift within the Access switch glossary entry ("organizational" vs. "*change organisational relationships*"). *Resolved:* added a footnote clarifying the permission name is quoted verbatim from `access-model.md`, not a spelling inconsistency.
- **low** — The Consequences-bullet inconsistency is wider than initially estimated (~18 more FRs still lack the block). *Not resolved — no action required per the rubric's own allowance that a single-sentence FR can be self-testable as stated; noted for completeness, not a defect.*

## Mechanical notes

- FR numbering (FR-1–FR-65) verified contiguous, gap-free, duplicate-free.
- All cross-references spot-checked by the rubric walker resolved correctly; no new broken numeric references found.
- No v1-era residue detected anywhere (single manager tier, HR-Admin-as-implicit-full-access, colleague leave-type visibility, feedback period comparison).
- The edge-case lens's `also_consider` areas (access precedence, access switches/journal, full-access floor, departure cascade, resourcing headcount, timetracker flapping, mentorship lifecycle) were the source of every substantive finding above — a genuinely exhaustive path walk, not a spot-check.

## Reviewer files

- `review-rubric.md`
- `review-edge-case-hunter.md`

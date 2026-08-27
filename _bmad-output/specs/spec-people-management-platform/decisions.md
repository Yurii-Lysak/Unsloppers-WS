# Decision Log

Companion to `SPEC.md` — see there for the capabilities these decisions bear on. Concrete defaults resolving points where the source spec deliberately doesn't answer its own question. Every default is revisable — the sign-off line says which were worth a PO confirmation and which were safe to just build. Sourced from `PRD_parallel_delivery_plan.md` §3 and §8 (appendix), plus items added after review or after the `project-requirements-v2.md` update. D3 and D5 originally carried as `SPEC.md` Open Questions since the source itself flagged them for confirmation; both were confirmed directly by the user on 2026-08-21 and are folded into `SPEC.md`'s Assumptions. D9 and D10 are cross-cutting decisions the source material never addressed at all — they surfaced and were settled during the `bmad-architecture` coaching session. D11 and D12 are new open questions surfaced by the `project-requirements-v2.md` update (2026-08-26), not yet confirmed. D13–D20 resolve the 22 findings of a `bmad-review` edge-case-hunter lens pass against `project-requirements-v2.md` (2026-08-26) — each closes a concurrency race, a lifecycle gap, or an underspecified precedence rule the source left implicit; none were PO-confirmed individually, all are safe-default engineering resolutions of genuine gaps.

## D1 — Access-resolution caching (CAP-1)

A short-TTL, per-subject access cache keyed by subject ID, tagged with a monotonically increasing generation counter on the relationship graph (reports-to + department management + project assignment + PP assignment). Any write to that graph bumps the counter and invalidates affected cache entries synchronously — correctness comes from the generation bump, not the TTL. TTL exists purely as a performance knob.
*PO sign-off: not needed to start; revisit after performance testing if the approach doesn't hold under load.*

## D2 — Timeline conflict resolution (CAP-7)

A manually-edited timeline event is never silently overwritten by a later system-generated event covering the same change window; the system write is skipped and the skip is logged/auditable. A sync that silently re-breaks a manual correction defeats the feature.
*PO sign-off: not needed; document as an ADR per the spec's own instruction.*

## D3 — Timetracker sync failure behavior and access (CAP-13)

**Fail-safe for access, fail-soft for display, now with concrete numbers (refined 2026-08-26 by `project-requirements-v2.md` §5.1).** A confirmed project assignment takes effect as an access grant within **15 minutes** — the platform's stated access-control freshness window. If the sync fails outright, the platform serves last-known S10/S11 data behind a visible "not fresh" banner, and **withdraws project-derived Manager access after 4 continuous hours** of failed sync, falling back to the relations the platform owns outright (reports-to, department management, PP). This is a bounded, deliberate trade-off — an access grant may survive up to 4 hours past the relationship's real end during a prolonged outage — rather than hard-failing every access decision during any interruption. The original v1 framing ("treat unknown as not confirmed, never still active") is superseded by this explicit numeric policy; the underlying fail-safe/fail-soft split is unchanged.
*PO sign-off: original qualitative behavior confirmed by the user, 2026-08-21; the concrete 15-minute/4-hour numbers come directly from the v2 source document, not a fresh default — no separate sign-off needed unless the numbers themselves are contested.*

## D4 — Mentorship status when the self-flag is off but the pair ends (CAP-9)

Status reverts to "open to mentoring" when a mentor's last active pair ends **only if** their self-flag is still on at that moment; otherwise it reverts to no status at all. The flag is the independent source of truth for willingness — reverting against an explicit off-flag would be a bug. *(v2 additionally confirms un-flagging mid-pair never touches the active pair itself — status stays "mentor" until the pair actually ends, consistent with this decision.)*
*PO sign-off: not needed.*

## D5 — Profile header mentor visibility (CAP-1)

§4.11's explicit "visible to the reporting/project line and PP" governs the header's mentor field specifically, overriding the general Colleague-R grant on identity-card content — the more specific, later rule is authoritative.
*PO sign-off: confirmed by the user, 2026-08-21.*

## D6 — Mentorship-end feedback storage (CAP-9)

Store the mandatory closing note as its own field directly on the mentorship pair record, **not** routed through the general Feedback (S8/CAP-11) entity. Deliberate decoupling — removes what would otherwise be a hard CAP-9 → CAP-11 dependency. **Read scope, made explicit by `project-requirements-v2.md` §4.11:** the closure note is readable by the reporting line, the project line, and PP; **never** by the mentor, the mentee, or colleagues. A departure-triggered auto-close (CAP-14) writes a system-generated closure note and bypasses the mandatory-closure-note gate, since a departed person cannot supply one.
*PO sign-off: not needed.*

## D7 — IDP self-complete ownership (CAP-8)

Both the manager/PP maintenance side and the employee's self-complete checkbox live in the same story, owned by the same developer, rather than being split across CAP-3 (Self-Service) and CAP-8. Avoids a two-developer handoff on one small feature.
*PO sign-off: not needed.*

## D8 — Cross-system identity strategy (CAP-13)

Starting schema (interface contract C5): a dedicated `external_identities` mapping table keyed by `(system, externalId) → employeeId`, populated explicitly on first link — never inferred from email alone — with an explicit `supersededBy` pointer for re-hires. A Wave-0 ratification, not a final answer. **Ordering note (v2):** the platform's own seeded record (§4.17), not a PeopleForce candidate, is now the primary identity anchor — a person may never have a PeopleForce record at all — but the candidate ID is still retained wherever one exists, so a hired candidate links back to the resourcing request they came from.
*PO sign-off: not needed to start; the investigation itself (D12) is where real answers get confirmed.*

## D9 — Authentication mechanism (cross-cutting, architecture spine AD-9)

Simple local auth (email/password or magic-link against the platform's own `User` table, JWT session) — no source document addressed this before the architecture coaching session. Kept as a strictly separate concern from access-role resolution. **`project-requirements-v2.md` §4.17 independently confirms this stays correct**: no SSO, no Entra ID this iteration, authentication is the platform's own implementation over the seeded population.
*PO sign-off: confirmed by the user, 2026-08-21 (during architecture coaching); reaffirmed by the v2 source document itself.*

## D10 — Deployment target (cross-cutting, architecture spine AD-12)

Single containerized deployment — `docker-compose` extended to backend + frontend + Postgres, one host or a simple container platform, one environment this iteration. Satisfies "deployed and demonstrable, not on a laptop" (`SPEC.md` Success signal) without k8s/cloud-native complexity disproportionate to scope.
*PO sign-off: confirmed by the user, 2026-08-21 (during architecture coaching).*

## D11 — Default functional-role permission assignments (CAP-1) — OPEN

`project-requirements-v2.md` §2.3 explicitly flags this as needing PO confirmation before the roles admin screen is built, naming five specific gaps the source itself doesn't settle: who may hold *manage custom fields*, who may hold *assign and end mentorships*, and the launch defaults for *approve or reject proposed candidates*, *edit the career timeline*, and *create feedback*. Current default pending sign-off: seed the pre-update hardcoded assignments as the starting grants — HR Admin + managers for custom fields; UM + managers/PP for mentorship-assign; DM for candidate approve/reject; UM/PP for timeline edit; managers/PP for feedback — then let HR Admin adjust via the roles UI from there.
*PO sign-off: needed before the roles admin screen ships defaults; not needed to start building the permission mechanism itself, which is default-agnostic.*

## D12 — Timetracker sync model: events vs. state snapshots (CAP-13) — OPEN

`project-requirements-v2.md` §5.1 instructs the team to establish, from the timetracker's own documentation, whether the platform receives **change events** or only **state at sync time**. This determines whether a "why did this person have access on 14 August" question is answerable at all from the relationship journal, and should be settled during the same investigation that ratifies D8's identity-mapping shape.
*PO sign-off: not needed; this is a technical investigation outcome, not a product decision — record the finding here once known.*

## D13 — Multi-audience overlap resolves as union (CAP-1)

When a single viewer's relationships resolve to more than one audience for the same subject — e.g. they hold both PP and Project-line access, or Reporting-line and PP — effective access is the **union**, evaluated per section: for every section, the least-restrictive access among all resolved audiences applies. There is no single "winning" audience and no precedence order to get wrong. This is the only reading consistent with the matrix already being defined per-audience-per-section rather than as a single ranked role list.
*PO sign-off: not needed — this is the only implementation-safe reading of the existing matrix; the alternative (precedence) would silently deny or over-grant depending on ranking choice.*

## D14 — Shared-link exposure re-clamps to creator's current access, continuously (CAP-1)

A shared link's exposed sections are not fixed at creation time. On **every** view, each `cfg` section the link exposes is re-checked against the creator's *currently held* access to that section — not merely whether the creator's relationship to the subject still exists at all. If a creator's access narrows (e.g. Reporting line → Project line after a reassignment), any section the narrower tier doesn't grant stops rendering for the recipient on the very next view, with no separate revocation step required. This generalizes the existing "creator's access re-checked on every view" rule from a binary alive/dead check to a per-section clamp.
*PO sign-off: not needed — this is the literal meaning of "re-checked on every view" once a creator's access can narrow without fully ending, which v2's reporting-line/project-line split makes newly possible.*

## D15 — Organisational-relationship write safety (CAP-1, `OrgRelationshipWriter`/C9)

Four write-time guards on the four access-switch fields, closing gaps the source left to implementation judgment:

- **No self-managed department.** `changeDepartmentManager` rejects a `newManagerId` who is a current member of the target department or any of its nested sub-departments — a department can never manage itself into existence as a backdoor to Reporting-line access over one's own profile.
- **No reporting cycles.** `changeManager` rejects a `newManagerId` who is already a descendant of the subject in the reporting chain — the transitive-closure walk that resolves Manager access must never be allowed to loop.
- **No silent concurrent overwrite.** A concurrent conflicting write to the same access-switch field is rejected with a conflict error via an optimistic-concurrency check (e.g. a version/timestamp precondition) — the losing write is rejected outright and never reaches the journal as if it had applied.
- **No orphaning clear.** Clearing a department's manager through the direct edit path (not a departure) is blocked unless a replacement manager is designated in the same change — the same "never leave a department headless" principle CAP-14 already applies to departures, generalized to the direct-edit path FR-7 exposes.

*PO sign-off: not needed — each guard prevents a state the spec's own invariants (no self-assignment, access-control correctness as the primary quality attribute) already rule out; these just make the write path actually enforce it.*

## D16 — Full-access zero-holder protection (CAP-1)

Two refinements to "removing the last holder is blocked":

- A revocation of full profile access re-checks the **current** holder count at commit time — not a value read earlier in the request — and is rejected if it would fall to zero. This closes the race where two holders each revoke the other at the same instant and both reads see "count = 2" before either write lands.
- Recording a departure for the sole remaining full-access holder is blocked by the exact same gate CAP-14 already uses for a departing manager/PP who still has reports — the grant must be transferred to another holder first, before the departure can be recorded.

*PO sign-off: not needed — both are the same "never zero holders" invariant already stated, just enforced against concurrency and against the departure path specifically, not only the direct-revoke path.*

## D17 — Departure-cascade completeness and idempotency (CAP-14)

Three refinements to the CAP-14 cascade, each closing a case where the cascade could double-write, silently fail to write, or race itself:

- **Mentorship auto-close is idempotent.** The step is a no-op if the pair record is already ended — so when both members of a pair depart on overlapping effective dates, the second departure's cascade trigger never re-writes or double-logs the system-generated closure note.
- **Shared-link revocation is explicit.** The cascade explicitly revokes every shared link the departing employee created, as its own named step — not left as an implicit side-effect of D14's continuous re-clamp (which, for a departed creator with zero remaining access, would already zero out every section but leaves the link record itself technically "alive"). SM-6 says the link "should stop working"; this makes that a direct guarantee rather than an emergent one.
- **Action-item auto-cancel is idempotent.** The step is a no-op against an item already in a cancelled state — so a concurrent manual author-cancellation and a departure-triggered auto-cancellation can arrive in either order without one overwriting the other's cancellation reason/cancelled-by field.

*PO sign-off: not needed — these are correctness fixes to a cascade the spec already requires to be atomic and immediate; none change what the cascade does, only that it does it safely under concurrency and repetition.*

## D18 — Resourcing lifecycle robustness (CAP-6)

Six refinements, all closing gaps in the approve/reject/close lifecycle the source states informally ("always accurate," "never auto-closes") without specifying the enforcement:

- **Approval is reversible.** An approved candidate can be reversed to rejected with a written reason, freeing the headcount slot it had filled — otherwise a later-discovered ineligibility has no path back and the slot stays permanently marked filled.
- **Headcount can't be shrunk under water.** A headcount edit that would drop the total below the already-approved/filled count is rejected outright; the DM must first reverse enough approved candidates (previous bullet) before lowering headcount.
- **Zero remaining blocks further approval.** Approving a new candidate is blocked once remaining headcount is zero, until the DM raises headcount or closes the request — a request never over-fulfills beyond its stated headcount, closing the gap left by "only the DM's explicit close ends the request."
- **UM routing is live, not pinned.** A pending request's routing to a UM re-resolves to the department's current UM on every view, never pinning to whoever held that role at creation time.
- **Departed UM's proposals get reassigned.** If the previous bullet's re-resolution finds no successor UM for the department (e.g. it's temporarily headless), a pending proposal authored by the now-departed UM is flagged for reassignment rather than left silently orphaned.
- **Departed DM's pending decisions get reassigned.** A request awaiting a decision from a now-departed DM auto-routes to a live DM on the same project if one is resolvable; otherwise it is flagged for reassignment to a Full-access holder as backstop — mirroring CAP-1's "full-access holders are the backstop revoker" pattern for shared links.

*PO sign-off: not needed to start — each is an enforcement detail of an already-stated informal rule; worth a confirmation pass once the resourcing UI is reviewed end-to-end.*

## D19 — Sync-clock semantics refine D3 (CAP-13)

Two clarifications to D3's concrete 15-minute/4-hour numbers, both edge cases the numeric policy itself didn't spell out:

- **The 4-hour clock is cumulative, not resettable.** The withdrawal clock accumulates total failed sync time within a rolling window, rather than resetting to zero on any brief recovery — a sync that flaps (fails, briefly recovers, fails again) without ever sustaining 4 *continuous* failed hours, but whose cumulative failed time within the window exceeds 4 hours, still triggers withdrawal. A resettable clock would let a flapping sync accumulate unbounded staleness while the safeguard never fires.
- **A single success confirms.** A new project assignment is confirmed the moment one successful sync observes it — `confirmedAt` (C3) is stamped on first successful observation, not after 15 uninterrupted minutes of continuous success. The 15-minute figure is the maximum bound on confirmation latency, not a sustained-success requirement; reading it the other way would mean a flapping sync during the grant window could never confirm a genuinely new assignment at all.

*PO sign-off: not needed — both read the existing 15-minute/4-hour numbers the only way that doesn't contradict their own stated purpose (a bounded, deliberate trade-off that still eventually fires).*

## D20 — Mentorship pair write-time safety (CAP-9)

Two guards on pair creation and ending:

- **Consent is checked server-side, at write time.** Creating a mentor-mentee pair is rejected unless the prospective mentor's open-to-mentoring flag is currently on, checked in the write path itself — never relying on the pool UI to have already filtered non-flagged people out, since a stale UI or a direct API call must not be able to assign a non-consenting employee as a mentor.
- **Concurrent pair-ending is a rejected conflict, not a silent overwrite.** Ending a pair is rejected if the pair's status has already transitioned to `ended` by a concurrent request — the same optimistic-concurrency pattern as D15's concurrent-write guard. This is deliberately a *rejection* (the second closer sees a conflict and knows someone beat them to it), unlike D17's departure auto-close, which is an idempotent *no-op* — an automated cascade must never fail a transaction on a race it itself caused, but two humans racing to close the same pair should each know what happened.

*PO sign-off: not needed — consent-on-write was already implied by "the pool UI" language being descriptive, not the enforcement mechanism; the conflict/no-op distinction follows directly from D17 already establishing the no-op precedent for system-triggered closes.*

---

## Appendix — smaller product questions worth a quick PO answer

None of these block early work; each already ships with a reasonable default. Listed so they don't get lost before the relevant capability is picked up.

- **Photo/certificate file constraints (CAP-3).** Format/size limits unspecified; sane defaults apply. Confirm before shipping if the org has specific requirements (e.g. max upload size).
- **Reopening a completed IDP (CAP-8).** Can a manager/PP un-complete an IDP the employee already marked done? Current default: no — disallow until told otherwise.
- **Saved views with a static membership list (CAP-2).** The spec's "manually-maintained bench list" example implies a saved view might sometimes be an explicit list of people rather than a pure filter. Current default: filter-based views only for v1; confirm if a static-list variant is actually needed.
- **Assessment "assessor" field shape (CAP-8).** Free text or a reference to a person record? Either works as written; pick one before building the UI.
- **Timeline event deletion — hard vs. soft delete (CAP-7).** Given access-control correctness is the platform's primary quality attribute, soft-delete-with-audit-trail is the safer default; confirm if hard delete is acceptable instead.
- **Action item cancellation after the author loses access (CAP-4).** Current default: the author can still cancel their own historical item even without live Manager/PP access to the assignee, since authorship is a historical fact, not a live permission. Distinct from CAP-14's departure-triggered auto-cancellation ("cancelled — departed"), which is system-driven and unconditional. Confirm the author-cancellation default matches intent.
- **Shared profile link default expiry (CAP-1).** Source states a concrete default explicitly: 24 hours, configurable at creation — **except** links auto-generated by Resourcing (CAP-6), which instead live until the request is decided (`project-requirements-v2.md` §4.8). Not a gap — recorded here since it's the kind of small parameter easy to lose track of.
- **Accessibility conformance target (Constraints).** Source requires "accessibility and responsive layout" for List/Profile/Dashboard but states no conformance level. Current default: WCAG 2.1 AA, the industry-standard baseline; confirm if the org targets something else (e.g. AAA, or a lower bar).
- **Saved view ownership on creator departure (CAP-2).** An edge-case-hunter finding proposed transferring a departed creator's saved view to "a current manager/PP holder" — but a saved view isn't a profile-scoped entity, so there's no clean manager/PP *of a view* to transfer to. Current default: the view becomes **ownerless** on departure — it stays visible/usable by anyone it was shared with, but can't be edited or deleted until an HR Admin (or whoever holds *manage custom fields*) explicitly adopts it as the new owner. Confirm if a hard-retire (auto-archive on departure) is preferred instead.

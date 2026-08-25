# Decision Log

Companion to `SPEC.md` — see there for the capabilities these decisions bear on. Concrete defaults resolving points where the source spec deliberately doesn't answer its own question. Every default is revisable — the sign-off line says which were worth a PO confirmation and which were safe to just build. Sourced from `PRD_parallel_delivery_plan.md` §3 and §8 (appendix), plus two items added after review (link expiry, accessibility target) that the source left unstated. D3 and D5 originally carried as `SPEC.md` Open Questions since the source itself flagged them for confirmation; both were confirmed directly by the user on 2026-08-21 (not sourced from the original PRD draft, which only recommended confirmation) and are now folded into `SPEC.md`'s Assumptions along with the rest of this log. D9 and D10 are cross-cutting decisions the source material never addressed at all — they surfaced and were settled during the `bmad-architecture` coaching session, not derived from any prior document.

## D1 — Access-resolution caching (CAP-1)

A short-TTL, per-subject access cache keyed by subject ID, tagged with a monotonically increasing generation counter on the relationship graph (reports-to + project assignment + PP assignment). Any write to that graph bumps the counter and invalidates affected cache entries synchronously — correctness comes from the generation bump, not the TTL. TTL exists purely as a performance knob.
*PO sign-off: not needed to start; revisit after performance testing if the approach doesn't hold under load.*

## D2 — Timeline conflict resolution (CAP-7)

A manually-edited timeline event is never silently overwritten by a later system-generated event covering the same change window; the system write is skipped and the skip is logged/auditable. A sync that silently re-breaks a manual correction defeats the feature.
*PO sign-off: not needed; document as an ADR per the spec's own instruction.*

## D3 — Timetracker sync failure behavior and access (CAP-13)

**Fail-safe for access, fail-soft for display.** If the sync cannot currently confirm a project assignment, that assignment does not grant Manager access (treat "unknown" as "not confirmed," never "still active"). Separately, if the sync is merely slow/briefly unavailable, S10/S11 display shows "temporarily unavailable" rather than blocking the profile. Two different axes — a security control fails closed; a display feature fails soft.
*PO sign-off: confirmed by the user, 2026-08-21.*

## D4 — Mentorship status when the self-flag is off but the pair ends (CAP-9)

Status reverts to "open to mentoring" when a mentor's last active pair ends **only if** their self-flag is still on at that moment; otherwise it reverts to no status at all. The flag is the independent source of truth for willingness — reverting against an explicit off-flag would be a bug.
*PO sign-off: not needed.*

## D5 — Profile header mentor visibility (CAP-1)

4.11's explicit "visible to manager line and PP" governs the header's mentor field specifically, overriding the general Colleague-R grant on identity-card content — the more specific, later rule is authoritative.
*PO sign-off: confirmed by the user, 2026-08-21.*

## D6 — Mentorship-end feedback storage (CAP-9)

For v1, store the mandatory closing feedback as its own field directly on the mentorship pair record, **not** routed through the general Feedback (S8/CAP-11) entity. Deliberate decoupling — removes what would otherwise be a hard CAP-9 → CAP-11 dependency. Revisit unifying into S8 later if mentorship feedback should be queryable alongside other feedback.
*PO sign-off: not needed.*

## D7 — IDP self-complete ownership (CAP-8)

Both the manager/PP maintenance side and the employee's self-complete checkbox live in the same story, owned by the same developer, rather than being split across CAP-3 (Self-Service) and CAP-8. Avoids a two-developer handoff on one small feature.
*PO sign-off: not needed.*

## D8 — Cross-system identity strategy (CAP-13)

Starting schema (interface contract C5): a dedicated `external_identities` mapping table keyed by `(system, externalId) → employeeId`, populated explicitly on first link — never inferred from email alone — with an explicit `supersededBy` pointer for re-hires. A Wave-0 ratification, not a final answer; the actual PeopleForce/timetracker investigation may refine it, but it unblocks every other story that needs to reference "this employee's external record" before that investigation completes.
*PO sign-off: not needed to start; the investigation itself is where real answers get confirmed.*

## D9 — Authentication mechanism (cross-cutting, architecture spine AD-9)

Simple local auth (email/password or magic-link against the platform's own `User` table, JWT session) — no source document addressed this at all before the architecture coaching session. Kept as a strictly separate concern from access-role resolution (auth answers "who is this"; `AccessResolver` answers "what can they see about X"), so a later SSO swap touches only the `auth` module, never the access model.
*PO sign-off: confirmed by the user, 2026-08-21 (during architecture coaching).*

## D10 — Deployment target (cross-cutting, architecture spine AD-12)

Single containerized deployment — `docker-compose` extended to backend + frontend + Postgres, one host or a simple container platform, one environment this iteration. Satisfies "deployed and demonstrable, not on a laptop" (`SPEC.md` Success signal) without k8s/cloud-native complexity disproportionate to scope.
*PO sign-off: confirmed by the user, 2026-08-21 (during architecture coaching).*

---

## Appendix — smaller product questions worth a quick PO answer

None of these block early work; each already ships with a reasonable default. Listed so they don't get lost before the relevant capability is picked up.

- **Photo/certificate file constraints (CAP-3).** Format/size limits unspecified; sane defaults apply. Confirm before shipping if the org has specific requirements (e.g. max upload size).
- **Reopening a completed IDP (CAP-8).** Can a manager/PP un-complete an IDP the employee already marked done? Current default: no — disallow until told otherwise.
- **Saved views with a static membership list (CAP-2).** The spec's "manually-maintained bench list" example implies a saved view might sometimes be an explicit list of people rather than a pure filter. Current default: filter-based views only for v1; confirm if a static-list variant is actually needed.
- **Assessment "assessor" field shape (CAP-8).** Free text or a reference to a person record? Either works as written; pick one before building the UI.
- **Timeline event deletion — hard vs. soft delete (CAP-7).** Given access-control correctness is the platform's primary quality attribute, soft-delete-with-audit-trail is the safer default; confirm if hard delete is acceptable instead.
- **Action item cancellation after the author loses access (CAP-4).** Current default: the author can still cancel their own historical item even without live Manager/PP access to the assignee, since authorship is a historical fact, not a live permission. Confirm this matches intent.
- **Shared profile link default expiry (CAP-1).** Source states a concrete default explicitly: 24 hours, configurable at creation (project-requirements.md §4.8). Not a gap — recorded here since it's the kind of small parameter easy to lose track of.
- **Accessibility conformance target (Constraints).** Source requires "accessibility and responsive layout" for List/Profile/Dashboard but states no conformance level. Current default: WCAG 2.1 AA, the industry-standard baseline; confirm if the org targets something else (e.g. AAA, or a lower bar).

---
title: Reconciliation Review — ARCHITECTURE-SPINE.md vs. 2026-08-26 SPEC/PRD update
date: 2026-08-26
reviewer: bmad-review (reconciliation pass, ad hoc)
scope: >
  Does ARCHITECTURE-SPINE.md's 2026-08-26 update (AD-14–AD-18 + amendments to
  AD-1/AD-2/AD-4/AD-7) fully account for project-requirements-v2.md / SPEC.md /
  access-model.md / decisions.md (D13–D20) / interface-contracts.md? And does
  anything the spine added misread those sources?
inputs:
  - _bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md
  - _bmad-output/specs/spec-people-management-platform/SPEC.md
  - _bmad-output/specs/spec-people-management-platform/access-model.md
  - _bmad-output/specs/spec-people-management-platform/decisions.md
  - _bmad-output/specs/spec-people-management-platform/interface-contracts.md
  - git diff of _bmad-output/planning-artifacts/prds/prd-people-management-2026-08-21/prd.md
---

# Verdict

**Mostly sound, with one critical omission and two real cross-module gaps mislabeled as "no new AD needed."** AD-14 through AD-18 correctly absorb the large majority of the 2026-08-26 update — the three-relation/two-tier hierarchy (AD-14), multi-audience union (AD-15/D13), shared-link continuous re-clamp (AD-16/D14), org-relationship write guards (AD-17/D15), and the departure cascade's transactional/idempotent shape (AD-18/D16/D17) are all well-reasoned and traceable to specific source lines. But the departure cascade as architected **cannot actually satisfy FR-63's "the person's account deactivates" clause** — no module is wired to do it — and the spine's claim that D18 (resourcing robustness) needs no new AD **does not hold** for its "Full-access holder as backstop" sub-clause, which is a genuine cross-module read with no contract behind it. A smaller, PRD-level inconsistency (FR-65 vs. the S4 matrix) also passes through unflagged.

---

## Findings

### Finding 1 — CRITICAL: FR-63's "account deactivates" step has no owner, no hook, no path to exist

**Source:** SPEC.md CAP-14 intent ("the person's account deactivates"); PRD FR-63 ("the person's account deactivates, and every other access grant they held ends"); `interface-contracts.md` C11 ("the person's `User` account → deactivated").

**Spine gap:** AD-18's cascade is fully enumerated — both in prose and in its own mermaid diagram — as exactly four `departure-hook` registrants: `action-items` (cancel open items), `mentorship` (end active pairs), and `access` ×2 (revoke created `SharedLink`s, revoke held `FullAccessGrant`). Account deactivation is not one of them, in either the rule text or the diagram (`ARCHITECTURE-SPINE.md:236-247`).

This isn't a rounding error — AD-1's 2026-08-26 update is explicit about which modules newly need a `registry` dependency *because* they register a departure-hook: "`actionitems` and `mentorship` gain a `registry` dependency here because each now *registers* a `departure-hook` provider (AD-18)" (`ARCHITECTURE-SPINE.md:85`). `auth` is not listed, and nowhere else in the module dependency graph does `auth` depend on `registry`. Combined with AD-9's deliberate wall ("`auth` has zero knowledge of sections, roles, or the access matrix" — `ARCHITECTURE-SPINE.md:168`), there is no architected path for the departure cascade to ever reach `auth`'s `User` table. A grep of the whole spine for "deactivat" or "account" returns zero hits.

**Why it matters:** This isn't cosmetic — FR-63 states it as a required cascade step, SM-7 tests for it ("Departure cascade — ... account, access) with no orphaning..."), and C11's own signature commentary in `interface-contracts.md` names it explicitly. As written, a developer implementing AD-18 faithfully would ship a cascade that does everything *except* deactivate the account, and nothing in the spine would catch that in review, because the spine itself doesn't ask for it.

**Recommendation:** Add `auth` as a fifth `departure-hook` registrant (idempotent, same transaction client), and add `auth --> registry` to AD-1's dependency graph and its 2026-08-26 delta note.

---

### Finding 2 — HIGH: "Full-access holder as backstop" is a real cross-module contract with no contract behind it — D18 is not correctly assessed as no-new-AD

**Source:** `decisions.md` D18 ("a request awaiting a decision from a now-departed DM ... is flagged for reassignment to a Full-access holder as backstop"); SPEC.md CAP-14 / `decisions.md` D16 (departure blocked for the sole remaining full-access holder); `access-model.md`'s Full profile access section.

**Spine gap:** The Capability→Architecture Map states: "reversible approval/headcount-floor/departed-DM reassignment (D18) stay `resourcing`-internal business logic, no new AD" (`ARCHITECTURE-SPINE.md:380`). The reversible-approval and headcount-floor guards genuinely are internal to `resourcing`'s own `ResourcingRequest`/`ResourcingProposal` tables — that part of the claim holds. But "flagged for reassignment to a Full-access holder as backstop" requires `resourcing` to know *who currently holds full access* — data owned by `access` (`FullAccessGrant`, per the Structural Seed). AD-1 forbids `resourcing` from importing `access` or querying its tables directly; per AD-2, the only legitimate path is a `contracts`-declared interface. **No such contract exists.** C1 (`AccessResolver`) resolves audience for a `(viewer, subject)` pair, not "list current full-access holders." C8 (`PermissionChecker`) is a boolean per-user permission check, not a holder lookup. There is no C13.

The same gap exists a second time, in the opposite direction: AD-18 requires `employment.recordDeparture` to block "if... `subjectId` is the sole remaining Full-access holder" (`ARCHITECTURE-SPINE.md:235`) — another cross-module read of `access`'s `FullAccessGrant` state, again with no contract named for it (C11's signature in the AD-2 table takes no dependency on any such query).

**Corroborating evidence the spine itself half-recognizes this pattern but never ratifies it:** AD-16 says shared-link revocation falls to "a Full-access holder as the backstop... (**mirrors AD-14/AD-18's departed-manager backstop pattern**)" (`ARCHITECTURE-SPINE.md:224`). But neither AD-14 (department hierarchy/two-tier resolution) nor AD-18 (departure cascade for the *departing person's own* grants) actually contains a "route this decision to whoever holds full access" mechanism — AD-18 revokes grants the departed person *held*, it never reassigns *other people's* pending work to a full-access holder. The citation is factually wrong, which is itself evidence that this "backstop" pattern has been reused informally across three places (AD-16 shared-link revocation, D18 DM/UM reassignment, D16/AD-18 sole-holder departure block) without ever being formalized as its own contract — exactly the kind of implicit, undocumented cross-module coupling AD-1/AD-2 exist to prevent.

**Recommendation:** Either fold a `FullAccessHolderDirectory`-shaped read (e.g., `getCurrentHolderIds()`) into C1's or a new contract's surface, exposed via `contracts`+`registry` like C12, and correct AD-16's miscitation; or explicitly narrow D18's Capability-Map claim to exclude the backstop clause from "no new AD."

---

### Finding 3 — MEDIUM-HIGH: Department-tree restructuring has no guarded write path and no cycle guard, unlike the analogous employee case

**Source:** `access-model.md`'s Functional roles table adds **"manage departments"** as a new (bolded, i.e. genuinely new) permission, and lists "departments" under HR Admin's administration remit; AD-14 makes `Department.parentId` self-referential and load-bearing for C12's `getAncestorIds`/`getDescendantIds`, which multiple capabilities (directory, cds, resourcing, dashboards) depend on being resolved correctly and "live."

**Spine gap:** C9/AD-17 (`OrgRelationshipWriter`) is explicitly scoped to exactly four access-switch fields: employee-manager, employee-PP, employee-department (membership), and department's-manager. It never covers *creating* a `Department` or editing its own `parentId` (moving a department to a different parent in the tree) — that's the "manage departments" permission's job, and no contract, guard, or AD addresses it at all. D15's four write-time guards (self-managed department, no cycles, no concurrent overwrite, no orphaning clear) are all scoped to `changeManager`/`changeDepartmentManager` — i.e., to *employee*-to-department/manager assignment — never to the *department-to-department* parent/child edges themselves.

Concretely: nothing in the spine stops HR Admin from setting Department A's parent to Department B while B's parent is (transitively) A, producing a cycle in the tree AD-14's C12 walks for `getAncestorIds`/`getDescendantIds`. That would either infinite-loop or return undefined results for every consumer of C12 — directory's department column/filter, CDS's matrix dictionary, resourcing's live UM routing, dashboards' department grouping — i.e., exactly the class of bug D15 was written to close for the *employee* reporting chain ("the transitive-closure walk that resolves Manager access must never be allowed to loop"), left open one level up, in the tree that closure walk actually runs on.

**Recommendation:** Extend C9 (or add a small companion write contract) to cover Department create/reparent, with an analogous no-cycle guard on `parentId`, and note this explicitly under AD-14 or AD-17.

---

### Finding 4 — MEDIUM: FR-65 (employment status hidden from Self) is unaddressed by the spine and appears to conflict with access-model.md's own S4 matrix

**Source:** PRD FR-65: "Employment status appears in S4 (Employment) and is readable by reporting line, project line, and PP — **never by Self or Colleague**."

**Conflict:** `access-model.md`'s S4 row lists "employment status (see CAP-14)" as one of S4's Contents, and grants **Self = `R`** for the whole section (`access-model.md:99`). Colleague is already `—` for all of S4, so FR-65's "never Colleague" clause is trivially already true — but "never Self" directly contradicts Self's blanket `R` grant over the same section that contains the field.

**Spine gap:** The architecture has a precedent mechanism for exactly this shape of problem — a single field inside an otherwise-broadly-granted section needing a stricter, narrower rule (AD-2/AD-5's mentor-field override, `access-model.md` Rule 8, cited explicitly in the spine at `ARCHITECTURE-SPINE.md:142`). Nothing analogous is proposed for the employment-status field, and the spine doesn't flag the apparent PRD/access-model.md disagreement for resolution the way it flags other open questions (e.g., D11, D12, the AD-11 UX-spine gap). If FR-65 is correct, `C1`/`SectionProvider(S4)` needs a field-level carve-out identical in shape to the mentor-field one; if access-model.md's blanket Self-`R` is correct, FR-65 is simply wrong and should be corrected upstream. Either way, this is unresolved and untouched by the 2026-08-26 spine update.

**Recommendation:** Flag back to the SPEC/PRD authors for a one-line disambiguation, then encode whichever answer wins as a named exception under AD-2's C1 row (next to the existing mentor-field carve-out) if Self-exclusion is confirmed.

---

## Areas checked and found correctly reconciled (no material gap)

- **CAP-14 / FR-61–FR-65 overall.** Beyond Findings 1 and 4 above, FR-61 (temporal fact, never conflated with `leaver`, reactivation explicitly deferred) is fully covered by AD-7's fifth temporal dimension + the Deferred section's explicit FR-61 callout (`ARCHITECTURE-SPINE.md:401`); FR-62 (departure blocked while still holding live Reporting-line/PP responsibility, or while the sole full-access holder) is covered by AD-18's blocking rule, modulo Finding 2's "how does `employment` know it's the sole holder" gap; FR-64 (excluded from default list, still filterable) is covered by AD-3's `field`/`dashboard-summary` registry families reading C11.
- **AD-18's scoping of the departure block to "Reporting-line/PP" (not Project-line).** This is deliberate and correct, not an oversight: AD-14 defines Reporting-line as reports-to ∪ department-management, so a departing department manager is already covered under "Reporting-line responsibility." Project-line (PM/DM) responsibility is intentionally handled differently — via D18's *live* re-resolution/reassignment rather than a hard pre-departure block — which is architecturally consistent with Project-line assignments being multi-holder and dynamic rather than single-owner the way manager/PP/department-manager are.
- **Three-relation/two-tier hierarchy (AD-14).** Reports-to, department-management, and project-assignment are correctly unioned into exactly two tiers (Reporting line = first two; Project line = third, narrower), matching `access-model.md`'s Hierarchy resolution section and Rule 2 verbatim, including the narrowed section list (no S2/S3, S5 = CV+certs only) and the "manager two-or-more-levels-up inherits everything nested beneath" consequence. C12's `getManagedDepartmentIds` is correctly specified as "resolved live — never pinned," matching D18's UM-routing requirement.
- **Access switches (AD-17/C9/D15).** All four D15 guards (no self-managed department, no reporting cycles, no silent concurrent overwrite, no orphaning clear) are present in C9's spine signature and rule text, matching `decisions.md` D15 and `access-model.md`'s Access switches section point-for-point.
- **D13 (multi-audience union) → AD-15.** Correctly specified as a per-section least-restrictive union over the full audience set, explicitly forbidding a ranked-precedence reimplementation, matching `access-model.md` Rule 10 and `decisions.md` D13 exactly.
- **D14 (shared-link re-clamp) → AD-16.** Correctly generalizes from a binary alive/dead check to a continuous per-section re-clamp on every view, and correctly separates "exposure re-clamp" (automatic, via AD-16) from "explicit revocation on departure" (a named AD-18 step, per D17) rather than conflating the two — matching the SPEC's own framing that D17 exists precisely because the re-clamp alone leaves the link record "technically alive."
- **D19 (sync clock semantics) → AD-8.** The single-success-confirms and cumulative-non-resettable-window semantics are both captured with the correct concrete numbers (15 min / 4 hr), correctly distinguished from the pre-existing display-freshness flag.
- **D20 (mentorship write-time safety) — correctly assessed as needing no new AD.** Both guards (server-side consent check at write time, optimistic-concurrency rejection on concurrent pair-close) operate entirely on data `mentorship` already owns (`MentorshipPair`, the employee's own open-to-mentor flag) using a concurrency pattern (expected-version precondition) already established elsewhere in the spine for C9. Unlike D18's backstop clause, nothing here crosses a module boundary that AD-1 would otherwise block — I agree with the spine's classification.

---

## Summary for downstream action

| # | Severity | One-line finding |
|---|---|---|
| 1 | Critical | FR-63's "account deactivates" cascade step has no registered hook, no owning module wired via `registry`, and no path to be implemented as architected. |
| 2 | High | D18's "Full-access holder as backstop" (and AD-18's "sole remaining holder" departure check) is a genuine cross-module read with no contract — the "no new AD needed" claim for D18 is incorrect for this sub-clause; AD-16's own citation of this pattern points at two ADs (AD-14, AD-18) that don't actually contain it. |
| 3 | Medium-High | The new "manage departments" permission (Department create/reparent) has no guarded write contract and no cycle guard on `Department.parentId`, unlike the equivalent employee-reporting-chain guard (D15) it should mirror. |
| 4 | Medium | FR-65 ("employment status ... never Self") conflicts with access-model.md's S4 matrix (Self = R covers the whole section including employment status); the spine neither resolves nor flags this. |

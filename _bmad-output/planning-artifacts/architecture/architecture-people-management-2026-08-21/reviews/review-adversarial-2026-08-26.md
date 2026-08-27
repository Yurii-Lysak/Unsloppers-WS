---
name: 'Adversarial Review — ARCHITECTURE-SPINE.md (2026-08-26 update)'
type: review
reviewer: adversarial-architecture-review
target: '_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
target_version: 'updated 2026-08-26'
companions_reviewed:
  - '_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '_bmad-output/specs/spec-people-management-platform/interface-contracts.md'
  - '_bmad-output/specs/spec-people-management-platform/decisions.md (D13-D20 focus)'
date: '2026-08-26'
---

# Adversarial Review — Architecture Spine (2026-08-26 update)

## Verdict

**Not build-ready as a single, unambiguous contract.** The spine holds up well as a *narrative* of the 2026-08-26 changes, but it fails its own stated job — "the ONLY contract that keeps independently-built modules compatible" — in at least one place where two developers each following AD-14 to the letter would ship genuinely incompatible, security-relevant behavior (S7 access for Project-line PMs vs. DMs), and in several places where the AD-1 dependency graph, AD-3's registry mechanism, and AD-18's departure cascade do not actually compose into a buildable design without an invented, unratified extension. None of these are nitpicks about prose; each is a place where "obeys every AD to the letter" does not converge on one implementation.

## Top findings at a glance

| # | Severity | Finding |
| --- | --- | --- |
| 1 | Critical | AD-14's "Project line — everything else identical" contradicts `access-model.md`'s own S7 matrix cell (DM: RW vs PM: R+flag), and C1's role enum has no PM/DM sub-distinction to even implement the correct behavior |
| 2 | High | AD-1's module dependency graph omits a `registry` edge for `timeline` and `resourcing`, contradicting AD-3's own text/diagram, which requires both to register providers |
| 3 | High | AD-18's `recordDeparture` preconditions (live reports/PP check, sole-full-access-holder check) need reads that no C1–C12 contract exposes, while AD-1 forbids `employment` from querying `access`'s tables directly |
| 4 | High | The `departure-hook` registry family needs one-to-many fan-out ("call every registrant"), but AD-3's registry mechanism is specified as a single-provider-per-`(family,id)` lookup — the fan-out query shape is never defined |
| 5 | High | Departed-employee login handling is contradictory: `interface-contracts.md`'s C11 requires deactivating the `User` account on departure; AD-18's cascade lists exactly four hook registrants and none is `auth`, and AD-1 gives `auth` no `registry` dependency at all |

Full detail below, organized by the review's seven focus areas plus a cross-cutting section.

---

## 1. Department entity / DepartmentDirectory (C12) / two-tier hierarchy (AD-14)

### 1.1 [Critical] "Everything else identical" is false for S7, and the role enum can't express the true rule

AD-14's rule text says:

> the **Project line** (project-assignment closure alone) gets a **strictly narrower** map — no S2, no S3, S5 limited to CV+certificates, **everything else identical** (`access-model.md` Rule 2)

This is a verbatim paraphrase of `access-model.md` Rule 2. But `access-model.md`'s own section matrix contradicts Rule 2 for S7:

> S7 | Management notes | ... | Reporting line: RW | **Project line: DM: RW · PM: R, only *visible for PM*** | ...

So within the single audience AD-14 calls "Project line," a PM and a DM get materially different S7 access (R+flag-gated vs. unconditional RW) — not "everything else identical." This isn't a new bug in `access-model.md` (Rule 2 vs. its own S7 row already disagreed before 2026-08-26), but AD-14 elevates the *wrong* half of that pre-existing self-contradiction into the architecture spine's normative rule text, without ever mentioning the PM/DM split.

It gets worse at the contract level. C1's resolved shape (AD-2's table, and `interface-contracts.md` C1) only exposes:

```
role: Self | ReportingLine | ProjectLine | PP | Colleague | SharedLink | FullAccess
sections: Record<SectionId, "R" | "RW" | "none">
```

There is no PM/DM sub-audience anywhere in this enum. AD-14 additionally states, as a hard rule:

> A `SectionProvider` (AD-3) must never special-case "Manager" as one audience — it receives the resolved tier (`ReportingLine` or `ProjectLine`) from C1 ... the narrowing lives entirely in `AccessResolver`, never duplicated in a provider.

**Two developers, each fully compliant with AD-14's text, diverge here:**

- **Developer A** reads AD-14's rule literally ("everything else identical") and implements Project line as one fixed section-grant map with S7 = `RW` for every Project-line viewer, since nothing in AD-14 (as written) tells them PM and DM differ on S7.
- **Developer B** reads `access-model.md`'s S7 row and Rule 3 and knows PM must get only `R` with the *visible for PM* flag respected — but has no contract-shaped way to get that distinction, because C1 only ever hands their S7 `SectionProvider` a `ProjectLine` tier, indistinguishable from any other Project-line viewer. To honor Rule 3, Developer B is forced to do exactly what AD-14 forbids: have the S7 `SectionProvider` (or `AccessResolver` internally) special-case whether the specific project relation is via `pmId` or `dmId` — information AD-14 never says is preserved past the union step, and never appears in any contract signature.

Neither implementation is fully spec-compliant, and they produce different real access outcomes (a PM either can or cannot write management notes on their project's people) — a genuine data-visibility bug on both sides, exactly the class of leak `access-model.md` Rule 1 calls "a critical defect regardless of which section it happens in." At minimum, AD-14 needs an explicit statement that `AccessResolver` internally resolves PM-line and DM-line as two distinct sub-audiences (unioned per AD-15) before collapsing them to the single externally-visible `ProjectLine` label — and even then, the *content-level* flag check (`visible for PM`) still needs the PM/DM distinction to survive somewhere the S7 `SectionProvider` can read it, which no contract currently provides.

### 1.2 [Low] ERD cardinality on `Department.managerId` is ambiguous against C12's array-returning contract

The Structural Seed ERD declares:

```
Department |o--o| Employee : "managed by (current, AD-14/17)"
```

`|o--o|` is Mermaid's "zero-or-one to zero-or-one" notation — read literally, it says each `Employee` manages **at most one** `Department` directly. But C12's `getManagedDepartmentIds(employeeId) → id[]` returns an array, and nothing in AD-14 or `access-model.md` states whether that array can ever contain more than one *directly*-managed department (as opposed to one directly-managed department plus its descendants). Real orgs routinely have one manager heading two sibling departments (e.g., interim coverage). Two developers could reasonably diverge: one enforces a de-facto unique constraint on `Department.managerId` per employee (trusting the ERD's cardinality), the other allows an employee to be `managerId` on multiple `Department` rows (trusting nothing forbids it and the contract already returns an array). Worth one explicit sentence in AD-14 either way.

### 1.3 [Medium] No guard against a `Department.parentId` cycle, and no contract writes `parentId`/`name` at all

AD-17's enforcement list (the thing the dependency-cruiser/code-review check greps for) is explicit:

> a check that greps for direct writes to `Employee.managerId`/`ppId`/`departmentId` or `Department.managerId` outside `access`'s C9 implementation

`Department.parentId` and `Department.name` are **not** in this list, and C9 (`OrgRelationshipWriter`) has no method that writes either one — only `changeDepartmentManager`. C12 (`DepartmentDirectory`) is read-only. So: who creates a `Department`, and who reparents one? `access-model.md` mentions a *manage departments* functional permission, but no contract exposes the write. Two developers building "the department admin screen" mentioned as a Deferred UX gap would each invent their own write path (one might add it to C9, another might add a raw controller endpoint in `access` calling Prisma directly) — and since AD-17's guard list doesn't cover `parentId`, **neither** implementation is required to prevent a `Department` parent cycle (A's parent is B, B's parent is A), which would make C12's `getAncestorIds`/`getDescendantIds` (both explicitly living, recursive walks) loop forever. AD-17 is meticulous about reporting-chain cycles and self-managed departments; the symmetric case for the department tree itself is an unguarded gap.

---

## 2. Multi-audience union resolution (AD-15)

No outright contradiction found, but see 1.1 above: AD-15's union mechanism is actually the *only* plausible way to reconcile AD-14's single `ProjectLine` label with `access-model.md`'s PM/DM-differentiated S7 cell (by treating PM-line and DM-line as two internally-unioned audiences). AD-15 never says this, and AD-14 never cross-references AD-15 for this purpose — the reconciliation currently has to be reverse-engineered by whoever implements it, which is exactly the kind of thing a spine is supposed to make unnecessary.

---

## 3. Shared link continuous re-clamp (AD-16)

### 3.1 [Medium] "Manager" is used post-AD-14 without saying which tier

AD-16 states:

> Revocation rights and C10 journal-read rights for a link follow **whoever currently holds** Manager/PP access over the subject

AD-14 just finished eliminating "Manager" as a single resolvable audience — C1's enum only has `ReportingLine` and `ProjectLine`, which AD-14 explicitly says are *not* interchangeable (one gets the full section map, the other a narrowed one). AD-16 never says whether "Manager" here means `ReportingLine` only, or `ReportingLine ∪ ProjectLine`. The same ambiguous term appears in `interface-contracts.md`'s C10 (`readFor` "gated to full-access holders and the subject's current manager and PP") and in `access-model.md`'s journal section ("readable by ... the current manager and people partner of the subject") — three places, same unresolved word.

Two developers implementing the two related checks (shared-link revoke-authorization, and `C10.readFor` gating) could each pick a different tier scope for "Manager" — and could even diverge *from each other* on the same codebase, since nothing forces the two checks to share one definition. Given a Project-line PM has meaningfully less access than a Reporting-line manager (no S2/S3, S5 limited), whether a PM can revoke a shared link or read the relationship journal for that subject is a real, user-visible permission question this spine leaves unanswered.

---

## 4. Guarded org-relationship / full-access write paths (AD-17)

### 4.1 [Medium] C9's literal signature disagrees between the two documents that both claim to be current

The spine's AD-2 table gives C9 as one generic template:

```
changeManager/changePeoplePartner/changeDepartment/changeDepartmentManager(actorId, subjectId, newValue, expectedVersion) → JournalEntry
```

`interface-contracts.md`'s C9 section (updated 2026-08-26, per its own "Four write-time guards, added 2026-08-26" note) gives four distinct, non-generic signatures instead:

```
changeManager(actorId, subjectId, newManagerId) → JournalEntry
changePeoplePartner(actorId, subjectId, newPpId) → JournalEntry
changeDepartment(actorId, subjectId, newDepartmentId) → JournalEntry
changeDepartmentManager(actorId, departmentId, newManagerId) → JournalEntry
```

Two disagreements, not one: (a) `changeDepartmentManager`'s second parameter is `subjectId` in the spine's generic form vs. `departmentId` in `interface-contracts.md` — these are not the same value (a department has no single "subject" employee); (b) `expectedVersion` appears as an explicit fourth parameter in the spine but is only described in prose (never in the signature) in `interface-contracts.md`. `interface-contracts.md`'s own preamble says the spine supersedes it only "for C1–C6 ... and C7/C8" — it never disclaims C9–C12, so a developer who reads `interface-contracts.md` as authoritative for C9 (reasonably, since it's the more detailed and more recently-touched-looking of the two for this contract) will write a different method signature than one who reads the spine's table literally. This is a "second developer starts coding against a contract before the first developer finishes implementing it" scenario (interface-contracts.md's own stated purpose) failing on the exact contract it was supposed to de-risk.

### 4.2 See §1.3 above — Department creation/reparenting has no C9 method and no AD-17 write-path guard at all.

---

## 5. Departure cascade / departure-hook registry family / `employment` module (AD-18)

### 5.1 [High] Pre-departure precondition checks have no backing contract

AD-18's rule text:

> `recordDeparture` first blocks (structured error) if the subject still holds live Reporting-line/PP responsibility over anyone, or is the sole remaining Full-access holder (AD-17)

To implement this, `employment` needs answers to two questions: "does this employee currently manage anyone (via reports-to, department-management, or PP-assignment), and if so who," and "is this employee the sole full-access holder." Scanning the entire C1–C12 table:

- C1 (`AccessResolver.resolveAudience`) only answers "what can viewer X see of subject Y" for a *given pair* — it is not a reverse index of "who reports to me."
- C12 (`DepartmentDirectory`) only covers the department-management leg (`getManagedDepartmentIds`), not reports-to or PP.
- No contract exposes a full-access holder count or a "is X the sole holder" check.

AD-1 forbids `employment` from importing `access` or querying its Prisma tables directly — its only sanctioned inputs are `contracts` and `registry`. So the precondition check AD-18 requires **cannot currently be built** without either (a) adding a new contract nobody has ratified, or (b) a developer quietly reaching around AD-1's boundary "just this once" because the alternative doesn't exist. Two developers facing this gap independently would very plausibly invent two different ad hoc extensions (e.g., one bolts a `hasDirectReports(employeeId)` method onto C1, another adds a new `OrgGraphReader` contract) that are not interchangeable and were never reviewed against the rest of the access model.

### 5.2 [High] The `departure-hook` family needs a fan-out query shape AD-3 never defines

AD-3 specifies the registry mechanism as a `(family, id)` → single-provider index, with collision on the same `(family, id)` treated as a **bootstrap-time failure** — i.e., the model assumes at most one provider per key, looked up by a caller who already knows the specific `id` it wants (e.g., `getSection(S6)` looks up `(section, S6)`).

AD-18 needs the opposite query shape. `employment`'s cascade must "call every provider registered under a new `departure-hook` Provider Registry family" — and `access` itself registers **two** separate departure-hooks (shared-link revocation and full-access-grant revocation) under that one family. This is fundamentally a "list everything registered in this family, regardless of id, and invoke all of them" operation, which:

- Is never described as a capability of the `registry` module (only single-key lookup is specified in AD-3's rule text).
- Cannot use bootstrap-collision-as-failure semantics, since multiple registrants under the same family are not just tolerated here but *required*.
- If implemented by giving each hook a unique `id` (e.g., `id: "shared-link-revoke"`) so no `(family, id)` pair collides, then `employment` must still somehow enumerate *all* ids ever registered under `departure-hook` without hardcoding them — a "list all providers in a family" API that doesn't exist anywhere in AD-3.

If `employment`'s developer instead hardcodes the known hook ids (`"action-items-cancel"`, `"mentorship-end"`, `"shared-link-revoke"`, `"full-access-revoke"`) to work around the missing enumeration API, that reintroduces exactly the per-module coupling AD-18 claims to avoid ("reuses AD-3's registry mechanism ... instead of inventing a second cross-module-call mechanism") — `employment` would need to know about `action-items` and `mentorship` by name after all.

### 5.3 [High] Contradiction over whether login/`User` deactivation is part of the cascade, and who owns it

`interface-contracts.md`'s C11 section (carrying its own "2026-08-26" idempotency updates, so it is not stale pre-v2 text) states the cascade includes:

> the person's `User` account → deactivated; every access grant they held → revoked immediately (falls out naturally once C1 re-resolves against the now-`dismissed` status and severed relationships)

AD-18's cascade transaction, by contrast, lists exactly four `departure-hook` registrants: `action-items` (cancel items), `mentorship` (end pairs), and `access` twice (revoke SharedLinks, revoke FullAccessGrant). There is no `auth` registrant anywhere in AD-18's rule text or its accompanying mermaid diagram, and AD-1's dependency graph gives `auth` no `registry` edge at all (`auth --> contracts` only) — nor does the 2026-08-26 AD-1 update note (which explicitly calls out that `actionitems` and `mentorship` "gain a `registry` dependency here") mention `auth` gaining one.

So: is a departed employee's `User` account actually deactivated, and if so, by what mechanism, given `auth` has no sanctioned way to participate in the cascade? Either AD-18 is missing a fifth hook family member (`auth`) and a corresponding AD-1 edge, or `interface-contracts.md`'s C11 prose is stale and should be struck — as written, a developer building `employment` off the spine alone would not deactivate login on departure, while a developer (or a QA test) checking against `interface-contracts.md` would expect it to be blocked. Separately, `interface-contracts.md`'s "falls out naturally" framing for revoked grants directly contradicts D17/AD-18's explicit correction ("**Shared-link revocation is explicit** ... not left as an implicit side-effect") — `interface-contracts.md` was not updated to match the decision that superseded it.

---

## 6. AD-1's module graph vs. C9–C12 contract ownership, and vs. AD-3

### 6.1 [High] The graph omits `registry` edges for two modules AD-3 requires to have one

Cross-checking AD-1's mermaid graph against AD-3's own text and diagram:

- **`timeline`**: AD-1's graph shows `timeline --> contracts` only — no `registry` edge. But AD-3's *own* worked-example diagram (line ~126 of the spine) explicitly names `TimelineProvider["timeline: SectionProvider(S9)"]` as a registry-discovered provider, and AD-3's prose lists "one per S1–S16-owning module" — S9 (Career timeline) is owned by `timeline` per the Capability Map. A `SectionProvider` can only be discovered via `@RegisterProvider`, which requires depending on `registry`.
- **`resourcing`**: AD-1's graph shows `resourcing --> contracts` only. But AD-3's prose explicitly lists dashboard aggregates as including "resourcing counts" ("Dashboards' per-source aggregates: risk counts, leave status, **resourcing counts**, active/dismissed counts"), which is only possible if `resourcing` registers a `DashboardSummaryProvider` — again requiring `registry`.

Every other section/field/dashboard-summary-owning module in the same graph (`access`, `directory`, `actionitems`, `risks`, `cds`, `mentorship`, `feedback`, `dashboards`, `employment`) correctly has a `registry` edge, which shows the graph's author *does* track this deliberately elsewhere — making these two omissions look like a real oversight rather than an intentional design choice. A developer building `timeline` or `resourcing` and treating AD-1's diagram as the definitive statement of their module's dependencies would never add the `registry` import, never register the provider AD-3 requires, and — per AD-3's own "missing provider is a runtime error at first call, never a silent omission" rule — Dashboards' resourcing-count widget and the profile assembler's S9 section would both fail loudly in production rather than simply not existing. Meanwhile a second developer building `dashboards`/`access` in good faith, trusting AD-3's prose over AD-1's diagram, would build the consuming side expecting a provider that the first developer's reading of AD-1 never told them to build.

### 6.2 C9–C12 ownership itself is internally consistent

For completeness: C9/C10/C12 owned by `access`, C11 owned by `employment`, consumed cross-module only via `contracts` + DI token or `registry`, matching AD-1's core rule. No contradiction found here in isolation — the problems are all at the edges (§5.1, §6.1), not in the ownership table itself.

### 6.3 [Low] The source tree's own `access/` comment is internally inconsistent

```
access/  # ... departure-hook registrants for shared links + full access (AD-18) — AD-1/3/4/5/14/15/16/17
```

The same line credits `access` with AD-18 responsibility (departure-hook registrants) and then lists the module's governing ADs without including AD-18. Trivial to fix, but it's the kind of inconsistency that erodes confidence in the rest of the freshly-added annotations.

### 6.4 [Low] Capability Map omits AD-2 for the capability that owns the most contracts

CAP-1's "Governed by" column lists `AD-1, AD-3, AD-4, AD-5, AD-9, AD-14, AD-15, AD-16, AD-17` — no AD-2, despite CAP-1/`access` owning six of the twelve lettered contracts (C1, C7, C8, C9, C10, C12). Every other capability row that owns a contract (CAP-2, CAP-4, CAP-7, CAP-9, CAP-10, CAP-11, CAP-13, CAP-14) does cite AD-2 explicitly. Someone triaging "which ADs govern CAP-1's stories" off the Capability Map alone would miss the entire contracts-table discipline for the capability that needs it most.

---

## 7. ERD / source tree / Capability Map vs. AD text

Covered inline above (§1.2 ERD cardinality, §6.3/§6.4 source tree and Capability Map nits). No further contradictions found beyond those already listed; the ERD's new entities (`Department`, `EmploymentStatusHistory`, `FullAccessGrant`, `RelationshipJournalEntry`) otherwise match their AD-14/17/18 descriptions.

---

## 8. Minor / cosmetic

- **AD-4's graph count is confusingly worded.** "the relationship graph (reports-to, department management, project assignment, PP assignment — 2026-08-26: department management added as a third graph, AD-14)" lists department management *second* in the parenthetical but calls it "a third graph." Purely cosmetic, but combined with the fact that `decisions.md`'s D1 (not among the six decisions the changelog flags as touched by the 2026-08-26 pass — that's D13–D20) already lists "department management" in its own generation-counter description, it's unclear whether D1 was silently edited during this update or always anticipated a department concept. Worth a one-line changelog clarification so a reader doesn't have to guess.

---

## What would close these out

1. Add one sentence to AD-14 (or a new sub-rule) stating explicitly how PM-line and DM-line resolve differently for S7, and whether that's modeled as two internal audiences unioned per AD-15 before the `ProjectLine` label is assigned externally.
2. Add the missing `registry` edges for `timeline` and `resourcing` to AD-1's diagram (or explain why they're intentionally absent, if AD-3's text is wrong instead).
3. Add a C1 (or new) contract method that lets `employment` answer "does subject X still have live direct reports / PP assignees" and "is X the sole full-access holder" without violating AD-1.
4. Specify the `departure-hook` family's lookup semantics explicitly in AD-3 or AD-18 (a `getAll(family)`-style enumeration, distinct from the other three families' single-key lookup) and give the hook provider interface an explicit signature, including how the shared transaction client is typed without introducing a Prisma import into `contracts` (AD-2's "zero Prisma imports" rule).
5. Resolve the `auth`/`User`-deactivation question one way or the other, and update whichever of AD-18 / `interface-contracts.md` C11 is wrong.
6. Reconcile C9's two documented signatures (spine vs. `interface-contracts.md`) into one, and state explicitly that `interface-contracts.md`'s C9–C12 sections are (or are not) superseded by the spine, matching the disclaimer already given for C1–C8.
7. Add an AD-17-style guard (or an explicit "out of scope for v1, HR Admin edits directly via Prisma admin tooling" statement) for `Department.parentId`/`name` writes and cycle prevention.
8. Disambiguate "Manager" in AD-16/C10 as `ReportingLine`-only or `ReportingLine ∪ ProjectLine` for shared-link revocation and journal-read rights.

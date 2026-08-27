# Interface Contracts

Companion to `SPEC.md` — see there for capability context. Minimum interface surface — signatures only, not full designs — that lets a second developer start coding against a contract before the first developer finishes implementing it. Sourced from `PRD_parallel_delivery_plan.md` §2; each already has a natural owner among the epics in `SPEC.md`'s Capabilities.

**Superseded for exact signatures as of 2026-08-21:** the architecture spine (`ARCHITECTURE-SPINE.md`, AD-2's contract table) is now the authoritative, review-ratified source for C1–C6's precise shapes, plus two contracts this document never anticipated (C7 `CurrentUserProvider`, C8 `PermissionChecker` — closing real gaps the architecture review found). C1–C6 below are synced to match; consult the spine directly for C7/C8 and for any future signature changes — this file is no longer where those land.

## C1 — AccessResolver

```
resolveAudience(viewerId, subjectId) → {
  role: Self | ReportingLine | ProjectLine | PP | Colleague | SharedLink | FullAccess,
  sections: Map<SectionId, "R" | "RW" | "none">   // in-process only — see below
}
```

**Role enum updated 2026-08-26 (`project-requirements-v2.md` §2–3):** `ManagerLine` splits into `ReportingLine` (reports-to ∪ department-management closures, full Manager grant) and `ProjectLine` (project-assignment closure only, the narrower grant per `access-model.md` Rule 2 — no S2/S3, S5 limited to CV+certificates). `HRAdmin` is **removed** from this enum — HR Admin is a functional role that grants no data access on its own (see C8) and must never appear as a resolved audience here. `FullAccess` replaces it as the "sees everything" terminal case, resolved from the dedicated full-access grant (never from holding any functional role).

Backed by the union of the reports-to closure, the department-management closure, and the project-assignment closure for Manager access (three graphs, not two — `SPEC.md` Constraints), and the PP-assignment + HR-line closure for PP access. Must be called server-side on every request touching profile-scoped data — never cached client-side, never assumed stable across a session. Owner: CAP-1 (Epic 1). An early stub can return a fixed permissive role for seeded dev/test data so other tracks can start building against the contract immediately. **Wire-safety rule (architecture spine AD-2):** `Map` is the in-process type only; any controller returning access-resolution results over HTTP must convert `sections` to a plain `Record<SectionId, "R"|"RW"|"none">` first — `Map` doesn't survive JSON/OpenAPI serialization.

## C2 — FieldRegistry

```
defineField(name, type, visibility) → fieldId
setValue(employeeId, fieldId, value)
```

Plus a query interface that treats built-in, derived, and custom fields uniformly for filtering/sorting/columns. Owner: CAP-2 (Epic 3, custom fields). Everything downstream — All Employees, exports, saved views, dashboards — filters through this registry's query shape rather than hand-rolling field access.

## C3 — ProjectAssignment (internal, pre-integration)

```
{ employeeId, projectId, pmId, dmId, startDate, endDate, confirmed: boolean, confirmedAt: DateTime }
```

Internally-owned and queryable independent of the timetracker being live, so CAP-1's access model doesn't block on CAP-13's integration. Owner: CAP-1 initially (seeded data); CAP-13 later becomes the *writer* of this same table from the real timetracker feed. Nothing downstream changes when that swap happens. **This table's only legitimate writers are CAP-1's seed path and CAP-13's timetracker sync — CAP-6 (Resourcing) reads it but never writes to it** (a resourcing approval does not create a project assignment; see `SPEC.md` Constraints). **`confirmed`/`confirmedAt` (added during architecture review, AD-8):** Manager access grants only from rows where `confirmed = true` **and** `confirmedAt` is within a bounded freshness window — a sticky one-time-confirmed row must age out during a prolonged sync outage, not grant access indefinitely. **Concrete numbers ratified 2026-08-26 (`decisions.md` D3):** a confirmed row grants access within 15 minutes of the real-world change; the freshness window that ages a row out on sync failure is 4 hours — `AccessResolver`'s freshness-window config default should converge on these, superseding AD-8's placeholder "sync interval × 2." **Clock semantics refined 2026-08-26 (`decisions.md` D19, edge-case-hunter):** `confirmed`/`confirmedAt` are set on the **first** successful sync that observes the row — never requiring 15 continuous minutes of sustained success, since a flapping sync must still be able to confirm a genuinely new assignment. The 4-hour withdrawal clock accumulates **total failed sync time within a rolling window**, not a continuous-failure streak that resets on any brief recovery — a flapping sync whose cumulative failed time exceeds 4 hours still withdraws access even if no single outage lasted that long.

## C4 — Timeline Event Writer

```
recordTimelineEvent(employeeId, type, effectiveDate, oldValue, newValue, source: "system" | "manual", authorId?)

markSystemWriteSkipped(manualEventId, skippedAt)
```

Anything that changes a tracked field (grade, position, department, FTE↔subcontractor, extended leave, mentorship pair start/end, joining) calls `recordTimelineEvent` instead of writing timeline rows itself. Owner: CAP-7 (Epic 7). An early stub can log-and-no-op, so CAP-1 (profile edits), CAP-9 (mentorship), and CAP-13 (timetracker-derived leave/FTE changes) can all call it without waiting on Epic 7's UI to exist. **Typing rule (architecture spine AD-2):** `oldValue`/`newValue` are always the raw typed value for `type` (an enum value, an ISO date, a boolean) — never a caller-pre-formatted string; the timeline UI formats per `type` at render time. When a system-sourced write would overwrite a manual correction in the same effective window (D2), the history write is suppressed and `markSystemWriteSkipped(manualEventId, skippedAt)` attaches skip metadata to the **existing manual** `TimelineEvent` — per `EXPERIENCE.md`'s Career Timeline affordance; never a silent no-op, and never a separate timeline row for the skip itself.

## C5 — External identity mapping

```
external_identities: {
  system: "peopleforce" | "timetracker",
  externalId,
  employeeId,
  supersededBy?
}
```

A dedicated mapping table keyed by `(system, externalId)` rather than email — see `decisions.md` D8 — supporting re-hires and identity changes via an explicit `supersededBy` pointer. Schema shape is a starting point, ratified early; CAP-13's PeopleForce/timetracker investigation owns validating/refining it, but any story can reference "this employee's PeopleForce candidate ID" from the start without redesigning later.

## C6 — Action item creation

```
createActionItem({
  assigneeId, authorId, title, description?, dueDate, link?,
  source: "manual" | "campaign", campaignId?
}) → ActionItem
```

Owner: CAP-4 (Epic 4). CAP-10's campaign-activation story calls this directly — recommended resolution is assigning Epic 4 and Epic 10 to the same developer in whichever wave, so this "dependency" never produces a cross-developer wait.

## C7 — CurrentUserProvider, C8 — PermissionChecker

Two contracts this document never anticipated — the architecture review found both gaps live (no sanctioned way for a controller to learn who's calling, and no contract for functional-role feature-permission checks distinct from data-visibility checks). Both are owned by `access`/`auth` and fully specified in `ARCHITECTURE-SPINE.md` AD-2's table — not duplicated here to avoid two sources of truth for a fast-moving pair of contracts.

## C9 — OrgRelationshipWriter *(new, `project-requirements-v2.md` §2.1/§3.4)*

```
changeManager(actorId, subjectId, newManagerId) → JournalEntry
changePeoplePartner(actorId, subjectId, newPpId) → JournalEntry
changeDepartment(actorId, subjectId, newDepartmentId) → JournalEntry
changeDepartmentManager(actorId, departmentId, newManagerId) → JournalEntry
```

The **only** legitimate write path for the four access-switch fields (manager, PP, department, department's manager) — never writable through the general profile-edit path that backs S1's `RW` grant. Every method: (1) checks `PermissionChecker.hasPermission(actorId, "change_organisational_relationships")` (C8), (2) rejects self-assignment (`actorId === newManagerId`/`newPpId`, or the actor not already entitled to manage the target department), (3) writes the change so it's visible on the very next request (no cache to invalidate beyond C1's per-request resolution), and (4) calls C10 to journal it. Owner: `access` (Epic 1). CAP-14's departure re-parenting flow calls `changeManager`/`changeDepartmentManager` directly rather than re-implementing the switch-write path.

**Four write-time guards, added 2026-08-26 (`decisions.md` D15, edge-case-hunter):**

- `changeDepartmentManager` additionally rejects a `newManagerId` who is a current member of `departmentId` or any nested sub-department (no self-managed department).
- `changeManager` additionally rejects a `newManagerId` already a descendant of `subjectId` in the reporting chain (no cycles).
- Every method takes an expected-version/timestamp precondition and rejects with a conflict error (not a silent overwrite) if the field changed underneath it since the caller last read it.
- `changeDepartmentManager` rejects a clear (`newManagerId = null`) unless the same call also designates a replacement — never leaves a department headless via the direct-edit path.

## C10 — RelationshipJournal *(new, `project-requirements-v2.md` §3.4)*

```
record(actorId, subjectId, kind: "manager" | "people_partner" | "department" | "department_manager" | "full_access_grant" | "full_access_revoke" | "shared_link_access", before, after) → JournalEntry
readFor(subjectId, readerId) → JournalEntry[]
```

A narrow, dedicated log — not the general application audit log. `record` is called by C9 (relationship changes), the full-access grant/revoke flow, and CAP-1's shared-link access path (profile sharing, §4.8); never by ad hoc call sites. `readFor` is gated to full-access holders and the subject's *current* manager and PP (resolved via C1 at read time, not snapshotted). Owner: `access` (Epic 1).

## C11 — EmploymentStatusService *(new, `project-requirements-v2.md` §4.16, Epic 14)*

```
recordDeparture(actorId, subjectId, effectiveDate, reason) → EmploymentStatusRecord
```

Blocks with a structured error if `subjectId` still holds live Manager or PP responsibility over anyone (the caller UI then offers the one-click re-parent-to-own-manager flow via C9, per §4.16), **or if `subjectId` is the sole remaining full-access holder** (the grant must be transferred first — `decisions.md` D16). On the effective date, cascades in one transaction: profile → read-only + dropped from the default employee list (still filterable); open `ActionItem`s → `cancelled` with reason `"departed"` (distinct from an author's own cancellation, C6) **as a no-op if the item is already cancelled**, so it never races a concurrent author-cancellation; active `MentorshipPair`s → ended with a system-generated closure note that bypasses D6's mandatory-closure-note gate, **as a no-op if the pair is already ended** (covers both members of a pair departing on overlapping dates); every shared link `subjectId` created → explicitly revoked, as its own named step, not left as an implicit consequence of C1's per-view re-clamp (D14); the person's `User` account → deactivated; every access grant they held → revoked immediately (falls out naturally once C1 re-resolves against the now-`dismissed` status and severed relationships). Owner: `access`/`directory` boundary (Epic 14 — see `SPEC.md` CAP-14). Analytics (good-to-have) and All-Employees' default-list exclusion both read this table rather than deriving their own departure definition. **Idempotency and the sole-holder check added 2026-08-26 (`decisions.md` D16/D17, edge-case-hunter).**

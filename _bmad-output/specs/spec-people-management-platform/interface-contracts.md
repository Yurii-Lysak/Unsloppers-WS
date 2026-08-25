# Interface Contracts

Companion to `SPEC.md` — see there for capability context. Minimum interface surface — signatures only, not full designs — that lets a second developer start coding against a contract before the first developer finishes implementing it. Sourced from `PRD_parallel_delivery_plan.md` §2; each already has a natural owner among the epics in `SPEC.md`'s Capabilities.

**Superseded for exact signatures as of 2026-08-21:** the architecture spine (`ARCHITECTURE-SPINE.md`, AD-2's contract table) is now the authoritative, review-ratified source for C1–C6's precise shapes, plus two contracts this document never anticipated (C7 `CurrentUserProvider`, C8 `PermissionChecker` — closing real gaps the architecture review found). C1–C6 below are synced to match; consult the spine directly for C7/C8 and for any future signature changes — this file is no longer where those land.

## C1 — AccessResolver

```
resolveAudience(viewerId, subjectId) → {
  role: Self | ManagerLine | PP | Colleague | SharedLink | HRAdmin,
  sections: Map<SectionId, "R" | "RW" | "none">   // in-process only — see below
}
```

Backed by the union of the reports-to closure and the project-assignment closure for Manager access, and the PP-assignment + HR-line closure for PP access. Must be called server-side on every request touching profile-scoped data — never cached client-side, never assumed stable across a session. Owner: CAP-1 (Epic 1). An early stub can return a fixed permissive role for seeded dev/test data so other tracks can start building against the contract immediately. **Wire-safety rule (architecture spine AD-2):** `Map` is the in-process type only; any controller returning access-resolution results over HTTP must convert `sections` to a plain `Record<SectionId, "R"|"RW"|"none">` first — `Map` doesn't survive JSON/OpenAPI serialization.

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

Internally-owned and queryable independent of the timetracker being live, so CAP-1's access model doesn't block on CAP-13's integration. Owner: CAP-1 initially (seeded data); CAP-13 later becomes the *writer* of this same table from the real timetracker feed. Nothing downstream changes when that swap happens. **This table's only legitimate writers are CAP-1's seed path and CAP-13's timetracker sync — CAP-6 (Resourcing) reads it but never writes to it** (a resourcing approval does not create a project assignment; see `SPEC.md` Constraints). **`confirmed`/`confirmedAt` (added during architecture review, AD-8):** Manager access grants only from rows where `confirmed = true` **and** `confirmedAt` is within a bounded freshness window — a sticky one-time-confirmed row must age out during a prolonged sync outage, not grant access indefinitely.

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

# Epic 7 Context: Career Timeline

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver a system-generated event log on each employee profile (S9) that stays current as tracked employment facts change, while allowing People Partners and Unit Managers to backfill history or correct wrongly inferred events. The timeline is never a separately maintained record — it is derived from (or reconciled with) underlying change sources. Manual corrections must survive later syncs: a system write that would overwrite a human fix is skipped and surfaced, not applied silently.

## Stories

- Story 7.1: Auto-Generate Timeline Events
- Story 7.2: Manual Timeline Edits
- Story 7.3: Resolve Conflicts Between System-Generated and Manually-Edited Timeline Events

## Requirements & Constraints

- The system auto-generates one timeline event per tracked change across six types: joining, grade change, position change, department change, FTE↔subcontractor transition, extended leave, and mentorship pair start/end. All writes go through a single shared event writer — no per-feature bespoke paths.
- Each auto-generated event records type, old value, new value, effective date, and a system-generated source flag. Values are stored in raw typed form; formatting belongs to the UI at render time.
- Holders of the *edit the career timeline* functional permission (launch default: UM and PP) may manually add, edit, or delete events for backfill or correction. Manual entries are tagged as manually entered and sorted into chronological position — not merely appended.
- Deletion is soft-delete with an audit trail. A departure is never a timeline event; employment status lives elsewhere.
- Write access to S9 is narrowed to UM and PP only — the one documented exception within Manager line's section-level RW. A DM with project-line Manager access but who is not the assigned UM or PP is rejected server-side on any write attempt.
- When a later system-generated write would cover the same change window as an existing manual correction, the system must not overwrite or duplicate the manual entry. It skips the write, attaches skip metadata to the manual event, and leaves exactly one row at the corrected date. Unrelated manual entries must not suppress future, unrelated system events.
- A failed timeline write (e.g. transient error during timetracker sync) must not crash the application or block other profile functionality. The underlying fact change (such as extended leave in S10) still lands; the missing timeline write is logged for retry.

## Technical Decisions

- The `timeline` module owns **C4 `TimelineEventWriter`** (`recordTimelineEvent`, `markSystemWriteSkipped`) and registers **SectionProvider(S9)** via the provider registry. Other modules (`access`, `mentorship`, `integrations`) consume C4 through contracts — they never write timeline rows directly.
- **AD-7** couples the four temporal history tables (`GradeHistory`, `PositionHistory`, `DepartmentHistory`, `EmploymentTypeHistory`) to C4 via a Prisma Client Extension: every write to those tables auto-fires the timeline writer; no service method may bypass this path. `EmploymentStatusHistory` is explicitly excluded — departures never produce timeline events.
- On conflict (D2), the extension suppresses the incoming system-sourced history write and calls `markSystemWriteSkipped(manualEventId, skippedAt)` on the existing manual event. Skip metadata never becomes a separate timeline row.
- Story 1.20 (Epic 1) establishes the history tables and extension wiring with C4 stubbed until this epic ships the real implementation.

## UX & Interaction Patterns

- S9 renders on the Employee Profile as a vertical chronological list (`career-timeline` component). Each entry shows a typed icon and a secondary Badge reading "system" or "manual" — equal visual weight, no implied trust hierarchy.
- When a system write is skipped due to a manual correction, the manual entry shows a small info affordance: "A system update was skipped here — {date}". No invisible skip and no extra row.
- PP/UM see add/edit/delete affordances on each entry; all other viewers see the timeline read-only.

## Cross-Story Dependencies

- **Story 1.20** (Epic 1) must be in place first — effective-dated history tables and the Prisma extension that structurally couples them to C4.
- **Epic 1** access resolution and functional-role permission gating (`edit the career timeline`) gate every read/write on S9.
- **Epic 9** (Mentorship) triggers pair start/end events through C4.
- **Epic 13** (Timetracker integration) is a source for extended-leave changes; timeline write failure there must fail soft without blocking S10 or other profile surfaces.
- Story 7.3 depends on 7.1 (auto-write path) and 7.2 (manual edit path) being complete.

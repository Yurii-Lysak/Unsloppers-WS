# Epic 13 Context: External Integrations — Timetracker & PeopleForce

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Wire the platform to two external systems: TimeTracker (load-bearing for leave display and project-based access) and PeopleForce (candidate/vacancy data for resourcing, with a fallback outbound-link path). Leave and project feeds must degrade gracefully on outage — display fails soft (S10/S11 show "temporarily unavailable"), while project-derived Manager access fails safe (Story 13.2). Cross-system identity must never rely on email inference alone.

## Stories

- Story 13.1: Integrate Timetracker Leaves API
- Story 13.2: Integrate Timetracker Projects & People API (Permission-Critical)
- Story 13.3: Investigate & Integrate PeopleForce Candidate/Vacancy API with Fallback Link
- Story 13.4: Resolve Cross-System Identity (PeopleForce ↔ Platform ↔ Timetracker)

## Requirements & Constraints

- Leave data (S10) and project/PM/DM assignment data (S11, feeding ProjectAssignment) are pulled from TimeTracker's test environment against the seeded bootcamp population (FR-58).
- On sync failure, S10/S11 display fails soft: last-known data may be served behind a visible freshness/unavailable indicator; the profile and list must not crash (NFR-4, AD-8 display axis).
- Project-derived Manager access fails safe on prolonged sync outage (Story 13.2 — separate code path from display; 13.1 must not touch access logic).
- Colleague viewers see S10 **dates only**; leave **type** is withheld server-side (access matrix Rule 4).
- Self-service S10 includes a link out to TimeTracker for leave management; no in-platform balances (FR-19).
- External candidates on resourcing carry stored PeopleForce ID + link regardless of full integration (FR-59 — Story 13.3).
- Every external record resolves via explicit `(system, externalId) → employeeId` mapping with `supersededBy` for re-hires — never email-only match (FR-60, C5).

## Technical Decisions

- **`integrations` module** owns C3 writer (13.2), C5 implementation, and TimeTracker/PeopleForce clients (ARCHITECTURE-SPINE CAP-13, AD-8).
- **No dedicated Leaves REST endpoint** in the checked-in OpenAPI spec (`docs/api-external-openapi.json`); leave types arrive via `POST /api/accounting/report` as `DayStatus` on `WorkingDay` rows — Story 13.1 derives normalized leave periods from this feed unless investigation discovers a new endpoint.
- **`TimetrackerService`** already exists (`services/backend/src/modules/timetracker/`) with typed accounting/talents clients, 15s timeout, and `TimetrackerApiError` — currently seed-only; Epic 13 enables runtime use.
- **Two API keys** (`TIMETRACKER_ACCOUNTING_API_KEY`, `TIMETRACKER_TALENTS_API_KEY`) — non-interchangeable per OpenAPI security schemes.
- **S10 SectionProvider** registers via `@RegisterProvider('section', 'S10')`; consumers resolve through `ProviderRegistryService` (AD-3), not direct module imports.
- **Display freshness vs access confirmation** are independent axes (AD-8): implementing S10 unavailable state must not share code with `ProjectAssignment.confirmed` logic (13.2).
- **Extended leave** transitions should call C4 `TimelineEventWriter` when detected; timeline write failure must not block S10 display (Epic 7 coupling rule).

## UX & Interaction Patterns

- S10/S11 show inline **"Temporarily unavailable"** when the feed is unreachable; remainder of profile/list stays interactive (EXPERIENCE.md).
- Leave type visible to Manager/PP/Self; hidden from Colleague (dates still shown).

## Cross-Story Dependencies

- **13.1** supplies S10 data only; does not alter the access matrix. Depends on C5 mapping (bootstrap via seed + minimal table acceptable before 13.4 finalizes schema). Blocked on **Story 1.6** (`ProfileAssemblerService`) for end-to-end profile assembly; provider + sync layer can land first.
- **13.2** becomes the live writer for C3 `ProjectAssignment` with `confirmed`/`confirmedAt` — independent of 13.1's display path.
- **13.3** time-boxed PeopleForce investigation; Epic 6 never blocks on it (fallback link).
- **13.4** validates/refines C5 against real investigation findings; 13.1 may ship with seed-populated mappings before 13.4 completes.

# Epic 8 Context: CDS: Career Development System

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Give managers, PPs, and employees a single registry and hub for career development data — a link to the person's current skills matrix, a running log of past assessments, and Individual Development Plan (IDP) tracking — without the system ever acting as an assessment engine itself (no scoring, rating, or judgment logic lives here, only records and links). This matters because skills-matrix files and assessment results are maintained externally; CDS's job is to keep every profile pointing at the right file and keep a trustworthy history of what was assessed, when, and what came of it, including plans still in progress.

## Stories

- Story 8.1: Skills Matrix Link and Assessment Log
- Story 8.2: IDP Records
- Story 8.3: Manager/PP Maintain Assessments and Conclusions
- Story 8.4: Filter by Assessment Recency and Open IDP

## Requirements & Constraints

- The skills-matrix link resolves from the person's department entity + position through a centrally maintained dictionary — never a free-text department string. Updating the dictionary or renaming the owning department changes what every affected profile resolves to, with no per-employee record edited.
- Assessment log entries carry date, assessor, a result-file link, and a text conclusion. The assessor field may be free text or a person reference (implementer's choice — no further constraint). No numeric score or rating field exists anywhere in the model; that boundary is intentional (registry, not an assessment engine).
- Adding a new assessment entry always appends; it never modifies prior entries. Editing a conclusion changes only that field on the targeted entry — date/assessor/result-link stay unchanged and no new log entry is created.
- An IDP record holds description, deadline, and an external file link. Only a manager or PP can create/update it; the employee's only control over it is a "mark complete" checkbox, which records today's completion date. Employees cannot create, edit, or delete IDP records.
- A completed IDP cannot be reopened by a manager or PP by default.
- All Employees directory filtering must support last-assessment-date (before/after/between) with "never assessed" as its own distinct, separately selectable option — never an empty value that silently falls out of results — plus has-open-IDP (yes/no).
- Write access to assessment/conclusion records is permission-gated (the *maintain CDS records* feature permission) and enforced server-side regardless of what the UI exposes; someone without a Manager/PP relationship to the subject, or without the permission, must be rejected even via direct API calls.
- CDS section visibility follows the resolved profile audience: Colleague viewers see no trace of the CDS section at all, and CDS-derived directory filters return nothing for anyone the viewer lacks Manager/PP access to.

## Technical Decisions

- Owned by its own backend module (`cds`), consistent with the general one-module-per-capability layout.
- The matrix-link lookup reads the department hierarchy exclusively through the shared department-directory contract (never a private copy of the department tree or a direct table read) — this is what makes "update once, resolves everywhere" work.
- CDS participates in the cross-module provider registry twice: as a `SectionProvider` (assembles the CDS section shown on a profile) and as a `FieldProvider` (exposes last-assessment-date and has-open-IDP as derived, filterable fields). The Directory module queries these through the registry rather than reading CDS's tables directly — CDS is the data owner, Directory is the query surface only.
- Data shape: an employee has at most one CDS record, which owns many assessment entries and many IDP entries (one-to-many each, independent of each other).
- A registration gap (a section/field that should have a provider but doesn't) must surface as an explicit "temporarily unavailable" state at first call, never a silent omission that reads the same as "not granted."

## UX & Interaction Patterns

- CDS renders as one Profile Section Card in the profile's fixed section ordering. Where the viewer has write access the card shows inline edit affordances directly on it; where access is read-only, no edit affordance renders at all (absent, not merely disabled).
- The IDP row displays a positive status badge once the employee marks it complete.
- When a resourcing candidate submission auto-generates a shared link for the reviewing manager, CDS is one of the sections included in that link's scope by default.

## Cross-Story Dependencies

- Story 8.4 depends on Story 8.1's field-provider registration and on the Directory module's field-registry query surface (Epic 3) to actually expose the filters in All Employees.
- Story 8.2's employee-facing "mark complete" checkbox is delivered as part of the Self-Service surface (Epic 2), routed to write into CDS for that one action only.
- All four stories depend on the profile access-resolution work (Epic 1) to determine, per viewer, whether the CDS section and its filters are visible or writable at all.

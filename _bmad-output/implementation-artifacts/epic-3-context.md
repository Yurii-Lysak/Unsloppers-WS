# Epic 3 Context: All Employees Directory & Custom Fields

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver the employee directory experience: a single list page whose columns, filters, and data vary by viewer access, backed by a real FieldRegistry (C2) that treats built-in, derived, and custom fields uniformly. Custom fields are defined at runtime without schema migrations, with per-field visibility enforced everywhere data can leak.

## Stories

- Story 3.1: Sortable, Filterable Employee List — **done**
- Story 3.2: Define Custom Fields at Runtime — **done** (real FieldRegistry on backend `main`)
- Story 3.3: Inline Editing on the List — **in progress**
- Story 3.4: Saved and Shared Views
- Story 3.5: Export to Excel
- Story 3.6: Colleague Mode of the List
- Story 3.7: All Employees List Performance at 500+ Records Under 2 Seconds

## Requirements & Constraints

- Custom fields support text, number, date, boolean, single-select, and multi-select types.
- Each field has a visibility level: management (default), employee, or colleague.
- Visibility must hold across profile S16, list columns/filters, and exports — no inferring hidden values via filter side-channels.
- Field creation requires the `manage_custom_fields` functional permission (C8 PermissionChecker).
- New fields are immediately usable — no publish step or deploy.
- NFR-2: list with 500+ records and arbitrary filters must respond within 2 seconds (validated in 3.7).

## Technical Decisions

- C2 FieldRegistry lives in the `directory` NestJS module; real implementation replaces the Wave-0 stub via DI override (same pattern as AuthModule's CurrentUserProvider).
- AD-6: one `CustomFieldDefinition` + one `CustomFieldValue` table with typed nullable columns; rows created lazily on first write; unused columns are SQL NULL.
- FieldRegistry.query is the only place that branches on field type — consumers never special-case custom vs built-in.
- AccessResolver (C1) stub is deny-by-default until Track A delivers the real implementation; visibility logic must still be wired correctly for when it lands.
- AD-1: no feature-to-feature imports; directory depends only on contracts and registry.

## Cross-Story Dependencies

- Story 3.2 blocks 3.1 and all downstream field-visibility enforcement.
- Story 3.1 depends on 3.2 (FieldRegistry) and AccessResolver (stub → real mid-wave).
- Story 3.6 coordinates with Track A Story 1.8 (colleague whitelist) — same rule, independent enforcement.

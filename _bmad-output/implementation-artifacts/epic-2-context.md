# Epic 2 Context: Self-Service

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Give every employee a self-service view of their own profile: read access to most sections, write access to a narrow, explicitly-scoped subset (S1 photo, S2 personal contacts, S3 emergency contacts, S5 certificate upload, S12 IDP-complete checkbox, S13 open-to-mentoring flag, S14 own action-item completion), with risk (S6) and any unflagged S7 management note permanently invisible regardless of any other role the viewer holds. Section access is resolved per-relationship, not per-user — an employee who is also a manager/PP elsewhere never inherits that RW when viewing their own profile.

## Stories

- Story 2.1: View Own Employment Summary
- Story 2.2: Edit Own Personal and Emergency Contacts
- Story 2.3: Upload Photo and Certificates
- Story 2.4: View Own Timeline, Leaves, Projects, CDS and Mentorship
- Story 2.5: View Shared Feedback, Flagged Notes and Own Action Items

## Requirements & Constraints

- `AccessResolver` (C1) already resolves the Self audience's per-section grant map correctly (Stories 1.1–1.5) — S4/S9/S10/S11/S12 `R`, S2/S3/S13 `RW`, S1 `R` except the photo exception, S6/S15 `none`. Section providers must trust this grant map and never re-derive or widen it from another simultaneously-held role.
- S6 (risk level, trend, history) is `—` for Self with no exception anywhere — profile, dashboards, notifications.
- Leave balances never render in-platform; the timetracker is the only place balances live — S10 shows leave dates and links out for management.
- Feedback (S8) and management notes (S7) are filtered per-record for Self by an explicit visibility flag, not a section-level toggle — an unflagged record is entirely absent, not shown-but-redacted.
- A section with no value set for a given field must render an explicit empty/placeholder state — never a silently omitted field or a missing section.
- Self write access on S12/S13/S14 is narrowly scoped (IDP-complete checkbox, open-to-mentoring flag, action-item completion) — no other field on those sections is writable for Self.
- Compensation/salary data does not exist on the profile at all.

## Technical Decisions

- `ProfileAssemblerService` + `SectionProvider` registry (Story 1.6) already wires C1 resolution to per-section providers; each Epic 2 story adds one `@RegisterProvider('section', Sx)` provider following the `IdentitySectionProvider`/`ProjectsSectionProvider` pattern — no assembler changes needed.
- AD-7: grade, position, department, and employment type are effective-dated history tables (`GradeHistory`, `PositionHistory`, `DepartmentHistory`, `EmploymentTypeHistory`); the current value is the row where `effectiveTo IS NULL`, resolved via the existing `currentHistoryValue()` helper (`directory/employee-query.helpers.ts`) — never re-implemented per-section. Full change history is S9's job (career timeline, Epic 7); other sections show current value only, never a duplicated history list.
- Any Self-scoped field outside AD-7's five-dimension temporal set (e.g. seniority, English level, probation status, contract type) is a plain current-value column — AD-7 explicitly does not extend to it.
- UX-DR7 (Profile Section Card): RW vs R-only sections are visually distinguished by the presence/absence of an edit affordance; a pure-read section (like S4) renders with no edit control at all rather than a disabled one.
- UX-DR16: empty/placeholder states must not rely on color alone; UX-DR17: every new label is an i18n key under `employeeProfile.sections.*` in `locales/en/translation.json`, never hardcoded copy.

## Cross-Story Dependencies

- Story 2.1 has no dependency on other Epic 2 stories and is the first to exercise a purely read-only section provider — later stories reuse its pattern for their own read-only fields.
- Story 2.4 depends on Epic 7 (career timeline), Epic 9 (mentorship) for S13, and Epic 8 (CDS) for S12's IDP data; Story 2.5 depends on Epic 1's S7/S8 visibility-flag mechanics and Epic 11 (feedback). Per the sprint plan, Epic 2 is scheduled last within its track because it consumes these other epics' outputs.
- Story 2.2 (S2/S3 write) and Story 2.3 (S1 photo / S5 certificate write) are independent of each other and of 2.1/2.4/2.5.

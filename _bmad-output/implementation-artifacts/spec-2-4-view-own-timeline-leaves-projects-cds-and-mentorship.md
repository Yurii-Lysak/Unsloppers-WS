---
title: 'View Own Timeline, Leaves, Projects, CDS and Mentorship'
type: 'feature'
created: '2026-09-10'
status: 'draft'
review_loop_iteration: 1
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-2-context.md'
  - '{project-root}/_bmad-output/specs/spec-people-management-platform/access-model.md'
  - '{project-root}/services/backend/AGENTS.md'
  - '{project-root}/services/frontend/AGENTS.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Self already reaches S9 (timeline), S10 (leaves), S12 (CDS/IDP-complete) and S13 (mentorship flag) end-to-end today via the generic `SectionProvider` registry and already-built self-write endpoints from Epics 7/8/9/13 — nothing there is actually broken. Two real gaps remain: S11's `ProjectsSectionProvider` is still Story 1.6's name-only stub even though `access-model.md` specifies Project/PM/DM/period for every R-holding audience, and the S10 frontend renderer never surfaces `manageLeaveUrl` or distinguishes AD-8's fail-soft `unavailable` sync state from a true empty list. Nothing yet exercises the Self path end-to-end in tests, which is why this story is still `backlog`.

**Approach:** Enrich `ProjectsSectionProvider` with PM/DM display names and period from the already-populated `ProjectAssignment` rows (Colleague keeps the existing name-only carve-out); extend the S10 renderer with the manage-leave link and a distinct unavailable-state message; add Self-viewer test coverage across S9–S13. S9/S12/S13 backend and frontend are confirmed correct for Self already — no changes there beyond tests.

## Boundaries & Constraints

**Always:**
- S11 enrichment applies to every audience holding `R` on S11 (Self/ReportingLine/PP/ProjectLine per `access-model.md`), not a Self-only branch — the provider has no per-role logic today besides the existing (currently dead) Colleague check.
- `pmId`/`dmId` resolve to employee display names before leaving the provider — never expose raw employee IDs in the S11 response; reuse whatever shared id→displayName lookup `mentorship-section.provider.ts` / `identity-section.provider.ts` already call for mentor/manager names, don't reimplement.
- S10's renderer shows the distinct "leave data unavailable" state whenever `availability === 'unavailable'`, never conflating it with the true-empty state, and renders `manageLeaveUrl` as a link only when non-null (Self only, per existing backend).
- Every new label (PM/DM/period, manage-leave link, unavailable message) is a new i18n key under `employeeProfile.sections.*` (UX-DR17) — never hardcoded copy.
- This story adds no new write endpoints — S12 IDP-complete and S13 open-to-mentoring stay exactly as already implemented.

**Ask First:** Enriching `ProjectsSectionProvider` is the first time real PM/DM identity leaves the S11 stub for ReportingLine/PP/ProjectLine audiences, not just Self — confirm this is acceptable before implementation (it matches `access-model.md`'s existing R grants for those roles, but it's a behavior change beyond this story's Self-facing AC).

**Never:**
- No rebuild of S9/S10/S11 into dedicated Card components this story (they stay lightweight inline renderers, matching current code) — S12/S13 already have Card components and are untouched.
- No changes to `timeline-section.provider.ts`, `leaves-section.provider.ts` (S10 backend), `cds-section.provider.ts`, `idp-records.controller.ts`, `mentorship-section.provider.ts`, or `mentorship.controller.ts` write logic — all confirmed already correct for Self.
- No changes to AD-8's confidence-gating / sync machinery (CAP-13 scope) — this story only surfaces the existing `availability: 'unavailable'` state distinctly in the UI.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| SELF_VIEW_TIMELINE | Employee opens own profile | S9 renders own timeline events read-only, no edit affordance | — |
| SELF_VIEW_LEAVES_OK | Timetracker sync healthy | S10 shows leave date ranges + a "manage leave" link to `manageLeaveUrl` | — |
| SELF_VIEW_LEAVES_DEGRADED | `availability: 'unavailable'` | S10 shows a distinct "leave data temporarily unavailable" message, not the generic empty state | — |
| SELF_VIEW_PROJECTS_ENRICHED | Employee has an active `ProjectAssignment` | S11 shows project name, PM name, DM name, and period (start–end) | — |
| COLLEAGUE_VIEW_PROJECTS_NAME_ONLY | Colleague-role viewer with no relationship views S11 | Only the project name is present in the payload — PM/DM/period absent server-side, not just hidden client-side | — |
| SELF_COMPLETE_IDP | Employee ticks "complete" on their own open IDP record | Completion date recorded and displayed; assessment conclusion/IDP deadline stay non-editable for Self | 403 on any other field write |
| SELF_TOGGLE_MENTORING | Employee flips their own "open to mentoring" switch | Flag updates immediately, visible on their own profile | — |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/access/projects-section.provider.ts:20-33` -- enrich `getSection` with PM/DM display name + period from `ProjectAssignmentDto` (`pmId`/`dmId`/`startDate`/`endDate` already populated, `contracts/project-assignment.contract.ts:17-26`); keep the Colleague branch name-only. No batch id→displayName helper actually exists yet (see next bullet), so the constructor gains a new dependency -- update `access.module.ts` wiring and the existing `__tests__/projects-section.provider.spec.ts` `TestingModule.providers` array (it currently only provides `ProjectAssignment`) accordingly
- `services/backend/src/modules/access/entities/projects-section.entity.ts` -- extend `ProjectNameEntryEntity` with `pm`, `dm` (display names, nullable -- null when a `pmId`/`dmId` no longer resolves to a current employee, mirroring `identity-section.provider.ts`'s try/catch mentor-omission pattern rather than throwing), `startDate`, `endDate`; make all four new fields optional (`?`), since Colleague entries omit them entirely rather than nulling them
- Id→displayName resolution has no existing reusable helper to call: `mentorship-section.provider.ts` does no name resolution itself (it delegates to `mentorship.service.ts`), and that service's private `displayName(user: {name, email})` (`mentorship.service.ts:185-187`) -- like `identity-section.provider.ts`'s `relationDisplayName` -- only formats a `{name, email}` object from an already-loaded Prisma relation; neither takes a bare id. Write a new batch lookup (e.g. `prisma.employee.findMany({ where: { id: { in: [pmId, dmId] } }, include: { user: { select: { name: true, email: true } } } })`) reusing that same `name?.trim() || email` formatting rule
- `services/backend/src/modules/access/project-assignment.service.ts:38-43` -- `listByEmployee` currently returns every row unfiltered (no `confirmed`/`endDate` filter) -- see Spec Change Log Q2 for whether S11 enrichment must filter to active/confirmed rows before display; sort results by `startDate` ascending for deterministic ordering when an employee has multiple concurrent assignments
- `services/backend/src/modules/access/__tests__/projects-section.provider.spec.ts` -- the existing fixture mocks `listByEmployee` with only `{ projectId, pmId, dmId }`, missing `employeeId`/`startDate`/`endDate`/`confirmed`/`confirmedAt` the real `ProjectAssignmentDto` contract requires; fill it out to the full shape before asserting period fields, keeping the existing Colleague `not.toHaveProperty('pm')` assertion passing
- `services/frontend/src/types/employee-profile.ts:245-248` -- extend `LeavesSection` with `availability: 'ok' | 'unavailable'` (the type currently omits it entirely, even though `LeavesSectionEntity` already returns it and the S10 renderer needs to branch on it)
- `services/frontend/src/types/employee-profile.ts:262-264` -- extend `ProjectsSection.projects` entries with `pm?: string | null`, `dm?: string | null`, `startDate?: string`, `endDate?: string | null` (optional, not just nullable -- Colleague entries carry only `name`)
- `services/frontend/src/pages/EmployeeProfilePage/profile-sections.tsx:213-246` -- S10 renderer: render the `manageLeaveUrl` link whenever it is non-null, independent of whether `leaves` is empty (today's `leaves.length === 0` branch would otherwise hide it for a Self viewer with no upcoming leave), and check `availability === 'unavailable'` before the empty/list branching, not nested inside it; S11 renderer: render pm/dm/period alongside name, falling back to a not-set label when `pm`/`dm` is null and to an "ongoing" label when `endDate` is null. Note: `services/frontend/e2e/employee-profile-assembly.spec.ts:40,117` already stub `S10: { accessLevel: 'R', status: 'unavailable' }` -- that `status` is the section-envelope-level `ProfileSectionUnavailable` (whole-section fetch failure), a different concept from `data.availability`; don't mistake that fixture for existing AD-8 coverage
- `services/frontend/src/locales/en/translation.json` -- `employeeProfile.sections.leaves` and `.projects` (lines 641-642) are currently bare leaf strings consumed directly by `PROFILE_SECTION_TITLE_KEYS.S10`/`.S11` (`profile-sections.tsx:98-99`); nesting new children under them would break that direct-string lookup. Restructure both into objects with a `.title` field plus the new keys (matching the `sections.employment.title` + `.fields.*` convention already used for S4, another section with no dedicated Card component) -- `leaves: { title, manageLink, unavailable }`, `projects: { title, pm, dm, period, notSet, ongoing }` -- and update `PROFILE_SECTION_TITLE_KEYS.S10`/`.S11` to append `.title`
- `services/backend/src/modules/access/__tests__/` -- extend/add unit coverage for enriched S11 provider (Colleague vs. R-holding audiences)
- `services/backend/test/employee-profile.e2e-spec.ts` -- add Self-viewer coverage across S9/S10/S11/S12/S13 read + S12/S13 self-write
- `services/frontend/e2e/employee-profile-assembly.spec.ts` (or a new Playwright spec) -- add frontend coverage for the new S10/S11 renderer branches (manage-leave link, unavailable message, pm/dm/period); today only `typecheck`/`lint`/`build` and the API-level backend e2e spec would catch a regression here, and none of them exercise the actual rendered output
- Confirmed already correct, no code changes needed: `timeline/timeline-section.provider.ts` (S9), `integrations/leaves-section.provider.ts` (S10 backend, incl. `manageLeaveUrl` + fail-soft `availability`), `cds/cds-section.provider.ts` + `cds/idp-records.controller.ts:83-99` + `cds.service.ts`'s `completeIdpRecord` (S12 read + self-complete -- already 409s on re-completing a completed record and uses an atomic `updateMany` guarded on `completedAt: null` for the concurrent-completion race, so no further hardening needed there), `mentorship/mentorship-section.provider.ts` + `mentorship/mentorship.controller.ts:50-86` (S13 read + self-flag write), `EmployeeProfilePage/components/CdsSection/CdsSection.tsx` (`canCompleteIdp = audienceRole === 'Self'`), `.../MentorshipSection/MentorshipSection.tsx` (`canEditFlag = audienceRole === 'Self' && accessLevel === 'RW'`)

## Tasks & Acceptance

**Execution:**
- [x] Enrich S11 provider (per Code Map) -- closes the `access-model.md` gap blocking a real "my projects" view; includes the new id→displayName batch lookup, the DI/module wiring it requires, and fixing the existing provider unit-test fixture
- [x] Extend `types/employee-profile.ts` (per Code Map: `LeavesSection.availability`, `ProjectsSection.projects` optional fields) -- typed client
- [x] `profile-sections.tsx` S10/S11 renderers + `translation.json` restructure (per Code Map) -- closes the AC gap ("leaves with a timetracker link") without breaking the S10/S11 section-title lookup
- [x] Backend unit tests for the enriched S11 provider -- I/O matrix coverage (Colleague vs. R-holding audiences)
- [x] `employee-profile.e2e-spec.ts` -- Self-viewer coverage across S9/S10/S11/S12/S13 read + S12/S13 self-write -- confirms the already-built pieces work together end-to-end for Self
- [x] Frontend Playwright coverage (per Code Map) for the new S10/S11 renderer branches -- closes the gap where no automated check exercises the actual rendered output

### Review Findings

- [x] [Review][Patch] Make `LeavesSection.availability` optional for Colleague wire shape [`employee-profile.ts:267`]
- [x] [Review][Patch] Harden S10 wire mapping (`leaves ?? []`, coerce availability) [`profile-assembler.service.ts:201-206`]
- [x] [Review][Patch] Restore Colleague S10 degraded sync as section unavailable [`profile-assembler.service.ts:165-172`]
- [x] [Review][Patch] Treat missing audience as name-only S11 [`projects-section.provider.ts:35`]
- [x] [Review][Patch] Add PP/ProjectLine/ordering S11 unit tests [`projects-section.provider.spec.ts`]
- [x] [Review][Patch] Add Self S12 read, IDP completion display, and deadline-edit denial e2e [`employee-profile.e2e-spec.ts`]
- [x] [Review][Patch] Add Playwright coverage for leaves+link, section titles, and ended project period [`employee-profile-assembly.spec.ts`]
- [x] [Review][Defer] `confirmedAt` freshness filtering for S11 assignments — deferred, pre-existing open Q2 in spec
- [x] [Review][Defer] SharedLinkManagerDialog S10/S11 label e2e — deferred, no shared-link Playwright harness yet

**Acceptance Criteria:**
- Given I open my own profile, when each section loads, then I see my timeline (S9), leaves with a timetracker link (S10), projects (S11), and CDS (S12) as read-only, and my open-to-mentoring flag as the one writable field in S13
- Given I view S12, when I tick "complete" on my own IDP, then a completion date is recorded and displayed, and I cannot edit an assessment conclusion or IDP deadline — only the checkbox is writable for Self

## Spec Change Log

- 2026-09-10: `bmad-review` full review applied. Code Map, Tasks & Acceptance, Design Notes, and Verification corrected/expanded (see diff) to fix a real i18n key-collision bug (`sections.leaves`/`.projects` are consumed as bare title strings, so nesting new children under them would break `PROFILE_SECTION_TITLE_KEYS`), an incomplete provider-test fixture, a missing frontend `LeavesSection.availability` type field, a cited-but-nonexistent id→displayName helper, and undocumented new-DI/module-wiring and Playwright-coverage needs. Two open questions below are surfaced for the frozen Intent/Boundaries owner to resolve before or during implementation — nothing in the frozen block was edited.
- **Q1 (open):** Should S11 PM/DM/period enrichment also reach Full-Access and Shared-Link (when S11's `cfg` is enabled on the link) viewers, not just the four roles the frozen "Always" bullet names? `access-model.md` grants S11 `R` broadly and Rule 11 requires a shared link's exposure to re-clamp continuously to the creator's current access — neither is addressed by the current Self/ReportingLine/PP/ProjectLine framing.
- **Q2 (open):** Should S11 enrichment filter out ended (`endDate` in the past) or unconfirmed/stale (`confirmed: false`, or `confirmedAt` outside AD-8's freshness window) `ProjectAssignment` rows before displaying PM/DM/period, or is surfacing every historical/unconfirmed row intentional for this story? The frozen I/O matrix's `SELF_VIEW_PROJECTS_ENRICHED` scenario says "an active `ProjectAssignment`," implying filtering, but `listByEmployee` applies none today.

## Design Notes

Per Boundaries, S9/S10/S11 stay inline renderers; that also trivially satisfies UX-DR7 since none of the three are writable.

"Period" (per `access-model.md`'s S11 row) is realized as the two raw `startDate`/`endDate` ISO fields, formatted client-side into a date range (with an "ongoing" fallback when `endDate` is null) — matching how S9/S10 already format dates inline. No combined `period` field is added to the backend entity.

## Verification

**Commands:**
- `cd services/backend && npm run build && npm run lint` -- expect clean
- `cd services/backend && npm test -- projects mentorship cds timeline && npm run test:e2e -- employee-profile` -- expect new/extended specs to pass
- `cd services/frontend && npm run typecheck && npm run lint && npm run build` -- expect clean after type/renderer changes
- `cd services/frontend && npm run test` -- expect the new/extended Playwright coverage for S10/S11 to pass

**Manual checks:**
- As a seeded employee, open your own profile; confirm timeline/leaves/projects/CDS render read-only, the leaves section shows a "manage leave" link, an active project shows PM/DM/dates, ticking an open IDP's complete checkbox records a date without exposing conclusion/deadline edit, and toggling "open to mentoring" updates immediately

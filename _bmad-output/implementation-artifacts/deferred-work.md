# Deferred Work

## Deferred from: code review of spec-12-5-people-partner-dashboard (2026-09-10)

- Extract shared HR-line walk helper with `AccessResolverService` — story design notes accept per-service duplication for now; lock-step parity is enforced by tests rather than shared code.

## Deferred from: code review of spec-2-5-view-shared-feedback-flagged-notes-and-own-action-items (2026-09-10)

- `SELF_RISK_INVISIBLE` dashboards/notifications scope — profile-only assertion matches story backend scope; dashboard risk surfaces are out of story 2.5.

## Deferred from: code review of spec-2-4-view-own-timeline-leaves-projects-cds-and-mentorship (2026-09-10)

- `confirmedAt` freshness filtering for S11 active assignments — spec open Q2; current filter covers `confirmed: false` and past `endDate` only.
- SharedLinkManagerDialog S10/S11 label Playwright coverage — no shared-link e2e harness exists yet; `sectionLabel` keys updated to `.title` paths.

## Deferred from: code review of spec-2-3-upload-photo-and-certificates (2026-09-10)

- Playwright e2e for photo/certificate upload UI — frontend has no employee-profile e2e harness yet; backend profile e2e covers API contract per spec Verification commands.
- Swagger `@ApiConsumes` / `@ApiBody` multipart docs for new identity/documents routes — follows same gap as other feature controllers; not required by story AC.

## Deferred from: code review of spec-2-2-edit-own-personal-and-emergency-contacts (2026-09-10)

- Playwright e2e for S2/S3 profile UI (add/edit/delete, read-only manager rendering) — frontend has no employee-profile e2e harness yet; backend profile e2e covers API contract.
- Dedicated service/controller unit tests beyond section providers — write paths covered by `employee-profile.e2e-spec.ts` I/O matrix; provider specs cover access denial.

## Deferred from: code review of spec-2-1-view-own-employment-summary (2026-09-10)

- Manual seed verification after `db:seed` — spec Verification requires confirming S4 empty-state copy on real Story 1.16 accounts; fixture/stub e2e cannot substitute; manual gate only.
- Wire `EmploymentSectionDto` into `EmployeeProfileEntity` OpenAPI — section entities exist but profile assembly uses generic `Record<string, unknown>` envelope like other sections (same gap as S15, 6.4 review).

## Deferred from: code review of spec-8-4-filter-by-assessment-recency-and-open-idp (2026-09-10)

- Campaign-audience filters for `last_assessment_date` / `has_open_idp` — export resolved; campaigns remain out of story 8.4 scope.
- Provider-unavailable (503) over HTTP e2e — requires module-level provider unregistration harness; display/export `fieldsUnavailable` covered by unit tests.

## Deferred from: code review of spec-8-3-manager-pp-maintain-assessments-and-conclusions (2026-09-09)

- `forbidNonWhitelisted` 400 on PATCH with extra body fields — global `ValidationPipe` in `bootstrap.ts` strips unknown properties before the route-level pipe runs; behavior matches IDP routes; not observable at e2e boundary.
- Playwright coverage for CDS assessment create/conclusion-edit UI and write gating in `CdsSection.tsx` — frontend has no employee-profile e2e harness (same gap as stories 8.1/8.2); backend profile e2e covers API contract.

## Deferred from: code review of spec-8-2-idp-records (2026-09-09)

- Playwright coverage for IDP UI (create/edit/self-complete/checkbox absence for non-Self viewers) — frontend e2e not in story 8.2 verification commands; backend profile e2e covers API contract.
- `idp-record-input.ts` service-layer normalization duplicates DTO rules — intentional defense-in-depth; validator imports shared `isValidIdpDeadline` helper.

## Deferred from: code review of spec-8-1-skills-matrix-link-and-assessment-log (2026-09-09)

- S12-specific shared-link 400 beyond-grant e2e — `shared-link.service.spec.ts` already covers grant validation generically with S6.
- Playwright coverage for `CdsSectionCard` UI branches — frontend e2e not in story 8.1 verification commands; backend profile e2e covers the API contract.
- Resourcing auto-generated manager links should include S12 by default — cross-epic gap (epic-8-context UX pattern); frozen spec change-log item 5 keeps this out of story 8.1 scope.

## Deferred from: code review of spec-6-4-request-history (2026-09-09)

- S15 OpenAPI on profile endpoint — section entities documented but profile assembly stays generic envelope like other sections.
- Frontend Playwright for S15 card — no component-test harness in frontend; manual/bootcamp verification applies.
- Bootcamp seed manual S15 fixture — no decided-proposal seed row; flag at manual QA time per spec Ask First.
- FixedClock reversal timestamp distinctness — e2e uses single instant; unit tests assert `decidedAt` write on each decide.

## Deferred from: code review of spec-6-3-dm-reviews-and-approves-rejects-candidates (2026-09-09)

- Concurrent headcount-race e2e under real DB contention — transactional unit tests cover the conditional-update path; full parallel e2e disproportionate for this story.
- Frontend Playwright coverage for reviewing-DM UI (approve/reject/reverse, reason dialog, shared-link link) — frontend has no component-test harness; manual/bootcamp verification applies.
- Partial shared-link creation on submit failure — 6.2 orchestration pattern; atomic rollback out of 6.3 scope.
- Frontend unit test for `expiresInHours: 168` submit fix — no vitest in frontend; backend consume path covered by e2e.

## Deferred from: code review of spec-12-3-delivery-manager-dashboard-with-project-selector.md (2026-09-09)

- Missing frontend e2e for unassigned/clear-selection paths in `dashboard-engine.spec.ts` — selector refetch e2e covers primary flow; unassigned UX deferred to manual or follow-up.

## Deferred from: code review of spec-12-4-project-manager-dashboard.md (2026-09-09)

- source_spec: `_bmad-output/implementation-artifacts/spec-12-4-project-manager-dashboard.md`
  summary: Dashboard counter values (risk levels, resourcing counts) are never asserted against real seeded data anywhere in the test suite — backend e2e only checks counter-catalog shape (ids/order) and frontend e2e only checks counter test-id visibility, for UM, DM, and now PM alike.
  evidence: Confirmed pre-existing and systemic, not introduced by this story — the shipped, reviewed `spec-12-3` DM config/summary e2e tests have the identical gap (no `Risk` records seeded, no counter-value assertions), and the frontend `dashboard-engine.spec.ts` never asserts rendered counter text for any variant. A dedicated follow-up should seed real risk/resourcing data and assert computed values across all dashboard variants at once, rather than patching only the PM path.

- source_spec: `_bmad-output/implementation-artifacts/spec-12-4-project-manager-dashboard.md`
  summary: A PM dashboard viewer who is also listed as `dmId` on some `ProjectAssignment` row (independent of holding the Delivery Manager functional role) would see that project's DM-level resourcing requests and comp-band data through `ResourcingService.listRequests`/`canViewExpectedCompBand`, not just their own PM-authored requests.
  evidence: Verified in `prisma/schema.prisma` (`Employee.pmProjectAssignments` / `dmProjectAssignments` are independent relations with no constraint tying `dmId`/`pmId` to functional-role holdings) and `resourcing.service.ts`'s `buildListWhere`, whose DM-visibility OR-clause activates on raw `dmId` match regardless of caller context. This is Story 6.1's existing, deliberate authorization model (employee-level `dmId`/`pmId` grants visibility everywhere, not dashboard-variant-scoped) — spec-12-4's own Boundaries explicitly forbid changing `ResourcingService.listRequests`/`buildListWhere`, calling it "already correct." Worth revisiting only if dashboard-variant scoping is later required to be stricter than raw employee-level resourcing authorization.

## Deferred from: investigation for spec-6-3-dm-reviews-and-approves-rejects-candidates (2026-09-09)

- source_spec: none
  summary: `CLOSE_RESOURCING_REQUESTS` permission is defined and seeded onto the Delivery Manager role (`permission-keys.ts`, `seed.functional-roles.ts`), but no epic-6 story (6.1–6.4 per `sprint-status.yaml`) has an acceptance criterion for an explicit "DM closes the request" action, and 6.3's own epics.md ACs only cover approve/reject — the permission has no consuming endpoint or UI anywhere in the codebase.
  evidence: Confirmed via grep — `CLOSE_RESOURCING_REQUESTS` appears only in the permission catalog and seed grant list, never in `resourcing.controller.ts`/`resourcing.service.ts` routes or any frontend action. D18 (`decisions.md`) requires "only the DM's explicit close ends the request," but no story currently owns building it; kept out of spec-6-3's scope to match its epics.md ACs and the SCOPE STANDARD single-goal target — needs a home in 6.4 or a new small story.

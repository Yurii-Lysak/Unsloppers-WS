# Deferred Work

## Deferred from: code review of spec-12-3-delivery-manager-dashboard-with-project-selector.md (2026-09-09)

- Missing frontend e2e for unassigned/clear-selection paths in `dashboard-engine.spec.ts` — selector refetch e2e covers primary flow; unassigned UX deferred to manual or follow-up.

## Deferred from: code review of spec-12-4-project-manager-dashboard.md (2026-09-09)

- source_spec: `_bmad-output/implementation-artifacts/spec-12-4-project-manager-dashboard.md`
  summary: Dashboard counter values (risk levels, resourcing counts) are never asserted against real seeded data anywhere in the test suite — backend e2e only checks counter-catalog shape (ids/order) and frontend e2e only checks counter test-id visibility, for UM, DM, and now PM alike.
  evidence: Confirmed pre-existing and systemic, not introduced by this story — the shipped, reviewed `spec-12-3` DM config/summary e2e tests have the identical gap (no `Risk` records seeded, no counter-value assertions), and the frontend `dashboard-engine.spec.ts` never asserts rendered counter text for any variant. A dedicated follow-up should seed real risk/resourcing data and assert computed values across all dashboard variants at once, rather than patching only the PM path.

- source_spec: `_bmad-output/implementation-artifacts/spec-12-4-project-manager-dashboard.md`
  summary: A PM dashboard viewer who is also listed as `dmId` on some `ProjectAssignment` row (independent of holding the Delivery Manager functional role) would see that project's DM-level resourcing requests and comp-band data through `ResourcingService.listRequests`/`canViewExpectedCompBand`, not just their own PM-authored requests.
  evidence: Verified in `prisma/schema.prisma` (`Employee.pmProjectAssignments` / `dmProjectAssignments` are independent relations with no constraint tying `dmId`/`pmId` to functional-role holdings) and `resourcing.service.ts`'s `buildListWhere`, whose DM-visibility OR-clause activates on raw `dmId` match regardless of caller context. This is Story 6.1's existing, deliberate authorization model (employee-level `dmId`/`pmId` grants visibility everywhere, not dashboard-variant-scoped) — spec-12-4's own Boundaries explicitly forbid changing `ResourcingService.listRequests`/`buildListWhere`, calling it "already correct." Worth revisiting only if dashboard-variant scoping is later required to be stricter than raw employee-level resourcing authorization.

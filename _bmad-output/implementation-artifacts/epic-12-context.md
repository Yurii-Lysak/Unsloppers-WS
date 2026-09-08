# Epic 12 Context: Dashboards

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver one shared dashboard engine (**FR-53**) that renders four role-scoped home views (Unit Manager, Delivery Manager, Project Manager, People Partner) from configuration rather than separate page implementations. Each view aggregates counters, scoped tables, the viewer's own action items, and quick navigation — all resolved through the same C1 access layer as profiles and directory, never as a bypass surface. Epic 12 is scheduled last because dashboards aggregate data from Epics 4–6, 10, and 13 (leave/resourcing feeds); Epic 9 (mentorship) is out of scope until a later counter/widget story. The engine shell (Story 12.1) can start earlier against partial provider coverage.

## Stories

- Story 12.1: Build Shared Dashboard Engine
- Story 12.2: Unit Manager Dashboard
- Story 12.3: Delivery Manager Dashboard with Project Selector
- Story 12.4: Project Manager Dashboard
- Story 12.5: People Partner Dashboard
- Story 12.6: Accessibility and Responsive Layout Pass

## Requirements & Constraints

- A single engine renders summary counters, scoped people/project tables (active risk + leave status), own action items (due-date sorted, overdue highlighted), and quick nav — reused across all four role dashboards.
- Role views differ only in grouping dimension (people vs. project vs. department) and which functional blocks appear; fixes to a counter type or widget propagate identically.
- All dashboard data must pass through C1 `AccessResolver` / the Provider Registry — no dashboard-specific access bypass.
- **Variant-scoped populations:** UM scope is reporting-hierarchy Manager reach only (`listManagerSubordinateIds` — transitive `managerId` descendants, **never** the viewer's own row, never PP- or project-line-only subjects). DM/PM scope follows project responsibility; PP scope follows PP assignment — each population must be testable independently. The orchestrator passes the resolved subject (or project-group) set into summary merges; it must **not** call existing providers with viewer-only scope when that provider's default audience is broader (e.g. `risks` `DashboardSummaryProvider` today uses `listS6SubjectIds`, which unions Reporting-line, Project-line, and PP).
- Missing integration feeds (leave, resourcing, campaigns) and unregistered registry providers must fail soft per **AD-3** — show "temporarily unavailable" on affected cells/counters, never fabricated zeros or softened access rules. AD-8's sync-lag copy applies only to timetracker-backed columns once those providers exist.
- **Multi-role viewers:** when a seed employee (or future functional-role wiring) matches more than one dashboard variant, the backend resolver picks a single variant by fixed priority **UM → DM → PM → PP** and documents the choice in the config response; no client-side variant switching in 12.1.
- The standalone Risk Dashboard (Epic 5) remains a separate nav surface; Epic 12 home dashboards consume the same risk semantics via `DashboardSummaryProvider`, but re-scoped to each variant's audience.
- Story 12.6 handles WCAG 2.1 AA and responsive layout after surfaces exist — not part of 12.1–12.5 delivery.
- Table pagination for large populations is deferred to Stories 12.2–12.5; 12.1 may cap proof rows in tests only.

## Technical Decisions

- Backend `dashboards` module orchestrates registry lookups only — depends on `contracts` + `registry` per AD-1, never imports feature modules directly.
- AD-3 `DashboardSummaryProvider` family: one registrant per aggregate source (`risks` exists; leave, resourcing, employment, action-items, campaigns still needed). The orchestrator calls `registry.get('dashboard-summary', id)` and merges typed fragments into a config-driven response **for the variant-scoped subject set** (extend provider contract or filter provider output in the orchestrator — do not reuse S6-wide risk counts on the UM dashboard).
- Audience listing lives in a dedicated **`dashboard-audience` contract** implemented in `access` (expose `listManagerSubordinateIds`, `listProjectGroups`, and later `listPpAssignedIds`) — reuse the private `listReportingLineDescendantIds` pattern in `section-access-gate.service.ts` L125–144; do not extend `SectionAccessGate`'s public surface for dashboard-only queries.
- REST routes: `GET /api/v1/dashboards/config` and `GET /api/v1/dashboards/summary` (global prefix/version from `main.ts` — never omit in specs or clients).
- Frontend organizing axis is `pages/DashboardPage/` with shared engine components; role variant selected from server config — not four independent page folders.
- Reuse existing UI primitives: `RiskBadge`, `TrendArrow`, `OverdueIndicator`, shadcn `Card`; generalize counter/table patterns from `pages/RiskDashboardPage/` without modifying the Risk Dashboard route.
- Home route (`/`) becomes the role dashboard per EXPERIENCE.md IA; `HomePage` placeholder is replaced. Viewers with no resolvable dashboard variant get **403** on dashboard APIs and the existing protected-route access-denied pattern on `/` (same as other forbidden surfaces — not a silent redirect to an empty home).

## UX & Interaction Patterns

- Canonical composition reference: `mockups/dashboard-um.html` — five counter tiles, "Your people" table (risk badge + trend arrow, comfortable row height), "Your action items" widget with overdue left-border treatment, quick nav links.
- Counter tiles are numeric + label cards; table rows click through to Employee Profile; action items reuse Epic 4 overdue derivation and authored-list S14 visibility filtering.
- DM/PM layouts group tables by project (selector filters whole page in 12.3); PP layout omits resourcing block entirely (12.5).
- Nav item "Home" is the dashboard; functional-role permissions gate visibility of other nav entries per FR-5.

## Cross-Story Dependencies

- **12.1 blocks 12.2–12.5** — engine, config layer, shared components, backend orchestrator, and audience contracts must exist first.
- **12.1 depends on:** Epic 1 C1 access resolution; AD-3 registry substrate; `risks` `DashboardSummaryProvider` (Story 5.3, done) with orchestrator-side re-scoping; action-items authored list endpoint (Epic 4, done).
- **12.2–12.5 depend on:** respective data epics (4 action items, 5 risks, 6 resourcing, 8 CDS, 10 campaigns, 13 timetracker sync) for full counter feeds — 12.1 may ship with partial provider coverage and unavailable placeholders.
- **12.3 before 12.4** — PM dashboard reuses DM engine shape.
- **12.6 depends on** Epics 1, 3, and 12 dashboards all substantially built.

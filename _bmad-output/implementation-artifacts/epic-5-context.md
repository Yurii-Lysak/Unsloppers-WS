# Epic 5 Context: Risks and Risk Dashboard

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver a retained-history, five-level risk record per employee with trend indication and a dedicated Risk Dashboard scoped to Manager and People Partner access. Risk data is management-only: employees never see their own risk level, trend, or history on any surface. This epic realizes the core flight-risk workflow — managers and PPs can record escalating concern, see whether risk is improving or worsening, and scan their population by severity.

## Stories

- Story 5.1: Record a Risk
- Story 5.2: Show Risk Trend
- Story 5.3: Risk Dashboard

## Requirements & Constraints

- Managers and PPs can record a risk for anyone in their access scope: level (low, need attention, medium, high, leaver), description, details, and date.
- Risk history is append-only; the current level is always the most recent record. Risks are never closed — they only move between levels, including back down to low.
- Profile section S6 is entirely absent for Self — no keys, no controls, no inference on any surface (profile, dashboards, APIs).
- Trend compares the current record to the previous one and appears only when the level actually changed; no arrow on the first-ever record.
- Fixed severity ordering for trend computation: low < need attention < medium < high < leaver.
- "Active risk" means any level above low; dashboard counters and active-risk figures use this definition exclusively.
- `leaver` is a prediction about someone still employed and must never be conflated with a `dismissed` employment-status fact.
- The Risk Dashboard shows severity-sorted active-risk counts (medium, high, and leaver visually emphasised), a drill-through table filterable by department, project, PP, and manager, scoped to the viewer's Manager/PP population. Ordinary employees are denied the surface entirely.
- Clicking a severity count filters the table; clicking a row opens the employee profile. Table sorts by severity then date.
- A functional role may grant "create/edit risks" as a feature permission, but it never widens underlying data access — writes still require Manager or PP relationship to the subject.
- Shared profile links may include S6 only when explicitly enabled; it is excluded by default alongside other sensitive sections.

## Technical Decisions

- Backend module: `risks`, owning CAP-5. Data model includes `RiskRecord` linked to `Employee`.
- Registers two provider types via the shared Provider Registry: `SectionProvider` for S6 (profile assembly) and `DashboardSummaryProvider` for dashboard risk aggregates.
- Profile assembly calls the S6 provider only when access resolution grants the section; unauthorized sections are never fetched.
- Trend (`up` | `down` | `flat`) is computed once on the backend in the risk DTO, diffed against the previous `RiskRecord` — Profile, All Employees, and Risk Dashboard all consume the same precomputed value.
- Frontend surface: `pages/RiskDashboard/` per the IA; page may call backend APIs directly but does not import other feature modules.
- Risk severity uses dedicated design tokens (five-step scale plus success for improving trends), separate from the generic destructive palette.
- Access-matrix negative tests must cover S6 absence for Self (cross-cutting, established in Epic 1).

## UX & Interaction Patterns

- **Risk Badge + Trend Arrow** — one shared component instance on Profile S6, the All Employees risk column, and the Risk Dashboard. Renders level label, color, and directional arrow; never re-implemented per surface.
- **Risk Dashboard** — reached from nav "Risks"; Manager line and PP only. Counter cards at top, drill-through table below using comfortable row density.
- **Filter/Audience Builder** — reused from All Employees for dashboard filters (department, project, PP, manager).
- **Empty state** — "No risk history recorded." No CTA pushing the viewer to add a risk.
- **Voice** — direct and precise about risk; no gamification, celebratory patterns, or falsely upbeat copy.
- **Accessibility** — risk severity is never conveyed by color alone; every badge carries its text label. Trend direction must be perceivable to screen-reader users.
- **i18n** — all user-facing strings are translation keys.

## Cross-Story Dependencies

- **Within epic:** Story 5.2 depends on 5.1 (records must exist to compute trend). Story 5.3 depends on 5.1 and 5.2 (dashboard displays levels, counts, badges, and trends).
- **Epic 1 (Access Control):** S6 access matrix enforcement, profile assembler via Provider Registry, functional-role permission for create/edit risks, Self-absence guarantees.
- **Epic 3 (All Employees):** Risk level and trend appear as a list column for entitled viewers.
- **Epic 12 (Dashboards):** Role dashboards consume risk summary counts via `DashboardSummaryProvider`; drill-through links into this epic's Risk Dashboard.

---
title: 'Show Risk Trend'
type: 'feature'
created: '2026-09-04'
status: 'done'
review_loop_iteration: 0
baseline_commit: '0eb60796f12c2ca735d3c67149138632da66a210'
story_key: '5-2-show-risk-trend'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-5-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-5-1-record-a-risk.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Story 5.1 delivers append-only risk history and `currentLevel`, but no `trend` field or visual indicator — managers and PPs cannot see whether risk is improving or worsening versus the prior record.

**Approach:** Compute `trend` once on the backend in `RisksSectionEntity` by comparing the two most recent records using fixed severity ordering (`low < need_attention < medium < high < leaver`). Add shared `RiskBadge` and `TrendArrow` components and wire them into profile S6's current-level header.

## Boundaries & Constraints

**Always:**
- **Severity ordering:** reuse `RISK_LEVELS` in `risk-input.ts` — single source for compare/index.
- **Trend computation:** in `risks.service.ts` `toSectionDto()` — compare `records[0].level` vs `records[1].level` using the same sort as `currentLevel` (`recordedAt desc`, `createdAt desc`, `id desc`). Result: `'up'` (worsening), `'down'` (improving), or `'flat'` (unchanged level).
- **Omit `trend` when `records.length < 2`** — no previous record to compare; do not send `null` or `flat` for first-ever record.
- **Section DTO:** add optional `trend?: 'up' | 'down' | 'flat'` on `RisksSectionEntity`; Swagger `@ApiPropertyOptional({ enum: ['up', 'down', 'flat'] })`. Parallel `GET /employees/:employeeId/risks` and profile S6 inherit automatically.
- **Frontend types:** mirror `trend?: 'up' | 'down' | 'flat'` on `RisksSection` in `employee-profile.ts`.
- **Shared components:** `RiskBadge` (level pill + text label per DESIGN.md `risk-badge` variants) and `TrendArrow` (chevron adjacent to badge). Render arrow only when `trend === 'up' | 'down'` — hide for `flat`, absent, or first record. Up arrow uses destination level's risk color; down arrow uses `success` token.
- **Design tokens:** add `risk-*` and `success` OKLCH tokens to `index.css` per DESIGN.md L15–38; map via Tailwind `@theme` — no palette hex or manual `dark:` overrides.
- **Profile S6:** replace text-only `currentLevel` paragraph in `RisksSection.tsx` with `<RiskBadge>` + conditional `<TrendArrow>`. History list items may use `RiskBadge` for level label (optional polish, not required for AC).
- **a11y:** severity never color-only; trend direction exposed via `aria-label` i18n keys (e.g. worsening/improving).
- **Self:** unchanged — S6 omitted; trend never visible to subject employee.

**Ask First:**
- If `index.css` token additions conflict with an in-flight design-system refactor, ship backend + unit/e2e tests first.

**Never:**
- Client-side trend derivation; `DashboardSummaryProvider` or Risk Dashboard page (5.3); All Employees risk column / `FieldProvider` (Epic 3); update/delete risk records; conflate `leaver` with `dismissed` employment status; duplicate badge/arrow markup outside shared components.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Worsening | Prior `low`, new `medium` | `trend: 'up'`; up chevron in S6 header | N/A |
| Improving | Prior `high`, new `low` | `trend: 'down'`; down chevron in success color | N/A |
| Unchanged level | Prior `medium`, new `medium` (different description) | `trend: 'flat'`; no arrow rendered | N/A |
| First record | `records.length === 1` | `currentLevel` set; `trend` key absent | N/A |
| No history | `records.length === 0` | No `currentLevel`, no `trend` | N/A |
| Self viewer | Employee E views own profile | No S6 section; trend never exposed | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/risks/risk-input.ts` L10–16 — `RISK_LEVELS` ordering; add `computeRiskTrend(current, previous)` helper here.
- `services/backend/src/modules/risks/risks.service.ts` L72–77 — `toSectionDto()`; derive and attach `trend`.
- `services/backend/src/modules/risks/entities/risk-record.entity.ts` L35–41 — `RisksSectionEntity`; add optional `trend` field.
- `services/backend/src/modules/risks/risks.controller.ts` L39–64 — GET returns section DTO unchanged structurally.
- `services/backend/src/modules/risks/__tests__/risks.service.spec.ts` — unit tests for up/down/flat/omit cases.
- `services/backend/test/risks.e2e-spec.ts` L96+ — two-record sequence asserting `trend` on GET + profile S6.
- `services/frontend/src/types/employee-profile.ts` L53–78 — add `trend` to `RisksSection`.
- `services/frontend/src/index.css` — add `risk-*` + `success` tokens per DESIGN.md.
- `services/frontend/src/components/RiskBadge/` (new) — shared level pill; base on shadcn `Badge`.
- `services/frontend/src/components/TrendArrow/` (new) — directional chevron; consumes `trend` + `level`.
- `services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/RisksSection.tsx` L44–49 — replace text header with badge + arrow.
- `services/frontend/src/locales/en/translation.json` L194+ — trend aria labels under `employeeProfile.s6.*`.
- `services/frontend/e2e/employee-profile-assembly.spec.ts` L102+ — extend S6 mock/assertions if present.

**Read-only (5.1 — do not change):** append-only model, create auth/routes, Self omission, colleague 403, provider failure → 503.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/risks/risk-input.ts` -- add `computeRiskTrend` using `RISK_LEVELS` -- single ordering source
- [x] `services/backend/src/modules/risks/risks.service.ts` -- attach `trend` in `toSectionDto` -- backend-computed field
- [x] `services/backend/src/modules/risks/entities/risk-record.entity.ts` -- optional `trend` on section entity -- API contract
- [x] `services/backend/src/modules/risks/__tests__/risks.service.spec.ts` -- unit tests for all trend cases -- regression guard
- [x] `services/backend/test/risks.e2e-spec.ts` -- two-record e2e asserting `trend` on GET/profile -- epic AC
- [x] `services/frontend/src/index.css` -- risk + success design tokens -- visual foundation
- [x] `services/frontend/src/components/RiskBadge/` -- shared level badge component -- reuse for 5.3/Epic 3
- [x] `services/frontend/src/components/TrendArrow/` -- shared trend chevron -- reuse for 5.3/Epic 3
- [x] `services/frontend/src/types/employee-profile.ts` -- `trend` on `RisksSection` -- type parity
- [x] `services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/RisksSection.tsx` -- wire badge + arrow in header -- UJ-1 UX
- [x] `services/frontend/src/locales/en/translation.json` -- trend aria labels -- a11y

**Acceptance Criteria:**
- Given employee E has a previous record at `low` and I hold Manager/PP access, when a new record is saved at `medium`, then `trend` is `up` on `GET /employees/E/risks` and profile S6 shows an up arrow alongside the current level badge
- Given employee E has no prior risk records, when the first record is saved, then `trend` is absent from the section DTO and no arrow renders in S6
- Given employee E is the subject, when E views their own profile, then S6 remains absent and no trend is exposed via any API they can call

## Spec Change Log

## Design Notes

Trend compares only the two most recent records by the existing sort — not a rolling window. `flat` is computed server-side but the UI hides the arrow (per DESIGN.md: arrow only on actual level change). Down-arrow color uses `success`, not the destination level's risk color.

## Verification

**Commands:**
- `cd services/backend && npm test -- risks.service` -- expected: trend unit tests pass
- `cd services/backend && npm run test:e2e -- risks` -- expected: two-record trend assertion passes
- `cd services/frontend && npm run lint && npm run typecheck` -- expected: pass

**Manual checks:**
- Manager opens employee with two+ risk records at different levels → S6 header shows badge + directional arrow; single-record employee shows badge only.

## Suggested Review Order

**Trend computation (backend)**

- Single ordering source and compare helper for severity direction
  [`risk-input.ts:18`](../../services/backend/src/modules/risks/risk-input.ts#L18)

- Attach derived trend only when two or more records exist
  [`risks.service.ts:72`](../../services/backend/src/modules/risks/risks.service.ts#L72)

- Optional trend field on section DTO for GET/profile consumers
  [`risk-record.entity.ts:39`](../../services/backend/src/modules/risks/entities/risk-record.entity.ts#L39)

**Shared UI components**

- Risk severity tokens mapped through Tailwind theme
  [`index.css:65`](../../services/frontend/src/index.css#L65)

- Reusable level pill with text label for every surface
  [`RiskBadge.tsx:12`](../../services/frontend/src/components/RiskBadge/RiskBadge.tsx#L12)

- Directional chevron; hidden for flat/absent; success color on improve
  [`TrendArrow.tsx:15`](../../services/frontend/src/components/TrendArrow/TrendArrow.tsx#L15)

**Profile S6 integration**

- Badge + guarded arrow in current-level header
  [`RisksSection.tsx:46`](../../services/frontend/src/pages/EmployeeProfilePage/components/RisksSection/RisksSection.tsx#L46)

**Tests**

- Unit coverage for up/down/flat/omit trend cases
  [`risks.service.spec.ts:110`](../../services/backend/src/modules/risks/__tests__/risks.service.spec.ts#L110)

- E2e two-record trend on API and profile S6
  [`risks.e2e-spec.ts:155`](../../services/backend/test/risks.e2e-spec.ts#L155)

- Playwright stubs for up/down/flat/absent arrow visibility
  [`employee-profile-assembly.spec.ts:111`](../../services/frontend/e2e/employee-profile-assembly.spec.ts#L111)

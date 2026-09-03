---
title: 'Overdue Highlighting'
type: 'feature'
created: '2026-09-03'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'c014c9962c916d0411f0e5e12f5df43c4e936d8f'
story_key: '4-3-overdue-highlighting'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-4-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/spec-4-2-complete-and-cancel-action-items.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md'
  - '{project-root}/_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/EXPERIENCE.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Stories 4.1–4.2 persist and mutate action items but expose no overdue signal. FR-23 requires one consistent derivation everywhere items render (`epics.md` labels this FR-20); without a shared backend field, each future surface (profile S14, self-service, dashboard widget, campaign table) would re-implement date logic and drift.

**Approach:** Add a derived `isOverdue` boolean on every action-item read DTO and on C6 `ActionItemDto`, computed at map time via a shared UTC calendar-date utility that injects `Clock`. Ship backend + tests first; wire the Overdue Indicator UI when profile infrastructure exists.

## Boundaries & Constraints

**Always:**

- **Derivation rule:** `isOverdue === true` only when `status === 'open'` **and** `dueDate` (UTC calendar) is **strictly before** today's UTC calendar date. Due **today** → `false`. Any non-`open` status (`completed`, `cancelled`, or future values) → always `false`, regardless of `dueDate`.
- **Never stored:** no Prisma column, no batch job, no client-only derivation for API consumers.
- **Single source of truth:** `isActionItemOverdue(status, dueDate, clock)` in `action-item-input.ts` (colocated with `formatActionItemDueDate` / `parseActionItemDueDate`). Call from `toReadDto`, `toAuthoredDto`, and `toContractDto` only — section provider and controller stay pass-through.
- **DTO shape:** add required `isOverdue!: boolean` to `ActionItemReadEntity`, `AuthoredActionItemReadEntity`, and C6 `ActionItemDto` (Swagger `@ApiProperty` on entities).
- **Surfaces** (existing paths only — no new routes; all must return `isOverdue` per the derivation rule):
  - `GET /employees/:employeeId/action-items`
  - Profile assembler S14 (`buildSection` → `toReadDto`)
  - `GET /me/authored-action-items` (open items only — overdue applies to subset)
  - POST create / complete / cancel response bodies (`ActionItemReadEntity`)
  - C6 `createActionItem` return via `toContractDto` (`ActionItemDto`)
- **Clock:** use injected `Clock.now()` for "today"; never `new Date()` / `Date.now()` in derivation. E2e uses `FixedClock` (`DEFAULT_TEST_INSTANT = 2026-01-05T09:00:00.000Z`).
- **Date comparison:** normalize both `dueDate` and `clock.now()` to UTC calendar ms via `Date.UTC(getUTCFullYear(), getUTCMonth(), getUTCDate())` — same pattern as `access-resolver.service.ts` project-assignment bounds. Ignore time-of-day on `clock.now()`; only the UTC calendar date matters. `dueDate` is `@db.Date`; existing parse stores UTC midnight via `T00:00:00.000Z`. `formatActionItemDueDate` is output-only for DTO serialization, not part of the comparison.
- **Campaign source:** `source: 'campaign'` items use identical derivation (no separate logic).

**Ask First:**

- If `EmployeeProfilePage` / `ActionItemsSection.tsx` exist when implementing, add `OverdueIndicator` per DESIGN.md (`{components.overdue-indicator}`: 3px `destructive` left border, `destructive` label text, transparent background, label copy e.g. `Overdue — was due Aug 18`; never full-row red fill). Overdue status must be conveyed by text/label, not color alone (Epic 15). Otherwise ship backend + e2e only (same deferral as 4.1–4.2).

**Never:**

- Dashboard overdue counters or widget UI (Epic 12); campaign per-person completion table (Epic 10); self-service page shell (Epic 2 Story 2.5); notifications; schema migration; reopening terminal items; author/assignee mutation changes.

## I/O & Edge-Case Matrix

Delta scenarios not fully spelled out in Boundaries or Acceptance Criteria above:

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Clock retreats | Open item past due; `FixedClock.set()` moves clock to earlier UTC day | `isOverdue` becomes `false` on next read without DB write | N/A |
| Due today at late UTC time | `status: open`, `dueDate` equals clock's UTC calendar day, `clock.now()` at 23:59:59Z | `isOverdue: false` | N/A |
| Create response | POST create with `dueDate` before clock today | 201 body `isOverdue: true` | N/A |
| Complete/cancel response | POST complete or cancel on open past-due item | 200 body `isOverdue: false` immediately | N/A |
| C6 contract return | `createActionItem` (via `toContractDto`) for open past-due item | `ActionItemDto.isOverdue: true` | N/A |
| E2e fixture dates | `FixedClock` at `2026-01-05`; overdue fixtures must use `dueDate` **before** `2026-01-05` (e.g. `2025-12-31`) — not `2026-08-01` (future relative to clock) | Overdue assertions use correct fixture dates | N/A |

</frozen-after-approval>

## Code Map

Implementation hints — negotiable during build; frozen intent is above.

| Path | Scope |
|------|-------|
| `services/backend/src/modules/action-items/action-item-input.ts` | Add `utcCalendarDateMs(date: Date): number` and `isActionItemOverdue(status, dueDate: Date, clock: Clock): boolean`; default `false` for any `status !== 'open'` |
| `services/backend/src/modules/action-items/entities/action-item.entity.ts` | `ActionItemReadEntity.isOverdue` (`@ApiProperty`) |
| `services/backend/src/modules/action-items/action-items.service.ts` | `toReadDto` / `toAuthoredDto` / `toContractDto`: call shared util with `this.clock`; `Clock` already injected |
| `services/backend/src/modules/contracts/action-item-creation.contract.ts` | Required `isOverdue: boolean` on `ActionItemDto` |
| `services/backend/src/modules/action-items/action-items-section.provider.ts` | Read-only; inherits `isOverdue` via `buildSection` → `toReadDto` |
| `services/backend/src/clock/clock.service.ts` | Read-only; documents overdue as a Clock use-case |
| `services/backend/test/support/fixed-clock.ts` | `DEFAULT_TEST_INSTANT`, `FixedClock.set()` / `advance()` for boundary e2e |
| `services/backend/src/modules/action-items/__tests__/action-item-input.spec.ts` (new) | Table-driven unit tests on `isActionItemOverdue` / `utcCalendarDateMs` with mocked `Clock` |
| `services/backend/src/modules/action-items/__tests__/action-items.service.spec.ts` | Mapper integration: `isOverdue` populated on all three DTO paths |
| `services/backend/src/modules/action-items/__tests__/action-items-section.provider.spec.ts` | Assert `isOverdue` on S14 provider read mapping (mirror 4.2 terminal-field pattern) |
| `services/backend/test/action-items.e2e-spec.ts` | New overdue cases: past-due (`2025-12-31`), due-today, future, terminal on mutation responses, clock advance + retreat, profile S14, authored list, C6 create return; do **not** assert `isOverdue: true` on `'stores past due dates and treats empty link as omitted'` (`dueDate: 2026-08-01` is future under default clock) |
| `services/frontend/...` | No action-item components in `services/frontend` today; `OverdueIndicator` when profile page lands (per Ask First) |

**Read-only references:**

- `services/backend/src/modules/access/access-resolver.service.ts` (~L578) — UTC calendar comparison pattern
- `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/DESIGN.md` — `{components.overdue-indicator}` tokens
- `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-08-21/mockups/dashboard-um.html` — label copy reference

## Tasks & Acceptance

**Execution** (see Code Map for file index):

- [x] Backend derivation + DTO wiring per Code Map rows through `action-item-input.spec.ts`
- [x] Section provider spec + service spec mapper coverage
- [x] E2e overdue matrix per delta scenarios (correct fixture dates under `DEFAULT_TEST_INSTANT`)
- [ ] `OverdueIndicator` + `ActionItemsSection.tsx` — only if profile page exists per Ask First

**Acceptance Criteria:**

- **AC-1:** Given an open item with `dueDate` strictly before today (UTC), when it is returned from any **Surface** above, then `isOverdue` is `true`
- **AC-2:** Given an open item due today or in the future, when it is returned from any **Surface** above, then `isOverdue` is `false`
- **AC-3:** Given an item is `completed` or `cancelled` (even with a past `dueDate`), when it next renders on any **Surface** above, then `isOverdue` is `false`
- **AC-4:** Given a campaign-sourced open item past its due date, when returned on any **Surface** above, then `isOverdue` uses the same derivation as a manual item (no alternate logic)
- **AC-5:** Given the system clock advances past an open item's `dueDate` with no mutation, when the item is read again, then `isOverdue` becomes `true` without any database write; given the clock retreats so a previously overdue item's `dueDate` is no longer before today, then `isOverdue` becomes `false` without any database write
- **AC-6:** Given an open past-due item, when POST create, complete, or cancel returns **201**/**200**, then the response body includes `isOverdue` matching the derivation rule for the item's terminal or open state
- **AC-7:** Given `createActionItem` returns via `toContractDto`, when the item is open and past due, then `ActionItemDto.isOverdue` is `true` using the same derivation as read DTOs
- **AC-8:** Given `EmployeeProfilePage` exists when implementing, when an item has `isOverdue: true`, then `OverdueIndicator` renders per DESIGN.md with a visible text label (not color/icon alone)

### Review Findings

- [x] [Review][Patch] FR traceability [`Intent`] — cite FR-23; note `epics.md` FR-20 mapping
- [x] [Review][Patch] E2e fixture dates [`I/O matrix`, `Code Map`] — overdue fixtures before `2026-01-05`; do not assert on `2026-08-01` test
- [x] [Review][Patch] C6 `ActionItemDto` required [`Boundaries`, `Code Map`] — `isOverdue` required on contract DTO; AC-7
- [x] [Review][Patch] Mutation response AC [`Boundaries`, `AC-6`] — POST create/complete/cancel bodies include `isOverdue`
- [x] [Review][Patch] Section provider spec [`Code Map`] — `action-items-section.provider.spec.ts` for `isOverdue`
- [x] [Review][Patch] Date comparison clarity [`Boundaries`, `Code Map`] — `Date.UTC` on calendar parts; `formatActionItemDueDate` output-only
- [x] [Review][Patch] Clock retreat + UTC-day boundary [`I/O matrix`, `AC-5`] — backward `FixedClock.set()` and late-UTC-day due-today cases
- [x] [Review][Patch] Pure util unit tests [`Code Map`] — `action-item-input.spec.ts` on `isActionItemOverdue`
- [x] [Review][Patch] Frontend a11y AC [`Ask First`, `AC-8`] — text label required when profile UI ships
- [x] [Review][Patch] C6 surface explicit [`Boundaries`, `AC-7`] — `toContractDto` / `createActionItem` return listed
- [x] [Review][Patch] Structure dedup [`Code Map`, `Tasks`, `I/O matrix`] — Code Map canonical; matrix delta-only; Tasks net-new work
- [ ] [Review][Defer] Frontend `OverdueIndicator` — deferred per Ask First when `EmployeeProfilePage` absent
- [ ] [Review][Defer] `epic-4-context.md` alignment — refresh parent doc `dueDate < today` wording on epic-context regen (strict UTC, due-today false)

#### Code review (2026-09-03)

- [x] [Review][Patch] Stage untracked unit test file [`action-item-input.spec.ts`] — new file exists on disk but is not tracked; `npm test` passes locally yet the file would be omitted from the branch commit
- [x] [Review][Patch] AC-3 follow-up read on terminal transitions [`action-items.e2e-spec.ts`:1098] — overdue e2e asserts `isOverdue: false` on complete/cancel **responses** only; extend to GET list + profile S14 after terminal transition (existing 4.2 complete e2e checks terminal fields on profile but omits `isOverdue`)

## Verification

**Commands:**

- `cd services/backend && npm run build` — expected: compile clean
- `cd services/backend && npm run lint` — expected: no errors
- `cd services/backend && npm test -- --testPathPatterns=action-items` — expected: unit tests pass
- `cd services/backend && npm run test:e2e -- action-items.e2e-spec` — expected: e2e pass (Postgres up)

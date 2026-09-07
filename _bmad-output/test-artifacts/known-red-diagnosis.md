# Known-red diagnosis — `Unsloppers-BE` @ `77cfaca`

**No production code was changed** by this investigation. Three test-only files
were fixed after being root-caused (see "Fixed in this pass" below) —
`.github/workflows/ci.yml` and `lint:check` in `package.json` are the only other
backend diff, both from the earlier CI-pipeline work.

Context: run locally on Windows, Node 22, Postgres 18 in Docker, against the tip
of `origin/main` (`77cfaca`, Story 10.2). `package-lock.json` is unmodified, so
CI installs the same dependency tree — anything reproducing here should reproduce
on a runner.

This supersedes the initial labels in `ci-pipeline-progress.md`. Two of the three
original "known-red" items were mischaracterised on the first pass: there was
**no transaction deadlock** and **no S16 data leak**.

## Fixed in this pass (test-only, safe, verified)

| File | Change | Verified |
|---|---|---|
| `src/prisma/__tests__/temporal-history.extension.spec.ts` | Added `AND schemaname = current_schema()` to the `pg_indexes` query (item 2) | Reproduced the original bug on demand (created a throwaway schema with the same migrations, confirmed 4 failures), applied the fix, confirmed 65/65 with the stray schema still present, then dropped it |
| `test/jest-e2e.global-teardown.ts` | Moved `dropSchemas(...)` into a `finally` so a thrown coverage assertion can no longer skip cleanup (item 2, root cause) | Full unit suite: 707/707 |
| `test/campaigns.e2e-spec.ts` | Fixture used `employmentStatus: 'inactive'`, not a valid `EmploymentStatus` value — changed to `'dismissed'` (item 3b) | `campaigns.e2e-spec.ts`: 19/19 |

Full e2e re-run after these three fixes: **10 failures across 6 suites** (down
from 11/7). No new failures introduced. Everything below reflects this fixed
state; items 2 and 3b are kept in this document for the paper trail, not because
they're still open.

---

## 1. Build — `TS2322` in `campaigns.service.ts` (real, blocks deploy)

```
src/modules/campaigns/campaigns.service.ts:146:9
Type 'FieldFilter[]' is not assignable to type 'JsonNullClass | InputJsonValue | undefined'
```

`saveAudience` writes `normalized.filters` (a `FieldFilter[]`) into the
`audienceFilters` Json column via `formCampaign.updateMany`. Prisma wants
`Prisma.InputJsonValue`.

There is an established pattern for this in the codebase — `toJsonValue` in
`src/modules/timeline/timeline.service.ts:245` and
`timeline-event-writer.service.ts:76`, both returning
`Prisma.InputJsonValue | typeof Prisma.DbNull`. The read side already goes
through `parseStoredAudienceFilters` (lines 396 and 440), so only the write side
is untyped.

**Impact beyond CI:** Render's build command is
`npm install --include=dev && npm run build`. `nest build` fails here, so the
backend deploy for `77cfaca` should have failed and production is serving an
older build. Worth confirming in the Render dashboard.

**Caution:** fixing this unblocks auto-deploy on the next push to `main`, and
`postbuild` runs `prisma migrate deploy` against the shared Neon database. That
is the risky part, not the type fix. Per `docs/deployment.md` there is no staging
tier and no separate dev database.

---

## 2. Unit tests — false alarm, local environment artifact — FIXED

Originally recorded as "3 failures … transaction deadlock". It is neither a
deadlock nor a product bug.

`src/prisma/__tests__/temporal-history.extension.spec.ts:840` asserts a live-DB
index definition:

```sql
SELECT indexdef FROM pg_indexes WHERE indexname = ${indexName}
```

The query does not filter by `schemaname`. My local database still contained a
leftover `tea_test_w1` schema from an earlier e2e run, which carries the same
migrations and therefore the same index names, so each of the four
`describe.each` dimensions got 2 rows instead of 1.

Why the schema was left behind — `test/jest-e2e.global-teardown.ts` runs
`assertMatrixCoverage(...)` **before** `dropSchemas(...)`. Any run that does not
cover the full matrix (a filtered run, or a run with failures) throws in the
coverage assert and never reaches the drop. The schema then survives and poisons
the next unit run.

Verified: after dropping the leftover schema, `temporal-history.extension.spec.ts`
passes **65/65**.

Two independent test-hygiene defects, both cheap and safe to fix, neither
affecting product code — **both fixed** (see "Fixed in this pass"):

- the `pg_indexes` query is now schema-scoped;
- teardown now drops schemas in a `finally`, so the coverage assertion cannot
  skip cleanup.

Re-verified by deliberately recreating the original failure mode: manually
`migrate deploy`-ed a throwaway schema with the full migration set into the same
database, confirmed the pre-fix query still returns 4 failures against it, then
confirmed the fixed query returns 65/65 with that schema still present. Dropped
the throwaway schema afterward.

**CI was unaffected either way** — `unit-tests` and `e2e` are separate jobs with
separate Postgres service containers, so no leftover schema can exist there. The
fix mainly protects local dev loops (and anyone else hitting Ctrl-C mid e2e-run).

---

## 3. E2E — 10 failures across 6 suites (was 11/7 before the campaigns fixture fix)

Reproduced deterministically: the same failures show up whether run as the full
24-suite sweep or as just the affected suites in isolation. `maxWorkers` is
already 1, so parallelism is not a factor.

### 3a. S16 "custom-field leak" — false alarm, fragile assertion

`test/employee-profile-custom-fields.e2e-spec.ts:194`

The access control is **correct**. The structured assertions immediately above
(lines 184–189) pass: the Colleague viewer receives exactly the
colleague-visibility field (`Nickname` / `Sam`) and nothing else. The management
field (`at-risk`) and the employee-only field name (`Shirt size`) are both
correctly absent.

What fails is a raw substring guard on the whole serialized response:

```ts
expect(colleagueRaw).not.toContain('L');
```

`'L'` is the *value* of the employee-only "Shirt size" field — a single
character. The colleague response legitimately contains the key
`manageLeaveUrl` in the S10 section, which contains a capital `L`. So the
assertion trips on an unrelated key.

No leak, no security finding. The test needs a value that is not a single common
character, or a structured assertion instead of a substring scan.

### 3b. `campaigns.e2e-spec.ts` — test bug, wrong enum value — FIXED

`rejects inactive added employee ids` (line 741) did:

```ts
data: { managerId: ..., employmentStatus: 'inactive' }
```

`PrismaClientValidationError: Invalid value for argument 'employmentStatus'.
Expected EmploymentStatus.` The enum in `prisma/schema.prisma:35` is
`active | dismissed` — there is no `inactive`. The fixture, not the product.

Changed to `'dismissed'` — confirmed this preserves the test's intent:
`campaigns.service.ts:313` filters `where: { employmentStatus: 'active' }` when
validating added audience members, so any non-`'active'` status (including
`'dismissed'`) still gets rejected as intended. `campaigns.e2e-spec.ts` now
passes 19/19.

### 3c. Three 403s — endpoints now require an Employee record

- `custom-fields.e2e-spec.ts:23` — `GET /api/v1/custom-fields`, expected 200, got 403
- `custom-fields.e2e-spec.ts:41` — `GET /api/v1/custom-fields/values/:id`, expected 404, got 403
- `auth.e2e-spec.ts:89` — `GET /api/v1/users`, expected 200, got 403

All three authenticate through `loginAsOperator` (`test/support/login.ts`), which
creates a bare `User` with **no `Employee` row**.
`CustomFieldsController.resolveViewerEmployeeId` (line 118) throws
`ForbiddenException('Authenticated user has no employee record')`. `git log`
attributes this to `5501c98` — "enforce colleague whitelist on all API routes
(Story 1.8)", which added viewer-employee resolution to both the custom-fields
and users controllers.

This looks like **intended hardening with stale fixtures**: the leak harness
asserts exactly this 403 for a user with no employee record
(`access-matrix-leaks.e2e-spec.ts:160`). But the failing test's own name says
"returns an array for authenticated users", so the intended contract for *listing
definitions* needs an owner decision — harden the fixture, or exempt the
definition-list route.

### 3d. `employees.e2e-spec.ts` — two 400s, genuine and reproducible

`filters employees by derived years_with_company > 3` and `returns an empty row
set when filters match no employees`. Both send a JSON-encoded `filters` query
param; both get 400. A request with no `filters` returns 200, so it is the
filters path specifically.

The actual response body (captured with a throwaway diagnostic spec, since
supertest only surfaces the status):

```json
{"message":"Unknown field \"undefined\"","error":"Bad Request","statusCode":400}
```

Thrown at `field-registry.service.ts:615`. So validation passes but `filter.fieldId`
arrives as `undefined` — the nested filter objects lose their properties during
DTO transformation.

The suspect is `dto/list-employees-query.dto.ts:83-105`, where `filters` carries
`@Transform` (JSON.parse), `@Type(() => EmployeeFieldFilterDto)` and
`@ValidateNested({ each: true })` together, under a global
`ValidationPipe({ whitelist: true, transform: true })` (`src/bootstrap.ts:21`).
When `@Transform` supplies the value, the array items are not instantiated as
`EmployeeFieldFilterDto`, and `whitelist` then strips properties it has no
metadata for.

What is odd, and why this needs the owner: none of the three pieces changed
recently. The DTO is untouched since `0e8c8fe` (#17), the pipe since `9c376b3`
(#2), and `validateFilters` / the `Unknown field` message already existed before
`bd4c4d1` (#39). `BUILTIN_FIELD_IDS` are intact. So either this has been broken
since the list feature landed and nothing ran the suite, or something
environmental differs. `package-lock.json` is clean, so the first green/red
signal from the new `e2e` job is the cheapest way to settle it.

### 3e. `colleague-whitelist.e2e-spec.ts` — NOT a stale test; looks like a real access gap

`returns S1-safe directory list entries` (line 241) treats
`GET /api/v1/employees` as a bare array of `{ id, displayName }`:

```ts
const rows = res.body as Array<Record<string, unknown>>;
expect(rows.length).toBeGreaterThan(0);   // received: undefined
for (const row of rows) {
  expect(Object.keys(row).sort()).toEqual(['displayName', 'id']);
}
```

I initially read this as a stale shape assertion (the endpoint returns the
paginated `{ total, rows, fields }` object `employees.e2e-spec.ts` uses). On
closer read of `employees.service.ts`, that's not the whole story:

- `filterVisibleFields` (line 169) only ever filters **custom** fields by
  `visibility`; every builtin field (`name`, `grade`, `position`, `department`,
  `employment_type`, `years_with_company`) is pushed through unconditionally
  for every viewer.
- `maskRowCells` (line 202) only masks **custom** field cells per row; it never
  touches builtin cells.
- Nothing in `listEmployees` calls `sectionGate` or otherwise checks the
  viewer's relationship (Colleague vs. Manager vs. PP vs. Self) to each row's
  subject.

So today, `GET /api/v1/employees` returns full builtin-field data — grade,
position, department, tenure — for every employee to any authenticated caller
with an `Employee` record, Colleague included. There is no per-row S1-safe
narrowing on this route at all, which is exactly what the failing test expects
to exist.

I did not touch this one. Whether this is a real gap (the list endpoint needs
audience-aware row narrowing, mirroring what the profile endpoint already does
for S1) or intentional (this route is meant to be reachable only by viewers who
already have broad directory access, gated by something upstream I haven't
found) is exactly the kind of access-control call this document's own caution
applies to — I would rather flag it than guess and quietly narrow an API
response. **Recommend routing this to whoever owns the access matrix (AD-14),
not silently patching the test to match current behavior.**

### 3f. `employee-profile.e2e-spec.ts` — three 401s, not root-caused

Failing at lines 367, 405 and 440 — the last three tests in the file. In each
case the failure is on the **first** request of the test, before any mutation, so
the earlier hypothesis (that reassigning a manager invalidates sessions) is
wrong. The shared `colleagueAgent` cookie is simply no longer accepted by the
time these tests run.

Something earlier in the file invalidates that session, or the `FixedClock` used
by this suite interacts badly with the JWT `exp`/`iat` window. This is the one
item I could not pin down; it needs someone who knows the intended session
semantics.

---

## Summary

| # | Item | Verdict |
|---|---|---|
| 1 | `TS2322` build failure | Real. Blocks Render deploy. Known fix pattern exists. Not fixed — needs the deploy-risk conversation first. |
| 2 | 4 unit failures | **Fixed.** Was a false alarm — leftover test schema + unfiltered `pg_indexes` query. |
| 3a | S16 "leak" | False alarm — `not.toContain('L')` matches `manageLeaveUrl`. Access control correct. Not fixed (test-only, low priority). |
| 3b | campaigns fixture | **Fixed.** `'inactive'` was not in the `EmploymentStatus` enum; corrected to `'dismissed'`. |
| 3c | 3 × 403 | Likely intended Story 1.8 hardening, stale fixtures. Needs owner call. |
| 3d | 2 × 400 on filters | Real API defect — `fieldId` stripped in DTO transform. Cause predates recent commits. |
| 3e | colleague-whitelist | **Escalated, not fixed.** Looks like a real access-control gap, not a stale test — `GET /api/v1/employees` has no per-row audience narrowing at all; every builtin field is visible to every authenticated viewer regardless of role. |
| 3f | 3 × 401 | Not root-caused. Session invalid before the test's first request. |

Two items fixed this pass (2, 3b) — both verified safe, test-only, no product
code touched. **3d and 3e are the two that matter most now**: 3d is a confirmed
API defect on a real endpoint, and 3e may be a bigger finding than originally
scoped — a live authorization gap, not test debt. Both need the person who owns
the access matrix / directory list feature, not a guess from here. 1 remains the
only item with deployment consequences.

---
name: 'Adversarial Review — Architecture Spine (People Management Platform)'
type: review
lens: adversarial
target: '_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
created: '2026-08-21'
---

# Adversarial Review — Architecture Spine

Method: for each AD, construct two developer tracks that each read only the spine (and its named companions) and could build something that is letter-compliant yet fails to interoperate. Ten findings below; each names the two divergent-but-compliant builds and a concrete AD fix.

---

## Finding 1 — C3's `confirmed` field is invented by AD-8 but absent from the ratified DTO shape

**AD(s):** AD-2, AD-8

**The gap:** AD-2's table ratifies C3 as owned by `access` (seed) / `integrations` (writer), pointing at `interface-contracts.md`. That document's C3 shape is `{ employeeId, projectId, pmId, dmId, startDate, endDate }` — no `confirmed` field. AD-8, written later in the same spine, casually adds "`ProjectAssignment` rows carry a `confirmed: boolean` (or equivalent sync-state) column" without updating AD-2's table or C3's canonical shape, and hedges with "or equivalent sync-state."

**Two compliant builds:**
- Developer A (owns `access`'s seed path, reads AD-2 + interface-contracts.md as the source of truth for C3) ships `ProjectAssignment` with exactly the five original fields — no `confirmed` column exists yet in their migration.
- Developer B (owns `integrations`' sync writer, reads AD-8 closely) adds `confirmed: boolean`, but because "or equivalent sync-state" is explicitly offered as an alternative, chooses instead a `syncStatus: 'confirmed' | 'pending' | 'stale'` enum column — semantically the same intent, structurally incompatible with anything hard-coding a boolean.

Either pairing breaks `AccessResolver`'s Manager-access leg (AD-8's core promise): if Developer A's schema ships first and Developer B's sync writer expects a `confirmed` boolean that doesn't exist, the fail-closed guarantee silently doesn't compile or silently no-ops. If both build independently against their own reading, the migration that lands second conflicts with the first.

**Fix:** Update AD-2's contract table (or C3 itself in `interface-contracts.md`) to include the exact field name and type — `confirmed: boolean` (not "or equivalent") — as part of C3's ratified shape, not a satellite detail buried in AD-8's prose.

---

## Finding 2 — No sanctioned path from `auth` to every other module for `viewerId`

**AD(s):** AD-1, AD-9

**The gap:** AD-9 splits `auth` (produces `userId`) from `access` (`resolveAudience(viewerId, subjectId)`, consumes it). But C1 is only called *from* controllers across every feature module (risks, directory, cds, feedback, …) — each of those controllers must itself obtain the authenticated `viewerId` from the incoming request before it can call anything that leads to `AccessResolver`. AD-1's dependency diagram lists `auth` as a peer feature module (`auth --> contracts`) exactly like `risks` or `cds` — meaning **no module may import `auth` directly** per AD-1's own rule. Yet nothing in `contracts` (per AD-2's C1-C6 table) exposes a "get current user from this request" token. There is no C7 for session extraction.

**Two compliant builds:**
- Developer A (`risks`) pragmatically imports `JwtAuthGuard`/`@CurrentUser()` straight from `../auth/guards/...` — it works, ships fast, and is the obvious thing to do — but violates AD-1's letter (a feature module importing another feature module directly, not through `contracts`).
- Developer B (`directory`), reading AD-1 literally and refusing to import `auth`, hand-rolls its own JWT-verification middleware local to `directory` to extract `userId` from the bearer token — fully AD-1-compliant, but now there are two independent JWT-parsing implementations in the codebase that can drift (different clock-skew tolerance, different claim names, different expiry handling). This is precisely what AD-9 says it prevents ("a future SSO swap touch[ing] the access model") — except now an SSO swap has to touch N independent auth implementations instead of one.

**Fix:** Add a contract token (e.g. C7 `AuthenticatedRequest` / `CurrentUserProvider`) to `contracts`, implemented and bound by `auth`, consumed via DI/decorator by every controller — the same registry-of-interfaces pattern AD-3 already uses for profile assembly, applied to the much more basic "who is calling" need every module actually has.

---

## Finding 3 — No contract exists for functional-role/feature-permission checks

**AD(s):** AD-1, AD-2

**The gap:** `access-model.md` establishes a second, independent axis beyond C1's section-access roles: functional roles (UM/DM/PM/PP/HR Admin) gate *features* ("create resourcing requests," "manage custom fields," "assign mentors," etc.), are "data, not code," and a permission revocation "takes effect immediately for everyone holding it." C1's `resolveAudience` answers section R/RW/none for a *subject profile* — it says nothing about feature-permission checks like "can this user create a resourcing request." AD-2's table has no contract for this at all.

**Two compliant builds:**
- Developer A (`resourcing`, needs to gate "create request" behind DM/PM permission) queries `FunctionalRoleAssignment`/`FunctionalRolePermission` directly via Prisma inside `resourcing`'s own service — no feature-module import happens, so AD-1's literal rule ("never import another feature module directly") isn't violated, even though this reaches straight into `access`'s owned tables and duplicates permission-resolution logic.
- Developer B (`cds`, same need for "maintain CDS records") instead blocks on `access` shipping a proper `PermissionChecker` contract token and consumes it via DI once available.

Now half the codebase checks permissions through live, uncached Prisma reads with module-local logic, and half goes through whatever `access` eventually exposes — and if any raw-Prisma module later adds its own local cache for performance (nothing in AD-4 stops it, since AD-4's caching rule is scoped explicitly to `AccessResolver`), "revocation takes effect immediately" silently stops being true for that one module only.

**Fix:** Add a C7 `PermissionChecker` (or fold into C1 as a second method) to AD-2's table, owned by `access`, and state explicitly that no module may query `FunctionalRoleAssignment`/`FunctionalRolePermission` tables directly — mirroring AD-6's "only FieldRegistry branches on type" closure clause.

---

## Finding 4 — `dashboards`' cross-module aggregate read has no legal path under AD-1/AD-3

**AD(s):** AD-1, AD-3

**The gap:** AD-3 solves cross-module reads only for **single-subject profile assembly** — `SectionProvider.getSection(viewerId, employeeId)` is shaped around one profile at a time, invoked only for sections the resolved audience grants. `dashboards` (CAP-12) needs the opposite access pattern: aggregates across many employees/projects (risk counts by unit, overdue action items, resourcing pipeline volume) filtered by the *viewer's* Manager/PP/functional scope — not "assemble sections for one profile." AD-2's table lists no dashboard-facing contract (C2 `FieldRegistry` is scoped to audience *filters*, not bulk data pulls from risks/action-items/resourcing/etc.). The capability map governs CAP-12 by AD-1 alone — no AD-3-style registry extension for it exists. AD-1's rule is also enforceable only at the TypeScript-import level ("never import another feature module directly"); it says nothing about the shared Prisma schema, so a module can reach another module's tables via Prisma without ever importing that module's code.

**Two compliant builds:**
- Developer A (`dashboards`), needing risk/action-item/resourcing counts, has no sanctioned contract to call, so injects the shared Prisma client directly in `dashboards`'s own service and queries `RiskRecord`, `ActionItem`, `ResourcingRequest` tables by hand — technically doesn't "import another feature module," so it passes AD-1's literal text, but it now hard-codes structural knowledge of every other module's schema, exactly the coupling AD-1 exists to prevent.
- Developer B (`dashboards`, different sprint, different developer), reading AD-3's registry pattern as "the" way cross-module reads are done here, invents an unratified `DashboardDataProvider` contract, adds it to `contracts` unilaterally, and expects `risks`/`action-items`/`resourcing` to implement it — but those modules, built independently against AD-2's actual (unchanged) table, never implemented it, so `dashboards` ships with silent empty/broken widgets for anything it didn't get to stub itself.

Both are "compliant" with the words of AD-1; neither produces a system where dashboards' data path is predictable, and the two developers' two dashboards modules are not interchangeable.

**Fix:** Either (a) extend AD-3's registry pattern with a second, bulk-shaped contract per data-owning module (e.g. `DashboardSummaryProvider.getSummary(viewerScope): SummaryDTO`) explicitly named in AD-2's table, or (b) explicitly bless a read-only cross-module query path (e.g. a dedicated read-model/view layer) — but the spine must pick one and say direct Prisma cross-module reads are forbidden regardless of whether a TS import occurred, closing the loophole AD-1's current wording leaves open.

---

## Finding 5 — SectionProvider registry collision and missing-registration behavior is unspecified

**AD(s):** AD-3

**The gap:** AD-3 says each section-owning module "registers it against its section id via a multi-provider DI token," and that a provider for a `—` section is never invoked. It does not say what happens when (a) two providers register against the same section id, or (b) a section the resolved audience *does* grant has no registered provider at all (e.g. that track hasn't shipped yet, or a config/registration typo omits it).

**Two compliant builds:**
- Developer A implements `ProfileAssemblerService`'s registry resolution as `providers.find(p => p.sectionId === id)` — on a collision, the first-registered module silently wins with no error, and on a missing registration, `getSection` is simply never called and the section is silently absent from the response (indistinguishable from "not granted").
- Developer B implements it as an assertion at module-bootstrap time — collisions throw a startup error (fail loud, fail fast), and a missing registration for a granted section throws a 500 at request time rather than silently omitting the section.

Both satisfy AD-3's literal text. But they produce opposite operational behavior: under Developer A's registry, a half-finished Wave-0 build (missing provider) looks identical to "this user isn't authorized" to the frontend — which is exactly the kind of silent-failure that `access-model.md` Rule 1 calls "a critical defect regardless of which section it happens in," since a section a user *should* see is now indistinguishable from one they shouldn't. Under Developer B's registry, an incomplete Wave-0 deployment 500s the entire profile endpoint instead of degrading section-by-section.

**Fix:** AD-3 needs an explicit rule: registration collision is a build/bootstrap-time error (never a silent last-wins/first-wins), and a granted section with no registered provider must render as an explicit "unavailable" state distinguishable from "not granted" — never a silent omission. Extend AD-2's Wave-0 stub-provider allowance (already granted for C1-C6 tokens) explicitly to `SectionProvider` too, so "not yet built" has one sanctioned representation instead of two ad-hoc ones.

---

## Finding 6 — `SectionProvider.getSection` returning `null` is ambiguous between "no data yet" and other states

**AD(s):** AD-3

**The gap:** The contract signature is `getSection(viewerId, employeeId): SectionDTO | null`. AD-3 never defines what `null` means versus an empty-but-present `SectionDTO`. For a section the audience *is* granted, but the subject simply has no records yet (e.g. an employee with zero logged risks, or zero CDS assessments), is the correct return `null` or an empty-shaped DTO?

**Two compliant builds:**
- `risks`' `SectionProvider` returns `null` when an employee has no risk records ("no data, so nothing to send").
- `cds`'s `SectionProvider` returns an empty-but-present DTO, e.g. `{ assessments: [], idp: null }`, for the equivalent "no data yet" case.

`ProfileAssemblerService` now receives two different representations of the same underlying situation ("authorized, empty") from two sections in the same response. If the assembler (or the frontend consuming it) treats `null` as "omit the key entirely" and a present-but-empty object as "render an empty-state card," the same conceptual state renders inconsistently across the profile page — one section vanishes as if ungranted, the other shows a proper empty state, and there's no way for a third developer building a new section (say, mentorship, S13) to know which convention to follow since the spine endorses both.

**Fix:** AD-3 should state explicitly that `null` is reserved for "not authorized / not applicable" only (a state the assembler should never actually observe, since it only calls granted sections) and that "authorized but no records" must always be a present, empty-shaped `SectionDTO` — never `null`. This also removes the temptation to overload `null` as a poor-man's "not authorized" signal that could mask the exact leak `access-model.md` Rule 1 warns against.

---

## Finding 7 — AD-4's generation-counter cache and AD-8's `confirmed` flip can race, because it's undefined whether toggling `confirmed` bumps the counter

**AD(s):** AD-4, AD-8 (compounds Finding 1)

**The gap:** AD-4/D1's cache-invalidation counter is described as bumping on "reports-to, project assignment, PP assignment" graph writes. AD-8 makes Manager-access grant conditional on `ProjectAssignment.confirmed = true`, written by `integrations`' sync. Toggling `confirmed` on an *existing* row is an `UPDATE`, not a create/delete of an assignment — and because C3's ratified shape (per Finding 1) doesn't even list `confirmed`, whoever designs the generation-counter write path may reasonably scope "project assignment" writes to inserts/deletes of assignment rows, not flips of a sync-state column that (as far as they know from AD-2's table) doesn't exist.

**Two compliant builds:**
- Developer A implements the cache/counter (if AD-4's deferred implementation is ever triggered by NFR.2 load) by bumping the counter only on `ProjectAssignment` row create/delete — the natural reading of "project assignment" as a graph edge coming/going.
- Developer B (`integrations`) implements the sync writer per AD-8, flipping `confirmed` on existing rows during normal sync cycles and, critically, during an outage-recovery event (mass `confirmed: false → true` or vice versa) — without calling whatever bumps the counter, since AD-8 never says confirmation-flag writes must invalidate the cache.

Result: during exactly the outage window AD-8/D3 is designed to fail closed for, a request served from a still-valid cache entry (generation counter unbumped because only the `confirmed` flip happened, not a row create/delete) continues granting Manager access from a now-stale `confirmed=true` snapshot — or continues denying it after the row is reconfirmed. This is a correct-per-AD-4, race-with-AD-8 cache, exactly the scenario the task brief predicted.

**Fix:** AD-4 (or wherever the deferred cache is eventually specified) must explicitly state that any write to `ProjectAssignment.confirmed` — not just row creation/deletion — bumps the generation counter. This should be pinned now, in the same place Finding 1's fix pins the field's existence, so the two facts don't drift apart again.

---

## Finding 8 — `CustomFieldValue` NULL-vs-sentinel and row-existence semantics are unspecified

**AD(s):** AD-6

**The gap:** AD-6 says "one column populated per row, typed by `definition.type`" but never states (a) that unused columns must be SQL `NULL` rather than a typed sentinel default, or (b) whether a `CustomFieldValue` row exists for every `(employee, fieldDefinition)` pair eagerly (created when the field is defined) or only lazily (created on first write).

**Two compliant builds:**
- Developer A creates rows lazily — a `CustomFieldValue` row for a given employee/field pair exists only once someone actually sets a value; absence of a row means "unset." Unused sibling columns on that row are left `NULL` by Prisma's defaults.
- Developer B, building a different custom field feature (e.g. a "% profile completeness" widget or an export "list all defined custom fields with their current values including blanks"), assumes rows are created eagerly at field-definition time for every employee, and to satisfy a NOT-NULL-friendly indexing strategy they added for performance, populates unused columns with typed sentinels (`''`, `0`, `false`) instead of `NULL`.

Both are compliant with AD-6's literal text (each row is still "typed by `definition.type`," and downstream consumers still branch on `definition.type` rather than introspecting columns, so simple filtering doesn't immediately break). But: a partial index built later "per commonly-filtered field" (AD-6 explicitly allows this as a query-time optimization, e.g. `WHERE valueNumber IS NOT NULL`) silently excludes or wrongly includes rows depending on which developer's convention the table actually followed, and any "how many custom field values exist for employee X" / "field completeness" query gives different answers depending on eager-vs-lazy row creation — with no way for a third developer to know which one is authoritative, since neither behavior is written down.

**Fix:** AD-6 should state explicitly: (1) unused value columns are always `NULL`, never a sentinel, and (2) `CustomFieldValue` rows are created lazily, only on first write — a missing row is the sole representation of "unset," never a present row with all-NULL value columns.

---

## Finding 9 — C1's `Map<SectionId, "R"|"RW"|"none">` return type is not representable over the wire AD-10 mandates

**AD(s):** AD-2, AD-10

**The gap:** C1's signature (`interface-contracts.md`, ratified by AD-2) types `sections` as a TypeScript `Map`. AD-10 requires every backend controller to carry `@nestjs/swagger` decorators and the frontend to consume generated types from `/api/docs-json` — but OpenAPI has no representation of a JS/TS `Map`, and NestJS's default serializer (`class-transformer`) turns a `Map` into `{}` over JSON unless a custom transform is written. AD-2 never states whether C1's return shape is an internal-only TypeScript type (never crosses an HTTP boundary directly) or is meant to be returned as-is from some "my access to this profile" endpoint.

**Two compliant builds:**
- Developer A treats C1's `Map` as purely internal to `ProfileAssemblerService`'s composition logic (per AD-3) and never exposes it directly over HTTP — no problem surfaces.
- Developer B, building a "what can I see on this profile" debug/self-check endpoint that directly returns `resolveAudience`'s result (a reasonable, spec-adjacent thing to want, and nothing in AD-2/AD-10 forbids it), returns the `Map` as-is from a controller. It swagger-decorates the DTO as best it can, but the generated OpenAPI schema and the actual runtime JSON (`{}`) diverge, and the frontend's generated type for that endpoint doesn't match what the server actually sends.

Two endpoints that both "implement C1" now behave differently — one is fine because it's never serialized, the other is silently broken because AD-2 never said C1's shape must be converted to a `Record`/plain object before crossing any HTTP boundary, and AD-10 never said contracts-module types need a wire-safe variant.

**Fix:** AD-2 (or C1 itself) should state that `Map` is the *in-process* type; any controller returning access-resolution results over HTTP must convert to `Record<SectionId, "R"|"RW"|"none">` (or an array of `{sectionId, access}` pairs) — and AD-10 should note this as the general rule for any `contracts` type that isn't already JSON-serializable, not just C1.

---

## Finding 10 — C4's `oldValue`/`newValue` typing is unspecified, so timeline rows from different callers won't share a shape

**AD(s):** AD-2, AD-7

**The gap:** `recordTimelineEvent(employeeId, type, effectiveDate, oldValue, newValue, source, authorId?)` types `oldValue`/`newValue` with no shape at all. AD-7 says the timeline "answers 'what happened, readably'" — which nudges toward human-readable strings — but C4 is called from at least three different modules per AD-2's consumer list (`access` for profile edits, `mentorship` for pair start/end, `integrations` for timetracker-derived leave/FTE changes), each tracking a structurally different kind of change (an enum grade, a date range, a boolean FTE↔subcontractor flip).

**Two compliant builds:**
- Developer A (`access`, logging a grade change) passes raw typed values — `oldValue: 'L3'`, `newValue: 'L4'` (the enum values themselves) — expecting any timeline-rendering UI to format them per `type`.
- Developer B (`integrations`, logging a timetracker-derived FTE↔subcontractor change) pre-formats a human-readable string at the call site — `oldValue: 'Full-time employee'`, `newValue: 'Subcontractor'` — reading AD-7's "readably" as an instruction to format before calling C4, not after.

The `TimelineEvent` table (and any single generic timeline-rendering component built against it, per AD-11's page-per-surface convention) now contains rows where `oldValue`/`newValue` are sometimes raw codes and sometimes pre-formatted prose, with no `type`-driven branch that reliably tells a renderer which convention a given row used, since both callers are equally "compliant" with C4's untyped signature.

**Fix:** C4 should pin `oldValue`/`newValue` to one convention — raw typed values only (letting the timeline UI format per `type`, consistent with AD-6's "only the registry branches on type" precedent) — and state it explicitly in AD-2's table or in C4's own signature, not leave it to each caller's reading of AD-7's prose.

---

## Summary

10 findings, spanning: two missing contracts (auth-context passthrough, functional-permission checks), one field silently added out-of-band (C3's `confirmed`), one cross-AD race condition (cache/counter vs. confirmation flag), two unspecified SectionProvider edge behaviors (collision/missing-registration, null-vs-empty), one unaddressed aggregate-read capability (dashboards), one schema-level ambiguity (CustomFieldValue NULL/sentinel + row lifecycle), and two under-specified DTO field types (C1's `Map`, C4's `oldValue`/`newValue`). Every finding names a concrete pair of independently-plausible, letter-compliant implementations and a specific tightening to close the gap.

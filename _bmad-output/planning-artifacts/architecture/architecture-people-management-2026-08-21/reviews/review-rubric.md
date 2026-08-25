---
reviewed: ARCHITECTURE-SPINE.md (architecture-people-management-2026-08-21)
reviewer: bmad-architecture Reviewer Gate (rubric walk)
date: 2026-08-21
---

# Review — ARCHITECTURE-SPINE.md (People Management Platform)

## Overall verdict

This spine does real technical work — the stack table's version pins all checked out live against npm, the module/folder conventions faithfully ratify the brownfield `services/backend`/`services/frontend` scaffolds with no contradictions found, and the Section Provider Registry (AD-3) is a genuinely load-bearing idea, not decoration. But it has three gaps serious enough to let the three parallel developers diverge or silently violate the SPEC: the confirmed single-container deployment decision never actually made it into the spine as a real AD (the Deferred section references "AD's single-container deployment target" — no such AD exists), the exact cross-module-read problem AD-3 solves for profile assembly recurs unsolved for Directory filters (FR-34) and Dashboards (FR-47–51), and AD-3's own "multi-provider DI token" doesn't describe a mechanism NestJS actually supports. Fix those three and this spine clears the bar; as written it is **not yet ready to gate epics/stories**.

---

## 1. Fixes the real divergence points for the level below — misses none

**Verdict: THIN**

The spine correctly identifies and fixes the *headline* divergence risk (profile assembly across 13 section-owning modules touching one another without violating AD-1) via AD-3. But the identical shape of problem recurs at least twice more and is left unfixed:

- **[CRITICAL]** Directory's `FieldRegistry` (C2, owned by `directory`) must serve FR-34's "last assessment date" / "has open IDP" filters, whose source data (`CDSAssessment`, `IDP`) is owned by `cds`. Because FR-7's 2s/500-record NFR requires this to be a single server-side SQL query (not a frontend join across two API calls), `directory` needs a read path into CDS-owned data — but AD-1 forbids importing `cds` directly, and no SectionProvider-equivalent "FieldProvider" contract exists for FieldRegistry the way it does for profile sections. Two developers (directory owner, CDS owner) have no shared pattern to build against.
  **Fix:** Extend the SectionProvider idea (or add a new contract, e.g. C7 `FieldProvider`) so any module can register derived/filterable fields against `FieldRegistry` the same way section-owners register against `ProfileAssemblerService`.
- **[CRITICAL]** Dashboards (CAP-12) needs risk level (`risks`), leave status (`integrations`), and resourcing counts (`resourcing`) composed into one table (FR-47–51). The source-tree comment asserts "`dashboards`: read-only composer, depends on contracts only (per AD-1)" as if this were already solved — it isn't. No contract in the C1–C6 table, and no analog to AD-3, covers a dashboard's cross-module reads.
  **Fix:** Same as above — either generalize AD-3's registry pattern to a generic "read-contract" mechanism reusable by both Directory and Dashboards, or add explicit per-source contracts (risk-level lookup, leave-status lookup, resourcing-count lookup).
- **[MEDIUM]** FR-46 (Feedback → "request feedback" flow creates a Form Campaign owned by `campaigns`) has no stated resolution: is this a backend `feedback → campaigns` contract call (needing a C7-style contract, since none exists), or is it purely frontend-orchestrated (the Feedback page calling `campaigns`' own create-campaign endpoint directly, which AD-11 explicitly permits for any page)? The spine doesn't say, so one developer could build a backend link the other doesn't expect.
  **Fix:** One sentence in AD-2 or a new bullet resolving this explicitly (recommended: frontend-orchestrated, consistent with AD-11, with `campaigns` never calling back into `feedback`).
- **[MEDIUM]** CAP-6's proposal flow ("using profile-sharing for internal candidates the DM lacks access to," FR-26) may need `resourcing` to trigger Shared Link creation (owned by `access`) programmatically, or this may be a manual, separate UM action via the profile UI. Unaddressed either way — low-confidence finding since the SPEC itself is ambiguous on this point, but the spine had the opportunity to resolve it and didn't.

---

## 2. Every AD's Rule is enforceable and actually prevents its stated divergence

**Verdict: ADEQUATE**

Several ADs are enforced by construction (AD-4's "no cache by default" is trivially self-enforcing; AD-5 is structurally reinforced by AD-3's "provider never even called" mechanism; AD-6/AD-7 are schema-level and easy for one developer to keep straight). But three ADs rely on undocumented discipline rather than a named mechanism:

- **[MEDIUM]** AD-1 and AD-2's "a feature module may never import another feature module directly" is not mechanically enforced anywhere named in the spine. Nothing in the NestJS/TypeScript toolchain blocks `import { RisksService } from '../risks/risks.service'` in another module — it compiles fine. In a 3-developer parallel team this is exactly the kind of rule that erodes silently.
  **Fix:** Name a concrete enforcement mechanism — `eslint-plugin-boundaries` or `dependency-cruiser` with a CI-run rule forbidding `modules/<a>/**` from importing `modules/<b>/**` except `contracts`.
- **[MEDIUM]** AD-7's claim that "a history-table write is exactly what triggers a TimelineEventWriter (C4) call" describes an intent, not a mechanism. Nothing stops a developer writing to `GradeHistory` via Prisma directly without also calling C4 — the two writes aren't atomic or structurally coupled.
  **Fix:** Either wrap both writes in a single internal helper/service method that every mutation path must call, or use a Prisma Client Extension (`$extends`) that fires the C4 call on writes to the four history tables.
- **[LOW]** AD-10's "generated, not hand-written" frontend contract has no named CI check ensuring the committed generated-types file hasn't drifted from the live `/api/docs-json`. Low risk (a stale file just breaks at compile time eventually) but easy to close.
  **Fix:** A CI step that regenerates and diffs against the committed file.

---

## 3. Nothing under Deferred could let two independently-built units diverge

**Verdict: ADEQUATE**

Most Deferred items are safe: caching (AD-4 already constrains the shape), Notifications/Analytics (no current dependents), permission vocabulary (data, not schema). Two concerns:

- **[MEDIUM]** "PeopleForce API specifics" is deferred with the fallback (outbound link) cited as the reason nothing blocks — true for the *unintegrated* path, but doesn't cover the DTO shape `integrations` hands to `resourcing` once the API *is* live (FR-26's "pulled PeopleForce data" case). If `resourcing`'s candidate-review UI is built before that DTO is frozen, the two developers can build against different assumed shapes.
- **[CRITICAL — cross-referenced from §7]** "Horizontal scaling / multi-instance deployment" is deferred by reference to "AD's single-container deployment target (user-confirmed)" — but no AD in this document states that. See §7 below; this Deferred bullet is a dangling reference to a decision that exists only in the memlog.

---

## 4. Named tech is verified-current

**Verdict: STRONG**

Every "(existing)" row cross-checked exactly against `services/backend/package.json` and `services/frontend/package.json`: NestJS 11 (`@nestjs/core ^11.0.1`), Prisma 7 + `@prisma/adapter-pg` (`^7.9.1`/`^7.9.1`), Node ≥22.12 (`engines`), React 19 (`^19.2.7`), Vite 8 (`^8.1.0`), Tailwind v4 (`^4.3.1`), React Router v7 (`^7.18.0`), TanStack Query v5 (`^5.101.2`) — all match. PostgreSQL 18 and Node 22 confirmed against backend `CLAUDE.md`.

The seven "new" pins were spot-checked live (web search, 2026-08-21): `openapi-typescript` 7.13.0, `@nestjs/jwt` 11.0.2, `@nestjs/passport` 11.0.5, `passport-jwt` 4.0.1, `react-hook-form` 7.85.0, `zod` 4.4.3, `@hookform/resolvers` 5.9.1 — all confirmed as the current published latest. No stale or fabricated version found anywhere in the Stack table.

---

## 5. Ratifies rather than contradicts the brownfield codebase

**Verdict: STRONG**

Checked against both `CLAUDE.md`s, both `package.json`s, and `schema.prisma`:
- `modules/<kebab-name>` convention, `modules/users` as reference — matches backend `CLAUDE.md` exactly.
- Prisma models PascalCase singular — matches existing `model User`.
- IDs as UUID — matches existing `User.id String @id @default(uuid())`.
- `pages/<PascalCaseSurface>/`, `components/` one-folder-per-component, `LayoutContext` as sole cross-cutting UI state, TanStack Query exclusively for server state — all match frontend `CLAUDE.md` verbatim.
- AD-10's Swagger claim ("already the established pattern") matches — `@nestjs/swagger ^11.4.6` is installed and Swagger is live at `/api/docs` per `CLAUDE.md`.
- Auth/JWT correctly identified as net-new (no `@nestjs/jwt`/passport packages currently installed; frontend `CLAUDE.md` confirms "no auth yet").

No contradictions found. One minor omission, not a conflict: the Consistency Conventions table doesn't restate the existing `@@map("users")`-style snake_case Postgres table-naming convention documented in the backend's own Prisma rules file — worth a one-line addition for completeness, **[LOW]**.

---

## 6. Covers the driving SPEC's 13 capabilities

**Verdict: ADEQUATE**

At the coarse level the Capability → Architecture Map is complete: all 13 CAP rows are present, and every AD cited in the table (AD-1 through AD-9) actually exists in the document — no dangling AD references inside this particular table. Two precision issues:

- **[MEDIUM]** CAP-3's "Lives in: `access`" entry is misleading. Per the memlog itself, CAP-3 has no dedicated module — but its *write* actions (IDP self-complete, action-item self-complete, mentorship self-flag, contact edits) are actually owned by `cds`, `action-items`, `mentorship`, and `directory` respectively, not `access`. Only the aggregated self-profile *read* genuinely lives in `access`. A developer scanning this table for "where does self-service live" would look in the wrong module for most of its functionality.
  **Fix:** Reword to "`access` (Self-scoped profile read); writes distributed to owning modules per action."
- **[HIGH — detailed in §1]** The table's completeness is coarse-grained (capability → module), not FR-grained. At the FR level, FR-34 and FR-47–51 have real architectural needs (cross-module reads into Directory/Dashboards) that no cited AD actually resolves — see §1.

---

## 7. Every dimension the initiative altitude owns is decided, deferred, or an open question — especially the operational/environmental envelope

**Verdict: BROKEN**

This is the weakest section of the spine.

- **[CRITICAL]** The memlog records a real, user-confirmed decision: *"deployment: single containerized deploy — docker-compose extended to backend+frontend+Postgres, one host/simple container platform."* This decision **never appears in `ARCHITECTURE-SPINE.md` itself** — not as an AD, not in the Stack table, not in the Structural Seed diagrams (the system diagram shows logical Client/Server/External boxes, not a deployment topology). The only trace in the spine is a dangling reference in the Deferred section: *"Horizontal scaling / multi-instance deployment — AD's single-container deployment target (user-confirmed) doesn't need it"* — this cites "AD's ... target" as though an AD states it, and none of AD-1 through AD-11 does. Per this task's own framing (rationale lives in memlog, the *decision* belongs in the spine), this is a decided dimension that silently failed to make it into the binding document.
  **Fix:** Add an AD (or at minimum a Stack/Structural Seed entry) stating the deployment target explicitly, and repoint the Deferred bullet at it.
- **[HIGH]** **Test architecture** is completely silent — not decided, not deferred, not flagged as an open question anywhere in the spine. This is notable because the PRD's own §1 Process and Data Guardrails names "test architecture" as one of four mandatory foundation-phase outputs, and SM-1 (the SPEC's *primary* success metric) is entirely about exhaustive access-matrix test coverage. An initiative-altitude spine governing a system whose #1 success signal is test coverage should say *something* architectural about how that coverage is structured (e.g., a parameterized test-matrix convention driven off `access-model.md`'s table, a fixture/seed-data convention for the 500+-record NFR, unit vs. e2e split for AccessResolver specifically) — even a one-line Deferred/decision entry would clear this bar. It has none.
- **[HIGH]** **Non-production pseudonymized-data guardrail** (PRD §1: *"real personal data must never appear in agent contexts, logs, screenshots, or the repository"*; repeated in SPEC Constraints) has zero architectural treatment — no AD, no Deferred bullet, nothing describing how seed/dev/test data gets pseudonymized or how the boundary between prod and non-prod data is enforced. Given the spine explicitly surfaces AD-7 as satisfying a different PRD §1 guardrail (temporal data), the omission of this one looks like an oversight rather than a deliberate scope call.
- Other operational concerns (secrets management for `TIMETRACKER_API_KEY`/`PEOPLEFORCE_API_KEY` in the single-container deploy, backup/DR for Postgres, observability beyond structured logging) are not addressed either, but these are reasonably deferrable *once the deployment AD itself exists* — they're downstream of the missing AD above, not independent gaps worth separate line items.

---

## Specific check — does AD-3's Section Provider Registry actually resolve the AD-1 tension?

**Verdict: PARTIAL / THIN**

For its stated scope (profile assembly across S1–S16), the *goal* is achievable, but the mechanism as written has a real technical gap: NestJS does **not** have Angular-style multi-providers (`{ provide: TOKEN, useClass: X, multi: true }` merging into an injectable array). "Registers it against its section id via a multi-provider DI token" describes a capability the framework doesn't natively have. Two providers registered under the same token in two different feature modules do not automatically merge into an array visible to a third module — Nest's DI is module-scoped by default.

The pattern *is* achievable in Nest, but only via a specific, nameable mechanism the spine doesn't cite:
- `DiscoveryService` (`@nestjs/core`) reflecting over the whole compiled application graph at bootstrap to find provider classes tagged with a custom decorator (e.g. `@RegisterSectionProvider('S6')`) — the standard Nest pattern for exactly this kind of plugin registry (used internally by `@nestjs/schedule`'s `@Cron()` discovery and `@nestjs/event-emitter`'s `@OnEvent()` discovery); **or**
- Explicit manual aggregation in `app.module.ts` (the composition root, which already imports every feature module and is therefore exempt from AD-1's feature-to-feature restriction).

Without the spine naming one of these, two developers implementing "their" SectionProvider registration are likely to guess differently — one assuming Nest auto-merges duplicate-token providers (it doesn't; this would silently produce "last one wins" or a runtime provider-collision error depending on wiring), another correctly reaching for DiscoveryService. This is a load-bearing mechanism (all 13 section-owning modules depend on it) and deserves one precise paragraph naming the actual implementation approach.

Beyond this mechanism gap, the pattern's scope is also too narrow — see §1: the identical AD-1-vs-cross-module-read tension recurs, unsolved, in Directory's FieldRegistry and in Dashboards.

---

## Specific check — any capability or FR with no home in the Capability → Architecture Map?

**Verdict: capability-level — no gaps; FR-level — yes**

All 13 CAP-n rows have a "Lives in" and "Governed by" entry, and every AD cited exists. But at the FR granularity the map hides real gaps: FR-34 (CDS-derived Directory filters), FR-46 (feedback-initiated campaign creation), and FR-47–51 (Dashboards' cross-module composed table) each depend on architectural machinery no AD actually provides — see §1 for the concrete fixes. The map's capability-level completeness is real but insufficient on its own to certify "every FR has architectural coverage."

---

## Specific check — does AD-8's confidence-state mechanism correctly implement D3 (fail-safe access / fail-soft display, as independent axes)?

**Verdict: PARTIAL**

The **independence of the two axes is correctly implemented**: `confirmed` gates `AccessResolver`'s Manager-access grant; a separate frontend-visible sync-freshness flag gates S10/S11 display; the rule explicitly states implementing one must never touch the other's code path. This matches D3's "two different axes — a security control fails closed; a display feature fails soft" framing well.

- **[HIGH]** The **fail-safe axis itself has a gap**. `confirmed: boolean` as described is a one-time, sticky flag set by `integrations` on write — nothing un-sets it during an outage, since an outage is precisely the condition under which `integrations` isn't writing anything. This correctly prevents a *brand-new, never-confirmed* assignment from granting access (the scenario AD-8's own example covers: "an outage that stops the sync stops confirming new rows"). But D3's actual wording is broader: *"If the sync **cannot currently confirm** a project assignment, that assignment does not grant Manager access (treat 'unknown' as 'not confirmed,' never 'still active')."* This reads as an ongoing-confirmation requirement, not a one-time flag — and it matters because `access-model.md` separately requires "when a project assignment ends, the derived access ends with it immediately." If a person rolls off a project during a prolonged outage, the existing row stays `confirmed = true` (last known good state) and continues granting Manager access indefinitely, which is stale-grant behavior, not fail-closed behavior, for that specific case.
  **Fix:** Add a `confirmedAt` timestamp alongside `confirmed`, and have `AccessResolver` also require `confirmedAt` within a bounded freshness window (paired with a documented staleness threshold) — or explicitly state in the AD that this staleness case is accepted as out of scope and why (e.g., "a person actively appearing in a stale-but-once-confirmed row is an acceptable risk given the outage is expected to be short-lived, per NFR.X"). As written, the AD is silent on this distinction, which is the actual finding.

---

## Summary of findings by severity

- **Critical (3):** deployment/operational-envelope decision missing from spine (dangling AD reference); AD-3's DI mechanism doesn't match NestJS's real capabilities; Directory/Dashboards cross-module reads have no AD-3-equivalent contract.
- **High (3):** test architecture entirely unaddressed; non-prod pseudonymized-data guardrail entirely unaddressed; AD-8's `confirmed` flag has no staleness/expiry handling for pre-existing rows during a prolonged outage.
- **Medium (6):** AD-1/AD-2 dependency rule has no named enforcement tool; AD-2's C3 consumer list omits `resourcing` (contradicts `interface-contracts.md`); AD-7's history-write→timeline-event coupling has no structural enforcement; FR-46 feedback→campaign path unresolved; CAP-6→Shared-Link trigger unresolved; CAP-3's "Lives in: access" map entry is misleading.
- **Low (2):** AD-10 has no named CI drift-check; Consistency Conventions omits the existing `@@map` snake_case table-naming convention.

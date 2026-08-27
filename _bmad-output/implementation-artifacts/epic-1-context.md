# Epic 1 Context: Access Control, Roles & Employee Profile

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Every profile-scoped read or write in the platform passes through this epic's access resolution — nothing else can ship correctly before it exists. Epic 1 makes "who sees what" a first-class, testable guarantee rather than an assumption: it resolves a viewer's access role (Self/Manager/PP/Colleague/Shared-Link/HR Admin) across the reporting, project, and PP graphs; assembles employee profiles section-by-section (S1–S16) so ungranted sections are absent, not hidden; enforces the Colleague whitelist and management-notes visibility flags server-side; lets HR Admins define functional roles/permissions at runtime; and issues scoped, expiring, revocable shareable profile links. Its later stories also stand up the Wave-0 substrate every other epic depends on: the `contracts` and `registry` modules, the CI module-boundary rule, JWT auth, temporal employment-history tables, single-container deployment, and the parameterized access-matrix test scaffold — so Wave-1 feature tracks can build against frozen interfaces from day one with zero cross-developer blocking.

## Stories

- Story 1.1: Derive Manager Access from Reporting Hierarchy
- Story 1.2: Extend Manager Access via Project Assignment
- Story 1.3: Assign People Partner Relationships
- Story 1.4: Define Functional Roles and Permissions via UI
- Story 1.5: Assign Functional Roles to People
- Story 1.6: Assemble Employee Profile by Section Access
- Story 1.7: Profile Header Shows Manager, PP and Mentor
- Story 1.8: Enforce the Colleague Whitelist Everywhere
- Story 1.9: Management Notes with Visibility Flags
- Story 1.10: Per-Field Custom Field Visibility
- Story 1.11: Generate a Shareable Profile Link
- Story 1.12: Shared Link Expiry, Logging and Revocation
- Story 1.13: Cache Access Resolution Safely and Revoke Immediately on Project-Assignment End
- Story 1.14: Prevent Section Leaks Across All Surfaces for Every Denied Audience
- Story 1.15: Access-Control Test Suite Architecture
- Story 1.16: Pseudonymized Seed Data Tool
- Story 1.17: Intelligent Repository, Process Setup, and Single-Container Deployment
- Story 1.18: Authentication
- Story 1.19: Backend Substrate — Contracts and Provider Registry Modules
- Story 1.20: Temporal Employment History Tables and Timeline Coupling

## Requirements & Constraints

- Access role must be re-resolved on every request per subject, never cached across a session by default (FR-1, NFR-8); any future cache must invalidate synchronously on a relationship-graph generation bump, never a bare TTL.
- Profile responses must be assembled strictly by resolved section access (S1–S16): an ungranted section is entirely absent from the API response — no key, no empty placeholder (FR-2).
- The Colleague whitelist (S1, S10 incl. leave type, S11 project name only) must be enforced identically across profile view, direct API calls, list view, and export — never a UI-only restriction (FR-3).
- Management notes (S7) carry independent "visible-for-employee"/"visible-for-PM" flags, both defaulting off (FR-4).
- Functional roles and permissions must be definable/assignable at runtime with no code deploy; a functional role only grants features, never widens the data access resolved separately (FR-5).
- Shareable profile links are section-scoped, default 24h expiry (configurable), revocable, and every access attempt (success or failure) is logged; S3/S7/S13 can never be included under any configuration (FR-6).
- Zero cross-developer blocking during delivery: cross-feature dependencies resolve via the C1–C8 frozen contracts, stub providers, or same-developer sequencing — never a wait (NFR-6).
- A missing provider registration for a granted section/field/summary must surface as an explicit "unavailable" state — never a silent, leak-shaped omission.
- Non-production environments use pseudonymized data only — never real personal data in agent contexts, logs, screenshots, or the repo (NFR-5).
- Access-matrix test coverage must include every audience × relationship-path × section combination and every negative case, enforced in CI (NFR-1).
- Deployment target is a single containerized stack (backend, frontend, Postgres) via docker-compose, one environment, no staging tier (AD-12).

## Technical Decisions

- **AD-1 (module dependency direction):** A feature module may depend only on `contracts` and `registry` — never import another feature module directly, and never query another module's Prisma tables. Enforced in CI via a `dependency-cruiser` rule forbidding `modules/<a>/**` importing `modules/<b>/**` (except `contracts`/`registry`), run on every PR.
- **AD-2 (contracts module):** `contracts` exports abstract classes/injection tokens and DTO shapes only — zero business logic, zero Prisma imports. Owning modules bind concrete implementations in their own providers array; a Wave-0 stub can substitute for any token with zero change to consumers. Any DTO crossing an HTTP boundary must be plain-JSON-serializable (e.g. C1's `sections` map is a `Record`, never a `Map`, once it crosses that boundary).
- **The eight contracts (C1–C8):** C1 `AccessResolver` (`access`) — `resolveAudience(viewerId, subjectId) → { role, sections }`. C2 `FieldRegistry` (`directory`) — uniform query interface over built-in/derived/custom fields. C3 `ProjectAssignment` (seeded by `access`, real writer `integrations`) — includes `confirmed`/`confirmedAt`. C4 `TimelineEventWriter` (`timeline`) — `recordTimelineEvent(...)` and `markSystemWriteSkipped(manualEventId, skippedAt)`. C5 `ExternalIdentityMapping` (`integrations`) — maps external system ids to employees. C6 `ActionItemCreation` (`action-items`) — `createActionItem(...)`, called by `campaigns`. C7 `CurrentUserProvider` (`auth`) — `getCurrentUser(request) → { userId }`, the only sanctioned way any module learns the current user (AD-1 forbids importing `auth` directly). C8 `PermissionChecker` (`access`) — `hasPermission(userId, permissionKey) → boolean`, the single enforcement point for functional-role feature gates.
- **AD-3 (Provider Registry):** `registry` uses `@nestjs/core`'s `DiscoveryService` + `Reflector` — any injectable decorated `@RegisterProvider(family, id)` (families: `section`, `field`, `dashboard-summary`) is discovered and indexed at bootstrap. `registry` depends only on `@nestjs/core`, never a feature module. A registration collision is a bootstrap-time failure. A missing registration for a granted section/field/summary is a runtime error at first call, rendered as "temporarily unavailable" — never indistinguishable from "not granted." `null` from a provider means "not authorized" (a state `ProfileAssemblerService` should never hit, since it only calls providers for already-granted sections); "authorized, no records yet" is always a present, empty-shaped DTO.
- **AD-4 (caching):** No caching in v1 by default; the only acceptable future shape is a per-subject cache invalidated by a monotonic generation counter on the relationship graph, bumped on any write to `ProjectAssignment` including a bare `confirmed`/`confirmedAt` flip.
- **AD-9 (auth is distinct from access):** `auth` only answers "who is this session" (JWT) and exposes `userId` exclusively via C7 — never via direct import of `auth`'s guards/services; `access` has zero knowledge of how the session was established.
- **NFR-6 (zero cross-developer blocking):** cross-feature dependencies resolve via C1–C8, stub providers, sanctioned fallbacks, or same-developer sequencing — why Story 1.19 (contracts + registry) must land before any feature module depends on them.
- **Existing conventions to extend, not replace:** one NestJS module per `modules/<kebab-name>`; thin/routing-only controllers; services own Prisma access and map known Prisma errors (P2002→Conflict, P2025→NotFound); Jest unit tests live in each module's own `__tests__/`, matching `users`/`health`.

## Cross-Story Dependencies

- Story 1.19 (contracts + registry) and Story 1.18 (auth, C7) must land before Story 1.6 (`ProfileAssemblerService`) and any `SectionProvider` implementation across every epic — they are the substrate Wave-1 feature tracks build against.
- Story 1.20's temporal history tables feed Epic 7 (Career Timeline) via C4, stubbed until Epic 7 lands the real implementation.
- Story 1.1 and 1.2 are unioned to produce full Manager access; 1.2's `ProjectAssignment` (C3) is seeded internally here and becomes real once Epic 13 (integrations) supplies a live writer.
- Story 1.11's Shared Link Manager is reused by Epic 6 (Resourcing) for candidate review when a DM lacks standing access.
- Story 1.15's parameterized test strategy extends beyond Epic 1 into Risks (Epic 5), Resourcing (Epic 6), and Action Items (Epic 4).
- Story 1.16's seed-data tool is a prerequisite for Story 3.7's (Epic 3) load-test at 500+ records.
- Story 1.10 (custom-field visibility) is jointly owned with Epic 3's `FieldRegistry` (C2), whose real implementation Epic 3 Story 3.2 provides.

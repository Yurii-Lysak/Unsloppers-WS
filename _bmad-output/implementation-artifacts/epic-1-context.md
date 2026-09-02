# Epic 1 Context: Access Control, Roles & Employee Profile

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Every profile-scoped read or write in the system passes through this epic's access resolution — nothing else can ship correctly before it exists. This epic makes "who sees what" a first-class, testable guarantee rather than an assumption: resolving a viewer's access role (Self / Manager-line / PP / Colleague / Shared Link / HR Admin) against a subject, assembling a profile section-by-section (S1–S16) by that resolved access, enforcing the Colleague whitelist server-side everywhere, supporting dual-visibility management notes, letting HR Admin define functional roles/permissions at runtime with zero deploy, and issuing shareable/expiring/revocable profile links. Its opening stories also stand up the Wave-0 substrate (contracts/registry modules, CI module-boundary rule, JWT auth, temporal history tables, deployment, access-matrix test scaffold) every other epic depends on.

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

- Access role is re-resolved on every request, never cached across a session; any future cache invalidates only on a relationship-graph generation bump, never a bare TTL.
- Profile assembly returns data only for sections the resolved role grants — an ungranted section is absent entirely, never present-but-empty or hidden client-side.
- The Colleague whitelist (S1 identity, S10 leave dates/type, S11 project name) is enforced identically on every surface — profile, direct API, export, search — as a resolved grant, never a post-hoc filter.
- Management notes carry independent "visible-for-employee" / "visible-for-PM" flags, both defaulting off; UM/DM/PP retain full read/write regardless of flags.
- Functional roles grant features only, never data access — a functional role can never widen what access resolution already determined.
- Shared links are section-scoped, must structurally exclude S3/S7/S13 under any configuration, default to 24h expiry (configurable), log every access attempt, are revocable, and give indistinguishable responses for expired/revoked/invalid.
- Every audience x relationship-path x section combination in the access matrix, including every negative case, needs a passing automated test — the platform's primary quality bar.
- Non-production environments use only pseudonymized data — never real PII in agent contexts, logs, screenshots, or the repo.
- Cross-feature dependencies resolve via frozen interface contracts, stubs, or sanctioned fallbacks — no developer should block waiting on another track.

## Technical Decisions

- `AccessResolver` (C1) is the single per-request entry point: `resolveAudience(viewerId, subjectId) → { role, sections: Record<SectionId, "R"|"RW"|"none"> }`, converted to a plain `Record` before crossing any HTTP boundary.
- `ProfileAssemblerService` resolves C1 once per request, then calls a `SectionProvider` (via the Provider Registry, `@RegisterProvider(family, id)`, discovered through `DiscoveryService`) only for granted sections; a missing registration surfaces as an explicit "unavailable" state, never a silent omission.
- Module boundary rule: a feature module depends only on `contracts` and `registry`, never imports another feature module directly (including its Prisma tables) — enforced via a `dependency-cruiser` CI rule from commit one.
- `ProjectAssignment` (C3) is seeded/internally written in this epic; Epic 13's timetracker sync becomes the real writer later, same contract shape.
- Custom-field visibility (management/employee/colleague) is a generic, field-agnostic gate at query time, shared with Epic 3's `FieldRegistry` (C2).
- Four temporal history tables (Grade/Position/Department/EmploymentType, current row = `effectiveTo IS NULL`) are coupled to `TimelineEventWriter` (C4) via a Prisma Client Extension — no service writes history any other way; a system write that would overwrite a manual timeline correction is suppressed and flagged via `markSystemWriteSkipped`, never silently dropped.
- Auth (JWT, local email/password) is fully decoupled from access resolution: it only answers "who is this session," exposing `userId` via `CurrentUserProvider` (C7) — no controller imports `auth` directly.
- No access-resolution caching by default; if built later, it invalidates synchronously on any reports-to/project-assignment/PP-assignment write, never on a timer.
- Deployment is managed PaaS, one environment, no staging tier: frontend on Vercel, backend on Render, Postgres on Neon, both auto-deploying on push to `main`; cross-origin split requires `SameSite=None; Secure` cookies and CORS echoing the exact frontend origin with credentials.
- Every `SectionProvider`/`FieldProvider`/`DashboardSummaryProvider` ships its own parameterized access-matrix test next to its code; `AccessResolver` carries the master suite driven off the access-model matrix.

## UX & Interaction Patterns

- Access Scope Chip: muted pill at the top of any other person's profile ("Viewing as Manager line" / "Colleague view" / "Shared link — expires in 4h") — always on someone else's profile, never on My Profile.
- Section Gate: dashed "note exists, not shared with you" placeholder — reserved strictly for the documented PM/S7 exception; a Colleague- or Self-denied section is simply absent, never gated.
- Profile Section Card: RW cards show a pencil inline-edit affordance; R-only cards show no affordance at all.
- Shared Link Manager: section picker with S3/S7/S13 structurally excluded, expiry setting, and active-link revocation.
- Visibility Flag Toggle: dual-flag variant for management notes (S7) — separate employee/PM toggles, both default off.
- Accessibility: resolved access scope must be announced to screen readers, not conveyed by color alone; full keyboard operability on inline edit and required-reason dialogs.
- Every new string is a translation key (`react-i18next`) — no hardcoded copy.

## Cross-Story Dependencies

- Stories 1.19 (contracts/registry) and 1.18 (auth) are Wave-0 substrate every other Epic 1 story — and every other epic — depends on; they land first.
- Story 1.2's `ProjectAssignment` is seeded/internal until Epic 13 supplies a live timetracker-backed writer against the same C3 contract.
- Story 1.20's `TimelineEventWriter` (C4) is stubbed here; Epic 7 (Career Timeline) supplies the real implementation.
- Story 1.9's management-notes visibility state (S7) is consumed by Epic 2's Self-Service story.
- Story 1.10's custom-field visibility gate is jointly owned with Epic 3's `FieldRegistry` (C2).
- Story 1.13's no-caching-by-default rule is exercised under real load only in Epic 3's Story 3.7.
- Story 1.15's test-strategy pattern extends into Epic 5 (Risks) and Epic 6 (Resourcing) access-control coverage.
- Story 1.16's 24-account seed manifest is the dataset every other epic's stories are built and tested against; 500+-record scale is a separate, later concern (Story 3.7).

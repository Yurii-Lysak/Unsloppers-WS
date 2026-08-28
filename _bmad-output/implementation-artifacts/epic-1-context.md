# Epic 1 Context: Access Control, Roles & Employee Profile

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Every profile-scoped read or write in the platform passes through this epic's access resolution — nothing else can ship correctly before it exists. Epic 1 makes "who sees what" a first-class, testable guarantee: it resolves a viewer's access role (Self / Reporting line / Project line / People Partner / Colleague / Shared Link / Full Profile Access) from three underlying relations — reports-to, department management, and project assignment — assembles employee profiles section-by-section (S1–S16) so ungranted sections are absent, not hidden, and enforces the Colleague whitelist and management-notes visibility flags server-side. It also stands up functional roles/permissions defined at runtime, expiring/revocable shareable profile links, and the Wave-0 substrate (contracts, registry, auth, temporal history tables, CI module-boundary rule, single-container deploy) every other epic depends on. **Note:** the epic's story list (1.1–1.20) predates a requirements revision (v1.2→v1.5) that added a Department entity, split "Manager" into two differently-scoped tiers, and introduced a separate, journaled Full Profile Access grant and an organisational-relationship write path — none of which the existing stories or their acceptance criteria mention yet. Requirements/Technical Decisions below reflect the current (v1.5) model; the story list itself likely needs new/amended stories before build.

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

- Manager access now derives from **three** relations — reports-to, department management, project assignment — unioned into **two** differently-scoped tiers: **Reporting line** (reports-to ∪ department-management, both transitive) gets the full manager section-grant; **Project line** (project-assignment alone) is strictly narrower — no S2, no S3, S5 limited to CV+certificates, and S7 splits (DM gets RW, PM gets R gated by the visible-for-PM flag).
- **Department** is a first-class, self-nesting entity with a manager; "Unit Manager" is just the name for a department's manager — there is no separate unit entity. A department manager sees everyone in the department and its sub-departments.
- **HR Admin is configuration-only** (custom fields, dictionaries, departments, functional roles/permissions) and grants zero data access on its own. Full-profile-access is a wholly separate, non-self-assignable grant: seeded at deployment, its holder count re-checked at commit time so it can never reach zero, independent of every functional role including HR Admin.
- A write requires **both** the access matrix's section grant (R/RW) **and** the relevant functional-role permission — neither is sufficient alone.
- When a viewer resolves to more than one audience for the same subject (e.g. Project line + PP), effective per-section access is the **union** (least-restrictive), never a ranked precedence.
- Manager, PP, department, and department-manager are **access switches**: never writable via S1's general edit or the All-Employees inline grid; changed only through a dedicated, permissioned screen that rejects self-assignment, reporting/department cycles, self-managed departments, orphaned departments, and stale concurrent writes — every change journaled.
- A narrow **relationship/grant journal** (distinct from any general audit log) records manager/PP/department/department-manager changes, full-access grants/revocations, and shared-link accesses; readable by full-access holders and the subject's current manager/PP.
- Shared links are **authenticated and explicitly named at creation** — no anonymous "anyone with the link" mode. All `cfg` sections default off except S1; never-share set is {S3, S7, S13, S14}. Exposure **re-clamps continuously** to the creator's live access on every view, not just at creation. Revocation follows whoever currently holds Reporting-line/Project-line/PP access (full-access holder as backstop), never the original creator.
- Colleague whitelist is exactly S1, S10 (dates only — leave type hidden), S11 (project name only), enforced identically on every surface (API, export, search, notifications). One exception: a campaign author sees name + that campaign's completion status for their own recipients only, ending when the campaign closes.
- Revocation timing is split: platform-owned relationship changes take effect on the next request; project-derived access within 15 minutes — stated as an access guarantee, not best-effort.
- Non-production environments use only the **given seeded population** — **24 bootcamp test accounts** from `docs/bootcamp-seed-accounts-source.csv` / `bootcamp-identities.json` (see `bootcamp-scope-overrides.md`). Never real employee data. No employee-provisioning flow and no SSO/AD; auth is local JWT over that seeded population. TimeTracker leave/project sync is Epic 13.
- Access-matrix test coverage (CI-enforced) must cover every audience × section combination, every negative case, every narrowed project-line cell, and the multi-audience union case.

## Technical Decisions

- Contracts now number **C1–C13** (grown from the original C1–C8): C9 `OrgRelationshipWriter` and C10 `RelationshipJournal` gate access-switch writes; C12 `DepartmentDirectory` is the only sanctioned department read; C13 exposes full-access-holder lookups and live-responsibility checks. Feature modules still depend only on `contracts`/`registry`, never on each other directly (dependency-cruiser CI rule).
- **Three independent gating axes**, never conflated: C1 `AccessResolver` gates section visibility; C8 `PermissionChecker` gates functional features; C9 `OrgRelationshipWriter` gates access-switch writes (manager/PP/department/department-manager, full-access grant) — a section RW grant and a feature permission are each necessary but neither is sufficient for these four fields.
- A new `employment` module (not `access`) owns employment status and the departure cascade: one atomic transaction, idempotent hooks fanned out via the registry's enumerating `departure-hook` family — `action-items` cancels open items, `mentorship` closes pairs with a system note, `access` revokes the departed person's shared links and full-access grant, `auth` deactivates the account. A departure is blocked while the subject still holds live Reporting-line/PP responsibility or is the sole full-access holder. A `dismissed` employment status caps every viewer's section access at `R`, checked once centrally.
- Temporal history now spans five effective-dated tables (Grade/Position/Department/EmploymentType History plus EmploymentStatusHistory); the Prisma Client Extension auto-firing the timeline writer covers the first four only — a departure is explicitly never a career-timeline event.
- Deployment stays a single containerized stack (docker-compose: backend, frontend, Postgres), one environment.
- **Open contradiction, unresolved:** the PRD states employment status is never visible to Self, but the access matrix grants Self a blanket read over the section containing it — needs a one-line disambiguation before that section's provider is built.

## Cross-Story Dependencies

- Story 1.2 must reflect the narrowed Project-line grant (no S2/S3, S5 = CV+certs, split S7) rather than treating project line as equal-strength Manager access.
- Story 1.20 needs a fifth history table (EmploymentStatusHistory, owned by a new `employment` module) alongside the four originally scoped, plus the departure-cascade orchestrator.
- Story 1.16 (done): seed loads **24 accounts** from bundled JSON manifest (`bootcamp-identities.json`), not TimeTracker Accounting. See `bootcamp-scope-overrides.md`. Original epic note about 26 August / TimeTracker import superseded for seed identity source; TT sync remains Epic 13.
- No existing story yet covers: the Department entity and its admin screen, the organisational-relationship write path (FR-7), full profile access grant/revoke (FR-8), or the relationship/grant journal (FR-9) — these need new or amended stories, and their UX (three new admin screens) has not yet had a design pass.
- Story 1.19 (contracts + registry) and Story 1.18 (auth, C7) must still land before Story 1.6 (`ProfileAssemblerService`) and any `SectionProvider` across every epic.
- Story 1.11's Shared Link mechanism is reused by Epic 6 (Resourcing) for candidate review when a DM lacks standing access.
- Story 1.15's parameterized test strategy extends into Risks (Epic 5), Resourcing (Epic 6), and Action Items (Epic 4).

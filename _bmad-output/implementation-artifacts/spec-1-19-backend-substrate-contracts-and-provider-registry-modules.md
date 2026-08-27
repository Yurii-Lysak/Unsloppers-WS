---
title: 'Backend Substrate — Contracts and Provider Registry Modules'
type: 'feature'
created: '2026-08-27'
status: 'done'
review_loop_iteration: 1
baseline_commit: 'cc2aba57bebe6890b309842318d0af0961321be2' # services/backend HEAD — implementation happens in that submodule
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md'
  - '{project-root}/services/backend/.claude/rules/nest-modules.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** No feature module can be built yet without blocking on another developer, because there are no frozen cross-feature interfaces (C1–C8) and no mechanism for section/field/dashboard providers to register and be discovered at runtime.

**Approach:** Stand up a `contracts` module (C1–C8 as abstract-class DI tokens, each with a Wave-0 stub implementation) and a `registry` module (`DiscoveryService`-based, `@RegisterProvider(family, id)` decorator), then wire a `dependency-cruiser` rule + CI check that forbids feature-to-feature imports.

## Boundaries & Constraints

**Always:**
- `contracts` and `registry` live at `src/modules/contracts/` and `src/modules/registry/`, per ARCHITECTURE-SPINE.md's Structural Seed — the dependency-cruiser rule (below) carries an explicit `(except contracts, registry)` allowlist rather than a single blanket rule.
- Each C1–C8 contract is an abstract class (method signatures only, no implementation, no Prisma import) used directly as both the TS type and the Nest DI token.
- Each contract has a Wave-0 stub class bound via `{ provide: ContractClass, useClass: StubClass }`, swappable later with zero consumer-code change. The C1 `AccessResolver` and C8 `PermissionChecker` stubs default deny: `AccessResolverStub.resolveAudience` returns no granted sections, `PermissionCheckerStub.hasPermission` always returns `false` — never a permissive default, since these are the two security-relevant contracts.
- All DTOs/return shapes crossing HTTP are plain-JSON-serializable generally (no `Map`, no `Date` object, no `undefined`, no class instance — e.g. dates as ISO strings).
- `ContractsModule` and `RegistryModule` are both `@Global()` and `export` every token/service they provide, registered once in `app.module.ts`. (`@Global()` alone only removes the need to re-import the module elsewhere — providers must still be listed in `exports` to be injectable.)
- `@RegisterProvider(family, id)` families are `'section' | 'field' | 'dashboard-summary'`; `id` uniqueness is scoped per-family (two providers may share an `id` under different `family` values).
- `@RegisterProvider`-decorated classes must stay Nest's DEFAULT scope — `DiscoveryService` cannot statically enumerate REQUEST/TRANSIENT-scoped providers, so a scoped provider would be silently missing from the registry.
- Registry indexes providers on `onApplicationBootstrap`; a duplicate `(family, id)` throws and fails bootstrap, naming all colliding providers (two or more).
- `registry.get(family, id)` returns a discriminated result (`{status:'available', provider} | {status:'unavailable'}`) — never `undefined`.
- `dependency-cruiser` is added as a devDependency with a config forbidding `src/modules/<a>/**` → `src/modules/<b>/**` imports (except `contracts`, `registry`), and separately forbidding `src/modules/contracts/**` and `src/modules/registry/**` from importing any other `src/modules/<x>/**` — closing the path for a Wave-0 stub to reintroduce feature coupling. A new GitHub Actions workflow in the backend submodule runs it on `pull_request`.

**Ask First:** none identified.

**Never:** no real section/field provider implementations (future stories consume the registry — this story only builds the substrate), no full CI pipeline beyond the dependency-cruiser gate (build/test/lint stay out of scope), no changes to `users`/`health` modules beyond registering the two new modules in `app.module.ts`.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Happy registration | Class decorated `@RegisterProvider('section','S1')` | Discovered and indexed at bootstrap; retrievable via `registry.get('section','S1')` | N/A |
| Collision | Two classes decorated with the same `(family, id)` | Bootstrap throws, naming both providers | App fails to start |
| Unregistered lookup | `registry.get(family, id)` for an id never registered | Returns `{status:'unavailable'}` | Caller must branch on `status`, never gets `undefined` |
| Boundary violation | `modules/users/**` imports from `modules/health/**` directly | `dependency-cruiser` check fails | CI job fails the PR |

</frozen-after-approval>

## Code Map

- `services/backend/src/modules/users/users.module.ts`, `users.service.ts` -- existing module/DI convention to mirror (module = controllers+providers only, service owns all logic/Prisma access)
- `services/backend/src/modules/health/` -- closer analog for `contracts`/`registry`'s minimal shape (no DTOs/entities/swagger); `contracts` and `registry` are a deliberate, recognized exception to `nest-modules.md`'s standard anatomy (no controller either) — do not "fix" them to match `users`
- `services/backend/src/app.module.ts` -- imports-only root module; add `ContractsModule` and `RegistryModule` here
- `services/backend/package.json` -- NestJS 11.0.1 (`DiscoveryService` available in `@nestjs/core`); add `dependency-cruiser` devDependency + a `depcruise` script
- `services/backend/tsconfig.json` -- strict mode already on, no path aliases; new files follow existing style
- `services/backend/src/modules/*/__tests__/*.spec.ts` -- Jest test convention: tests live in a module's own `__tests__/`, named `<name>.<layer>.spec.ts`

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/src/modules/contracts/*.contract.ts` (8 files, one per C1–C8) -- abstract class per contract with method signatures from ARCHITECTURE-SPINE.md (e.g. `access-resolver.contract.ts` → `AccessResolver.resolveAudience`, `current-user-provider.contract.ts` → `CurrentUserProvider.getCurrentUser`) -- frozen interfaces Wave-1 builds against. C7 `CurrentUserProvider` is the handoff point to Story 1.18 (Authentication), which implements it for real — coordinate the exact method signature with whoever picks up 1.18 before freezing it here.
- [x] `services/backend/src/modules/contracts/stubs/*.stub.ts` (8 files) -- Wave-0 stub implementation per contract, safe hardcoded/no-op behavior -- unblocks feature work before real implementations land
- [x] `services/backend/src/modules/contracts/contracts.module.ts` -- `@Global()` module binding each contract token to its stub via `useClass`
- [x] `services/backend/src/modules/registry/register-provider.decorator.ts` -- `@RegisterProvider(family, id)` using `SetMetadata`/`Reflector`
- [x] `services/backend/src/modules/registry/provider-registry.service.ts` -- uses `DiscoveryService` to scan all providers on `onApplicationBootstrap`, builds a `family -> id -> instance` map, throws on collision, exposes `get(family, id)`
- [x] `services/backend/src/modules/registry/registry.module.ts` -- `@Global()` module importing Nest's `DiscoveryModule` and providing `ProviderRegistryService`
- [x] `services/backend/src/app.module.ts` -- register `ContractsModule`, `RegistryModule` in `imports`
- [x] `services/backend/package.json`, `.dependency-cruiser.cjs` -- add `dependency-cruiser`; rule forbidding `src/modules/<a>/**` → `src/modules/<b>/**`
- [x] `services/backend/.github/workflows/ci.yml` -- new workflow (backend submodule currently has none), triggers on `pull_request`, runs `npm ci` + the `depcruise` script
- [x] `services/backend/src/modules/registry/__tests__/provider-registry.service.spec.ts` -- covers discovery, collision throw, unavailable lookup
- [x] `services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts` -- verifies each token resolves to its stub via DI

**Acceptance Criteria:**
- Given the `contracts` module is stood up, when any feature module is scaffolded, then C1–C8 exist as abstract-class DI tokens with zero business logic or Prisma imports, each with a Wave-0 stub available.
- Given the `registry` module is stood up, when a class is decorated `@RegisterProvider(family, id)`, then it is discovered and indexed at bootstrap.
- Given the `dependency-cruiser` config and CI workflow are wired, when a PR is opened against the backend submodule, then a feature-to-feature import is caught and fails the check.

### Review Findings

**Decision needed:** none outstanding — all 4 resolved by human below (2026-08-27), moved to Patch.

**Patch:**
- [x] [Review][Patch] **(resolved decision)** Move `contracts`/`registry` inside `src/modules/` to match ARCHITECTURE-SPINE.md's Structural Seed — update Boundaries & Constraints (line 24) to drop the "siblings of `src/modules/`" wording and all downstream file paths (Code Map, Tasks, Verification) accordingly. Resolution: follow architecture, not spec. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:24]
- [x] [Review][Patch] **(resolved decision)** Log in `## Spec Change Log` that the discriminated-union return for `registry.get()` (`{status:'unavailable'}`) is the confirmed interpretation of AD-3's "runtime error at first call" wording — spec's mechanism stands, no code-facing change. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:97]
- [x] [Review][Patch] **(resolved decision)** Add an explicit deny-by-default constraint for the C1 `AccessResolver` and C8 `PermissionChecker` Wave-0 stubs — `AccessResolver` stub returns no granted sections, `PermissionChecker` stub's `hasPermission()` always returns `false`. Add to Boundaries & Constraints (Always). [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:26]
- [x] [Review][Patch] **(resolved decision)** Extend the dependency-cruiser rule (line 32) to also forbid `contracts/**` and `registry/**` from importing `modules/**`, closing the stub-reimport loophole. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:32]
- [x] [Review][Patch] Add `exports` to `contracts.module.ts`/`registry.module.ts` task bullets — `@Global()` alone does not make bound tokens/`ProviderRegistryService` injectable elsewhere in Nest; the module must still `export` them. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:62,65]
- [x] [Review][Patch] Add Design Notes clarification: `(family, id)` uniqueness is scoped per-family, not global (two providers can share an `id` under different `family` values). [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:79]
- [x] [Review][Patch] Add Design Notes caveat: `@RegisterProvider`-decorated classes must stay DEFAULT-scoped — `DiscoveryService` cannot statically enumerate REQUEST/TRANSIENT-scoped providers, so a scoped provider would be silently missing from the registry. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:79]
- [x] [Review][Patch] Broaden the "never `Map`" DTO wording (line 27) via a Design Notes note — the real requirement is plain-JSON-serializable values generally (no `Date` objects, no `undefined`, no class instances), not just "not a Map". [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:79]
- [x] [Review][Patch] Add a cross-story coordination note for C7 `CurrentUserProvider` — this story creates the stub/contract that Story 1.18 (Authentication) must later implement; note the handoff expectation rather than treating C7 as an interchangeable 1-of-8 file. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:60]
- [x] [Review][Patch] Code Map: add `modules/health` alongside `modules/users` as a convention reference — `health` is the closer analog for `contracts`/`registry`'s minimal shape (no DTOs/entities/swagger). [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:51]
- [x] [Review][Patch] Verification: note that no existing repo-wide/frontend CI workflow exists to mirror (checked, greenfield), and recommend opening a draft PR to confirm the new workflow actually triggers and passes — it's the first CI workflow in this submodule, so there's no working example to lean on. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:90-92]
- [x] [Review][Patch] Design Notes: clarify "matches the AC's own wording" (line 81) — it refers to AD-2's rule text in `ARCHITECTURE-SPINE.md` ("abstract classes/injection tokens"), not this document's own Acceptance Criteria section. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:81]
- [x] [Review][Patch] Add a one-line note (Code Map or Design Notes) that `contracts`/`registry` are a deliberate, recognized exception to `nest-modules.md`'s standard module anatomy (no controller, no DTO/entities/swagger) — prevents a future contributor from "fixing" them to match `users`. [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:49]
- [x] [Review][Patch] Design Notes: clarify the bootstrap collision error names *all* colliding providers when 3+ share a `(family, id)`, not just two (current wording says "naming both"). [spec-1-19-backend-substrate-contracts-and-provider-registry-modules.md:30]

## Spec Change Log

**2026-08-27 — code review resolution:**
- `contracts`/`registry` relocated from siblings of `src/modules/` to inside `src/modules/` (`src/modules/contracts/`, `src/modules/registry/`), to match `ARCHITECTURE-SPINE.md`'s Structural Seed and AD-1's `(except contracts, registry)` exception clause. All file paths in Code Map/Tasks/Verification updated accordingly.
- Confirmed `registry.get()`'s discriminated-union return (`{status:'unavailable'}`) as the intended interpretation of AD-3's "runtime error at first call" wording — spine wording was ambiguous; this is the ratified reading, not a deviation.
- Added explicit deny-by-default requirement for the C1 `AccessResolver` / C8 `PermissionChecker` Wave-0 stubs (previously unspecified).
- Extended the dependency-cruiser rule to also forbid `contracts`/`registry` → other `modules/**` imports, closing a stub-reimport loophole.

## Design Notes

Contracts use abstract classes (not interface+separate token) so one file per contract serves as both the compile-time type and the runtime DI token — matches AD-2's rule text in `ARCHITECTURE-SPINE.md` ("abstract classes/injection tokens") and avoids a token-registry file. `registry.get()` returns a discriminated union rather than throwing on miss, because the AC requires "unavailable" to be a first-class state a caller can render, not an exception path — confirmed as the intended reading of AD-3's "runtime error at first call" wording during code review (see Spec Change Log).

## Verification

**Commands (re-run 2026-08-27 after the review-fix pass, against the real repo, not an isolated copy):**
- `cd services/backend && npx tsc --noEmit` -- PASS, no errors (`npm run build`'s `nest build` also compiled clean; its `postbuild` `prisma migrate deploy` step separately failed only because no local Postgres is running — pre-existing infra dependency, unrelated to this change)
- `cd services/backend && npm run lint` -- PASS, 0 errors
- `cd services/backend && npm test` (full suite) -- PASS, 6 suites / 37 tests
- `cd services/backend && npm run depcruise` -- PASS, "no dependency violations found (66 modules, 156 dependencies cruised)"

**Manual checks performed:**
- Temporarily imported `HealthController` into `users.service.ts` (an intentional feature-to-feature violation), re-ran `depcruise`: it failed with `error no-cross-feature-module-imports: src/modules/users/users.service.ts → src/modules/health/health.controller.ts`. Reverted the import; `depcruise` is clean again. Confirms the boundary-violation I/O matrix row for real, not just by code inspection.
- Temporarily imported `PrismaService` into `permission-checker.contract.ts` (a post-review-added rule), re-ran `depcruise`: it failed with `error contracts-no-prisma-imports: src/modules/contracts/permission-checker.contract.ts → src/prisma/prisma.service.ts`. Reverted; clean again.
- Read `.github/workflows/ci.yml`: triggers on `pull_request` and `push` to `main`, `actions/checkout` + `actions/setup-node@v4` (node 22) + `npm ci` + `npm run depcruise`, no `working-directory` override needed since this workflow lives at the backend submodule's own repo root. Not yet verified running in GitHub Actions itself (no existing workflow in this submodule to compare against, and no PR opened yet) — recommend opening a draft PR to confirm it actually triggers and passes before relying on it. Per review resolution, this CI job deliberately does not run `npm test` — see Spec Change Log and `deferred-work.md`.

## Suggested Review Order

**Provider Registry — discovery & failure semantics**

- Entry point: scans providers at bootstrap, fails loudly on collisions, invalid ids, and non-DEFAULT scope instead of silent omission.
  [`provider-registry.service.ts:31`](../../services/backend/src/modules/registry/provider-registry.service.ts#L31)

- Lookup never returns `undefined` — `unavailable` is a first-class, renderable state.
  [`provider-registry.service.ts:119`](../../services/backend/src/modules/registry/provider-registry.service.ts#L119)

- `@RegisterProvider(family, id)` decorator and its DEFAULT-scope requirement.
  [`register-provider.decorator.ts:20`](../../services/backend/src/modules/registry/register-provider.decorator.ts#L20)

**C1–C8 Contracts substrate**

- C1 `AccessResolver` — the security-relevant, deny-by-default contract Wave-1 depends on.
  [`access-resolver.contract.ts:47`](../../services/backend/src/modules/contracts/access-resolver.contract.ts#L47)

- C8 `PermissionChecker` — the other security-relevant, deny-by-default contract.
  [`permission-checker.contract.ts:13`](../../services/backend/src/modules/contracts/permission-checker.contract.ts#L13)

- `@Global()` module binding all eight tokens to their Wave-0 stubs.
  [`contracts.module.ts:29`](../../services/backend/src/modules/contracts/contracts.module.ts#L29)

**Wiring**

- `ContractsModule`/`RegistryModule` registered once at the root.
  [`app.module.ts:17`](../../services/backend/src/app.module.ts#L17)

**Boundary enforcement (CI / dependency-cruiser)**

- AD-1's module-boundary rule — no direct feature-to-feature imports.
  [`.dependency-cruiser.cjs:18`](../../services/backend/.dependency-cruiser.cjs#L18)

- Post-review addition: `contracts` may never import Prisma (AD-2 "Always" invariant, now enforced not just documented).
  [`.dependency-cruiser.cjs:49`](../../services/backend/.dependency-cruiser.cjs#L49)

- First CI workflow in this submodule — runs the boundary check on every PR and on push to `main`.
  [`ci.yml`](../../services/backend/.github/workflows/ci.yml)

**Tests**

- Collision, invalid-registration, and metadata-inheritance edge cases added during review.
  [`provider-registry.service.spec.ts`](../../services/backend/src/modules/registry/__tests__/provider-registry.service.spec.ts)

- Confirms the real `RegistryModule` wiring resolves `ProviderRegistryService` (not just a hand-built test module).
  [`registry.module.spec.ts`](../../services/backend/src/modules/registry/__tests__/registry.module.spec.ts)

- Confirms every C1–C8 token resolves to its stub, and the two security-relevant stubs deny by default.
  [`contracts.module.spec.ts`](../../services/backend/src/modules/contracts/__tests__/contracts.module.spec.ts)

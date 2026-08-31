---
title: 'Authentication'
type: 'feature'
created: '2026-08-31'
status: 'done'
review_loop_iteration: 0
baseline_commit: '387176c8fccbd4bb0cc3d4d96f14f60aa4fee700'
context:
  - '{project-root}/_bmad-output/implementation-artifacts/epic-1-context.md'
  - '{project-root}/_bmad-output/implementation-artifacts/bootcamp-scope-overrides.md'
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** The application has no authentication: every API and SPA route is open, and C7 returns a fixed stub user. The 24 seeded users also have identity-anchor hashes, not usable credentials.

**Approach:** Deliver email/password login across backend and frontend. Provision initial password hashes from an operator-supplied environment secret, issue JWTs only in secure HttpOnly SameSite cookies, protect every route except login and health by default, and replace C7's stub with the authenticated request user.

## Boundaries & Constraints

**Always:**
- Keep authentication separate from access-role resolution. Other feature modules obtain `{ userId }` only by injecting C7 `CurrentUserProvider`; controllers never import `auth`.
- Treat `User.hash` as immutable identity metadata. Store credentials in a separate nullable `passwordHash`; seed it only for newly created users or existing users whose value is null, never overwrite an established password.
- Read `JWT_SECRET` and `BOOTCAMP_INITIAL_PASSWORD` through configuration. Never commit, return, or log either secret or any password/hash.
- Sign a finite-lived JWT with `sub=userId`; expose it only through an HttpOnly cookie with matching lifetime. Enable credentialed CORS for the configured origin.
- Default-deny backend and frontend surfaces. Only login and health are public; invalid, expired, missing, or userless sessions resolve to 401 and clear stale browser state.

**Ask First:**
- Any additional public endpoint, cross-site cookie requirement, credential source, or API contract that differs from this spec.

**Never:**
- Never authenticate with the deterministic bootcamp identity `hash`, store plaintext passwords, persist JWTs in local/session storage, add refresh tokens, password reset/change, account registration, SSO, or access-role logic in this story.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|----------------|
| Valid login | Seeded email + correct password | Set session cookie; session endpoint returns `userId`; protected SPA/API loads after reload | N/A |
| Invalid login | Unknown email or wrong password | No cookie or account-disclosure difference | Generic 401 |
| Missing/invalid session | No cookie, malformed/expired JWT, or deleted user | Protected API returns 401; SPA redirects to login | Clear stale cookie/session state |
| Initial credential provisioning | New user or migrated user with null `passwordHash` | Hash environment password and persist it | Missing seed password fails before affected writes |
| Existing credential | User already has `passwordHash` | Seed rerun preserves it | N/A |

</frozen-after-approval>

## Code Map

- `services/backend/prisma/schema.prisma` -- `User` needs nullable `passwordHash`; add a new migration without editing applied migrations.
- `services/backend/src/prisma/seed/seed.service.ts` -- existing idempotent user upsert; provision missing credentials without redefining `hash`.
- `services/backend/src/config/env.validation.ts`, `.env.example` -- established Joi/config surface for JWT and seed-password settings.
- `services/backend/src/modules/contracts/current-user-provider.contract.ts` -- frozen C7 API returning `CurrentUserDto { userId }`.
- `services/backend/src/modules/contracts/contracts.module.ts` -- remove application-level C7 stub binding; retain the stub file for isolated tests.
- `services/backend/src/modules/auth/` -- new login/session service, JWT strategy/global guard, password hashing, real C7 provider, DTOs, Swagger, and public-route metadata.
- `services/backend/src/main.ts`, `src/app.module.ts`, `src/modules/health/health.controller.ts` -- cookie/CORS bootstrap, module registration, and explicit public health route.
- `services/backend/src/modules/users/` -- protected-module pattern; both hashes must remain absent from responses.
- `services/frontend/src/api/client.ts` -- existing interceptor seam; enable cookies and centralize 401 session invalidation.
- `services/frontend/src/contexts/LayoutContext.tsx`, `src/router/index.tsx` -- provider and route-layout patterns for AuthContext and protected routing.
- `services/frontend/src/pages/LoginPage/`, `src/locales/en/translation.json` -- new validated login form and copy; login lives outside `AppLayout`.
- `services/backend/test/users.e2e-spec.ts`, `services/frontend/e2e/app.spec.ts` -- existing integration-test patterns to extend.

## Tasks & Acceptance

**Execution:**
- [x] `services/backend/package.json`, Prisma schema/migration, config files -- add JWT support and separate password credentials with secret-safe configuration.
- [x] `services/backend/src/prisma/seed/` -- hash the operator-supplied initial password for users lacking credentials; preserve idempotency and existing passwords; unit-test matrix cases.
- [x] `services/backend/src/modules/auth/`, contracts/health/app/bootstrap wiring -- implement login, session inspection, logout, cookie JWT validation, global default-deny guard, and C7 provider; add unit and e2e coverage.
- [x] `services/frontend/src/contexts/AuthContext.tsx`, API client, router -- bootstrap session from the HttpOnly cookie, send credentials, redirect protected routes, and clear state on 401.
- [x] `services/frontend/src/pages/LoginPage/`, UI primitives, locale/types -- implement accessible validated login with generic errors; add the repository-prescribed form dependencies and Playwright flow.

**Acceptance Criteria:**
- Given a provisioned user submits valid credentials, when login succeeds, then a JWT cookie is issued and authenticated requests resolve that user's ID through C7 without feature modules importing `auth`.
- Given an unauthenticated browser or API client requests any protected route, when the request is evaluated, then the API returns 401 and the SPA presents login; login and health remain public.
- Given a valid session cookie survives a page reload, when the SPA bootstraps, then it restores the authenticated route without exposing the token to JavaScript.
- Given invalid credentials, when login is attempted, then the response is a generic 401 and neither account existence nor sensitive values are exposed.

## Spec Change Log

## Design Notes

Use the session endpoint as the SPA's source of truth because JavaScript cannot inspect an HttpOnly JWT. Use constant-time password verification and a random per-user salt. SameSite cookies and single-origin CORS provide the browser boundary; broader CSRF infrastructure is deferred unless deployment becomes cross-site.

## Verification

**Commands:**
- `npm run lint && npx tsc --noEmit && npm test && npm run test:e2e && npm run depcruise` in `services/backend` -- all auth, seed, boundary, and regression checks pass against Postgres.
- `npm run typecheck && npm run lint && npm run build && npm run test` in `services/frontend` -- login, redirect, reload persistence, invalid login, and 401-expiry flows pass.
- Manual browser smoke -- login with a seeded account, reload, verify protected navigation, logout, and confirm the JWT is HttpOnly and absent from Web Storage.

## Suggested Review Order

**Application boundary**

- Shared bootstrap makes production routing, CORS, cookies, Swagger, and e2e behavior identical.
  [`bootstrap.ts:12`](../../services/backend/src/bootstrap.ts#L12)

- Global guard defaults every controller route to authenticated unless explicitly public.
  [`auth.module.ts:14`](../../services/backend/src/modules/auth/auth.module.ts#L14)

- Swagger middleware closes the non-controller documentation bypass.
  [`swagger-auth.middleware.ts:12`](../../services/backend/src/modules/auth/swagger-auth.middleware.ts#L12)

**Authentication core**

- Controller exposes only login, session inspection, and logout lifecycle operations.
  [`auth.controller.ts:24`](../../services/backend/src/modules/auth/auth.controller.ts#L24)

- Login verifies indistinguishably and signs only the authenticated user identifier.
  [`auth.service.ts:20`](../../services/backend/src/modules/auth/auth.service.ts#L20)

- Guard clears stale browser cookies whenever protected authentication fails.
  [`jwt-auth.guard.ts:18`](../../services/backend/src/modules/auth/jwt-auth.guard.ts#L18)

- Real C7 provider returns request identity without leaking auth imports to features.
  [`authenticated-current-user.provider.ts:8`](../../services/backend/src/modules/auth/authenticated-current-user.provider.ts#L8)

**Credential provisioning**

- Dedicated nullable credential storage preserves the existing identity-anchor hash.
  [`schema.prisma:23`](../../services/backend/prisma/schema.prisma#L23)

- Seed validates the operator secret before any identity writes.
  [`seed.service.ts:38`](../../services/backend/src/prisma/seed/seed.service.ts#L38)

- Idempotent upsert provisions only missing credentials and preserves established ones.
  [`seed.service.ts:82`](../../services/backend/src/prisma/seed/seed.service.ts#L82)

**Browser session**

- Auth context centralizes session bootstrap, cache purging, login, and logout state.
  [`AuthContext.tsx:27`](../../services/frontend/src/contexts/AuthContext.tsx#L27)

- Route boundary preserves destinations while distinguishing unavailable sessions from anonymous users.
  [`ProtectedRoute.tsx:11`](../../services/frontend/src/components/ProtectedRoute/ProtectedRoute.tsx#L11)

- Login page binds accessible form behavior without exposing the HttpOnly token.
  [`LoginPage.tsx:14`](../../services/frontend/src/pages/LoginPage/LoginPage.tsx#L14)

- Cookie-aware client supplies credentials and broadcasts global unauthorized responses.
  [`client.ts:15`](../../services/frontend/src/api/client.ts#L15)

**Verification**

- Backend e2e proves real bootstrap routes, cookie lifecycle, C7, and default denial.
  [`auth.e2e-spec.ts:31`](../../services/backend/test/auth.e2e-spec.ts#L31)

- Real cross-origin browser flow proves automatic cookie persistence through reload and logout.
  [`real-authentication.spec.ts:12`](../../services/frontend/e2e/integration/real-authentication.spec.ts#L12)

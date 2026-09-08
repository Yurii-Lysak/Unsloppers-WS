## What this is

CI pipeline work for both services, plus a diagnosis of what the new pipelines found when run against `main`. **Companion service PRs are already merged:**

- Backend: [Unsloppers-BE#43](https://github.com/Yurii-Lysak/Unsloppers-BE/pull/43) → `main` (`1e7d532`)
- Frontend: [Unsloppers-FE#24](https://github.com/Yurii-Lysak/Unsloppers-FE/pull/24) → `main` (`03866dd`)

This workspace PR updates the **gitlinks** to those merge commits and carries the write-up: [`known-red-diagnosis.md`](_bmad-output/test-artifacts/known-red-diagnosis.md).

## Status update (2026-09-07)

- ~~**Item 1 — TS2322 build blocker:**~~ **Fixed** on backend `main` via [Unsloppers-BE#45](https://github.com/Yurii-Lysak/Unsloppers-BE/pull/45). BE CI `build` job is green.
- **Items 2–5** — unchanged; see diagnosis doc for owner decisions.

## Decisions still needed

1. ~~**Blocks the Render deploy:** `TS2322`~~ — **Fixed (#45)**
2. **Stale fixtures or route should be exempt?** 3 tests hit 403 on `custom-fields`/`users` routes — [details](_bmad-output/test-artifacts/known-red-diagnosis.md)
3. **Real API defect:** `GET /api/v1/employees?filters=...` 400s — [details](_bmad-output/test-artifacts/known-red-diagnosis.md)
4. **Possible access-control gap:** `GET /api/v1/employees` has no per-row audience narrowing — [details](_bmad-output/test-artifacts/known-red-diagnosis.md)
5. **Not root-caused:** 3 tests 401 in `employee-profile.e2e-spec.ts` — [details](_bmad-output/test-artifacts/known-red-diagnosis.md)

## Merge note

CI is live on both service repos. This PR syncs workspace gitlinks and records the known-red diagnosis. Full backend e2e may still be red (10 pre-existing failures) — informational only; no required status checks configured.

<!-- bmad:context -->
<!-- Verified 2026-08-13 against 26b53fc. Managed by bmad-project-context; edits inside this block are replaced on refresh. Keep anything you want preserved outside the markers. -->

## workspace (people management)

Spec-driven development workspace for the "people management" product, run with the
BMad Method. Application code lives in two git submodules — `services/backend`
(NestJS 11 + Prisma 7 + PostgreSQL) and `services/frontend` (React 19 + Vite 8) —
each a separate GitHub repository. This root repo holds only BMad planning/spec
artifacts, workspace skills, and submodule gitlinks.

## Policy

- Never put application code in this repo — it belongs in `services/backend` or
  `services/frontend`. The root commits only BMad artifacts and gitlink updates.
- Commit each service in its own repository first, then update the workspace
  gitlink — the `commit-and-push-services` skill runs the full sequence.
- Never run `git add` on your own initiative — leave reviewing and staging changes
  to the developer. Only stage/commit when explicitly asked, or when running a
  skill whose job is specifically to stage, commit, and push (e.g.
  `commit-and-push-services`).
- Never commit or push directly to `main` in the workspace or in either submodule.
  Always work on a feature/chore branch and push that branch; if currently on
  `main`, create a branch first before making any changes. Opening a PR is always
  a separate, explicit ask.

## Where things are

- Product owner requirements: `docs/project-requirements.md` (configured as BMad
  `project_knowledge`) — the source scope for all planning work.
- Planning artifacts: `_bmad-output/planning-artifacts/`; implementation artifacts:
  `_bmad-output/implementation-artifacts/`
- Backend agent instructions: `services/backend/AGENTS.md`; deeper guide:
  `services/backend/CLAUDE.md` + per-area rules in `services/backend/.claude/rules/`
- Frontend agent instructions: `services/frontend/AGENTS.md`; deeper guide:
  `services/frontend/CLAUDE.md` + per-area rules in `services/frontend/.claude/rules/`
- BMad config: `_bmad/config.toml`, merged via
  `uv run _bmad/scripts/resolve_config.py --project-root .`

## Running and verifying

- Nothing runs from the root — root `npm test` is a stub. Run service commands from
  inside each service directory, per its own guide.
- Fresh clone: `npm run services:init` pulls both submodules AND checks each out
  onto `main` (`git submodule update` alone leaves them on a detached HEAD — never
  commit inside a submodule without confirming `git branch --show-current` first).
- After a service repo advances (new commits land on its `main`), the workspace's
  recorded gitlink is stale until `npm run services:sync` (or a fresh commit +
  `git add services/<name>`) updates it — `git status` shows the submodule as
  modified content when this happens.

<!-- /bmad:context -->

# Git Submodules Cheatsheet (Agent Reference)

> **Purpose:** Single-page reference for agents working in `Unsloppers-WS`. Read this first instead of searching the repo for submodule workflow.
>
> **Sources:** `AGENTS.md`, `WORKSPACE-SETUP.md`, `.gitmodules`, `package.json`, skills under `.claude/skills/{workspace-status,sync-workspace,commit-and-push-services}/`.

---

## 1. Three-repo layout

| Repo | Path | Remote | Contents |
|------|------|--------|----------|
| **Workspace** | repo root | `Unsloppers-WS` | BMad specs, skills, docs, **gitlinks only** |
| **Backend** | `services/backend` | `Unsloppers-BE` | NestJS 11 API (application code) |
| **Frontend** | `services/frontend` | `Unsloppers-FE` | React 19 SPA (application code) |

**Hard rule:** Application code never lives in the workspace root. Workspace commits only planning artifacts + updated submodule pointers (gitlinks).

Both submodules track branch `main` (see `.gitmodules`).

---

## 2. What belongs where

| Change type | Commit in |
|-------------|-----------|
| NestJS modules, Prisma, tests | `services/backend` |
| React pages, components, e2e | `services/frontend` |
| PRD, specs, BMad output, skills | workspace root |
| After service commits land | workspace root (gitlink update) |

Run service commands (`npm test`, `npm run build`, etc.) **inside** each service directory — root `npm test` is a stub.

---

## 3. npm scripts (workspace root only)

```bash
npm run services:init    # After fresh clone: init submodules + checkout main
npm run services:status  # Show pinned vs checked-out commit per submodule
npm run services:sync    # Pull latest main in each submodule + checkout main
```

**Critical:** Every script appends `git submodule foreach "git checkout main"`. Never run bare `git submodule update` — it leaves detached HEAD.

---

## 4. The #1 footgun: detached HEAD

`git submodule update` (with or without `--init` / `--remote`) checks out a **specific commit**, not a branch.

| Symptom | Meaning |
|---------|---------|
| `git branch --show-current` prints nothing | Detached HEAD — **do not commit** |
| Submodule looks "clean" but no branch name | Same risk |

**Before any work in a submodule:**
```bash
git -C services/backend branch --show-current   # must print "main" or feature branch
git -C services/frontend branch --show-current
```

**Fix detached HEAD:**
```bash
git -C services/backend checkout main
git -C services/frontend checkout main
# Or use: npm run services:init  (fresh clone) / npm run services:sync  (update)
```

---

## 5. Git status meanings (workspace root)

| Indicator | Meaning |
|-----------|---------|
| ` m services/backend` | Submodule has uncommitted changes **or** is at a different commit than the workspace gitlink |
| `-` prefix in `services:status` | Submodule not initialized → run `npm run services:init` |
| Clean workspace + `m` on submodule | Gitlink drift — service advanced, workspace pointer stale |

**Gitlink drift** is normal after committing inside a submodule. Update with `git add services/` in workspace (see commit workflow below).

---

## 6. Branch & commit policy (all 3 repos)

| Rule | Detail |
|------|--------|
| Never commit/push to `main` | Create a feature/chore branch first |
| On `main` with changes | STOP → ask user for branch name → `git checkout -b <name>` |
| Detached HEAD with changes | STOP → ask user how to proceed |
| Never `git add` on your own | Only when user asks, or when running `commit-and-push-services` |
| Never `--force` push | Can destroy remote history |
| Opening a PR | Separate explicit ask — pushing a branch ≠ opening PR |

---

## 7. Agent skills — when to use which

| Skill | Trigger phrases | Modifies repo? |
|-------|-----------------|----------------|
| **`workspace-status`** | "what changed?", "check status", before sync/commit | No (read-only) |
| **`sync-workspace`** | "sync", "pull latest", "update from remote" | Yes (pull only) |
| **`commit-and-push-services`** | "commit", "push", "save changes" | Yes (commit + push) |

**Skill paths:** `.claude/skills/workspace-status/SKILL.md`, `sync-workspace/SKILL.md`, `commit-and-push-services/SKILL.md`

---

## 8. Decision tree

```
User wants to know state?
  └─► workspace-status

User wants latest remote code?
  └─► workspace-status first (must be clean)
      └─► sync-workspace
          1. git pull origin main          (workspace)
          2. npm run services:sync         (submodules)

User wants to save/commit work?
  └─► commit-and-push-services
      1. Check branches (not main, not detached)
      2. Show plan → wait for approval
      3. Commit + push backend (if changed)
      4. Commit + push frontend (if changed)
      5. git add services/ → commit workspace (gitlink + any docs)
      6. Offer PRs (don't auto-open)
```

---

## 9. Commit order (manual or via skill)

Always **service first, workspace last**:

```bash
# 1. Backend (if changed)
cd services/backend
git checkout -b feature/my-work    # if needed
git add .
git commit -m "message"
git push -u origin HEAD

# 2. Frontend (if changed)
cd ../frontend
# same pattern

# 3. Workspace (gitlink + docs)
cd ../..
git add services/                  # updates gitlinks
git add .                          # any workspace files
git commit -m "Update backend/frontend gitlinks for ..."
git push -u origin HEAD
```

One workspace commit can combine gitlink updates with other workspace changes.

---

## 10. Sync order (via sync-workspace)

Requires **clean working trees** in all repos:

```bash
git pull origin main               # workspace
npm run services:sync              # submodules: remote update + checkout main
npm run services:status            # verify
git -C services/backend branch --show-current    # → main
git -C services/frontend branch --show-current   # → main
```

`services:sync` updates local checkouts but **does not** commit a new gitlink. If submodule `main` advanced on remote, the workspace pointer may still be stale until someone commits the gitlink update.

---

## 11. Fresh clone

```bash
git clone https://github.com/Yurii-Lysak/Unsloppers-WS.git
cd Unsloppers-WS
npm run services:init
```

Do **not** use `git submodule update --init` alone.

---

## 12. Common scenarios

### Implemented a backend feature
1. Work in `services/backend` on a feature branch
2. Run tests/build from `services/backend`
3. User asks to commit → `commit-and-push-services`
4. Workspace gitlink commit records the new backend SHA

### Workspace shows `m services/backend` but I didn't edit backend
Submodule HEAD ≠ gitlink. Either commit the gitlink update or reset submodule to pinned commit:
```bash
git -C services/backend checkout $(git ls-tree HEAD services/backend | awk '{print $3}')
```

### Push rejected ("Updates were rejected")
Remote has commits you lack → run `sync-workspace` first (after committing or stashing local work).

### Need to add submodule from scratch (rare)
```bash
git submodule add -b main https://github.com/Yurii-Lysak/Unsloppers-BE.git services/backend
git submodule add -b main https://github.com/Yurii-Lysak/Unsloppers-FE.git services/frontend
```
`submodule add` checks out the branch directly (no detached HEAD).

---

## 13. Quick diagnostic commands

```bash
# All three repos at once
git status
git -C services/backend status && git -C services/backend branch --show-current
git -C services/frontend status && git -C services/frontend branch --show-current
npm run services:status

# Submodule commit vs gitlink
git submodule status
# Leading space = matches gitlink; + = ahead; - = not initialized
```

---

## 14. Cross-references for agents

| Topic | File |
|-------|------|
| Workspace policy | `AGENTS.md` |
| Human setup guide | `WORKSPACE-SETUP.md` |
| Submodule config | `.gitmodules` |
| Backend conventions | `services/backend/AGENTS.md`, `CLAUDE.md` |
| Frontend conventions | `services/frontend/AGENTS.md`, `CLAUDE.md` |
| Product scope | `docs/project-requirements.md` |
| Parallel-team rationale | PRD §8.2 — submodules enable parallel feature work |

---

## 15. Safety checklist (before any git write)

- [ ] Confirmed current branch in each repo with changes (not `main`, not detached)
- [ ] Application code changes are in the correct submodule, not workspace root
- [ ] User explicitly asked to commit/push (or running commit skill)
- [ ] For sync: all working trees clean
- [ ] After service commits: workspace gitlink will be updated
- [ ] PR creation is a separate user request

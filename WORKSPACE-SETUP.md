# AI Bootcamp Workspace Setup

## Step 1: Clone and Initialize Submodules

`backend` and `frontend` are git submodules of this workspace, mounted at
`services/backend` and `services/frontend`. Fresh clone:

```bash
git clone https://github.com/andrii-tarasenko-altex/workspace.git
cd workspace
npm run services:init
```

`services:init` runs `git submodule update --init --recursive` **and then checks
out `main` in each submodule**. This second step matters: `git submodule update`
alone leaves each submodule on a detached HEAD (checked out at a specific commit,
not on any branch) — a classic submodule footgun. If you commit while detached and
later `git checkout main` without noticing, those commits become unreachable except
via `git reflog`. Always confirm you're on `main` (`git -C services/backend branch
--show-current`) before making changes inside a submodule.

If you ever need to (re-)add a submodule from scratch — not needed for a normal
clone, only if setting this up fresh:

```bash
git submodule add -b main https://github.com/andrii-tarasenko-altex/backend.git services/backend
git submodule add -b main https://github.com/andrii-tarasenko-altex/frontend.git services/frontend
```

`git submodule add` (unlike `update --init`) checks out the branch directly, so
this path doesn't have the detached-HEAD issue — only `update --init` does.

---

## Step 2: Package.json Management Scripts

Already set up in `package.json`:

```json
{
  "scripts": {
    "services:init": "git submodule update --init --recursive && git submodule foreach \"git checkout main\"",
    "services:status": "git submodule status",
    "services:sync": "git submodule update --remote && git submodule foreach \"git checkout main\""
  }
}
```

**What these do:**
- `npm run services:init` — initialize both submodules, landed on `main` (use after cloning workspace)
- `npm run services:status` — show each submodule's pinned commit vs. what's checked out
- `npm run services:sync` — pull the latest `main` commit into each submodule, landed on `main` (not detached HEAD)

Note that `services:sync` updates your local submodule checkouts but does **not**
update the workspace's recorded gitlink — that's a separate commit. See
`commit-and-push-services` / `sync-workspace` skills.

---

## Step 3: Claude Code Skills for Git Management

Already committed under `.claude/skills/`:

- `workspace-status` — check status across workspace + both submodules
- `sync-workspace` — pull latest in workspace and both submodules (handles the
  detached-HEAD checkout automatically)
- `commit-and-push-services` — commit/push each submodule, then update the
  workspace's gitlink to point at the new commits

---

## Step 4: Install BMAD for Spec-Driven Development

```bash
# Install BMAD method
npx bmad-method install
```

This sets up the BMad framework for specification-driven development in the workspace.

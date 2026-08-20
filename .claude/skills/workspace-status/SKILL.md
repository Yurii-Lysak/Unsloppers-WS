---
name: workspace-status
description: Check Git status of workspace and both submodules in services/ (backend, frontend). Analyzes uncommitted changes, unpushed commits, detached HEAD, and suggests next actions. Use when user asks to check status, see what changed, before sync/commit, or to verify clean state.
---

# Workspace Status

## Algorithm

### Step 1: Check Workspace Root Status

```bash
echo "=== Workspace Repository Status ==="
git status
git branch -vv
```

Analyze:
- Are there uncommitted changes in workspace?
- Is workspace ahead/behind remote?

### Step 2: Check All Submodules Overview

```bash
echo "=== All Submodules Overview ==="
npm run services:status
```

This shows each submodule's checked-out commit vs. what the workspace has recorded
(a `-` prefix means uninitialized — tell the user to run `npm run services:init`).

### Step 3: Check Each Submodule Detailed — Including Branch

**Backend (services/backend):**
```bash
echo "=== Backend Detailed Status ==="
cd services/backend
git status
git branch --show-current
cd ../..
```

**Frontend (services/frontend):**
```bash
echo "=== Frontend Detailed Status ==="
cd services/frontend
git status
git branch --show-current
cd ../..
```

**If `git branch --show-current` prints nothing for a submodule:** it's on a
detached HEAD (not on any branch). Flag this explicitly — it means `git submodule
update` ran without the branch-checkout step (`npm run services:init`/`services:sync`
handle this correctly; a raw `git submodule update` does not). Committing while
detached risks orphaning commits. Recommend `git -C services/<name> checkout main`
before any further work there, unless the user has a specific reason to be detached
(e.g. inspecting an old pinned commit).

### Step 4: Analyze Results

**Status meanings:**
- **Clean working tree** = no uncommitted changes
- **Changes not staged** / **Changes to be committed** / **Untracked files** — standard git meanings
- **Your branch is ahead/behind** = local vs. remote commits differ
- **Submodule modified content** (shown as `m` in `git status --short` at the
  workspace root) = the submodule's working tree has uncommitted changes, OR it's
  checked out at a different commit than the workspace's recorded gitlink

**Categorize changes:**
- **Workspace changes** = changes in workspace root (not in services/)
- **Service changes** = changes inside services/backend or services/frontend
- **Gitlink drift** = a submodule advanced (new commits pushed) but the workspace's
  recorded pointer hasn't been updated yet — `commit-and-push-services` handles this

### Step 5: Suggest Next Actions to User

**If there are changes in services (backend/frontend):**
- List which services have changes and what changed
- Suggest: "Would you like me to commit and push these changes?"
- Can use `commit-and-push-services` skill

**If there are changes in workspace root:**
- List what changed in workspace (e.g., new files, modified files)
- Suggest: "Would you like to commit these workspace changes?"

**If both workspace and services have changes:**
- Suggest committing services first (using `commit-and-push-services`), which also
  updates the workspace gitlink as part of the same flow

**If a submodule is on detached HEAD:**
- Flag it and suggest `git -C services/<name> checkout main` before proceeding

**If working trees are clean but behind remote:**
- Suggest: "Would you like to sync with remote to get latest changes?" (`sync-workspace`)

**If everything is clean and up-to-date:**
- Confirm: "✅ All repositories are clean and up-to-date"

**If there are conflicts or errors:**
- Explain the problem clearly; suggest manual resolution steps

## Important Notes

- This skill is **READ-ONLY** - never modifies anything
- Always run before sync or commit operations
- Always check submodule branch state, not just `git status` — detached HEAD looks
  "clean" but is a real risk

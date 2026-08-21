---
name: sync-workspace
description: Synchronize workspace and both submodules by pulling latest changes from remote, landing each submodule on main (not detached HEAD). Checks for uncommitted changes first. Use when user asks to sync, pull, update, or get latest changes.
---

# Sync Workspace

## Algorithm

### Step 1: Safety Check - Verify Clean State

**CRITICAL:** Before syncing, check for uncommitted changes using `workspace-status` skill.

```bash
git status
cd services/backend && git status && cd ../..
cd services/frontend && git status && cd ../..
```

**Decision point:**
- **If ANY repository has uncommitted changes** → STOP and tell user:
  - "⚠️ Cannot sync - uncommitted changes detected in [repo name]"
  - "Please commit your changes first or stash them"
  - Suggest: "Would you like me to commit these changes?" (use `commit-and-push-services` skill)
  - **Do NOT proceed with sync**

- **If all clean** → Continue to Step 2

### Step 2: Pull Workspace Repository

```bash
echo "=== Syncing Workspace Repository ==="
git pull origin main
```

**Check result:**
- If pull succeeds → Continue to Step 3
- If conflicts → Stop, explain conflicts, suggest manual resolution

### Step 3: Update Both Submodules — Landed on `main`, Not Detached

```bash
echo "=== Updating Submodules ==="
npm run services:sync
```

This runs `git submodule update --remote` (pulls each submodule's latest `main`)
**followed by `git submodule foreach "git checkout main"`** — the second part is
required, since `git submodule update` alone always leaves each submodule on a
detached HEAD regardless of flags. Do not run the bare `git submodule update
--remote` without the foreach step.

### Step 4: Verify Sync Results

```bash
echo "=== Verification ==="
npm run services:status
git status
git -C services/backend branch --show-current
git -C services/frontend branch --show-current
```

Both branch checks should print `main`. If either prints nothing, something ran
`git submodule update` outside this skill — checkout `main` manually before
continuing.

### Step 5: Report Results to User

**If everything updated successfully:**
- "✅ Workspace synced successfully"
- List what was updated: "Updated workspace: [commits info]"
- List submodules updated: "Backend: [old hash] → [new hash]"
- List submodules updated: "Frontend: [old hash] → [new hash]"

**If nothing changed:**
- "✅ Already up-to-date with remote"

**If sync failed:**
- Explain which step failed, show error, suggest next action

### Step 6: Suggest Next Actions

After successful sync: "You can now start working with the latest code."

## Safety Rules

⚠️ **NEVER sync with uncommitted changes** - data loss risk
⚠️ **Always verify results** after sync, including branch state (not just status)
⚠️ **Stop on conflicts** - don't force through
⚠️ **NEVER run `git submodule update` without the `foreach checkout main` step** -
  leaves the dev on detached HEAD without them realizing it

## Important Notes

- Always run `workspace-status` skill first
- Sync updates to latest remote commits in workspace and both submodules
- Does not create any commits — the workspace's gitlink still needs updating
  separately if a submodule's `main` advanced (see `commit-and-push-services`)
- Safe operation if working tree is clean

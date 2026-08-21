---
name: commit-and-push-services
description: Commit and push changes in service submodules (backend/frontend), then update workspace gitlinks. Never pushes to main directly — commits go on the current feature branch (or a new one if currently on main) in the workspace and in each submodule. Asks user for commit messages, commits each service separately, pushes to remote, updates workspace. Use when user asks to save changes, commit, push, or save work.
---

# Commit and Push Services

## Algorithm

### Step 1: Identify What Changed

```bash
echo "=== Checking for Changes ==="
git status
cd services/backend
git status
cd ../frontend
git status
cd ../..
```

Analyze the output to determine:
- Does **workspace** have changes? What files?
- Does **backend** (services/backend) have changes?
- Does **frontend** (services/frontend) have changes?
- Are changes staged or unstaged?

### Step 1.5: Branch Check — NEVER commit or push to `main`, and never commit on a detached HEAD

For each of the workspace, backend, and frontend that has changes, check the
current branch:

```bash
git branch --show-current
cd services/backend && git branch --show-current && cd ../..
cd services/frontend && git branch --show-current && cd ../..
```

**If any repo with changes prints nothing (detached HEAD):** STOP. This means
`git submodule update` ran without the `foreach checkout main` step. Do not commit
on a detached HEAD — those commits become hard to find later. Ask the user how to
proceed (typically: stash or note the changes, `checkout main`, decide whether to
create a branch and re-apply).

**If any repo with changes is currently on `main` (or `master`):** STOP before
staging anything in that repo. Ask the user: "[repo] is on `main` — I won't commit
there directly. What should I name a branch for this work?" Wait for a name, then
`git checkout -b <name>` before proceeding to Step 2 for that repo.

Repos already on a non-main branch need no action here.

### Step 1.6: Show Plan and Get Approval

**CRITICAL: Human-in-the-loop checkpoint**

```
I see the following changes:

Workspace (branch: <name>):
- New files: [list files]
- Modified files: [list files]
- Untracked files: [list files]

Backend (services/backend, branch: <name>):
- [list changes or "no changes"]

Frontend (services/frontend, branch: <name>):
- [list changes or "no changes"]

Plan:
1. [What will be committed in each repo]
2. [What commit messages will be used]
3. Push each submodule to its current branch (not main)
4. Update the workspace's gitlink to point at the new submodule commits, then
   commit and push the workspace to its own current branch (not main)

⚠️ IMPORTANT: I will commit ALL changes as-is.
- I will NOT create .gitignore entries
- I will NOT skip any files
- I will NOT make decisions about what should/shouldn't be committed

Do you approve this plan?
```

**STOP and wait for user confirmation.**

If user says NO or wants changes → Ask what to modify
If user approves → Proceed to Step 2

### Step 2: Process Backend Changes (if any)

1. **Ask user for commit message:**
   - "I see changes in backend. What commit message should I use?"

2. **Commit and push backend to its current branch:**
   ```bash
   cd services/backend
   git add .
   git commit -m "USER_PROVIDED_MESSAGE"
   git push -u origin HEAD
   cd ../..
   ```

3. **Check push result:**
   - If successful → Report: "✅ Backend committed: [commit hash] on [branch]"
   - If failed → Show error, suggest solution (see Step 7)

**If no backend changes:** skip to Step 3

### Step 3: Process Frontend Changes (if any)

1. **Ask user for commit message:**
   - "I see changes in frontend. What commit message should I use?"

2. **Commit and push frontend to its current branch:**
   ```bash
   cd services/frontend
   git add .
   git commit -m "USER_PROVIDED_MESSAGE"
   git push -u origin HEAD
   cd ../..
   ```

3. **Check push result:**
   - If successful → Report: "✅ Frontend committed: [commit hash] on [branch]"
   - If failed → Show error, suggest solution (see Step 7)

**If no frontend changes:** skip to Step 4

### Step 4: Update Workspace Gitlinks (if any submodule was committed)

If backend and/or frontend were committed in Steps 2-3, the workspace will show
that submodule as modified content (`git status` shows ` m services/<name>`) —
the recorded gitlink is now behind the submodule's actual HEAD.

```bash
git status
```

**If any submodule shows as modified:**

```bash
git add services/
```

Fold this into the same workspace commit as any other pending workspace changes
(Step 5) rather than committing it separately — one commit, one message, covering
everything staged in the workspace repo.

### Step 5: Commit Workspace Changes (if any — including gitlink updates from Step 4)

**If workspace has changes (docs, skills, and/or gitlink updates from Step 4):**

1. **Ask user for workspace commit message** (mention if it includes gitlink updates):
   - "I see changes in workspace[, including updated submodule pointers]. What commit message should I use?"

2. **Commit and push workspace to its current branch:**
   ```bash
   git add .
   git commit -m "USER_PROVIDED_MESSAGE"
   git push -u origin HEAD
   ```

3. **Check push result:**
   - If successful → Report: "✅ Workspace committed: [commit hash] on [branch]"
   - If failed → Show error, suggest solution

**If no workspace changes and no gitlink updates:** skip to Step 6

### Step 6: Offer to Open PRs

For each repo just pushed on a non-main branch, ask (don't do this automatically):

"Want me to open a PR from `<branch>` into `main` for [repo]?"

Only run `gh pr create` (or equivalent) if the user says yes.

### Step 7: Final Verification

```bash
echo "=== Final Status ==="
git status
npm run services:status
```

### Step 8: Handle Errors

**If git push fails:**

- **"Permission denied"** → "Cannot access remote. Check SSH keys or credentials"
- **"Updates were rejected"** → "Remote has changes you don't have. Run `sync-workspace` first"
- **"Protected branch"** → "Branch is protected. You may need to create a Pull Request"
- **Network error** → "Network issue. Check internet connection"

**Suggest recovery:**
- If behind remote: "Would you like me to sync with remote first?" (use `sync-workspace` skill)
- If access issue: "Please check your Git credentials"

### Step 9: Report Final Summary

**If all operations succeeded:**
```
✅ All changes saved successfully:
- Backend: committed and pushed [hash] on [branch]
- Frontend: committed and pushed [hash] on [branch]
- Workspace: gitlinks updated, committed and pushed [hash] on [branch]
```

**If nothing to commit:**
- "No uncommitted changes found"
- "All repositories are already up-to-date"

**If partial success:**
- List what succeeded, what failed and why, and suggest next steps

## Safety Rules

⚠️ **NEVER commit or push directly to `main`/`master`** - in the workspace or in
either submodule; if currently on main, stop and ask for a branch name first
⚠️ **NEVER commit on a detached HEAD** - stop and resolve it first (see Step 1.5)
⚠️ **NEVER use --force flag** - can destroy remote history
⚠️ **Always ask user for commit messages** - never generate automatically
⚠️ **Commit each service separately** - clearer history; the gitlink update rides
  along with the workspace's own commit, not a separate one
⚠️ **NEVER create or modify .gitignore** - user decides what to ignore
⚠️ **NEVER skip files automatically** - commit everything or ask user
⚠️ **Show plan and get approval BEFORE committing** - human-in-the-loop required
⚠️ **Opening a PR is a separate ask** - pushing a branch does not imply opening a PR

## Important Notes

- **Always show plan first, then wait for approval**
- Each repository gets its own commit message from user
- Never create or modify .gitignore automatically
- Never skip files without asking user
- Committing backend/frontend always leaves the workspace's gitlink stale until
  Step 4-5 runs — that's expected, not an error, and is folded into the same
  workspace commit
- Pushes always target the current branch (`git push -u origin HEAD`), never a
  hardcoded `main`
- If push fails, changes remain committed locally (safe)

## Example Flow

```
User: "Save my changes"

Agent:
1. Checks status → sees changes in backend only, workspace clean
2. Checks branch → backend is on `fix/auth-bug` (not main, not detached)
3. Shows plan:
   "I see the following changes:

   Workspace: no changes (yet — will need a gitlink update after backend commits)

   Backend (branch: fix/auth-bug):
   - Modified: src/auth/auth.service.ts

   Plan:
   1. Commit backend changes
   2. Push to fix/auth-bug (not main)
   3. Update workspace gitlink to point at the new backend commit, commit workspace

   ⚠️ I will commit ALL files as-is, no .gitignore changes.

   Do you approve?"

4. Waits for user confirmation
5. User: "Yes, approve"
6. Asks: "What commit message for backend?"
7. User: "Add JWT authentication"
8. Commits and pushes backend to fix/auth-bug
9. Stages services/backend gitlink update in workspace
10. Asks: "What commit message for workspace?"
11. User: "Update backend gitlink for JWT auth"
12. Commits and pushes workspace to its current branch
13. Asks: "Want me to open PRs for backend and/or workspace?"
14. Reports: "✅ All changes saved"
```

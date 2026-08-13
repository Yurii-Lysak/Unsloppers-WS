---
name: commit-and-push-services
description: Commit and push changes in service submodules (backend/frontend), then update workspace gitlinks. Asks user for commit messages, commits each service separately, pushes to remote, updates workspace. Use when user asks to save changes, commit, push, or save work.
---

# Commit and Push Services

## Algorithm

Follow these steps to safely commit and push changes:

### Step 1: Identify What Changed

Check which repositories have uncommitted changes:

```bash
echo "=== Checking for Changes ==="
cd services/backend
git status
cd ../frontend
git status
cd ../..
```

Analyze the output to determine:
- Does **backend** (services/backend) have changes?
- Does **frontend** (services/frontend) have changes?
- Are changes staged or unstaged?

### Step 2: Process Backend Changes (if any)

**If backend has changes:**

1. **Ask user for commit message:**
   - "I see changes in backend. What commit message should I use?"
   - Wait for user's response

2. **Commit and push backend:**
   ```bash
   cd services/backend
   git add .
   git commit -m "USER_PROVIDED_MESSAGE"
   git push origin main
   cd ../..
   ```

3. **Check push result:**
   - If successful → Report: "✅ Backend committed: [commit hash]"
   - If failed → Show error, suggest solution (see Step 6)

**If no backend changes:**
- Skip to Step 3

### Step 3: Process Frontend Changes (if any)

**If frontend has changes:**

1. **Ask user for commit message:**
   - "I see changes in frontend. What commit message should I use?"
   - Wait for user's response

2. **Commit and push frontend:**
   ```bash
   cd services/frontend
   git add .
   git commit -m "USER_PROVIDED_MESSAGE"
   git push origin main
   cd ../..
   ```

3. **Check push result:**
   - If successful → Report: "✅ Frontend committed: [commit hash]"
   - If failed → Show error, suggest solution (see Step 6)

**If no frontend changes:**
- Skip to Step 4

### Step 4: Update Workspace Gitlinks

After pushing services, workspace will show submodules as modified (new commit hashes).

Check workspace status:
```bash
git status
```

**If submodules show as modified:**

```bash
# Stage gitlink updates
git add services/

# Commit workspace
git commit -m "Update services to latest commits"

# Push workspace
git push origin main
```

Report: "✅ Workspace gitlinks updated"

### Step 5: Final Verification

Verify everything is clean and pushed:

```bash
echo "=== Final Status ==="
git status
npm run services:status
```

### Step 6: Handle Errors

**If git push fails:**

Analyze the error and provide specific guidance:

- **"Permission denied"** → "Cannot access remote. Check SSH keys or credentials"
- **"Updates were rejected"** → "Remote has changes you don't have. Run `sync-workspace` first"
- **"Protected branch"** → "Branch is protected. You may need to create a Pull Request"
- **Network error** → "Network issue. Check internet connection"

**Suggest recovery:**
- If behind remote: "Would you like me to sync with remote first?" (use `sync-workspace` skill)
- If access issue: "Please check your Git credentials"

### Step 7: Report Final Summary

**If all operations succeeded:**
```
✅ All changes saved successfully:
- Backend: committed and pushed [hash]
- Frontend: committed and pushed [hash]  
- Workspace: gitlinks updated [hash]
```

**If nothing to commit:**
- "No uncommitted changes found"
- "All repositories are already up-to-date"

**If partial success:**
- List what succeeded
- List what failed and why
- Suggest next steps

## Safety Rules

⚠️ **NEVER use --force flag** - can destroy remote history  
⚠️ **Always ask user for commit messages** - never generate automatically  
⚠️ **Commit each service separately** - clearer history  
⚠️ **Check branch before pushing** - should be on main

## Important Notes

- Each service gets its own commit message from user
- Workspace commit message is automatic: "Update services to latest commits"
- If push fails, changes remain committed locally (safe)
- User can always fix issues and re-run this skill

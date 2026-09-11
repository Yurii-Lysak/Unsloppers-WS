---
title: 'Fix Vercel SPA routing 404s on refresh, unknown paths, and new-tab navigation'
type: 'bugfix'
created: '2026-09-11'
status: 'done'
review_loop_iteration: 0
context: []
route: 'one-shot'
---

# Fix Vercel SPA routing 404s on refresh, unknown paths, and new-tab navigation

## Intent

**Problem:** On the deployed Vercel frontend, refreshing a client-side route, opening a parameterized route (e.g. an employee profile) in a new tab, or hitting an unknown path all land on Vercel's own static 404 page instead of the app — because Vercel serves the raw request path from disk and finds no matching physical file, so it never falls back to `index.html` and React Router's existing `*` → `/` redirect never gets a chance to run.

**Approach:** Add a catch-all `rewrites` rule to `services/frontend/vercel.json` so every request falls back to `/index.html`, letting the already-correct client-side router (which already has a `*` → `<Navigate to="/" replace />` route) handle both direct deep-link loads and unknown-path redirection. No React Router changes were needed — the router-side fix already existed.

## Suggested Review Order

- The actual fix: Vercel now falls back to the SPA shell for any path with no matching static file, so client-side routing (including the existing `*` → `/` redirect) can run.
  [`vercel.json:3`](../../services/frontend/vercel.json#L3)

- Documents the rewrite as a required gotcha so it isn't dropped in a future `vercel.json` edit.
  [`CLAUDE.md:47`](../../services/frontend/CLAUDE.md#L47)

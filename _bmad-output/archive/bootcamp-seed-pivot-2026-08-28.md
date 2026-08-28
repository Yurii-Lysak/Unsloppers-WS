# Bootcamp seed pivot — 2026-08-28

## What changed

| Before | After |
|---|---|
| Identity source: TimeTracker `POST /api/accounting/report` | Bundled manifest: `bootcamp-identities.json` |
| Floor: 500 identities (Ask First below) | Fixed list: **24** bootcamp test accounts |
| Required VPN + API keys to seed | Seed runs offline from JSON — no TimeTracker |
| Talents endpoint for project membership warnings | Deferred to Epic 13 sync |

## Why

The bootcamp test task pivoted from a 500+ pseudonymized population to a **24-account**
curated set. TimeTracker dev data never reached the 500 floor (2–16 identities depending
on endpoint/month). The original 500+ requirement in `project-requirements-v2.md` §1/§7
targets **performance validation** (All Employees at scale — Story 3.7), not the seed
manifest size for this iteration.

## Identity manifest shape

Each record:

```json
{
  "id": 1,
  "email": "person@example.com",
  "name": "Display Name",
  "hash": "<64-char hex>",
  "countryCode": "UA"
}
```

Fields match the TimeTracker `Employee` identity anchor (minus monthly `days[]`).

`hash` is not in the source CSV — generated deterministically as
`SHA256("bootcamp-seed-v1:" + lower(email))` (uppercase hex) so reruns stay stable.

## Locations

- **Source CSV (workspace):** `docs/bootcamp-seed-accounts-source.csv`
- **Runtime (backend submodule):** `services/backend/src/prisma/seed/data/bootcamp-identities.json`
- **Archive (workspace):** `_bmad-output/archive/bootcamp-seed-identities.json` + `bootcamp-seed-accounts-source.csv`

Keep both files in sync when the delivered account list changes.

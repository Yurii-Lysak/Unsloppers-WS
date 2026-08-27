# Input Reconciliation — review-edge-case-hunter.md / validation-report.md

- **Input:** `review-edge-case-hunter.md` (22 raw findings) + `validation-report.md` (same 22 findings, grouped by threatened guarantee with suggested fixes)
- **Against:** `prd.md` (post-update)

## Coverage walk

| # | Finding (validation-report.md grouping) | Landed in prd.md |
|---|---|---|
| 1 | No precedence rule for multi-role viewers (FR-1/FR-2) | FR-1 new consequence bullet (union rule, D13) |
| 2 | Self-managed department loophole (FR-7) | FR-7 new consequence bullet (D15) |
| 3 | Manager-chain cycle (FR-7) | FR-7 new consequence bullet (D15) |
| 4 | Concurrent conflicting access-switch writes (FR-7/FR-9) | FR-7 new consequence bullet (D15) |
| 5 | Orphaned department on manager-field clear (FR-7) | FR-7 new consequence bullet (D15) |
| 6 | Concurrent mutual revocation drives full-access holders to zero (FR-8) | FR-8 new consequence bullet (D16) |
| 7 | Departure of sole remaining full-access holder bypasses the floor (FR-8/FR-62) | FR-8 + FR-62 new consequence bullets (D16) |
| 8 | Shared links outlive their creator's departure (FR-63/SM-6) | FR-63 body + consequence bullet, SM-6 updated (D17) |
| 9 | Saved views become permanently frozen on creator departure (FR-13) | FR-13 new consequence bullet — resolved as ownerless, not transfer (judgment call, logged as divergence) |
| 10 | Double-departure race on a shared mentorship pair (FR-63) | FR-63 consequence bullet (D17) |
| 11 | Concurrent cancellation race on action items (FR-22/FR-63) | FR-22 consequence bullet, cross-referenced from FR-63 (D17) |
| 12 | Shared-link scope doesn't re-clamp to creator's current access (FR-6) | FR-6 consequence bullet rewritten (D14) |
| 13 | No un-approve path for an approved candidate (FR-28-31) | FR-30 new consequence bullet (D18) |
| 14 | Headcount editable below filled count (FR-31) | FR-31 new consequence bullet (D18) |
| 15 | No cap enforcement at zero remaining (FR-31) | FR-31 new consequence bullet (D18) |
| 16 | UM/DM departure mid-decision (FR-28-33) | FR-30 (DM) + FR-33 (UM) new consequence bullets (D18) |
| 17 | Stale routing on UM reassignment (FR-33) | FR-33 new consequence bullet (D18) |
| 18 | 4-hour withdrawal clock resets on brief recovery (FR-58) | FR-58 consequence bullet rewritten — cumulative rolling window (D19) |
| 19 | 15-minute grant window has the same flapping gap (FR-58) | FR-58 consequence bullet rewritten — single-success confirms (D19) |
| 20 | No server-side consent gate on pair creation (FR-42) | FR-42 new consequence bullet (D20) |
| 21 | Concurrent pair-ending race (FR-43) | FR-43 new consequence bullet (D20) |

*(21 fix rows above — the validation-report.md groups the 22 raw findings into 21 distinct "Fix:" recommendations; the self-managed-department and manager-cycle findings share one grouping row in the report's "Access-model precedence and self-reference" section but were logged as two separate findings in the raw `review-edge-case-hunter.md`. Both raw findings are independently covered above under rows 2 and 3.)*

## Divergence flagged

Row 9 (saved views) does not adopt the report's literal fix ("transfer to a current manager/PP holder") — a saved view is not a profile-scoped entity, so there is no manager/PP *of a view* to transfer to. Resolved instead as **ownerless on departure**: usable by anyone it was shared with, not editable/deletable until an HR Admin (or *manage custom fields* holder) adopts it. Recorded in `decisions.md`'s appendix and cross-referenced from `prd.md` §9/§10. Flagged for the user to confirm if a hard-retire (auto-archive) is preferred instead.

## Verdict

All 22 raw findings land in `prd.md`, consistent with the same resolutions already ratified as D13–D20 in the canonical SPEC's `decisions.md` this session. No gaps found; the one deliberate divergence (saved views) is explicitly recorded, not silent.

# Input Reconciliation: SPEC v2 → PRD

**Run:** 2026-08-26  
**Input:** `_bmad-output/specs/spec-people-management-platform/SPEC.md` + companions (re-derived from `docs/project-requirements-v2.md` v1.5)  
**Target:** `prd.md`

## Summary

PRD updated to match the v2 spec package. No conflicts with prior PRD decisions — v2 supersedes v1 where they differ, and prior user-confirmed defaults (D3, D5, appendix items) remain valid where v2 doesn't change them.

## Gaps addressed in this update

| Area | v1 PRD gap | v2 resolution in PRD |
|---|---|---|
| Capabilities | 13 features (CAP-1–13) | Added §5.14 Employment Status & Departure (CAP-14), FR-61–65 |
| Manager access | Single "Manager line" tier | Split into Reporting line / Project line (FR-1, FR-2, Glossary) |
| HR Admin | Implied full data access | Explicitly grants no data access; Full profile access as separate grant (FR-5, FR-8) |
| Org relationships | Inline-editable implied | Access switches via dedicated permission + journal (FR-7, FR-9, FR-12) |
| Colleague S10 | Leave type visible | Dates only, type hidden (FR-3) |
| Campaign sender | Not mentioned | Campaign-sender exception added (FR-3, FR-22) |
| Resourcing | Manual shared links, optional project | Auto-generated links, department routing, headcount slots, comp-band scoping, Unassigned bucket (FR-28–33) |
| PeopleForce | Full integration required | Candidate ID + link required; prefill good-to-have (FR-59) |
| Timetracker | Qualitative fail-safe | 15-min grant / 4-hour withdrawal (FR-58, D3) |
| Risk | No active definition | Active = above low; leaver ≠ dismissed (FR-24–27) |
| Feedback | Period comparison | Removed; period filtering only (FR-51) |
| Timeline | Departure as event possible | Explicitly never a timeline event (FR-35) |
| Self-service leaves | Balances implied | Balances never in-platform (FR-19) |
| Non-goals | 11 items | Added provisioning, SSO, leave balances, PeopleForce vacancy sync, prefill |
| Open questions | None blocking | D11 (role defaults), D12 (timetracker sync model) surfaced in §9 |
| FR count | FR-1–55 | Renumbered to FR-1–65 (new FRs for access switches, full access, journal, departure) |

## Items intentionally not duplicated in PRD

- Interface contract signatures (C1–C11) — live in `interface-contracts.md`
- Architecture spine module/entity design for CAP-14 — flagged in spec memlog as architecture follow-up
- Delivery waves and story-level AC — remain input for `bmad-sprint-planning` / `bmad-create-epics-and-stories`

## Residual gaps (non-blocking)

- UJ-2 updated narratively for auto-generated links but not re-validated with a live PM/DM
- Default functional-role permission assignments (D11) logged as open — PRD references the pending defaults without deciding them

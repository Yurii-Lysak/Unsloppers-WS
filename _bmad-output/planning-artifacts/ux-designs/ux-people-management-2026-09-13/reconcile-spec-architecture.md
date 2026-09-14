# Reconcile — SPEC + Architecture → spines (2026-09-13)

Sources: `_bmad-output/specs/spec-people-management-platform/SPEC.md`, `access-model.md`, `interface-contracts.md`, `decisions.md`; `_bmad-output/planning-artifacts/architecture/architecture-people-management-2026-08-21/ARCHITECTURE-SPINE.md`.

Coverage: access-matrix strictness → absent-not-hidden guardrail in both spines; colleague whitelist → directory + search + palette rules; S7 flags → composer component + flag matrix; resourcing lifecycle + headcount guards → Flow 1–2 + request-detail mock; campaign atomic freeze → Flow 3 + campaign-detail mock; mentorship close-gate → Flow 4; timeline single-writer + skip-annotation → Timeline component; departure cascade → read-only state + admin tab; frontend constraints (shadcn inheritance, permission-driven rendering, generated types, TanStack server-state only) → Foundation + Do's/Don'ts.

Deltas from validation (applied): button-primary corrected to inherited primary; measured contrast pairs (all ≥ 4.5:1); row keyboard/SR + `aria-disabled` contracts; S7/S8 composer rows.

Open: Appendix defaults (uploads, IDP reopen, assessor shape, timeline delete, post-access-loss cancel), offline queue-vs-reject — Open items, pending confirm.

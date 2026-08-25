# Input Reconciliation — PRD vs. SPEC

Checked: `prd.md` (all sections 0–9) against `SPEC.md` (Why, all 13 capabilities' intent+success, ~20 Constraints bullets, all 9 Non-goals, Success signal, Assumptions D1–D8), `access-model.md` (access roles, functional-role table + rules, section matrix, matrix rules 1–7), `interface-contracts.md` (C1–C6, filtered for product-behavior relevance), and `decisions.md` (D1–D8 plus the 8 appendix defaults). Every capability's FR set, every non-goal, every constraint, and every decision/default was traced to its PRD location (Features §4, Non-Goals §5, Success Metrics §7, Assumptions Index §9) and checked for meaning-fidelity, not wording match.

Overall the PRD is a faithful, well-grounded narration of the SPEC — all 13 capabilities' FRs map cleanly, all 9 non-goals are reproduced near-verbatim, all 8 low-stakes appendix defaults are referenced, and D1–D4/D6/D8 are represented as testable FR consequences. The gaps below are the exceptions.

## Gaps

- **Mentor field's stricter header-visibility rule (D5) is absent, and the Glossary's Colleague definition contradicts it.**
  SPEC claim: `access-model.md` rule 6 and `decisions.md` D5 (PO-confirmed 2026-08-21) state the profile header's mentor field (part of S1) is visible only to Manager line and PP, overriding the general S1 Colleague-R grant. CAP-1's own intent line and CAP-9's intent line ("The mentor is shown in every profile header alongside manager and PP") both call this out as a real, testable product behavior.
  Where the PRD should reflect it: Feature 4.1 (Access Control) FR-1/FR-2 consequences, or Feature 4.9 (Mentorship Hub), or the Glossary's "Colleague" entry.
  What's actually there: FR-1–FR-6 never mention the mentor field or the header exception. Worse, the PRD's own Glossary (§3) defines **Colleague** as seeing "a strict whitelist (S1, S10 including leave type, S11 project name only)" — stating colleague access to S1 without carving out the mentor-field exception, which reads as (and technically is) a misstatement of D5's override. §8/§9 acknowledge D5 was "resolved this session" but never restate what it resolved to.
  Severity: moderate.

- **Foundation-phase and intelligent-repository governance constraint is missing.**
  SPEC claim (Constraints, last bullet): the intelligent repository is mandatory documentation, and a foundation phase (prototyping/design approach, best-practice research, test architecture, named owners per topic) must run *before* feature implementation starts, with findings aligned and written down first; inter-team communication and status must be captured for analysis.
  Where the PRD should reflect it: §1 Vision's process paragraph, or §7 Success Metrics (alongside SM-5/SM-6, which cover adjacent process claims).
  What's actually there: SM-6 covers only "specs match shipped behavior at demo time" — nothing about a gating foundation phase preceding implementation, or about capturing inter-team communication/status. No other section mentions it either.
  Severity: moderate.

- **Success signal's explicit deployment requirement ("not running on someone's laptop") is missing.**
  SPEC claim (Success signal): "The platform is deployed and demonstrable — not running on someone's laptop..."
  Where the PRD should reflect it: §7 Success Metrics, most naturally alongside SM-2/SM-3 which already use "demonstrated live"/"demonstrated build" language.
  What's actually there: SM-2 and SM-3 say "demonstrated," which is compatible with but doesn't state the deployment requirement — a local demo on a laptop would satisfy the PRD's wording as written but not the SPEC's.
  Severity: minor.

- **BMad-as-mandated-framework constraint is missing.**
  SPEC claim (Constraints): BMad is the mandated planning/process framework for at least the start of the project; migrating away from it must be a recorded deliberate decision, never silent drift.
  Where the PRD should reflect it: §1 Vision's process paragraph (which already discusses spec-driven, parallelized delivery) or §0 Document Purpose.
  What's actually there: §0 and §1 mention BMad workflows as downstream consumers/tools but never state BMad's mandated status or the "no silent drift" rule.
  Severity: minor.

- **Pseudonymized-data-only constraint for non-production environments is missing.**
  SPEC claim (Constraints): non-production environments use pseudonymized data only (real structure/volume, substituted names/contacts) — never real personal data in agent contexts, logs, screenshots, or the repository.
  Where the PRD should reflect it: no natural single home in the current structure; closest would be a data-handling note near §1 Vision or as an NFR.
  What's actually there: no mention anywhere in the PRD.
  Severity: minor.

- **Temporal/time-bounded modeling of grade, position, department, and employment type is not represented as a stated requirement.**
  SPEC claim (Constraints): grade, position, department, and employment type are temporal — modeled as time-bounded records, not scalar fields plus a bolted-on audit log.
  Where the PRD should reflect it: Feature 4.2 (custom fields/directory) or 4.7 (Career Timeline), or Feature 4.3 FR-13 (own employment summary).
  What's actually there: FR-28 covers timeline *events* generated when these fields change, which gives user-visible history, but nothing in the PRD states the underlying time-bounded-record modeling requirement — arguably an architecture-level concern the PRD legitimately leaves for `bmad-architecture`, but flagged since it's an explicit SPEC constraint with no PRD echo at all.
  Severity: minor.

## Verdict

The PRD faithfully represents the SPEC's product substance with no critical gaps or contradictions in behavioral scope; the handful of misses are process/governance constraints (foundation phase, BMad mandate, pseudonymized data, deployment requirement) plus one specific access-control nuance (D5's mentor-field header exception, which the Glossary's Colleague definition actually understates) — all fixable with small, targeted additions rather than a rewrite.

# Epic 11 Context: Feedback

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Deliver permission-gated feedback records on employee profiles (section S8) with a single management-only / shared-with-employee visibility flag, chronological viewing, and a named-colleague feedback-request flow built on Form Campaigns. Managers and PPs can capture sensitive feedback privately and selectively share individual records with the employee; employees see only records explicitly flagged for them. This epic closes the gap between access resolution (C1 already grants S8) and actual persistence, API, and profile UI.

## Stories

- Story 11.1: Record Feedback with a Visibility Flag
- Story 11.2: View Feedback Over Time and Compare Periods
- Story 11.3: Request Feedback from Named Colleagues via a Form Campaign

## Requirements & Constraints

- Feedback records carry subject employee, author, `recordedAt` (date), context (project/event/period label), body, and a single visibility flag (`sharedWithEmployee`) defaulting to management-only.
- Flipping visibility to shared-with-employee makes that record immediately visible on the employee's own profile — no stale grant.
- Colleagues never receive read access to feedback about someone else (S8 is `none` for Colleague audience).
- Self sees only records explicitly shared with them; management-only records are absent from every Self surface (profile `records` and direct API). The S8 section key remains present for Self with `accessLevel: 'R'` and `records: []` when no shared records exist.
- Reporting line, PP, and project line (including PM) hold S8 `RW` — unlike S7, there is no PM-only read carve-out and no `hasHiddenNotes` gate.
- A functional role may grant *create feedback* (`CREATE_FEEDBACK`) as a feature permission for **POST only** — it never widens read access; reads and PATCH/DELETE still follow C1 section grants (`RW` required for mutations beyond create).
- Multi-audience union (access-model Rule 10): when union yields `S8: 'RW'`, apply the full RW path — all records, no PM carve-out (unlike S7).
- Joining-interview feedback is a feedback record (S8), not a document in S5 — **Story 11.1 does not model a separate joining-interview type**; any dedicated capture flow is a later story.
- Completing a feedback-request campaign never auto-creates a Feedback record — the requester authors one manually after reviewing external form responses (Story 11.3).
- Shared profile links may include S8 only when explicitly enabled; it is excluded by default alongside other sensitive sections. S7 remains never-share.

## Technical Decisions

- Backend module: `feedbacks`, owning CAP-11. Data model: `FeedbackRecord` linked to `Employee` (subject and author).
- Registers `SectionProvider` for S8 via `@RegisterProvider('section', 'S8')` and the shared Provider Registry pattern established in Epic 1.
- Profile assembly calls the S8 provider only when access resolution grants the section; unauthorized sections are never fetched.
- Parallel REST routes under `employees/:employeeId/feedbacks` mirror management-notes and risks controllers.
- Record-level flag filtering happens server-side in the provider — never fetch-then-strip in controllers.
- Story 11.3 reuses Epic 10's campaign flow frontend-orchestrated (no backend `feedback → campaigns` coupling).

## UX & Interaction Patterns

- **Visibility Flag Toggle (single-flag variant)** — one switch, "Shared with employee," off by default; immediate PATCH on flip; RW viewers only.
- **Feedback Panel** — S8 profile section: chronological list (`recordedAt`, author, context, body) plus add/edit for RW viewers. Self (`R`) sees the section with shared records only — empty `records: []` when none are shared, never omitted. Period-comparison columns and "Request feedback…" belong to Stories 11.2 and 11.3.
- **Voice** — direct and precise; feedback is sensitive HR data.
- **i18n** — all user-facing strings are translation keys.

## Cross-Story Dependencies

- **Epic 1** — C1 access resolution, profile assembler, section provider registry, colleague whitelist (S8 `none`).
- **Epic 2 Story 2.5** — Self profile consumes shared feedback records created here; record-level filtering must match.
- **Epic 10** — Story 11.3 builds on form campaigns; Story 11.1 does not depend on campaigns.
- **Epic 9** — mentorship closing feedback stays on the pair record (D6), not routed through S8.

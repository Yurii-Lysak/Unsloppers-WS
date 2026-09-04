# Epic 4 Context: Action Items and Tasks

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Introduce the platform's single task entity — action items — sourced manually by managers and People Partners or generated automatically when a form campaign activates. Assignees complete their own items; authors cancel with a reason; overdue state is derived consistently everywhere items appear (profile S14, self-service, dashboards, campaign tracking). Epic 4 owns the `action-items` module, C6 `ActionItemCreation`, and S14 `SectionProvider`; Epic 10 consumes C6 at campaign activation.

## Requirements & Constraints

- One task entity with fields: title, short description, assignee, author, due date, optional link, status, completion date, and source (`manual` or `campaign`).
- Lifecycle: `open` → `completed` (assignee only) or `cancelled` (author with reason). Terminal states are final.
- Manual creation: manager (UM/DM/PM) or PP, from the assignee's profile or the creator's dashboard, for anyone in their C1 access scope. Holders of the *create action items* functional permission may do the same **within** existing C1 access scope — the permission never widens visibility.
- Campaign activation generates exactly one item per frozen recipient, atomically, carrying campaign title, sender, due date, and link.
- Overdue is never a stored flag — derive at render time from `status === open` and `dueDate < today` using the shared `Clock` service; identical derivation on every surface.
- S14 access follows the access-model matrix; Colleague `none` except the campaign-sender exception (Rule 7). S14 is in the shared-link never-share set.
- Items appear on the assignee's profile (S14), in self-service, and on dashboards (authored items widget for managers tracking delegation).

## Technical Decisions

- **`action-items` module** owns CAP-4: Prisma `ActionItem` model, C6 `ActionItemCreation` implementation (replacing the Wave-0 stub), `@RegisterProvider('section', 'S14')`, parallel REST routes under `employees/:employeeId/action-items`, and a `departure-hook` registrant (AD-18 — auto-cancel open items on departure; wired in a later story or alongside 4.2).
- **C6 contract** (`action-item-creation.contract.ts`): `createActionItem({ assigneeId, authorId, title, description?, dueDate, link?, source, campaignId? }) → ActionItem`. `campaigns` is the only cross-module caller besides the module's own controller.
- **S14 access levels** (per `access-resolver.service.ts`): Self `R` (own items + mark complete in 4.2); ReportingLine and PP `RW`; ProjectLine `RW` when the viewer matched the DM leg, `R` on a PM-only project-line match (`access-model.md` Rule 2). Colleague `none` except Rule 7 campaign-sender exception.
- **Create authorization:** primary gate is C1 `S14: 'RW'` on the assignee via `SectionAccessGate`. Alternate path: C8 `create_action_items` permission **plus** non-`none` S14 access on the assignee — including PM-only ProjectLine `R`, which is the primary create path for PMs. Permission alone never grants Colleague-only viewers write access.
- **S14 provider filtering:** Self sees only items where `assigneeId === subjectId`; RW viewers see all items assigned to the subject (all statuses in 4.1); sorted by `dueDate` ascending then `createdAt`.
- **Dashboard data:** authored open items (`authorId === viewer`) whose assignee still has non-`none` S14 from the viewer's live C1 resolution (ReportingLine, PP, or ProjectLine) — queryable for Epic 12's shared dashboard engine; no bespoke dashboard UI in Story 4.1.

## UX & Interaction Patterns

- Inline creation on the assignee's Employee Profile S14 section (UJ-1 step 5) — no separate creation page. Optional link field; title and due date required.
- Action Item Row component (profile S14, dashboard, My Profile): checkbox for assignee completion (4.2), Overdue Indicator for past-due open items (4.3), Status Badge on completion.
- Save failure: destructive Toast retaining draft text (EXPERIENCE.md State Patterns).

## Stories & Dependencies

| Story | Summary | Depends on | Enables |
|-------|---------|------------|---------|
| **4.1** Manually Create an Action Item | Persistence, C6, S14 provider, create/list APIs | Epic 1 (C1, `SectionAccessGate`, profile assembler, C8 `PermissionChecker`, `create_action_items` seed) — complete | 4.2–4.4, Epic 2 Story 2.5 (self-service S14 Self path), Epic 12 dashboard widget data |
| **4.2** Complete and Cancel Action Items | Complete/cancel mutations | 4.1 persistence and routes | Terminal lifecycle |
| **4.3** Overdue Highlighting | Overdue derivation via `Clock` | 4.1 list surfaces | Consistent overdue everywhere |
| **4.4** Auto-Generate on Campaign Activation | C6 campaign-sourced creation | 4.1 C6 implementation and `ActionItem` model | Epic 10 campaign activation |

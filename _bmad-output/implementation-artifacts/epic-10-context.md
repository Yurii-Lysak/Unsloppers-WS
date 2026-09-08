# Epic 10 Context: Forms & Survey Campaigns

<!-- Compiled from planning artifacts. Edit freely. Regenerate with compile-epic-context if planning docs change. -->

## Goal

Give PPs, managers, and holders of a dedicated functional permission a way to distribute an externally-hosted form (Microsoft Forms, Google Forms, etc.) to a targeted, frozen group of employees, and to track who responded — without the platform ever hosting, embedding, or reading the form itself. A campaign is created in draft, has its recipient audience built from the same filter engine as the All Employees directory, and is activated in one atomic step that freezes the audience and generates one action item per recipient. From then on, "completion" is nothing more than each recipient marking their own generated action item done — the system has no way to verify the external form was actually filled in, by design. This is also the reusable building block Epic 11's named-colleague feedback-request flow is built on top of.

## Stories

- Story 10.1: Create a Form Campaign
- Story 10.2: Build and Freeze Campaign Audience
- Story 10.3: Activate a Campaign
- Story 10.4: Track Campaign Completion

## Requirements & Constraints

- Campaign fields: title, short description, purpose, external form link, due date. Creatable by a PP, a manager, or anyone holding the *create form campaigns* functional permission — the permission unlocks the feature but never widens who that person can target.
- A campaign starts in draft; all fields stay fully editable and nothing is generated or sent to anyone until activation.
- Audience is built with the same filter/preview engine used by the All Employees directory (including starting from a saved view), with individual add/remove after the filter resolves. Audience building is always bounded by the creator's own resolved access — they can never preview or target someone their access role wouldn't show them elsewhere.
- The audience is a snapshot, not a live query: it is frozen at the exact moment of activation. Anyone who starts matching afterward is never added; anyone who stops matching is never removed.
- Activation is a single atomic (or reliably idempotent) transition: it freezes the audience and generates exactly one action item per recipient (carrying the campaign's title, sender, due date, and link), or it fails cleanly and the campaign remains a fully-editable draft — never a partially-activated state. Once activated, title/description/link/due date lock; the campaign cannot be re-activated or reverted to draft.
- Completion tracking is a per-recipient table of completed / not-completed / overdue, driven entirely by each recipient's own action-item completion, using the exact same overdue derivation used elsewhere in the system. The system never reads or verifies the external form — a recipient who marks their item complete without actually filling in the form shows as "completed" regardless.
- The tracking table is visible only to the campaign's creator/sender (and equivalent-access holders), and must work identically at large audiences (hundreds of recipients) without a separate pagination/performance story.
- Colleague-level viewers normally see nothing of a form campaign, with one sanctioned exception: a campaign's creator can see, for that campaign only, each recipient's name plus that campaign's own action-item status — nothing else, and the exception ends the instant the campaign closes.

## Technical Decisions

- Owned by the `campaigns` backend module (CAP-10). Data model: `FormCampaign` 1—N `CampaignRecipient` (each referencing an `Employee`), and `FormCampaign` 1—N `ActionItem` (generated rows tagged `source: campaign`).
- Audience building consumes the directory module's filter engine (custom-field-aware `FieldRegistry` query interface) directly rather than reimplementing filtering or saved-view resolution.
- Activation calls a dedicated bulk contract entry point distinct from single-item creation: `createCampaignActionItems({ campaignId, authorId, title, description?, dueDate, link?, assigneeIds }, tx?) → ActionItem[]`, an atomic one-per-recipient batch. The single-item `createActionItem` contract explicitly rejects `source: 'campaign'` — campaigns must use the bulk path.
- The campaign-sender Colleague exception is computed server-side by the access resolver as a scoped grant keyed by campaign ID (name + that campaign's action-item status only) — never a frontend "also show this if I sent it" branch — and is withdrawn the moment the campaign closes.
- Feature-gating ("create form campaigns") runs through the permission-checker axis, entirely separate from section-visibility access; the two are never conflated.
- Recommended sequencing to avoid a cross-track wait: assign the same developer to Epic 4 (action items) and Epic 10, since Story 10.3 calls directly into Epic 4's action-item creation path.
- Frontend: a `campaigns` page/route per the app's IA, using `react-hook-form` + `zod` for the campaign-creation form (first form surface introducing these libraries to the frontend).

## UX & Interaction Patterns

- Draft vs. activated are the only two renderable states: draft shows all fields editable with an "Activate" primary action; activated shows locked/read-only fields with "Activate" replaced by the live completion table. No in-between state exists.
- Audience building reuses the same shared filter/audience-builder component as the All Employees list and Risk Dashboard: build a filter, live-preview the resolved count, confirm; the campaign variant additionally supports individual add/remove after the filter resolves.
- The completion table reuses the shared Overdue Indicator and positive Status Badge components (same visual language as the Action Item Row), updates live with no manual refresh, and must read identically whether it has 10 or hundreds of rows.
- Empty state: "No campaigns yet." with a single primary "New campaign" action for permitted roles.
- Save/activation failure uses a destructive toast ("Couldn't save. Try again.") and never silently reverts; a failed activation specifically leaves the campaign as a fully-editable draft, never a half-activated one.

## Cross-Story Dependencies

- Sequential within the epic: 10.1 → 10.2 → 10.3 → 10.4.
- 10.2 depends on Epic 3's filter engine and saved views being real (not just the Wave-0 stub).
- 10.3 depends on Epic 4's bulk action-item creation path; same-developer assignment across Epic 4 and Epic 10 turns this into in-sequence work rather than a cross-developer wait.
- 10.4 reuses Epic 4's overdue derivation exactly — no separate overdue logic is implemented in this epic.
- Epic 11's named-colleague feedback-request flow (Story 11.3) builds directly on top of Epic 10 (campaign creation plus the individual-add audience path) and is orchestrated from the frontend rather than a backend call between the two modules.

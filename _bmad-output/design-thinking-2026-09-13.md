# Design Thinking Session: people management

**Date:** 2026-09-13
**Facilitator:** Unslopper
**Design Challenge:** Validate implemented UI mockups `ux-people-management-2026-09-13` (14 HTML prototypes) against DESIGN.md + EXPERIENCE.md + access-model; plan fix list before story-dev.

---

## 🎯 Design Challenge

Validate that the 14 static HTML prototypes in `_bmad-output/planning-artifacts/ux-designs/ux-people-management-2026-09-13/mockups/` correctly embody the spines (spines win on conflict), respect absent-not-hidden access rules, and are testable with real users (UM / DM / PP / HR Admin / Colleague) before implementation.

---

## 👥 EMPATHIZE: Understanding Users

### User Insights

- UM Olena needs to propose internal candidates without widening standing access (auto Shared Link scoped to S1/S4/S11/S12/S5-CV only, bound to request lifetime).
- DM Dmytro needs project-scoped triage (selector + Unassigned bucket), approve/reject-with-written-reason, explicit close only.
- PP Marta/Daniela needs severity-sorted risk triage (medium/high/leaver emphasised, trend only on actual change) and campaign audience freeze + per-person completion (never external-form contents).
- HR Admin Priya needs runtime extensibility as data (functional roles grant features, never data) with immediate revoke.
- Colleague needs whitelist-only directory (S1 + S10-dates-only + S11-name-only) that still feels complete, never hinting at gated content.

### Key Observations

- Prior spine reviews (`review-rubric.md`: 1 critical / 5 high; `review-accessibility.md`: 3 high) already applied per reconciles: Flows 5–6 added, button-primary → inherited primary, contrast measured ≥4.5:1, S7/S8 composer rows, row keyboard/SR + aria-disabled contracts.
- Mockups are visually calm (shadcn radix-nova, Ink sidebar #15112E, 56px rows) but encode several spine violations as code patterns (dismissed in default view, `<b>` not link rows, unconditional mentor header, `.gated` placeholder box).
- Accessibility floor is stated in spines but not demonstrated in mock markup (no aria-disabled, aria-live, focus-trap, describedby).

### Empathy Map Summary

- Says: "Show me only what I may act on." Thinks: "If I see a lock icon, the data exists." Does: drills directory → profile → action; filters, proposes, approves, freezes, ends with note. Feels: trust when absent sections read continuously; distrust when `—`, locks, or empty states hint at hidden data.

---

## 🎨 DEFINE: Frame the Problem

### Point of View Statement

Engineering managers, delivery leads, people partners and HR admins need an access-correct HR tool that never implies denied data exists, because trust in people-data handling is the product.

### How Might We Questions

- HMW make denied sections truly absent (no DOM/fetch) while keeping surrounding copy continuous?
- HMW keep leaver (prediction) visually distinct from dismissed (status) and from heat-ramp severity?
- HMW demo keyboard/SR, focus, live-region and reduced-motion contracts in static prototypes so story-dev copies the right pattern?
- HMW cover failure/edge states (zero results, write failure, rollback, conflict, expired link, stale sync, offline) as first-class mocks?

### Key Insights

- Spine-wins must be enforced in mocks: every `—`, placeholder, or unconditional field is a future code bug.
- Static mocks need one correct a11y-annotated reference per pattern (row link, aria-disabled, role=status/alert, describedby definitions).
- Missing mocks: Sheet quick-view, Dialog confirms, CmdK states, expired-link screen, campaign-closed state, stale/offline banners.

---

## 💡 IDEATE: Generate Solutions

### Selected Methods

- Brainstorming (divergent fix list, defer judgment) + Analogous Inspiration (Linear CmdK discipline, Notion inline writethrough, shadcn vocabulary).

### Generated Ideas

- 14 adversarial fixes + 8 edge-case state mocks; annotate audience per mock (viewed-as); mark mock-only notes `data-mock-note`; add `<md` stacked-list variant; add one a11y-annotated reference mock.

### Top Concepts

1. Access-correctness pass (dismissed filter, mentor gate, gated-box removal, dash cleanup).
2. Interaction-contract pass (row links, aria-disabled, live regions, focus trap, Sheet/Dialog/CmdK mocks).
3. Edge-state pass (empty, failure, rollback, conflict, expired, stale, offline, closed).

---

## 🛠️ PROTOTYPE: Make Ideas Tangible

### Prototype Approach

Static HTML in `mockups/` (14 files: 01-directory through 13-shared-link-view + design-system.html) — rough enough to test, concrete enough to trace to code.

### Prototype Description

Directory, employee profile, my-profile, UM/DM dashboards, resourcing list/detail, risk dashboard, campaigns list/detail, mentorship hub, admin, shared-link view, design-system gallery. Reviewed 2026-09-13 via bmad-review adversarial + edge-case-hunter lenses.

### Key Features to Test

- Directory drill (row link, colleague whitelist, saved-view re-resolve, export scoping).
- Profile assembly per audience (S7/S8 flags, comp-band guard, mentor D5 gate, timeline skip-annotation, no departure event).
- Resourcing propose→decide→close (auto Shared Link scope/lifetime, PeopleForce ID-only, headcount guards, written-reason gates).
- Campaign draft→freeze→track→close (atomic freeze, per-person completion only, sender-exception termination).
- Mentorship assign→end (flag-on-at-write gate, closure-note gate, conflict + auto-close paths).
- Shared-link re-clamp per view + expiry/revoke screen.

---

## ✅ TEST: Validate with Users

### Testing Plan

5–7 users (UM, DM, PP, HR Admin, Colleague). Tasks: (1) find high-risk person, open S6, explain trend; (2) propose internal candidate, explain what DM sees; (3) freeze campaign, describe what changed; (4) end pair without note, narrate refusal. Observe actions, not statements. Record: where they hesitate, what they assume exists, what surprises them.

### User Feedback

(Prototype review substituted 2026-09-13 — no live users yet. Findings below are lens-derived hypotheses to validate in live testing.)

### Key Learnings

Adversarial (14) + edge-case (8) findings — see Fix List below. Highest risk: dismissed-in-default, non-link rows, unconditional mentor, gated placeholder copy-paste, rejected-row mislabel, history contradiction, mentorship CSS typo, journal gate ambiguity, dash-for-empty, IDP reopen checkbox, missing responsive/a11y demos; unmocked empty/failure/rollback/conflict/expired/stale/offline/closed/Sheet/Dialog/CmdK states.

---

## 🚀 Next Steps

### Refinements Needed

Fix list (source: bmad-review 2026-09-13, mockups/):

1. 01: remove dismissed from default tab; add Status filter demo.
2. 01: name cell → real `<a>`; row never tabindex=0; SR announcement pattern.
3. All: wire sidebar from `#` to real mock files or remove.
4. 02: gate Mentor per D5; add Colleague variant with mentor absent; annotate viewed-as audience.
5. 02: mark `.gated` MOCK-ONLY (`data-mock-note`); production renders nothing.
6. 07: relabel `Reverse to rejected…` → `Re-open/Approve…` with reason gate.
7. 07: reconcile history pending vs table proposed to single state machine.
8. 11: fix `.sidebar a active` → `.sidebar a.active`.
9. 12: journal gate → Reporting-line + Project-line + PP + Full-access.
10. 10: `—` in Completed → `Not completed + due` text.
11. 03: trim address leading space; add inline validation demo.
12. 03: completed IDP → read-only receipt (no editable checkbox).
13. All: add `<md` Sheet collapse + stacked row-drill + 200%/320px note (01- variant).
14. All: annotate one reference mock with aria-disabled, role=status/alert, aria-live, describedby risk definitions, trend text equivalents, focus-trap/return-focus notes.

Edge-state mocks to add: zero-results empty; inline-edit failure toast+retry; risk stale banner; zero-headcount aria-disabled block + shrink rejection; campaign rollback toast + closed read-only state + export-ready toast; mentorship conflict toast + auto-close system-note row + FR-44 unflag rule; expired-link full screen; CmdK empty/loading/stale; Sheet quick-view + Open full profile; Dialog single-level confirms; offline banner.

### Action Items

- [ ] Apply 14 adversarial fixes in mockups (owner: design; verify: spine-wins grep for `—`/lock/restricted in mocks).
- [ ] Add missing state mocks (Sheet, Dialog, CmdK, expired, closed, stale, offline, empty, failure).
- [ ] Annotate viewed-as audience on every mock header.
- [ ] Run 5-user task test (UM/DM/PP/Admin/Colleague); capture what they do vs say.
- [ ] Re-run bmad-review on updated mockups before story-dev handoff.

### Success Metrics

- Zero `—`/lock/restricted placeholders for denied content in mocks; dismissed absent from default view.
- Every pointer action has keyboard/SR equivalent in markup notes; disabled uses aria-disabled + reason; toasts/banners carry role + live region.
- All failure/edge paths (empty, failure, rollback, conflict, expired, stale, offline, closed) have a mock.
- Live test: users complete 4 tasks without asking about hidden data; leaver never misread as dismissed/severity.

---

_Generated using BMAD Creative Intelligence Suite - Design Thinking Workflow + bmad-review (adversarial, edge-case-hunter) 2026-09-13_

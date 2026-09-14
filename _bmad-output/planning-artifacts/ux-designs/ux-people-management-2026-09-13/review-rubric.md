# Spine Pair Review — people management

## Overall verdict (2-3 sentences)

Spines are coherent, inheritance-disciplined, and shape-conformant, with all 14 cited mockups present and a credible IA-closure claim across FR-1–FR-65. The pair is downstream-usable for directory, resourcing, risk, campaign, and mentorship work, but MyProfile, departure-cascade, CDS/IDP, and the seven admin sub-surfaces ride on "stated admin task" coverage rather than flows, mocks, or DESIGN visual specs, and two risk tokens plus a handful of referenced tokens diverge from or are missing in the cited CSS source. Fix the token divergences and add one self-service/departure flow plus admin visual coverage before build.

## 1. Flow coverage — [thin]

IA (`EXPERIENCE.md:30-56`) claims closure over FR-1–FR-65 with "every surface landed by a Key Flow or a stated admin task". Six Key Flows (`EXPERIENCE.md:159-220`) cover resourcing propose/decide (F1/F2), campaign build/track (F3), mentorship end (F4), risk triage (F5), and functional-role creation (F6) well, each with climax + failure branch.

- **F1 — MyProfile self-service journey has no flow. [high]** Location: `EXPERIENCE.md:38` (MyProfile row) vs `EXPERIENCE.md:159-220` (no self-service flow); FR-16–FR-19 per closure line 56. Fix suggestion: add Flow 7 (employee edits contacts/address, uploads certificate, self-completes IDP item, reads flagged note) or explicitly scope MyProfile out of this spine phase.
- **F2 — Departure recording / cascade has no walkthrough, only failure-branch mentions. [high]** Location: `EXPERIENCE.md:53-54` (departure + journal rows), `EXPERIENCE.md:111` (departure read-only state), F1/F2/F4 failure branches. Fix suggestion: add a short admin flow (record departure → re-parenting gate → cascade preview → read-only profile) exercising the atomic-cascade and sole-holder gates.
- **F3 — SharedLinkView recipient journey unwalked. [medium]** Location: `EXPERIENCE.md:47` (SharedLinkView row), `EXPERIENCE.md:91` (auto-link generation), `mockups/13-shared-link-view.html`. Fix suggestion: extend F1 with 2 recipient steps (DM opens link, sees clamped sections, link expires/re-clamps) or a standalone recipient micro-flow.
- **F4 — CDS touchpoints (S12/IDP/assessments) and Feedback records (S8) surfaces have no flow. [medium]** Location: `EXPERIENCE.md:45-46` vs flows F1–F6 (only S6/S15/S13/S9 exercised). Fix suggestion: fold one CDS-IDP self-complete + one feedback-request-via-campaign path into F3/F4, or mark explicitly as admin-task-covered with pointers.
- **F5 — Six admin sub-surfaces share one "stated admin task" each, no flow except functional roles. [medium]** Location: `EXPERIENCE.md:48-54` (custom fields, departments, org relationships, full-access, departure, journal) vs F6 only. Fix suggestion: accept per own definition but add entry-point guards (who reaches each, from where) — currently only the table's "Reached from" column carries this.

## 2. Token completeness — [adequate]

DESIGN frontmatter (`DESIGN.md:15-128`) deltas success + five risk levels, type roles, radius, layout tokens, and component anatomy refs. Base-system inheritance is explicit (`DESIGN.md:143`).

- **T1 — `risk-medium` and `risk-leaver` proposed values diverge from cited source. [high]** Location: `DESIGN.md:158,160` ("currently identical to attention in index.css; proposed distinct orange", "proposed plum") + frontmatter `DESIGN.md:34-45` vs claim "verbatim from index.css" (`DESIGN.md:21`). Fix suggestion: land the index.css change first, or demote frontmatter values to `proposed-` keys so no consumer treats them as built.
- **T2 — `display-sm` used but never defined. [medium]** Location: `EXPERIENCE.md:102-103` ("`display-sm`: 'No employees match.'") and Component Patterns empty-state row; `DESIGN.md:46-69` defines `display` note, `page-title`, `section-title`, body/label/caption/mono — no `display-sm`. Drift example defines it (`design-example-shadcn.md:24-28`); this spine dropped it. Fix suggestion: add `display-sm` to frontmatter typography or point the two call sites at `section-title`.
- **T3 — Body references ~15 tokens with no frontmatter entry and no verified source line. [medium]** Location: `DESIGN.md:194-229` (`--background-surface/stripe/scrim/elevated/muted/selected`, `--text-secondary/disabled`, `--border-focus/card/subtle/default`, `--accent-warning`, `shadow-card/shadow-modal`, `chart-*`) claimed "inherited from index.css" but frontmatter lists none and no index.css line numbers are cited. Fix suggestion: either add an `inherited-verified:` source-note with index.css line refs, or move the load-bearing ones (surface, border-default, focus) into frontmatter as aliases.
- **T4 — Minor: `rounded.xl`, label-caps header tracking, and layout gutters are ASSUMPTION-only tokens. [low]** Location: `DESIGN.md:176,186,203`. Fix suggestion: keep, but tag `xl` consumers (currently only empty-state illustration) so a later cleanup can drop the token if unused.

## 3. Component coverage — [adequate]

DESIGN Components (`DESIGN.md:207-232`, 22 entries) vs EXPERIENCE Component Patterns (`EXPERIENCE.md:81-96`, 13 rows). Core loop (Table, RiskPill, FilterBuilder, Sheet quick-view, Dialog confirms, Timeline, ProjectSelector) is covered on both sides with anatomy↔behavior split respected.

- **C1 — Admin components have behavior rows but no DESIGN visual spec. [medium]** Location: `EXPERIENCE.md:48-54` (roles, custom fields, departments, org-relationships, full-access, departure, journal) vs `DESIGN.md:207-232` (no admin-table, permission-toggle, journal-row, cascade-preview entries; Select/Input cover only atoms). Fix suggestion: add one "Admin table + guarded-writer confirm" visual entry or explicitly state admin surfaces compose from Table/Dialog/Select with no new anatomy.
- **C2 — Export and share-link config picker are behavior-only. [low]** Location: `EXPERIENCE.md:94-95` vs DESIGN (FileLink covers rows, but no Export-button/toast or share-config-toggle anatomy). Fix suggestion: add two short DESIGN bullets (export toast variant; per-section toggle + never-share structural exclusion note already at `DESIGN.md:231` is enough — just cross-link it).
- **C3 — Campaign wizard / AudienceBuilder frozen-receipt visual lock is described once. [low]** Location: `DESIGN.md:225`, `EXPERIENCE.md:90`. Fix suggestion: no new spec needed; add frontmatter `components.audience-frozen` alias (lock icon + timestamp) so mocks bind to a token.

## 4. State coverage — [adequate]

State Patterns table (`EXPERIENCE.md:97-111`, 11 rows) covers cold load, empty directory/Unassigned, absent-no-access, stale sync, offline, export-ready, link expiry, campaign frozen, orphan/zero-headcount, departure read-only. Failure branches in F1–F6 add conflict/rollback/reject paths.

- **S1 — Conflict/partial-failure states live only in flow failure branches, not the State table. [medium]** Location: `EXPERIENCE.md:180` (zero-headcount block), `:190` (activation rollback), `:200` (concurrent double-close), `:210` (row re-sort) vs table `:99-111`. Fix suggestion: promote three rows (optimistic-conflict → toast+refresh; activation all-or-nothing → draft-kept; approval-reversal → slot-frees) into the State table with treatments.
- **S2 — Empty states inconsistent between spines. [low]** Location: `DESIGN.md:227` ("risk-dashboard empty reads 'no active risks'") has no EXPERIENCE State row; `EXPERIENCE.md:102-103` empty-directory copy has no DESIGN anatomy beyond generic Empty. Fix suggestion: add risk/campaign/mentorship empty rows to the State table reusing the same Empty anatomy.
- **S3 — Offline posture is an open ASSUMPTION. [low]** Location: `EXPERIENCE.md:106` ("queue-or-reject") + Open items `EXPERIENCE.md:228`. Fix suggestion: keep flagged; block only offline-write stories, not the spine.

## 5. Visual reference coverage — [adequate]

All 14 composition references cited at `EXPERIENCE.md:58` resolve: directory listing of `mockups/` confirms `design-system.html` + `01`–`13` present (14 files). Spine-wins clause stated in both spines (`DESIGN.md:139`, `EXPERIENCE.md:18,58`).

- **V1 — Seven admin sub-surfaces share one mock. [medium]** Location: `EXPERIENCE.md:48-54` vs single `mockups/12-admin.html`. Fix suggestion: either scope `12-admin.html` explicitly (which sub-surfaces it shows; which defer to Table/Dialog atoms) in the composition line, or split out `12b-org-relationships.html` (the only admin surface with complex guards: cycle/self-managed/orphaning/concurrent-conflict).
- **V2 — PM/PP dashboard lenses have no dedicated mock. [low]** Location: `EXPERIENCE.md:34-35` vs mocks `04-dashboard-um.html`, `05-dashboard-dm.html` only. Fix suggestion: acceptable under the stated one-engine-four-lenses rule (`EXPERIENCE.md:24`); add one sentence to the composition line saying PM/PP reuse `05` with scoped blocks, so reviewers stop filing it as a gap.
- **V3 — CDS/Feedback surfaces ride inside `02-employee-profile.html` with no section anchor note. [low]** Location: `EXPERIENCE.md:45-46` vs `mockups/02-employee-profile.html`. Fix suggestion: annotate the composition line with covered sections (S8/S12 in `02`), matching how S6/S15 are already traceable via flows.

## 6. Bloat — [strong]

No bloat found. At 245 + 228 lines for a 65-FR / 14-CAP surface the pair is lean; retained Risk Alternatives B/C (`DESIGN.md:164-165`) read as decision record, not decoration, and the WCAG ratio table (`DESIGN.md:162`) is measured evidence. No new finding; do not trim.

## 7. Inheritance discipline — [strong]

Posture is exemplary against the shadcn example shape (`design-example-shadcn.md:55-56,93-94`): delta-only frontmatter, `{path.to.token}` anatomy refs, shadcn-defaults contract, absent-not-hidden guardrail (`DESIGN.md:209,238-239`), and spines-win conflict rule. Two discipline notes, both already self-flagged as ASSUMPTIONs:

- **I1 — Proposed risk values sit under a "verbatim" header. [medium]** Location: `DESIGN.md:16-20` header comment vs `DESIGN.md:158,160`. Fix suggestion: rename the two rows' keys or add `status: proposed` until index.css lands (same fix as T1; counted here for discipline, not double-counted in severity rollup beyond T1).
- **I2 — EXPERIENCE stays behavioral; no visual duplication detected. [strong]** Voice table (`EXPERIENCE.md:64-75`), primitives (`EXPERIENCE.md:113-121`), and a11y floor (`EXPERIENCE.md:123-132`) correctly defer contrast/focus-ring visuals to DESIGN/shadcn. No fix.

## 8. Shape fit — [strong]

Both files track the shadcn example shapes closely: DESIGN has frontmatter + Brand & Style + Colors + Typography + Layout + Elevation + Shapes + Components + Do's/Don'ts (cf. `design-example-shadcn.md`); EXPERIENCE has Foundation + IA + Voice + Component/State Patterns + Primitives + A11y + Responsive + Inspiration/Anti-patterns + Key Flows with climax + failure (cf. `experience-example-shadcn.md`). Deviations are additive and justified: named layout tokens under `spacing` (verbatim from index.css), access-aware wording rule (`EXPERIENCE.md:75`), and Open items (`EXPERIENCE.md:223-228`) with pending PO confirms. Coaching decisions are logged in `.memlog.md` (6 decisions + finalize event). No fix.

## Mechanical notes

- Sources read: `DESIGN.md` (245 lines), `EXPERIENCE.md` (228 lines), `.memlog.md` (11 lines), `mockups/` listing (14 files, all cited refs resolve), `bmad-ux/assets/experience-example-shadcn.md` (133 lines), `bmad-ux/assets/design-example-shadcn.md` (109 lines).
- Mockup contents were not opened (listing only, per brief); V-findings assume filenames match contents — spot-check `12-admin.html` scope and `02` section anchors during build.
- Token verification stopped at the spine boundary: index.css line-level verification of T3's inherited tokens was not performed; T1/T2 are verifiable inside the spines alone.
- Severity rubric used: downstream build impact (wrong-token-ships = high; missing-flow/mock = medium-high; inconsistency a builder must guess at = medium; polish/flagged-assumption = low).

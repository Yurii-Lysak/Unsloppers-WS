# PRD Rubric Review — People Management Platform

**Run at:** 2026-08-26T21:38:00+03:00

## Overall verdict
This update lands all 22 edge-case-hunter findings as testable Consequences bullets tied to named decisions (D13–D20), cross-referenced consistently across the affected FR bodies, §9 Open Questions, and §10 Assumptions Index — FR numbering stays contiguous (FR-1–FR-65, no gaps across all 14 features), and the one deliberate divergence from the source validation report (saved views resolved as "ownerless" rather than "transfer to a manager/PP") is transparently flagged rather than silently substituted. Decision-readiness, strategic coherence, and scope honesty all hold up well under the added weight.

The risks introduced are narrower but real: a genuine unresolved gap in whether the departure cascade explicitly revokes a departing employee's *Full profile access* grant when they aren't the sole holder (the one guarantee that didn't get the same "explicit step, not implicit side-effect" treatment the shared-link finding received under D17); a newly-introduced dependency on undefined SPEC capability codes (`CAP-1`, `CAP-13`, `CAP-14`) that cuts against the PRD's own standalone-readability goal; and two small mechanical slips (a miscounted "eight" list in §10, one redundant NFR sentence in §5.13).

## Dimension verdicts
- Decision-readiness — strong
- Substance over theater — adequate
- Strategic coherence — strong
- Done-ness clarity — adequate
- Scope honesty — strong
- Downstream usability — adequate
- Shape fit — strong

## Findings

### high (1)
**FR-8 (§5.1) / FR-63 (§5.14) / Glossary "Departure (departure cascade)"** — Full profile access revocation on departure is unconfirmed for non-sole holders
The Glossary's "Departure" entry promises the cascade includes "...account deactivation, and full access revocation (FR-63)." FR-63's body itself only says every access grant "ends per CAP-1's standard revocation rule" — a generic pointer, in contrast to the explicit, named treatment the shared-link revocation got in the very same FR ("The shared-link revocation is its own named step, not left as an implicit side-effect of FR-6's continuous re-clamp rule" — D17). FR-8 and FR-62 only address the *sole remaining holder* case (blocking the departure until the grant is transferred first); neither states whether a departing holder who is *not* the sole one has their Full profile access grant explicitly revoked as a cascade step. Per §1's own "Three independent access dimensions" guardrail and the Glossary's definition of Full profile access ("a separate grant... never bundled with a functional role"), it's specifically not part of the derived-access machinery "CAP-1's standard revocation rule" most naturally describes — so it's unclear the generic phrase actually covers it. SM-7's departure-cascade success metric only tests "no orphaning of ... the sole full-access holder," not revocation of a non-sole holder's grant. This is the same class of gap (implicit vs. explicit cascade step) D17 was created to close for shared links, left open for the sibling guarantee.
Fix: Add an explicit FR-63 (or FR-8) consequence bullet stating whether/how a departing full-profile-access holder who isn't the sole holder has that grant revoked as its own named cascade step, mirroring the shared-link bullet's language — and add it to SM-7's validation criteria if so.

### medium (1)
**Glossary "ProjectAssignment" / "Timeline event" / FR-63 (§5.14)** — New CAP-N references are undefined for a standalone reader
This update newly introduces three unexplained SPEC capability-code references that weren't in the prior versions of these entries: ProjectAssignment now reads "Written only by timetracker sync (CAP-13)"; Timeline event now reads "it lives in employment status (CAP-14)"; FR-63 now reads "every access grant they held ends per CAP-1's standard revocation rule." §0 Document Purpose only says Features are "grouped by the SPEC's 14 capabilities" — it never states the Feature↔Capability mapping (e.g., that §5.13=CAP-13) or defines what a bare "CAP-N" means for a reader without `SPEC.md` open. The rubric's Downstream usability dimension calls for "Each section makes sense pulled out alone — cross-references via Glossary terms, not 'see above'"; a `CAP-N` pointer into a companion document is exactly that pattern, one level removed.
Fix: Either add a one-line explicit Feature→Capability mapping note in §0, or replace the new `CAP-N` pointers with the PRD's own Feature/FR references (e.g., "ends per this platform's standard access-role rules (§5.1)" instead of "per CAP-1's standard revocation rule").

### low (2)
**§10 Assumptions Index, consolidated edge-case-guards bullet** — "Eight" guards bullet enumerates only seven
The bullet reads: "§5.1 FR-1/FR-6/FR-7/FR-8, §5.4 FR-22, §5.6 FR-30/FR-31/FR-33, §5.9 FR-42/FR-43, §5.14 FR-62/FR-63 — eight edge-case safety guards (multi-role union, shared-link continuous re-clamp, organisational-relationship write safety, full-access commit-time check, cascade idempotency, resourcing lifecycle robustness, mentorship pair write-time safety) added 2026-08-26..." That parenthetical names seven guards and the location list omits §5.13 FR-58 — the D19 sync-clock-semantics guard. §9's equivalent sentence correctly lists all eight, including "(D19, FR-58)." No information is actually lost (FR-58/D19 is fully covered by the immediately preceding bullet on FR-58), but the count in this specific sentence doesn't match its own enumeration.
Fix: Add the sync-clock guard and "§5.13 FR-58" to this bullet's parenthetical/location list, or change "eight" to "seven" if FR-58 is meant to stay carved out into its own separate bullet.

**§5.13 Feature-specific NFRs** — NFR restates FR-58 with vaguer, boilerplate framing
"External integration failures degrade gracefully and never take the application down, within the explicit bounds stated for the timetracker (D3)." "Degrade gracefully" is close to the exact phrase the rubric's Done-ness clarity dimension flags as a red flag ("system handles X gracefully"). FR-58's own consequences already state the real, testable bounds (single-sync confirmation within 15 minutes, 4-cumulative-hour withdrawal, fail-soft display) — this NFR sentence adds no new testable content beyond a "(D3)" pointer, and "never take the application down" is an absolute claim with no stated verification method.
Fix: Either drop this NFR line as redundant with FR-58, or replace it with a genuinely new, testable constraint not already covered there (e.g., an explicit availability/error-budget target for the integration boundary itself).

## Mechanical notes
- FR numbering is contiguous and gap-free across all 14 features (FR-1–FR-65; §5 table sums to 65), and the feature count in §0/§7.1 (14) matches the §5 table.
- SM numbering is contiguous (SM-1–SM-9 plus SM-C1/C2); D-numbering in `decisions.md` is contiguous (D1–D20) with no gaps.
- Capitalization is consistent: "Full profile access" (the Glossary term) stays capitalized when named as the grant; "full-access holder(s)" (the descriptive noun phrase) stays lowercase throughout, matching `access-model.md`'s own usage.
- The saved-views "ownerless" resolution (FR-13) is a genuine divergence from the validation report's literal suggested fix ("transfer to a manager/PP") but is honestly flagged in both `decisions.md`'s appendix and the PRD's own §10 entry — not silently substituted.
- FR-42's "(identity-card data + flag only, never S13)" phrasing is inherited verbatim from `SPEC.md`'s CAP-9 intent line; read literally it looks self-contradictory (the open-to-mentor flag is itself part of S13's row in `access-model.md`), but `SPEC.md`'s own success criterion ("the pool never exposes S13 data") confirms the flag is meant as a deliberate narrow carve-out, not an oversight. Worth a wording pass upstream in the SPEC rather than a PRD-only fix.

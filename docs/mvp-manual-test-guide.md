# MVP Manual Test Guide

Companion to `docs/project-requirements-v2.md` (the graded spec) and `_bmad-output/planning-artifacts/epics.md` (the story-level acceptance criteria this guide walks through by hand). Written 2026-09-11 against `main` on both `Unsloppers-BE` and `Unsloppers-FE`, sprint status: Epics 1–12 functionally done, Epic 13 done for the required timetracker integration (PeopleForce is out of MVP scope).

Use this to run a manual pass before the demo. It is **not** a substitute for the automated access-control suite (Story 1.14/1.15) — it's what a human clicking through the app should verify, especially the things a screenshot can't catch (a section silently missing from an API response, a role seeing one row differently from another row in the same table).

---



## 0. Before anything else



### 0.1 Re-seed the production database

See the chat message for the exact steps. Do this **first** — several demo-visible features (HR Admin access, Unit Manager routing for Resourcing, the CDS skills matrix, the Mentorship Hub having any data at all) depend on seed steps added after the first seed run. Confirm the "Seed complete" log line reports non-zero `functionalRolesUpserted`, `departmentsUpserted`, and `skillsMatrixEntriesUpserted` — not just `identitiesUpserted`.

### 0.2 Known gaps — read before you go looking for these in the UI

Don't burn testing time hunting for screens that don't exist yet:

- **No UI to define a new custom field.** `POST /api/v1/custom-fields` (see Swagger at `{backend}/api/docs`) works; there is no "create field" screen in the frontend. Test custom fields via API for the *definition* step, then verify the *value* (edit/filter/visibility) through the UI as normal — that part is fully built (`CustomFieldsSection` on the profile, filters/columns on All Employees).
- **No UI to change organisational relationships** (reassign someone's manager, their people partner, or a department's manager — §2.1's "dedicated screen" requirement). The profile header shows manager/PP/mentor as read-only links only. These currently only change via direct API calls or by re-running the seed. Flag this to the team — it's a normative requirement with zero frontend surface right now.
- **12.6 (accessibility/responsive pass) — not started.** Don't expect a dedicated a11y pass; spot-check keyboard nav and mobile width informally (§4 below) but don't treat gaps there as new bugs, they're known scope.
- **All Employees list has no DB-level pagination yet** (backend defect #05 — full table scan in `field-registry.service.ts`). At the 24-account seed size this is invisible. Don't try to prove/disprove the 500-record 2s budget (NFR-2) manually — that needs the load-test harness (Story 3.7), not clicking.
- **D11 (default functional-role permissions) is technically still "pending PO sign-off"** in the decision log, even though the defaults are already implemented and working (see §6 below for what's actually assigned). Worth a 2-minute confirmation, not a build task.



### 0.3 Build a "cast list" before you start role-based testing

The 24 seeded accounts get their **department** deterministically (a hash of their email — see `seed.synthetic.ts`), and their **Unit Manager** is whoever has the longest synthetic tenure in that department (see `seed.departments.ts`). Reporting lines and project/PM/DM assignments come from the live TimeTracker sync (Epic 13), which depends on real data in the TT test environment — not something this guide can predict from the repo. Spend 5 minutes after logging in as HR Admin building this table (keep it — you'll reuse it for every section below):


| Role you need                                    | How to find one                                                                                                                                                                                                                                                           | Notes                                                                                                                    |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **HR Admin**                                     | `tt.site-admin@altexsoft.com`                                                                                                                                                                                                                                             | Confirmed by seed code — the only guaranteed account                                                                     |
| **Self / plain employee**                        | Any of the 24 — pick one with no reports                                                                                                                                                                                                                                  |                                                                                                                          |
| **Unit Manager (UM)**                            | Open `/employees/{id}/functional-roles` (as HR Admin) for a few candidates, or check each department's `managerId` via Swagger `GET /api/v1/employees?filters=...` grouped by department, or just look at the Resourcing → Requests routing after creating a test request | One is guaranteed per department seeded                                                                                  |
| **Delivery Manager (DM) / Project Manager (PM)** | Open a profile's **S11 Projects** section — it names the PM/DM per project                                                                                                                                                                                                | Depends on TimeTracker sync having run; if S11 is empty everywhere, the sync hasn't populated yet — check §7 below first |
| **People Partner (PP)**                          | Not seeded automatically (Story 1.3 is a write operation, no default PP assignment shipped in seed) — assign one yourself as HR Admin via the API, or via profile if a UI exists for it                                                                                   | If none exists, this is itself worth flagging                                                                            |
| **Colleague** (no relationship)                  | Any two employees in different departments/projects with no PP link                                                                                                                                                                                                       |                                                                                                                          |
| **Mentor / Mentee (seeded pair)**                | Mentor: `artem.shamraiev@altexsoft.com` · Mentee: `tt.site-admin@altexsoft.com`                                                                                                                                                                                           | Deterministic, from `seed.mentorship.ts`                                                                                 |


Login: any email above + the `BOOTCAMP_INITIAL_PASSWORD` value from Render's environment tab.

---



## 1. Smoke test (5 min)

- [x] `{frontend}/login` loads, login form accepts email + password
- [x] Logging in with a wrong password shows an error, doesn't crash
- [x] After login, lands on `/` (Dashboard) with the sidebar visible
- [x] Sidebar nav items present/absent match the logged-in account's permissions (e.g. a plain employee shouldn't see Risks, Campaigns, Resourcing, Mentorship, or the Admin section — only Home + Employees)
- [x] Refreshing the page keeps the session (no bounce to `/login`)
- [x] Logging out and back in works
- [x] Hitting an unknown path (`/whatever`) redirects to `/` instead of a blank/broken page

---



## 2. Access control & Employee Profile (Epic 1)

This is the section that matters most for grading (§3.2/§3.3 are NORMATIVE, "every cell in the matrix is strict"). Test with at least three viewers against the same profile: **Self**, **their UM/DM**, and a **Colleague**.

Route: `/employees/{id}`

- [ ] **As Self**: S2 (personal contacts), S3 (emergency contacts) are editable; S4 (employment) is read-only; **S6 (risks) is completely absent** — no section, no placeholder, not even an empty card
- [ ] **As Self**: S15 (request history) is absent even if you were proposed on a resourcing request
- [ ] **As the person's UM/DM**: same profile now shows S6 (risks), S7 (management notes, RW), S15 — sections that were invisible to Self
- [ ] **As a Colleague** (no Manager/PP relationship): only S1 (identity), S10 (leave **dates only**, no leave type), S11 (**project name only**, no PM/DM/period) are visible — open browser devtools Network tab and inspect the raw JSON response, not just the rendered page, to confirm the other sections aren't just hidden in the UI
- [ ] **Colleague view of S10**: confirm the leave *type* (vacation/sick/parental) is genuinely absent from the API response, not just unstyled
- [ ] Profile header shows manager/PP/mentor as links; as a Colleague, the **mentor field is withheld** even though S1 is otherwise visible to Colleagues (D5's exception)
- [ ] Photo upload works for Self only (`ProfileHeader`'s upload button); a manager/PP viewing someone else's profile has no upload control
- [ ] **Management notes (S7)**: create one as a UM with both visibility flags off — confirm the employee (Self) and their PM cannot see it; flip "visible for PM" on and confirm the PM sees it read-only while the employee still doesn't
- [ ] **Project-line narrowing**: if you can identify a PM/DM who is *not* also the person's UM (project access only, not reporting-line), confirm they see S6 but **not** S2/S3, and S5 limited to CV/certificates only — this is the one deliberately narrower tier in the whole matrix, worth explicitly checking



### 2.1 Functional roles admin (Story 1.4/1.5)

Route: `/admin/roles` (HR Admin only — confirm a non-HR-Admin account gets denied, not just hidden from nav)

- [ ] Create a new role (e.g. "Security Champion"), grant it only "create form campaigns"
- [ ] Assign it to a random employee via `/employees/{id}/functional-roles`
- [ ] Log in as that employee, confirm they can create a campaign (`/campaigns`) but still cannot see anyone's S6/S7/etc. outside their own Colleague view — a functional role must never widen data access
- [ ] Remove the permission from the role, confirm the now-demoted user is denied on their very next action (no logout needed)



### 2.2 Shared profile links (Story 1.11/1.12)

- [ ] As a UM, open a profile you have access to and generate a share link, enabling S1 + S9 only, leaving S2/S5/S6/S8 at default (off)
- [ ] Open the link as the named recipient (a different logged-in account) — confirm exactly S1 + S9 render, nothing else, and a direct API call for the link's token can't pull S3/S7/S13 even if you try to force it via the request
- [ ] Confirm S3, S7, S13 were never even offered as checkboxes in the share-link creation dialog
- [ ] Let the link's expiry pass (or revoke it manually) and confirm the recipient is denied afterward
- [ ] Revoke an active link as the manager who created it — recipient denied immediately, and the revocation shows up wherever the access journal is exposed to you (full-access holders / current manager+PP of the subject)

---



## 3. Self-Service (Epic 2)

Log in as any seeded employee and open your own profile (`/employees/{yourId}`, or check whether the sidebar's Home link routes you there).

- [ ] S4 (employment) — grade, position, seniority, employment type, English level, probation, contract type — all read-only
- [ ] S2/S3 — edit personal phone, address, emergency contact; confirm it saves and reloads
- [ ] S1 — upload a new photo; confirm it replaces the old one and shows up immediately
- [ ] S5 — upload a certificate; confirm uploading a non-certificate document type (or trying to edit CV/contract) is rejected
- [ ] S9 (career timeline), S11 (projects), S12 (CDS) render read-only
- [ ] S10 (leaves) shows a link out to the timetracker
- [ ] S13 — toggle your own "open to mentoring" flag
- [ ] S12 — mark your own IDP complete if one exists (the seeded mentee/CDS demo account is a good candidate) — confirm a completion date appears and you can't edit the deadline or conclusion text
- [ ] S8 (feedback) — confirm you only see records explicitly flagged "shared with employee"; S7 — confirm you only see notes flagged "visible for employee"
- [ ] S14 — mark one of your own action items complete; confirm you can't edit its title/due date or cancel it (only the author can cancel)
- [ ] Confirm **S6 is absent everywhere on your own view**, including any dashboard-style widget that might surface a summary

---



## 4. All Employees directory (Epic 3)

Route: `/employees`

- [ ] As a manager/PP: sortable, filterable table loads with real columns
- [ ] Filter by a **derived field** — "years with company" — confirm it computes correctly from join date (no stored field to fudge)
- [ ] Add/remove columns via the column picker; confirm a management-only custom field, if one exists, is offered as a column only to someone entitled to see it
- [ ] **Inline edit** a field you have write access to (e.g. grade for a direct report) from the list — confirm it updates on the full profile too
- [ ] Attempt inline edit on someone you have no access to (open devtools, try a direct write) — rejected server-side
- [ ] Save a filter/column combo as a named view; confirm it appears as a tab and survives a page reload
- [ ] Share a saved view with another manager; log in as them and confirm they see only rows/columns *they're* entitled to, not a frozen snapshot of what the creator saw
- [ ] Export the current view to `.xlsx` — open the file and confirm columns match what you're entitled to see, and that a field you can't see for a specific row isn't smuggled into the export
- [ ] **Colleague mode**: log in as a plain employee, open `/employees` — confirm only S1 + leave-dates + project-name columns/filters are offered, and this is enforced per-row (i.e. rows where you *do* have manager access to that one person could differ — check at least one such row if your cast list has one)

---



## 5. Action items (Epic 4)

- [ ] As a UM/PP, create a manual action item for someone in your access scope (title, due date, optional link)
- [ ] It appears on the assignee's S14 and on your dashboard's "own action items" widget
- [ ] As the assignee, mark it complete — completion date recorded; confirm nobody else (author, manager) could complete it on your behalf
- [ ] As the author, cancel an open item without a reason — rejected; with a reason — succeeds, status `cancelled`
- [ ] Backdate a due date (or find a seeded item already overdue) and confirm the "overdue" indicator shows consistently everywhere it renders (profile, self-service, dashboard, campaign completion table if source is `campaign`)
- [ ] Complete or cancel an overdue item — confirm the overdue flag disappears immediately

---



## 6. Risks & Risk Dashboard (Epic 5)

Route: `/risks` (only visible to Manager/PP-holding accounts)

- [ ] As a UM/DM/PP, record a risk (level, description, details, date) for someone you're responsible for
- [ ] Confirm the employee themself never sees it — check their own profile and confirm S6 is entirely absent, not just empty
- [ ] Record a second risk at a different level — confirm the trend arrow appears (up/down) and that a first-ever record shows no arrow
- [ ] Open the Risk Dashboard — counts by level (medium/high/leaver visually distinct), scoped to *your* people only
- [ ] Click a count to drill into the filtered table; click a row to open the profile
- [ ] As a plain employee, try navigating directly to `/risks` — denied, not just hidden from nav

Reminder while testing: `leaver` (risk prediction) and `dismissed` (employment status fact) are two different things — if you're testing both Risks and Employment status in the same session, don't cross-check them against each other as if they should match.

---



## 7. Resourcing (Epic 6)

Route: `/resourcing` (DM/PM/UM only)

- [ ] As a DM or PM, create a request with vacancy details, comp level, duration, workload, headcount — **leave the project field empty** and confirm it still saves as a valid "Unassigned" request
- [ ] As the routed UM (via the request's department field), confirm the request appears in your queue
- [ ] Propose an internal candidate — confirm this **auto-generates a shared link** to that candidate's profile naming the reviewing DM, scoped to S1/S4/S11/S12/S5(CV+certs only), with S6 off by default (toggle it on and confirm it then appears)
- [ ] Propose an external candidate using just a PeopleForce ID/link (no live integration expected — this is the accepted fallback per §4.7, not a placeholder)
- [ ] As the DM, review and reject a candidate **without** a reason — blocked; with a reason — succeeds, reason stored
- [ ] Approve a candidate — headcount's filled/remaining count updates; confirm the request does **not** auto-close when headcount fills — only an explicit DM close ends it
- [ ] Confirm the comp-level band is visible to the request author, routed UM, and reviewing DM — and check that a PP account (if you can get one) or an export/shared-link surface never shows it
- [ ] Check the candidate's profile S15 — the proposal shows up there for the manager line/PP, never for the candidate themself
- [ ] Confirm approving a candidate does **not** create a project record locally — S11 only updates after the next timetracker sync, this is expected behavior, not a bug

---



## 8. Career Timeline (Epic 7)

On any profile with edit access (S9):

- [ ] Trigger a tracked change (grade, position, department, FTE↔Subcontractor) via the appropriate section and confirm a timeline event appears automatically — no manual step
- [ ] As someone with "edit career timeline" permission, manually add/edit/delete a timeline event (for backfill)
- [ ] If you can force a system-generated event into the same window as a manual one, confirm the manual entry wins and the system write is marked "skipped" rather than silently overwriting it — otherwise just confirm this behavior exists conceptually if you can't easily reproduce it manually

---



## 9. CDS (Epic 8)

On a profile's S12 section (post-reseed, should have skills-matrix dictionary entries and a couple of demo assessments):

- [ ] Confirm a link to the current skills matrix for the person's department+position renders
- [ ] As someone with "maintain CDS records," add an assessment log entry with a conclusion
- [ ] Add/update an IDP with a deadline; as the employee, mark it complete and confirm the completion date appears (only Self can tick the checkbox — manager/PP can't tick it on someone else's behalf)
- [ ] On `/employees`, filter by "assessed before [date]" and confirm "never assessed" is a selectable option, not just missing from the dropdown
- [ ] Filter by "has an open IDP"

---



## 10. Mentorship Hub (Epic 9)

Route: `/mentorship`

- [ ] The seeded pair (mentor `artem.shamraiev@altexsoft.com` → mentee `tt.site-admin@altexsoft.com`) shows up in the active pairs list
- [ ] As the mentee or mentor, confirm the mentor/mentee field shows on the other party's profile header (visible to reporting/project line + PP, withheld from Colleagues per D5)
- [ ] Flag a third employee as "open to mentoring" (self-service S13); confirm they appear in the company-wide mentor pool for a manager/PP with the assign permission — **pool is company-wide**, cross-department
- [ ] Assign a new pair from the pool; confirm mentee selection is scoped to people the assigner holds access over, and the newly-paired mentor's status flips from "open to mentoring" to "mentor"
- [ ] End a pair **without** a closure note — blocked; with one — succeeds, and confirm the closure note is visible to reporting/project line + PP but **not** to the mentor or mentee themselves
- [ ] Confirm ending the pair writes an end event to the career timeline, and if the mentor has no other active mentees, their status reverts to "open to mentoring" (only if their self-flag is still on)

---



## 11. Forms & Campaigns (Epic 10)

Route: `/campaigns` (needs "create form campaigns" permission)

- [ ] Create a draft campaign (title, description, purpose, external form link, due date)
- [ ] Build an audience via the filter engine (or a saved view), preview the resolved list, add/remove a person manually
- [ ] Activate — confirm the audience freezes (someone added to the org afterward doesn't get pulled in) and exactly one action item per recipient is generated with the campaign's link/due date/sender
- [ ] As a recipient, follow the action item to the external link, mark it complete
- [ ] As the campaign creator, open the per-person completion table — confirm you see recipient names + this campaign's task status **only** (the documented Colleague-whitelist exception, §3.3.7) — and confirm this visibility disappears once you navigate away from this campaign (you shouldn't retain broader visibility into these people elsewhere)

---



## 12. Feedback (Epic 11)

On a profile's S8 section:

- [ ] As a UM/PP, record feedback with a visibility flag (default "management only")
- [ ] Flip it to "shared with employee," confirm the employee now sees it on their own profile
- [ ] Confirm chronological ordering and period filtering work
- [ ] **Requested feedback flow**: run a campaign targeted at named individuals (§4.15), then manually enter the responses received as feedback records — confirm nothing becomes a record automatically just from campaign completion

---



## 13. Dashboards (Epic 12)

Route: `/` (content varies by functional role)

- [ ] **UM dashboard**: headcount, active-risk counts by level, open/overdue action items, active resourcing requests, open campaigns; table of your people with risk/trend/project/leave status; your own action items sorted by due date
- [ ] **DM dashboard**: one table per project you're responsible for; project selector defaulting to "All projects"; selecting a project filters the whole page and recalculates counters for just that project; an explicit **Unassigned** bucket for requests with no project, included in the all-projects totals
- [ ] **PM dashboard**: same shape as DM, scoped to your own projects only
- [ ] **PP dashboard**: same building blocks, groupable by department/project, **no resourcing block anywhere on the page**
- [ ] Cross-check: numbers on each dashboard's counters should match what you'd get filtering the equivalent lists (Risk Dashboard, Resourcing, All Employees) manually — if a dashboard count and a drill-down disagree, that's a real bug

---



## 14. Timetracker integration (Epic 13)

- [ ] Confirm S10 (leaves) on a few profiles shows real data sourced from the timetracker test environment, not placeholder text
- [ ] Confirm S11 (projects) and PM/DM on those projects reflect the live sync
- [ ] If you can simulate or wait out a sync gap: S10/S11 should show a visible "temporarily unavailable" banner rather than crashing or silently showing stale data unlabeled
- [ ] You will not be able to manually verify the 15-minute confirmation / 4-hour withdrawal windows in a single test session — treat this as covered by the story's automated tests (13.2) unless you have a specific reason to distrust them

---



## 15. Wrap-up

- [ ] Re-run through §2 (access control) once more at the very end with fresh eyes — it's the highest-value section for grading and the easiest to develop blind spots on after testing everything else
- [ ] Note every finding with: route, account used, expected vs. actual, and whether it's a new bug or one of the known gaps in §0.2
- [ ] File anything new the same way the existing ones are tracked — `services/backend/defects/bugs/` (or the frontend's equivalent if one exists) plus the ClickUp board, per the existing convention in `services/backend/defects/README.md`
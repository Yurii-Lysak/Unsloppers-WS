# Track B — Execution Tracker

> Quick-access state board for **Track B** extracted from `docs/PRD_parallel_delivery_plan.md`.
> Track B owns: **Epic 3** (Wave 1) → **Epic 6 → Epic 8 → Epic 2** (Wave 2), plus a share of Wave 3.
> Update the **Status** column and the **Current State** block as work lands.

---

## Current state (edit me)

- **Wave:** Wave 1 — FieldRegistry + filterable list landed; next is inline editing
- **Done:** Epic 1 complete (Track A); `3-2` (real FieldRegistry), `3-1` (sortable, filterable employee list)
- **In progress (Track B):** `3-3` (inline editing on the list)
- **Next up (Track B):** `3-3` → then `3-4` saved views
- **Blocked right now:** nothing — C1/C2 contracts, real FieldRegistry, and list query are on `main`
- **Track B is currently blocking:** nothing until Epic 3 saved views (`3-4`) is needed by Track A Epic 10 in Wave 2


Status legend: `TODO` · `WIP` · `DONE` · `BLOCKED`

---

## The one thing that gates Track B's start

Track B does **not** wait on any other developer to *finish*. It builds against frozen contracts + stubs from day one:

- **AccessResolver (C1)** — real owner is Track A / Epic 1. Track B uses the **Wave-0 stub** (fixed permissive role over seed data) and the swap to the real implementation mid-Wave-1 is **transparent** (no Track B code change).
- **FieldRegistry (C2)** — this is **Track B's own** deliverable (Story 3.2). Everything else in Track B and downstream (exports, saved views, dashboards, campaign audiences) filters through it.

So the only hard Wave-0 gate before Track B can start Wave 1 is: contracts frozen, decisions ratified, seed-data tool exists (NFR.3), repo stood up (NFR.5).

---

## Wave 1 — Epic 3: All Employees Directory & Custom Fields

Read top-to-bottom = valid build order.

| Status | Story | What | Depends on | Blocks | Stub available |
|---|---|---|---|---|---|
| DONE | **3.2** | Define Custom Fields at Runtime — **real FieldRegistry (C2)** | none (AccessResolver **stub** only) | `3.1` + all downstream field-visibility enforcement | — (this *is* the real impl) |
| DONE | **3.1** | Sortable, Filterable Employee List | `3.2`, AccessResolver (**real**) | Epic 10 audience builder (esp. `10.2`); own `3.3/3.4/3.5/3.6`; NFR.2 | — |
| WIP | **3.3** | Inline Editing on the List | `3.1` | — | — |
| TODO | **3.4** | Saved and Shared Views | `3.1` | Epic 10 Story `10.2` (audience-from-saved-view) | — |
| TODO | **3.5** | Export to Excel | `3.1` | — | — |
| TODO | **3.6** | Colleague Mode of the List | `3.1`, `1.8` (whitelist rule — coordinate w/ Track A, **same rule, independent enforcement, no blocking**) | — | — |

**Wave-1 exit for Track B:** real FieldRegistry + a working, filterable/sortable Directory with inline edit, saved views, export, and colleague mode.

---

## Wave 2 — Epic 6 → Epic 8 → Epic 2

### Epic 6 — Resourcing

| Status | Story | What | Depends on | Blocks / Notes |
|---|---|---|---|---|
| TODO | **6.1** | Create a Resourcing Request | AccessResolver (**real**, from Wave 1) | — |
| TODO | **6.2** | Fulfil a Request (internal / external candidates) | `6.1` | External path uses **spec-sanctioned PeopleForce-link fallback** — never blocks on Epic 13.3 |
| TODO | **6.3** | DM Reviews & Approves/Rejects Candidates | `6.2`, **Epic 1 `1.11`/`1.12`** (profile sharing, real by end of Wave 1) | — |
| TODO | **6.4** | Request History | `6.3` | Does **not** wait on Epic 13.2's real sync — S11 update after approval is out of scope here |

### Epic 8 — Career Development (CDS)

| Status | Story | What | Depends on | Blocks / Notes |
|---|---|---|---|---|
| TODO | **8.1** | Skills Matrix Link & Assessment Log | AccessResolver | — |
| TODO | **8.2** | IDP Records (manager/PP + employee self-complete, per **D7**) | `8.1` | Feeds Epic 2 `2.4` |
| TODO | **8.3** | Manager/PP Maintain Assessments & Conclusions | `8.1` | — |
| TODO | **8.4** | Filter by Assessment Recency & Open IDP | `8.2`, `8.3`, **Epic 3 filter engine** (own, real by now) | — |

### Epic 2 — Self-Service (picked up last: consumes the most other sections)

| Status | Story | What | Depends on | Blocks / Notes |
|---|---|---|---|---|
| TODO | **2.1** | View Own Employment Summary | AccessResolver, **S4 data model (Epic 1 / Track A)** | — |
| TODO | **2.2** | Edit Own Personal & Emergency Contacts | AccessResolver | — |
| TODO | **2.3** | Upload Photo & Certificates | AccessResolver | File-constraint PO question open (Appendix) |
| TODO | **2.4** | View Own Timeline, Leaves, Projects, CDS & Mentorship | **Epic 7 (Track C, real by end Wave 1)**, **Epic 8 `8.2` (own)**, **Epic 9 mentorship flag (Track C)** | Pick up last for exactly this reason |
| TODO | **2.5** | View Shared Feedback, Flagged Notes & Own Action Items | **Epic 1 `1.9` (Track A)**, **Epic 11 `11.1` (Track A, Wave 2)**, **Epic 4 `4.2` (Track A, Wave 2)** | — |

---

## Wave 3 — Track B's likely share (developers reassign freely)

| Status | Item | What | Depends on |
|---|---|---|---|
| TODO | **NFR.2** | All Employees List Performance at 500+ Records | `3.1` (real, Track B), `NEW A` real caching (Track A) |
| TODO | **NFR.4** | Accessibility & Responsive Layout Pass (incl. the List) | Epic 1 profile, **Epic 3 list (Track B)**, Epic 12 dashboards all existing |
| TODO | **12.x** | Dashboards (whichever slice is assigned) | `12.1` shared engine, source epics real |

---

## What BLOCKS Track B (inbound cross-track dependencies)

Ordered by when Track B hits them. None require *waiting* on unfinished work — each is a Wave-0 contract/stub or already-real-by-then output.

| When | Track B story | Needs (from) | How it's resolved (no wait) |
|---|---|---|---|
| Wave 1 start | `3.1` | **AccessResolver** (Track A / Epic 1) | Wave-0 **contract + stub**; real swap mid-wave is transparent |
| Wave 1 | `3.6` | `1.8` colleague-whitelist rule (Track A) | Same rule, **independent enforcement point** — coordinate, no block |
| Wave 2 | `6.3` | Epic 1 `1.11`/`1.12` **profile sharing** (Track A) | **Real by end of Wave 1**, before Wave 2 starts |
| Wave 2 | `2.1` | **S4 data model** (Epic 1 / Track A) | Real by end of Wave 1 |
| Wave 2 | `2.4` | Epic 7 timeline (Track C), Epic 9 mentorship (Track C), Epic 8 `8.2` (own) | All real by the time Epic 2 is picked up **last** in the track |
| Wave 2 | `2.5` | Epic 1 `1.9` (Track A), Epic 11 `11.1` (Track A), Epic 4 `4.2` (Track A) | Track A delivers these earlier in its own Wave 2 sequence |
| Wave 3 | `NFR.2` | `NEW A` real caching (Track A) | Track A Wave-1 deliverable, real by Wave 3 |

**Never blocks Track B:** Epic 13.3 PeopleForce (Epic 6 ships on the fallback link) and Epic 13.2 real timetracker sync (6.4 explicitly excludes the S11-after-approval update).

---

## What Track B BLOCKS (outbound — others wait on Track B's output)

| Track B output | Blocks (whose) | Detail |
|---|---|---|
| `3.2` FieldRegistry (C2) | all downstream field-visibility enforcement (all tracks) | Built first in Wave 1, so it's real before anyone needs it |
| `3.1` filter engine | **Epic 10 audience builder** (Track A, Wave 2) | Real by end of Wave 1 → no wait |
| `3.4` Saved & Shared Views | **Epic 10 `10.2`** audience-from-saved-view (Track A, Wave 2) | Real by end of Wave 1 → no wait |
| `3.1` (real) filter engine | own `8.4` (assessment-recency filter) | Same track, later |
| `3.1` (real) | **NFR.2** performance pass (Wave 3) | Same track / reassignable |

Everything Track B blocks is **real by the end of its wave**, so no other developer ever sits idle on Track B.

---

## Fast-scan summary

- **Start gate:** Wave 0 done (contracts frozen, decisions ratified, seed data + repo). Then go.
- **Built first:** `3.2` (FieldRegistry) + `3.1` (filterable list) — **DONE**; unblocks inline edit, saved views, export, colleague mode. Next: `3-3`.
- **Only real inbound waits are avoided by design:** AccessResolver (stub), profile sharing / S4 (real by Wave 1 end), Track C timeline/mentorship (Epic 2 done last).
- **Track B's critical outbound:** the **filter engine (`3.1`/`3.4`)** is what Track A's campaigns (Epic 10) build on — keep it green.

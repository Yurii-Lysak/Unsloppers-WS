# Access Model — Roles & Section Matrix

Companion to `SPEC.md` (CAP-1) — see there for capability context. Normative per `project-requirements.md` §2–3. Teams do not get to redesign this — it's the single most load-bearing part of the whole project.

## Access roles — derived from hierarchy, not assigned

| Access role | How it arises |
| --- | --- |
| **Employee (Self)** | Everyone. Grants access to one's own profile and the colleague view of everyone else. |
| **Manager** | Arises from a relationship. If A manages B, A holds Manager access *with respect to B*. |
| **People Partner (PP)** | Arises from assignment. If A is B's assigned people partner, A holds PP access *with respect to B*. |

**Hierarchy resolution.** Manager access is the transitive closure of two relations, unioned:

1. **Reports to** — B reports to A, at arbitrary depth.
2. **Is assigned to a project managed by** — B works on a project whose PM or DM is A.

Consequences (all must hold):

- A manager two or more levels up sees everything about every person nested beneath them, without an explicit grant.
- A DM sees full information about every person on their projects, at the same level as that person's own unit manager. Project assignment grants managerial access.
- A PM sees the people on their projects; the DM sits above the PM in the same chain and therefore sees everything the PM sees, plus the rest of their projects.
- Access is evaluated relationship-by-relationship, per request: the same person can be Manager for one profile, PP for a second, Colleague for a third, within a single session.
- When a project assignment ends, the derived access ends with it immediately — managerial access is not sticky.
- Exactly one documented exception to "Manager sees everything": management notes visible to a PM (S7) — see Rules below.

## Functional roles — assigned, extensible, never widen data access

| Functional role | Features |
| --- | --- |
| **Unit Manager (UM)** | Unit dashboard by people. Resourcing: fulfils requests, proposes candidates. Risks, action items, mentorship assignment, CDS for their people. |
| **Delivery Manager (DM)** | Delivery dashboard by project. Resourcing: creates requests, reviews/approves/rejects candidates, sees PM-created requests on their projects. Risks, action items, CDS for their people. |
| **Project Manager (PM)** | Same as DM, scoped to own projects. Resourcing: creates requests for own projects. Risks, action items, CDS for their people. |
| **People Partner (PP)** | People partner dashboard. All HR functionality: profiles, timeline maintenance, CDS, feedback, campaigns, risks. **No resourcing.** |
| **HR Admin** | Everything PP has, plus custom field definitions, system dictionaries, and management of functional roles/permissions. |

Rules:

- Functional roles never grant data access on their own — a DM sees their project's people because of the hierarchy rule above, not because "DM" is a permission.
- The starting set above is not final. Functional roles and their permissions are **data, not code** — HR Admin creates, names, and grants a new role a set of feature permissions through the UI, no deploy, no schema change.
- Feature permissions are independently grantable, at minimum: create form campaigns, create action items, create/edit risks, create resourcing requests, fulfil resourcing requests, assign mentors, maintain CDS records, manage custom fields, view a given dashboard.
- A new functional role never widens what data its holders can see about a person — it only unlocks features. Where a feature needs data, it operates within the holder's existing (Employee/Manager/PP) access role; a role with no Manager/PP relationship to someone sees them only through the Colleague view.
- Removing a permission from a role takes effect immediately for everyone holding it.

## Section access matrix

Audiences: **Self** · **Manager line** (anyone with Manager access per above) · **PP** (assigned PP + HR line above them) · **Colleague** (authenticated employee holding none of the above roles w.r.t. this profile) · **Shared link** (see CAP-1 profile sharing) · **HR Admin** (full access to everything, not shown as its own column below).

Legend: `RW` read+write · `R` read only · `—` no access, section not rendered at all · `cfg` off by default, can be enabled per shared link.

| # | Section | Contents | Self | Manager line | PP | Colleague | Shared link |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S1 | Identity card | Name, photo, position, dept/unit, country/city, work email/phone, birthday (day/month), start date, manager, PP, mentor, current project(s) | R (photo RW) | RW | RW | R | on by default |
| S2 | Personal contacts | Personal phone/email, messengers, residential address, place of stay | RW | R | RW | — | cfg |
| S3 | Emergency contacts | Contact person, relationship, phone | RW | R | RW | — | — |
| S4 | Employment | Type (FTE/Subcontractor), grade, seniority, position history, English level, probation, employment status, contract type | R | RW | RW | — | cfg |
| S5 | Documents | Contract, W8, cooperation form, Diia City, CV, joining feedback, certificates | R (own) + upload certs | R | RW | — | cfg |
| S6 | Risks | Level, trend, description, details, date, full history | — | RW | RW | — | cfg |
| S7 | Management notes | Free-form notes, visibility-flagged | R (only *visible for employee*) | RW; **PM exception: R, only *visible for PM*** | RW | — | — |
| S8 | Feedbacks | See CAP-11 | R (only *shared with employee*) | RW | RW | — | cfg |
| S9 | Career timeline | System-generated event log, see CAP-7 | R | RW | RW | — | cfg |
| S10 | Leaves and absences | Vacation/sick/parental/extended — dates and types | R | R | R | R (incl. type) | cfg |
| S11 | Projects | Project, PM, DM, period | R | R | R | R (project name only) | cfg |
| S12 | CDS | See CAP-8 | R (+ complete own IDP) | RW | RW | — | cfg |
| S13 | Mentorship | Open-to-mentor flag, mentor, mentees, ended pairs, mandatory closing feedback on pair end (D6) | RW (own flag), R (pairs, incl. closing feedback about their own pair) | RW | RW | — | — |
| S14 | Action items and tasks | See CAP-4 | R (own) + mark complete | RW | RW | — | — |
| S15 | Request history | Resourcing proposals for this person | — | R | R | — | cfg (DM sees own natively) |
| S16 | Custom fields | See CAP-2 | per field visibility | RW | RW | per field visibility | cfg |

## Rules that follow from the matrix

1. **Every cell is strict.** A `—` section must not reach that audience through any surface — UI, API, export, search result, notification, or error message. Same for flag-gated records. A leak is a critical defect regardless of which section it happens in.
2. **S7 defaults to invisible to the employee and to PMs.** Every note carries two independent off-by-default flags: *visible for employee* and *visible for PM* (read-only for PM). UM, DM, and PP create/read/edit notes about their people regardless of flags — this is the one documented exception to "Manager sees everything."
3. **Colleague view is a whitelist, not a blacklist.** Exactly S1, S10 (incl. leave type), and S11 project name. Everything else is absent — enforced at the API, not by hiding fields in the frontend.
4. **Access is evaluated server-side, per section, on every request**, after resolving the requester's access role.
5. **Custom fields carry their own visibility** (management default / employee / colleague); filters and list columns respect it — a filtering user must never infer a value they can't see.
6. **The profile header's mentor field (part of S1) follows the stricter 4.11 rule, not the general S1 Colleague-R grant** — visible only to Manager line and PP. This is the same shape as the S7 exception above: a specific, later rule overriding a general one for one field. *(Decision D5, `decisions.md`)*
7. **A Shared Link audience is always read-only**, regardless of which `cfg` sections are enabled for it — no shared-link surface ever returns `RW` (spec §4.8: "a shared link never grants write access").

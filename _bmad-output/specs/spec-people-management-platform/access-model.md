# Access Model — Roles & Section Matrix

Companion to `SPEC.md` (CAP-1) — see there for capability context. Normative per `project-requirements-v2.md` §2–3. Teams do not get to redesign this — it's the single most load-bearing part of the whole project.

## Access roles — derived from hierarchy, not assigned

| Access role | How it arises |
| --- | --- |
| **Employee (Self)** | Everyone. Grants access to one's own profile and the colleague view of everyone else. |
| **Manager** | Arises from a relationship. If A manages B, A holds Manager access *with respect to B*. Comes in two tiers — see Hierarchy resolution. |
| **People Partner (PP)** | Arises from assignment. If A is B's assigned people partner, A holds PP access *with respect to B*. |

Full access to every section of every profile (below) is **neither** of these — it's a separate grant with its own mechanism, never bundled with a functional role.

## Hierarchy resolution — three relations, two tiers

Manager access is the transitive closure of three relations, unioned:

1. **Reports to** — B reports to A, at arbitrary depth.
2. **Manages the department** — B belongs to a department managed by A, including nested departments.
3. **Manages the project** — B works on a project whose PM or DM is A.

Relations 1 and 2 form the **reporting line** and grant the full Manager section-set. Relation 3 is the **project line** and grants a **narrower** section-set — see the matrix and Rule 2. All three are transitive: anyone above a manager through the same kind of path inherits what that manager sees.

**People Partner resolution — the "HR line".** PP access extends upward through **the people partner's own reporting chain inside HR**, recursively, without limit — the *reports-to* relation restricted to the HR department. It does **not** walk up the employee's own chain, which climbs the delivery hierarchy and ends outside HR.

Consequences (all must hold):

- A manager two or more levels up sees everything about every person nested beneath them, without an explicit grant.
- A manager of a department sees everyone in it and in its sub-departments; reports-to survives alongside this, because individual exceptions to the department tree are real.
- A DM sees the people on their projects; a PM sees the people on theirs; the DM sits above the PM in the same chain.
- Access is evaluated relationship-by-relationship, per request: the same person can be Manager for one profile, PP for a second, Colleague for a third, within a single session.
- When a project assignment, a department membership, or a reporting line ends, the derived access ends with it immediately — managerial access is not sticky.

**Timing of revocation.** Two different guarantees, deliberately:

- **Relations the platform owns** — manager, people partner, department membership, department management — take effect on the **next request**. No grace period, no re-login, no cache that outlives the change.
- **Project assignment**, arriving from the timetracker, takes effect **within 15 minutes**. This is an access-control window, not a display-freshness window (see `SPEC.md` CAP-13).

## Access switches — a distinct class of write

Four fields look like reference data but are access switches: change one and access changes immediately for everyone nested beneath.

- The employee's **manager**, the employee's **people partner**, the employee's **department**, and a **department's manager**.
- None of them is covered by the write grant on the section that displays them (S1). They are displayed on the profile and edited elsewhere.
- Changing any of them requires the dedicated ***change organisational relationships*** permission, on a dedicated screen.
- **No self-assignment**, ever, regardless of what permissions the actor holds: nobody may write themselves into the manager or PP field of another person, or make themselves the manager of a department they aren't already entitled to manage.
- **No self-managed department.** A department's manager may never be a current member of that department or any of its nested sub-departments — writing such a value is rejected, closing off self-managed Reporting-line access as a backdoor. *(D15, `decisions.md`)*
- **No reporting cycles.** A manager change is rejected if the proposed new manager is already a descendant of the subject in the reporting chain — the transitive-closure walk that resolves Manager access must never be allowed to loop. *(D15)*
- **No silent concurrent overwrite.** A concurrent conflicting write to the same access-switch field is rejected with a conflict error (optimistic concurrency) rather than silently applied — the losing write never reaches the journal as if it had landed. *(D15)*
- **No orphaning clear.** Clearing a department's manager through this direct edit path is blocked unless a replacement manager is designated in the same change — mirroring the re-parenting gate CAP-14 applies on departure. *(D15)*
- Every change is journaled — see Relationship and grant journal below.

## Full profile access

The audience that sees every section of every profile exists, but it is **not** a functional role and is not bundled with one (in particular, **HR Admin does not imply it** — see Functional roles below).

- Granted by its own mechanism, separately from functional roles, and only by somebody who already holds it.
- **No self-assignment.**
- The first holder is seeded at deployment.
- Removing the last holder is blocked — the system must never reach a state where nobody holds it. The holder count is re-checked **at commit time**, not from an earlier read, so two holders revoking each other at the same instant can't race the count to zero. *(D16, `decisions.md`)*
- Recording a departure for the sole remaining holder is blocked by this same gate — the grant must be transferred to another holder first. *(D16)*
- Every grant and revocation is journaled.

## Functional roles — assigned, extensible, never widen data access

| Functional role | Features |
| --- | --- |
| **Unit Manager (UM)** | Department dashboard by people. Resourcing: fulfils requests routed to their department, proposes candidates. Risks, action items, mentorship assignment, CDS for their people. |
| **Delivery Manager (DM)** | Delivery dashboard by project. Resourcing: creates requests, reviews/approves/rejects candidates, closes requests, sees PM-created requests on their projects. Risks, action items, CDS for their people. |
| **Project Manager (PM)** | Same as DM, scoped to own projects. Resourcing: creates requests for own projects. Risks, action items, CDS for their people. |
| **People Partner (PP)** | People partner dashboard. HR functionality: profiles, timeline maintenance, CDS, feedback, campaigns, risks, recording departures. **No resourcing.** |
| **HR Admin** | **Administration of the platform itself**: custom field definitions, system dictionaries, departments, functional roles/permissions. **Grants no data access on its own** — see below. |

*"Unit" survives only inside the role name Unit Manager, which means the manager of a department. There is no separate unit entity.*

Rules:

- **HR Admin grants no data access.** It's a feature-administration role: somebody who needs to create custom profile fields gets exactly that, not thereby the ability to read 500 people's personal contacts and documents. Reading data is governed by the access roles above and by Full profile access, never by holding a functional role. A DM sees the people on their projects because of the hierarchy rule, not because they're a DM. *(This corrects v1's framing, where HR Admin implicitly inherited "everything a PP has" — that is no longer true; only Full profile access sees everything.)*
- The starting set above is not final. Functional roles and their permissions are **data, not code** — HR Admin creates, names, and grants a new role a set of feature permissions through the UI, no deploy, no schema change.
- **Both dimensions must permit an operation.** The section matrix below says which sections an audience may read/write; this role×permission list says which functional roles hold which feature permissions. A write happens only where both allow it — an audience's section-level `RW` does not itself grant every permission-gated action inside that section.
- Feature permissions are independently grantable, at minimum: create form campaigns, create action items, create/edit risks, create resourcing requests, fulfil resourcing requests, **approve or reject proposed candidates**, **close resourcing requests**, **assign and end mentorships**, maintain CDS records, **edit the career timeline**, **create feedback**, **record a departure**, manage custom fields, **manage departments**, **change organisational relationships**, view a given dashboard. *(Bold = new in this update; previously several of these were hardcoded to UM/DM/PP/manager+PP rather than permission-gated.)*
- A new functional role never widens what data its holders can see about a person — it only unlocks features. Where a feature needs data, it operates within the holder's existing (Employee/Manager/PP/Full-access) access role; a role with no Manager/PP relationship to someone sees them only through the Colleague view, subject to the single campaign-sender exception (Rule 7 below).
- Removing a permission from a role takes effect immediately for everyone holding it.

**Default assignments (open question).** Which starting role holds which permission on first launch still needs explicit PO confirmation for five points not settled by the source: who may manage custom fields, who may assign and end mentorships, and the launch defaults for approve/reject candidates, edit the career timeline, and create feedback. Current default pending sign-off: seed the pre-update hardcoded assignments (HR Admin + managers for custom fields; UM + managers/PP for mentorship-assign; DM for candidate approve/reject; UM/PP for timeline edit; managers/PP for feedback), then HR Admin adjusts via the roles UI. See `decisions.md`.

## Section access matrix

Audiences: **Self** · **Reporting line** (Manager access via *reports to* or *department management*, and everyone above through those paths) · **Project line** (the PM/DM of the person's projects, and everyone above **through the project path only** — narrower, see Rule 2) · **PP** (assigned PP + HR line above them) · **Colleague** (authenticated employee holding none of the above w.r.t. this profile) · **Shared link** (an authenticated, explicitly-named recipient — never anonymous, see CAP-1 profile sharing) · **Full access** (sees everything, not shown as its own column — see above; replaces v1's "HR Admin" column, which is now wrong).

Legend: `RW` read+write · `R` read only · `—` no access, section not rendered at all · `cfg` off by default, can be enabled per shared link · `never` cannot be shared under any configuration.

| # | Section | Contents | Self | Reporting line | Project line | PP | Colleague | Shared link |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S1 | Identity card | Name, photo, position, department, country/city, work email/phone, birthday (day/month), start date, manager, PP, mentor, current project(s) | R (photo RW) | RW¹ | RW¹ | RW¹ | R | on by default |
| S2 | Personal contacts | Personal phone/email, messengers, residential address, place of stay | RW | R | **—** | RW | — | cfg |
| S3 | Emergency contacts | Contact person, relationship, phone | RW | R | **—** | RW | — | **never** |
| S4 | Employment | Type (FTE/Subcontractor), grade, seniority, position history, English level, probation, employment status (see CAP-14), contract type | R | RW | RW | RW | — | cfg |
| S5 | Documents | Contract, W8, cooperation form, Diia City, CV, certificates | R (own) + upload certs | R | **R — CV and certificates only** | RW | — | cfg |
| S6 | Risks | Level, trend, description, details, date, full history | — | RW | RW | RW | — | cfg |
| S7 | Management notes | Free-form notes, visibility-flagged | R (only *visible for employee*) | RW | **DM: RW · PM: R, only *visible for PM*** | RW | — | **never** |
| S8 | Feedbacks | See CAP-11, incl. joining interview feedback | R (only *shared with employee*) | RW | RW | RW | — | cfg |
| S9 | Career timeline | System-generated event log, see CAP-7 | R | RW | RW | RW | — | cfg (shareable, off by default) |
| S10 | Leaves and absences | Vacation/sick/parental/extended — dates and types | R | R | R | R | **R — dates only, type hidden** | cfg |
| S11 | Projects | Project, PM, DM, period | R | R | R | R | R (project name only) | cfg |
| S12 | CDS | See CAP-8 | R (+ complete own IDP) | RW | RW | RW | — | cfg |
| S13 | Mentorship | Open-to-mentor flag, mentor, mentees, ended pairs, mandatory closure note on pair end | RW (own flag), R (pairs) | RW | RW | RW | — | **never** |
| S14 | Action items and tasks | See CAP-4 | R (own) + mark complete | RW | RW | RW | — (campaign-sender exception, Rule 7) | **never** |
| S15 | Request history | Resourcing proposals for this person | — | R | R | R | — | cfg |
| S16 | Custom fields | See CAP-2 | per field visibility | RW | RW | RW | per field visibility | cfg |

¹ The *manager*, *people partner*, and *department* fields are **displayed** in S1 but are **not writable through it** — changing them is a separate operation with its own permission (see Access switches above).

## Rules that follow from the matrix

1. **Every cell is strict.** A `—` section, and every narrowed cell, must not reach that audience through any surface — UI, API, export, search result, notification, or error message. Same for flag-gated records. A leak is a critical defect regardless of which section it happens in.
2. **Manager access comes in two tiers.** The reporting line gets the full manager grant. The project line is **narrower**: no S2, no S3, and S5 limited to CV and certificates. Everything else is identical, including S6 risks, which a delivery manager genuinely needs — a delivery role needs skills, availability, risks and history; a residential address and an emergency contact serve line management and HR, and nothing in running a project requires them.
3. **S7 defaults to invisible to the employee and to PMs.** Every note carries two independent off-by-default flags: *visible for employee* and *visible for PM* (read-only for PM). The reporting line, DMs, and PP create/read/edit notes about their people regardless of flags.
4. **Colleague view is a whitelist, not a blacklist.** Exactly S1, the **dates** in S10 (leave type hidden), and the S11 project name. Everything else is absent — enforced at the API, not by hiding fields in the frontend. A colleague sees that somebody is away from a date to a date; they never learn whether it's vacation, sick leave, or parental leave. *(v1 gave colleagues the leave type too — reversed in this update.)*
5. **Access is evaluated server-side, per section, on every request**, after resolving the requester's access role.
6. **Custom fields carry their own visibility** (management default / employee / colleague); filters and list columns respect it — a filtering user must never infer a value they can't see.
7. **Campaign senders are the one exception to the colleague whitelist.** Whoever created a form campaign sees, **for that campaign only**, each recipient by name plus the status of that campaign's own action item — nothing else from S14, nothing from any other section (the sender still sees these people through the colleague view everywhere else), and it ends when the campaign closes. No other feature may widen the colleague view without a new product decision.
8. **The profile header's mentor field (part of S1) follows the stricter §4.11 rule, not the general S1 Colleague-R grant** — visible only to the reporting/project line and PP. Same shape as the S7/campaign exceptions: a specific, later rule overriding a general one for one field. *(Decision D5, `decisions.md`)*
9. **A Shared Link audience is always read-only**, regardless of which `cfg` sections are enabled for it — no shared-link surface ever returns `RW`.
10. **Multi-audience overlap resolves as union, per section.** When a viewer's relationships resolve to more than one audience for the same subject (e.g. both PP and Project line), effective access for each section is the **least-restrictive** access among all resolved audiences for that section — never a single ranked "winning" audience. *(D13, `decisions.md`)*
11. **A shared link's exposure re-clamps continuously, not just at creation.** On every view, each `cfg` section the link exposes is re-checked against the creator's *currently held* access to that section — not merely whether the creator's relationship to the subject still exists. If the creator's access narrows (e.g. Reporting line → Project line), any section the narrower tier no longer grants stops rendering on the very next view. *(D14)*

There are exactly **two** documented exceptions to "a manager sees everything": the project-line narrowing (Rule 2) and the PM's flag-gated read of S7 (Rule 3).

## Relationship and grant journal

A narrow journal, not a general audit log. It records exactly the events that change who can see what:

- a change to a person's **manager**;
- a change to a person's **people partner**;
- a change to a person's **department**;
- a change to a **department's manager**;
- a grant or revocation of **full profile access**;
- an **access via a shared link**.

Each entry holds the actor, the subject, the before and after values, and the timestamp. Readable by holders of full profile access, and by the current manager and people partner of the subject.

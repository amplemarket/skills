# HubSpot

HubSpot's equivalent is a **workflow**, under Automation → Workflows. HubSpot also
ships **journeys** (Marketing → Journeys, Marketing Hub Enterprise), which are several
workflows drawn as one staged canvas; decompose a journey stage-by-stage through this
file. HubSpot **sequences** are not workflows — a sequence is HubSpot's sequence, and a
workflow that enrolls someone into one is describing sequence membership.

This file describes **HubSpot**. It deliberately doesn't say what each part maps to in
Amplemarket — `create_workflow` answers that against the live catalogue, and a mapping
table here would be a stale second opinion. See the division of labour section in
[SKILL.md](../SKILL.md).

HubSpot renamed **lists** to **segments** in its docs, and renamed **Operations Hub**
to **Data Hub**. Accounts and older UI still say "list" and "Operations Hub". Treat the
pairs as the same thing.

## Where to read it

- **Live (preferred):** the workflows list is at `app.hubspot.com/workflows/<portalId>/`
  and an individual workflow at
  `app.hubspot.com/workflows/<portalId>/platform/flow/<flowId>/edit`. Older workflows
  use `/flow/<flowId>/edit` without `platform`. The portal ID is in every in-app URL, so
  read it off whatever link the user gives you rather than asking for it separately.
- The editor is a canvas rather than a form. Screenshot it as well as reading it — the
  branch structure is the part that renders as a graph and reads badly as text, and it's
  the part that decides how many Amplemarket workflows this becomes.
- Enrollment triggers, unenrollment, suppression, goal, and the execution schedule are
  each behind their own panel. Open all of them; a workflow captured from the canvas
  alone is missing its enrollment semantics entirely.

Every workflow in the migration set gets pinned to its own `/edit` URL before capture.
HubSpot lets workflows share names and accounts routinely have hundreds, so a name is
not an identifier.

## Check the object type before anything else

A HubSpot workflow is bound to one object type at creation and **can never be changed
afterwards**, so the type is a fact about the workflow rather than a setting. The
roster is long: contacts, companies, deals, tickets, quotes, leads, invoices, orders,
credit memos, contracts, subscriptions, payments, conversations, calls, meetings,
tasks, emails, goals, media, projects, users, campaigns, social posts, feedback
submissions, and custom objects — most of the non-core ones gated behind a specific Hub
and tier.

Only **contact-based** and **company-based** workflows are about reaching people.
Everything else is record automation: a deal-based workflow moves deals through stages,
a ticket-based one routes support, a quote-based one chases signatures. Amplemarket
workflows enroll contacts, so those have no target to act on — a structural mismatch
rather than a missing trigger, and worth saying at scope time before a generation is
spent on it.

Read the type off the workflow header and put it in the scope table. It's the cheapest
filter you have on a "migrate everything" request, and in most portals it drops more
than half the list.

## HubSpot workflows branch — the other three providers' don't

Outreach triggers, Salesloft rules, and Apollo plays are all flat: trigger, conditions,
one ordered action list. A HubSpot workflow is a **graph** — delays, if/then branches,
value-equals branches with up to hundreds of outputs, percentage splits, and a "go to
action" connector that lets branches rejoin.

So the usual decomposition doesn't hold, and this is the single thing most likely to
produce a wrong capture. **Walk the canvas and enumerate the leaf paths** before writing
anything; step 3 of [SKILL.md](../SKILL.md) covers what that does to the scope table.

Two shapes don't decompose into leaf paths at all, and need saying out loud instead:

- **A branch on something evaluated mid-run** — a property the workflow itself set two
  actions earlier, or a delay-until-event that resolves differently per record — isn't
  an audience filter. Splitting it into two workflows changes when the decision is made.
- **Percentage splits** (Marketing Hub Pro/Enterprise) are A/B infrastructure. Capture
  the ratio and flag it; it isn't a condition on the contact at all.

**Delays** are their own capture: pause for a set duration, until a calendar date, until
a date held in a property, until a weekday or time of day, or until a named event
happens. A nurture workflow is mostly delays, and the delay pattern *is* the automation
— dropping it turns a six-week sequence of touches into one burst.

## Enrollment: filter triggers versus event triggers

HubSpot offers two kinds, and they mean genuinely different things.

**Filter triggers** ("when filter criteria is met") are a *standing condition* on the
record — `Lifecycle stage is any of "Marketing Qualified Lead"`, `Country is any of
"United States"`, segment membership, form submitted, page viewed, marketing email
clicked, ad interacted with, privacy consent status, import source, activity on
another workflow. A record enrolls the first time it starts matching.

The part that catches people: **whether the back catalogue enrolls is a choice made when
the workflow is switched on**, not a property of the trigger. HubSpot's default is to
enroll only records that start matching afterwards; activating with "enroll existing
records which meet the trigger criteria as of now" ticked instead sweeps in everyone who
already matches, which can be thousands at once. That choice isn't stored on the canvas,
so ask the user which way the source was turned on — it separates a workflow that has
already processed the whole database from one that has only ever seen new matches.

**Event triggers** ("when an event occurs") fire on the occurrence itself and **only
count occurrences after the workflow was turned on** — no backfill, and no option to ask
for one. HubSpot lists
over sixty of them across ads, calls, CRM changes, custom events, email, forms,
meetings, sequences, SMS, and website activity. Any additional filters attached to an
event trigger are evaluated only at the instant the event fires, which is stricter than
it looks: a record that would have matched a minute later doesn't enroll. Event triggers
also can't be combined with scheduled enrollment.

So capture *which kind* each trigger is, not just what it says. `Lifecycle stage is
Customer` as a filter trigger and `Lifecycle stage changed to Customer` as an event
trigger read almost identically on the canvas and behave completely differently on
activation.

Triggers combine in AND/OR groups; capture the grouping, because an OR group is usually
two automations sharing a canvas and it may want to become two workflows for the same
reason a branch does.

## Re-enrollment, unenrollment, suppression, and the goal

These four panels are where HubSpot keeps the behaviour that the canvas doesn't show,
and they change enrollment volume more than anything in the action list.

- **Re-enrollment is off by default.** A record enrolls once, ever, unless the user has
  explicitly toggled re-enrollment on and ticked the specific triggers to re-enroll on.
  Even then a record can't re-enter while it's still enrolled — only after it exits.
  When it does re-enter it starts from the top and runs every action again. Not every
  trigger is eligible: activities and activity properties, calculated properties,
  privacy consent events, and contact segment membership can't be re-enrollment
  triggers at all. Capture which triggers are ticked, not just that the toggle is on.
- **Unenrollment triggers** pull a record out mid-run. There's also a setting to
  unenroll records that stop meeting the enrollment criteria, which only applies to
  filter triggers.
- **Suppression segments** are an exclusion list evaluated continuously: a member never
  enrolls, and an enrolled record that joins one is removed immediately and receives no
  further actions. Teams use them for customers, competitors, and employees. A migrated
  workflow without the suppression is a workflow that now emails people it used to
  deliberately skip — capture the segment and what it's for.
- **Goal criteria** (contact-based) unenroll a record the moment it meets them, before
  the next action runs. A nurture workflow with a goal of "became a customer" stops
  touching people who convert. Without it, they keep getting the rest of the sequence.

Also read the **settings** tab: workflows can be restricted to specific days and times,
paused on named dates, and set to switch themselves off after a date. Contact-based
workflows can additionally unenroll a contact from other workflows on enrollment. All of
that is timing behaviour that won't be visible anywhere else in the capture.

## Properties: capture the internal name, not just the label

This is where HubSpot differs usefully from the other three. Salesloft's Person Stage
and Apollo's Contact stage are vendor-native labels that go nowhere. **A HubSpot
property is a CRM field** — if HubSpot is also the account's CRM, the field the source
workflow writes is the same field the migration can write.

That makes the capture more valuable and slightly more work. Every property has a
human-readable **label** and an **internal name** (`Lead status` / `hs_lead_status`;
`Lifecycle stage` / `lifecycle_stage`), and custom properties get portal-specific
internal names. Capture both, plus the exact picklist values as HubSpot spells them,
since enumeration values have their own internal values distinct from their labels.

It does **not** mean pasting the HubSpot name straight into the prompt. Harvest the
account's own vocabulary first per
[CRM fields are the one thing to resolve first](../SKILL.md#crm-fields-are-the-one-thing-to-resolve-first)
— the account may run Salesforce with HubSpot as a marketing front end, in which case
the HubSpot internal name is the wrong one, and CRM writes need the integration
configured regardless.

Two property families are provider-computed and don't travel: **calculated and rollup
properties** (score fields, "number of associated deals", "days to close") are derived
by HubSpot from HubSpot data, and **HubSpot score** and marketing contact status are
HubSpot's own models and billing state. A workflow gated on one of those doesn't have an
audience definition that survives the move.

## Actions

HubSpot's action catalogue is the largest of the four providers and heavily **tier-
gated**, which matters at capture time: a two-action workflow can depend on two separate
paid Hubs, and the user may not know that. Note the tier next to each captured action.

| Family | Actions | Gating worth noting |
| --- | --- | --- |
| Communication | Send marketing email, send internal email notification, in-app notification, SMS, WhatsApp, survey | Marketing/Service Hub Pro+; SMS needs the add-on |
| Sequences | Enroll in sequence, unenroll from sequence | Sales or Service Hub **Enterprise**, and the sender needs a paid seat and a connected personal inbox |
| CRM writes | Edit record, increase/decrease a numeric property, copy property, manage communication subscriptions, format data | Format data is Data Hub (Operations Hub) Pro+ |
| Record creation | Create contact, company, deal, ticket, lead, note, task, quote, invoice | Ticket/lead/custom-object creation gated by Hub and tier |
| Ownership | Rotate record to owner, rotate by skill, assign conversation owner | Sales/Service Hub Pro+, requires paid seats |
| Associations | Create associations, apply/update/remove association labels | — |
| Segments and ads | Add to / remove from static segment, add to / remove from ads audience, add to campaign | Marketing Hub Pro+ |
| External | Send a webhook, custom code (JavaScript) | **Data Hub Pro+** |
| Integrations | Slack, Google Chat, Zoom, Asana, Trello, Salesforce task/campaign, DocuSign, NetSuite | Needs the app connected |
| AI | Breeze data agents, record summarisation, prospecting agent, custom LLM | Consumes HubSpot credits |

Three of these need care beyond noting them.

**Send marketing email is the point of most HubSpot workflows**, and it is not sequence
enrollment. It sends a HubSpot-authored marketing email from the portal's sending
domain, gated by the contact's subscription type and — where data privacy settings are
on — by their legal basis for processing. Contacts without a valid basis are skipped
silently. So a nurture workflow carries an entire consent model that lives in HubSpot,
and moving the touches to rep-sent sequence steps changes the sender, the deliverability
profile, and which contacts are eligible at all. Establish with the user whether that's
what they want before it becomes a migration rather than a rebuild.

**Custom code actions** are arbitrary JavaScript with access to the record and to portal
secrets. Read what the code does and describe its effect; never transcribe a secret, and
record only that one is referenced. Most custom code actions are doing an integration
HubSpot didn't ship, which usually means the migration question is "where does this
logic live now", not "which action replaces it".

**Rotate record to owner** is round-robin assignment across a team, and it's the action
whose absence gets noticed the same afternoon.

## The loss report for a HubSpot migration

- **Marketing email sends.** The largest one by far. The emails stop going out from
  HubSpot, so the subscription types, the legal-basis gating, and every email
  performance report keyed to those sends go with them. Name the emails.
- **Suppression segments and goal criteria.** Both are *exclusions*, and both belong in
  the prompt described by meaning — never dropped on the assumption they won't survive.
  What doesn't travel is that HubSpot re-checks them continuously and ejects records
  mid-run; an enrollment-time filter only decides at the door. Whatever isn't reproduced
  means the workflow touches strictly more people than the original — customers and
  competitors it used to skip, converters it used to release.
- **The branch structure.** If one workflow became three, say so plainly and say which
  path each one covers, because from now on the user maintains three things that used to
  be edited in one place.
- **Owner rotation.** A round-robin that stops rotating leaves records unassigned or all
  on one person. Operational, immediate, and visible.
- **Internal notifications and Slack posts.** Nobody gets pinged any more. Offer the
  webhook rebuild if there's an incoming webhook to post to.
- **Custom code and webhook actions.** Whatever they integrated with stops being
  called. Name the external system.
- **Calculated properties and HubSpot score.** If they gated enrollment, the rebuilt
  audience is broader.
- **Properties that stop being written.** Same shape as Salesloft's Person Stage, with
  a twist: because HubSpot properties are CRM fields, reports, active segments, and
  *other HubSpot workflows* read them. A property this workflow set may be another
  workflow's enrollment trigger, so switching one off can quietly stall a second one.
  Search the portal for other workflows keyed on the same property before switching over.

## Worked example

Workflow **"MQL → SDR handoff"** — contact-based. Filter enrollment trigger:
`Lifecycle stage is any of "Marketing Qualified Lead"` AND `Country is any of "United
States", "Canada"`; re-enrollment off; suppression segment "Existing customers".
Actions: `Edit record` → `Lead status` (`hs_lead_status`) = "New"; `Rotate record to
owner` across the SDR team; an if/then branch on `Number of employees is greater than
500` enrolling the true path in sequence "Enterprise MQL follow-up" and the false path
in "SMB MQL follow-up"; both paths then `Send internal email notification` to the
contact owner.

The branch means this is **two** automations, so it gets two rows in the scope table and
two generations. Row 1, the enterprise path:

> When a contact's lifecycle stage becomes Marketing Qualified Lead, set their CRM lead
> status field to "New" and add them to the "Enterprise MQL follow-up" sequence. Only
> apply this to contacts in the United States or Canada whose company has more than 500
> employees, and don't enroll anyone who is already a customer. Enroll each contact once.

Row 2 is the same prompt with 500 or fewer employees and the "SMB MQL follow-up"
sequence.

Note that the suppression segment **is** in the prompt, described by what it does rather
than by its name. Leaving an exclusion out because it looks unsupported is the mistake
[SKILL.md](../SKILL.md) opens by warning about, and it's the expensive direction to get
wrong: a dropped exclusion produces a draft that enrolls the exact people the source was
built to skip. Describe it and let the planner rule. What the prompt genuinely doesn't
carry is the owner rotation and the internal notification.

**What you lose by moving this**

- Suppression segment "Existing customers" — the prompt asks for the exclusion, but
  HubSpot was evaluating live membership of a maintained segment and pulling already-
  enrolled contacts back out the moment they joined it — an enrollment-time condition
  can't do that second half. Agree what defines "already a customer" on this side, and
  check the draft actually carries the exclusion before activating.
- SDR round-robin assignment — records land unassigned instead of rotating across the
  team — decide who owns them, because it's the change the team notices the same day.
- Internal email to the contact owner — nobody gets notified on handoff — rebuild it as
  an HTTP request to an incoming webhook, or tell the team it's going away.
- One workflow becomes two — the employee-count branch is now an audience filter in two
  separate workflows — a future change to the shared part has to be made twice, and a
  contact sitting exactly at 500 employees should be re-checked against the boundary.
- HubSpot still owns `hs_lead_status` — other HubSpot workflows and active segments may
  be keyed on it, and one of them may have been triggered by this workflow's write.
  Search the portal for workflows enrolling on lead status before switching this off.

## Sources

- [Create workflows](https://knowledge.hubspot.com/workflows/create-workflows) — where workflows live, from-scratch vs template vs Breeze, the editor canvas
- [Understand workflow object types](https://knowledge.hubspot.com/workflows/understand-workflow-object-types) — the object roster and its subscription gating
- [Set filter enrollment triggers](https://knowledge.hubspot.com/workflows/set-filter-enrollment-triggers) — filter trigger categories and operators
- [Set event enrollment triggers](https://knowledge.hubspot.com/workflows/set-event-enrollment-triggers) — event triggers, no backfill, filters evaluated at the instant of the event
- [Add re-enrollment triggers to a workflow](https://knowledge.hubspot.com/workflows/add-re-enrollment-triggers-to-a-workflow) — enroll-once default, which triggers are ineligible
- [Choose your workflow actions](https://knowledge.hubspot.com/workflows/choose-your-workflow-actions) — the full action catalogue with tier gating
- [Manage workflow enrollment settings](https://knowledge.hubspot.com/workflows/manage-workflow-enrollment-settings) and [Manage your workflow settings](https://knowledge.hubspot.com/workflows/manage-your-workflow-settings) — suppression, unenrollment, execution schedule, connections
- [Use goals in contact-based workflows](https://knowledge.hubspot.com/workflows/use-goals-in-contact-based-workflows) — goal criteria unenroll before the next action
- [Enroll and unenroll contacts in sequences using workflows](https://knowledge.hubspot.com/sequences/enroll-and-unenroll-contacts-in-sequences-using-workflows) — Enterprise gating, connected inbox, one sequence at a time
- [Send automated emails in workflows](https://knowledge.hubspot.com/marketing-email/create-automated-emails-to-use-in-workflows) and [Track legal basis of processing](https://knowledge.hubspot.com/contacts/how-can-i-track-lawful-basis-of-processing-in-hubspot) — subscription types and legal-basis gating on sends
- [Create segments](https://knowledge.hubspot.com/segments/create-active-or-static-lists) — active vs static, and the lists → segments rename
- [Use journey automation](https://knowledge.hubspot.com/workflows/use-journeys) — journeys as several workflows on one canvas

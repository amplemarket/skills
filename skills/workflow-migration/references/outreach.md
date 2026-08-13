# Outreach

Outreach's equivalent of a workflow is a **trigger**, under Administration →
Workflow automations → Triggers. One Outreach trigger becomes one Amplemarket
workflow.

This file describes **Outreach**. It deliberately doesn't say what each part maps to
in Amplemarket — `create_workflow` answers that against the live catalogue, and a
mapping table here would be a stale second opinion. See the division of labour section
in [SKILL.md](../SKILL.md).

## Where to read it

- **Live (preferred):** the triggers list is at `web.outreach.io/admin-exp/triggers`
  and an individual trigger at `web.outreach.io/admin-exp/triggers/<id>/edit`. Read
  the page for the form structure and take a screenshot to confirm conditions and
  actions that render as chips.
- A screenshot or plain text the user pastes works equally well.

Every trigger in the migration set gets pinned to its own `/edit` URL before capture.
If a name the user gave matches more than one row in the list, or none, ask them for
the URL — Outreach lets triggers share names, so a name is not an identifier.

Capture the labels as Outreach writes them, typos included; normalize when you write
the prompt.

## Anatomy

The trigger form is: name, owner, event, target, frequency, conditions, actions.

1. **Event** — an **object** plus a change type
2. **Target** — the object the actions apply to, which is not always the object that
   fired the trigger
3. **Frequency** — the "Trigger only once per target" checkbox
4. **Conditions** — rows on Prospects, Accounts, Calls, Meetings, and Opportunities,
   offered according to the event chosen
5. **Actions** — six kinds, filtered by the target you picked

## Events: read the conditions, not the object

This is the single most important thing about Outreach, and the reason a naive capture
produces a wrong prompt.

**Outreach has no semantic events.** There is no "meeting booked" trigger. There is
`Meeting / Created or Updated`, and the meaning lives entirely in the conditions
attached to it. The same object-plus-change-type becomes a booked-meeting trigger, a
no-show trigger, or a cancellation trigger depending on a condition row further down
the form.

So: **never describe an Outreach trigger from its event alone.** Read the event and
its conditions together, work out what the trigger *means*, and put that meaning in
the prompt. `Mailing / Updated` with a condition on a replied state is "when a contact
replies to an email" — write that, not "when a mailing is updated".

Objects: **Prospect, Account, Meeting, Call, Mailing, Opportunity**, plus integration
objects (**Sendoso, Drift, Intercom, Seismic, Vidyard, BombBomb**) and custom objects
such as Campaign Member. Change types: **Created**, **Updated**, **Created or
Updated**. Prospect Created, Opportunity Created, and Opportunity Close Date can be
delayed; time-based triggers use a **Scheduled** event.

Three event families need care:

- **Opportunity events** are deal-stage automation. Amplemarket workflows enroll
  contacts, so there's no target to act on — a structural mismatch rather than a
  missing trigger. Flag it at scope time.
- **Scheduled events** are time-driven rather than event-driven. Say so plainly; a
  workflow that waits for an event is a different mechanism with different timing.
- **A Prospect or Account event is not a CRM event.** It fires on *Outreach's* record
  changing, not on Salesforce or HubSpot changing. If the intent really is "when the
  CRM record changes", say that; if it's "when the Outreach record changes", say that
  instead. They're different automations and the prompt has to pick one.

A trigger whose conditions accept any sequence-finished state, replied or not, is two
automations wearing one hat. Capture both halves and describe both — if only one half
survives, the other belongs in the loss report.

## Target and frequency

| Outreach target | What it means |
| --- | --- |
| Prospect | the person who fired the trigger |
| Primary Prospect | the account's designated primary contact — a distinction Outreach maintains and most tools don't |
| Prospect Roles | prospects filtered by their role on the account |
| All Prospects (For Account) | everyone at the account |
| Account | the company record |
| Opportunity · All Prospects (For Opportunity) | scoped through a deal |

"Trigger only once per target" checked means once per person, ever. Unchecked means it
re-fires. Capture which, because it changes enrollment volume substantially.

## Conditions

Conditions sit on Prospects, Accounts, Calls, Meetings, and Opportunities.

**Prospect and Account conditions** — title, seniority, location, industry, employees,
revenue, technologies, keywords — are ordinary audience criteria. State them in the
prompt in plain English and let the planner resolve field names. Note Outreach's
operator: `contains` is a loose match, `is` / `equals` is exact, and the difference
matters enough to preserve in the phrasing.

**Conditions on the Call, Meeting, or Opportunity record itself** — call disposition
or sentiment, meeting type or outcome, opportunity stage or amount — describe the
*event* that fired the trigger rather than the person. Capture the configured values
verbatim; a disposition label may well have a counterpart on the other side, and
quoting it gives the planner something to match. If a condition doesn't survive, the
automation fires on a wider audience than it used to — that's a loss-report line, and
wider means more contacts.

**Field-change semantics** — "when the value changes", "changes from X", "changes to
Y" — are about the *transition*, not the current value. A condition testing the
current state fires more often than one testing the change. Capture which one the
trigger uses.

## Actions

Outreach offers exactly six, and the list is filtered by the target — an Opportunity
target has no Add to Sequence, for instance.

| Outreach action | What it does |
| --- | --- |
| Add to Sequence | enrolls the target in a named sequence |
| Stop Sequences | marks sequences Finished — scoped to all, related, or specific ones. Capture the scope. |
| Create Task | a call, email, meeting, or SMS task, optionally assigned to a team queue |
| Set Field | writes a prospect or account field. Ask whether it's a CRM-synced field or Outreach-native — same control, different migration. |
| Add Tags | attaches free-form labels |
| Remove Tags | removes them |

Note what *isn't* there: Outreach triggers have no webhook or HTTP action — Outreach
webhooks are configured separately, outside triggers. So an Outreach migration
shouldn't produce an HTTP step unless the user asks for one, and if they do, that's
new scope rather than a migration.

## The loss report for an Outreach migration

- **Tags.** Outreach tags are used for segmentation and reporting, and a rule that
  tags people is feeding something. Ask what reads the tag.
- **Set Field on an Outreach-native field.** Same problem as Salesloft's Person Stage:
  the field stops being written, and whatever filters or reports on it goes stale.
- **Task queue and owner assignment.** Outreach can route a task to a team queue.
  If the migrated task lands unassigned or on one person, the routing is gone — say
  so, because it's an operational change someone notices immediately.
- **The Primary Prospect and Prospect Role distinctions.** If the trigger relied on
  them to pick one contact per account, losing them changes who gets touched.
- **Conditions on the call/meeting record.** As above: fewer conditions means more
  contacts enrolled.

## Worked example

Trigger **"Stop All Sequences When Meeting is Booked"** — Event `Meeting` /
`Created or Updated`; Target `Prospect`; "Trigger only once per target" checked;
meeting condition on a booked state; prospect condition `Title contains "VP"`;
account condition `Industry is "Technology"`; action `Stop Sequences` scoped to
"mark all sequences finished".

The meeting condition is what makes this a booked-meeting trigger — the event alone
would be ambiguous. That meaning goes into the prompt:

> When a meeting is booked with a contact, mark all of that contact's active sequences
> as finished. Only apply this to contacts whose job title contains "VP" and whose
> company is in the Technology industry. Enroll each contact only once.

**What you lose by moving this:** nothing — the rule's full effect is reproduced. The
meeting-state condition isn't restated as an audience filter because it's already
carried by "when a meeting is booked"; worth mentioning to the user so nobody goes
looking for it in the generated draft.

## Sources

- [Triggers Overview](https://support.outreach.io/support/solutions/articles/159000425797-triggers-overview) — objects, event types, actions
- [How To Create an Outreach Trigger](https://support.outreach.io/support/solutions/articles/159000426220-how-to-create-an-outreach-trigger) — form fields, targets, frequency, conditions
- [Opportunity Triggers](https://support.outreach.io/support/solutions/articles/159000426382-opportunity-triggers) — how the action list narrows by target
- [Create Triggers with Custom Objects](https://support.outreach.io/support/solutions/articles/159000425392-create-triggers-with-custom-objects)

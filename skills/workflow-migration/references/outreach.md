# Outreach → Amplemarket workflows

Outreach's equivalent of a workflow is a **trigger**, under Administration →
Workflow automations → Triggers. One Outreach trigger becomes one Amplemarket
workflow.

## Where to read it

- **Live (preferred):** the triggers list is at `web.outreach.io/admin-exp/triggers`
  and an individual trigger at `web.outreach.io/admin-exp/triggers/<id>/edit`. Read
  the page for the form structure and take a screenshot to confirm conditions and
  actions that render as chips.
- A screenshot or plain text the user pastes works equally well.

Every trigger in the migration set gets pinned to its own `/edit` URL before capture.
If a name the user gave matches more than one row in the list, or none, ask them for
the URL — Outreach lets triggers share names, so a name is not an identifier.

Capture the labels as Outreach writes them, typos included; normalize when mapping.

## Anatomy

The trigger form is: name, owner, event, target, frequency, conditions, actions.

1. **Event** — an **object** plus a change type. Outreach has no semantic events like
   "meeting booked"; you get "Meeting / Updated" and the meaning lives in the
   conditions. Read event and conditions together before mapping.
2. **Target** — the object the actions apply to, which is not always the object that
   fired the trigger
3. **Frequency** — the "Trigger only once per target" checkbox
4. **Conditions** — rows on Prospects, Accounts, Calls, Meetings, and Opportunities,
   offered according to the event chosen
5. **Actions** — six kinds, filtered by the target you picked

## Trigger event

Objects: **Prospect, Account, Meeting, Call, Mailing, Opportunity**, plus integration
objects (**Sendoso, Drift, Intercom, Seismic, Vidyard, BombBomb**) and custom objects
such as Campaign Member. Change types: **Created**, **Updated**, **Created or
Updated**. Prospect Created, Opportunity Created, and Opportunity Close Date can be
delayed; time-based triggers use a **Scheduled** event.

Map the object-plus-condition combination, not the object alone:

| Outreach event + the conditions that give it meaning | Amplemarket trigger |
| --- | --- |
| Meeting / Created or Updated, condition on a booked state | `event_meeting_booked` |
| Meeting / Updated, condition on no-show, canceled, or rescheduled | No equivalent |
| Mailing / Updated, condition on a replied state | `event_email_replied` |
| Mailing / Updated, condition on a bounced state | `event_email_bounced` |
| Mailing / Updated, condition on opened or clicked | No equivalent |
| Call / Created (a logged call) | `event_call_logged` |
| Prospect / Updated, condition on sequence membership starting | `event_sequence_started` or `event_sequence_lead_added_to_sequence` |
| Prospect / Updated, condition on a sequence finishing with no reply | `event_sequence_completed_with_no_reply` |
| Opportunity / Created · Updated · Close Date | No equivalent — Amplemarket workflows have no opportunity or deal concept |
| Scheduled (time-based) | No equivalent — Amplemarket workflows are event-driven only |
| Sendoso, Drift, Intercom, Seismic, Vidyard, BombBomb, Campaign Member, custom objects | No equivalent; if the source system can post a webhook, `webhook_received` is the rebuild, and it must be the workflow's only trigger |

**A Prospect or Account event is not a CRM event.** Amplemarket's `event_crm_*`
triggers watch *Amplemarket's own* Salesforce or HubSpot integration, not Outreach's
records. Only map an Outreach Prospect/Account trigger onto `event_crm_contact_*` or
`event_crm_account_*` when the intent really is "when the CRM record changes" and
Amplemarket has that CRM connected — otherwise it fires on a different thing entirely.

A trigger whose conditions accept any sequence-finished state, replied or not, only
half-maps: `event_sequence_completed_with_no_reply` covers the no-reply case and
nothing covers the replied case. Record that in the gap list instead of mapping the
whole trigger.

## Target and frequency

| Outreach target | Amplemarket |
| --- | --- |
| Prospect | scope `contact` |
| Primary Prospect | scope `contact`, but "primary" has no equivalent — note that the workflow won't distinguish primary contacts |
| Prospect Roles | scope `contact`; the role selection has no equivalent and has to be re-expressed as an audience filter, or dropped |
| All Prospects (For Account) | scope `account_contacts` |
| Account | scope `account_contacts` |
| Opportunity · All Prospects (For Opportunity) | No equivalent — the opportunity gate can't be expressed |
| "Trigger only once per target" checked | reenroll `never` |
| Unchecked | reenroll `always` |

## Conditions

Conditions can sit on Prospects, Accounts, Calls, Meetings, and Opportunities. Only
the first two migrate cleanly.

| Outreach condition | Amplemarket searcher field |
| --- | --- |
| Prospect Title | `person_titles` (`person_titles_exact_match` when the operator is "is") |
| Prospect Seniority / Level | `person_seniorities` |
| Prospect Country / Location | `person_location` |
| Account Industry | `company_industry` |
| Account Employees / Size | `company_size` or `company_headcount` |
| Account Country / Location | `company_location` |
| Account Revenue | `company_revenue_ranges` |
| Account technologies | `company_technology` |
| Account keywords | `company_keywords` |

Outreach `contains` stays loose (`person_titles` already matches on contains);
`is` / `equals` becomes the exact variant or a single picklist value. Prospect and
account fields that are CRM-synced map to **crm** filters instead (CRM-gated).

Conditions on the **Call, Meeting, or Opportunity record itself** — call disposition
or sentiment, meeting type or outcome, opportunity stage or amount — have no
equivalent. Amplemarket enrollment filters describe the person and the company, not
the event that fired the workflow. A trigger leaning on those will enroll a wider
audience than the original; say so, because wider means more contacts.

Field-change semantics ("when the value of a field changes", "changes from X", "to Y")
don't carry either: Amplemarket filters test the current value only, so a trigger that
depends on the transition rather than the state fires more often after migration.

## Actions

Outreach offers exactly six, and the list is filtered by the target — an Opportunity
target has no Add to Sequence, for instance.

| Outreach action | Amplemarket action |
| --- | --- |
| Add to Sequence | `add_to_sequence` |
| Stop Sequences (all, related, or specific; marks them Finished) | `complete_sequence` |
| Create Task (call, email, meeting, SMS; optionally assigned to a team queue) | `create_one_off_task` — the queue or owner assignment has no equivalent |
| Set Field | `update_crm_fields`, but only when the field is a CRM field Amplemarket can write; Outreach-native prospect and account fields have no equivalent |
| Add Tags | No equivalent |
| Remove Tags | No equivalent |

Note what *isn't* on that list: Outreach triggers have no webhook or HTTP action —
Outreach webhooks are configured separately, outside triggers — so an Outreach
migration rarely produces an `http_request` step. If the user wants one, that's new
scope rather than a migration.

Amplemarket actions with no Outreach counterpart (`add_to_lead_list`,
`remove_from_lead_list`, `add_to_exclusion_list`, `enrich_data`,
`remove_from_sequence`) are worth mentioning as upgrades the target platform offers,
but never invent them into a migration the source didn't ask for.

## No equivalent

- **Events:** opportunity created/updated/close date, scheduled and time-based
  triggers, email opened or clicked, meeting rescheduled or canceled, integration and
  custom objects, field-change transitions.
- **Targets:** opportunity-scoped targets, primary-prospect and prospect-role
  distinctions.
- **Conditions:** anything reading the call, meeting, or opportunity record.
- **Actions:** add or remove tags, set an Outreach-native field, task owner and team
  queue assignment.

## Worked example

Trigger **"Stop All Sequences When Meeting is Booked"** — Event `Meeting` /
`Created or Updated`; Target `Prospect`; "Trigger only once per target" checked;
meeting condition on a booked state; prospect condition `Title contains "VP"`;
account condition `Industry is "Technology"`; action `Stop Sequences` scoped to
"mark all sequences finished".

The meeting condition is what makes this a booked-meeting trigger, and that's exactly
what `event_meeting_booked` means, so it maps to trigger `event_meeting_booked`,
scope `contact`, reenroll `never`, searcher filters `person_titles` contains "VP" plus
`company_industry` "Technology", action `complete_sequence`:

> When a meeting is booked with a contact, mark all of that contact's active sequences
> as finished. Only apply this to contacts whose job title contains "VP" and whose
> company is in the Technology industry. Enroll each contact only once.

**Gaps:** none. The meeting-state condition isn't carried across as a filter because
the Amplemarket trigger already means it — worth stating, so nobody goes looking for
it in the generated draft.

## Sources

- [Triggers Overview](https://support.outreach.io/support/solutions/articles/159000425797-triggers-overview) — objects, event types, actions
- [How To Create an Outreach Trigger](https://support.outreach.io/support/solutions/articles/159000426220-how-to-create-an-outreach-trigger) — form fields, targets, frequency, conditions
- [Opportunity Triggers](https://support.outreach.io/support/solutions/articles/159000426382-opportunity-triggers) — how the action list narrows by target
- [Create Triggers with Custom Objects](https://support.outreach.io/support/solutions/articles/159000425392-create-triggers-with-custom-objects)

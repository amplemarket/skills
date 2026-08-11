# Apollo → Amplemarket workflows

Apollo calls this area **Workflows** and **Plays**. A workflow is the mechanism —
trigger → conditions → actions, either event-driven or run on a schedule — and a play
is a packaged workflow built around a signal, which is how most customers actually
use it. Both decompose the same way, so migrate either through this file. Apollo
**Sequences** are not workflows; a sequence maps to an Amplemarket sequence, and a
play that enrolls someone into one maps to `add_to_sequence`.

## Where to read it

Workflows and plays live in `app.apollo.io`, but Apollo moves this area around and
renames it, so this reference deliberately doesn't hardcode the URL — ask the user
for the link to their workflows or plays list, and for the URL of each one in the
migration set. Open each and read the trigger card, the filter set, and the action
list; the list view collapses filters, so it isn't enough on its own. A screenshot or
pasted text works as a fallback.

Because the roster changes, treat the tables below as the mapping for what a play
*means* rather than for a fixed UI. If the play uses a trigger or action that isn't
listed, map it by meaning and say in the gap list that the mapping is an inference.

## Anatomy

1. **Trigger** — a signal, an engagement event, or a schedule
2. **Target** — people or accounts
3. **Filters** — person and company criteria that gate it
4. **Actions** — ordered, run when the filters match
5. **Whether a person can enter more than once**

## Trigger

The split that decides most of the migration is signal triggers versus engagement
triggers. Engagement triggers map cleanly; signal triggers — the reason people build
plays in Apollo — don't map at all.

| Apollo trigger | Amplemarket trigger |
| --- | --- |
| Contact replies to a sequence email | `event_email_replied` |
| Email bounced | `event_email_bounced` |
| Contact added to a sequence | `event_sequence_lead_added_to_sequence` |
| Sequence started | `event_sequence_started` |
| Sequence finished with no reply | `event_sequence_completed_with_no_reply` |
| Call logged | `event_call_logged` |
| Meeting booked | `event_meeting_booked` |
| Meeting rescheduled or canceled | No equivalent — Apollo's meeting triggers cover all three, Amplemarket only the booking |
| LinkedIn connection accepted or replied | `event_linkedin_connection_accepted` / `event_linkedin_replied` |
| CRM record created or updated | `event_crm_contact_*` / `event_crm_account_*` / `event_crm_lead_*` (CRM-gated) |
| Inbound webhook | `webhook_received` (must be the workflow's only trigger) |
| Email opened or clicked | No equivalent |
| Scheduled run ("run this workflow on a schedule") | No equivalent — Amplemarket workflows are event-driven only |

**Signal triggers have no Amplemarket workflow trigger at all**: job change, funding
round, hiring and job postings, headcount growth, technology added, website visitor
identification, buying-intent score changes, news mentions, and the deal-stall
triggers behind Apollo's Deal Plays. Nothing fires a workflow off any of them.

What Amplemarket has instead is the same signals as *search* criteria:
`person_recently_changed_company`, `company_last_funding_types` and
`company_last_funding_amount`, `company_has_recent_job_openings` and
`company_recent_job_openings_seniorities`, `company_technology`, `company_news`. So a
signal play is rebuildable as a saved search or a periodically refreshed lead list
that feeds a sequence — a different mechanism, with different timing, and someone has
to own refreshing it. Say that plainly. Don't bend a signal into a trigger.

## Target and re-entry

Person-level plays become scope `contact`; plays acting on everyone at an account
become `account_contacts`. Apollo plays typically re-run whenever the signal recurs,
so default to reenroll `always` unless the play is explicitly once-per-person — and
confirm it, because it changes how many contacts get enrolled.

## Filters

This is the easy half of an Apollo migration. Apollo's person and company filters
line up almost one-to-one with Amplemarket searcher fields — title, seniority,
department, location, industry, headcount, revenue, technologies, keywords all have
direct equivalents in the field table in SKILL.md. Filters reading a synced CRM field
map to **crm** filters and need the CRM integration.

Apollo-native concepts don't carry: contact stage, list membership as a filter, owner,
Apollo score, email status, and buying-intent score. Buying intent is the one that
bites, because it's often the whole point of the play — there's no Amplemarket
equivalent as a filter or a trigger. If a play's filters rest entirely on Apollo-native
values, the migrated workflow is broader than the original; flag it, since broader
means more contacts enrolled.

## Actions

| Apollo action | Amplemarket action |
| --- | --- |
| Add to sequence | `add_to_sequence` |
| Remove from sequence | `remove_from_sequence` |
| Mark sequence finished | `complete_sequence` |
| Add to list | `add_to_lead_list` |
| Remove from list | `remove_from_lead_list` |
| Mark do-not-contact / opt out | `add_to_exclusion_list` |
| Enrich email · Enrich phone | `enrich_data` (one action covers both) |
| Update a CRM field / push to CRM / change CRM stage | `update_crm_fields` (CRM-gated, updateable fields only) |
| Create a task | `create_one_off_task` — Apollo's assignment to a specific rep has no equivalent |
| Send a webhook / HTTP request | `http_request` (auth, headers, and testing are editor-owned) |
| Send email or Slack notification | No equivalent |
| Assign or route to a rep / change owner | No equivalent |
| Convert to an opportunity, create a deal | No equivalent |
| AI-drafted email actions | No equivalent |

Notification actions are worth a sentence rather than a silent drop: nearly every
Apollo play ends in a Slack alert, and its absence is the change users notice first.
A Slack post can sometimes be rebuilt as an `http_request` to an incoming webhook —
propose it as the user's decision, and configure the URL in the editor rather than
putting it in the prompt.

## No equivalent

- **Triggers:** every signal (job change, funding, hiring, headcount growth,
  technology added, website visits, buying intent, news, deal stalls), email opened or
  clicked, meeting rescheduled or canceled, scheduled runs, contact stage changed.
- **Filters:** buying-intent score, contact stage, list membership, owner, Apollo
  score, email status.
- **Actions:** notifications, owner assignment and routing, record and deal creation,
  AI email drafting.

## Worked example

Play **"Champion job change"** — trigger `Job change` over a saved list of past
customers; filters `Title contains "VP"` and `Company headcount 200-5000`; actions
`Enrich email`, `Add to list`, `Add to sequence "Champion re-engagement"`, and
`Send Slack notification`.

The trigger doesn't migrate, so the play splits in two. The part that is a workflow:

> When a contact is added to the "Champion job changes" lead list, enrich their data
> and add them to the "Champion re-engagement" sequence. Only apply this to contacts
> whose job title contains "VP" and whose company has between 200 and 5000 employees.
> Enroll each contact once.

**Gaps:**

- Job change trigger — no Amplemarket workflow trigger covers signals — rebuild the
  detection half as a search on `person_recently_changed_company` over the
  past-customer list, refreshed on whatever cadence the team wants, feeding the lead
  list this workflow watches. Apollo fires inside its own detection window; this fires
  when the search is refreshed.
- Slack notification — no notification action — dropped; rebuildable as an
  `http_request` to an incoming webhook if the team wants it.

## Sources

- [Workflows Overview](https://knowledge.apollo.io/hc/en-us/articles/14296116597901-Workflows-Overview) and [Create a Workflow](https://knowledge.apollo.io/hc/en-us/articles/4413804036109-Create-a-Workflow) — trigger → conditions → actions, event-driven or scheduled
- [Engage decision makers who just changed jobs](https://www.apollo.io/academy/plays/engage-decision-makers-who-just-changed-jobs) — a signal play's trigger, filters, and action list (notify, enrich email, enrich phone, add to list, add to sequence)
- [Apollo release notes](https://knowledge.apollo.io/hc/en-us/articles/27365542998925-Release-Notes-2024) — meeting triggers (booked, rescheduled, canceled) and Deal Plays
- [Manage Sequence Rulesets](https://knowledge.apollo.io/hc/en-us/articles/4409396858509-Manage-Sequence-Rulesets) — sequence-level automation, distinct from plays

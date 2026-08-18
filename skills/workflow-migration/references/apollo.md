# Apollo

Apollo calls this area **Workflows** and **Plays**. A workflow is the mechanism —
trigger → conditions → actions, either event-driven or run on a schedule — and a play
is a packaged workflow built around a signal, which is how most customers actually use
it. Both decompose the same way, so migrate either through this file. Apollo
**Sequences** are not workflows; a sequence is Apollo's sequence, and a play that
enrolls someone into one is describing sequence membership.

This file describes **Apollo**. It deliberately doesn't say what each part maps to in
Amplemarket — `create_workflow` answers that against the live catalogue, and a mapping
table here would be a stale second opinion. See the division of labour section in
[SKILL.md](../SKILL.md).

## Where to read it

Workflows and plays live in `app.apollo.io`, but Apollo moves this area around and
renames it, so this reference deliberately doesn't hardcode the URL — ask the user
for the link to their workflows or plays list, and for the URL of each one in the
migration set. Open each and read the trigger card, the filter set, and the action
list; the list view collapses filters, so it isn't enough on its own. A screenshot or
pasted text works as a fallback.

## Anatomy

1. **Trigger** — a signal, an engagement event, or a schedule
2. **Target** — people or accounts
3. **Filters** — person and company criteria that gate it
4. **Actions** — ordered, run when the filters match
5. **Whether a person can enter more than once**

## Triggers: signal vs engagement

The split that decides most of an Apollo migration is **signal triggers versus
engagement triggers**, and it's a fact about Apollo rather than about the target.

**Engagement triggers** fire on things that happened in the outreach itself — a reply,
a bounce, a call logged, a meeting booked, a LinkedIn connection accepted, someone
added to or finishing a sequence, a CRM record changing, an inbound webhook. These are
ordinary automation events; describe them plainly.

Two are narrower than they read: Apollo's meeting triggers cover booking, rescheduling
*and* cancellation, and its email triggers include opens and clicks. Capture which of
those the play actually uses instead of collapsing them to "meeting" or "email".

**Signal triggers** are the ones people build Apollo plays for: job change, funding
round, hiring and job postings, headcount growth, technology added, website visitor
identification, buying-intent score changes, news mentions, and the deal-stall triggers
behind Deal Plays.

Here's the thing worth understanding about them: **the detection is Apollo's product,
not the automation's logic.** Apollo is continuously scanning its own dataset and
firing inside its own detection window. The play is just what happens afterwards. So a
signal play isn't really "a trigger plus actions" — it's "Apollo's data pipeline plus
actions", and only the second half is an automation at all.

That shapes the migration: the actions usually move, and the detection has to be
rebuilt as a search or a periodically refreshed list that someone owns refreshing.
Propose that split explicitly rather than presenting it as an equivalent, because the
timing genuinely changes — Apollo fires when it notices, a refreshed list fires when
it's refreshed.

**Scheduled runs** ("run this workflow every Monday") are time-driven rather than
event-driven. Capture the schedule and flag it; a workflow waiting for an event is a
different mechanism.

## Target and re-entry

Person-level plays act on the contact; plays acting on everyone at an account act on
the company's contacts. Apollo plays typically re-run whenever the signal recurs, so
default to describing them as re-triggering — and confirm it, because it changes how
many contacts get enrolled.

## Filters

This is the easy half of an Apollo migration. Apollo's person and company filters —
title, seniority, department, location, industry, headcount, revenue, technologies,
keywords — are ordinary audience criteria. State them in the prompt in plain English
and let the planner resolve field names.

**Apollo-native filter values** need capturing carefully, because they're computed by
Apollo and don't exist outside it:

| Filter | What it actually is |
| --- | --- |
| **Buying-intent score** | Apollo's own intent model over its data. Often the entire point of the play. |
| **Apollo score** | Apollo's fit ranking. |
| **Contact stage** | An Apollo-configured pipeline label, same shape as Salesloft's Person Stage. |
| **Email status** | Apollo's deliverability verdict on the address. |
| **List membership** | Which Apollo list the person is on. |
| **Owner** | The assigned Apollo user. |

Buying intent is the one that bites — a play whose filters rest entirely on it doesn't
have an audience definition that travels, and the migrated version will enroll a much
wider group. That's a loss-report line, and wider means more contacts.

## Actions

| Apollo action | What it does |
| --- | --- |
| Add to sequence · Remove from sequence · Mark sequence finished | sequence membership |
| Add to list · Remove from list | list membership |
| Mark do-not-contact / opt out | suppresses future outreach |
| Enrich email · Enrich phone | reveals contact data |
| Update a CRM field / push to CRM / change CRM stage | writes to the connected CRM |
| Create a task | optionally assigned to a specific rep |
| Send a webhook / HTTP request | posts to an external URL |
| Send email or Slack notification | alerts a person or channel |
| Assign or route to a rep / change owner | ownership |
| Convert to an opportunity, create a deal | creates CRM records |
| AI-drafted email actions | Apollo composes the message |

Two notes on capture. **Notification actions** end nearly every Apollo play, and their
absence is the change users notice on day one — capture the channel and what the
message says, because a Slack alert can sometimes be rebuilt as an HTTP request to an
incoming webhook, and that's the user's decision to make with real information.
**Record-creation and ownership actions** create or re-own CRM records; workflows that
enroll contacts have nothing to act on there, so it's a structural mismatch rather
than a missing feature.

## The loss report for an Apollo migration

- **The signal's timing.** The single biggest one. "Real-time when they change jobs"
  quietly becomes "whenever someone refreshes the search". Name the cadence and name
  who owns it.
- **Buying intent and Apollo score.** If they gated the play, the rebuilt audience is
  broader — sometimes dramatically. Quantify it if the user can.
- **The Slack alert.** Say explicitly that nobody gets pinged any more, and offer the
  webhook rebuild.
- **Rep assignment on tasks.** A task that used to land on a named rep and now doesn't
  is an operational change.
- **Contact stage.** Same as Salesloft's Person Stage — if it fed reporting or list
  filters inside Apollo, that reporting needs another home.

## Worked example

Play **"Champion job change"** — trigger `Job change` over a saved list of past
customers; filters `Title contains "VP"` and `Company headcount 200-5000`; actions
`Enrich email`, `Add to list`, `Add to sequence "Champion re-engagement"`, and
`Send Slack notification`.

The trigger is a signal, so the play splits: the detection half becomes a refreshed
search feeding a lead list, and the automation half watches that list.

> When a contact is added to the "Champion job changes" lead list, enrich their data
> and add them to the "Champion re-engagement" sequence. Only apply this to contacts
> whose job title contains "VP" and whose company has between 200 and 5000 employees.
> Enroll each contact once.

**What you lose by moving this**

- Apollo's job-change detection — Apollo fires inside its own detection window, a
  refreshed search fires when someone refreshes it — agree the cadence and who owns
  it, because "real-time" is quietly becoming "weekly".
- Slack post to #sales-wins — nothing in the migrated workflow notifies anyone — the
  alert the team actually watches disappears on day one; rebuild it as an HTTP request
  to an incoming webhook, or tell the channel it's going away.

## Sources

- [Workflows Overview](https://knowledge.apollo.io/hc/en-us/articles/14296116597901-Workflows-Overview) and [Create a Workflow](https://knowledge.apollo.io/hc/en-us/articles/4413804036109-Create-a-Workflow) — trigger → conditions → actions, event-driven or scheduled
- [Engage decision makers who just changed jobs](https://www.apollo.io/academy/plays/engage-decision-makers-who-just-changed-jobs) — a signal play's trigger, filters, and action list
- [Apollo release notes](https://knowledge.apollo.io/hc/en-us/articles/27365542998925-Release-Notes-2024) — meeting triggers (booked, rescheduled, canceled) and Deal Plays
- [Manage Sequence Rulesets](https://knowledge.apollo.io/hc/en-us/articles/4409396858509-Manage-Sequence-Rulesets) — sequence-level automation, distinct from plays

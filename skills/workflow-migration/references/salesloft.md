# Salesloft

Salesloft's equivalent of a workflow is an **Automation Rule** (Settings →
Automation Rules, listed in some accounts under Settings → Data → Automation Rules):
a trigger, criteria, and an action. Cadences are *not* automation rules — a cadence is
Salesloft's sequence, and a rule that adds or removes someone from one is describing
sequence membership.

This file describes **Salesloft**. It deliberately doesn't say what each part maps to
in Amplemarket — `create_workflow` answers that against the live catalogue, and a
mapping table here would be a stale second opinion. See the division of labour section
in [SKILL.md](../SKILL.md).

## Where to read it

Rules live in `app.salesloft.com` under Settings → Automation Rules; the list shows
each rule's trigger and action inline, and opening one shows its criteria. This
reference deliberately doesn't hardcode the URL — ask the user for the link to their
Automation Rules list, and for the URL of each rule in the migration set, rather than
guessing at a path. A screenshot or pasted text is fine as a fallback.

Salesloft revises this roster, and which triggers an account sees depends on its CRM
setup. Capture whatever is in front of you rather than expecting it to match the lists
below.

## Anatomy

1. **Trigger** — the event that fires the rule
2. **Criteria** — the conditions that narrow when it fires
3. **Action** — what it does
4. **On/off** — rules are toggled live after saving, so a captured rule may not be
   running; check before assuming it's in production

## Triggers

Capture the trigger verbatim and describe it in the prompt in plain English.

**Engagement triggers** — a Person replies to an email; an email link is clicked by a
Person; a call is logged; a meeting is booked; a Person completes the first step of a
cadence; a Person finishes a cadence; a Success is recorded for a Person.

Two of these mean something narrower than they read:

- **"A Person completes the first step of a cadence"** fires on the *step completing*,
  not on the cadence starting. If the rule's intent depends on that distinction, say so
  in the prompt rather than flattening it to "when a sequence starts".
- **"A Person finishes a cadence"** fires whether or not they replied. If the target
  can only express one of those halves, the other half's volume goes somewhere in the
  loss report — it's a real change in how many people the automation touches.

**"Success"** is a Salesloft-native concept: a rep-marked outcome on a cadence,
configured per team. It isn't a meeting, a reply, or a deal. Describe what the team
uses it *for* rather than the word itself, because the word won't travel.

**CRM-sync triggers** — a Person is not found in Salesforce; a Lead is not found in
Salesloft; a Contact is not found in Salesloft; a Lead is converted to a Contact in
Salesforce; a Person changes in Salesloft.

The first four are housekeeping that keeps Salesloft and Salesforce in step. They
aren't outreach automation, and a rule built on one usually shouldn't be migrated at
all — recommend leaving it in Salesloft or rebuilding it in the CRM. Say that at scope
time (step 3) so the user drops it before a generation is spent on it.

"A Person changes in Salesloft" is the ambiguous one: it fires on *any* watched field
changing. Find out which field the rule actually cares about and whether that field is
CRM-synced, because "a person record changed" and "this specific synced field changed"
are very different automations.

## Target and re-firing

Salesloft rules act on a person. The one account-level action ("Remove Company from
all cadences") acts on the company instead.

Rules re-fire whenever the trigger recurs — Salesloft has no once-per-person
checkbox the way Outreach does. Default to describing the rule as re-triggering every
time, and confirm it with the user, since it changes how many contacts get enrolled.

## Criteria

Person and account attribute criteria — title, seniority, location, industry, size —
are ordinary audience conditions. State them in the prompt as audience filters in
plain English and let the planner resolve field names.

**Salesloft-native criteria are the ones to look at carefully**, because they show up
in most real rules and they carry meaning that lives inside Salesloft:

| Criterion | What it actually is |
| --- | --- |
| **Person Stage** | A Salesloft-configured pipeline label on the person. Teams build reporting, cadence entry rules, and list filters on it. It is **not** a Salesforce field, so it must never reach a prompt by that name — resolve it to the account's own CRM field first. |
| **Call disposition and sentiment** | The rep-selected outcome of a logged call, from a per-account configured list. Quote the disposition label verbatim; a matching one may well exist on the other side. |
| **Cadence name or state** | Which cadence someone is on, or how far through it. |
| **Tags** | Free-form labels on people and accounts. |
| **Owner** | The assigned Salesloft user. |
| **Last-touch recency** | How long since the person was contacted. |

For each of these, capture the configured *value*, not just the field name. A
disposition called "Connected - Bad Account Fit" is a concrete label that can be
matched; "a disposition criterion" is not.

Criteria reading a CRM-synced field are worth flagging as such in the capture — they
behave differently from Salesloft-native ones and they need the CRM integration on the
other side.

## Actions

The documented action set is eight:

**Cadence actions** — Add Person to a cadence; Remove Person from all cadences; Remove
Company from all cadences.

**Field write** — Set Person Fields in Salesloft. Capture *which* field and *which
value*, and then answer the question that matters: **what does this field correspond to
on the other side?** A Salesloft-native field that goes nowhere and a field that drives
`Lead_Status__c` in Salesforce are the same UI control and completely different
migrations.

Don't put that question to the user first — they often don't know, and the account's own
workflows do. Harvest the CRM names with `get_workflow` before asking anything, per
[CRM fields are the one thing to resolve first](../SKILL.md#crm-fields-are-the-one-thing-to-resolve-first).
A Salesloft stage label routinely lands on **two** Salesforce picklists, a status and a
separate reason. Save the user question for what the harvest genuinely can't settle —
typically which of two plausible fields the team considers workflow-writable.

**CRM-sync actions** — Create a Contact in Salesforce; Create a Lead in Salesforce;
Create a Person in Salesloft; Set Salesloft Owner to Salesforce Owner. These create or
re-own records. Amplemarket workflows enroll existing contacts rather than creating
records, so a rule whose purpose is record creation is a structural mismatch, not a
missing feature. Flag it at scope time.

## The loss report for a Salesloft migration

Salesloft's specific failure mode is that **Person Stage is load-bearing**. It's the
field most rules write, and teams build reporting, cadence entry rules, and saved
filters on top of it. Two cases, and they need different reports:

- **The stage is Salesloft-only.** Nothing outside Salesloft can write it, so
  switching the rule off means the stage stops advancing. Every dashboard and cadence
  rule reading it goes stale. Name them if the user can tell you which exist.
- **The stage syncs onward to the CRM.** A workflow can write the CRM field directly,
  which looks lossless — but it's writing the *end* of the chain. The Salesloft stage
  itself still stops being set, and anything reading it *in Salesloft* still breaks.
  This one gets reported as a loss even though the migration succeeded.

The other recurring one: a rule gated on **call disposition** or **cadence
membership** enrolls exactly the people the rep's selection picked out. If a
condition doesn't survive the move, the automation fires on more people than it used
to, which is the direction that causes damage.

## Worked example

Rule **"Reply pauses everything"** — trigger `A Person replies to an email`,
criterion `Person Title contains "Director"`, action `Remove Person from all cadences`.

> When a contact replies to an email, remove them from their active sequences. Only
> apply this to contacts whose job title contains "Director". Re-trigger every time.

**What you lose by moving this:** nothing — the rule's full effect is reproduced. It
happens to use an engagement trigger, an ordinary audience condition, and a cadence
action, with no Salesloft-native state involved. Had it also set Person Stage, that
would be the whole story instead.

## Sources

- [List of Automation Rule Triggers](https://help.salesloft.com/s/article/List-of-Automation-Rule-Triggers) — the trigger roster and the eight actions
- [Automation Rule Criteria](https://salesloft.my.site.com/Support/s/article/Automation-Rule-Criteria?language=en_US) — how criteria narrow a rule
- [Using automation to move people between cadences](https://champions.salesloft.com/create-automation-65/using-automation-to-move-people-between-cadences-400) — engagement triggers and the add/remove-cadence actions in practice
- [Salesloft automation rules for G2 intent](https://documentation.g2.com/docs/salesloft) — Settings → Data → Automation Rules, person-update trigger, add-to-cadence action

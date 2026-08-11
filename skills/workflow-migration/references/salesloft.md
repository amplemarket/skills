# Salesloft → Amplemarket workflows

Salesloft's equivalent of a workflow is an **Automation Rule** (Settings →
Automation Rules, listed in some accounts under Settings → Data → Automation Rules):
a trigger, criteria, and an action. Cadences are *not* workflows — a cadence maps to
an Amplemarket sequence, and a rule that references one maps to `add_to_sequence` or
`remove_from_sequence` against that sequence.

The important thing to know before starting: Salesloft automation rules are much
narrower than they look. Most of the documented roster exists to keep Salesforce and
Salesloft in sync, and the only actions that touch outreach are the three cadence
ones plus a person-field write. A rule doing anything else usually doesn't migrate.

## Where to read it

Rules live in `app.salesloft.com` under Settings → Automation Rules; the list shows
each rule's trigger and action inline, and opening one shows its criteria. This
reference deliberately doesn't hardcode the URL — ask the user for the link to their
Automation Rules list, and for the URL of each rule in the migration set, rather than
guessing at a path. A screenshot or pasted text is fine as a fallback.

Salesloft revises this roster, and which triggers an account sees depends on its CRM
setup. If the rule in front of you uses something not in the tables below, map it by
meaning and say in the gap list that the mapping is an inference rather than a known
equivalence.

## Anatomy

1. **Trigger** — the event that fires the rule
2. **Criteria** — the conditions that narrow when it fires
3. **Action** — what it does
4. **On/off** — rules are toggled live after saving, so a captured rule may not be
   running; check before assuming it's in production

## Trigger

| Salesloft trigger | Amplemarket trigger |
| --- | --- |
| A Person replies to an email | `event_email_replied` |
| An email link is clicked by a Person | No equivalent — Amplemarket has no click trigger |
| A call is logged | `event_call_logged` |
| A meeting is booked | `event_meeting_booked` |
| A Person completes the first step of a cadence | `event_sequence_started` (approximate — Amplemarket fires on the sequence starting, not on the first step completing) |
| A Person finishes a cadence | `event_sequence_completed_with_no_reply`, but only for the no-reply half; see below |
| A Success is recorded for a Person | No equivalent — "success" is a Salesloft-native concept |
| A Person changes in Salesloft | No equivalent as-is; if the changing field is CRM-synced, `event_crm_contact_updated` is the closest, CRM-gated |
| A Person is not found in Salesforce · A Lead is not found in Salesloft · A Contact is not found in Salesloft · A Lead is converted to a Contact in Salesforce | No equivalent — these are CRM-sync housekeeping, not outreach automation. A rule built on one of them usually shouldn't be migrated at all; say so rather than finding it a home |

"A Person finishes a cadence" fires regardless of whether they replied, and
Amplemarket has no trigger for a sequence finishing *after* a reply. Half that
volume has nowhere to go — record it as a gap instead of mapping the whole rule to
the no-reply event.

## Target and re-firing

Salesloft rules act on a person, so scope `contact` is the default. The one
account-level action ("Remove Company from all cadences") implies `account_contacts`.
Rules re-fire whenever the trigger recurs, so default to reenroll `always` unless the
rule is explicitly one-shot — and confirm that with the user, since it changes how
many contacts get enrolled.

## Criteria

Person and account attribute criteria (title, seniority, location, industry, size)
map to **searcher** filters using the field table in SKILL.md. Criteria reading a
CRM-synced field map to **crm** filters and need the CRM integration.

Salesloft-native criteria are the common blocker, and they show up in most real rules:
**Person Stage**, **call disposition and sentiment**, cadence name or state, tags,
owner, and last-touch recency all have no Amplemarket equivalent. Call disposition is
worth calling out specifically — "add to a follow-up cadence when a call is logged
*with disposition X*" is a standard Salesloft rule, and Amplemarket can express the
"call logged" half but not the disposition half, so the migrated workflow enrolls
every logged call. That's more contacts, not fewer, and the user needs to know before
it goes live.

## Actions

The documented action set is eight, and only four of them mean anything outside CRM
sync:

| Salesloft action | Amplemarket action |
| --- | --- |
| Add Person to a cadence | `add_to_sequence` |
| Remove Person from all cadences | `remove_from_sequence` |
| Remove Company from all cadences | `remove_from_sequence` at scope `account_contacts` |
| Set Person Fields in Salesloft | `update_crm_fields` only if the field is CRM-synced and writable; a Salesloft-native person field has no equivalent |
| Create a Contact in Salesforce · Create a Lead in Salesforce · Create a Person in Salesloft | No equivalent — Amplemarket workflows don't create records |
| Set Salesloft Owner to Salesforce Owner | No equivalent — no ownership concept in workflows |

Amplemarket can do things Salesloft rules can't — `complete_sequence`,
`add_to_lead_list`, `add_to_exclusion_list`, `enrich_data`, `create_one_off_task`,
`http_request`, delays, and condition branches. Mention them as available if the user
asks what else is possible, but don't add them to a migration the source didn't ask
for.

## No equivalent

- **Triggers:** email link clicked, success recorded, and every Salesforce-sync
  trigger (person/lead/contact not found, lead converted).
- **Criteria:** person stage, call disposition and sentiment, cadence membership,
  tags, owner.
- **Actions:** creating Salesforce or Salesloft records, setting owner, writing
  Salesloft-native person fields.

A rule whose entire purpose is CRM sync has no Amplemarket workflow form at all.
Recommend leaving it in Salesloft or rebuilding it in the CRM rather than producing a
workflow that only does the leftover half.

## Worked example

Rule **"Reply pauses everything"** — trigger `A Person replies to an email`,
criterion `Person Title contains "Director"`, action `Remove Person from all cadences`.

Trigger `event_email_replied`, scope `contact`, searcher filter `person_titles`
contains "Director", action `remove_from_sequence`:

> When a contact replies to an email, remove them from their active sequences. Only
> apply this to contacts whose job title contains "Director".

**Gaps:** none for this rule — it happens to use the one trigger and the one action
that both migrate cleanly. Had it also set Person Stage, that half would be dropped:
Amplemarket workflows have no stage action, and if stage is how the team reports on
engagement, that reporting needs another home before the rule is switched off.

## Sources

- [List of Automation Rule Triggers](https://help.salesloft.com/s/article/List-of-Automation-Rule-Triggers) — the CRM-sync-oriented trigger roster and the eight actions
- [Automation Rule Criteria](https://salesloft.my.site.com/Support/s/article/Automation-Rule-Criteria?language=en_US) — how criteria narrow a rule
- [Using automation to move people between cadences](https://champions.salesloft.com/create-automation-65/using-automation-to-move-people-between-cadences-400) — engagement triggers and the add/remove-cadence actions in practice
- [Salesloft automation rules for G2 intent](https://documentation.g2.com/docs/salesloft) — Settings → Data → Automation Rules, person-update trigger, add-to-cadence action

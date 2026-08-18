# Provider references

One file per source tool, read on demand by [SKILL.md](../SKILL.md) at step 1 of a
migration.

**These files describe the source tool, not Amplemarket.** They cover where the
automations live, how to decompose one, what each provider-native concept actually
means, and what stops working in the source tool once the automation is switched off.
They deliberately contain **no mapping tables onto Amplemarket triggers, actions, or
fields** — `create_workflow` resolves those against the live catalogue for the account,
and anything written here is a snapshot that goes stale. See the division of labour
section in SKILL.md for why this rule exists and what it cost when it was broken.

| Provider | What gets migrated | File |
| --- | --- | --- |
| Outreach | Triggers (Administration → Workflow automations) | [outreach.md](outreach.md) |
| Salesloft | Automation Rules (Settings → Automation Rules) | [salesloft.md](salesloft.md) |
| Apollo | Workflows and Plays | [apollo.md](apollo.md) |
| HubSpot | Workflows and Journeys (Automation → Workflows) | [hubspot.md](hubspot.md) |

Each file ends with the vendor documentation it was written from. When a provider ships
UI changes, re-read those sources before trusting a description.

## A provider with no reference file

Work from the source automation in front of you:

1. **Read the vendor's own documentation first.** Every file here was written from the
   vendor's docs rather than from memory, and doing that caught real errors — Outreach
   triggers fire on object-plus-change-type rather than on semantic events, and
   Salesloft's action list is far shorter than it looks. Guessing a roster produces
   descriptions that read fine and don't exist.
2. **Decompose it into the five parts every one of these tools has** — the event that
   fires it, what it acts on (person vs company), whether it can re-fire for the same
   target, the conditions that gate it, and the ordered actions it runs. If the source is
   a canvas rather than a form the actions may branch, and each root-to-end path is its
   own automation — step 3 of [SKILL.md](../SKILL.md) covers how that changes the scope
   table, and [hubspot.md](hubspot.md) shows what the branch types look like in practice.
3. **Describe each part in plain English**, capturing configured values verbatim.
   Don't decide what it maps to; that's `create_workflow`'s job, and it has the
   catalogue.
4. **Look for the parts that don't travel.** Two recurring shapes:
   - **Provider-computed values** — intent scores, fit rankings, stage labels, tags,
     owners. These are produced by the source tool's own data or configuration. Capture
     what they *mean* and what reads them, because that's the loss report.
   - **Record-shaped automations** — deal and opportunity flows, ticket automation,
     record creation, ownership routing. Amplemarket workflows enroll contacts, so
     these have no target to act on. That's a structural mismatch, and it's worth
     saying at scope time before a generation is spent on it.
5. **Write the reference file afterwards** if the provider is going to come up again.
   Follow the section order the existing files use so they stay comparable, cite the
   vendor pages you worked from, and describe only the source tool — if you find
   yourself writing an Amplemarket column, stop.

## Reading the source

Where the source is a browser app the agent can reach, read the automation directly
rather than asking the user to retype it — read-tier browser access is enough. A
screenshot or pasted text works as a fallback.

Each file's **Where to read it** section says whether the URLs are known. Only
Outreach's and HubSpot's are hardcoded here; for the others, ask the user for the link
to their automations list and for the URL of each automation to migrate. Pin every automation
in the set to exactly one URL before capturing it — a name is not an identifier, and
step 3 of the skill won't let capture start without one.

Whatever the source, extract condition values and action scopes verbatim, and record
only the *presence* of a credential, never its value.

## Keeping these honest

When a migration ends and the generated draft differs from what the reference implied,
that's signal. Update the file. The most valuable correction is deleting something —
if a reference file is asserting what Amplemarket can or can't do, it has drifted back
into the job it isn't supposed to have.

# Provider references

One file per source tool, read on demand by [SKILL.md](../SKILL.md) at step 1 of a
migration. Each file covers where the automations live, how to decompose one, and
how its parts map onto the Amplemarket vocabulary — that vocabulary itself lives in
SKILL.md and is not repeated here.

| Provider | What gets migrated | File |
| --- | --- | --- |
| Outreach | Triggers (Administration → Workflow automations) | [outreach.md](outreach.md) |
| Salesloft | Automation Rules (Settings → Automation Rules) | [salesloft.md](salesloft.md) |
| Apollo | Workflows and Plays | [apollo.md](apollo.md) |

Each file ends with the vendor documentation it was written from. When a provider
ships UI changes, re-read those sources before trusting a mapping — the tables record
what the vendor documents, and vendors move faster than this folder.

## A provider with no reference file

Don't guess at a mapping table. Work from the source automation in front of you:

1. **Read the vendor's own documentation first.** Every file here was written from the
   vendor's docs rather than from memory, and doing that caught real errors — Outreach
   triggers fire on object-plus-change-type rather than on semantic events, and
   Salesloft's action list is far shorter than it looks. Guessing a roster produces
   mappings that read fine and don't exist.
2. **Decompose it into the five parts every one of these tools has** — the event that
   fires it, what it acts on (person vs company), whether it can re-fire for the same
   target, the conditions that gate it, and the ordered actions it runs.
3. **Map each part** onto the vocabulary in SKILL.md, semantically. An event whose
   meaning doesn't match any Amplemarket trigger is a gap, not a near-miss to round to.
4. **Treat anything that touches deals, opportunities, tickets, owners, tags, or
   notifications as a gap by default** — those recur in every tool and none of them
   have an Amplemarket workflow equivalent. Same for record-shaped automations
   (HubSpot deal workflows, Salesforce Opportunity flows): Amplemarket workflows
   enroll contacts, so only person- and company-level automations map at all.
5. **Write the reference file afterwards** if the provider is going to come up again.
   Follow the section order the existing files use so they stay comparable, cite the
   vendor pages you worked from, and only record mappings those pages support.

## Reading the source

Where the source is a browser app the agent can reach, read the automation directly
rather than asking the user to retype it — read-tier browser access is enough. A
screenshot or pasted text works as a fallback.

Each file's **Where to read it** section says whether the URLs are known. Only
Outreach's are hardcoded here; for the others, ask the user for the link to their
automations list and for the URL of each automation to migrate. Pin every automation
in the set to exactly one URL before capturing it — a name is not an identifier, and
step 3 of the skill won't let capture start without one.

Whatever the source, extract condition values and action scopes verbatim, and record
only the *presence* of a credential, never its value.

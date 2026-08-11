---
name: workflow-migration
description: >
  Create an Amplemarket workflow from natural language, or migrate an existing automation from Outreach, Salesloft, or Apollo into an Amplemarket workflow draft. Use when asked to build a workflow, port a trigger/rule/play into Amplemarket, inspect what an existing workflow does, or judge whether an automation is expressible in Amplemarket.
metadata:
  author: amplemarket
  version: "2.0.0"
  category: "Workflows"
compatibility: Requires the Amplemarket MCP server; migrations also need a browser tool that can read authenticated pages
---

# Workflow Migration

Turn a request — a sentence from the user, or an automation that already runs in
another tool — into an Amplemarket workflow draft, then report honestly on what
did and didn't carry over.

Both entry paths converge on the same call: everything ends as one natural-language
prompt handed to `create_workflow`, which runs the Amplemarket AI Workflows agent
and leaves a draft in the dashboard for a human to review and activate.

## Prerequisites

Check these before doing anything else, and say which one is missing rather than
working around it silently.

| Requirement | Needed for | If it's missing |
| --- | --- | --- |
| **Amplemarket MCP** with the workflow tools | Everything | If `create_workflow` isn't in the tool list, try `get_more_tools`; if it's still absent, tell the user to enable the Amplemarket connector. Nothing in this skill works without it. |
| An **admin** Amplemarket user with the `ai_assisted_workflows` rollout | Creating | Reading workflows still works. Creation returns a specific error — surface it and stop; it needs an admin or an enablement, not a retry. |
| A **browser tool that can read authenticated pages** (Claude in Chrome: `read_page`, `screenshot`) | Migration only | Provider admin pages sit behind a login, so `WebFetch` and `WebSearch` can't reach them. Fall back to screenshots or pasted text from the user. |
| An **active provider login** in that browser | Migration only | Ask the user to log in themselves and say when they're done. Never attempt to authenticate, and never ask for credentials. |

Read-tier browser access is enough — you need to read pages, not click or type.

## Instructions

### Tools

Tool names below are the bare MCP names. Depending on how the server is connected
they may be exposed prefixed, e.g. `mcp__claude_ai_Amplemarket__create_workflow`.

| Tool | Use it for |
| --- | --- |
| `list_workflows` | Find existing workflows before building. Filter by `status`, `name`, `created_by_user_email`; 20 per page with `cursor` pagination. |
| `get_workflow` | Read one workflow's triggers, enrollment filters, and steps in execution order. Takes the `id` from `list_workflows`. |
| `create_workflow` | Start creation from a `prompt`. Returns an ID immediately; generation is asynchronous. |
| `get_workflow_creation` | Poll that ID until the request reaches a terminal state. |

`list_sequences` and `list_lead_lists` are useful alongside them for resolving
sequence and list names the workflow will reference.

### Steps

1. **Pick the path.** If the user is describing what they want in their own words,
   go to step 5. If they're pointing at an automation that already exists somewhere
   else, read the matching file in `references/` first —
   [outreach.md](references/outreach.md), [salesloft.md](references/salesloft.md),
   [apollo.md](references/apollo.md). For a provider with no reference file, follow
   [references/README.md](references/README.md).

2. **Confirm access to the source** (migration only), in this order. Stop at the
   first one that works and tell the user which one you're on:

    1. **Browser available?** If no browser tool is connected, skip to (4).
    2. **Logged in?** Read the provider's automations list — the reference file gives
       the URL where it's known. A login, SSO, or workspace-picker page means you're
       not logged in: ask the user to log in in their browser and tell you when to
       retry. Don't try to authenticate, and don't ask for credentials.
    3. **Found the right automations?** If the list loads but you can't tell which
       entries the user means — the names are ambiguous, there are dozens, or the
       provider's automation area isn't where the reference says — **ask the user for
       the URLs of the specific automations to migrate**. Don't guess, and don't
       migrate the whole list because it was easier than asking.
    4. **Fall back to what the user can give you.** Screenshots or pasted text work
       fine; the reference names exactly which parts you need. Say that you're
       working from a paste, since anything not in it is invisible to you.

3. **Agree what to migrate** (migration only). Never start capturing until the set
   is explicit and every automation in it is pinned to a URL. Two cases:

    - **"Migrate everything."** Enumerate the provider's automations list first and
      present the scope table below. Get a yes on the set before touching any of
      them, and flag anything you can already tell won't migrate (a deal-stage
      trigger, a signal trigger) so the user can drop it now rather than after a
      wasted generation.
    - **Specific automations.** Resolve each one the user named to exactly one URL.
      If a name matches several entries, matches none, or the list isn't reachable,
      **ask the user for the URL** — a name alone is not a match.

    **Migration scope — [provider]**

    | # | Automation | URL | Decision |
    | --- | --- | --- | --- |
    | 1 | [name as the provider shows it] | [direct URL] | Migrate |
    | 2 | [name] | [URL] | Skip — [reason] |

    Migrate one at a time, in the order of that table, running steps 4-11 per
    automation. One source automation is one `create_workflow` call, and each call
    counts against the daily limit — so if the set is bigger than what's left today,
    say so upfront and agree where to stop rather than discovering it mid-run.

4. **Capture the source** (migration only). Each reference names the parts to
   extract. Take values verbatim, including condition values and action scopes. If
   the source exposes a credential — an API key, a bearer token, a signed URL —
   record only that authentication is present. Never transcribe the value.

5. **Check what already exists.** Call `list_workflows` with a `name` filter on a
   distinctive word from the request. If something close is already there, call
   `get_workflow` on it and ask the user whether they want a new workflow or an
   edit to that one — this skill can only create, so an edit is a dashboard task.
   An existing workflow is also the best evidence of how this account phrases things.

6. **Map to Amplemarket vocabulary.** The vocabulary section below is the target
   language; the provider reference gives the mapping. Where a source concept has
   no clean target, do **not** substitute something adjacent — record it as a gap.
   Silently replacing an unsupported action with `create_one_off_task` is the
   failure mode to avoid; the planner is instructed to refuse that substitution
   anyway, so it just wastes a generation.

7. **Write the prompt.** One short paragraph of plain English, phrased as a user
   would phrase it, using the mapped concepts and naming concrete resources
   (sequence names, list names) exactly as the user names them:

   > When `<trigger event>` for a `<contact | account's contacts>`, `<action(s)>`.
   > Only apply this to `<contacts/accounts matching the conditions>`.
   > `<Enroll each contact once | re-trigger every time>`.

   Rules:
   - One source automation → one prompt → one workflow. Migrate them individually.
   - State conditions as audience filters ("whose title contains VP", "whose company
     is in Technology"), not as raw `field=value` rows.
   - Chain multiple actions in execution order.
   - Leave editor-fillable choices (exact mailbox, unnamed sequence) unstated — the
     planner defers those rather than guessing.
   - Keep unsupported parts *out* of the prompt. They go in the gap list.
   - Never include credential values, and never invent URLs, HTTP methods, JSON
     paths, field types, or captured-field names the source didn't provide.

8. **Present the plan and get a go-ahead.** Always this format, whether the source
   is a sentence or another tool — creation counts against a daily per-user limit
   and leaves a draft someone has to clean up, so nothing gets created unseen:

    **Workflow plan — [name]** · source: [URL from the scope table]

    | Part | Source | Amplemarket |
    | --- | --- | --- |
    | Trigger | [source event, verbatim] | `[trigger]` |
    | Scope | [source target] | `contact` / `account_contacts` |
    | Re-enrollment | [source frequency] | `never` / `always` |
    | Conditions | [source rows, verbatim] | `[fields]` |
    | Actions | [source actions, in order] | `[subtypes, in order]` |

    **Prompt**

    > [the paragraph from step 7, verbatim — this is what gets sent]

    **Gaps** — one line each, as `[source part] — [why] — [what happens instead]`,
    or the single line `None — every part of the source has an equivalent.` Never
    leave this section out; an empty gap list is itself the finding.

    Then ask whether to create it. For a fresh build with no source, drop the source
    URL and the Source column, and keep the rest. When migrating a set, one plan per
    automation — never a merged plan covering several, since each one becomes its own
    workflow and the user approves them one at a time.

9. **Create.** Call `create_workflow` with the approved `prompt`. It returns
   `{id, status: "planning", message}`. Keep that `id` — it's the creation-request
   ID, not a workflow ID.

10. **Poll.** Wait at least 30 seconds, then call `get_workflow_creation` with that
    `id`. While `status` is `planning` or `generating`, wait another 30 seconds and
    poll again. Say you're waiting rather than going silent.

11. **Report the result.** Read `status` first, then `outcome`. On `completed`, call
    `get_workflow` with `workflow_id` so the summary describes the draft that exists
    rather than the one you asked for. Report in this format:

    **Workflow created — [name]** (or **Workflow creation failed**)

    | Detail | Value |
    | --- | --- |
    | Status | `completed` |
    | Outcome | `doable` / `needs_configuration` / `impossible` |
    | Workflow | [name] (`[workflow_id]`) |
    | Link | [workflow_url] |
    | Source | [provider automation URL, migrations only] |
    | Trigger | `[trigger types from get_workflow]` |
    | Filters | [enrollment filters, in words] |
    | Steps | [numbered, in execution order, each as "action — what it does"] |
    | State | **Draft** — nobody is enrolled and nothing runs until it's activated in Amplemarket |

    **Before activating** — always present, numbered, one item per task. It holds
    every `affected_steps` entry rewritten as the concrete editor task it implies
    (configure authentication, send a test request, select captured fields), plus
    every gap carried forward from step 8 so the plan and the result say the same
    thing. If there's genuinely nothing, the one item is `Review the draft against
    your intent, then activate it.`

    On `failed`, keep the same table with Status, Outcome, and a Reason row holding
    `failure_reason` verbatim — it carries the planner's own account of what couldn't
    be built — and follow it with what to change and whether it's worth another
    attempt. On `canceled`, report it and offer to retry. Never describe a draft as
    live, and never round `needs_configuration` up to done.

    When a set was migrated, close with one **Migration summary** table repeating the
    scope table's rows plus an Outcome and a link per row, so the user has a single
    place that says what came across and what didn't.

### Important Notes

- **Nothing here activates a workflow.** Generated workflows are always drafts, and
  activation is a manual dashboard step by design. Don't imply otherwise.
- **Creation is gated three ways.** The user must be an account admin, must have the
  `ai_assisted_workflows` rollout, and must be under the daily generation limit. Each
  failure returns a specific error — surface it as-is instead of retrying.
- **`webhook_received` and `http_request` generate partially.** Keep the supported
  step in the prompt with every grounded public value (URL, method, payload shape,
  failure behavior); authentication, headers, the test request, and captured-field
  selection are editor-owned. Expect `needs_configuration`, not a failure. Never
  request captured-field mappings — those only exist after a real test in the editor.
- **CRM capabilities are account-gated.** `event_crm_*` triggers, `crm` filters, and
  `update_crm_fields` only work when the account has a HubSpot or Salesforce
  integration configured. Without one, the planner drops them into `affected_steps`.
- **`get_workflow` redacts on purpose.** Request headers and captured test response
  bodies are stripped; you get header *names* only. Don't try to route around that.

## Amplemarket workflow vocabulary

This is the target language for every migration. The account's actual catalogue can
be narrower than the full enum (CRM gating, rollout state), so treat this as what
*can* exist rather than a promise for a given account.

**Triggers** (`WorkflowTriggerType`) — `event_meeting_booked`, `event_email_replied`,
`event_email_bounced`, `event_linkedin_replied`,
`event_linkedin_connection_accepted`, `event_call_logged`,
`event_sequence_started`, `event_sequence_lead_added_to_sequence`,
`event_sequence_completed_with_no_reply`, `event_crm_contact_created`,
`event_crm_contact_updated`, `event_crm_account_created`,
`event_crm_account_updated`, `event_crm_lead_created`, `event_crm_lead_updated`,
`webhook_received`.

`event_sequence_completed_with_no_reply` only covers sequences that finish *without*
a reply. There is no trigger for a sequence finishing after a reply — a source rule
that fires on any "sequence finished" state loses that half of its volume, and that
belongs in the gap list rather than being quietly folded into the no-reply event.
`webhook_received` must be the workflow's only trigger.

**Actions** (`WorkflowActionSubtype`) — `add_to_sequence`, `remove_from_sequence`,
`complete_sequence`, `add_to_lead_list`, `remove_from_lead_list`,
`add_to_exclusion_list`, `update_crm_fields`, `enrich_data`, `create_one_off_task`,
`http_request`.

**Structure** — steps are `action`, `condition`, `delay`, or `exit` nodes.
`WorkflowTargetScope` is `contact` or `account_contacts` (every contact on the
account). `WorkflowReenroll` is `never` (each target enrolls once) or `always`.

**Filters** (`WorkflowFilterType`) — `searcher` (person/company attributes),
`crm` (CRM record fields, CRM-gated, operators `eq`, `not_eq`, `in`, `not_in`,
`contains`, `contains_any`, `is_empty`, `gt`, `gte`, `lt`, `lte`, …), `composite`,
`dynamic_fields`.

Common searcher fields, which most condition rows land on:

| Intent | Field |
| --- | --- |
| Job title | `person_titles` (`person_titles_exact_match` when the operator is "is") |
| Seniority | `person_seniorities` |
| Person location | `person_location` (object list; set the `country` key) |
| Recently changed company | `person_recently_changed_company` |
| Industry | `company_industry` |
| Headcount | `company_size` or `company_headcount` |
| Company location | `company_location` (object list; set the `country` key) |
| Revenue | `company_revenue_ranges` |
| Technologies | `company_technology` |
| Keywords | `company_keywords` |
| Funding | `company_last_funding_types`, `company_last_funding_amount`, `company_total_funding_amount` |
| Hiring signal | `company_has_recent_job_openings`, `company_recent_job_openings_seniorities` |

Don't invent field names. If you're unsure a field exists, describe the intent in
words in the prompt and let the planner resolve it.

These enums can drift. Prefer the vocabulary above, and when unsure whether a
field or trigger exists, describe the intent in the prompt and let the planner
resolve it rather than inventing names.

## Examples

### Example 1: Fresh build from a sentence

**User prompt:** "When someone replies to an email, take them out of all their sequences and add them to my 'Warm replies' list."

**What the skill does:**

1. Calls `list_workflows` with `name: "repl"` — nothing similar exists.
2. Calls `list_lead_lists` to confirm "Warm replies" is the exact list name.
3. Maps: trigger `event_email_replied`, scope `contact`, actions `complete_sequence`
   then `add_to_lead_list`.
4. Presents the plan in the step 8 format — no Source column, since there's no source
   — with this prompt and `Gaps — None`:

   > When a contact replies to an email, mark all of their active sequences as
   > finished and add them to the "Warm replies" lead list. Enroll each contact once.

5. Calls `create_workflow`, waits 30 seconds, polls `get_workflow_creation` until
   `status: "completed"`, `outcome: "doable"`.
6. Calls `get_workflow` on `workflow_id` and reports in the step 11 format: the
   trigger, the two steps, `workflow_url`, and the draft state.

### Example 2: Outreach trigger migration

**User prompt:** "Migrate our 'Stop All Sequences When Meeting is Booked' Outreach trigger."

**What the skill does:**

1. Reads [references/outreach.md](references/outreach.md), confirms the browser tool
   is connected, and reads `web.outreach.io/admin-exp/triggers` — the list loads, so
   the Outreach session is live.
2. Finds one trigger matching that name and pins it to
   `web.outreach.io/admin-exp/triggers/482/edit`. Had two triggers matched, or none,
   it would have asked the user for the URL instead of picking one.
3. Opens it and captures: Event `Meeting` / `Created or Updated` with a condition on
   the meeting's booked state, Target `Prospect`, "Trigger only once per target"
   checked, prospect condition `Title contains "VP"`, account condition
   `Industry is "Technology"`, action `Stop Sequences` scoped to "mark all sequences
   finished".
4. Maps it, then presents the plan.

**Example plan:**

**Workflow plan — Stop All Sequences When Meeting is Booked** · source: `web.outreach.io/admin-exp/triggers/482/edit`

| Part | Source | Amplemarket |
| --- | --- | --- |
| Trigger | Meeting / Created or Updated, condition on booked state | `event_meeting_booked` |
| Scope | Prospect | `contact` |
| Re-enrollment | Trigger only once per target | `never` |
| Conditions | Title contains "VP"; Industry is "Technology" | `person_titles`, `company_industry` |
| Actions | Stop Sequences (mark all finished) | `complete_sequence` |

**Prompt**

> When a meeting is booked with a contact, mark all of that contact's active sequences
> as finished. Only apply this to contacts whose job title contains "VP" and whose
> company is in the Technology industry. Enroll each contact only once.

**Gaps** — None — every part of the source has an equivalent.

On approval it calls `create_workflow`, polls to `completed` / `doable`, and reads the
draft back with `get_workflow`.

**Example result:**

**Workflow created — Stop All Sequences When Meeting is Booked**

| Detail | Value |
| --- | --- |
| Status | `completed` |
| Outcome | `doable` |
| Workflow | Stop All Sequences When Meeting is Booked (`wf_9dK2mQ`) |
| Link | `app.amplemarket.com/workflows/wf_9dK2mQ` |
| Source | `web.outreach.io/admin-exp/triggers/482/edit` |
| Trigger | `event_meeting_booked` |
| Filters | Job title contains "VP"; company industry is Technology |
| Steps | 1. `complete_sequence` — marks every active sequence for the contact as finished |
| State | **Draft** — nobody is enrolled and nothing runs until it's activated in Amplemarket |

**Before activating**

1. Review the draft against your intent, then activate it.

### Example 3: A migration with real gaps

**User prompt:** "Port our Apollo play that catches champion job changes — it enriches them, adds them to a list and a sequence, and pings #sales-wins."

**What the skill does:**

1. Works from the pasted play rather than the browser, and says so — anything not in
   the paste is invisible to it, including filters the user didn't quote.
2. Reads [references/apollo.md](references/apollo.md) and finds the trigger is the
   half that doesn't migrate: no Amplemarket workflow trigger fires on a job-change
   signal. The actions mostly do — `enrich_data`, `add_to_lead_list`,
   `add_to_sequence` — so the play splits rather than dying.
3. Presents a plan for a workflow triggered by list membership instead, with:

   **Gaps**

   - Job change trigger — Amplemarket has no signal triggers — the detection half becomes a search on `person_recently_changed_company` that feeds the lead list this workflow watches, refreshed on a cadence someone owns. Apollo fires inside its own detection window; this fires when the search is refreshed.
   - Slack post to #sales-wins — no notification action — dropped; rebuildable as an `http_request` to an incoming webhook if you want it, with the URL configured in the editor rather than in the prompt.

4. On the user's go-ahead, creates the reduced workflow and repeats both gaps under
   **Before activating**, so the result says the same thing the plan did.

## Troubleshooting

| Problem | Solution |
| --- | --- |
| Workflow tools aren't available | Try `get_more_tools`. If they're still missing, the Amplemarket connector isn't enabled for this client — nothing in this skill can create without them. |
| No browser tool is connected | Reading provider admin pages needs one — `WebFetch` and `WebSearch` can't get past the login. Ask the user for screenshots or the automation's configuration as text, and say you're working from a paste so they know what you can't see. |
| The provider page shows a login or SSO screen | The session isn't live. Ask the user to log in in their own browser and tell you when to retry. Never attempt to authenticate and never ask for credentials. |
| The automations list isn't where the reference says | Providers move these pages. Ask the user for the URL of the list, or of the specific automations to migrate, and note the drift so the reference can be corrected. |
| A named automation matches several entries, or none | Don't pick one. Show what you found and ask the user for the URL of the one they mean — every automation in the scope table has to be pinned to a URL before capture starts. |
| "Only account admins can create workflows." | The connected user isn't an admin. Someone with an admin role has to run the creation, or build it in the dashboard. |
| "AI-assisted workflows are not enabled for this user." | The `ai_assisted_workflows` rollout is off for that user. Don't retry; it needs enabling first. |
| Daily creation limit reached | The error names the limit and the reset time. Stop creating — batch the remaining migrations for after the reset rather than burning retries. |
| `get_workflow_creation` still says `generating` after several polls | Normal for complex requests. Keep 30-second gaps between polls and keep the user posted; don't start a second `create_workflow` for the same request. |
| `outcome: "impossible"` | The plan was rejected outright, and `failure_reason` carries the blocked steps' own explanation. Cut the request to its supported core and try once more, rather than rewording the same ask. |
| `outcome: "needs_configuration"` | Expected for webhook and HTTP steps. Report `affected_steps` and hand the remaining editor work to the user; don't treat it as a failure. |
| The generated draft doesn't match the intent | Read it with `get_workflow`, name the specific divergence, and create a new one from a sharper prompt. There's no edit tool — fixes are either a fresh generation or dashboard edits. |
| Source automation fires on an event with no trigger equivalent | Say so plainly in the gap list. Email opened/clicked, meeting rescheduled or canceled, opportunity/deal stage changes, task completion, and non-CRM field changes have no Amplemarket trigger. |
| Source automation fires on a signal (job change, funding, hiring, intent) | No Amplemarket workflow trigger covers signals, but the searcher has the same criteria as *filters*. The rebuild is a saved search or refreshed lead list feeding a sequence — a different mechanism with different timing. Propose it, don't pass it off as an equivalent. |
| A referenced sequence or list can't be found | Resolve names with `list_sequences` / `list_lead_lists` before creating. If it doesn't exist yet, create it first — the planner can't invent a resource that isn't there. |

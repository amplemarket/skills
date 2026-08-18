---
name: workflow-migration
description: >
  Create an Amplemarket workflow from natural language, or migrate an existing automation from Outreach, Salesloft, Apollo, or HubSpot into an Amplemarket workflow draft. Use when asked to build a workflow, port a trigger/rule/play into Amplemarket, inspect what an existing workflow does, or judge whether an automation is expressible in Amplemarket.
metadata:
  author: amplemarket
  version: "3.2.0"
  category: "Workflows"
compatibility: Requires the Amplemarket MCP server; migrations also need a browser tool that can read authenticated pages
---

# Workflow Migration

Turn a request — a sentence from the user, or an automation that already runs in
another tool — into an Amplemarket workflow draft, then report honestly on what the
user loses by moving it.

Both entry paths converge on the same call: everything ends as one natural-language
prompt handed to `create_workflow`, which runs the Amplemarket AI Workflows agent and
leaves a draft in the dashboard for a human to review and activate.

## Division of labour

Read this before anything else. It's the thing this skill most easily gets wrong.

**`create_workflow` owns the Amplemarket side.** The planner behind it has the live
catalogue for *this account* — which triggers exist, which actions exist, which
searcher and CRM fields are available, which sequences, lead lists, tags, and call
dispositions are configured, and what the account's rollout state gates. It knows all
of that better than any table written here, because a table here is a snapshot and the
catalogue is the thing itself.

So **do not pre-map the source onto Amplemarket trigger, action, or field names.**
Don't decide in advance that something "has no equivalent." Describe what the source
automation does, in plain English, faithfully, and let the planner answer whether it's
expressible. When it isn't, `failure_reason` and `affected_steps` say so in the
planner's own words — which is authoritative in a way a hand-maintained mapping table
never is.

This was learned the hard way: an earlier version of this skill asserted that call
disposition had no Amplemarket equivalent and warned that migrated call rules would
over-enroll. It was wrong — `event_call_logged` takes `phone_call_trigger_ids`, and
had for a long time. The mapping table didn't just fail to help, it talked the agent
out of a correct migration.

**This skill owns the source side**, which the planner cannot see at all:

- Where the automations live and how to read them
- What a provider-native concept actually *means*, so it can be described in English
- Which automations shouldn't be migrated in the first place (CRM-sync housekeeping,
  record-creation plumbing)
- **The loss report** — what stops working *in the source tool* once the automation is
  switched off. The planner never sees the Salesloft rule, so it can never tell the
  user that a field it no longer writes was feeding their reporting.

That last one is the real deliverable. A migration that produces a working draft and
no loss report is half-done.

## CRM fields are the one thing to resolve first

A field name the account's catalogue doesn't have costs you the field, not the draft: the
planner keeps the step, leaves the field unset, and hands the selection back as an editor
task. That's the floor, not the target — every unmatched field is one more thing the user
has to finish before the workflow can run. Source-native names rarely survive: Salesloft's
**Person Stage** isn't a Salesforce field, and the account using it wrote `Lead_Status__c`
and `Reason_for_Not_Accepting__c` instead.

So harvest before writing. `get_workflow` on several existing workflows gives you exact
`field_name`, `field_type`, and picklist values from their `update_crm_fields` steps and
`crm` filters — the account's own vocabulary. **Harvest only from workflows that aren't
archived**, for the reason below: a retired workflow's field names may have been retired
from the CRM with it. Match on what a field
*means*, not how it's spelled: names run a word or two off the source's, and one source
field often lands on two, a status plus a separate reason.

**When the source tool is itself the CRM, the harvest gets easier but doesn't go away.**
A HubSpot property *is* a CRM field, unlike Salesloft's Person Stage, so the source's
internal name (`hs_lead_status`, not the `Lead status` label) is a real candidate rather
than a vendor label — but only a candidate, since plenty of accounts run Salesforce with
HubSpot as a marketing front end. Harvest first, then use the source name if the
account's own workflows corroborate it.

This isn't pre-mapping. Triggers and actions stay in English for the planner to pick;
CRM field names are account data, and quoting one you read out of that account's own
workflows is evidence.

**When the harvest doesn't settle it, under-specify.** Name the concept and leave the
specifics out — "set the contact's lead status CRM field to the rejected value". That
returns `needs_configuration`: a draft with the trigger, filters, and step order built,
and the field selection waiting in `affected_steps` as an editor task. Describing the
concept and asserting a name that doesn't exist land in the same place, so describe it
and let the report say so plainly rather than shipping a name you invented.

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

### Concurrency

A set migration is mostly independent work, and the plain reading of the steps below —
one automation all the way through, then the next — spends most of its wall clock
waiting. The steps say where to batch; this is the policy behind them.

**Independent reads go out together.** `list_workflows` across several distinctive
words, `get_workflow` across the workflows you're harvesting, `list_sequences` and
`list_lead_lists` alongside them — none of these depend on each other, so they belong in
one message rather than one per turn.

**Creation stays with the orchestrator, always.** The daily generation limit is a single
shared budget, and a subagent can't see how much of it the others have spent. So
`create_workflow` and `get_workflow_creation` are never delegated — whatever else fans
out, the calls that spend the budget are made in one place that can count them.

**Where subagents earn their place**, when a subagent tool is available and the work is
bulky reading whose result you want as a summary rather than in context: vendor
documentation for a provider with no reference file, one agent per page; capture from
bulk pasted material — a long export, a folder of screenshots — one agent per
automation, each returning the capture in the shape step 4 asks for; and reading a large
existing workflow set, when the user asked what their workflows do rather than for a new
one. Give every capture agent the harvested CRM vocabulary up front, or each invents its
own phrasing for the same field and the prompts end up disagreeing with each other.

**What stays serial.** Anything driving a shared browser: two agents reading provider
pages through one browser fight over the same tab — one navigates while the other
screenshots, and the capture silently belongs to the wrong automation. Read provider
pages one at a time, from one place. That's the usual case for a migration, and it's why
capture is the one stage that generally doesn't parallelize. The other is the user's
approval — everything fans out after step 7's go-ahead, never before.

### Reading what already exists

Reading existing workflows with `list_workflows` / `get_workflow` is the one reliable
way to see what this account actually supports. Two `event_call_logged` drafts with
`phone_call_trigger_ids` populated is real evidence about dispositions; a table in a
reference file is not. Prefer the evidence.

**Ignore archived workflows everywhere in this skill.** An archived workflow is retained
for history and can't run, so it's evidence about what the account used to do, not what
it does — the sequences, lists, dispositions, and CRM fields in it may be long gone.
`list_workflows` takes a single `status`, so it can't filter a status *out*: leave
`status` unset and drop every row whose `status` is `archived` before you use the list
for anything — the duplicate check, the CRM harvest, or a summary shown to the user.
Only bring one up if the user asks about it by name, and say it's archived when you do.

### Steps

1. **Pick the path.** If the user is describing what they want in their own words,
   go to step 5 — steps 2-4 are migration-only. If they're pointing at an automation that already exists somewhere
   else, read the matching file in `references/` first —
   [outreach.md](references/outreach.md), [salesloft.md](references/salesloft.md),
   [apollo.md](references/apollo.md), [hubspot.md](references/hubspot.md). For a
   provider with no reference file, follow
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
      them, and flag anything the reference file says is *provider-side* housekeeping
      — a CRM-sync rule, a record-creation play — so the user can drop it now.
    - **Specific automations.** Resolve each one the user named to exactly one URL.
      If a name matches several entries, matches none, or the list isn't reachable,
      **ask the user for the URL** — a name alone is not a match.

    **One source automation isn't always one row.** Where the source is a branching
    canvas rather than a flat rule — HubSpot workflows and journeys, most obviously —
    each root-to-end path through the branches is its own
    trigger-plus-conditions-plus-actions automation, and becomes its own Amplemarket
    workflow with the branch conditions folded into its audience filters. Enumerate the
    paths at scope time and give each one a row sharing the source URL, so the user sees
    upfront that one workflow they maintain is becoming several, and so the count against
    the daily generation limit is honest. The provider's reference file says whether this
    applies.

    **Migration scope — [provider]**

    | # | Automation | URL | Decision |
    | --- | --- | --- | --- |
    | 1 | [name as the provider shows it] | [direct URL] | Migrate |
    | 2 | [name] | [URL] | Skip — [reason] |

    Run steps 4-10 across the whole table rather than one automation at a time — see
    [Concurrency](#concurrency) for what batches and what doesn't. One source automation
    is one `create_workflow` call, so if the set is bigger than what's left of the daily
    limit today, agree where to stop now: the batch fires before any of it reports back,
    so there's no mid-run moment to catch it.

4. **Capture the source** (migration only). Each reference names the parts to
   extract. Take values verbatim, including condition values and action scopes,
   because the prompt is going to quote them. If the source exposes a credential — an
   API key, a bearer token, a signed URL — record only that authentication is present.
   Never transcribe the value.

   Capture what the automation *does*, not what you think Amplemarket will call it.
   "Sets Person Stage to Rejected - Bad Account Fit, which syncs onward to
   `Lead_Status__c` in Salesforce" is a capture. "`update_crm_fields`" is a guess.

5. **Check what already exists.** Call `list_workflows` with a `name` filter on a
   distinctive word from the request, and **discard the archived rows before you read
   any of them** — an archived match isn't a duplicate, and offering to edit one sends
   the user somewhere dead. If
   something close is still live, call `get_workflow` on it and ask the user whether
   they want a new workflow or an edit to that one — this skill can only create, so an
   edit is a dashboard task. An existing workflow is also the best evidence of how this
   account phrases things, and of which trigger settings it actually has configured.

   If every match is archived, treat it as nothing existing and carry on, mentioning in
   one line that a retired workflow covered similar ground — that's context the user may
   want, but it isn't a reason to stop.

   **If the source automation writes a field, harvest CRM names here** — call
   `get_workflow` on several existing non-archived workflows, not just the closest
   match, and read every `update_crm_fields` step and `crm` filter for `field_name`,
   `field_type`, and the picklist values in use. Send those `get_workflow` calls in one
   message, and when migrating a set do this **once for the whole scope table** rather
   than per automation — it's the same account either way, and one harvest is what keeps
   every prompt in the set spelling the field the same way. This is the step that decides
   whether the generation succeeds; see [CRM fields are the one thing to resolve first](#crm-fields-are-the-one-thing-to-resolve-first).

6. **Write the prompt.** One short paragraph of plain English, phrased as a user
   would phrase it, naming concrete resources (sequence names, list names,
   dispositions, CRM fields) exactly as the source names them:

   > When `<trigger event, in words>` for a `<contact | everyone at the account>`,
   > `<action(s), in words>`. Only apply this to `<contacts/accounts matching the
   > conditions>`. `<Enroll each contact once | re-trigger every time>`.

   Rules:
   - One source automation → one prompt → one workflow. Migrate them individually.
   - **Describe, don't map.** Write "when a call is logged with the disposition
     'Connected - Bad Account Fit'", not "`event_call_logged` with
     `phone_call_trigger_ids`". The planner resolves names against the live catalogue;
     an enum you half-remember will be wrong more often than the English will.
   - State conditions as audience filters ("whose title contains VP", "whose company
     is in Technology"), not as raw `field=value` rows.
   - Chain multiple actions in execution order.
   - Quote provider-native labels verbatim and let the planner match them. If the
     account has that disposition, tag, or field, it will find it; if it doesn't, it
     defers the choice to the editor. Both outcomes beat you deciding.
   - **Never pass a source-native CRM field name through.** Use a name harvested from
     this account's own workflows, or describe the field by meaning and leave the
     selection to the editor. "Person Stage" in a prompt buys nothing the description
     wouldn't, and reads to the user as a field this account has.
   - Leave genuinely open choices (exact mailbox, unnamed sequence, an unresolved CRM
     field or picklist value) unstated — the planner defers those rather than guessing,
     and a deferred choice costs a line in **Before activating**, not a generation.
   - Keep out anything the source automation didn't ask for. Extra actions "while
     we're here" are new scope, not a migration.
   - Never include credential values, and never invent URLs, HTTP methods, JSON
     paths, field types, or captured-field names the source didn't provide.

7. **Present the plan and get a go-ahead.** Always this format, whether the source
   is a sentence or another tool — creation counts against a daily per-user limit
   and leaves a draft someone has to clean up, so nothing gets created unseen:

    **Workflow plan — [name]** · source: [URL from the scope table]

    | Part | Source, verbatim | How the prompt asks for it |
    | --- | --- | --- |
    | Trigger | [source event] | [the phrase used in the prompt] |
    | Scope | [source target] | one contact / everyone at the account |
    | Re-firing | [source frequency] | once per contact / every time |
    | Conditions | [source rows] | [the audience phrasing] |
    | Actions | [source actions, in order] | [the phrasing, in order] |

    The right-hand column is the English going into the prompt, **not** an enum
    mapping. Don't put trigger or action identifiers in it — what the planner picks is
    its call, and step 10 reports what it actually built.

    **Prompt**

    > [the paragraph from step 6, verbatim — this is what gets sent]

    **What you lose by moving this** — see the section below. Never leave it out; an
    empty list is itself a finding, and it's the part of the plan the planner can't
    produce for you.

    Then ask whether to create it. For a fresh build with no source, drop the source
    URL and the Source column, and skip the loss report — there's nothing being left
    behind. When migrating a set, one plan block per automation — never a merged plan
    covering several, since each one becomes its own workflow — but present the blocks
    together in one message and ask for a single go-ahead covering the batch. The user
    still reads and approves each plan; what they no longer do is wait for one generation
    to finish before seeing the next plan. If they approve some and not others, create
    exactly the approved ones.

8. **Create.** Call `create_workflow` with each approved `prompt`. Every call returns
   `{id, status: "planning", message}`. Keep those `id`s — each is a creation-request
   ID, not a workflow ID, and it's what step 9 polls.

   With a batch, send every approved prompt in one message rather than one per turn, and
   pair each returned `id` back to the automation it came from before doing anything
   else — the responses carry no name or source URL, and an unlabelled `id` is a report
   you can't write. If one call comes back with the daily-limit error, the rest of the
   batch may well have landed: say which ones did, and hold the others until the limit
   resets instead of retrying them now.

9. **Poll.** Wait at least 15 seconds, then call `get_workflow_creation` on every
   outstanding `id` in a single message. Drop every `id` that came back terminal, and
   while any is still `planning` or `generating`, wait another 15 seconds and poll
   whatever is left. Repeat until nothing is outstanding. Say what you're waiting on and
   how many are still going rather than going silent — a batch takes as long as its
   slowest member, not the sum of all of them.

10. **Report the result.** Read `status` first, then `outcome`. On `completed`, call
    `get_workflow` with `workflow_id` so the summary describes the draft that exists
    rather than the one you asked for. Batch those reads too, and give each workflow its
    own table. Report in this format:

    **Workflow created — [name]** (or **Workflow creation failed**)

    | Detail | Value |
    | --- | --- |
    | Status | `completed` |
    | Outcome | `doable` / `needs_configuration` / `impossible` |
    | Workflow | [name] (`[workflow_id]`) |
    | Link | [workflow_url] |
    | Source | [provider automation URL, migrations only] |
    | Trigger | [trigger types from `get_workflow`] |
    | Filters | [enrollment filters, in words] |
    | Steps | [numbered, in execution order, each as "action — what it does"] |
    | State | **Draft** — nobody is enrolled and nothing runs until it's activated in Amplemarket |

    This table reports what `get_workflow` returned. Here the enum names are fine —
    they're observed facts about a draft that exists, not predictions.

    Compare it against the plan and say so if the planner made a different call than
    the prompt implied. That divergence is information: it's usually the catalogue
    correcting an assumption, and it's how this skill's provider references get
    better.

    **Before activating** — always present, numbered, one item per task. It holds
    every `affected_steps` entry rewritten as the concrete editor task it implies
    (configure authentication, send a test request, select a call disposition, map a
    CRM field), plus every item from the loss report so the plan and the result say
    the same thing. If there's genuinely nothing, the one item is `Review the draft
    against your intent, then activate it.`

    On `failed`, keep the same table with Status, Outcome, and a Reason row holding
    `failure_reason` verbatim — it carries the planner's own account of what couldn't
    be built, which is the authoritative answer on what Amplemarket can't express —
    and follow it with what to change and whether it's worth another attempt. On
    `canceled`, report it and offer to retry. Never describe a draft as live, and
    never round `needs_configuration` up to done.

    When a set was migrated, close with one **Migration summary** table repeating the
    scope table's rows plus an Outcome and a link per row, so the user has a single
    place that says what came across and what didn't.

## Writing the loss report

This is the deliverable the planner cannot produce. It is **not** a list of things
Amplemarket lacks — that's the planner's job, and it answers it in `failure_reason`
and `affected_steps`. It's a list of what stops happening **in the source tool** when
this automation is switched off.

One line each, as `[source part] — [what breaks] — [what to do about it]`:

- **A field that stops being written.** The strongest example. A Salesloft rule
  setting Person Stage may be feeding dashboards, cadence entry rules, and filters
  that have nothing to do with the migrated workflow. Even when the equivalent write
  migrates perfectly, the *source* field goes stale. Name what reads it.
- **A sync chain that gets shorter.** When the source writes field A which syncs to
  field B, and the workflow writes B directly, anything keyed on A is now orphaned.
  The migration looks lossless and isn't.
- **Volume changes in either direction.** A condition the prompt couldn't express
  means more contacts enrolled; a trigger that only covers half the source's cases
  means fewer. Both matter, and "more" is usually the one that surprises people.
- **Timing changes.** A signal-driven play that becomes a refreshed-list workflow
  fires on a different clock, and somebody now owns that refresh.
- **Anything the source did that isn't outreach.** Creating CRM records, setting
  owners, posting notifications. If it mattered, it needs another home before the
  source automation is turned off.

If the source automation genuinely leaves nothing behind, the line is
`Nothing — the source automation's full effect is reproduced.` Say it explicitly.

## Facts about Amplemarket workflows worth knowing

Deliberately short. These are behaviours the planner won't volunteer and that don't
drift the way the catalogue does. Everything catalogue-shaped — which triggers,
actions, filters, and fields exist — is intentionally **not** listed here.

- **Nothing here activates a workflow.** Generated workflows are always drafts, and
  activation is a manual dashboard step by design. Don't imply otherwise.
- **Creation is gated three ways.** The user must be an account admin, must have the
  `ai_assisted_workflows` rollout, and must be under the daily generation limit. Each
  failure returns a specific error — surface it as-is instead of retrying.
- **Workflows enroll contacts.** Scope is one contact or everyone at an account.
  Automations shaped around deals, opportunities, or tickets don't have a target to
  enroll, which is a structural mismatch rather than a missing feature.
- **Webhook and HTTP steps generate partially.** Keep the supported step in the prompt
  with every grounded public value (URL, method, payload shape, failure behaviour);
  authentication, headers, the test request, and captured-field selection are
  editor-owned. Expect `needs_configuration`, not a failure. Never request
  captured-field mappings — those only exist after a real test in the editor.
- **A webhook trigger must be the workflow's only trigger.**
- **CRM capabilities are account-gated.** CRM triggers, CRM filters, and CRM field
  writes need a configured HubSpot or Salesforce integration. Without one the planner
  drops them into `affected_steps` — which is how you find out, rather than by
  checking first.
- **`get_workflow` redacts on purpose.** Request headers and captured test response
  bodies are stripped; you get header *names* only. Don't try to route around that.

The catalogue drifts, so treat the vocabulary above as a guide rather than a
contract. When you're unsure whether a field or trigger exists, describe the intent
in the prompt and let the planner resolve it rather than inventing names.

## Examples

### Example 1: Fresh build from a sentence

**User prompt:** "When someone replies to an email, take them out of all their sequences and add them to my 'Warm replies' list."

**What the skill does:**

1. Calls `list_workflows` with `name: "repl"` — one hit, and it's archived, so it's
   discarded: nothing live is similar.
2. Calls `list_lead_lists` to confirm "Warm replies" is the exact list name. This is
   the one kind of pre-checking that pays: the planner can't invent a list that
   doesn't exist.
3. Presents the plan in the step 7 format — no Source column and no loss report, since
   nothing is being left behind — with this prompt:

   > When a contact replies to an email, mark all of their active sequences as
   > finished and add them to the "Warm replies" lead list. Enroll each contact once.

4. Calls `create_workflow`, waits 15 seconds, polls `get_workflow_creation` until
   `status: "completed"`, `outcome: "doable"`.
5. Calls `get_workflow` on `workflow_id` and reports in the step 10 format: the
   trigger and steps the planner actually chose, `workflow_url`, and the draft state.

### Example 2: Salesloft rule where the loss report is the whole point

**User prompt:** "Migrate our 'Call Automation - Bad Account Fit' rule." *(screenshot)*

**Source, captured verbatim:** trigger `When a call is logged for a person`; criterion
`Disposition equal "Connected - Bad Account Fit"`; action `Set Person Fields` →
`Person Stage = "Rejected - Bad Account Fit"`. The user says nothing about whether
Person Stage syncs anywhere, and won't be asked — the account's workflows answer it.

**What the skill does:** reads [salesloft.md](references/salesloft.md), notes it's
working from a screenshot, and harvests. `list_workflows` returns five
`event_call_logged` workflows; one is archived and is dropped without being read, and
`get_workflow` on the remaining four yields the account's CRM vocabulary:

| Field | Used as | Values |
| --- | --- | --- |
| `Lead_Status__c` | written by all four | `Attempting to Contact`, `Recycled`, `BDR/Sales Rejected`, … |
| `Reason_for_Not_Accepting__c` | written | `Bad Account Fit`, `Bad Person Fit` |
| `Lead_Lifecycle_Stage__c` | read only, as a filter | `Customer`, `Opportunity`, `Rejected Lead` |

The one Salesloft field becomes two, and the field that reads most like "Person Stage"
is only ever *read* here — writing it would contend with whatever Salesforce automation
owns it, which is a question for the user, not a decision for the skill.

**Prompt:**

> When a call is logged for a contact with the disposition "Connected - Bad Account
> Fit", update that contact's CRM record to set `Lead_Status__c` to "BDR/Sales Rejected"
> and `Reason_for_Not_Accepting__c` to "Bad Account Fit". Enroll each contact once.

**What you lose by moving this**

- Salesloft Person Stage — the workflow writes `Lead_Status__c` directly, so the
  Salesloft stage field itself stops being set — anything in Salesloft reading Person
  Stage (reporting, cadence entry rules, filters) goes stale once the rule is off.
  Check what reads it before switching over.
- One field becomes two — the Salesloft stage encoded rejection and reason in a single
  label; the target splits them across `Lead_Status__c` and `Reason_for_Not_Accepting__c`
  — any report expecting the combined string needs rewriting.
- Re-firing volume — Salesloft re-fired on every matching call, the account's own
  convention is `reenroll: never` — a contact who gets a second bad-fit call is no
  longer re-stamped. Confirm which behaviour is wanted.

Note what the loss report is *not*: it says nothing about which Amplemarket action gets
used, and nothing about `Lead_Lifecycle_Stage__c`. Whether a workflow may write that
field is an ownership question for the user's RevOps, so it belongs in the plan as an
open question — not in the loss report, and not silently resolved by the skill.

### Example 3: A play whose trigger is the wrong shape

**User prompt:** "Port our Apollo play that catches champion job changes — it enriches them, adds them to a list and a sequence, and pings #sales-wins."

**What the skill does:**

1. Works from the pasted play rather than the browser, and says so — anything not in
   the paste is invisible to it, including filters the user didn't quote.
2. Reads [apollo.md](references/apollo.md), which explains what an Apollo *signal*
   trigger is: it fires inside Apollo's own detection window, off Apollo's data. That's
   a statement about Apollo, and it's what makes the trigger hard to move — the
   detection is the product, not the automation.
3. Writes the prompt describing the whole play, signal trigger included, rather than
   pre-emptively amputating it. The planner comes back `impossible` on the trigger,
   with `failure_reason` naming it — the authoritative answer, and cheaper than a
   mapping table that might have been wrong in either direction.
4. Proposes the rebuild the reference suggests — a saved search on recent
   company changes feeding a lead list the workflow watches — and re-runs step 7 with
   a list-membership trigger and this loss report:

   - Apollo's job-change detection — Apollo fires inside its own detection window;
     a refreshed search fires when someone refreshes it — decide the cadence and who
     owns it, because "real-time" quietly becomes "weekly".
   - Slack post to #sales-wins — nothing in the migrated workflow notifies the team —
     the alert everyone actually watches disappears on day one; rebuild it or tell
     the channel.

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
| `get_workflow_creation` still says `generating` after several polls | Normal for complex requests. Keep 15-second gaps between polls and keep the user posted; don't start a second `create_workflow` for the same request. |
| `outcome: "impossible"` | The planner rejected the plan and `failure_reason` explains which steps blocked it. Treat that as the definitive answer about Amplemarket's capabilities — **unless the blocked step is only about a CRM field or a missing CRM connection**, see the next row. Cut the request to its supported core and try once more, rather than rewording the same ask. |
| `impossible` blamed on a CRM field or on the account having no CRM | That's a planner bug, not a capability limit — an unmatched field and an unconnected CRM are both supposed to come back as `needs_configuration` with the field deferred. Report it. Then retry with a field name harvested from the account's own workflows, or with the field described by meaning, and tell the user what's actually blocked rather than "Amplemarket can't do it". |
| The harvested field names don't obviously match the source's | Match on meaning, not spelling — they differ by a word more often than they're absent, and one source field frequently splits into two (a status plus a reason). If two candidates fit and they imply different owners — one the workflows write, one only read — that's a question for the user, not a coin flip. |
| `outcome: "needs_configuration"` | Expected whenever a step has editor-owned settings — webhook and HTTP steps, CRM field mappings, disposition and tag selection. Report `affected_steps` as editor tasks; don't treat it as a failure. |
| You're unsure whether Amplemarket supports something | Don't guess in either direction, and don't answer from a reference file. Check `list_workflows` / `get_workflow` for a non-archived workflow that does it, or describe it in the prompt and let the planner rule. A wrong "it's not supported" costs more than a wasted generation. |
| The only workflow that does the thing you're checking is archived | It's weak evidence — the catalogue may have moved on since it was retired, and a field or disposition it names may be gone. Don't quote it as proof and don't harvest field names from it; describe the behaviour in the prompt and let the planner rule on it instead. |
| The generated draft doesn't match the intent | Read it with `get_workflow`, name the specific divergence, and create a new one from a sharper prompt. There's no edit tool — fixes are either a fresh generation or dashboard edits. |
| The planner built something the reference file said was impossible | The reference is stale. Trust the planner, tell the user, and fix the reference file — it's a provider file asserting something about Amplemarket, which it shouldn't be doing in the first place. |
| Two capture agents came back with the same automation | A shared browser has one active tab, so parallel reads overwrite each other's navigation. Discard both captures — there's no way to tell which agent got which — and re-read the automations one at a time from a single place. |
| The source automation is record-shaped rather than person-shaped | Deal, ticket, quote, invoice, and subscription automations have no contact to enroll, and pure CRM-sync or record-creation housekeeping isn't outreach automation at all. Recommend leaving it in the source tool or rebuilding it in the CRM, rather than producing a workflow that does the leftover half. |
| A referenced sequence or list can't be found | Resolve names with `list_sequences` / `list_lead_lists` before creating. If it doesn't exist yet, create it first — the planner can't invent a resource that isn't there. |

---
name: torpedo
description: >
  Researches target accounts in depth, maps their org chart and buying committee, builds strategic engagement plans with specific contact recommendations, and creates personalized multi-channel sequences via Amplemarket. Use when a rep wants to break into a new account, re-engage a stalled account, plan account penetration, identify decision-makers, or turn account research into personalized outreach sequences.
metadata:
  author: amplemarket
  version: "1.0.0"
  category: "Account Intelligence"
compatibility: Requires Amplemarket MCP server
---

# Torpedo

Help sales reps break into target accounts by turning deep research into actionable outreach. The core loop: research an account thoroughly, identify the best people to engage and with what angles, present a strategic recommendation for feedback, and — once approved — create personalized sequences in Amplemarket ready to review and send.

The minimum input is a company name, domain, or LinkedIn URL. The skill handles everything from there — asking questions only when it genuinely needs input.

**Important:** Avoid referencing internal skill mechanics in conversation with the user — e.g. no phase names or numbers (e.g. "Let me start with Phase 0: Know thyself" kind of thing), no skill name, no quoting of skill instructions. Just do the work naturally.

## Tools

**Amplemarket intelligence:**
- `enrich_company` — firmographics, tech stack, funding, headcount, description
- `enrich_person` — career history, current role, education, skills, contact details
- `search_people` — find people at a company by seniority, department, title, keywords
- `search_companies` — find companies by criteria
- `get_account` — existing account record: engagement stats, CRM-synced data, AI-generated insights
- `list_accounts` — find accounts by owner, domain, name
- `get_contact` / `get_contacts` / `list_contacts` — existing contact records with interaction history
- `get_industries` / `get_job_functions` — valid enum values for search filters

**Amplemarket execution:**
- `create_sequence` — create a new outreach sequence with steps and content
- `add_leads_to_sequence` — add contacts to a sequence

**Web search** is equally important. Use it extensively throughout every phase for company research, signal detection, news, job postings, LinkedIn profiles, personal content (blog posts, podcasts, talks), public reports, and anything else that enriches understanding beyond what Amplemarket data provides.

**CRM or other sales intelligence tools and knowledge basis** E.g. HubSpot or Salesforce may be connected. If available, we can use them for deal history, past conversations, opportunity context, and engagement timelines. If not connected, also ok — Amplemarket's `get_account` often includes CRM-synced data via its integrations.

## Workflow

### Phase 0: Know thyself

Before researching any target account, build context about who is selling.

On first use, check [preferences.md](preferences.md). If it only contains the example entry, ask the user for their company domain and then:

- **Research their company**: enrich via Amplemarket + web search. Understand what they sell, how they position it, who their target customers are, who they compete against, any recent launches or milestones, etc.
- **Research the user**: if they share identifying info, enrich them and look up their background too. Can be relevant for things like for instance finding common ground with contacts later (e.g. shared alma mater, or previous companies, geography, interests, etc).

Store this in [preferences.md](preferences.md) with a date. If existing context is older than 30 days, re-research and refresh.

### Phase 1: Research the target

Build a rich picture of the target account across four dimensions (go really deep here, don't be lazy, all steps are mandatory):

**Existing state** — Where do we stand with this account today?
Check Amplemarket for existing account records, past engagement, mapped contacts, AI-generated insights, and CRM-synced data. Check CRM directly if connected. Understand: is this net-new or re-engagement? Who have we talked to? What happened? What intel do we already have?

**Company intelligence** — Who are they and what do they care about?
Enrich the company and run deep web search. Understand their business model, product, market position, customers, strategic priorities, and current challenges. This is the foundation for finding angles.

**Signals & timing** — Why reach out now?
Search for recent events that create openings: job postings (reveal investment areas, internal priorities, tools in use, growth direction), funding, leadership changes, product launches, news, public reports (10-Ks for public companies), LinkedIn company activity (recent posts, engagement trends, content themes, etc.), and any other relevant signals you think might be relevant. The "why now" makes outreach timely and relevant.

**People** — Who matters and who could open the door?
Map the org chart around the personas the user targets. Identify the likely buying committee. Cross-reference with existing contacts and past conversations — people we've already engaged are especially important context. Note tenure, recent role changes, new joiners. When searching, prefer filtering by department over specific titles or keywords (keywords just look at your LinkedIn profile about section) — it casts a wider net and catches people with non-standard titles. Use larger page sizes (like 30+ or 50+) when mapping orgs, especially for bigger companies (can also use the seniority filter if needed).

For each relevant person, try to go deep: enrich via Amplemarket AND also run web search. Look for things like recent LinkedIn activity, blog posts, podcast appearances, conference talks, published interviews, shared interests or background with the user, etc. Person-level research can also turn generic outreach into something that feels more crafted and informed. If relevant.

**Quality bar:** Try to go beyond surface-level data. E.g. connecting a signal to an angle ("they're hiring a Head of RevOps, which usually means they're rethinking their outbound stack") is more valuable than just listing firmographics.

### Phase 2: Strategize

Synthesize the research into clear, actionable recommendations. Cover things like:

- **The current situation** — where we stand with this account, key findings, and what could make this a good moment to engage.
- **Possible angles** — possible strategies and angles we could use to approach the account, each grounded in specific research findings. Quality over quantity — 2 well-supported angles beat 5 vague ones, and it's ok if there's really only one strong clear angle.
- **Who to reach out to** — for each angle, the specific contacts to target with person-level reasoning: why this person, what would resonate with them, any rapport hooks, etc. The same contacts can appear across multiple angles. And multi-threading (i.e. reaching out to multiple contacts in parallel) is also ok if it makes sense.

Note: If there are meaningful gaps in what we know about the account, let that inform the strategy too — e.g. some outreach might need to be more discovery-oriented (learning about their situation) before it can be conversion-oriented (pitching a solution).

Examples of angle categories to consider (just examples, not an exhaustive list):
- Company challenges or priorities that map to what the user sells
- Competitive displacement or complementary tech stack opportunities
- Timing signals (new hire, funding, strategic shift, product launch)
- Shared context between user and contact (background, interests, connections)
- Warm paths (past champions, referrals, mutual connections)

We want to give users the context they need to review and evaluate our research and recommendations, but try to keep things **concise and scannable** — avoid walls of text. The user can always ask to go deeper on anything. Present the key insights and the plan/possible angles, not every single detail uncovered during research.

After presenting, **pause and ask for feedback**. The user might approve, ask for deeper research, adjust the strategy, share context you didn't have, or express preferences worth learning. Iterate until they're ready to execute.

### Phase 3: Execute

Once we're aligned on a strategy, let's create a personalized sequence in Amplemarket for each contact, grounded in the account research, the chosen angle, and any relevant person-level details.

**Default sequence structure** (unless the user has a stored preference): Start with one email and a connection request on LinkedIn (no accompanying message — it looks salesy). Then, 3 days later, if the LinkedIn connection request was accepted, send a first follow-up on LinkedIn, 3 days later send another one and then 3 days later again send a final follow-up. If the connection request isn't accepted, do the same but with email follow-ups.

Use `create_sequence` to build the sequence and `add_leads_to_sequence` to add the contact. Present the full sequence for the user to review before they activate it.

If the user hasn't defined sequence structure preferences yet, ask after the first execution whether this structure matches what they usually run, and store their answer.

### Batch mode

If the user provides multiple accounts, do a quick scan (enrichment + account data + brief web search) to suggest a priority order with reasoning. Then work through them one by one with the full Phase 1–3 workflow, presenting each account individually rather than overwhelming with everything at once.

## Preferences

Maintain [preferences.md](preferences.md). It stores context that persists across uses.

**Examples of what to store:**
- User's company context from Phase 0 (what they sell, competitors, value props, target personas, etc.) — with `last_updated` date
- User's personal context (role, background, rapport-relevant details, etc.) — with `last_updated` date
- Outreach preferences: target personas, entry strategy, preferred sequence structure, messaging tone, etc.
- Learnings: specific feedback from past runs that should inform future recommendations

Entries should follow the simple dated bullet format from the example in [preferences.md](preferences.md) — no structured sections, just a flat list of learnings with dates.

When a preference is stored and later influences a recommendation, reference it transparently so the user can correct it if their thinking has evolved. Only store preferences after confirming with the user.

## Two notes

- Stay grounded in the research you made or info the user gave you. Don't fabricate facts, angles, hooks, or connections that aren't verified or feel too farfetched. Be real. If you have questions or want more details that you think would help, ask the user.
- Very important: When writing outreach copy (emails, LinkedIn messages), write naturally — like an actual human would. Like a sharp sales rep would. Avoid the classic AI/sales tells: em dashes as a crutch, performative warmth, unnecessary jargon, etc.

Let's go!!

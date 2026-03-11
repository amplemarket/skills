# Amplemarket Skills

Sales skills for AI agents, designed to be used with the [Amplemarket MCP](https://www.amplemarket.com/blog/mcp).

Each skill is a self-contained directory with a `SKILL.md` file following the [agentskills.io](https://agentskills.io) specification. Skills provide structured instructions that AI agents can follow to execute sales workflows — from prospecting and enrichment to pipeline review and team coaching.

## Available Skills

| Skill | Description |
|-------|-------------|
| account-expansion-mapper | Map untapped departments within existing accounts for expansion |
| account-whitespace-analyzer | Analyze customer accounts for untapped departments |
| ae-pipeline-review | Status check on AE accounts with open deals |
| build-targeted-lead-list | End-to-end workflow for building lead lists |
| champion-tracker | Track former champions who moved to new companies |
| closed-lost-revival-scanner | Scan closed-lost deals and score revival potential |
| competitive-account-research | Generate comprehensive account briefs with decision-maker mapping |
| competitor-displacement-targeting | Find companies using competitor products for displacement campaigns |
| conference-event-lead-finder | Find and enrich leads attending target conferences |
| conference-networking-prep | Prepare personalized networking briefs for conferences |
| daily-web-signal-prospector | Prospect based on daily web signals and triggers |
| deal-contact-gap-analysis | Identify missing stakeholders in active deals |
| deliverability-health-check | Check email deliverability health across mailboxes |
| deliverability-watchdog | Monitor deliverability and flag issues proactively |
| duo-copilot-dismiss-diagnostics | Diagnose Duo Copilot dismiss rate patterns |
| duo-copilot-performance-monitor | Monitor Duo Copilot performance metrics |
| enrich-and-score-lead | Enrich and score individual leads |
| funding-event-prospector | Find recently funded companies and their decision-makers |
| hiring-signal-prospector | Prospect into companies showing hiring signals |
| icp-refiner | Refine ICP based on closed-won analysis |
| job-change-signal-tracker | Track job changes among target contacts |
| lookalike-audience-builder | Build lookalike audiences from best customers |
| market-mapping | Map companies and contacts across a market segment |
| new-rep-ramp-tracker | Track new rep ramp progress against benchmarks |
| outreach-activity-dashboard | Summarize outreach activity metrics |
| outreach-personalization-research | Research prospects for personalized outreach |
| pipeline-account-review | Review pipeline accounts with engagement data |
| podcast-personalization-research | Research podcast appearances for personalized outreach |
| pre-call-research-brief | Generate research briefs before sales calls |
| prospect-icp-search | Search for prospects matching ICP criteria |
| rep-coaching-brief | Generate coaching notes for manager-rep 1:1s |
| sdr-account-prioritization-plan | Help SDRs prioritize accounts by signal and engagement |
| sequence-performance-analyzer | Compare outreach sequences to find top performers |
| stale-opportunity-reviver | Revive stale deals with fresh research and outreach plans |
| team-performance-review | Compare rep performance with leaderboard format |

## Contributing

1. Each skill lives in its own directory containing a `SKILL.md` file.
2. When modifying a skill, **you must update the `version` field** in the YAML frontmatter of `SKILL.md`. This version is used to tag and create GitHub releases automatically. Please use [semver](https://semver.org/) versioning.

   ```yaml
   ---
   name: my-skill
   description: >
     What the skill does.
   metadata:
     author: amplemarket
     version: "1.0.1"  # <-- bump this
   compatibility: Requires Amplemarket MCP server
   ---
   ```

3. On merge to `master`, a GitHub Action detects which skills changed and creates a release for each one, tagged as `<skill-name>-v<version>` with a zip of the skill directory attached.

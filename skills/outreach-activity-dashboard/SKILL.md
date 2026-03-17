---
name: outreach-activity-dashboard
description: >
  Multi-channel activity dashboard with volume trends, engagement snapshots, and consistency analysis across email, LinkedIn, and phone to help sales leaders spot gaps and keep the team on track.
metadata:
  author: amplemarket
  version: "2.0.0"
  category: "Analytics"
compatibility: Requires Amplemarket MCP server
---

# Outreach Activity Dashboard

Multi-channel activity dashboard covering email, LinkedIn, phone, meetings, and Duo — with volume trends, engagement snapshots, and consistency analysis to help sales leaders spot gaps and keep the team on track.

## Instructions

When a user wants to see outreach activity, query Amplemarket analytics for volume and engagement data across all active channels, identify trends, and present a structured dashboard.

### Clarify with the user

- Timeframe (default: last 30 days)
- Granularity: daily, weekly, or monthly (default: weekly for 30-day views, monthly for 90+ day views)
- Full team or specific reps/teams
- If they want a period comparison (e.g., this month vs last month)

### What to ask analytics about

- **Email:** volume over time, reply rate, interested rate, open rate
- **LinkedIn:** tasks completed over time, message reply rate, interested rate
- **Phone:** calls dialed over time, connected rate
- **Meetings:** booked count over time
- **Duo (if enabled):** leads generated, actioned, dismiss rate
- **Lead sourcing:** leads added to lists (leading indicator for future volume)

For period comparisons, query both periods.

### How to think about the results

- **Look for trends across the timeframe.** Volume spikes or drops, engagement changes, ramp-up or ramp-down patterns. If all channels drop simultaneously, something systemic happened (holiday, offsite, sequence pause).
- **Pair volume with engagement.** Activity volume alone doesn't indicate quality. A week with high volume but declining reply rates may be worse than a lower-volume week with strong engagement.
- **Flag consistency issues.** Periods with zero or very low activity on business days are worth calling out. If per-rep data is available, flag reps with significant gaps.
- **Lead sourcing is a leading indicator.** If lead additions drop, outreach volume will follow 1-2 weeks later. Flag this early.
- **Weekends and holidays naturally show low activity.** Don't flag these as consistency issues.
- This skill can also run on a recurring schedule (e.g., weekly Slack digest). When posting to Slack, keep it scannable — lead with a scorecard, flag significant changes, skip detail when everything is healthy.

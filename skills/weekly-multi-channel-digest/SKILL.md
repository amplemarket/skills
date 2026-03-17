---
name: weekly-multi-channel-digest
description: >
  Comprehensive weekly outreach report covering email, LinkedIn, phone, meetings, and Duo in a single digest — designed for recurring Slack delivery to sales leaders.
metadata:
  author: amplemarket
  version: "1.0.0"
  category: "Analytics"
compatibility: Requires Amplemarket MCP server
---

# Weekly Multi-Channel Digest

Comprehensive weekly outreach report covering email, LinkedIn, phone, meetings, and Duo in a single digest — designed for recurring Slack delivery to sales leaders.

## Instructions

When invoked (on-demand or on a recurring weekly schedule), query Amplemarket analytics for activity and engagement across all outreach channels and compile a single report.

### Clarify with the user

- Slack channel to post to (if recurring)
- Team scope: full account, specific team, or per-team reports
- Any custom metrics or alerts they want included

### What to ask analytics about

For both the current week and the prior week (for comparison):

- **Email:** volume, reply rate, interested rate
- **LinkedIn:** tasks completed (by message type if useful), reply rate, interested rate
- **Phone:** calls dialed, connected rate
- **Meetings:** booked count
- **Duo (if enabled):** leads generated, actioned, dismiss rate

### How to think about the results

- **Week-over-week comparison is the core value.** It answers "are we trending up or down?" Always include it.
- Keep the report scannable. Sales leaders want to read this in under 60 seconds. Lead with a scorecard, then channel detail.
- Flag any metric that moved significantly — more than 20% change in volume or more than 2 points change in rates.
- If all channels dropped simultaneously, something systemic happened (holiday, offsite, sequence pause). Call this out rather than flagging each channel individually.
- If a channel has zero activity, include it with a note rather than omitting it. The absence of activity is itself useful information.
- For recurring Slack delivery, format using Slack mrkdwn. Only post alerts when something is worth flagging — don't spam the channel when everything is healthy.
- If posting per-team reports, keep them separate. Don't combine teams. Offer cross-team comparison only if asked.

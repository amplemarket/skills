---
name: persona-effectiveness-report
description: >
  Analyze which target personas convert best across sequences, channels, and signals to help teams double down on what works and deprioritize what doesn't.
metadata:
  author: amplemarket
  version: "1.0.0"
  category: "Analytics"
compatibility: Requires Amplemarket MCP server
---

# Persona Effectiveness Report

Analyze which target personas convert best across sequences, channels, and signals to help teams double down on what works and deprioritize what doesn't.

## Instructions

When a user wants to understand which personas are performing best, query Amplemarket analytics for per-persona metrics across the lead funnel, channels, and Duo signals.

### Clarify with the user

- Timeframe (default: last 30 days)
- Full account or specific teams

### What to ask analytics about

- **Lead funnel by persona:** leads sequenced, reply rate, interested rate, meeting booked rate
- **Email by persona:** reply rate, interested rate
- **LinkedIn by persona:** message reply rate, interested rate
- **Phone by persona:** interested rate
- **Duo signals by persona:** actioned rate, dismissed rate
- **Deals by persona:** pipeline revenue, closed revenue (if the user cares about revenue attribution)

### How to think about the results

- **Interested rate** is the key metric for persona evaluation. Reply rate can be misleading — many replies are OOO, not-right-person, or other non-interest signals.
- Look for **channel affinity per persona**. Some personas respond much better on LinkedIn than email, or vice versa. This is actionable — teams can adjust their sequence channel mix per persona.
- Always consider volume. A persona with a 15% interested rate on 20 leads is not reliably better than one with 5% on 500 leads.
- If a persona has high reply rate but low interested rate, dig into why — it may signal the messaging is generating responses but not the right kind.
- For low-performing personas, the question is whether to fix the approach (messaging, channel mix) or deprioritize the persona entirely.
- If personas aren't configured in the account, the analytics engine will return no persona data. Suggest analyzing by contact title as an alternative.

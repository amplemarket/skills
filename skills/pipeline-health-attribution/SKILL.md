---
name: pipeline-health-attribution
description: >
  Full pipeline picture — where revenue is coming from, where deals are stuck, and which sequences, personas, and reps drive the most pipeline and closed revenue.
metadata:
  author: amplemarket
  version: "1.0.0"
  category: "Analytics"
compatibility: Requires Amplemarket MCP server
---

# Pipeline Health & Attribution

Full pipeline picture — where revenue is coming from, where deals are stuck, and which sequences, personas, and reps drive the most pipeline and closed revenue.

## Instructions

When a user wants to understand their pipeline, query Amplemarket analytics for deal metrics across multiple dimensions.

### Clarify with the user

- Timeframe (default: current quarter)
- Pipeline created, closed revenue, or both
- Full account or specific teams/reps

### What to ask analytics about

- **Pipeline overview:** total pipeline revenue created, total closed revenue, deal stage distribution
- **Attribution by sequence:** which sequences generated the most pipeline and closed revenue
- **Attribution by persona:** pipeline and closed revenue by persona
- **Attribution by rep:** meetings booked per rep (leading indicator), pipeline per rep if available
- **Time comparison:** this quarter vs last quarter, or this month vs last month

### How to think about the results

- **Pipeline created** shows what's feeding the funnel. **Closed revenue** shows what's actually converting. Both matter but they tell different stories.
- Look at **deal stage distribution** to find bottlenecks. If a disproportionate share of pipeline sits in one stage, deals may be stalling there.
- For sequence attribution, focus on **efficiency** (revenue per lead or close rate), not just total revenue. A sequence with lower total pipeline but higher close rate may be more valuable to scale.
- For persona attribution, check for mismatches — personas that create pipeline but don't close may indicate product-market fit issues with that segment, or that these contacts aren't the economic buyers.
- Meetings booked is a useful leading indicator for pipeline but lives in separate data. It can't be directly joined with deal revenue — treat them as complementary views.
- If deal data is empty, the account may not have CRM integration. Ask the user to confirm.

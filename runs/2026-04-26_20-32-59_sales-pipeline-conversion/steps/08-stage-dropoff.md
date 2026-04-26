# Step 8: Stage Dropoff

> Where does the pipeline leak? Shows absolute count and value lost at each stage transition — the diagnostic tool for rep coaching conversations.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 5 |
| Row delta | -20 (80.0% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ Highest drop-off: Proposal → Negotiation — 1 deal(s) lost (25.0%)
- ℹ Note: drop-off is calculated from cumulative stage counts, not longitudinal tracking — a 'drop' means fewer deals are at the next stage, not that specific deals moved backward
- ℹ Use this tab to open coaching conversations, not to assign blame — context from rep notes often explains apparent drop-offs

## Preview

| Transition | Deals Lost | Value Lost ($) | Drop-off Rate |
| --- | --- | --- | --- |
| Prospecting → Qualified | 0 | 0.0 | 0.0% |
| Qualified → Demo | 0 | 0.0 | 0.0% |
| Demo → Proposal | 0 | 500.0 | 0.0% |
| Proposal → Negotiation | 1 | 71000.0 | 25.0% |
| Negotiation → Won | 0 | 0.0 | 0.0% |

---
*Step logged at 2026-04-26T20:32:59*
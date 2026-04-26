# Step 4: Pipeline Conversion Value

> Aggregate deal value at each pipeline stage to show how much revenue sits at each stage and how it converts forward. Conversion rate is calculated relative to the prior stage's total value.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 7 |
| Row delta | -18 (72.0% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ Total active pipeline: $575,500 across 16 deals
- ℹ Total won revenue: $185,700 across 5 deals
- ℹ Prospecting rows show $0 total value — expected, as early-stage deals typically have no amount set
- ℹ Conversion rates >100% indicate average deal size increases as deals progress (healthy sign)

## Preview

| Stage | Deal Count | Total Value ($) | Avg Deal Size ($) | Conv Rate vs Prior |
| --- | --- | --- | --- | --- |
| Prospecting | 2 | 0 | 0 | — |
| Qualified | 3 | 68500 | 22833 | — |
| Demo | 4 | 193000 | 48250 | 281.8% |
| Proposal | 4 | 192500 | 48125 | 99.7% |
| Negotiation | 3 | 121500 | 40500 | 63.1% |
| Won | 5 | 185700 | 37140 | 152.8% |
| Lost | 4 | 107700 | 35900 | 58.0% |

---
*Step logged at 2026-04-26T20:32:59*
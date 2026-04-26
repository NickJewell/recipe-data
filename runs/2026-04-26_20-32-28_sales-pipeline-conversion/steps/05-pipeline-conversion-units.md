# Step 5: Pipeline Conversion Units

> Count deals at each stage rather than summing value. Unit conversion rates can diverge significantly from value conversion rates when deal size varies by stage.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 7 |
| Row delta | -18 (72.0% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ Compare unit rates vs value rates in Step 4 — large divergence means deal size varies by stage
- ℹ Win rate by units: 5/9 closed deals won

## Preview

| Stage | Deal Count | Conv Rate vs Prior |
| --- | --- | --- |
| Prospecting | 2 | — |
| Qualified | 3 | 150.0% |
| Demo | 4 | 133.3% |
| Proposal | 4 | 100.0% |
| Negotiation | 3 | 75.0% |
| Won | 5 | 166.7% |
| Lost | 4 | 80.0% |

---
*Step logged at 2026-04-26T20:32:28*
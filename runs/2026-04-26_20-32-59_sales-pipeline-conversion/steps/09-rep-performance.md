# Step 9: Rep Performance

> Pipeline and won revenue by rep. One of the most politically sensitive tabs — present facts, not judgements. Sort by Revenue Won descending.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 3 |
| Row delta | -22 (88.0% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ 3 reps in this dataset: Alice Chen, Bob Patel, Carol Smith
- ✓ Rep revenue total ($185,700) matches Won aggregate from Step 4
- ℹ Top rep by revenue: Alice Chen
- ℹ Win rate varies by rep — consider deal mix (enterprise vs SMB) before drawing conclusions
- ℹ Reps with 0 Won deals may have a shorter tenure or different territory — validate before flagging

## Preview

| Rep | Total Deals | Active Pipeline ($) | Deals Won | Revenue Won ($) | Deals Lost | Win Rate |
| --- | --- | --- | --- | --- | --- | --- |
| Alice Chen | 9 | 276500.0 | 2 | 112200.0 | 1 | 66.7% |
| Bob Patel | 8 | 150500.0 | 2 | 61500.0 | 1 | 66.7% |
| Carol Smith | 8 | 148500.0 | 1 | 12000.0 | 2 | 33.3% |

---
*Step logged at 2026-04-26T20:32:59*
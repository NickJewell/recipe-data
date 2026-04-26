# Step 6: Win Rate By Quarter

> Calculate win rate (won / (won + lost)) per closed quarter. Win rate by $ and by units are both shown — a gap between the two means deal size differs significantly between won and lost deals.

## Stats

| | |
|---|---|
| Rows in | 9 |
| Rows out | 2 |
| Row delta | -7 (77.8% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ 2 quarter(s) with closed deals in this dataset
- ℹ Typical B2B win rates: 20–40% (transactional), 15–25% (enterprise) — adjust benchmark for your market
- ℹ A gap between Win Rate (units) and Win Rate ($) means larger deals close at a different rate — worth investigating
- ℹ 9 total closed deals: 5 Won, 4 Lost

## Preview

| Quarter | Lost Count | Won Count | Lost Value | Won Value | Win Rate (units) | Win Rate ($) |
| --- | --- | --- | --- | --- | --- | --- |
| Q1 2024 | 4 | 4 | 107700.0 | 152700.0 | 50.0% | 58.6% |
| Q4 2024 | 0 | 1 | 0.0 | 33000.0 | 100.0% | 100.0% |

---
*Step logged at 2026-04-26T20:32:28*
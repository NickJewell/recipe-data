# Step 7: Sales Cycle Length

> For Won deals, calculate days from created_date to close_date. Average and median sales cycle are the most useful metrics for quota planning and forecasting.

## Stats

| | |
|---|---|
| Rows in | 5 |
| Rows out | 4 |
| Row delta | -1 (20.0% drop) |
| Nulls in key columns | 0 |

## Data Quality

- ℹ Average sales cycle: 221 days — benchmark: 30–90 days (SMB SaaS), 90–180 days (enterprise)
- ℹ 5 of 5 Won deals have both created_date and close_date for cycle calculation
- ✓ No suspiciously short sales cycles (<7 days)
- ⚠ Very long cycles (>365 days) may indicate deals that were never properly closed out

## Preview

| Metric | Value |
| --- | --- |
| Average Sales Cycle (Won deals) | 221 days |
| Median Sales Cycle (Won deals) | 155 days |
| Fastest close | 35 days |
| Slowest close | 415 days |

---
*Step logged at 2026-04-26T20:32:28*
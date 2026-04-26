# Step 3: Parse Dates Coerce Amounts

> Convert close_date and created_date to proper datetimes (handles DD/MM/YYYY, YYYY-MM-DD, and ISO formats). Coerce amount to numeric (strips $, £, commas). Flag stale open deals where close_date is in the past but stage is not Won or Lost.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 25 |
| Row delta | +0 (0.0% gain) |
| Nulls in key columns | 0 |

## Data Quality

- ✓ close_date parsed — 0 NaT values
- ℹ 3 rows have amount = 0 (blank or non-numeric in source — expected for Prospecting/early-stage deals)
- ℹ Amount range: $4,200 – $112,000 — review for obvious data entry errors
- ⚠ 16 stale open deals detected: close_date is in the past but stage is not Won or Lost — these appear in the Source Data tab with `is_stale = True` and are counted in the Exec Summary

## Preview

| opportunity_name | stage_canonical | amount | close_date | close_quarter | is_stale |
| --- | --- | --- | --- | --- | --- |
| Acme Corp - Annual License | Won | 45000.0 | 2024-01-15 00:00:00 | Q1 2024 | False |
| TechStart Inc - Enterprise | Won | 28500.0 | 2024-01-15 00:00:00 | Q1 2024 | False |
| Riverside Group - Pro Plan | Won | 12000.0 | 2024-03-02 00:00:00 | Q1 2024 | False |
| Delta Systems - Expansion | Won | 67200.0 | 2024-02-20 00:00:00 | Q1 2024 | False |
| Nexus Retail - Platform | Won | 33000.0 | 2024-10-03 00:00:00 | Q4 2024 | False |
| Pinnacle Finance - SMB | Lost | 0.0 | 2024-01-20 00:00:00 | Q1 2024 | False |
| Summit Corp - Starter | Lost | 8500.0 | 2024-01-28 00:00:00 | Q1 2024 | False |
| Crestview Ltd - API | Lost | 4200.0 | 2024-02-14 00:00:00 | Q1 2024 | False |
| Horizon Tech - Enterprise | Lost | 95000.0 | 2024-01-03 00:00:00 | Q1 2024 | False |
| Atlas Group - Pro | Negotiation | 54000.0 | 2024-04-15 00:00:00 | Q2 2024 | True |

---
*Step logged at 2026-04-26T20:32:28*
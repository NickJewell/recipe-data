# Step 1: Load Standardise Columns

> Load the CRM export and normalise column names to lowercase with underscores. Map CRM-specific column name variants (e.g. deal_name, deal_stage) to the recipe's standard names.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 25 |
| Row delta | +0 (0.0% gain) |
| Nulls in key columns | 3 |

## Data Quality

- ✓ All 5 required columns present: `opportunity_name`, `stage`, `amount`, `close_date`, `owner`
- ℹ No column name remapping needed — names already match standard schema
- ℹ Optional columns present: `created_date`, `closed_date`
- ✓ 25 rows loaded from `data/inputs/sample/crm-export-sample.csv`

## Preview

| opportunity_name | stage | amount | close_date | owner | created_date | closed_date |
| --- | --- | --- | --- | --- | --- | --- |
| Acme Corp - Annual License | Closed Won | 45000 | 2024-01-15 | Alice Chen | 2023-10-01 | 2024-01-15 |
| TechStart Inc - Enterprise | deal won | $28,500 | 15/01/2024 | Bob Patel | 2023-11-12 | 15/01/2024 |
| Riverside Group - Pro Plan | Won | 12000 | 2024-02-03 | Carol Smith | 2023-12-01 | 2024-02-03 |
| Delta Systems - Expansion | Closed Won | $67,200 | 20/02/2024 | Alice Chen | 2023-10-15 | 2024-02-20 |
| Nexus Retail - Platform | Won | 33000 | 2024-03-10 | Bob Patel | 2024-01-05 | 2024-03-10 |
| Pinnacle Finance - SMB | Closed Lost | nan | 2024-01-20 | Carol Smith | 2023-11-01 | 2024-01-20 |
| Summit Corp - Starter | deal lost | 8500 | 28/01/2024 | Alice Chen | 2023-12-15 | 2024-01-28 |
| Crestview Ltd - API | Lost | 4200 | 2024-02-14 | Bob Patel | 2024-01-10 | 2024-02-14 |
| Horizon Tech - Enterprise | Closed Lost | $95,000 | 2024-03-01 | Carol Smith | 2023-09-20 | 2024-03-01 |
| Atlas Group - Pro | Negotiation/Review | $54,000 | 2024-04-15 | Alice Chen | 2024-01-20 | nan |

---
*Step logged at 2026-04-26T20:32:28*
# Step 2: Normalise Stage Names

> Map CRM-specific stage names to 7 canonical stages (Prospecting → Qualified → Demo → Proposal → Negotiation → Won → Lost) so the conversion funnel is consistent regardless of which CRM the data came from.

## Stats

| | |
|---|---|
| Rows in | 25 |
| Rows out | 25 |
| Row delta | 0 (no change) |
| Nulls in key columns | 0 |

## Data Quality

- ✓ All 25 stage values mapped successfully
- ✓ Demo: 4, Lost: 4, Negotiation: 3, Proposal: 4, Prospecting: 2, Qualified: 3, Won: 5

## Preview

| opportunity_name | stage_raw | stage_canonical | amount | owner |
| --- | --- | --- | --- | --- |
| Acme Corp - Annual License | Closed Won | Won | 45000 | Alice Chen |
| TechStart Inc - Enterprise | deal won | Won | $28,500 | Bob Patel |
| Riverside Group - Pro Plan | Won | Won | 12000 | Carol Smith |
| Delta Systems - Expansion | Closed Won | Won | $67,200 | Alice Chen |
| Nexus Retail - Platform | Won | Won | 33000 | Bob Patel |
| Pinnacle Finance - SMB | Closed Lost | Lost | nan | Carol Smith |
| Summit Corp - Starter | deal lost | Lost | 8500 | Alice Chen |
| Crestview Ltd - API | Lost | Lost | 4200 | Bob Patel |
| Horizon Tech - Enterprise | Closed Lost | Lost | $95,000 | Carol Smith |
| Atlas Group - Pro | Negotiation/Review | Negotiation | $54,000 | Alice Chen |

---
*Step logged at 2026-04-26T20:32:59*
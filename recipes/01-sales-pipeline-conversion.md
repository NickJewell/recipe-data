---
recipe: Sales Pipeline Conversion + Win Rate Analysis
version: 1.0.0
cadence: weekly
villain: "Copy-pasting CRM data into a pivot table every week to tell the VP of Sales what the pipeline looks like"
sources:
  - name: crm_export
    description: Open opportunities export from your CRM — one row per deal
    format: xlsx or csv
    required_columns:
      - opportunity_name
      - stage
      - amount
      - close_date
      - owner
    common_issues:
      - Stage names differ by CRM (Salesforce uses "Closed Won", HubSpot uses "Deal Won")
      - Amount may be blank for early-stage deals
      - close_date may be in the past for stale open deals
      - Owner field may be full name, email, or user ID depending on export settings
output: pipeline-conversion.xlsx
output_tabs:
  - Source Data
  - Pipeline Conversion ($)
  - Pipeline Conversion (units)
  - Win Rate by Quarter
  - Sales Cycle Length
  - Stage Drop-off
  - Rep Performance
  - Exec Summary
tags:
  - sales
  - pipeline
  - RevOps
  - weekly
---

# Sales Pipeline Conversion + Win Rate Analysis

Every RevOps analyst rebuilds some version of this report every week. The raw CRM export gets pasted into a pivot, stages get manually bucketed, win rates get hand-calculated, and the whole thing is emailed to the VP of Sales by Friday at 4pm. This recipe automates the entire sequence — load the CRM export, normalise stage names across CRM systems, calculate conversion at each stage, compute win rates by quarter and rep, and write a board-ready output workbook.

This recipe is the generalised version of the "instruction file that replaced my analyst" pattern. It handles Salesforce, HubSpot, and Pipedrive exports out of the box with automatic stage normalisation.

## Sources

### crm_export

Open and closed opportunities export from your CRM. One row per deal.

Tested against: Salesforce, HubSpot, Pipedrive. Should work with any CRM that exports one-row-per-opportunity.

Expected columns (the recipe will ask you to map if column names differ):
- `opportunity_name` / `Deal Name` / `Name` — deal identifier
- `stage` / `Stage` / `Deal Stage` — current pipeline stage
- `amount` / `Amount` / `Deal Value` — deal value in base currency
- `close_date` / `Close Date` / `Expected Close Date` — projected or actual close date
- `owner` / `Owner` / `Deal Owner` / `Assigned To` — rep name
- `created_date` / `Create Date` — when the deal was created (for sales cycle calc)
- `closed_date` / `Close Date (Actual)` — actual close date for won/lost deals (may be same as close_date)

Common issues:
- Salesforce: stage = "Closed Won" / "Closed Lost"; HubSpot: "Deal Won" / "Deal Lost"; Pipedrive: "Won" / "Lost"
- Amount is often blank for Prospecting/Qualification stage deals — recipe treats blank as 0 for count metrics, excludes from $ metrics
- close_date includes past dates for deals that haven't been updated — recipe flags these as "stale" in the output

## Steps

### Step 1: Load and standardise column names

Load the export, normalise column names to lowercase with underscores, and map
any CRM-specific column names to the recipe's standard names.

```python
import pandas as pd
import re

# Load the file
if source_crm_export.endswith('.xlsx'):
    df = pd.read_excel(source_crm_export)
else:
    df = pd.read_csv(source_crm_export)

# Normalise column names
df.columns = df.columns.str.strip().str.lower().str.replace(r'[\s\-/]+', '_', regex=True)

# Map common CRM column name variants to standard names
COLUMN_MAP = {
    'deal_name': 'opportunity_name',
    'name': 'opportunity_name',
    'deal_stage': 'stage',
    'pipeline_stage': 'stage',
    'deal_value': 'amount',
    'value': 'amount',
    'deal_amount': 'amount',
    'expected_close_date': 'close_date',
    'projected_close_date': 'close_date',
    'deal_owner': 'owner',
    'assigned_to': 'owner',
    'rep': 'owner',
    'create_date': 'created_date',
    'creation_date': 'created_date',
}
df = df.rename(columns={k: v for k, v in COLUMN_MAP.items() if k in df.columns})

print(f"Loaded {len(df)} rows. Columns: {list(df.columns)}")
```

**Preview:** Confirm row count matches expected deal count. Check that `stage`, `amount`, `close_date`, `owner` columns are present.

### Step 2: Normalise stage names to canonical pipeline stages

Map CRM-specific stage names to 7 canonical stages so the conversion funnel
is consistent regardless of which CRM the data came from.

```python
# 7 canonical stages: Prospecting → Qualified → Demo → Proposal → Negotiation → Won → Lost
STAGE_MAP = {
    # Salesforce
    'prospecting': 'Prospecting',
    'qualification': 'Qualified',
    'needs analysis': 'Qualified',
    'value proposition': 'Demo',
    'id. decision makers': 'Demo',
    'perception analysis': 'Proposal',
    'proposal/price quote': 'Proposal',
    'negotiation/review': 'Negotiation',
    'closed won': 'Won',
    'closed lost': 'Lost',
    # HubSpot
    'appointment scheduled': 'Prospecting',
    'qualified to buy': 'Qualified',
    'presentation scheduled': 'Demo',
    'decision maker bought-in': 'Proposal',
    'contract sent': 'Negotiation',
    'deal won': 'Won',
    'deal lost': 'Lost',
    # Pipedrive
    'lead in': 'Prospecting',
    'contact made': 'Qualified',
    'demo scheduled': 'Demo',
    'proposal made': 'Proposal',
    'negotiations started': 'Negotiation',
    'won': 'Won',
    'lost': 'Lost',
    # Generic / freeform CRM exports
    'qualified': 'Qualified',
    'demo': 'Demo',
    'proposal': 'Proposal',
    'proposal sent': 'Proposal',
    'negotiation': 'Negotiation',
    'in negotiation': 'Negotiation',
}

df['stage_raw'] = df['stage'].copy()
df['stage_canonical'] = df['stage'].str.strip().str.lower().map(STAGE_MAP)

# Flag unmapped stages so the analyst knows what got dropped
unmapped = df[df['stage_canonical'].isna()]['stage_raw'].unique()
if len(unmapped) > 0:
    print(f"WARNING: {len(unmapped)} stage values could not be mapped: {unmapped}")
    print("These rows will appear in Source Data but be excluded from funnel metrics.")

df_mapped = df[df['stage_canonical'].notna()].copy()
```

**Preview:** Check `stage_canonical` value counts. Any "None" values are unmapped stages — review the warning list above.

### Checkpoint: Stage mapping coverage

```
Assert: df_mapped['stage_canonical'].notna().all()
On failure: Some stage values were not mapped to canonical stages. Review the unmapped list
            printed above. Either add the missing stage name to STAGE_MAP in Step 2,
            or filter those rows from the source file if they represent data quality issues.
```

### Step 3: Parse dates and coerce amounts

Convert close_date and created_date to proper datetimes. Coerce amount to
numeric (handles currency symbols and comma separators).

```python
from dateutil import parser as dateutil_parser
import numpy as np

def parse_date_safe(val):
    if pd.isna(val) or str(val).strip() == '':
        return pd.NaT
    try:
        return pd.Timestamp(dateutil_parser.parse(str(val)))
    except Exception:
        return pd.NaT

df_mapped['close_date'] = df_mapped['close_date'].apply(parse_date_safe)
if 'created_date' in df_mapped.columns:
    df_mapped['created_date'] = df_mapped['created_date'].apply(parse_date_safe)

# Coerce amount — handles "$1,234.56", "£1,234", "1234.56"
df_mapped['amount'] = pd.to_numeric(
    df_mapped['amount'].astype(str).str.replace(r'[^\d.\-]', '', regex=True),
    errors='coerce'
).fillna(0)

# Derive close quarter
df_mapped['close_quarter'] = df_mapped['close_date'].apply(
    lambda d: f"Q{d.quarter} {d.year}" if pd.notna(d) else 'Unknown'
)

# Flag stale open deals (close date in the past, not Won or Lost)
today = pd.Timestamp.today().normalize()
df_mapped['is_stale'] = (
    (df_mapped['close_date'] < today) &
    (~df_mapped['stage_canonical'].isin(['Won', 'Lost']))
)

print(f"Stale open deals: {df_mapped['is_stale'].sum()}")
```

**Preview:** Check that `close_date` parsed correctly (no unexpected NaT values). Check `amount` min/max for sanity. Review stale deal count.

### Step 4: Build pipeline conversion funnel ($)

Aggregate deal value at each pipeline stage to show how much revenue is
sitting at each stage and how it converts forward.

```python
STAGE_ORDER = ['Prospecting', 'Qualified', 'Demo', 'Proposal', 'Negotiation', 'Won', 'Lost']

# Active pipeline (excluding Won/Lost)
active = df_mapped[~df_mapped['stage_canonical'].isin(['Won', 'Lost'])].copy()

# Won deals
won = df_mapped[df_mapped['stage_canonical'] == 'Won'].copy()

# Conversion table by $
funnel_value = []
for stage in STAGE_ORDER:
    stage_df = df_mapped[df_mapped['stage_canonical'] == stage]
    funnel_value.append({
        'Stage': stage,
        'Deal Count': len(stage_df),
        'Total Value': stage_df['amount'].sum(),
        'Avg Deal Size': stage_df[stage_df['amount'] > 0]['amount'].mean() if len(stage_df) > 0 else 0,
    })

pipeline_conversion_value = pd.DataFrame(funnel_value)
pipeline_conversion_value['Total Value'] = pipeline_conversion_value['Total Value'].round(0)
pipeline_conversion_value['Avg Deal Size'] = pipeline_conversion_value['Avg Deal Size'].round(0)

# Conversion rate vs prior stage
vals = pipeline_conversion_value['Total Value'].values
pipeline_conversion_value['Conv Rate vs Prior'] = [
    f"{vals[i]/vals[i-1]:.1%}" if i > 0 and vals[i-1] > 0 else '—'
    for i in range(len(vals))
]
```

**Preview:** Check that stage totals are plausible. Won and Lost stages should reflect closed deals.

### Step 5: Build pipeline conversion funnel (units)

Same as Step 4 but counting deals rather than value — important because deal
count and deal value can tell very different stories.

```python
funnel_units = []
for stage in STAGE_ORDER:
    stage_df = df_mapped[df_mapped['stage_canonical'] == stage]
    count = len(stage_df)
    funnel_units.append({'Stage': stage, 'Deal Count': count})

pipeline_conversion_units = pd.DataFrame(funnel_units)
counts = pipeline_conversion_units['Deal Count'].values
pipeline_conversion_units['Conv Rate vs Prior'] = [
    f"{counts[i]/counts[i-1]:.1%}" if i > 0 and counts[i-1] > 0 else '—'
    for i in range(len(counts))
]
```

**Preview:** Compare unit conversion rates vs value conversion rates — a big difference means deal size varies significantly by stage (early pipeline skewed by large deals).

### Step 6: Win rate by quarter

Calculate win rate (won / (won + lost)) for each closed quarter and trailing
12-month overall. This is what the VP of Sales actually cares about.

```python
closed = df_mapped[df_mapped['stage_canonical'].isin(['Won', 'Lost'])].copy()

win_rate_by_quarter = (
    closed.groupby(['close_quarter', 'stage_canonical'])
    .agg(count=('opportunity_name', 'count'), value=('amount', 'sum'))
    .unstack(fill_value=0)
    .reset_index()
)
win_rate_by_quarter.columns = ['Quarter', 'Lost Count', 'Won Count', 'Lost Value', 'Won Value']
win_rate_by_quarter['Win Rate (units)'] = (
    win_rate_by_quarter['Won Count'] /
    (win_rate_by_quarter['Won Count'] + win_rate_by_quarter['Lost Count'])
).apply(lambda x: f"{x:.1%}" if pd.notna(x) else '—')
win_rate_by_quarter['Win Rate ($)'] = (
    win_rate_by_quarter['Won Value'] /
    (win_rate_by_quarter['Won Value'] + win_rate_by_quarter['Lost Value'])
).apply(lambda x: f"{x:.1%}" if pd.notna(x) else '—')

# Sort chronologically
win_rate_by_quarter = win_rate_by_quarter.sort_values('Quarter')
```

**Preview:** Win rates should be between 10–60% for a typical B2B pipeline. Values outside this range usually mean closed deals were not consistently logged as Won/Lost.

### Step 7: Sales cycle length

For Won deals, calculate the number of days from created_date to close_date.
This is the single most useful metric for forecasting and quota planning.

```python
if 'created_date' in df_mapped.columns:
    won_with_dates = won[won['created_date'].notna() & won['close_date'].notna()].copy()
    won_with_dates['sales_cycle_days'] = (
        won_with_dates['close_date'] - won_with_dates['created_date']
    ).dt.days
    won_with_dates = won_with_dates[won_with_dates['sales_cycle_days'] >= 0]

    sales_cycle = pd.DataFrame([{
        'Metric': 'Average Sales Cycle (Won deals)',
        'Value': f"{won_with_dates['sales_cycle_days'].mean():.0f} days"
    }, {
        'Metric': 'Median Sales Cycle (Won deals)',
        'Value': f"{won_with_dates['sales_cycle_days'].median():.0f} days"
    }, {
        'Metric': 'Fastest close',
        'Value': f"{won_with_dates['sales_cycle_days'].min():.0f} days"
    }, {
        'Metric': 'Slowest close',
        'Value': f"{won_with_dates['sales_cycle_days'].max():.0f} days"
    }])
else:
    sales_cycle = pd.DataFrame([{
        'Metric': 'Sales cycle',
        'Value': 'created_date column not found in source — cannot calculate'
    }])
```

**Preview:** Average sales cycle for B2B SaaS is typically 30–90 days. Enterprise is 90–180. Very short cycles (<7 days) often mean the deal was created in the system only at close.

### Step 8: Stage drop-off analysis

Where does the pipeline leak? This shows the absolute count and value lost
at each stage transition — the diagnostic tool for coaching conversations.

```python
stage_dropoff = []
for i in range(len(STAGE_ORDER) - 2):  # exclude Won and Lost from drop-off
    from_stage = STAGE_ORDER[i]
    to_stage = STAGE_ORDER[i + 1]
    from_count = len(df_mapped[df_mapped['stage_canonical'] == from_stage])
    to_count = len(df_mapped[df_mapped['stage_canonical'] == to_stage])
    from_value = df_mapped[df_mapped['stage_canonical'] == from_stage]['amount'].sum()
    to_value = df_mapped[df_mapped['stage_canonical'] == to_stage]['amount'].sum()
    stage_dropoff.append({
        'Transition': f"{from_stage} → {to_stage}",
        'Deals Lost': max(0, from_count - to_count),
        'Value Lost': max(0, from_value - to_value),
        'Drop-off Rate': f"{max(0, (from_count - to_count) / from_count):.1%}" if from_count > 0 else '—',
    })

stage_dropoff_df = pd.DataFrame(stage_dropoff)
```

**Preview:** The stage with the highest drop-off rate is the #1 coaching priority. Present this to sales leadership as a conversation starter, not a verdict.

### Step 9: Rep performance summary

Pipeline and won revenue by rep. One of the most politically sensitive tabs —
present facts, not judgements.

```python
rep_perf = df_mapped.groupby('owner').agg(
    total_deals=('opportunity_name', 'count'),
    pipeline_value=('amount', lambda x: x[df_mapped.loc[x.index, 'stage_canonical'].isin(
        ['Prospecting', 'Qualified', 'Demo', 'Proposal', 'Negotiation'])].sum()),
    won_deals=('stage_canonical', lambda x: (x == 'Won').sum()),
    won_value=('amount', lambda x: x[df_mapped.loc[x.index, 'stage_canonical'] == 'Won'].sum()),
    lost_deals=('stage_canonical', lambda x: (x == 'Lost').sum()),
).reset_index()

rep_perf.columns = ['Rep', 'Total Deals', 'Active Pipeline ($)', 'Deals Won', 'Revenue Won ($)', 'Deals Lost']
rep_perf['Win Rate'] = (
    rep_perf['Deals Won'] / (rep_perf['Deals Won'] + rep_perf['Deals Lost'])
).apply(lambda x: f"{x:.1%}" if pd.notna(x) and (rep_perf.loc[rep_perf['Win Rate'].isna() == False, 'Deals Won'] + rep_perf.loc[rep_perf['Win Rate'].isna() == False, 'Deals Lost']).sum() > 0 else '—')
rep_perf = rep_perf.sort_values('Revenue Won ($)', ascending=False)
```

**Preview:** Check that rep names are clean (no duplicates from capitalisation differences). Check that the total of "Revenue Won ($)" matches the Won aggregate from Step 4.

### Step 10: Build Exec Summary

Top-line KPIs for a board or exec update. One table, one page.

```python
import datetime

total_pipeline = active['amount'].sum()
total_won = won['amount'].sum()
total_lost = df_mapped[df_mapped['stage_canonical'] == 'Lost']['amount'].sum()
overall_win_rate_units = len(won) / (len(won) + len(df_mapped[df_mapped['stage_canonical'] == 'Lost'])) if (len(won) + len(df_mapped[df_mapped['stage_canonical'] == 'Lost'])) > 0 else 0

exec_summary = pd.DataFrame([
    ['Total Active Pipeline ($)', f"${total_pipeline:,.0f}"],
    ['Total Deals Won ($)', f"${total_won:,.0f}"],
    ['Overall Win Rate (units)', f"{overall_win_rate_units:.1%}"],
    ['Active Deal Count', f"{len(active):,}"],
    ['Stale Open Deals', f"{df_mapped['is_stale'].sum():,}"],
    ['Number of Reps', f"{df_mapped['owner'].nunique():,}"],
    ['', ''],
    ['Run Date', datetime.date.today().isoformat()],
    ['Source File', source_crm_export],
], columns=['Metric', 'Value'])
```

## Output Tabs

### Source Data

Raw input as loaded (after column name normalisation). Includes `stage_raw`, `stage_canonical`, and `is_stale` columns added by the recipe.

### Pipeline Conversion ($)

`pipeline_conversion_value` dataframe. One row per canonical stage. Shows deal count, total value, average deal size, and conversion rate vs the prior stage.

### Pipeline Conversion (units)

`pipeline_conversion_units` dataframe. Same structure as above but counting deals rather than value.

### Win Rate by Quarter

`win_rate_by_quarter` dataframe. One row per closed quarter. Won count, lost count, won value, lost value, win rate by units and by value. Sorted chronologically.

### Sales Cycle Length

`sales_cycle` dataframe. Average, median, min, max sales cycle in days for Won deals. Note: requires `created_date` column in source.

### Stage Drop-off

`stage_dropoff_df` dataframe. One row per stage transition. Shows deals lost, value lost, and drop-off rate at each transition.

### Rep Performance

`rep_perf` dataframe. One row per rep. Total deals, active pipeline value, deals won, revenue won, deals lost, win rate. Sorted by revenue won descending.

### Exec Summary

`exec_summary` dataframe. Top-line KPIs. Run date and source file name.

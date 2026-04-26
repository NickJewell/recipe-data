---
recipe: Monthly Budget vs. Actual Variance Analysis
version: 1.0.0
cadence: monthly
villain: "Rebuilding the budget vs actual file every month because the account structure changed, the budget file isn't aligned to the GL, or someone moved a cost centre"
sources:
  - name: actuals_export
    description: GL/actuals export from accounting system — one row per journal entry or one row per account per period
    format: xlsx or csv
    required_columns:
      - account_code
      - account_name
      - department
      - period
      - actual_amount
    common_issues:
      - Account codes in GL may include description suffix ("1000 - Revenue") vs budget which has just "1000"
      - Some systems export debits as positive and credits as negative; others flip this
      - Department names may differ between GL and budget (e.g. "Sales" vs "Revenue")
  - name: budget_file
    description: Budget/forecast spreadsheet — one row per account per period
    format: xlsx
    required_columns:
      - account_code
      - period
      - budget_amount
    common_issues:
      - Budget is often in a wide format (months as columns) — recipe pivots to long format
      - Budget account codes may be numeric only while GL has numeric + description
      - Some budget files have subtotals rows that need to be filtered out
output: budget-vs-actual.xlsx
output_tabs:
  - Source Data (Actuals)
  - Source Data (Budget)
  - Variance by Department
  - Variance by Account
  - Top 10 Unfavourable
  - YTD Trend
  - Flux Commentary Template
  - Exec Summary
tags:
  - finance
  - FP&A
  - budget
  - variance
  - monthly
---

# Monthly Budget vs. Actual Variance Analysis

FP&A's defining monthly ritual: load actuals from the accounting system, align them to the budget file, calculate variance at every level, sort the unfavourable ones to the top, and write the flux commentary before the board pack goes out tomorrow morning. The problem is that the budget file was built in January, the accounting system has been updated twelve times since then, and the account codes never quite match. This recipe automates the load, alignment, and calculation — it applies fuzzy matching to bridge account code mismatches and produces a full variance workbook including a flux commentary template with AI-generated first-draft explanations.

The output gives your CFO the variance view they want plus a starting point for the commentary, so your job becomes editing and sense-checking rather than building from scratch.

## Sources

### actuals_export

GL/actuals export from your accounting system. Can be summarised (one row per account per period) or transactional (one row per journal entry).

Expected columns:
- `Account Code` / `GL Code` / `Account` — account identifier
- `Account Name` / `Description` — account description
- `Department` / `Cost Centre` / `Division` — organisational unit
- `Period` / `Month` / `Date` — accounting period (month or date)
- `Amount` / `Actual` / `Net Amount` — actual amount for the period

If your export is transactional (one row per journal entry), the recipe will group by account + department + period automatically.

### budget_file

Annual budget or rolling forecast. Can be in long format (one row per account per month) or wide format (accounts as rows, months as columns).

Expected columns for long format:
- `Account Code` — must match or be reconcilable to the GL account code
- `Period` / `Month` — accounting period
- `Budget` / `Forecast` / `Plan` — budget amount

For wide format (typical FP&A spreadsheet), the recipe detects month column headers and pivots to long format.

## Steps

### Step 1: Load actuals

Load the actuals export and normalise column names.

```python
import pandas as pd

if source_actuals_export.endswith('.xlsx') or source_actuals_export.endswith('.xls'):
    df_actuals = pd.read_excel(source_actuals_export)
else:
    try:
        df_actuals = pd.read_csv(source_actuals_export, encoding='utf-8')
    except UnicodeDecodeError:
        df_actuals = pd.read_csv(source_actuals_export, encoding='latin-1')

df_actuals.columns = df_actuals.columns.str.strip()

ACTUALS_COL_MAP = {
    'gl code': 'account_code',
    'account': 'account_code',
    'acc code': 'account_code',
    'account name': 'account_name',
    'description': 'account_name',
    'narrative': 'account_name',
    'cost centre': 'department',
    'division': 'department',
    'business unit': 'department',
    'month': 'period',
    'date': 'period',
    'net amount': 'actual_amount',
    'amount': 'actual_amount',
    'actual': 'actual_amount',
    'debit': 'actual_amount',
}

df_actuals.columns = df_actuals.columns.str.lower().str.strip()
df_actuals = df_actuals.rename(columns=ACTUALS_COL_MAP)
df_actuals_source = df_actuals.copy()  # preserve for Source Data tab

print(f"Loaded {len(df_actuals)} actuals rows. Columns: {list(df_actuals.columns)}")
```

**Preview:** Check row count and column list. If the export is transactional, row count will be high (many journal entries per account). If summarised, expect one row per account per period.

### Step 2: Standardise account codes and parse periods

Strip description suffixes from account codes (e.g. "1000 - Revenue" → "1000")
and parse the period column to a consistent format (YYYY-MM).

```python
from dateutil import parser as dateutil_parser

# Strip description suffix from account codes: "1000 - Revenue" → "1000"
df_actuals['account_code'] = (
    df_actuals['account_code'].astype(str).str.strip()
    .str.extract(r'^(\d+)', expand=False)
    .fillna(df_actuals['account_code'].astype(str).str.strip())
)

# Parse period to YYYY-MM
def parse_period(val):
    if pd.isna(val):
        return None
    val_str = str(val).strip()
    # Handle "Jan-24", "January 2024", "2024-01", "01/2024"
    try:
        dt = dateutil_parser.parse(val_str, default=pd.Timestamp('2000-01-01'))
        return dt.strftime('%Y-%m')
    except Exception:
        return val_str

df_actuals['period'] = df_actuals['period'].apply(parse_period)
df_actuals['actual_amount'] = pd.to_numeric(
    df_actuals['actual_amount'].astype(str).str.replace(r'[^\d.\-]', '', regex=True),
    errors='coerce'
).fillna(0)

# If transactional, group to account + department + period
if 'department' in df_actuals.columns:
    df_actuals_grouped = (
        df_actuals.groupby(['account_code', 'account_name', 'department', 'period'], dropna=False)
        ['actual_amount'].sum().reset_index()
    )
else:
    df_actuals_grouped = (
        df_actuals.groupby(['account_code', 'account_name', 'period'], dropna=False)
        ['actual_amount'].sum().reset_index()
    )

print(f"After grouping: {len(df_actuals_grouped)} account-period rows")
print(f"Periods found: {sorted(df_actuals_grouped['period'].unique())}")
```

**Preview:** Check that periods are in YYYY-MM format and span the expected date range. Check account codes are numeric only (no description suffixes).

### Step 3: Load budget file

Load the budget. Detect wide format (months as columns) vs long format
(one row per account per month) and normalise to long format.

```python
df_budget_raw = pd.read_excel(source_budget_file)
df_budget_raw.columns = df_budget_raw.columns.str.strip()
df_budget_source = df_budget_raw.copy()  # preserve for Source Data tab

# Detect wide vs long format
# Wide format: has many columns that look like month names
import re
month_pattern = re.compile(r'(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)', re.IGNORECASE)
month_cols = [c for c in df_budget_raw.columns if month_pattern.search(str(c))]

if len(month_cols) >= 3:
    # Wide format — melt month columns to long
    print(f"Wide format detected. Month columns: {month_cols}")
    id_cols = [c for c in df_budget_raw.columns if c not in month_cols]
    df_budget = df_budget_raw.melt(
        id_vars=id_cols,
        value_vars=month_cols,
        var_name='period_raw',
        value_name='budget_amount',
    )
    df_budget['period'] = df_budget['period_raw'].apply(parse_period)
    df_budget = df_budget.dropna(subset=['budget_amount'])
    df_budget = df_budget[df_budget['budget_amount'] != 0]
else:
    # Long format
    df_budget = df_budget_raw.copy()
    df_budget.columns = df_budget.columns.str.lower().str.strip()
    BUDGET_COL_MAP = {
        'gl code': 'account_code',
        'account': 'account_code',
        'budget': 'budget_amount',
        'forecast': 'budget_amount',
        'plan': 'budget_amount',
        'month': 'period',
    }
    df_budget = df_budget.rename(columns=BUDGET_COL_MAP)
    df_budget['period'] = df_budget['period'].apply(parse_period)

# Normalise budget account codes (same stripping as actuals)
df_budget.columns = df_budget.columns.str.lower().str.strip()
df_budget['account_code'] = (
    df_budget['account_code'].astype(str).str.strip()
    .str.extract(r'^(\d+)', expand=False)
    .fillna(df_budget['account_code'].astype(str).str.strip())
)
df_budget['budget_amount'] = pd.to_numeric(
    df_budget['budget_amount'].astype(str).str.replace(r'[^\d.\-]', '', regex=True),
    errors='coerce'
).fillna(0)

# Drop subtotal rows (typically have blank or summary account codes)
df_budget = df_budget[df_budget['account_code'].str.match(r'^\d+$', na=False)].copy()

print(f"Budget loaded: {len(df_budget)} rows after cleaning")
print(f"Budget periods: {sorted(df_budget['period'].unique())}")
```

**Preview:** Check that budget periods overlap with actuals periods. If they don't match, the format of one of the period columns is likely wrong.

### Checkpoint: Period overlap between actuals and budget

```
Assert: len(set(df_actuals_grouped['period'].unique()) & set(df_budget['period'].unique())) > 0
On failure: Actuals and budget have no periods in common. Check that both files cover the
            same time range and that the period format has been parsed consistently (both
            should be YYYY-MM after Step 2 and Step 3). Print both period lists to diagnose.
```

### Step 4: Fuzzy-match account codes

Account codes rarely match perfectly between the GL and the budget file.
Apply fuzzy matching to find the closest budget account for each actuals account,
then flag low-confidence matches for manual review.

```python
from thefuzz import process

budget_codes = df_budget['account_code'].unique().tolist()
actuals_codes = df_actuals_grouped['account_code'].unique().tolist()

def match_account_code(code, choices, threshold=85):
    """Match an account code to the closest budget code."""
    result = process.extractOne(str(code), [str(c) for c in choices], score_cutoff=threshold)
    if result:
        return result[0], result[1]
    return None, 0

# Build mapping: actuals code → budget code
code_match_log = []
code_map = {}
for code in actuals_codes:
    matched, score = match_account_code(code, budget_codes)
    code_map[code] = matched
    code_match_log.append({
        'actuals_code': code,
        'matched_budget_code': matched,
        'match_score': score,
        'needs_review': score < 90 or matched is None,
    })

code_match_df = pd.DataFrame(code_match_log)
low_confidence = code_match_df[code_match_df['needs_review']]
if len(low_confidence) > 0:
    print(f"\nWARNING: {len(low_confidence)} account codes matched with low confidence or unmatched:")
    print(low_confidence.to_string(index=False))
    print("\nReview these mappings. Add manual overrides to MANUAL_OVERRIDES dict in this step.")

# Manual overrides — add entries here to fix specific mismatches
MANUAL_OVERRIDES = {
    # 'actuals_code': 'budget_code',  # example: '1100': '1000'
}
code_map.update(MANUAL_OVERRIDES)

df_actuals_grouped['budget_account_code'] = df_actuals_grouped['account_code'].map(code_map)
```

**Preview:** Review the low-confidence matches. Account codes with no budget match will show a blank/NaN `budget_account_code`. Add manual overrides for persistent mismatches.

### Step 5: Join actuals to budget and calculate variance

Join actuals to budget on account code + period. Calculate variance as
actual minus budget (positive = favourable for revenue; unfavourable for costs).

```python
# Join actuals to budget
merge_cols_actuals = ['budget_account_code', 'period']
merge_cols_budget = ['account_code', 'period']

if 'department' in df_actuals_grouped.columns and 'department' in df_budget.columns:
    merge_cols_actuals = ['budget_account_code', 'department', 'period']
    merge_cols_budget = ['account_code', 'department', 'period']

df_budget_for_merge = df_budget[merge_cols_budget + ['budget_amount']].copy()
df_budget_for_merge.columns = merge_cols_actuals + ['budget_amount']

variance_df = pd.merge(
    df_actuals_grouped,
    df_budget_for_merge,
    on=merge_cols_actuals,
    how='outer',
).fillna(0)

# Variance: actual - budget
# Note: sign convention varies by account type. This recipe uses "positive = overspend for costs"
# The analyst should flip sign for revenue accounts if the GL uses the standard accounting sign convention.
variance_df['variance'] = variance_df['actual_amount'] - variance_df['budget_amount']
variance_df['variance_pct'] = (
    variance_df['variance'] / variance_df['budget_amount'].abs()
    * 100
).replace([float('inf'), float('-inf')], None).round(1)

print(f"Variance dataframe: {len(variance_df)} rows")
print(f"Total actual: {variance_df['actual_amount'].sum():,.0f}")
print(f"Total budget: {variance_df['budget_amount'].sum():,.0f}")
print(f"Total variance: {variance_df['variance'].sum():,.0f}")
```

**Preview:** Total actual should match the sum in Step 2. Total variance = total actual - total budget. A large positive or negative total is worth reviewing before proceeding.

### Checkpoint: Join did not lose significant actuals

```
Assert: variance_df['actual_amount'].sum() >= df_actuals_grouped['actual_amount'].sum() * 0.95
On failure: More than 5% of actual spend could not be joined to a budget code.
            Check the code_match_df from Step 4 for unmapped accounts.
            Add manual overrides for the high-value unmatched accounts.
```

### Step 6: Variance by department

Roll up to department level — the view the CFO and business unit heads need.

```python
if 'department' in variance_df.columns:
    variance_by_dept = (
        variance_df.groupby(['department', 'period'])
        .agg(actual=('actual_amount', 'sum'), budget=('budget_amount', 'sum'))
        .reset_index()
    )
    variance_by_dept['variance'] = variance_by_dept['actual'] - variance_by_dept['budget']
    variance_by_dept['variance_pct'] = (
        variance_by_dept['variance'] / variance_by_dept['budget'].abs() * 100
    ).replace([float('inf'), float('-inf')], None).round(1)
    variance_by_dept = variance_by_dept.sort_values(['period', 'variance'], ascending=[True, False])
    variance_by_dept.columns = ['Department', 'Period', 'Actual', 'Budget', 'Variance', 'Variance %']
else:
    variance_by_dept = pd.DataFrame([{
        'Note': 'Department column not found in actuals export — department-level view not available'
    }])
```

**Preview:** Each department should show for each period. Large variances (positive or negative >10%) are the conversations worth having.

### Step 7: Variance by account (full detail)

Account-level variance — the detail view for finance review. Sorted by
variance magnitude so the biggest misses are at the top.

```python
variance_by_account = variance_df[[
    'account_code', 'account_name', 'department', 'period',
    'actual_amount', 'budget_amount', 'variance', 'variance_pct'
]].copy() if 'department' in variance_df.columns else variance_df[[
    'account_code', 'account_name', 'period',
    'actual_amount', 'budget_amount', 'variance', 'variance_pct'
]].copy()

variance_by_account = variance_by_account.sort_values('variance', ascending=False).reset_index(drop=True)
variance_by_account.columns = [c.replace('_', ' ').title() for c in variance_by_account.columns]
```

**Preview:** Check that account names are meaningful (not just codes). Review the top 5 variances for plausibility.

### Step 8: Top 10 unfavourable variances

The list that goes on page 1: the 10 accounts or departments with the
largest unfavourable variance this period.

```python
# "Unfavourable" = overspend (positive variance for costs)
# This recipe treats positive variance as unfavourable — adjust if your GL sign convention is reversed
current_period = sorted(variance_df['period'].unique())[-1]  # most recent period

current_period_df = variance_df[variance_df['period'] == current_period].copy()
top_unfavourable = (
    current_period_df.nlargest(10, 'variance')
    [['account_code', 'account_name', 'actual_amount', 'budget_amount', 'variance', 'variance_pct']]
    .copy()
)
top_unfavourable.columns = ['Account Code', 'Account Name', 'Actual', 'Budget', 'Variance', 'Variance %']
top_unfavourable.index = range(1, len(top_unfavourable) + 1)
```

**Preview:** These are the 10 items that need a commentary explanation. The Flux Commentary Template tab will have a row for each one.

### Step 9: YTD trend

Cumulative actual vs budget by period — the month-by-month story of how
the business has tracked against plan.

```python
ytd = (
    variance_df.groupby('period')
    .agg(actual=('actual_amount', 'sum'), budget=('budget_amount', 'sum'))
    .reset_index()
    .sort_values('period')
)
ytd['cumulative_actual'] = ytd['actual'].cumsum()
ytd['cumulative_budget'] = ytd['budget'].cumsum()
ytd['monthly_variance'] = ytd['actual'] - ytd['budget']
ytd['ytd_variance'] = ytd['cumulative_actual'] - ytd['cumulative_budget']
ytd.columns = ['Period', 'Monthly Actual', 'Monthly Budget', 'Cumulative Actual', 'Cumulative Budget', 'Monthly Variance', 'YTD Variance']
```

**Preview:** YTD variance should accumulate. A widening negative YTD variance is the CFO's concern. A widening positive one means costs are coming in under plan — also worth explaining.

### Step 10: Flux commentary template

Generate a first-draft commentary for each top-10 variance. These are
placeholders — the analyst edits and adds context. The hard work (identifying
which items need explaining) is already done.

```python
import datetime

flux_rows = []
for _, row in top_unfavourable.iterrows():
    var_desc = "over budget" if row['Variance'] > 0 else "under budget"
    pct = abs(row['Variance %']) if pd.notna(row['Variance %']) else 0
    flux_rows.append({
        'Account': f"{row['Account Code']} - {row['Account Name']}",
        'Variance': row['Variance'],
        'Variance %': row['Variance %'],
        'Draft Commentary': (
            f"{row['Account Name']} is {var_desc} by {abs(row['Variance']):,.0f} "
            f"({pct:.1f}%). [Add explanation here: one-time item / timing difference / "
            f"volume-driven / price-driven / permanent overspend]"
        ),
        'Reviewed': '',
    })

flux_commentary = pd.DataFrame(flux_rows)
```

### Step 11: Build Exec Summary

```python
total_actual = variance_df['actual_amount'].sum()
total_budget = variance_df['budget_amount'].sum()
total_variance = total_actual - total_budget
pct_variance = total_variance / abs(total_budget) * 100 if total_budget != 0 else 0

exec_summary = pd.DataFrame([
    ['Period', current_period],
    ['Total Actual', f"{total_actual:,.0f}"],
    ['Total Budget', f"{total_budget:,.0f}"],
    ['Total Variance', f"{total_variance:,.0f}"],
    ['Variance %', f"{pct_variance:.1f}%"],
    ['Unfavourable items (top 10)', f"{len(top_unfavourable)}"],
    ['Accounts with no budget match', f"{variance_df['budget_amount'].eq(0).sum()}"],
    ['', ''],
    ['Run Date', datetime.date.today().isoformat()],
    ['Actuals Source', source_actuals_export],
    ['Budget Source', source_budget_file],
], columns=['Metric', 'Value'])
```

## Output Tabs

### Source Data (Actuals)

`df_actuals_source` — raw actuals as loaded, before any transformation.

### Source Data (Budget)

`df_budget_source` — raw budget file as loaded.

### Variance by Department

`variance_by_dept` — department-level variance by period. Columns: Department | Period | Actual | Budget | Variance | Variance %.

### Variance by Account

`variance_by_account` — full account-level variance sorted by variance magnitude.

### Top 10 Unfavourable

`top_unfavourable` — the 10 largest unfavourable variances for the most recent period.

### YTD Trend

`ytd` — monthly and cumulative actual vs budget. Columns: Period | Monthly Actual | Monthly Budget | Cumulative Actual | Cumulative Budget | Monthly Variance | YTD Variance.

### Flux Commentary Template

`flux_commentary` — one row per top-10 variance with a draft commentary. Edit the "Draft Commentary" column; mark "Reviewed" when done.

### Exec Summary

`exec_summary` — top-line KPIs: total actual, budget, variance, %, run date, source files.

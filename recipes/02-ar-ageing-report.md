---
recipe: Monthly AR Ageing Report
version: 1.0.0
cadence: monthly
villain: "Exporting from Xero, pasting into a pivot table, manually bucketing by due date, then reformatting — every single month"
sources:
  - name: ar_export
    description: Debtors/AR export from accounting system — one row per open invoice
    format: xlsx or csv
    required_columns:
      - invoice_date
      - due_date
      - amount
      - customer
      - invoice_number
    common_issues:
      - Xero uses "Due Date"; NetSuite uses "Due Date/Time" (includes time — strip it)
      - QuickBooks exports amount as string with currency symbol ("$1,234.56")
      - Sage 50 exports credit notes as negative amounts — filter these out
      - Column names vary significantly across accounting systems
output: ar-ageing-report.xlsx
output_tabs:
  - Source Data
  - Ageing Buckets
  - Top 20 Debtors
  - Ageing by Customer
  - Month-over-Month Trend
  - Collection Priority
  - Exec Summary
tags:
  - finance
  - AR
  - accounts-receivable
  - monthly
  - FP&A
---

# Monthly AR Ageing Report

Every finance team runs AR ageing every month. The standard approach is: export from the accounting system, paste into Excel, manually calculate days outstanding, bucket by due date, pivot by customer, format, and email to the CFO. The whole thing takes 45–90 minutes and has to be rebuilt from scratch every time because the source export format changes slightly or someone adds a new customer. This recipe automates the entire sequence — load the AR export, normalise the schema regardless of which accounting system it came from, calculate days outstanding, bucket into standard ageing brackets, identify collection priorities, and write a board-ready workbook.

The output is the answer to the question your CFO asks every month: "How much do we have outstanding, who owes us the most, and what's overdue by how long?"

## Sources

### ar_export

Open invoices export from your accounting system. One row per open invoice.

Tested against: Xero, QuickBooks Online, NetSuite, Sage 50, FreeAgent, Wave.

Expected columns (exact names vary by system — the recipe will ask you to map if they differ):
- `Invoice Date` / `InvoiceDate` / `Date` — date the invoice was raised
- `Due Date` / `DueDate` / `Payment Due` / `Due Date/Time` — date payment is due
- `Amount` / `Outstanding` / `Balance Due` / `Remaining` — amount owed (open balance, not original invoice value)
- `Customer` / `Customer Name` / `Contact` / `Client` — debtor name
- `Invoice Number` / `Invoice No` / `Reference` / `Inv #` — unique invoice identifier

Common issues by system:
- **Xero:** Exports all invoices (paid and unpaid) — filter by "Status = AUTHORISED". Due Date column is clean.
- **QuickBooks:** Amount column contains currency symbol and commas ("$1,234.56") — recipe strips these. Sometimes exports in PDF layout with merged cells — export as CSV not Excel.
- **NetSuite:** "Due Date/Time" includes timestamp ("2024-01-31 00:00:00") — recipe strips the time component.
- **Sage 50:** Includes credit notes as negative amounts — recipe filters amount > 0. Date format is often DD/MM/YYYY — handled by dateutil.

## Steps

### Step 1: Load the AR export

Load the file and do an initial inspection — column names, row count, data types.
Do not transform anything yet; this step is purely diagnostic.

```python
import pandas as pd

if source_ar_export.endswith('.xlsx') or source_ar_export.endswith('.xls'):
    df_raw = pd.read_excel(source_ar_export)
else:
    # Try UTF-8 first, fall back to latin-1 for Windows-exported CSVs
    try:
        df_raw = pd.read_csv(source_ar_export, encoding='utf-8')
    except UnicodeDecodeError:
        df_raw = pd.read_csv(source_ar_export, encoding='latin-1')

# Strip whitespace from column names
df_raw.columns = df_raw.columns.str.strip()

print(f"Loaded {len(df_raw)} rows and {len(df_raw.columns)} columns")
print(f"Columns: {list(df_raw.columns)}")
print(df_raw.head(3).to_string())
```

**Preview:** Check that the row count matches your accounting system's open invoice count. Review the column list to identify which columns map to invoice_date, due_date, amount, customer, invoice_number.

### Step 2: Normalise column names

Map the accounting-system-specific column names to standard names.
Update the COLUMN_MAP below if your column names differ from the defaults.

```python
# Map known variants to standard names
COLUMN_MAP = {
    # Invoice date variants
    'invoicedate': 'invoice_date',
    'date': 'invoice_date',
    'invoice date': 'invoice_date',
    'txn date': 'invoice_date',
    'transaction date': 'invoice_date',
    # Due date variants
    'due date': 'due_date',
    'duedate': 'due_date',
    'payment due': 'due_date',
    'due date/time': 'due_date',
    'terms due': 'due_date',
    # Amount variants
    'outstanding': 'amount',
    'balance due': 'amount',
    'remaining': 'amount',
    'open balance': 'amount',
    'amount due': 'amount',
    'balance': 'amount',
    # Customer variants
    'customer name': 'customer',
    'contact': 'customer',
    'client': 'customer',
    'debtor': 'customer',
    'customer/job': 'customer',
    # Invoice number variants
    'invoice no': 'invoice_number',
    'invoice no.': 'invoice_number',
    'invoice #': 'invoice_number',
    'inv #': 'invoice_number',
    'reference': 'invoice_number',
    'ref': 'invoice_number',
    'doc no.': 'invoice_number',
}

# Normalise to lowercase for matching, then rename
df = df_raw.copy()
df.columns = df.columns.str.lower().str.strip()
df = df.rename(columns=COLUMN_MAP)

required = ['invoice_date', 'due_date', 'amount', 'customer', 'invoice_number']
missing = [c for c in required if c not in df.columns]
if missing:
    print(f"MISSING COLUMNS: {missing}")
    print(f"Available columns: {list(df.columns)}")
    raise ValueError(f"Cannot proceed — please map the missing columns in COLUMN_MAP above")
```

**Preview:** Confirm that all 5 required columns are present after mapping. If any are missing, the step will halt with a list of available columns to help you update the map.

### Step 3: Parse dates and coerce amounts

Convert date columns to proper datetimes and amount to numeric.
Strip currency symbols and commas from the amount column.

```python
from dateutil import parser as dateutil_parser

def parse_date_safe(val):
    """Parse a date from any reasonable format, including DD/MM/YYYY and YYYY-MM-DD HH:MM:SS."""
    if pd.isna(val) or str(val).strip() in ('', 'nan'):
        return pd.NaT
    # Strip timestamp if present (NetSuite exports "2024-01-31 00:00:00")
    val_str = str(val).strip().split(' ')[0]
    try:
        return pd.Timestamp(dateutil_parser.parse(val_str, dayfirst=True))
    except Exception:
        return pd.NaT

df['invoice_date'] = df['invoice_date'].apply(parse_date_safe)
df['due_date'] = df['due_date'].apply(parse_date_safe)

# Coerce amount — handles "$1,234.56", "£1,234", "1,234.56", "(1,234.56)" for credits
df['amount_raw'] = df['amount'].copy()
df['amount'] = (
    df['amount'].astype(str)
    .str.replace(r'[^\d.\-]', '', regex=True)  # strip currency symbols, commas
    .str.replace(r'^\((.*)\)$', r'-\1', regex=True)  # convert (1234) to -1234 for credits
)
df['amount'] = pd.to_numeric(df['amount'], errors='coerce').fillna(0)

# Filter: keep only open invoices with a positive balance
df = df[df['amount'] > 0].copy()
df['customer'] = df['customer'].astype(str).str.strip()
df['invoice_number'] = df['invoice_number'].astype(str).str.strip()

print(f"After filtering to positive balances: {len(df)} open invoices")
print(f"Total AR outstanding: {df['amount'].sum():,.2f}")
```

**Preview:** Check that row count after filtering matches your accounting system's open invoice count. Confirm total AR outstanding matches your system's AR balance.

### Checkpoint: No null due dates

```
Assert: df['due_date'].notna().all()
On failure: Some invoices have no due date. Show the invoices where due_date is null.
            Common cause: draft invoices included in export, or invoices set to "no due date".
            Either fix the source export (filter to Authorised/Approved status only)
            or fill null due dates with invoice_date + standard_terms if you know the payment terms.
```

### Checkpoint: No null amounts

```
Assert: df['amount'].notna().all() and (df['amount'] > 0).all()
On failure: After filtering, some amounts are still null or zero. Show the failing rows.
            Check the amount_raw column to see the original value before parsing.
```

### Step 4: Calculate days outstanding and ageing buckets

For each invoice, calculate how many days it has been outstanding (relative to today)
and assign it to one of 5 standard AR ageing buckets.

```python
today = pd.Timestamp.today().normalize()

# Days outstanding = today minus due_date (positive = overdue)
df['days_outstanding'] = (today - df['due_date']).dt.days

def ageing_bucket(days):
    if days <= 0:
        return 'Current'
    elif days <= 30:
        return '1-30 days'
    elif days <= 60:
        return '31-60 days'
    elif days <= 90:
        return '61-90 days'
    else:
        return '90+ days'

BUCKET_ORDER = ['Current', '1-30 days', '31-60 days', '61-90 days', '90+ days']
df['ageing_bucket'] = df['days_outstanding'].apply(ageing_bucket)
df['ageing_bucket'] = pd.Categorical(df['ageing_bucket'], categories=BUCKET_ORDER, ordered=True)
```

**Preview:** Check the distribution of ageing buckets. A healthy AR book has the majority in "Current". High 90+ balance is the red flag to highlight.

### Checkpoint: All invoices bucketed

```
Assert: df['ageing_bucket'].notna().all()
On failure: Some invoices could not be bucketed — likely because due_date is null (caught
            by the earlier checkpoint). Should not reach here if the prior checkpoint passed.
```

### Step 5: Build ageing buckets summary table

Aggregate by bucket to produce the headline AR summary — the table that goes
on page 1 of the board pack.

```python
ageing_summary = (
    df.groupby('ageing_bucket', observed=True)
    .agg(
        invoice_count=('invoice_number', 'count'),
        total_amount=('amount', 'sum'),
    )
    .reset_index()
)
ageing_summary.columns = ['Ageing Bucket', 'Invoice Count', 'Total Amount']
total_ar = ageing_summary['Total Amount'].sum()
ageing_summary['% of Total AR'] = (ageing_summary['Total Amount'] / total_ar * 100).round(1).astype(str) + '%'
ageing_summary['Total Amount'] = ageing_summary['Total Amount'].round(2)

# Add totals row
totals_row = pd.DataFrame([{
    'Ageing Bucket': 'TOTAL',
    'Invoice Count': ageing_summary['Invoice Count'].sum(),
    'Total Amount': total_ar.round(2),
    '% of Total AR': '100%',
}])
ageing_buckets_tab = pd.concat([ageing_summary, totals_row], ignore_index=True)
```

**Preview:** TOTAL row should match total AR outstanding from Step 3.

### Step 6: Top 20 debtors

Rank customers by total outstanding. This is the collection priority shortlist —
the 20 customers that, if they paid, would clear the majority of outstanding AR.

```python
top_debtors = (
    df.groupby('customer')
    .agg(
        invoice_count=('invoice_number', 'count'),
        total_outstanding=('amount', 'sum'),
        oldest_invoice_date=('invoice_date', 'min'),
        oldest_due_date=('due_date', 'min'),
        primary_bucket=('ageing_bucket', lambda x: x.value_counts().index[0]),
    )
    .reset_index()
    .sort_values('total_outstanding', ascending=False)
    .head(20)
    .reset_index(drop=True)
)
top_debtors.columns = ['Customer', 'Invoice Count', 'Total Outstanding', 'Oldest Invoice Date', 'Oldest Due Date', 'Primary Bucket']
top_debtors['Total Outstanding'] = top_debtors['Total Outstanding'].round(2)
top_debtors.index = range(1, len(top_debtors) + 1)
```

**Preview:** Confirm that the top debtor names look correct and totals are plausible.

### Step 7: Ageing by customer (full breakdown)

Full customer-level breakdown showing how much each customer owes in each
ageing bucket. This is what the collections team works from.

```python
ageing_by_customer = (
    df.pivot_table(
        index='customer',
        columns='ageing_bucket',
        values='amount',
        aggfunc='sum',
        fill_value=0,
        observed=True,
    )
    .reset_index()
)
ageing_by_customer.columns.name = None

# Ensure all bucket columns present even if empty
for bucket in BUCKET_ORDER:
    if bucket not in ageing_by_customer.columns:
        ageing_by_customer[bucket] = 0.0

ageing_by_customer = ageing_by_customer[['customer'] + BUCKET_ORDER].copy()
ageing_by_customer.columns = ['Customer'] + BUCKET_ORDER
ageing_by_customer['Total'] = ageing_by_customer[BUCKET_ORDER].sum(axis=1)
ageing_by_customer = ageing_by_customer.sort_values('Total', ascending=False).reset_index(drop=True)
```

**Preview:** Row count should equal number of unique customers. Total column should sum to total AR.

### Step 8: Month-over-month trend placeholder

If prior month data is available, calculate MoM movement. If not, add
a placeholder that explains how to supply the prior month file.

```python
# MoM trend requires a prior-month AR export bound as `ar_export_prior_month`
# If you have it, bind it as a second source in the recipe frontmatter.
# For now, this tab shows a note explaining what data is needed.

mom_trend = pd.DataFrame([
    ['Month-over-Month trend', ''],
    ['To enable MoM trend:', 'Add a second source (ar_export_prior_month) to the recipe frontmatter'],
    ['and rerun. The recipe will calculate movement per customer per bucket.', ''],
    ['', ''],
    ['Current month total AR', f"{df['amount'].sum():,.2f}"],
    ['Run date', today.strftime('%Y-%m-%d')],
], columns=['Note', 'Value'])
```

**Preview:** This tab is a placeholder. To enable MoM trend, add the prior month AR export as a second source.

### Step 9: Collection priority list

Rank open invoices by urgency: overdue first, largest balance first within each
bucket. This is the list a collections analyst works through in order.

```python
overdue = df[df['ageing_bucket'] != 'Current'].copy()
overdue_sorted = overdue.sort_values(
    ['ageing_bucket', 'amount'],
    ascending=[False, False]  # worst bucket first, largest amount first
).reset_index(drop=True)

collection_priority = overdue_sorted[[
    'customer', 'invoice_number', 'invoice_date', 'due_date',
    'amount', 'days_outstanding', 'ageing_bucket'
]].copy()
collection_priority.columns = [
    'Customer', 'Invoice Number', 'Invoice Date', 'Due Date',
    'Amount Outstanding', 'Days Overdue', 'Ageing Bucket'
]
collection_priority.index = range(1, len(collection_priority) + 1)
```

**Preview:** First row should be the oldest/largest overdue invoice. Confirm that "Current" invoices are excluded.

### Step 10: Build Exec Summary

```python
import datetime

overdue_amount = df[df['ageing_bucket'] != 'Current']['amount'].sum()
total_ar = df['amount'].sum()
pct_overdue = overdue_amount / total_ar * 100 if total_ar > 0 else 0
largest_debtor = top_debtors.iloc[0]['Customer'] if len(top_debtors) > 0 else 'N/A'
largest_debtor_amount = top_debtors.iloc[0]['Total Outstanding'] if len(top_debtors) > 0 else 0

exec_summary = pd.DataFrame([
    ['Total AR Outstanding', f"{total_ar:,.2f}"],
    ['Number of Open Invoices', f"{len(df):,}"],
    ['Number of Debtors', f"{df['customer'].nunique():,}"],
    ['% Overdue (1-30 days or more)', f"{pct_overdue:.1f}%"],
    ['90+ Days Outstanding', f"{df[df['ageing_bucket'] == '90+ days']['amount'].sum():,.2f}"],
    ['Largest Single Debtor', largest_debtor],
    ['Largest Debtor Balance', f"{largest_debtor_amount:,.2f}"],
    ['', ''],
    ['Run Date', datetime.date.today().isoformat()],
    ['Source File', source_ar_export],
], columns=['Metric', 'Value'])
```

## Output Tabs

### Source Data

Raw input as loaded (before any transformation). Includes `days_outstanding` and `ageing_bucket` columns added by the recipe.

### Ageing Buckets

`ageing_buckets_tab` dataframe. One row per ageing bucket plus a TOTAL row. Columns: Ageing Bucket | Invoice Count | Total Amount | % of Total AR.

### Top 20 Debtors

`top_debtors` dataframe. Top 20 customers by total outstanding. Columns: Customer | Invoice Count | Total Outstanding | Oldest Invoice Date | Oldest Due Date | Primary Bucket. Sorted by outstanding descending.

### Ageing by Customer

`ageing_by_customer` dataframe. Full customer breakdown. One row per customer, one column per ageing bucket, plus Total. Sorted by Total descending.

### Month-over-Month Trend

Placeholder if prior month data not provided. Bind a second source (`ar_export_prior_month`) to enable MoM movement calculation per customer per bucket.

### Collection Priority

`collection_priority` dataframe. All overdue invoices ranked by urgency (worst bucket first, largest amount first). Columns: Customer | Invoice Number | Invoice Date | Due Date | Amount Outstanding | Days Overdue | Ageing Bucket.

### Exec Summary

`exec_summary` dataframe. Top-line KPIs: total AR, open invoice count, debtor count, % overdue, 90+ balance, largest debtor, run date, source file.

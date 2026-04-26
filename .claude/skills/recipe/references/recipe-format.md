# Recipe Format Specification

A recipe is a plain markdown file. It has a YAML frontmatter block and a structured markdown body. Nothing proprietary. Opens in any editor. Diffs in Git.

---

## Frontmatter

```yaml
---
recipe: Human-Readable Recipe Name
version: 1.0.0
cadence: monthly          # monthly | quarterly | weekly | ad-hoc
villain: "The specific painful thing this replaces, in plain English"
sources:
  - name: snake_case_identifier    # used to refer to this source in steps
    description: What this file is and where it comes from
    format: xlsx                    # xlsx | csv | tsv
    required_columns:
      - column_name_1
      - column_name_2
    common_issues:
      - Trailing whitespace in customer names
      - Mixed date formats (DD/MM/YYYY and YYYY-MM-DD)
      - ALL CAPS in some exports
output: output-filename.xlsx
output_tabs:
  - Source Data
  - Tab Two Name
  - Tab Three Name
  - Exec Summary
tags:
  - finance
  - monthly
  - FP&A
---
```

### Frontmatter field reference

| Field | Required | Notes |
|-------|----------|-------|
| `recipe` | Yes | Human-readable name. This is the display name. |
| `version` | Yes | Semver. Increment minor for new steps, patch for bug fixes. |
| `cadence` | Yes | How often this runs. Sets user expectations. |
| `villain` | Yes | The specific painful thing this replaces. One sentence. Make it punchy. |
| `sources` | Yes | One entry per input file. |
| `sources[].name` | Yes | Snake_case identifier used in step code to refer to this dataframe. |
| `sources[].required_columns` | Recommended | Column names the recipe depends on. Used for schema validation before step 1. |
| `sources[].common_issues` | Recommended | Known quirks in real-world exports. Tells the analyst what to expect. |
| `output` | Yes | Filename for the output workbook. |
| `output_tabs` | Yes | Ordered list of worksheet names. Must include "Source Data" and "Exec Summary". |
| `tags` | Recommended | For search and filtering in the recipe hub. |

---

## Body Structure

The body has five sections in this order. Do not reorder them.

### 1. Description (required)

One paragraph. What this recipe does and why it matters. Name the villain. Name the analyst persona who benefits.

```markdown
Every finance team runs AR ageing every month. The standard approach is: export from Xero, 
paste into a pivot table, manually bucket by due date, format, send. This recipe automates 
that entire sequence — load the AR export, calculate days outstanding, bucket into standard 
ageing brackets, identify collection priorities, and write a board-ready output workbook.
```

### 2. Sources (required)

One `###` subsection per source. Describe the expected schema, common column name variants, and known quirks.

```markdown
## Sources

### ar_export

Debtors/AR export from your accounting system. One row per open invoice.

Tested against: Xero, QuickBooks Online, NetSuite, Sage 50.

Expected columns (exact names vary by system — the recipe will ask you to map if they differ):
- `Invoice Date` / `InvoiceDate` / `Date` — date the invoice was raised
- `Due Date` / `DueDate` / `Payment Due` — date payment is expected
- `Amount` / `Outstanding` / `Balance Due` — amount owed (in document currency)
- `Customer` / `Customer Name` / `Contact` — debtor name
- `Invoice Number` / `Reference` — unique invoice ID

Common issues:
- Xero exports use "Due Date"; NetSuite uses "Due Date/Time" (includes time component — strip it)
- QuickBooks sometimes exports amount as a string with currency symbol ("$1,234.56") — the recipe strips non-numeric characters
- Sage 50 exports "Credit" rows as negative amounts — the recipe filters these out
```

### 3. Steps (required)

One `###` subsection per step. Each step has exactly:
1. A plain-English description (1–3 sentences)
2. A fenced Python code block
3. A `**Preview:**` line describing what to check in the output

Steps are numbered sequentially: `### Step 1: Name`, `### Step 2: Name`, etc.

Checkpoints are inline between steps where needed (see below).

```markdown
## Steps

### Step 1: Load and standardise the AR export

Load the source file, strip column name whitespace, coerce amount to numeric, 
and remove any credit/reversal rows (negative balances).

```python
import pandas as pd

df = pd.read_excel(source_ar_export) if source_ar_export.endswith('.xlsx') else pd.read_csv(source_ar_export)
df.columns = df.columns.str.strip()

# Coerce amount — handles "$1,234.56" style strings
df['amount'] = pd.to_numeric(df['amount'].astype(str).str.replace(r'[^\d.\-]', '', regex=True), errors='coerce')

# Drop credits and fully-paid invoices
df = df[df['amount'] > 0].copy()
df = df.reset_index(drop=True)
```

**Preview:** Check row count (should match open invoices in your system), confirm amount column is numeric, no nulls in customer or invoice number.

### Checkpoint: No null amounts

```
Assert: df['amount'].notna().all()
On failure: Check for rows where amount could not be parsed (currency symbols, text). 
            Show the failing rows and ask the user to fix the source file or map the column.
```
```

### 4. Output Tabs (required)

One `###` subsection per output tab (excluding Source Data, which is automatic). Describe what goes in each tab, what the key metrics are, and any formatting notes.

```markdown
## Output Tabs

### Ageing Buckets

Summary table: one row per ageing bucket (Current, 1–30, 31–60, 61–90, 90+).
Columns: Bucket | Invoice Count | Total Amount | % of Total AR.
Sort: bucket order (Current first, 90+ last).

### Top 20 Debtors

One row per customer, sorted by total outstanding descending.
Columns: Customer | Invoice Count | Total Outstanding | Oldest Invoice Date | Oldest Due Date | Primary Bucket.

### Exec Summary

Standard exec summary format. Key metrics:
- Total AR outstanding
- Number of open invoices
- % current vs overdue
- Largest single debtor
- Run date and source file name
```

---

## Checkpoint Format

Checkpoints sit between steps as a dedicated `### Checkpoint: Name` subsection.

```markdown
### Checkpoint: [Name]

**Assert:** `[Python expression that evaluates to True when data is healthy]`
**On failure:** [What the analyst should do — check source data, map a column, etc.]
```

Examples:
```markdown
### Checkpoint: No duplicate invoice numbers

**Assert:** `df['invoice_number'].duplicated().sum() == 0`
**On failure:** Show the duplicate rows. Likely cause: the export included both an invoice and 
               its credit note. Filter to rows where amount > 0 (already done in Step 1) 
               or check for partial payments split across rows.

### Checkpoint: Join retained expected row count

**Assert:** `len(merged_df) >= len(left_df) * 0.9`
**On failure:** More than 10% of rows were lost in the join. Check that the join key column 
               is consistent between both files. Common cause: trailing whitespace or 
               ALL CAPS in one file but not the other.
```

---

## Naming Conventions

**Recipe files:** `recipes/NN-kebab-name.md` where NN is a zero-padded sequence number (01, 02, ...).

**Source identifiers:** snake_case, descriptive. `ar_export`, `crm_closed_won`, `ad_spend`, `budget_file`.

**Output files:** kebab-case matching the recipe name. `ar-ageing-report.xlsx`, `pipeline-conversion.xlsx`.

**Step names:** Verb phrase. "Load and standardise", "Calculate ageing buckets", "Join on campaign".

---

## What Makes a Good Recipe

- **Granular steps.** One transformation per step. Don't combine load + parse + join into one block.
- **Every step has code.** Never write a step without a Python block. The code IS the step.
- **Named villains.** The `villain` field and the description should name the specific pain. "Rebuilding the AR pivot table in Excel each month" beats "data transformation".
- **Honest about schema variance.** Real exports are messy. Document the known quirks. Tell the analyst what the recipe expects and what it will do when it doesn't find it.
- **Exec Summary always last.** Every recipe ends with an Exec Summary tab. Always.
- **Nothing locked in.** A recipe is a `.md` file. It can be opened in Obsidian, edited in VS Code, diffed in GitHub, and run by any Claude Code instance pointed at it. No vendor lock-in.

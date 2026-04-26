---
name: recipe
description: Recipe skill for the by-analysts-for-analysts data prep platform. Activate when the user wants to create a recipe, run a recipe, execute a data workflow, build a workflow for AR ageing, sales pipeline, budget vs actual, ad spend reconciliation, ecommerce reconciliation, commission calculation, churn analysis, inventory reconciliation, time-tracking billing, headcount reconciliation, or any analyst data transformation task. Also activate when the user says "run this recipe", "create a recipe for", "execute this workflow", or drops a .md recipe file and asks to run it against data. Do NOT activate for general coding tasks, database schema design, or non-analyst workflows.
---

# Recipe Skill

Build and execute markdown-based data transformation workflows — human-readable, AI-editable, locally runnable. Every workflow is a recipe. Every recipe is forkable.

Read the full format spec before creating a new recipe: `.claude/skills/recipe/references/recipe-format.md`
The seed recipe catalogue is at: `.claude/skills/recipe/references/seed-recipes.md`

## Three Modes

### List Mode

When the user asks what recipes are available, or asks to pick a recipe:

1. Glob `recipes/*.md` to find all recipe files
2. Read the frontmatter of each to extract `recipe` name and `villain`
3. Display as a numbered list: `N. Recipe Name — replaces: villain`
4. Ask the user to pick one or describe a new workflow

### Create Mode

When the user describes a workflow they want to automate:

1. Check `references/seed-recipes.md` — if it matches one of the 15 seed patterns, use that description as the template
2. Ask for: source file (or describe the columns you expect)
3. Examine the source file if provided (read first 20 rows, list columns, note data types, flag obvious issues)
4. Generate a complete recipe `.md` file following the format in `references/recipe-format.md`
5. Include all steps with working Python code, checkpoints after critical steps, and output tab definitions
6. Save to `recipes/<nn>-<kebab-name>.md` (next available number)
7. Confirm: "Recipe saved. Run it with: 'run this recipe against [your file]'"

Recipe creation principles:
- Every step has a plain-English description AND a Python code block — never one without the other
- Steps should be granular: one transformation per step, not one giant block
- Checkpoints go after any join, deduplication, or date parse — wherever data shape could silently go wrong
- The `villain` frontmatter field should name the specific painful thing this replaces (e.g. "rebuilding AR pivot tables in Excel each month")
- Use the Exec Summary tab pattern from the format spec on every recipe

### Run Mode

When the user provides a recipe file and data file(s):

**Setup**
1. Read the recipe file — parse frontmatter YAML to get sources, output filename, and output tabs
2. Before asking for file paths, check `data/inputs/` for files matching each source's name or format. If found, confirm: "Found `crm-export.xlsx` in data/inputs/ — use this as `crm_export`?"
3. For any source not auto-resolved, ask the user for the file path
4. Confirm the full binding before proceeding: "I'll use `data/inputs/crm-export.xlsx` as `crm_export`. Proceed?"

**Run directory**
Create a timestamped run directory before executing any steps:
```
runs/YYYY-MM-DD_HH-MM-SS_<recipe-slug>/
  steps/       ← 10-row CSV preview written after each step
  output/      ← Excel workbook written at end
```
Where `<recipe-slug>` is the recipe name lowercased with spaces replaced by hyphens (e.g. `sales-pipeline-conversion`).

Write an initial `run.json` to the run directory at the start:
```json
{
  "recipe": "<recipe name from frontmatter>",
  "recipe_file": "recipes/<filename>.md",
  "recipe_version": "<version>",
  "run_id": "<YYYY-MM-DD_HH-MM-SS_recipe-slug>",
  "run_timestamp": "<ISO timestamp>",
  "sources": { "<source_name>": { "path": "<path>", "rows": <N> } },
  "steps": [],
  "checkpoints": [],
  "output": {},
  "run_status": "running"
}
```

**Execution protocol — follow this exactly**
Execute each Step section in order:

```
Before step N:
  - Print: "Step N: [step name] — [description]"
  - Show current row count

Execute the Python code block for the step.

After step N:
  - Print a preview table:
    rows_in → rows_out | nulls in key columns | distinct values in key column
    Show 5-row sample (head) of the transformed dataframe
  - Write a step markdown file to runs/<run-id>/steps/NN-<step-slug>.md using this template:

    ```markdown
    # Step N: [Step Name]

    > [Plain-English description copied from the recipe step]

    ## Stats

    | | |
    |---|---|
    | Rows in | X |
    | Rows out | Y |
    | Row delta | +/- N (P% change) |
    | Nulls in key columns | Z |

    ## Data Quality

    [One bullet per notable finding — use ✓ for expected/clean, ⚠ for issues needing attention, ℹ for informational]
    - ✓ All required columns present
    - ⚠ 1 row dropped: stage value `'proposal'` not in STAGE_MAP — add to map or fix in source
    - ℹ 2 rows have blank amounts (expected for Prospecting stage)

    ## Preview

    [First 10 rows rendered as a markdown table]

    ---
    *Step logged at [ISO timestamp]*
    ```

    Write data quality bullets based on what the step actually does:
    - Row count drops: flag any unexpected drop; note if expected (e.g. filtering to active deals)
    - Null counts: flag nulls in key columns, explain likely cause
    - Mapping coverage: report unmapped values by name
    - Value ranges: call out min/max if they look suspicious
    - Schema changes: note any columns added, renamed, or dropped

  - Append step result to run.json steps array:
    {"step": N, "name": "<step name>", "rows_in": X, "rows_out": Y, "nulls": Z, "status": "ok"}
  - If row count dropped >50% unexpectedly, pause and ask the user to confirm before continuing
```

For each Checkpoint section:
- Run the assertion
- If it passes: print "✓ Checkpoint passed: [name]"
  Append to run.json checkpoints: {"name": "<name>", "status": "pass"}
- If it fails: print the failure message, show the failing rows, and STOP — do not continue to the next step until the user resolves it or explicitly overrides
  Append to run.json checkpoints: {"name": "<name>", "status": "fail", "message": "<failure detail>"}

**Output**
1. After all steps complete, write the output Excel file to `runs/<run-id>/output/<output_filename>`
2. Create one worksheet per output tab in the order listed in the recipe frontmatter
3. The "Source Data" tab always contains the raw input as-loaded (before any transformations)
4. The "Exec Summary" tab is always last — use the standard pattern below
5. Finalise run.json: set `output` field and update `run_status` to `"success"` (or `"failed"`)
6. Print: "Output written to runs/<run-id>/output/<filename> — [N] tabs, [N] rows total"
7. Print: "Run artifacts saved to runs/<run-id>/"

**Exec Summary tab standard pattern**
```python
import datetime
summary_rows = [
    ["Recipe", recipe_name],
    ["Run date", datetime.date.today().isoformat()],
    ["Source file(s)", ", ".join(source_paths)],
    ["", ""],
    ["Step", "Rows in", "Rows out"],
]
# append one row per step from the step_log list built during execution
```

## Python Stack

Always use this stack — do not import alternatives unless the recipe explicitly requires them:

```python
import pandas as pd
import duckdb
import openpyxl
from thefuzz import fuzz, process  # for fuzzy joins only
from dateutil import parser as dateutil_parser  # for mixed date formats
```

**Date parsing:** When a date column has mixed formats or is ambiguous, use:
```python
df['date_col'] = df['date_col'].apply(lambda x: pd.Timestamp(dateutil_parser.parse(str(x))) if pd.notna(x) else pd.NaT)
```

**Fuzzy joining:** When joining on dirty string keys (campaign names, account codes):
```python
from thefuzz import process
def fuzzy_match(val, choices, threshold=80):
    result = process.extractOne(str(val), choices, score_cutoff=threshold)
    return result[0] if result else None
df['matched_key'] = df['raw_key'].apply(lambda x: fuzzy_match(x, reference_list))
```

**Writing Excel output:**
```python
from openpyxl import Workbook
from openpyxl.utils.dataframe import dataframe_to_rows

wb = Workbook()
wb.remove(wb.active)  # remove default sheet

for tab_name, df_tab in tabs.items():
    ws = wb.create_sheet(tab_name)
    for row in dataframe_to_rows(df_tab, index=False, header=True):
        ws.append(row)

wb.save(output_path)
```

## Show Me the Code Principle

Never hide the Python. Every recipe step exposes its code. If the user asks "show me the full script", generate a single runnable `.py` file by concatenating all step code blocks in order, wrapped with imports and the output writer. Save it alongside the recipe as `recipes/<name>.py`.

## Handling Schema Variance

Real analyst files are messy. When loading a source file:
1. Read all columns, strip whitespace from column names
2. Lowercase column names for matching
3. If a required column is missing, list the actual columns and ask the user to map them: "I expected 'invoice_date' but found: ['InvDate', 'Date Created', 'Due']. Which column is the invoice date?"
4. Store the mapping and apply it before step 1 runs

Never silently drop rows or columns. If something unexpected happens, stop and explain.

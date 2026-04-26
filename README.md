# Recipe — data prep workflows for analysts, by analysts

Every analyst has a version of the same problem: a painful, repetitive data transformation that lives in a maze of Excel formulas, a half-documented Python script, or someone's head. Every Monday, someone rebuilds the AR ageing report. Every quarter, someone reconciles the budget vs actuals by hand. Every week, someone copy-pastes the CRM export into a pivot table.

**Recipe** turns those workflows into human-readable markdown files that Claude Code can execute against your data.

A recipe is a `.md` file. It describes — in plain English and working Python — every step of a data transformation, from raw source files to a formatted, multi-tab Excel output. It's auditable, forkable, and executable. You can read it like documentation and run it like code.

---

## The idea in one sentence

> What if every analyst data workflow was a markdown file you could read, fork, edit, and run?

---

## How it works

Point Claude Code at a recipe and your data:

```
"Run the AR ageing recipe against my Xero export at ~/Downloads/debtors.xlsx"
```

Claude Code will:
1. Read the recipe file and bind your data file to the source
2. Execute each step in order, showing row counts, null counts, and a 10-row preview
3. Validate every checkpoint — pausing if something looks wrong
4. Write a formatted, multi-tab Excel workbook
5. Save a timestamped run folder with per-step markdown runbooks and a `run.json` audit log

Or describe a workflow and Claude Code will write the recipe for you:

```
"Create a recipe for monthly commission calculation from a Salesforce closed-won export"
```

---

## The run directory pattern

Every recipe execution produces a timestamped folder in `runs/`:

```
runs/2026-04-26_20-32-59_sales-pipeline-conversion/
├── run.json                  ← step log, checkpoint results, source SHA, run status
├── steps/
│   ├── 01-load-standardise-columns.md
│   ├── 02-normalise-stage-names.md
│   ├── 03-parse-dates-coerce-amounts.md
│   └── ...
└── output/
    └── pipeline-conversion.xlsx
```

Each step file is a **runbook document** — not a raw data dump:

```markdown
# Step 2: Normalise Stage Names

> Map CRM-specific stage names to 7 canonical stages so the conversion
> funnel is consistent regardless of which CRM the data came from.

## Stats
| | |
|---|---|
| Rows in | 25 |
| Rows out | 25 |
| Row delta | 0 (no change) |

## Data Quality
- ✓ All 25 stage values mapped successfully
- ✓ Demo: 4, Lost: 4, Negotiation: 3, Proposal: 4, Prospecting: 2, Qualified: 3, Won: 5

## Preview
| opportunity_name | stage_raw | stage_canonical | ...
```

The step files and `run.json` are plain text — tracked in git, diffable between runs. Run the same recipe next week and `git diff` shows exactly what changed in the data. The Excel output is gitignored (binary).

---

## The five Tier 1 recipes

These cover the highest-frequency, most-painful analyst workflows. Each one is the answer to a real search query that has no good result today.

### [01 — Sales Pipeline Conversion + Win Rate Analysis](recipes/01-sales-pipeline-conversion.md)

**Replaces:** Copy-pasting the CRM export into a pivot table every week to tell the VP of Sales what the pipeline looks like.

**Input:** One CRM export — Salesforce, HubSpot, or Pipedrive. One row per opportunity.

**What it does:** Normalises stage names across CRM systems to 7 canonical stages, builds a conversion funnel by value and by count, calculates win rates by quarter, measures sales cycle length, identifies stage drop-off, and produces a rep performance summary.

**Output tabs:** Source Data · Pipeline Conversion ($) · Pipeline Conversion (units) · Win Rate by Quarter · Sales Cycle Length · Stage Drop-off · Rep Performance · Exec Summary

**Key challenge:** Stage names differ across every CRM. "Closed Won" in Salesforce, "Deal Won" in HubSpot, "Won" in Pipedrive — the recipe normalises all of them to a consistent funnel automatically.

---

### [02 — Monthly AR Ageing Report](recipes/02-ar-ageing-report.md)

**Replaces:** Rebuilding the AR ageing pivot in Excel every month, manually bucketing invoices and fighting with date formats from the accounting system export.

**Input:** Debtors / AR export from Xero, QuickBooks, NetSuite, or Sage.

**What it does:** Parses mixed date formats, calculates days outstanding for every invoice, buckets into Current / 1–30 / 31–60 / 61–90 / 90+ days, identifies top debtors, tracks month-on-month ageing trend, and generates a collection priority list sorted by risk.

**Output tabs:** Source Data · Ageing Buckets · Top 20 Debtors · Ageing by Customer · MoM Trend · Collection Priority · Exec Summary

**Key challenge:** Every accounting system exports date and amount columns under different names and in different formats. The recipe probes for the due-date column and handles `DD/MM/YYYY`, `YYYY-MM-DD`, and mixed ISO formats automatically.

---

### [03 — Budget vs Actual Variance Analysis](recipes/03-budget-vs-actual.md)

**Replaces:** The quarterly ritual of copying actuals from the GL export into the budget spreadsheet, fighting with account code formats, and manually writing flux commentary for every significant variance.

**Input:** Two files — a GL / actuals export and a budget file (supports both flat and wide/pivot format).

**What it does:** Standardises account codes between the two systems (using fuzzy matching for codes that don't quite align), calculates variance by department and account, sorts unfavourable variances to the top, builds a YTD cumulative trend, and generates a flux commentary template with AI-drafted first-pass text for the top 10 variances.

**Output tabs:** Source Data (Actuals) · Source Data (Budget) · Variance by Department · Variance by Account · Top 10 Unfavourable · YTD Trend · Flux Commentary Template · Exec Summary

**Key challenge:** Account code formats almost never match exactly between the GL and the budget. The recipe applies fuzzy matching with a configurable threshold and a `MANUAL_OVERRIDES` dict for persistent mismatches.

---

### [04 — Ad-Spend to CRM Revenue Reconciliation](recipes/04-ad-spend-to-crm.md)

**Replaces:** The quarterly spreadsheet archaeology project of trying to figure out which campaigns actually drove revenue, when ad platform campaign names and CRM source fields were entered by different people at different times.

**Input:** Two files — an ad platform export (Google Ads, Meta, LinkedIn) and a CRM closed-won export.

**What it does:** Normalises campaign names from both systems (stripping UTM prefixes, standardising separators), fuzzy-matches them to bridge the gap, calculates blended CAC and ROAS by channel, estimates payback period, and surfaces unmatched deals and orphaned spend for review.

**Output tabs:** Source Data (Ads) · Source Data (CRM) · Spend Summary · Closed-Won by Source · Blended CAC by Channel · Payback Period · Attribution Issues · Exec Summary

**Key challenge:** Campaign names in the ad platform rarely match UTM source values in the CRM. This recipe's fuzzy-blend step is the showpiece — it classifies matches as High / Medium / Low confidence and surfaces everything unmatched for manual review.

---

### [05 — Multi-Store Ecommerce Reconciliation](recipes/05-ecommerce-reconciliation.md)

**Replaces:** Manually combining Shopify, Amazon, and Etsy exports into one spreadsheet every month — fighting with different column names, date formats, fee structures, and SKU codes across three platforms.

**Input:** Three channel exports — Shopify CSV, Amazon tab-delimited .txt, Etsy CSV.

**What it does:** Normalises all three sources to a common 13-column schema, combines them, deduplicates, calculates net revenue after platform fees (with configurable fee rate tables per channel), analyses sales by channel and SKU, tracks returns rate, and produces an inventory sell-through view.

**Output tabs:** Source Data (Shopify) · Source Data (Amazon) · Source Data (Etsy) · Combined Orders · Sales by Channel · Sales by SKU · Fees and Net Revenue · Returns Rate · Inventory Sell-through · Exec Summary

**Key challenge:** Fee structures differ completely across channels and change frequently. The recipe encodes each as named constants at the top of Step 1 — easy to update without touching the transformation logic.

---

## Getting started

### Prerequisites

```bash
pip install pandas duckdb openpyxl thefuzz python-dateutil
```

### Running a recipe

Open this project in Claude Code, then:

```
"Run the AR ageing recipe against ~/Downloads/my-xero-export.xlsx"
```

Claude Code will find the recipe, bind your file, execute step by step, and write the output to `runs/<timestamp>_ar-ageing-report/output/ar-ageing-report.xlsx`.

### Creating a new recipe

```
"Create a recipe for monthly commission calculation from a Salesforce closed-won export"
```

Claude Code will generate a complete recipe file in `recipes/` with all steps, checkpoints, and output tab definitions.

### Listing available recipes

```
"List available recipes"
```

### Running against the sample data

The repo includes a 25-row sample CRM export for recipe 01:

```
"Run the sales pipeline recipe against data/inputs/sample/crm-export-sample.csv"
```

---

## Recipe format

Every recipe file has the same structure:

**Frontmatter** (YAML) — recipe name, version, cadence, villain, source definitions with required columns and common issues, output filename, output tab list, tags.

**Body sections:**
1. One-paragraph description — what it does and why it matters
2. Sources — one subsection per source, schema and common issues
3. Steps — each step has a plain-English description, working Python code block, and a preview note
4. Checkpoints — `Assert:` + `On failure:` after any join, dedup, or date parse where data shape could silently go wrong
5. Output tabs — one subsection per tab describing layout and key metrics

Full format spec: [`.claude/skills/recipe/references/recipe-format.md`](.claude/skills/recipe/references/recipe-format.md)

---

## Python stack

| Library | Used for |
|---|---|
| `pandas` | All tabular transformations, joins, pivots, groupby |
| `duckdb` | SQL-style queries for larger files or complex aggregations |
| `openpyxl` | Multi-tab Excel output with formatting |
| `thefuzz` | Fuzzy string matching for dirty key joins (account codes, campaign names) |
| `python-dateutil` | Parsing mixed date formats (DD/MM/YYYY, YYYY-MM-DD, ISO, etc.) |

---

## Show me the code

Nothing is hidden. Every recipe step exposes its Python. If you want a standalone script, ask Claude Code:

```
"Export the AR ageing recipe as a runnable Python script"
```

It will concatenate all step code blocks in order, wrap with imports and the output writer, and save it alongside the recipe.

---

## Forking a recipe

The fastest way to create a new recipe is to fork an existing one:

1. Copy a recipe file: `cp recipes/02-ar-ageing-report.md recipes/06-my-workflow.md`
2. Update the frontmatter (recipe name, villain, sources, output)
3. Adjust the steps — most of the structural code (date parsing, column normalisation, Excel writing) can be reused directly
4. Run it against your data and iterate

Or describe the workflow to Claude Code and let it generate the recipe from scratch.

---

## Directory structure

```
recipes/                              Ready-to-run seed recipe files
data/inputs/                          Drop your source files here (gitignored)
data/inputs/sample/                   Small sample files for testing (tracked)
runs/                                 Timestamped run artifacts
  YYYY-MM-DD_HH-MM-SS_recipe-name/
    run.json                          Step log + checkpoint results (tracked)
    steps/NN-step-name.md             Per-step runbook documents (tracked)
    output/report.xlsx                Excel output (gitignored)
.claude/skills/recipe/SKILL.md        Teaches Claude Code to create and run recipes
.claude/skills/recipe/references/     Format spec and full seed recipe catalogue
```

---

## What's coming

The seed recipe catalogue has 15 recipes planned across three tiers. Tier 1 (shipped above) covers the highest-frequency workflows. Next up:

- **Tier 2:** Payroll reconciliation · Inventory reconciliation · SaaS churn analysis · Time-tracking to billing · Headcount reconciliation
- **Tier 3:** Customer LTV segmentation · Marketing attribution multi-touch · Manufacturing yield analysis · Project profitability · Subscription revenue waterfall

Full catalogue: [`.claude/skills/recipe/references/seed-recipes.md`](.claude/skills/recipe/references/seed-recipes.md)

---

## The idea behind this

This project is the productised version of the "instruction file that replaced my analyst" pattern — the idea that a sufficiently detailed, well-structured instruction document is itself a kind of program. Markdown recipes are that instruction document: readable by humans, executable by AI, forkable by anyone.

The goal for Phase 0 is simple: do analysts actually use this? If you ran a recipe against your own data, opened a PR to improve a step, or forked a recipe for your own workflow — that's the signal. GitHub stars and forks are how we're measuring it.

---

*Built with [Claude Code](https://claude.ai/code).*

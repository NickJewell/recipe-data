# Recipe

A by-analysts-for-analysts data prep platform where every workflow is a human-readable, AI-editable markdown recipe.

## What this is

Each file in `recipes/` is a self-contained markdown recipe that describes a complete data transformation workflow — sources, steps, checkpoints, and output tabs. Recipes are:

- **Readable** — plain English descriptions of every step, nothing hidden behind a UI
- **Auditable** — Python code is always visible; every decision is explained inline
- **Forkable** — copy a recipe, repoint the sources, tweak the logic, ship it
- **Executable** — Claude Code reads a recipe and runs it against your data files

## Phase 0 goal

Validate that analysts will adopt markdown-recipe thinking. Measure: GitHub stars, forks, and "I used this" replies. The 5 Tier 1 seed recipes cover the highest-frequency, most-painful analyst workflows. Each recipe is the answer to a real search query that has no good result today.

## How to run a recipe

Point Claude Code at a recipe and your data file:

> "Run the AR ageing recipe against my Xero export at ~/Downloads/debtors.xlsx"

Claude Code will read the recipe, bind your file to the source, execute each step with inline previews (row count, null count, 10-row sample), validate every checkpoint, and write the output Excel workbook.

## How to create a new recipe

Describe the workflow:

> "Create a recipe for monthly commission calculation from a Salesforce closed-won export"

Claude Code will generate a complete recipe file with proper frontmatter, steps, checkpoints, and output tab definitions, and save it to `recipes/`.

Or fork an existing recipe manually: copy a file, update the frontmatter, adjust the steps. The format spec is at `.claude/skills/recipe/references/recipe-format.md`.

## Directory structure

```
recipes/                              Ready-to-run seed recipe files
.claude/skills/recipe/SKILL.md        Teaches Claude how to create and run recipes
.claude/skills/recipe/references/     Format spec and seed recipe catalogue
```

## Python stack

Recipes use: `pandas`, `duckdb`, `openpyxl`, `thefuzz`, `python-dateutil`. Install once with:

```bash
pip install pandas duckdb openpyxl thefuzz python-dateutil
```

# Seed Recipe Catalogue

15 high-frequency analyst workflows ranked by Phase-0 priority. Each recipe has a named villain, a target analyst persona, and a distribution hook.

Selection criteria applied to every entry: single-source or simple multi-source blend; recurring cadence; genuinely painful today; search-friendly title; non-overlapping roles.

---

## Tier 1 — Ship First (weeks 1–2 through 9–10)

These are the highest-search-volume, most-universally-painful workflows. Get these right and the repo has traction before anything else ships.

### 1. Sales Pipeline Conversion + Win Rate Analysis

**File:** `recipes/01-sales-pipeline-conversion.md`  
**Status:** Shipped  
**Input:** CRM export (Salesforce, HubSpot, Pipedrive) — one row per opportunity  
**Output:** Pipeline Conversion ($) · Pipeline Conversion (units) · Win Rate by Quarter · Sales Cycle Length · Stage Drop-off · Rep Performance · Exec Summary  
**Villain:** "Copy-pasting CRM data into a pivot table every week to tell the VP of Sales what the pipeline looks like"  
**Persona:** RevOps analyst, Sales Ops manager  
**Distribution hook:** Cite and credit the Stephen Smith Substack post; publish as the generalised template. r/RevOps, r/salesforce, RevOps Co-op Slack.  
**Key challenge:** Stage names differ across CRMs — Salesforce uses "Closed Won" / "Closed Lost", HubSpot uses "Deal Won" / "Deal Lost". Recipe normalises to 7 canonical stages.

---

### 2. Monthly AR Ageing from an ERP/Accounting Export

**File:** `recipes/02-ar-ageing-report.md`  
**Status:** Shipped  
**Input:** Debtors/AR export from Xero, QuickBooks, NetSuite, Sage — one row per open invoice  
**Output:** Ageing Buckets · Top 20 Debtors · Ageing by Customer · Month-over-Month Trend · Collection Priority List · Exec Summary  
**Villain:** "Exporting from Xero, pasting into a pivot table, manually bucketing by due date, then reformatting — every single month"  
**Persona:** Finance manager, AR analyst, FP&A analyst  
**Distribution hook:** r/Accounting, r/FPandA, CFO-focused LinkedIn. Highest single search volume in the set.  
**Key challenge:** Due-date column named differently per system; Xero uses "Due Date", NetSuite uses "Due Date/Time" (has time component).

---

### 3. Monthly Budget vs. Actual Variance Analysis

**File:** `recipes/03-budget-vs-actual.md`  
**Status:** Shipped  
**Input:** Two files — GL/actuals export from accounting system + budget file (flat .xlsx)  
**Output:** Variance by Department · Variance by Account · Top 10 Unfavourable Variances · YTD Trend · Flux Commentary Template · Exec Summary  
**Villain:** "Rebuilding the budget vs actual file every month because someone changes the account structure or the budget file isn't aligned to the GL"  
**Persona:** FP&A analyst, management accountant, finance business partner  
**Distribution hook:** r/FPandA, CFO newsletters, Corporate Finance Institute audience.  
**Key challenge:** Account code formats rarely match between GL and budget file. Recipe applies fuzzy match + exposes an override map for persistent mismatches.

---

### 4. Ad-Spend to CRM Revenue Reconciliation

**File:** `recipes/04-ad-spend-to-crm.md`  
**Status:** Shipped  
**Input:** Two files — ad platform export (Google Ads / Meta / LinkedIn Ads) + CRM closed-won export  
**Output:** Spend Summary · Closed-Won by Source · Blended CAC by Channel · Payback Period · Attribution Issues · Exec Summary  
**Villain:** "The monthly argument between marketing and finance about whether the campaign that spent £40k actually generated any revenue"  
**Persona:** Marketing ops analyst, growth analyst, demand gen manager  
**Distribution hook:** r/PPC, r/bigseo, marketing-ops communities, growth Slack groups.  
**Key challenge:** Campaign names in ad platforms rarely match UTM source in CRM. This is the showpiece fuzzy-blend step — demonstrates the multi-source blend angle better than any other seed.

---

### 5. Multi-Store Ecommerce Sales Reconciliation (Shopify + Amazon + Etsy)

**File:** `recipes/05-ecommerce-reconciliation.md`  
**Status:** Shipped  
**Input:** Three order exports from three channels — different schemas, date formats, fee structures  
**Output:** Source Data (each channel) · Combined Orders · Sales by Channel · Sales by SKU · Fees and Net Revenue · Returns Rate · Inventory Sell-through · Exec Summary  
**Villain:** "Manually copying orders from three dashboards into one spreadsheet each month to figure out which channel is actually profitable after fees"  
**Persona:** Solo ecommerce operator, small brand, ecommerce agency analyst  
**Distribution hook:** r/ecommerce, r/FulfilledByAmazon, r/shopify, ecommerce Twitter.  
**Key challenge:** Fee structures differ completely per channel. Recipe encodes each as a configurable rate table in frontmatter so it can be updated without touching the step code.

---

## Tier 2 — Ship Next (weeks 3–4, after Tier 1)

Broadens the repo to more roles and verticals. Each targets a different community.

### 6. Time-Tracking to Client Billing Reconciliation

**Input:** Time-tracking export (Toggl / Harvest / Clockify) + rate card .xlsx  
**Output:** Billable vs Non-Billable · Utilisation by Person · Invoice-Ready by Client · Over/Under Budget by Project · Rate Card Applied · Exec Summary  
**Villain:** "Manually applying rate cards to time entries every month — especially when different people have different rates for different clients"  
**Persona:** Agency ops, consultancy finance, freelance studio owner  
**Distribution hook:** r/consulting, r/freelance, agency finance communities.

---

### 7. Inventory Reconciliation: Physical Count vs. System

**Input:** System-of-record inventory export + physical count file (often a phone-typed spreadsheet)  
**Output:** Variance by SKU · Top Shrinkage Items · Variance by Location · Cost Impact · Items to Investigate · Exec Summary  
**Villain:** "Spending two days after every stocktake manually comparing system quantities to paper count sheets"  
**Persona:** Retail ops analyst, warehouse manager, supply chain analyst  
**Distribution hook:** r/ProductManagement, supply-chain LinkedIn, retail ops communities.

---

### 8. Customer Churn and Cohort Retention Analysis

**Input:** Subscription export (Stripe, Chargebee, custom CSV) — one row per subscription event or per active customer per month  
**Output:** Monthly Churn Rate · Net Revenue Retention · Cohort Retention Table · Churn by Reason · At-Risk Accounts · Exec Summary  
**Villain:** "Building the cohort retention table by hand in Excel — every quarter someone asks for a slightly different cut and you rebuild the whole thing"  
**Persona:** SaaS analyst, customer success ops, product analyst  
**Distribution hook:** Indie Hackers, SaaS finance communities, r/startups.

---

### 9. Commission Calculation from CRM Closed-Won

**Input:** CRM closed-won export + commission plan (encoded as rules in the recipe)  
**Output:** Commissions by Rep · Accelerators Triggered · SPIFF Bonuses · Manager Overrides · Payout Summary · Exec Summary  
**Villain:** "A £200 discrepancy in someone's commission payout that takes four hours to debug because the logic lives in a spreadsheet nobody can fully understand"  
**Persona:** Sales operations analyst, RevOps manager, finance business partner (revenue)  
**Distribution hook:** r/salesops, RevOps communities, sales finance LinkedIn.

---

### 10. Payroll Headcount Reconciliation for Finance Close

**Input:** HRIS headcount export (BambooHR / Rippling / ADP) + payroll register + GL cost centre mapping  
**Output:** Active Headcount by Department · New Starters · Leavers · Payroll vs HRIS Variance · Unmapped Cost Centres · Exec Summary  
**Villain:** "The monthly close where HRIS and payroll never agree on headcount and someone has to manually reconcile the diff at 6pm on close day"  
**Persona:** Finance business partner (HR), payroll analyst, FP&A  
**Distribution hook:** r/FPandA, HR finance LinkedIn, people operations communities.

---

## Tier 3 — Hold in Reserve or Accept as Community Contributions

Real workflows worth covering, but slightly off-pattern, more vertical-specific, or with more direct competition. Publish once Tier 1 and 2 have pulled in an audience, or offer as bounties.

### 11. Credit Card and Bank Statement Reconciliation

**Input:** Bank/card export (CSV) + accounting system cash account export  
**Why Tier 3:** Huge audience but Dext, Ramp, and Expensify have been automating this for years. Harder to carve a distinctive wedge.

---

### 12. Google Analytics Export to Marketing Performance Dashboard

**Input:** GA4 export + ad platform spend  
**Why Tier 3:** Looker Studio does this free. The wedge is "portable workbook you email to the CMO" — real but narrower than Tier 1 picks.

---

### 13. Support Ticket SLA and Volume Analysis

**Input:** Zendesk / Intercom / Freshdesk export  
**Why Tier 3:** Every support platform ships a dashboard. Wedge is "export + custom slices + exec format" — contested.

---

### 14. Project Portfolio Burn vs. Plan (Jira/Linear/Asana Exports)

**Input:** Project management tool export of issues + story points + time tracking  
**Why Tier 3:** Engineering ops pain but slightly off from the core analyst persona Phase 0 targets. Good contributor-bait for the community phase.

---

### 15. Expense Report Consolidation and Policy Violation Flagging

**Input:** Expense system export (Expensify / SAP Concur / Pleo) + policy rules file  
**Why Tier 3:** Narrower audience and most expense tools now have native policy flagging. Best as a "we'll build this if you ask" item.

---

## Patterns Worth Noting

**Finance and ops dominate Tier 1.** Highest ratio of recurring-painful-workflows to tooling options under £100/month. The Recipe brand should lean into "for analysts who live in spreadsheets", not "for data teams" — different tribes.

**Every Tier 1 recipe has a named villain.** The recipe isn't just useful — it's *against* something. That's what makes content around it punchy rather than dry.

**Blend beats single-source for differentiation.** Five of the 15 recipes are two-or-more-source joins (3, 4, 5, 6, 10). These showcase the multi-file blend angle where visual prep tools shine today. Keep blend-heavy recipes prominent in the README.

**Phase-0 launch order:** Recipe 1 (ladders off the Stephen Smith post) → Recipe 2 (broadest search volume) → Recipe 3 (captures FP&A) → Recipe 4 (demonstrates blend, captures marketing ops) → Recipe 5 (captures ecommerce solopreneurs, the best early-adopter tribe).

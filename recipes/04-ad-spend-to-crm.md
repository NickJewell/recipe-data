---
recipe: Ad-Spend to CRM Revenue Reconciliation
version: 1.0.0
cadence: monthly
villain: "The monthly argument between marketing and finance about whether the campaign that spent £40k actually generated any revenue — because campaign names in the ad platform never match UTM sources in the CRM"
sources:
  - name: ad_spend
    description: Monthly spend export from ad platforms (Google Ads, Meta Ads, LinkedIn Campaign Manager) — one row per campaign
    format: xlsx or csv
    required_columns:
      - campaign_name
      - platform
      - spend
    common_issues:
      - Google Ads exports use "Campaign" as the column name; Meta uses "Campaign name"
      - Spend may be in local currency or already converted — check the currency column
      - Campaigns may be at campaign, ad set, or ad level — recipe expects campaign level
  - name: crm_closed_won
    description: Closed-won opportunities export from CRM — one row per deal with attribution source
    format: xlsx or csv
    required_columns:
      - deal_value
      - lead_source
      - close_date
    common_issues:
      - Lead source / UTM source is not consistently filled in for all deals
      - Campaign attribution in CRM may be first-touch, last-touch, or not tracked at all
      - Deal value may use a different currency than ad spend
output: ad-spend-to-crm.xlsx
output_tabs:
  - Source Data (Ad Spend)
  - Source Data (CRM)
  - Spend Summary
  - Closed-Won by Source
  - Blended CAC by Channel
  - Payback Period
  - Attribution Issues
  - Exec Summary
tags:
  - marketing
  - attribution
  - CAC
  - RevOps
  - monthly
---

# Ad-Spend to CRM Revenue Reconciliation

Every month, marketing reports on spend and impressions while finance reports on revenue, and neither talks to the other until someone asks "what's our CAC?". The problem isn't the calculation — it's that campaign names in Google Ads are "Q4-2024-Brand-Exact-Match-UK" while the UTM source in Salesforce is "google_brand" and the CRM lead source is "Google - Paid". These are the same thing, but a VLOOKUP will never find them.

This recipe is the showpiece blend recipe: it loads both files, applies fuzzy matching to bridge the campaign attribution gap, calculates blended customer acquisition cost by channel, estimates payback periods, and flags every deal where attribution is missing or ambiguous. The fuzzy-match step is the thing Power Query analysts have been asking for since 2018 and still can't do natively.

## Sources

### ad_spend

Monthly spend export from your ad platforms. One row per campaign.

Export separately from: Google Ads (Reports → Campaigns), Meta Ads Manager (Campaigns tab), LinkedIn Campaign Manager (Campaign performance). Combine them into one file with a "Platform" column, or export separately and the recipe will combine them.

Expected columns:
- `Campaign` / `Campaign Name` / `Campaign name` — campaign identifier
- `Platform` / `Source` / `Channel` — which platform (Google, Meta, LinkedIn)
- `Spend` / `Amount spent` / `Cost` — total spend in the period (numeric or currency-formatted)
- `Impressions` / `Clicks` — optional, included in Spend Summary if present
- `Period` / `Month` / `Date` — the period this spend covers

### crm_closed_won

Closed-won opportunities from your CRM. One row per deal.

Expected columns:
- `Deal Value` / `Amount` / `Close Value` — revenue from this deal
- `Lead Source` / `Source` / `UTM Source` / `Channel` — how this lead arrived
- `Campaign` / `UTM Campaign` / `Campaign Name` — specific campaign attribution
- `Close Date` / `Closed Date` — when the deal was won
- `Owner` / `Rep` — for segmenting by rep (optional)

## Steps

### Step 1: Load ad spend and normalise

Load the ad spend export. Handle multiple platforms in one file or combine
multiple single-platform exports.

```python
import pandas as pd
import re

if source_ad_spend.endswith('.xlsx') or source_ad_spend.endswith('.xls'):
    df_ads = pd.read_excel(source_ad_spend)
else:
    try:
        df_ads = pd.read_csv(source_ad_spend, encoding='utf-8')
    except UnicodeDecodeError:
        df_ads = pd.read_csv(source_ad_spend, encoding='latin-1')

df_ads.columns = df_ads.columns.str.strip()
df_ads_source = df_ads.copy()

ADS_COL_MAP = {
    'campaign': 'campaign_name',
    'campaign name': 'campaign_name',
    'ad campaign': 'campaign_name',
    'amount spent': 'spend',
    'cost': 'spend',
    'total spend': 'spend',
    'channel': 'platform',
    'source': 'platform',
    'network': 'platform',
    'month': 'period',
    'date': 'period',
}
df_ads.columns = df_ads.columns.str.lower().str.strip()
df_ads = df_ads.rename(columns=ADS_COL_MAP)

# Coerce spend
df_ads['spend'] = pd.to_numeric(
    df_ads['spend'].astype(str).str.replace(r'[^\d.\-]', '', regex=True),
    errors='coerce'
).fillna(0)

df_ads = df_ads[df_ads['spend'] > 0].copy()

# If no platform column, prompt — for now default to 'Unknown'
if 'platform' not in df_ads.columns:
    df_ads['platform'] = 'Unknown'

# Normalise platform names
PLATFORM_MAP = {
    'google': 'Google', 'google ads': 'Google', 'adwords': 'Google',
    'meta': 'Meta', 'facebook': 'Meta', 'instagram': 'Meta', 'fb': 'Meta',
    'linkedin': 'LinkedIn', 'li': 'LinkedIn',
    'bing': 'Bing', 'microsoft': 'Bing',
    'tiktok': 'TikTok',
}
df_ads['platform'] = df_ads['platform'].str.lower().str.strip().map(
    lambda x: next((v for k, v in PLATFORM_MAP.items() if k in str(x).lower()), x.title())
)

print(f"Ad spend: {len(df_ads)} campaigns, total spend: {df_ads['spend'].sum():,.0f}")
print(f"Platforms: {df_ads['platform'].value_counts().to_dict()}")
```

**Preview:** Check total spend matches your ad platform invoices. Check platform distribution looks right.

### Step 2: Load CRM closed-won deals

Load the CRM export and normalise attribution columns.

```python
if source_crm_closed_won.endswith('.xlsx') or source_crm_closed_won.endswith('.xls'):
    df_crm = pd.read_excel(source_crm_closed_won)
else:
    try:
        df_crm = pd.read_csv(source_crm_closed_won, encoding='utf-8')
    except UnicodeDecodeError:
        df_crm = pd.read_csv(source_crm_closed_won, encoding='latin-1')

df_crm.columns = df_crm.columns.str.strip()
df_crm_source = df_crm.copy()

CRM_COL_MAP = {
    'amount': 'deal_value',
    'close value': 'deal_value',
    'revenue': 'deal_value',
    'arr': 'deal_value',
    'deal amount': 'deal_value',
    'lead source': 'lead_source',
    'source': 'lead_source',
    'utm source': 'lead_source',
    'acquisition source': 'lead_source',
    'utm campaign': 'crm_campaign',
    'campaign': 'crm_campaign',
    'campaign name': 'crm_campaign',
    'closed date': 'close_date',
    'close date': 'close_date',
    'won date': 'close_date',
}
df_crm.columns = df_crm.columns.str.lower().str.strip()
df_crm = df_crm.rename(columns=CRM_COL_MAP)

df_crm['deal_value'] = pd.to_numeric(
    df_crm['deal_value'].astype(str).str.replace(r'[^\d.\-]', '', regex=True),
    errors='coerce'
).fillna(0)
df_crm = df_crm[df_crm['deal_value'] > 0].copy()

from dateutil import parser as dateutil_parser

def parse_date_safe(val):
    if pd.isna(val):
        return pd.NaT
    try:
        return pd.Timestamp(dateutil_parser.parse(str(val)))
    except Exception:
        return pd.NaT

if 'close_date' in df_crm.columns:
    df_crm['close_date'] = df_crm['close_date'].apply(parse_date_safe)
    df_crm['close_month'] = df_crm['close_date'].dt.strftime('%Y-%m')

print(f"CRM closed-won: {len(df_crm)} deals, total value: {df_crm['deal_value'].sum():,.0f}")
```

**Preview:** Check that deal count and total value match your CRM dashboard. Check what % of deals have a non-blank lead_source.

### Checkpoint: Reasonable attribution coverage

```
Assert: df_crm['lead_source'].notna().mean() >= 0.5
On failure: More than 50% of closed-won deals have no lead source attribution.
            This recipe can still run (it will flag these as 'Unattributed') but the
            CAC calculation will be unreliable. Consider enriching CRM attribution
            before relying on this output for budget decisions.
```

### Step 3: Build spend summary by platform and channel

Aggregate ad spend to the platform level for the spend summary tab.
This is what finance wants to see — total spend per channel.

```python
spend_summary = (
    df_ads.groupby('platform')
    .agg(
        campaigns=('campaign_name', 'nunique'),
        total_spend=('spend', 'sum'),
    )
    .reset_index()
    .sort_values('total_spend', ascending=False)
)
spend_summary.columns = ['Platform', 'Campaign Count', 'Total Spend']
total_spend = spend_summary['Total Spend'].sum()
spend_summary['% of Total Spend'] = (spend_summary['Total Spend'] / total_spend * 100).round(1).astype(str) + '%'

# Add totals
spend_summary = pd.concat([spend_summary, pd.DataFrame([{
    'Platform': 'TOTAL',
    'Campaign Count': spend_summary['Campaign Count'].sum(),
    'Total Spend': total_spend,
    '% of Total Spend': '100%',
}])], ignore_index=True)
```

**Preview:** Total Spend should match Step 1 total. Platform split should match your media plan.

### Step 4: Closed-won revenue by lead source

Aggregate CRM deals by lead_source. This is raw — before attribution matching.

```python
crm_by_source = (
    df_crm.groupby(df_crm['lead_source'].fillna('Unattributed'))
    .agg(
        deal_count=('deal_value', 'count'),
        total_revenue=('deal_value', 'sum'),
        avg_deal_size=('deal_value', 'mean'),
    )
    .reset_index()
    .sort_values('total_revenue', ascending=False)
)
crm_by_source.columns = ['Lead Source', 'Deal Count', 'Total Revenue', 'Avg Deal Size']
crm_by_source['Avg Deal Size'] = crm_by_source['Avg Deal Size'].round(0)
```

**Preview:** Check what % of revenue is "Unattributed". High unattributed % = the attribution discussion you need to have with the marketing team.

### Step 5: Fuzzy-match campaign names to ad platform campaigns

This is the key step. Bridge the gap between ad platform campaign names
and CRM lead source / campaign values using fuzzy string matching.

```python
from thefuzz import process, fuzz

# Build a normalised campaign name lookup
def normalise_campaign_name(name):
    """Remove noise from campaign names for better fuzzy matching."""
    if pd.isna(name):
        return ''
    name = str(name).lower().strip()
    # Remove common prefixes/suffixes that vary between systems
    name = re.sub(r'(utm_|utmcampaign=|campaign=)', '', name)
    name = re.sub(r'[-_|/]+', ' ', name)
    name = re.sub(r'\s+', ' ', name)
    return name.strip()

# Reference list: normalised ad campaign names + their original name and platform
ad_campaign_lookup = {
    normalise_campaign_name(row['campaign_name']): (row['campaign_name'], row['platform'])
    for _, row in df_ads.iterrows()
}
ad_campaign_keys = list(ad_campaign_lookup.keys())

# For each CRM lead source, find the best matching ad campaign
attribution_matches = []
unique_sources = df_crm['lead_source'].dropna().unique()

for source in unique_sources:
    norm_source = normalise_campaign_name(source)
    if not norm_source:
        continue
    result = process.extractOne(norm_source, ad_campaign_keys, score_cutoff=60)
    if result:
        matched_key, score = result[0], result[1]
        original_campaign, platform = ad_campaign_lookup[matched_key]
        attribution_matches.append({
            'crm_source': source,
            'matched_campaign': original_campaign,
            'matched_platform': platform,
            'match_score': score,
            'match_quality': 'High' if score >= 85 else 'Medium' if score >= 70 else 'Low',
        })
    else:
        attribution_matches.append({
            'crm_source': source,
            'matched_campaign': None,
            'matched_platform': None,
            'match_score': 0,
            'match_quality': 'Unmatched',
        })

attribution_map_df = pd.DataFrame(attribution_matches)
print(f"\nAttribution matching results:")
print(attribution_map_df['match_quality'].value_counts().to_string())

# Apply mapping to CRM deals
source_to_platform = dict(zip(
    attribution_map_df['crm_source'],
    attribution_map_df['matched_platform']
))
source_to_campaign = dict(zip(
    attribution_map_df['crm_source'],
    attribution_map_df['matched_campaign']
))
df_crm['matched_platform'] = df_crm['lead_source'].map(source_to_platform).fillna('Unmatched')
df_crm['matched_campaign'] = df_crm['lead_source'].map(source_to_campaign)
```

**Preview:** Check the match quality distribution. "Unmatched" sources are the attribution issues tab. "Low" matches should be reviewed manually.

### Step 6: Calculate blended CAC by channel

CAC = total ad spend / number of new customers won (or deals closed) via that channel.

```python
# Revenue attributed to paid channels
paid_revenue_by_platform = (
    df_crm[df_crm['matched_platform'].notna() & (df_crm['matched_platform'] != 'Unmatched')]
    .groupby('matched_platform')
    .agg(
        deals_won=('deal_value', 'count'),
        revenue_attributed=('deal_value', 'sum'),
    )
    .reset_index()
)
paid_revenue_by_platform.columns = ['Platform', 'Deals Won', 'Revenue Attributed']

# Join to spend
cac_df = pd.merge(
    spend_summary[spend_summary['Platform'] != 'TOTAL'][['Platform', 'Total Spend']],
    paid_revenue_by_platform,
    on='Platform',
    how='outer',
).fillna(0)

cac_df['Blended CAC'] = (cac_df['Total Spend'] / cac_df['Deals Won']).replace([float('inf')], None).round(0)
cac_df['ROAS'] = (cac_df['Revenue Attributed'] / cac_df['Total Spend']).replace([float('inf')], None).round(2)
cac_df['Revenue / Spend'] = cac_df['ROAS'].apply(lambda x: f"{x:.2f}x" if pd.notna(x) else 'N/A')

cac_df = cac_df.sort_values('Total Spend', ascending=False)
```

**Preview:** ROAS > 1 means the channel is generating more revenue than it costs (at first-touch attribution). ROAS < 1 is the number that starts the budget conversation.

### Step 7: Payback period estimate

Rough payback period = CAC / (monthly revenue per customer). Useful for
subscription businesses where LTV matters more than first-deal value.

```python
# Monthly revenue per customer = avg deal size / 12 (assuming annual contracts)
# This is a rough approximation — adjust for your business model
avg_deal_size_by_platform = (
    df_crm[df_crm['matched_platform'] != 'Unmatched']
    .groupby('matched_platform')['deal_value']
    .mean()
    .reset_index()
)
avg_deal_size_by_platform.columns = ['Platform', 'Avg Deal Size']

payback_df = pd.merge(cac_df[['Platform', 'Blended CAC']], avg_deal_size_by_platform, on='Platform', how='left')
payback_df['Monthly Revenue (est.)'] = (payback_df['Avg Deal Size'] / 12).round(0)
payback_df['Payback (months)'] = (
    payback_df['Blended CAC'] / payback_df['Monthly Revenue (est.)']
).replace([float('inf')], None).round(1)
payback_df['Payback (months)'] = payback_df['Payback (months)'].apply(
    lambda x: f"{x:.1f}" if pd.notna(x) else 'N/A'
)

# Note about methodology
payback_note = pd.DataFrame([{
    'Note': 'Payback period = CAC / (Avg Deal Size / 12). Assumes annual contract value. Adjust for subscription ARR, usage-based, or transactional models.'
}])
payback_tab = pd.concat([payback_df, payback_note], ignore_index=True)
```

**Preview:** Payback > 24 months is a red flag for most SaaS businesses. Payback < 12 months means the channel is paying back within the year.

### Step 8: Attribution issues — what couldn't be matched

Flag every deal where attribution is missing or the fuzzy match was low confidence.
This is the transparency layer — the analyst should review these before presenting.

```python
# Deals with no attribution
unattributed_deals = df_crm[df_crm['lead_source'].isna() | (df_crm['lead_source'] == '')].copy()

# Deals with low-confidence attribution match
low_confidence_sources = attribution_map_df[
    attribution_map_df['match_quality'].isin(['Low', 'Unmatched'])
]['crm_source'].tolist()
low_confidence_deals = df_crm[df_crm['lead_source'].isin(low_confidence_sources)].copy()

# Ad spend with no CRM attribution match (orphaned spend)
matched_campaigns = df_crm['matched_campaign'].dropna().unique()
orphaned_spend = df_ads[~df_ads['campaign_name'].isin(matched_campaigns)].copy()
orphaned_spend_total = orphaned_spend['spend'].sum()

attribution_issues = pd.DataFrame([
    ['Deals with no lead source', len(unattributed_deals), f"{unattributed_deals['deal_value'].sum():,.0f}"],
    ['Deals with low-confidence match', len(low_confidence_deals), f"{low_confidence_deals['deal_value'].sum():,.0f}"],
    ['Ad campaigns with no CRM match', len(orphaned_spend), f"{orphaned_spend_total:,.0f}"],
    ['% of spend unattributed to revenue', '', f"{orphaned_spend_total / total_spend * 100:.1f}%"],
], columns=['Issue', 'Count', 'Value'])

# Detail: the specific CRM sources that couldn't be matched
unmatched_source_detail = attribution_map_df[
    attribution_map_df['match_quality'].isin(['Low', 'Unmatched'])
][['crm_source', 'matched_campaign', 'match_score', 'match_quality']]
```

**Preview:** The attribution issues tab is what you bring to the conversation with the marketing team. Every row is a question: "What channel did this actually come from?"

### Step 9: Build Exec Summary

```python
import datetime

total_revenue = df_crm['deal_value'].sum()
attributed_revenue = df_crm[df_crm['matched_platform'] != 'Unmatched']['deal_value'].sum()
pct_attributed = attributed_revenue / total_revenue * 100 if total_revenue > 0 else 0
overall_cac = total_spend / len(df_crm) if len(df_crm) > 0 else 0
overall_roas = total_revenue / total_spend if total_spend > 0 else 0

exec_summary = pd.DataFrame([
    ['Total Ad Spend', f"{total_spend:,.0f}"],
    ['Total Closed-Won Revenue', f"{total_revenue:,.0f}"],
    ['Revenue Attributed to Paid Channels', f"{attributed_revenue:,.0f}"],
    ['% Revenue Attributed', f"{pct_attributed:.1f}%"],
    ['Blended CAC (all channels)', f"{overall_cac:,.0f}"],
    ['Overall ROAS', f"{overall_roas:.2f}x"],
    ['Deals with no attribution', f"{len(unattributed_deals)}"],
    ['', ''],
    ['Run Date', datetime.date.today().isoformat()],
    ['Ad Spend Source', source_ad_spend],
    ['CRM Source', source_crm_closed_won],
], columns=['Metric', 'Value'])
```

## Output Tabs

### Source Data (Ad Spend)

`df_ads_source` — raw ad spend as loaded.

### Source Data (CRM)

`df_crm_source` — raw CRM closed-won as loaded.

### Spend Summary

`spend_summary` — spend by platform with campaign count, total spend, and % of total.

### Closed-Won by Source

`crm_by_source` — CRM deals grouped by raw lead source. Shows what the CRM has before attribution matching.

### Blended CAC by Channel

`cac_df` — platform-level CAC: total spend, deals won, revenue attributed, blended CAC, ROAS.

### Payback Period

`payback_tab` — estimated payback period by channel based on average deal size. Methodology note included.

### Attribution Issues

`attribution_issues` summary + `unmatched_source_detail` — deals with no attribution, low-confidence matches, and ad spend with no CRM match.

### Exec Summary

`exec_summary` — total spend, revenue, attribution rate, blended CAC, overall ROAS, attribution gap.

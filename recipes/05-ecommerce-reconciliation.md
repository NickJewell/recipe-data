---
recipe: Multi-Store Ecommerce Sales Reconciliation (Shopify + Amazon + Etsy)
version: 1.0.0
cadence: monthly
villain: "Manually copying orders from three dashboards into one spreadsheet each month to figure out which channel is actually profitable after fees"
sources:
  - name: shopify_orders
    description: Shopify orders export — one row per order
    format: csv
    required_columns:
      - Name
      - Created at
      - Total
      - Lineitem name
      - Lineitem sku
      - Lineitem quantity
    common_issues:
      - Shopify exports in UTC — convert to local timezone if needed
      - Refunded orders appear as separate rows with negative amounts
      - Discount codes appear as line items with negative values
  - name: amazon_orders
    description: Amazon Seller Central order report — one row per order item
    format: txt or csv
    required_columns:
      - amazon-order-id
      - purchase-date
      - item-price
      - product-name
      - asin
      - quantity-purchased
    common_issues:
      - Amazon exports as tab-delimited .txt — the recipe handles both CSV and TSV
      - item-price is per unit; total = item-price * quantity-purchased
      - Amazon FBA fees are in a separate "Fee Preview" report — bind as amazon_fees if available
  - name: etsy_orders
    description: Etsy Orders CSV export — one row per order
    format: csv
    required_columns:
      - Order ID
      - Sale Date
      - Order Value
      - Item Name
      - SKU
      - Quantity
    common_issues:
      - Etsy includes transaction fees in the export under "Fees & Taxes"
      - Order Value includes shipping — recipe separates these if "Shipping" column present
      - Etsy exports currencies as symbols ("$", "£") — recipe strips these
output: ecommerce-reconciliation.xlsx
output_tabs:
  - Source Data (Shopify)
  - Source Data (Amazon)
  - Source Data (Etsy)
  - Combined Orders
  - Sales by Channel
  - Sales by SKU
  - Fees and Net Revenue
  - Returns Rate
  - Inventory Sell-through
  - Exec Summary
tags:
  - ecommerce
  - Shopify
  - Amazon
  - Etsy
  - reconciliation
  - monthly
---

# Multi-Store Ecommerce Sales Reconciliation

You sell on three channels. Each one has a different dashboard, a different export format, different fee structures, and different SKU naming conventions. To answer "which channel made us the most money last month?" you currently download three reports, paste them into a spreadsheet, manually apply fee rates, and spend an hour trying to reconcile SKUs that are named differently on each platform. This recipe does all of that in one run.

The key challenge is that fee structures differ completely between platforms — Shopify takes a payment processing fee (2.9% + 30c), Amazon charges referral fees (8–15%), FBA fulfilment fees (per unit), and a monthly subscription; Etsy charges a listing fee (20c), transaction fee (6.5%), and payment processing (3%). The recipe encodes each as a configurable rate table in the frontmatter, so you can update rates without touching the step code.

## Fee Configuration

Update these rates to match your actual fee agreements before running:

```
SHOPIFY_FEE_RATE = 0.029        # Shopify Payments: 2.9% + $0.30
SHOPIFY_FEE_FIXED = 0.30        # Per-transaction fixed fee
AMAZON_REFERRAL_RATE = 0.15     # Amazon referral fee (category-specific; 15% is electronics/most)
AMAZON_FBA_FEE_PER_UNIT = 3.22  # Average FBA fulfilment fee per unit (varies by size/weight)
ETSY_LISTING_FEE = 0.20         # Per listing per 4 months ($0.20)
ETSY_TRANSACTION_RATE = 0.065   # Etsy transaction fee: 6.5%
ETSY_PAYMENT_RATE = 0.03        # Etsy Payments processing: 3% + $0.25
ETSY_PAYMENT_FIXED = 0.25       # Per-transaction fixed fee
```

## Sources

### shopify_orders

Download from: Shopify Admin → Orders → Export → All time or date range → CSV.

Key columns in a Shopify export:
- `Name` — order number (#1001, #1002, ...)
- `Created at` — order timestamp (UTC)
- `Total` — order total including shipping
- `Subtotal` — order total excluding shipping
- `Shipping` — shipping charged to customer
- `Taxes` — tax collected
- `Lineitem name` — product name
- `Lineitem sku` — SKU
- `Lineitem quantity` — units ordered
- `Lineitem price` — price per unit
- `Refunded Amount` — if this is a refund row

### amazon_orders

Download from: Seller Central → Reports → Fulfillment → Orders → Download → Date range.

The download is a tab-delimited .txt file. Key columns:
- `amazon-order-id` — order identifier
- `purchase-date` — order date (ISO format)
- `product-name` — listing title
- `asin` — Amazon product identifier
- `sku` — your SKU (may differ from Shopify)
- `quantity-purchased` — units ordered
- `item-price` — price per unit (multiply by quantity for order total)
- `item-tax` — tax per unit
- `shipping-price` — shipping charged

### etsy_orders

Download from: Etsy Seller Dashboard → Orders & Shipping → Export orders → CSV.

Key columns:
- `Order ID` — Etsy order number
- `Sale Date` — order date
- `Order Value` — order total
- `Item Name` — product title
- `SKU` — your SKU (may differ from other platforms)
- `Quantity` — units ordered
- `Shipping Charges` — shipping to customer (if present)
- `Fees & Taxes` — Etsy fees deducted

## Steps

### Step 1: Load Shopify orders

```python
import pandas as pd
import re

# Load Shopify CSV
df_shopify = pd.read_csv(source_shopify_orders, encoding='utf-8', low_memory=False)
df_shopify_source = df_shopify.copy()
df_shopify.columns = df_shopify.columns.str.strip()

# Fee configuration
SHOPIFY_FEE_RATE = 0.029
SHOPIFY_FEE_FIXED = 0.30
AMAZON_REFERRAL_RATE = 0.15
AMAZON_FBA_FEE_PER_UNIT = 3.22
ETSY_LISTING_FEE = 0.20
ETSY_TRANSACTION_RATE = 0.065
ETSY_PAYMENT_RATE = 0.03
ETSY_PAYMENT_FIXED = 0.25

def parse_amount(val):
    """Strip currency symbols and parse to float."""
    if pd.isna(val):
        return 0.0
    return float(re.sub(r'[^\d.\-]', '', str(val)) or 0)

def parse_date_safe(val):
    if pd.isna(val):
        return pd.NaT
    try:
        return pd.Timestamp(str(val)[:10])  # take YYYY-MM-DD prefix
    except Exception:
        return pd.NaT

# Shopify: one row per line item — group to order level for order metrics
# Keep line-item level for SKU analysis

df_shopify['order_number'] = df_shopify['Name'].astype(str)
df_shopify['order_date'] = df_shopify['Created at'].apply(parse_date_safe)
df_shopify['order_month'] = df_shopify['order_date'].dt.strftime('%Y-%m')
df_shopify['gross_revenue'] = df_shopify.get('Subtotal', df_shopify.get('Total', pd.Series())).apply(parse_amount)
df_shopify['shipping_revenue'] = df_shopify.get('Shipping', pd.Series(0, index=df_shopify.index)).apply(parse_amount)
df_shopify['sku'] = df_shopify.get('Lineitem sku', '').astype(str).str.strip()
df_shopify['product_name'] = df_shopify.get('Lineitem name', '').astype(str).str.strip()
df_shopify['quantity'] = pd.to_numeric(df_shopify.get('Lineitem quantity', 1), errors='coerce').fillna(1)
df_shopify['unit_price'] = df_shopify.get('Lineitem price', pd.Series()).apply(parse_amount)
df_shopify['channel'] = 'Shopify'

# Mark refunds
df_shopify['is_refund'] = df_shopify.get('Refunded Amount', pd.Series(0, index=df_shopify.index)).apply(parse_amount) > 0

# Calculate fees
df_shopify['platform_fee'] = df_shopify['gross_revenue'] * SHOPIFY_FEE_RATE + SHOPIFY_FEE_FIXED
df_shopify['net_revenue'] = df_shopify['gross_revenue'] - df_shopify['platform_fee']

print(f"Shopify: {df_shopify['order_number'].nunique()} orders, {df_shopify['gross_revenue'].sum():,.2f} gross revenue")
```

**Preview:** Check order count and gross revenue against your Shopify dashboard. Note: each order appears multiple times (one row per line item).

### Step 2: Load Amazon orders

```python
# Amazon exports as tab-delimited .txt
try:
    df_amazon = pd.read_csv(source_amazon_orders, sep='\t', encoding='utf-8', low_memory=False)
except Exception:
    df_amazon = pd.read_csv(source_amazon_orders, encoding='utf-8', low_memory=False)

df_amazon_source = df_amazon.copy()
df_amazon.columns = df_amazon.columns.str.strip()

df_amazon['order_number'] = df_amazon.get('amazon-order-id', df_amazon.get('Order ID', '')).astype(str)
df_amazon['order_date'] = df_amazon.get('purchase-date', df_amazon.get('Order Date', '')).apply(parse_date_safe)
df_amazon['order_month'] = df_amazon['order_date'].dt.strftime('%Y-%m')
df_amazon['quantity'] = pd.to_numeric(df_amazon.get('quantity-purchased', df_amazon.get('Quantity', 1)), errors='coerce').fillna(1)
df_amazon['unit_price'] = df_amazon.get('item-price', df_amazon.get('Unit Price', pd.Series())).apply(parse_amount)
df_amazon['gross_revenue'] = df_amazon['unit_price'] * df_amazon['quantity']
df_amazon['shipping_revenue'] = df_amazon.get('shipping-price', pd.Series(0, index=df_amazon.index)).apply(parse_amount)
df_amazon['sku'] = df_amazon.get('sku', df_amazon.get('SKU', '')).astype(str).str.strip()
df_amazon['product_name'] = df_amazon.get('product-name', df_amazon.get('Product Name', '')).astype(str).str.strip()
df_amazon['channel'] = 'Amazon'
df_amazon['is_refund'] = False

# Amazon fees: referral fee (%) + FBA fee (per unit)
df_amazon['referral_fee'] = df_amazon['gross_revenue'] * AMAZON_REFERRAL_RATE
df_amazon['fba_fee'] = df_amazon['quantity'] * AMAZON_FBA_FEE_PER_UNIT
df_amazon['platform_fee'] = df_amazon['referral_fee'] + df_amazon['fba_fee']
df_amazon['net_revenue'] = df_amazon['gross_revenue'] - df_amazon['platform_fee']

print(f"Amazon: {df_amazon['order_number'].nunique()} orders, {df_amazon['gross_revenue'].sum():,.2f} gross revenue")
```

**Preview:** Check against Amazon Seller Central. Note: Amazon fees in this recipe are estimates — the actual fees depend on product category and size tier. Bind the Fee Preview report as a fourth source for exact figures.

### Step 3: Load Etsy orders

```python
df_etsy = pd.read_csv(source_etsy_orders, encoding='utf-8', low_memory=False)
df_etsy_source = df_etsy.copy()
df_etsy.columns = df_etsy.columns.str.strip()

# Etsy column name variants
ETSY_COL_MAP = {
    'order id': 'order_number',
    'sale date': 'order_date',
    'order value': 'gross_revenue',
    'item name': 'product_name',
    'quantity': 'quantity',
    'shipping charges': 'shipping_revenue',
    'fees & taxes': 'etsy_fees_reported',
}
df_etsy.columns = df_etsy.columns.str.lower().str.strip()
df_etsy = df_etsy.rename(columns=ETSY_COL_MAP)

df_etsy['order_number'] = df_etsy['order_number'].astype(str)
df_etsy['order_date'] = df_etsy['order_date'].apply(parse_date_safe)
df_etsy['order_month'] = df_etsy['order_date'].dt.strftime('%Y-%m')
df_etsy['gross_revenue'] = df_etsy['gross_revenue'].apply(parse_amount)
df_etsy['shipping_revenue'] = df_etsy.get('shipping_revenue', pd.Series(0, index=df_etsy.index)).apply(parse_amount)
df_etsy['quantity'] = pd.to_numeric(df_etsy.get('quantity', 1), errors='coerce').fillna(1)
df_etsy['sku'] = df_etsy.get('sku', '').astype(str).str.strip()
df_etsy['product_name'] = df_etsy.get('product_name', '').astype(str).str.strip()
df_etsy['channel'] = 'Etsy'
df_etsy['is_refund'] = df_etsy['gross_revenue'] < 0

# Etsy fees: use reported fees if available, otherwise calculate
if 'etsy_fees_reported' in df_etsy.columns:
    df_etsy['platform_fee'] = df_etsy['etsy_fees_reported'].apply(parse_amount).abs()
else:
    df_etsy['platform_fee'] = (
        df_etsy['gross_revenue'] * (ETSY_TRANSACTION_RATE + ETSY_PAYMENT_RATE)
        + ETSY_PAYMENT_FIXED
        + ETSY_LISTING_FEE
    )
df_etsy['net_revenue'] = df_etsy['gross_revenue'] - df_etsy['platform_fee']

print(f"Etsy: {df_etsy['order_number'].nunique()} orders, {df_etsy['gross_revenue'].sum():,.2f} gross revenue")
```

**Preview:** Check against Etsy's monthly statement. Etsy's reported fee column (if present) is more accurate than the calculated estimate.

### Checkpoint: All three channels loaded

```
Assert: len(df_shopify) > 0 and len(df_amazon) > 0 and len(df_etsy) > 0
On failure: One or more source files is empty after loading. Check the file path and that
            the export covers the date range you expect.
```

### Step 4: Normalise to common schema and combine

All three dataframes now get the same columns so they can stack cleanly.

```python
COMMON_COLS = [
    'channel', 'order_number', 'order_date', 'order_month',
    'product_name', 'sku', 'quantity',
    'unit_price', 'gross_revenue', 'shipping_revenue',
    'platform_fee', 'net_revenue', 'is_refund',
]

def select_common_cols(df):
    """Select common columns, filling missing ones with 0 or empty string."""
    for col in COMMON_COLS:
        if col not in df.columns:
            df[col] = 0 if col in ['quantity', 'unit_price', 'gross_revenue', 'shipping_revenue', 'platform_fee', 'net_revenue'] else ''
    return df[COMMON_COLS].copy()

df_shopify_norm = select_common_cols(df_shopify)
df_amazon_norm = select_common_cols(df_amazon)
df_etsy_norm = select_common_cols(df_etsy)

combined = pd.concat([df_shopify_norm, df_amazon_norm, df_etsy_norm], ignore_index=True)

# Separate returns from sales
returns = combined[combined['is_refund']].copy()
sales = combined[~combined['is_refund']].copy()

print(f"\nCombined: {len(sales)} sale rows, {len(returns)} return rows")
print(f"Channels: {sales['channel'].value_counts().to_dict()}")
print(f"Total gross revenue: {sales['gross_revenue'].sum():,.2f}")
print(f"Total net revenue: {sales['net_revenue'].sum():,.2f}")
```

**Preview:** Combined row count should equal Shopify + Amazon + Etsy rows. Check that gross and net revenue totals are plausible.

### Checkpoint: Net revenue less than gross

```
Assert: combined['net_revenue'].sum() < combined['gross_revenue'].sum()
On failure: Net revenue is greater than or equal to gross revenue, which means fees
            were calculated as negative (added rather than subtracted). Check the
            fee configuration at the top of this recipe.
```

### Step 5: Sales by channel

Monthly breakdown by channel — the primary comparison view.

```python
sales_by_channel = (
    sales.groupby(['channel', 'order_month'])
    .agg(
        order_count=('order_number', 'nunique'),
        units_sold=('quantity', 'sum'),
        gross_revenue=('gross_revenue', 'sum'),
        platform_fees=('platform_fee', 'sum'),
        net_revenue=('net_revenue', 'sum'),
    )
    .reset_index()
)
sales_by_channel['fee_rate'] = (
    sales_by_channel['platform_fees'] / sales_by_channel['gross_revenue'] * 100
).round(1).astype(str) + '%'
sales_by_channel['avg_order_value'] = (
    sales_by_channel['gross_revenue'] / sales_by_channel['order_count']
).round(2)
sales_by_channel = sales_by_channel.sort_values(['order_month', 'channel'])
sales_by_channel.columns = [c.replace('_', ' ').title() for c in sales_by_channel.columns]
```

**Preview:** Check that each channel appears for each month in your date range. Large month-on-month swings are worth flagging.

### Step 6: Sales by SKU across channels

Which products sell best, and on which channel? This view shows where
each SKU performs — useful for pricing and inventory decisions.

```python
# Normalise SKU (some systems add dashes, spaces, or suffixes)
sales['sku_norm'] = sales['sku'].str.upper().str.strip().str.replace(r'[-_\s]+', '-', regex=True)

sales_by_sku = (
    sales.groupby(['sku_norm', 'product_name', 'channel'])
    .agg(
        units_sold=('quantity', 'sum'),
        gross_revenue=('gross_revenue', 'sum'),
        net_revenue=('net_revenue', 'sum'),
        avg_unit_price=('unit_price', 'mean'),
    )
    .reset_index()
    .sort_values(['sku_norm', 'channel'])
)

# Pivot to show channels side by side
sku_pivot = sales_by_sku.pivot_table(
    index=['sku_norm', 'product_name'],
    columns='channel',
    values=['units_sold', 'net_revenue'],
    fill_value=0,
).reset_index()
sku_pivot.columns = [
    f"{col[1]} {col[0]}".strip() if col[1] else col[0]
    for col in sku_pivot.columns
]
sku_pivot['Total Units'] = sku_pivot[[c for c in sku_pivot.columns if 'units_sold' in c]].sum(axis=1)
sku_pivot['Total Net Revenue'] = sku_pivot[[c for c in sku_pivot.columns if 'net_revenue' in c]].sum(axis=1)
sku_pivot = sku_pivot.sort_values('Total Net Revenue', ascending=False).reset_index(drop=True)
```

**Preview:** SKUs should appear in all channels where you sell them. A SKU present on Shopify but absent on Amazon might be an untapped listing opportunity.

### Step 7: Fees and net revenue breakdown

Transparent fee accounting by channel — the table that explains where
gross revenue goes before reaching your bank account.

```python
fees_summary = (
    sales.groupby('channel')
    .agg(
        gross_revenue=('gross_revenue', 'sum'),
        shipping_revenue=('shipping_revenue', 'sum'),
        platform_fees=('platform_fee', 'sum'),
        net_revenue=('net_revenue', 'sum'),
        orders=('order_number', 'nunique'),
        units=('quantity', 'sum'),
    )
    .reset_index()
)
fees_summary['effective_fee_rate'] = (
    fees_summary['platform_fees'] / fees_summary['gross_revenue'] * 100
).round(1).astype(str) + '%'
fees_summary['net_margin'] = (
    fees_summary['net_revenue'] / fees_summary['gross_revenue'] * 100
).round(1).astype(str) + '%'

# Add totals row
totals = fees_summary[['gross_revenue', 'shipping_revenue', 'platform_fees', 'net_revenue', 'orders', 'units']].sum()
totals_row = pd.DataFrame([['TOTAL'] + totals.tolist() + ['', '']], columns=fees_summary.columns)
fees_breakdown = pd.concat([fees_summary, totals_row], ignore_index=True)
fees_breakdown.columns = [c.replace('_', ' ').title() for c in fees_breakdown.columns]
```

**Preview:** Net Revenue should equal Gross Revenue minus Platform Fees. Check the effective fee rate matches what you expect from each platform's pricing.

### Step 8: Returns rate by channel

Track return rates — high return rates erode profitability faster than fees.

```python
returns_by_channel = (
    returns.groupby('channel')
    .agg(
        return_count=('order_number', 'nunique'),
        return_value=('gross_revenue', lambda x: x.abs().sum()),
    )
    .reset_index()
)

sales_counts = sales.groupby('channel').agg(
    sale_count=('order_number', 'nunique'),
    sale_value=('gross_revenue', 'sum'),
).reset_index()

returns_rate = pd.merge(returns_by_channel, sales_counts, on='channel', how='right').fillna(0)
returns_rate['Return Rate (units)'] = (
    returns_rate['return_count'] / returns_rate['sale_count'] * 100
).round(1).astype(str) + '%'
returns_rate['Return Rate (value)'] = (
    returns_rate['return_value'] / returns_rate['sale_value'] * 100
).round(1).astype(str) + '%'
returns_rate.columns = [c.replace('_', ' ').title() for c in returns_rate.columns]
```

**Preview:** Return rate above 5% (value) for most physical product categories is worth investigating. Amazon typically has higher return rates than DTC channels.

### Step 9: Inventory sell-through estimate

Estimate sell-through rate from units sold vs a simple reorder-point view.
(Full inventory tracking requires a beginning inventory balance — bind as a
fourth source if you have it.)

```python
sku_units = (
    sales.groupby(['sku_norm', 'product_name', 'channel'])['quantity']
    .sum()
    .reset_index()
    .sort_values('quantity', ascending=False)
)
sku_units.columns = ['SKU', 'Product Name', 'Channel', 'Units Sold']

# Monthly run rate
months_in_data = sales['order_month'].nunique()
sku_units['Monthly Run Rate'] = (sku_units['Units Sold'] / max(months_in_data, 1)).round(1)

inventory_note = pd.DataFrame([{
    'Note': 'Sell-through rate requires beginning inventory balance. '
            'Add an inventory source (inventory_snapshot) to the recipe frontmatter '
            'to calculate: Sell-Through % = Units Sold / (Beginning Inventory + Units Received).'
}])
inventory_tab = pd.concat([sku_units, inventory_note], ignore_index=True)
```

**Preview:** Monthly run rate by SKU is useful for reorder planning even without a beginning inventory balance.

### Step 10: Build Exec Summary

```python
import datetime

total_gross = sales['gross_revenue'].sum()
total_fees = sales['platform_fee'].sum()
total_net = sales['net_revenue'].sum()
total_orders = sales['order_number'].nunique()
total_units = sales['quantity'].sum()
return_rate = len(returns) / (len(sales) + len(returns)) * 100 if (len(sales) + len(returns)) > 0 else 0
best_channel = fees_summary.sort_values('net_revenue', ascending=False).iloc[0]['channel'] if len(fees_summary) > 0 else 'N/A'

exec_summary = pd.DataFrame([
    ['Total Gross Revenue', f"{total_gross:,.2f}"],
    ['Total Platform Fees', f"{total_fees:,.2f}"],
    ['Total Net Revenue', f"{total_net:,.2f}"],
    ['Effective Fee Rate', f"{total_fees/total_gross*100:.1f}%" if total_gross > 0 else 'N/A'],
    ['Total Orders', f"{total_orders:,}"],
    ['Total Units Sold', f"{total_units:,}"],
    ['Return Rate (units)', f"{return_rate:.1f}%"],
    ['Highest Net Revenue Channel', best_channel],
    ['Months Covered', f"{months_in_data}"],
    ['', ''],
    ['Run Date', datetime.date.today().isoformat()],
    ['Shopify Source', source_shopify_orders],
    ['Amazon Source', source_amazon_orders],
    ['Etsy Source', source_etsy_orders],
], columns=['Metric', 'Value'])
```

## Output Tabs

### Source Data (Shopify)

`df_shopify_source` — Shopify orders as loaded.

### Source Data (Amazon)

`df_amazon_source` — Amazon order report as loaded.

### Source Data (Etsy)

`df_etsy_source` — Etsy orders CSV as loaded.

### Combined Orders

`combined` — all three channels in a normalised schema. One row per line item. Includes `channel`, `order_number`, `order_date`, `sku`, `quantity`, `gross_revenue`, `platform_fee`, `net_revenue`, `is_refund`.

### Sales by Channel

`sales_by_channel` — monthly summary by channel. Columns: Channel | Order Month | Order Count | Units Sold | Gross Revenue | Platform Fees | Net Revenue | Fee Rate | Avg Order Value.

### Sales by SKU

`sku_pivot` — SKU performance pivoted by channel. Columns: SKU | Product Name | [Channel] Units Sold | [Channel] Net Revenue | Total Units | Total Net Revenue. Sorted by total net revenue descending.

### Fees and Net Revenue

`fees_breakdown` — channel-level fee accounting. Columns: Channel | Gross Revenue | Shipping Revenue | Platform Fees | Net Revenue | Orders | Units | Effective Fee Rate | Net Margin.

### Returns Rate

`returns_rate` — return count and value by channel with return rate. Columns: Channel | Return Count | Return Value | Sale Count | Sale Value | Return Rate (units) | Return Rate (value).

### Inventory Sell-through

`inventory_tab` — units sold and monthly run rate by SKU. Note included explaining how to enable full sell-through calculation with a beginning inventory source.

### Exec Summary

`exec_summary` — total gross, fees, net revenue, effective fee rate, orders, units, return rate, best channel, run date, source files.

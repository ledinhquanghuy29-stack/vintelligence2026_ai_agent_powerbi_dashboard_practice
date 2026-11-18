# Supply Chain & Sales Performance Dashboard

A Power BI dashboard built from scratch on a synthetic e-commerce fashion dataset (13 CSV files, ~2.5M rows, Jul 2012 – Dec 2022). 

**Stack:** Power BI (PBIP/TMDL project format), Power Query, DAX, Python (data profiling)

---

## What this project shows

- End-to-end analytics workflow: raw CSVs → data profiling → agreed KPI definitions → star-schema semantic model → DAX measures → a 6-page interactive report.
- Every KPI is backed by a written definition and a reconciliation check (see [Data Quality page](#dashboard-pages)) — Net Revenue, Gross Margin %, and COGS are tied out against independent daily benchmark files down to the dollar before being trusted in any visual.
- The whole semantic model and report were generated programmatically (Python scripts writing TMDL and PBIR JSON directly), which made every table, relationship, and measure reviewable as plain text and reproducible from the raw data.

---

## Dataset

| Group | Tables |
|---|---|
| Orders & fulfilment | `orders`, `order_items`, `shipments`, `returns` |
| Inventory | `inventory` (monthly product snapshot) |
| Customers & marketing | `customers`, `reviews`, `web_traffic`, `promotions` |
| Reference | `products`, `geography` |
| Reconciliation only | `sales`, `payments` (daily/independent totals used to validate the model) |

```
customers ──< orders >── geography (zip)
                │
   ┌────────┬───┼────────┬─────────┐
 order_items  payments  shipments  returns / reviews
   │ └── promotions
 products ──< inventory
```

## Business questions

| Area | Question |
|---|---|
| Sales & margin | How do revenue and margin trend, and how much of that movement comes from promotions vs. product mix? |
| Delivery | What share of orders arrives within the agreed service level, and what does shipping cost look like? |
| Inventory | Where is stock overstocked or understocked, and is stock growing faster than sales? |
| Returns | What share of orders comes back, why, and how soon? |
| Customers | How many customers are active, how many are repeat buyers, and does acquisition channel predict customer value? |
| Satisfaction | What do ratings look like, and how much of the catalog do they actually cover? |

---

## Key findings

- **Revenue grew to a 2016 peak, then order volume dropped ~40% from 2019** while average order value kept climbing — fewer, larger orders, not a demand collapse.
- **Inventory kept building while sales fell**: median days-of-supply rose from 96 (2012) to 454 (2020). Stock and demand moved in opposite directions for years — the central supply chain story in this dataset.
- **Promotions run at a loss**: lines with a promotion average **−14% gross margin**, vs. **+20%** without. ~27% of all order lines sell below cost.
- **A margin decline in 2021–2022 traced to product mix, not price or cost**: a shift toward lower-margin SKUs explains the drop almost exactly (confirmed with a shift-share decomposition), while list price and unit cost stayed constant.
- **Delivery and return rates are flat across every dimension tested** (region, category, size, year) — a useful finding in itself: there's no weak link to chase, so the dashboard tracks these as stable KPIs rather than manufacturing a false trend.
- **Reviews cover only 17% of orders and 59% of products** — ratings are directional, not a full quality signal.
- **Web traffic can't be tied to orders**: daily sessions correlate only 0.15 with daily delivered orders and share no join key, so it's kept to a small reference panel rather than a funnel narrative the data can't support.

Full findings, with every supporting number, are in [`Dashboard/DASHBOARD_INSTRUCTION.md`](Dashboard/DASHBOARD_INSTRUCTION.md).

---

## Dashboard pages

| Page | Covers |
|---|---|
| **Executive Overview** | Revenue by year, margin, delivery SLA %, return rate, distinct customers, repeat rate |
| **Sales and Margin** | Revenue trend, seasonality, product mix, promotion effect, promo-channel breakdown |
| **Delivery and Returns** | Lead time vs. the 7-day SLA, shipping cost, return reasons and refunds |
| **Inventory** | Monthly stock snapshot, days of supply, a sorted, filtered list of currently overstocked SKUs |
| **Customers and Satisfaction** | New vs. repeat customers, acquisition channel, ratings, review coverage, web traffic (reference) |
| **Data Quality and Definitions** | Reconciliation checks, KPI definitions, data exceptions |

All six pages share one 1,920×1,300 canvas and one colour theme.

---

## Key decisions

A few judgment calls an analyst has to make before the numbers mean anything — documented in full in the instruction file:

| Decision | Choice | Why |
|---|---|---|
| Revenue recognition | Only `delivered` orders | The customer actually received the goods; conservative |
| Delivery SLA | ≤ 7 days | No promised-delivery-date field exists in the data, so this is an agreed target, not a contractual one |
| Stockout | `stockout_days` ≥ 3 | At ≥ 1 day the rate hits 67% — too sensitive to be useful |
| Overstock | `days_of_supply` > 180 | Median days of supply is 240, so this threshold actually separates cases |
| Currency | VND for prices/costs, USD for shipping fee | Shipping fee is ~1,000x smaller than price columns — clearly a different scale |

## Repository layout

```
Power BI with Ai/
├── Data/                            13 source CSV files (read-only)
├── Dashboard/
│   ├── RetailOps_Dashboard.pbip     Power BI project
│   ├── DASHBOARD_INSTRUCTION.md     Data model, DAX measures, page design
│   └── BUILD_LOG.md                 Round-by-round build history: bugs found and fixed, lessons learned
└── README.md                        This file
```

## Opening the project

1. Open `Dashboard/RetailOps_Dashboard.pbip` in Power BI Desktop.
2. The data source parameter (`DataFolder`) points at `Data/`; if you move the repository, update that one parameter in Power Query.

## Known limitations

- The dataset is synthetic: several dimensions (lead time, return rate by category) are statistically flat by construction. The dashboard tracks these honestly rather than inventing a pattern that isn't there.
- Inventory is a monthly snapshot and covers roughly a third the volume of actual order quantities — it's never compared directly to order-level numbers.
- Cost is a static catalog value per product, so cost *trends* can't be analyzed from this data; only price and product-mix effects on margin can be.

# Dashboard Design & Data Model

Technical reference for the Supply Chain & Sales Performance dashboard: the semantic model, the DAX measures, and the page-by-page design. For findings and business context, see the main [README](../README.md). For the round-by-round build history — bugs found, root causes, and fixes — see [`BUILD_LOG.md`](BUILD_LOG.md).

---

## 1. Data model

Star schema, single-direction relationships (dimension → fact).

```
Date[Date]            -> Orders[order_date]        (active)
Date[Date]            -> Inventory[snapshot_date]   (active)
Date[Date]            -> WebTraffic[date]           (active)
Return Date[Date]     -> Returns[return_date]       (active)
Review Date[Date]     -> Reviews[review_date]       (active)
Geography[zip]        -> Orders[zip]
Orders[order_id]      -> OrderItems[order_id]
Orders[order_id]      -> Shipments[order_id]
Orders[order_id]      -> Returns[order_id]
Orders[customer_id]   -> Customers[customer_id]
Orders[order_id]      -> Reviews[order_id]          (inactive — see note)
Products[product_id]  -> OrderItems / Returns / Inventory / Reviews [product_id]
Promotions[promo_id]  -> OrderItems[promo_id]        (active)
Promotions[promo_id]  -> OrderItems[promo_id_2]      (inactive — a table pair allows only one active relationship)
```

`Orders → Reviews` is inactive because `Reviews` already reaches `Orders` indirectly and keeping both active creates an ambiguous filter path through `OrderItems` (which also relates to both `Orders` and `Products`). `Customers[zip]` is **not** related to `Geography`, since order-level geography already exists via `Orders[zip]` and a second path would be ambiguous too.

### Filter reach
Not every slicer reaches every page — filters only flow from the "one" side to the "many" side:

| Slicer | Sales | Delivery (Shipments) | Returns | Inventory | Reviews |
|---|---|---|---|---|---|
| Year / Month | ✅ (order date) | ✅ (order date) | ✅ (order cohort) | ✅ (snapshot date) | ❌ (own `Review Date`) |
| Category / Product | ✅ | ❌ | ✅ | ✅ | ✅ |
| Region (Geography) | ✅ | ✅ | ✅ | ❌ | ❌ |
| Acquisition Channel | ✅ | ✅ | ✅ | ❌ | ❌ |
| Promo Channel | ✅ (OrderItems only) | ❌ | ❌ | ❌ | ❌ |

### Data preparation
- A single Power Query parameter, `DataFolder`, holds the absolute path to `Data/`; every CSV source is built from it.
- Date columns arrive as text and are cast to `Date` on import.
- `OrderItems` has 16 duplicate (`order_id`, `product_id`) lines — kept as separate rows, no synthetic line key was added.
- `products.price` matches `order_items.unit_price` in only ~24% of lines, so revenue is always computed from `unit_price`, never from the catalog price.
- `inventory` carries its own copies of `product_name`/`category` — hidden; `Products` is the source of truth for those attributes.

---

## 2. Agreed KPI definitions

| Topic | Definition |
|---|---|
| Revenue recognition | Only `order_status = "delivered"` |
| Net Revenue | `Σ(quantity × unit_price) − Σ(discount_amount)` (verified against an independent payments total) |
| Cost basis | `Σ(quantity × products.cogs)` — a static catalog cost per SKU |
| Delivery SLA | Lead time (`delivery_date − order_date`) ≤ 7 days |
| Stockout | `stockout_days ≥ 3` in a product-month |
| Overstock | `days_of_supply > 180` |
| Return rate | Units returned ÷ units sold, over `delivered` + `returned` orders, excluding orders placed in the last 31 days of data (their return window — max observed order-to-return lag is 31 days — hadn't closed) |
| Currency | All money values in $ (source data states no currency); `shipping_fee` is on a much smaller scale and never summed with prices |
| Calendar | Standard Jan–Dec year; 2012 is a partial year (Jul–Dec only) |

---

## 3. Calculated columns

| Table | Column | Logic |
|---|---|---|
| `OrderItems` | `line_gross` / `line_net` / `line_cost` | Standard revenue/cost line math |
| `OrderItems` | `has_promo` | `LEN(promo_id) > 0 OR LEN(promo_id_2) > 0` (empty CSV cells load as `""`, not blank, so `ISBLANK` can't be used) |
| `OrderItems` | `promo_channel_label` | `Promotions[promo_channel]` via `RELATED`, or `"No Promotion"` if unmatched |
| `Products` | `list_margin` / `list_margin_tier` | `1 − cogs/price`, bucketed into Low/Mid/High terciles |
| `Shipments` | `lead_days` / `processing_days` / `transit_days` | Date differences vs. the related order |
| `Returns` | `days_to_return` | `return_date − order_date` |
| `Customers` | `first_delivered_order_date` | `MIN(Orders[order_date])` filtered to delivered, via relationship context transition |
| `Inventory` | `is_latest_overstock` | Flags rows at the latest snapshot with `days_of_supply > 180`, used to drive the Inventory page's exception table |

---

## 4. DAX measure catalog

### Sales & Margin
| Measure | DAX |
|---|---|
| Net Revenue | `[Gross Sales] − [Discount Amount]` |
| Gross Margin % | `DIVIDE([Gross Profit], [Net Revenue])` |
| AOV | `DIVIDE([Net Revenue], [Order Count])` |
| Promo Revenue Share % | `DIVIDE([Promo Net Revenue], [Net Revenue])` |
| Weighted List Margin (validation) | Average `Products[list_margin]` weighted by non-promo revenue — should equal `Gross Margin % (Non-promo)` exactly |
| Seasonality Index | Average revenue for a calendar month across years, divided by the true grand average (`ALL('Date')`, not a specific-column `ALLSELECTED`, to avoid filter-context bugs) |

### Delivery
| Measure | DAX |
|---|---|
| Avg Lead Time | `CALCULATE(AVERAGE(Shipments[lead_days]), Delivered)` |
| Delivered Within SLA % | Delivered shipments with `lead_days ≤ [SLA Days]` ÷ all delivered shipments, both restricted to order dates ≤ (last delivery date − 10); max observed lead time is 10 days, so later orders only show the ones that happened to arrive fast |
| High-Fee Order Share % | Share of shipped orders with `shipping_fee ≥ 10` |

### Returns
| Measure | DAX |
|---|---|
| Return Rate % | `DIVIDE([Return Units], [Units Received])`, over `delivered` + `returned` orders, both restricted to order dates ≤ (last return date − 31) |
| Refund % of Net | Refund amount over net revenue of the same order cohort |

### Inventory
| Measure | DAX |
|---|---|
| Ending Stock | `SUM(stock_on_hand)` at the latest snapshot only (semi-additive — never summed across dates) |
| Stockout / Overstock Rate % | Share of product-months past the agreed threshold (both driven by what-if parameters) |
| Median Days of Supply | `MEDIAN`, not `AVERAGE` — the max value is 68,100 days, an extreme outlier |
| Sell-Through % | Volume-weighted: `SUM(units_sold) / (SUM(stock_on_hand) + SUM(units_sold))` |

### Customers & Satisfaction
| Measure | DAX |
|---|---|
| Distinct Customers | Delivered-order customers |
| Repeat Customer Rate % | Customers with > 1 delivered order ÷ all buying customers |
| New Customers | Customers whose `first_delivered_order_date` falls inside the current Date filter |
| Average Rating / Review Count | `AVERAGE`/`COUNTROWS` on `Reviews`, filtered with `TREATAS(VALUES(Orders[order_id]), Reviews[order_id])` so reviews follow the order-date, region and channel filters. The `Orders → Reviews` relationship stays inactive (activating it recreates the Round 4 ambiguous path) |

What-if parameters exposed as slicers: **SLA Days** (default 7), **Stockout Days Threshold** (default 3), **Overstock Days Threshold** (default 180).

---

## 5. Page design

All six pages share one 1,920×1,300 canvas, a page navigator in the header, a shared top-down layout (title → purpose line → slicers → KPI cards, each with a YoY or vs-target line → main trend → driver charts → a detail table), and one custom colour theme.

| Page | Main trend | Drivers | Detail |
|---|---|---|---|
| **Executive Overview** | Slicers: year, month, region, category, sales channel. Net Revenue by month across all 10 years; KPI cards with a YoY or vs-target status line | Median days of supply (rolling 12 months), Net Revenue by category | Gross margin % by category, overstock rate % by category vs the 30% target, gross margin % (rolling 12 months). No tables on this page |
| **Sales and Margin** | Net Revenue + Order Count combo; gross margin % by monthly promo revenue share band (under 20% / 20–60% / 60% and above) | Seasonality index, product-mix tier, gross margin % by promo status | Category table + gross margin % by year, bars coloured by promotion intensity (amber = 6-campaign years, navy = 4) + Net Revenue by Promo Channel |
| **Delivery and Returns** | SLA % and Return Rate % by month, each with its target drawn as a second line | Order status mix, shipping fee group, refund amount by return reason (ranked) | Orders-by-status table, returns-by-reason table, return units by category |
| **Inventory** | Ending stock and sell-through by month | Days-of-supply seasonality, ending stock value by category | Category summary + a filtered **SKU-level overstock exception list**, ranked by stock value |
| **Customers and Satisfaction** | Distinct Customers by month | Revenue by acquisition channel, positive review % by month, a compact Web Traffic reference panel | Review summary table + review count by rating |
| **Data Quality and Definitions** | — | — | Reconciliation table (model totals vs. independent benchmarks), known data exceptions, KPI glossary |

### Design principles used throughout
- **Flat dimensions aren't charted as if they vary.** Lead time, return rate and shipping fee are statistically flat across region/category/year in this dataset — those comparisons were deliberately left out rather than manufacturing a misleading chart.
- **A table is sized to its content, not stretched to fill the page.** Power BI's table visual doesn't auto-fill unused width; any space freed next to a table holds a paired chart on the same dimension instead of blank canvas.
- **Chart type matches the question**: bar/column for comparisons (no donuts — one category holds ~80% of revenue, which makes small slices unreadable), a combo chart with a secondary axis whenever two series are on different scales.
- **Time trends use a continuous monthly axis** (`'Date'[Month Start]`), so all 10 years fit without a scrollbar and the chart still has a shape when a single year is selected. No chart uses Year as its axis.
- **Titles name what is shown, not what it means.** Interpretation belongs to the reader; a title carries at most a definition or a target (e.g. "target ≥ 95%").
- **Volatile monthly ratios get a rolling view on the Overview, and an explanation on the detail page.** Gross margin swings below zero in promo-heavy months (correlation with promo revenue share: −0.83), so the Overview shows a 12-month rolling margin, and the Sales page pairs monthly margin with promo revenue share so the dips explain themselves.
- **Targets are drawn, not just stated.** A constant measure per target (`SLA Target`, `Return Rate Target`, `Overstock Rate Target`) plots as a second series on the same axis, so the gap to target is visible.
- **One colour, one meaning, on every page.** Navy = the main actual value; teal = volume (orders, units); amber = promotions; green / red = good / bad only (within vs above target, positive vs negative margin); grey dashed = targets and reference lines. Review ratings 1→5 run red → green.
- **Ranked bars are sorted largest first.** Every horizontal bar chart sorts descending by its measure; categories with a natural order (rating 1–5, calendar month) keep that order.

---

## 6. Validation (reconciliation)

The model is checked against two independent, pre-aggregated source files (`sales.csv`, `payments.csv`) that were never used to build any measure — every figure below is recomputed from `order_items`/`orders` and must match:

| Check | Expected |
|---|---|
| Gross Sales (all statuses) = `sales.csv` Revenue | 16.43B |
| COGS (all statuses) = `sales.csv` COGS | 14.16B |
| Net Revenue (all statuses) = `payments.csv` total | 15.68B |
| Weighted List Margin = Non-promo Gross Margin % | Exact match, every year |

These four checks are reproduced live on the **Data Quality** page.

---

## How this was built

The semantic model (TMDL) and report pages (PBIR JSON) were generated by small Python scripts rather than built by hand in Power BI Desktop — every table, relationship, measure, and visual is plain, reviewable text, and the whole report can be regenerated from the data and the agreed KPI definitions above. Power BI's own loader (relationship cardinality checks, ambiguous-path detection) acted as the validator each time the project was opened.

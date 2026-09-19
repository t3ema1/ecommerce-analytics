# Power BI Model Documentation

Data model and measure reference for `ecommerce_analytics.pbix`.

The report reads **only** from the gold-layer CSV exports in `data/bi/`. It does not touch the
110-million-row event tables. What each aggregate contains, and why, is recorded in the decision
log under "BI model design".

---

## Contents

- [Design principles](#design-principles)
- [Tables](#tables)
- [Relationships](#relationships)
- [Measures](#measures)
- [Report pages](#report-pages)
- [Known limitations](#known-limitations)

---

## Design principles

**Counts are stored, not rates.** `agg_funnel` holds `viewed`, `carted` and
`carted_and_purchased` as counts. Rates are computed as measures. This is deliberate:
percentages cannot be summed or averaged across a hierarchy, but counts can, so Power BI
recalculates the ratio from its components at whatever level is being displayed. It is what
makes conversion rates correct at every level of the category drill-down by construction rather
than by careful measure-writing.

**Non-additive measures are kept out of tables grained finer than they support.** Sessions can
touch several categories, so session counts are not additive across them. They therefore live in
a day-only table (`agg_daily_sessions`) rather than in `agg_daily_category`. The cost is that
conversion rate cannot be filtered by category on the overview page; the benefit is that it
cannot silently produce a total exceeding the true session count.

**Dimensions carry only the keys they need.** `agg_product_totals` originally carried
`category_id` alongside `product_id`, creating two paths to `dim_category` and forcing Power BI
to deactivate one. The redundant column was removed at source; category is reached through
`dim_product`.

**Single-direction relationships throughout.** No bidirectional cross-filtering anywhere in the
model.

---

## Tables

### Fact and aggregate tables

| Table | One row represents | Rows | Notes |
|---|---|---|---|
| `agg_daily_sessions` | One day | 61 | Session counts and revenue. Day-only grain because sessions are not additive across categories. |
| `agg_daily_category` | One day × one category | 24,360 | Revenue and item counts. Covers 60 days, not 61 — 15 November has no purchase events (see limitations). |
| `agg_funnel` | One day × category × brand | 445,447 | View, cart and purchase counts. Source table was 68,023,553 rows; a 153x reduction with full drill-down retained. |
| `agg_product_totals` | One product | 68,079 | Whole-period totals plus ABC classification. Only products that sold appear; 138,797 of the 206,876-product catalogue never sold. |
| `agg_user_weekly_orders` | One customer × one week they ordered | 979,709 | Raw material for the cohort grid, deliberately not pre-aggregated so the retention logic is built in DAX. |

### Dimension tables

| Table | One row represents | Rows | Notes |
|---|---|---|---|
| `dim_user` | One purchasing customer | 697,470 | Filtered to purchasers. The full user base is 5,316,649; 87% never purchase and no report page concerns them. Four full-precision timestamp columns were dropped — see optimisations. |
| `dim_product` | One product | 206,876 | Filtered to the current price version. The source `dim_product` is a type-2 slowly changing dimension with 772,840 rows covering price history; no BI page needs that history, and loading it would fan out on join. |
| `dim_category` | One category | 691 | Keyed on `category_id`, not `category_code` — the same readable code maps to several distinct ids (`apparel.shoes` maps to 27). Hierarchy split into four level columns. |
| `dim_date` | One calendar day | 61 | Marked as the official date table. Includes a promotional-period flag for 14–17 November. |
| `dim_time` | One hour of the day | 24 | Built for hour-of-day analysis. Currently unused by any page; retained for future use. |

### Disconnected tables

| Table | Rows | Purpose |
|---|---|---|
| `dim_period_offset` | 9 | Values 0–8. Supplies the column headers for the cohort retention matrix. Has no relationship to anything, and cannot: "weeks since first purchase" is relative to a cohort, not a property of any row. |
| `Top N` | 10 | What-if parameter, values 5–50 in steps of 5. Drives how many products the top-N table displays. |
| `_Measures` | 0 | Empty table holding all measures. Organisational only. |

---

## Relationships

All single-direction, from dimension to fact.

| From | To | On | Cardinality |
|---|---|---|---|
| `dim_date` | `agg_daily_sessions` | `date_key` | 1 → * |
| `dim_date` | `agg_daily_category` | `date_key` | 1 → * |
| `dim_date` | `agg_funnel` | `date_key` | 1 → * |
| `dim_category` | `agg_daily_category` | `category_id` | 1 → * |
| `dim_category` | `agg_funnel` | `category_id` | 1 → * |
| `dim_product` | `agg_product_totals` | `product_id` | 1 → * |
| `dim_user` | `agg_user_weekly_orders` | `user_id` | 1 → * |

`dim_period_offset`, `Top N` and `dim_time` have no relationships. The first two are disconnected
by design; `dim_time` is simply unused.

---

## Measures

All measures live in `_Measures`. Ratios use `DIVIDE()` rather than the division operator, so a
zero denominator returns blank instead of breaking the visual.

### Revenue and orders

| Measure | Definition | Notes |
|---|---|---|
| `Revenue` | `SUM(agg_daily_category[revenue])` | Total revenue. Responds to date and category filters. |
| `Items Sold` | `SUM(agg_daily_category[items])` | Item count **by date and category**. See the note below on the two item measures. |
| `Revenue 7d Avg` | `AVERAGEX(DATESINPERIOD(dim_date[date_key], MAX(dim_date[date_key]), -7, DAY), [Revenue])` | Seven-**day** rolling average, not seven rows. This distinction matters here: 15 November has no purchase rows, so a row-based window would silently span eight calendar days. |

### Sessions and conversion

| Measure | Definition | Notes |
|---|---|---|
| `Sessions` | `SUM(agg_daily_sessions[sessions])` | Total visits. |
| `Purchasing Sessions` | `SUM(agg_daily_sessions[purchasing_sessions])` | Visits ending in a purchase. |
| `Conversion Rate` | `DIVIDE([Purchasing Sessions], [Sessions])` | 6.88% overall. Not filterable by category — see design principles. |

### Funnel

| Measure | Definition | Notes |
|---|---|---|
| `Viewed` | `SUM(agg_funnel[viewed])` | Product-sessions where the product was viewed. |
| `Carted` | `SUM(agg_funnel[carted])` | Product-sessions where the product was carted. |
| `Carted And Purchased` | `SUM(agg_funnel[carted_and_purchased])` | Carted **and** purchased in the same session. |
| `View to Cart Rate` | `DIVIDE([Carted], [Viewed])` | 3.73% overall. |
| `Cart to Purchase Rate` | `DIVIDE([Carted And Purchased], [Carted])` | 39.30% overall. |
| `Cart Abandonment Rate` | `1 - DIVIDE([Carted And Purchased], [Carted])` | 60.70% overall. The complement of the above. |

**Why `Carted And Purchased` rather than a plain purchase count.** A third of all purchases
(507,589, or 33.8%) have no cart event in the same session. A cart-to-purchase rate must count
only journeys that actually passed through the cart. Using a plain purchase count gives 59.33%
— a figure that nearly inverts the finding, making it look as though most carts convert when in
fact most are abandoned.

### Products

| Measure | Definition | Notes |
|---|---|---|
| `Product Revenue` | `SUM(agg_product_totals[revenue])` | Revenue by product. |
| `Product Items Sold` | `SUM(agg_product_totals[items_sold])` | Item count **by product**. |
| `Product Orders` | `SUM(agg_product_totals[orders])` | Order count by product. |
| `Product Count` | `COUNTROWS(agg_product_totals)` | Number of products in the current filter context. |
| `Product Rank` | `RANKX(ALL(dim_product), [Product Revenue], , DESC, Dense)` | Revenue rank. Ties share a rank, with the next rank following immediately. |
| `Is In Top N` | `IF([Product Rank] <= [Top N Value], 1, 0)` | Filter flag driven by the what-if parameter. |

**`Items Sold` versus `Product Items Sold`.** Both return 1,503,806 at the grand total, but they
sum different tables and therefore respond to different filters. `Items Sold` comes from
`agg_daily_category` and responds to date and category; `Product Items Sold` comes from
`agg_product_totals` and responds to product. Using the wrong one in a product-grained visual
returns the grand total on every row, because `agg_daily_category` has no relationship to
`dim_product`. The names were left unchanged to avoid rewiring existing visuals; the distinction
is documented here instead.

**`ALL(dim_product)` versus `ALL(dim_product[product_id])` in `Product Rank`.** The narrower form
clears the filter on `product_id` only. With `brand` also in the visual, ranking then happens
*within brand* — producing 1, 2, 3, 4, 1, 2, 5, 3 rather than a clean sequence. `ALL(dim_product)`
clears filters on the whole table, which is what a global ranking requires. The difference is
invisible until a second column from the same table appears in the visual.

### Retention

| Measure | Definition | Notes |
|---|---|---|
| `Cohort Size` | `CALCULATE(DISTINCTCOUNT(dim_user[user_id]), ALLEXCEPT(dim_user, dim_user[acquisition_cohort_week]))` | Customers acquired in each cohort week. `ALLEXCEPT` keeps the cohort filter and clears everything else, so the denominator stays fixed. |
| `Retained Customers` | See below | Customers from a cohort who ordered in a given week offset. |
| `Retention Rate` | `DIVIDE([Retained Customers], [Cohort Size])` | Column 0 is 100% by definition. |

```dax
Retained Customers = 
VAR SelectedOffset = SELECTEDVALUE(dim_period_offset[period_offset])
VAR CohortWeek = SELECTEDVALUE(dim_user[acquisition_cohort_week])
VAR TargetWeek = CohortWeek + (SelectedOffset * 7)
RETURN
CALCULATE(
    DISTINCTCOUNT(agg_user_weekly_orders[user_id]),
    agg_user_weekly_orders[order_week] = TargetWeek
)
```

**How the disconnected table works here.** `SELECTEDVALUE` reads the current matrix column from
`dim_period_offset` and the current row from `dim_user`, then computes the target calendar week
arithmetically. The cohort filter reaches `agg_user_weekly_orders` through the existing
relationship from `dim_user`. The two tables are never joined — they are bridged by the
calculation.

Placing `dim_period_offset` in a matrix without this measure produces "Can't determine
relationships between the fields". That error is the expected signature of a correctly
disconnected table, not a fault. **Do not accept Power BI's offer to create the relationship** —
doing so would break the design.

---

## Report pages

### Overview
Headline cards (revenue, items, sessions, purchasing sessions, conversion rate), daily revenue
with a seven-day rolling average, revenue by category, and a date-range slicer.

### Funnel
Funnel presented as five cards rather than a funnel visual. At 68.0M → 2.5M → 1.0M, both a funnel
chart and a bar chart rendered the lower two stages illegibly — 96% of the drop happens at the
first step, so the smaller values disappear on any linear scale. Cards show all five figures
legibly. A log scale was considered and rejected as harder to read correctly.

Also on the page: the category table with three-level drill-down, cart abandonment by category,
and the brand breakdown. Both charts carry volume filters to exclude small categories and brands,
where a handful of events produces extreme percentages.

### Products
Top-N product table driven by the what-if slider, and the ABC classification summary.

### Retention
Cohort size table and the retention matrix, with cohort weeks as rows and period offsets as
columns.

---

## Optimisations applied

| Change | Effect |
|---|---|
| Loaded aggregates rather than detail | Largest source table reduced from 68,023,553 rows to 445,447 |
| `dim_user` filtered to purchasers | 5,316,649 → 697,470 rows |
| `dim_product` filtered to current price | 772,840 → 206,876 rows, and removes a join fan-out |
| Dropped four full-precision timestamp columns from `dim_user` | **120.44 MB → 49.27 MB, a 59% reduction from four columns.** High-cardinality datetime columns compress poorly |
| Removed redundant `category_id` from `agg_product_totals` | Eliminated an ambiguous second path and its inactive relationship |
| Auto date/time disabled | Removes the hidden calendar table Power BI generates behind every date column |

Total export: **186.80 MB → 115.62 MB** before Power BI's own compression.

## Performance findings

Two DAX calculations exceeded available resources and were resolved by moving work upstream.

**`RANKX` over the full product catalogue.** Ranking 206,876 products, evaluated once per visible
row, failed outright on an unfiltered table. Resolved by capping the visual at the top N — which
is also what the page should show, so the performance fix and the design fix coincided.

**Cumulative revenue for ABC classification.** A running total requires, for each row, the sum of
revenue across all higher-ranked products. Attempted in DAX with `FILTER` over `ALL(dim_product)`;
exceeded available resources even after narrowing to the 68,079 products that sell. Moved to a
SQL window function in the pipeline, where it is a single pass, and loaded as a stored column.

The trade-off is worth stating: the stored column is fixed, so ABC classes do not recalculate
under filters. The revenue figures alongside them do. For this report that is acceptable —
classification is a property of the whole period, not of a filtered slice — but it would not be
if a user expected the classes to respond to a date range.

---

## Known limitations

**15 November has no purchase data.** All purchase events for that date are missing from the
source — a tracking failure, not a quiet day. Views and carts are present at record levels
(5.7M views, 468,262 carts, the highest cart volume in the dataset). Consequences: the day is
absent from `agg_daily_category`, appears as a gap in the daily revenue line, and affects the
rolling average window. `dim_date` still contains the date, which is why the rolling average is
computed over days rather than rows.

**Conversion rate is not filterable by category.** A deliberate consequence of keeping sessions
out of the category-grained table. Category-level funnel conversion is available on the funnel
page, which is built at the correct grain for it.

**ABC classes are static.** Computed across the whole period in the pipeline. They do not respond
to date filters.

**Category names are missing for about a third of activity.** 409 categories were never assigned
a readable name at source and appear as `unknown_<category_id>`. They are genuine categories with
real volume — the largest has 3M views, more than apparel — but cannot be labelled. Volume
filters on the funnel charts remove most of them, which is convenient but should be understood
as a filter rather than an absence.

**Brand is missing on 14% of events**, grouped as `unknown`. It is the largest single brand
grouping in the model at 10.1M views.

**Two months of data.** The retention grid supports nine weekly cohorts, of which the last two
have almost no observation window. The blank lower-right region is elapsed time that does not
exist, not missing data.

**Q10 not attempted.** Incremental refresh, Performance Analyzer timings and VertiPaq statistics
were scoped out. The optimisations and performance findings above were produced in the course of
building the model rather than through a formal performance pass.

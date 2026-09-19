# E-Commerce Behaviour Analytics

Analysis of **110 million user events** from a large online retailer (October–November 2019),
covering $451.8m of revenue across 5.3 million users.

Built with Python, DuckDB and Power BI. The raw data is a flat event log with no order IDs, no
customer table and no product table — the dimensional model was derived from it.

---

## Headline findings

**The funnel loses almost everything at one step.** 3.73% of viewed products get added to a
cart; 39.30% of those are then bought in the same session. Cart abandonment is 60.70%. The
bottleneck is not checkout — it is getting people to cart anything at all.

**A third of purchases have no cart event.** 507,589 purchases (33.8%) have no matching cart
action in the same session, and 92% of those have none anywhere in the data for that
user-product pair. A strict funnel therefore covers only two-thirds of revenue.

**The catalogue is extraordinarily concentrated.** 727 products — 1.07% of those that sell
anything, and 0.35% of the full catalogue — generate 80% of revenue. Two thirds of products
never sold a single unit in two months.

**Buyers do not browse more widely than non-buyers.** Both look at a median of two products in
one category. Purchasing sessions are 4.6x longer but no broader. Visitors arrive knowing
roughly what they want.

**Repricing is targeted, not universal.** 52.7% of products never changed price. The 183
products repriced 100+ times receive a median of 64,597 views each, against 25 for products
that never move — repricing is applied selectively to high-traffic items.

**The November promotion brought volume but poor-quality customers.** It generated an estimated
+$53.7m in incremental revenue over four days and 83,047 additional first-time buyers — but
that cohort returned at 7.2% the following week, against 11–20% for every other cohort.

**Revenue is concentrated on the customer side too.** 12% of customers produce 42% of revenue.

---

## Things the data hid

Three findings that only appeared because results were checked rather than trusted:

**The spike is not Black Friday.** The dataset is widely described as containing a Black Friday
event. Black Friday 2019 fell on 29 November and shows entirely normal activity here. The real
spike is **14–17 November**, found by measuring daily view counts rather than assuming the date.

**Purchase tracking failed for a full day.** 15 November has 5.7m views and 468,262 cart events
— the highest cart volume in the dataset — and **zero recorded purchases**. An earlier
completeness check passed it, because that check tested total daily volume, and the day had 6.2m
total events. The gap only appears when completeness is checked per event type.

**Average order value rose 14.3% during a discount promotion.** Decomposed: baskets grew 8.8%
larger and the average item was 5.3% more expensive — a mix shift toward premium goods rather
than the discount effect one would expect.

---

## Architecture

```
Raw CSV (15 GB)
   ↓  typed, partitioned, ~4x compressed
Parquet (bronze)
   ↓  deduplicated, categories resolved, sessions and orders reconstructed
Silver
   ↓  dimensional model
Gold
   ↓  purpose-built aggregates
Power BI
```

**Fact tables**

| Table | One row is | Rows |
|---|---|---|
| `fact_order_item` | one product in one order | 1,503,806 |
| `fact_session` | one visit | 18,776,366 |
| `fact_session_product` | one product in one visit | 68,023,553 |

**Dimensions**

| Table | One row is | Rows |
|---|---|---|
| `dim_user` | one person | 5,316,649 |
| `dim_product` | one product during one price period | 772,840 |
| `dim_category` | one category | 691 |
| `dim_date` | one day | 61 |

`dim_product` is a type-2 slowly changing dimension: prices change over time, so each product
has one row per price period with a validity window, joined to purchases on both product and
timestamp.

Three fact tables, not five, because this source contains only two genuine business processes —
a visit and a purchase. There is no payment, delivery, review or returns data, so there are no
fact tables for them.

---

## Technical approach

**Scale.** The November file alone is 9 GB and will not load into memory conventionally. Data is
converted to Parquet (3.7x smaller) and queried with DuckDB, which runs SQL directly over the
files. pandas is used only on aggregated results.

**Sessions were rebuilt, not inherited.** The source supplies a `user_session` column with
undocumented rules. An independent 30-minute inactivity rule was implemented with window
functions and compared against it: the rebuilt version produces 18% fewer sessions, because the
source resets after gaps as short as 25 minutes. The rebuilt definition is used because every
boundary in it can be justified from the data.

**Orders were reconstructed.** No order ID exists. Orders are defined as purchases within one
session, with 174 bot-like sessions flagged (one contained 89 purchases of computer monitors
spread evenly over four hours). Repeated purchases of the same product within an order are
treated as duplicate tracking events rather than quantity — a decision worth **10.3% of total
revenue**, measured and disclosed rather than assumed.

**Every transformation is reconciled.** Row counts and revenue totals are checked before and
after each step. This caught three real bugs: a same-second price collision producing
zero-length validity windows in `dim_product`; a join fan-out that inflated view counts by 8.2%
because 62 products carried two brand values; and a drift metric that returned exactly 1.0 by
construction rather than by measurement.

---

## The report

Four pages, built on purpose-shaped aggregates rather than the detail tables. The largest source
table — 68,023,553 rows — is loaded as 445,447 aggregate rows: a 153x reduction with full
category and brand drill-down retained.

### Overview

![Overview page](docs/screenshots/01_overview.png)

Headline metrics, daily revenue with a seven-day rolling average, and revenue by category. The
rolling average is computed over seven **days** rather than seven rows — a distinction that
matters here, because 15 November has no purchase rows and a row-based window would silently
span eight calendar days.

### Funnel

![Funnel page](docs/screenshots/02_funnel.png)

Conversion rates remain correct at every level of the three-level category hierarchy. This works
by construction: the aggregate stores view, cart and purchase *counts*, not rates. Percentages
cannot be summed or averaged across hierarchy levels, but counts can, so the ratio is recomputed
from its components at whatever level is displayed.

The funnel itself is shown as cards rather than a funnel visual. At 68.0M → 2.5M → 1.0M, both a
funnel chart and a bar chart rendered the lower two stages illegibly — 96% of the drop happens
at the first step, so the smaller values vanish on any linear scale.

*The brand table carries a volume filter excluding brands under 100,000 views, so its total
reads 3.75% rather than the unfiltered 3.73%.*

### Products

![Products page](docs/screenshots/03_products.png)

Top-N ranking driven by a what-if parameter, with ABC classification. The cumulative-revenue
calculation behind ABC was attempted in DAX, exceeded available resources at 206,876 products,
and was moved to a SQL window function in the pipeline.

### Retention

![Retention page](docs/screenshots/04_retention.png)

Cohort logic built in DAX rather than pre-computed, using a disconnected period-offset table.
"Weeks since first purchase" is relative to a cohort, not a property of any row, so it cannot be
joined — the two tables are bridged by the measure. The blank lower-right region is elapsed time
that does not exist, not missing data.

Full model reference: [`powerBI/model_documentation.md`](powerBI/model_documentation.md)

---

## Repository contents

```
Notebooks/
  01_explore_data.ipynb     exploration and profiling
  02_data_quality.ipynb     duplicates, price validity, session integrity
  03_modelling.ipynb        sessions, orders, dimensional model
  04_analysis.ipynb         the five analysis questions
docs/
  project_brief.md          the ten questions, and which were revised after checking the data
  question_traceability.md  which table answers which question, and why
  decision_log.md           every decision, why, and what it cost
  screenshots/              report pages
  *.png                     analysis charts
powerBI/
  ecommerce_analytics.pbix
  model_documentation.md    tables, grains, relationships, measures, limitations
```

**`docs/decision_log.md` is the substantive document.** It records every modelling decision with
the alternatives rejected and the measured impact of each, the data-quality findings, and the
full analysis write-ups.

**`docs/question_traceability.md`** records the exercise of mapping each question to the grain it
requires *before* building — which found three gaps in a model that had already been partly
built.

---

## Reproducing this

Data is not included — it is 15 GB and excluded from version control.

1. Download `2019-Oct.csv` and `2019-Nov.csv` from
   [the REES46 dataset on Kaggle](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store)
2. Place both in `data/`
3. `pip install pandas duckdb matplotlib scipy jupyter`
4. Run the notebooks in order

Expect the modelling notebook to take a while — several steps process the full 110 million rows,
and one required splitting into disk-backed stages to stay within memory.

---

## Limitations

- **Two months of data.** Retention analysis supports nine weekly cohorts, the last two of which
  have almost no observation window. Monthly cohorts are impossible.
- **No causal claims.** The promotion's impact is an estimate against a weekday-matched
  baseline, not a controlled comparison. Price and demand cannot be separated, because the
  repricing system evidently reacts to demand.
- **32% of category labels are missing** and unrecoverable — 409 categories were never assigned
  a readable name at source. They are preserved as distinct unlabelled categories rather than
  merged or dropped.
- **Brand is missing on 14% of events**, grouped as `unknown` — the largest single brand grouping
  in the model at 10.1M views.
- **No checkout events**, so checkout abandonment is not measurable — only "carted but not
  bought".
- **Q10 of the brief was not attempted.** Incremental refresh, Performance Analyzer timings and
  VertiPaq statistics were scoped out. The optimisations recorded in the model documentation were
  produced while building rather than through a formal performance pass.

---

*Data credit: [REES46 Marketing Platform](https://rees46.com/), published on Kaggle.*

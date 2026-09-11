# Project Brief

This is a self-directed practice project. The brief below was written before the data was
examined, in the style of an analytics-engineering take-home test, then **revised after checking
every question against what the data could actually support**.

Several questions turned out to rest on assumptions the data contradicted. Those are marked and
the corrections explained. Making those corrections — and being able to show the evidence for
them — is part of the work, not a departure from it.

---

## The data

REES46 multi-category store behaviour events, October and November 2019.
Roughly 110 million rows across two CSV files totalling 15 GB.
Available on [Kaggle](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store).

Nine columns: timestamp, event type, product ID, category ID, category name, brand, price,
user ID, session ID. No order IDs, no product table, no customer table, no calendar.

---

## Engineering requirements

**E1 — Load at scale.** Convert to a columnar format with explicitly declared types. No step in
the pipeline may load a full month into memory. Loading must be safe to re-run. Report the size,
memory and runtime comparison against the naive approach.

**E2 — Push work to the query engine.** Aggregations over roughly a million rows or more run in
SQL against the files, not in pandas. Demonstrate window functions, rolling windows, CTE chains,
`QUALIFY`, conditional aggregation and anti-joins.

**E3 — Keep a data-quality register.** At least twelve checks covering completeness, uniqueness,
validity, consistency and coverage. Record what failed, how much, what was done, and what the
number would have been under the alternative.

**E4 — Reconstruct sessions and orders.** Validate the supplied session ID, then rebuild sessions
independently and quantify the divergence. Define what constitutes an order, decide how to treat
repeated purchases of the same product, and measure what that decision costs in revenue.

**E5 — Build the dimensional model.** State what one row of every table represents. Split the
category hierarchy into levels. Replace nulls in dimensions with explicit unknown members.
Implement product price history as a type-2 slowly changing dimension.

**E6 — Prepare for the BI layer.** Decide what Power BI loads. Neither 110 million rows nor a
single flat summary is an acceptable answer.

---

## Analysis questions — Python

### Q1. Conversion funnel
Measure view → cart → purchase conversion overall, by category and by brand. Compute cart
abandonment. Test whether two comparable categories differ, reporting effect size and confidence
interval alongside significance.

> **Revised.** The original asked for cart abandonment without qualification. This dataset has no
> checkout-stage events — only view, cart and purchase — so *checkout* abandonment is not
> measurable. Only "added to cart, not bought" can be reported. The question now requires stating
> which window is used and why.

### Q2. Session behaviour
Characterise sessions and identify which characteristics accompany a purchase. Report percentiles
rather than means on skewed distributions. State the timezone assumption and show its effect.

### Q3. Price movement and demand
Establish how prices change, build the price history, and examine whether demand shifts around
price changes. Explain why this is not a measurement of price sensitivity.

> **Revised.** The original assumed prices change occasionally. Investigation found some products
> repriced dozens of times a day — algorithmic repricing — while over half never change at all.
> The question now asks about the *pattern* of movement, and treats the difficulty of isolating a
> clean before-and-after window as part of the analysis.

### Q4. Customer retention
Group customers by acquisition period and measure return rates. Handle the censoring problem
where recent cohorts have had less time to return. Show cohort sizes alongside rates.

> **Revised.** Monthly cohorts are impossible with two months of data. Specified as weekly, giving
> nine cohorts of which the last two carry almost no observation window.

### Q5. Segmentation and promotional impact
Score customers on recency, frequency and monetary value; name and size the segments; recommend
actions. Then identify the promotional period from the data, build a counterfactual baseline,
estimate incremental impact, and test whether demand was pulled forward.

> **Revised.** The original called this "the Black Friday shock". Black Friday 2019 fell on 29
> November and shows entirely normal activity in this data. The real spike is 14–17 November,
> found by measuring daily counts. The question now requires identifying the period from the data
> rather than assuming a date.

---

## Analysis questions — Power BI

### Q6. Overview page
Headline metrics with period comparison and a rolling average, built as measures using date
intelligence rather than pre-computed columns.

> **Revised.** Month-on-month comparison is near-meaningless here: two months exist, and one
> contains the promotion, so the comparison mostly measures the promotion. Week-on-week is used
> instead, or the confounding is stated explicitly.

### Q7. Funnel page
The Q1 funnel, interactive, with category drill-down and brand breakdown. Conversion rates must
remain correct at every level of the hierarchy and under any filter combination — percentages
cannot be aggregated, they must be recomputed from their components at each level.

### Q8. Retention grid
The Q4 cohort grid rebuilt in DAX rather than pre-computed, using a disconnected period table.
The grid will be small — two months gives eight weekly cohorts — but the technique is the
deliverable and scales to any horizon.

### Q9. Product and brand performance
Dynamic top-N ranking with a user-controlled parameter and an "Others" bucket that reconciles to
the grand total. ABC classification by cumulative revenue share. Price bands via a disconnected
table.

### Q10. Performance and refresh *(optional)*
Incremental refresh configuration, measured before-and-after optimisation using Performance
Analyzer, model size statistics, and written model documentation.

> **Scoped as optional.** The most tool-advanced question in the set, and the one that makes most
> sense after building several report pages. Attempted after Q6–Q9.

---

## How the brief was checked

Before building the model, every question was mapped to the grain it requires and the table that
would serve it. That exercise is recorded in
[`question_traceability.md`](question_traceability.md) and found three gaps in a model that had
already been partly built:

- The funnel questions need one row per product per session. A session-grain table cannot answer
  them — a session touches many categories, and that detail is destroyed on aggregation.
- Four questions need per-customer attributes. No customer dimension existed.
- The session table had no revenue column.

All three were corrected before the analysis began. The general lesson: work backwards from the
questions to the required grain, then to the tables. Building tables that seem sensible and hoping
they fit is how the gaps arose in the first place.

---

## Standards applied throughout

- Every number quoted traces to a result that can be pointed at.
- Row counts reconciled before and after every filter and join.
- Where a choice was not obvious, the alternative and its measured cost are recorded.
- No causal language where only association is supported.
- Where the data contradicted the brief, the brief was changed and the evidence recorded.

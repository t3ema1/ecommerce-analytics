# Question Traceability — Requirements to Model

This document maps every analysis question to the data grain it requires, the table that serves
it, and any constraint the dataset imposes. It was produced *after* the dimensional model was
partly built, in order to check whether the model actually supports the questions — an exercise
that surfaced three real gaps that would otherwise have been discovered much later.

**Method:** for each question, ask two things in order — *what grain does one row need to
represent?* and *does an existing table provide that grain?* Working in this direction
(requirements → grain → table) is what surfaced the gaps. Working the other way (build sensible
tables, hope they fit) is what created them.

---

## Summary of findings

**Model gaps identified:**

| # | Gap | Needed by | Status |
|---|-----|-----------|--------|
| 1 | `fact_session_product` | Q1, Q7 | Must build — highest priority |
| 2 | `dim_user` | Q4, Q5, Q6, Q8 | Must build |
| 3 | Revenue column on `fact_session` | Q6 | Must add |
| 4 | `dim_time` | Q2 (Power BI only) | Must build before BI phase |
| 5 | `dim_period_offset` (disconnected) | Q8 | Trivial, build during BI phase |

**Tables validated as correctly designed:** `fact_session` (Q2), `dim_product` (Q3),
`dim_date`, `dim_category`.

**Question corrections required:**

| Question | Correction | Reason |
|----------|-----------|--------|
| Q1 | Clarify "cart abandonment" ≠ "checkout abandonment" | No checkout events exist in this dataset |
| Q3 | Reframe for algorithmic repricing | Prices change 3–4×/day, not occasionally |
| Q4 | Specify weekly cohorts | Two months cannot support monthly cohorts |
| Q5 | Remove "Black Friday" framing | Spike is 14–17 Nov; Black Friday 2019 was 29 Nov |
| Q6 | Reframe MoM, or require confounding caveat | Only 2 months, and one contains the promo spike |
| Q8 | Note horizon limitation explicitly | ~8 weekly cohorts produces a visually thin triangle |
| Q10 | Treat as stretch goal | Most tool-advanced question; do after Q6–Q9 |

---

## Python questions

### Q1 — Funnel and conversion

**Grain required:** one row per (session, product)

**Table:** `fact_session_product` — **missing**

**Why this grain:** session grain destroys the detail the question needs. A session that carts
three electronics items and two apparel items, purchasing one of each, collapses to
`cart_count = 5, purchase_count = 2` — from which category-level conversion cannot be recovered.
Product grain preserves everything and rolls up to category, brand, or hierarchy level as needed.

**Principle established:** build fact tables at the finest grain any question requires, then
aggregate upward in queries. Rolling product → category → level 1 is always possible; the
reverse never is.

**Data constraint:** the dataset contains only `view`, `cart`, and `purchase` events. There are
no checkout-stage events. Therefore *checkout* abandonment is not measurable; only *cart*
abandonment ("carted but not purchased") is. These are different metrics and the write-up must
not conflate them.

**Decision taken:** same-session cart abandonment as the primary metric, with "abandoned ever"
reported as a secondary figure.

**Reasoning:** same-session abandonment has no censoring problem — every session has a complete,
observable outcome by the time it ends, including sessions starting on the final day of data.
It also matches the funnel framing, which describes a single continuous flow. The secondary
"ever" figure is reported because same-session alone overstates abandonment; quantifying the gap
between the two is itself a finding.

**Rejected alternative:** a fixed-window definition (e.g. abandoned if not purchased within
7 days) is methodologically defensible but requires excluding the final N days of data,
introducing a second avoidable censoring problem alongside the unavoidable one in Q4.

---

### Q2 — Session behaviour and engagement

**Grain required:** one row per session

**Table:** `fact_session` — already built ✅

Every metric the question asks for (duration, event counts, distinct products and categories,
purchase flag) is an existing column. This validates the table's design even though it turned
out not to serve Q1.

**Timezone note:** the question requires stating the timezone assumption and demonstrating how
hour-of-day patterns shift under a different one. This is an analysis step, not a modelling one
— hour-of-day is derivable from `session_start` at query time, and alternative offsets can be
tested without storing anything extra.

**Power BI gap:** hour-of-day analysis in the BI layer will need `dim_time` (one row per hour,
with daypart groupings). Not required for the Python work.

---

### Q3 — Price dynamics and the SCD

This question has two halves with different requirements.

**Half 1 — price change history**

Grain: one row per (product, price period). Served by `dim_product` ✅ — already built as a
Type 2 SCD. This half of the question is essentially already complete.

**Half 2 — demand around price changes**

Grain: time-bucketed product activity — one row per (product, hour or day) with view, cart and
purchase counts.

**Table: none — computed at query time** from `events_with_sessions`.

**Reasoning:** `fact_order_item` cannot serve this, because it contains only purchases while the
question asks about view volume and cart rate. Session×product grain would technically work but
aligns poorly to a timeline: price changes occur 3–4× daily, sometimes hours apart, and a session
straddling a change point cannot be cleanly assigned to "before" or "after". Time-bucketed grain
slices cleanly at the change timestamp.

**Principle established:** a grain the *analysis* requires is not automatically a table the
*model* requires. Aggregate tables earn their place through repeated use — particularly by a BI
tool — not because one notebook needs them once. Building `agg_product_hourly` for a single
analysis would be over-engineering.

**Question reframing required:** the question as written assumes occasional, discrete price
changes. Investigation of product 1004856 found prices changing 3–4 times daily within a ~$3
band, consistent with algorithmic/dynamic repricing. The "why this is not an elasticity estimate"
requirement becomes *more* important under this pattern, not less — the confounding is heavier.

---

### Q4 — Cohort retention and repeat purchase

**Grain required:** one row per user (for cohort assignment)

**Table:** `dim_user` — **missing**. Required attributes: `first_activity_date`,
`first_purchase_date`, `acquisition_cohort_week`, plus `total_orders`, `total_revenue` and
`last_activity_date` for Q5.

The retention triangle itself is computed at query time by joining `dim_user` (for cohort
membership) to `purchases_final` or `fact_session` (for subsequent activity). No dedicated fact
table is needed.

**Viability — confirmed empirically:**

| Orders per user | Users |
|-----------------|-------|
| 1 | 464,424 |
| 2 | 125,652 |
| 3 | 46,840 |
| 4 | 21,638 |
| 5 | 11,628 |

Roughly 35% of purchasing users place more than one order, with a tail extending well past ten.
Repeat-purchase behaviour is genuinely present, so the retention triangle will contain real
content rather than being near-empty. (This is where several comparable public e-commerce
datasets fail — one-and-done customers dominate.)

**Constraint:** two months of data supports approximately 8 weekly cohorts. Monthly cohorts
(2 of them) are not viable. Right-censoring is severe rather than incidental: the most recent
cohorts have had almost no observation window, and the censored region must be visually marked
rather than read as poor retention.

---

### Q5 — RFM segmentation and promotional impact

**Part 1 — RFM**

Grain: one row per user. Table: `dim_user` — **missing** (same table as Q4, which strengthens
the case for building it properly rather than computing these ad hoc).

**Decision taken:** manual frequency bins rather than quantile binning.

**Reasoning:** frequency is highly discrete. 464,424 users share the single value "1 order" —
roughly 68% of the purchasing base. Splitting that into five equal-sized quantile bins is
impossible: the first three bin boundaries all land inside the block of 1s. `pd.qcut` will either
raise a "bin edges must be unique" error, or — with `duplicates='drop'` — silently return three
bins while the analyst continues assuming five. Every downstream segment definition expecting
F=4 or F=5 would then be quietly broken.

Manual bins chosen from the observed distribution (e.g. 1 → 1, 2 → 2, 3–4 → 3, 5–9 → 4, 10+ → 5)
produce unequal group sizes but each bin corresponds to a distinguishable behaviour pattern, and
the choice is explainable.

**Part 2 — promotional impact**

Grain: daily revenue, orders and new users. Tables: `fact_order_item` (revenue, orders) plus
`dim_user` (new users, via `first_purchase_date`), aggregated at query time.

**Question correction — the framing is factually wrong.** The document describes this as "the
Black Friday shock" and states that Tier A "contains the Black Friday period." Black Friday 2019
fell on **29 November**. Measured daily view counts show 29 November at baseline (~1.6M). The
actual spike is **14–17 November**:

| Date | Views |
|------|-------|
| 13 Nov | 1,924,916 |
| 14 Nov | 2,877,071 |
| 15 Nov | 5,737,078 |
| 16 Nov | 6,027,799 |
| 17 Nov | 5,783,122 |
| 18 Nov | 1,909,738 |

`dim_date.is_promo_period` was corrected to 14–17 November on this evidence. The *analysis*
survives intact because the question already required identifying the period from the data rather
than assuming dates — but the surrounding framing must be reworded to "promotional period."

---

## Power BI questions

### Q6 — Executive overview

**Tables:** `fact_order_item` (revenue, orders, AOV), `dim_user` (purchasing users, and as a
sliceable dimension for segment/cohort), `fact_session` (session conversion via `had_purchase`),
`dim_date`.

**Gap:** `fact_session` currently has event counts but no revenue column. Session-level value
metrics require adding one.

**Note on conversion rate:** the instinct to reach for the raw event table is correct in
substance but wrong in layer — Power BI should not query 110M silver-layer rows. The gold
equivalent is `fact_session` (session-level conversion) or `fact_session_product` (product-level
funnel conversion), depending on the definition chosen.

**Question correction:** month-over-month comparison is close to meaningless here. Two months
yields exactly one comparison, and it is confounded — November contains the promotional spike and
October does not, so "revenue grew X% MoM" measures the promotion, not an underlying trend.
Either reframe to week-over-week (8 weeks, more comparisons, though still promo-affected), or
retain MoM with a mandatory stated caveat. The caveat is the analytical content.

---

### Q7 — Funnel and conversion in Power BI

**Tables:** `fact_session_product` (**missing**) + `dim_category` (hierarchy drill-down) +
`dim_product` (brand).

This is the second question depending on `fact_session_product`, making it the highest-priority
gap to close.

**Data constraint:** `brand` is null on 13.95% of rows. The model needs an explicit "Unknown"
brand member rather than blanks, or Power BI will handle nulls inconsistently in hierarchies —
the same principle applied to `unknown_<category_id>` for categories.

**Core difficulty to preserve:** conversion measures must remain correct at every hierarchy level
under any slicer combination. Ratios are non-additive; they must be recomputed from their
components at each level, never aggregated. This is the most commonly failed DAX task in the set
and should not be simplified away.

---

### Q8 — Cohort retention matrix in DAX

**Tables:** `dim_user` (with `acquisition_cohort_week`) + `dim_period_offset` — a **disconnected
table** containing only the values 0–7.

**Why disconnected:** "weeks since acquisition" is not a property of any row in the data. It is a
relative concept that only exists once a cohort is fixed, so it cannot be joined via a normal
relationship. The table sits unrelated in the model and is referenced inside DAX measures to
drive the matrix columns.

**Viability:** technically buildable, visually thin. An 8×8 triangle is half-empty by definition
— roughly 36 filled cells of 64 — where the question clearly imagined a 12- or 24-month grid.

**Decision:** build it anyway. The deliverable is the DAX capability — disconnected tables,
filter-context manipulation, dynamic measures — which is demonstrated regardless of triangle size.
The write-up should state plainly that the matrix is thin because the dataset spans two months,
and that the technique scales to any horizon.

**Worth noting in the write-up:** pre-computing the triangle in Python is the pragmatic
production choice. Building it in DAX proves understanding of filter context. Doing it in DAX and
then explaining why production would pre-aggregate demonstrates both.

---

### Q9 — Product, brand and category performance

**Tables:** `fact_order_item` (revenue — the measure) + `dim_product` (brand, price attributes,
price bands) + `dim_category` (hierarchy).

No new tables required, no question changes. This one works as written.

**Distinction worth recording:** dimensions describe *things*; facts measure *what happened*.
Any "how much / how many" question needs at least one fact table with dimensions attached to
slice it. Answering a measurement question with dimensions alone means something is missing.

---

### Q10 — Performance, refresh and governance

**Tables:** none — this question is about operational quality, not analysis, so there is no grain
to resolve.

**Scope decision: treat as a stretch goal**, attempted after Q6–Q9. It is the most tool-advanced
question in the set and the most dependent on hands-on Power BI familiarity. It will make
substantially more sense after building several report pages and experiencing a model run slowly.

**Feasibility note:** incremental refresh requires a Power BI capacity to actually execute. The
question anticipates this — the configuration and the written reasoning are the deliverable even
where execution is not possible.

---

## What this exercise demonstrated

Every gap found here surfaced from asking *what grain?* before *what table?*

The original model was reasoned from business processes — "a visit" and "a transaction" — which
is a legitimate heuristic but not a substitute for working backwards from the questions. That
shortcut produced a `fact_session` table that serves Q2 well and Q1 not at all, and omitted
`dim_user` entirely despite four questions depending on it.

The corrections to the questions themselves were all evidence-based rather than preference-based:
the promotional period was measured, not assumed; repeat-purchase viability was checked before
retention work was committed to; the brand null rate was quantified before Q9 was accepted as
written. Where the data contradicted the question, the question was changed — but the analytical
difficulty was deliberately preserved rather than sanded off.

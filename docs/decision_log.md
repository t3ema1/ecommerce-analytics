# Decision Log — E-Commerce Behaviour Analytics

Every non-obvious decision made during this project: what was decided, why, what alternatives
were considered, and what evidence backed the choice.

**Dataset:** REES46 multi-category store behaviour events, October–November 2019.
109,820,004 events after cleaning. Total revenue $451,807,232.56.

**Structure:** Part 1 records decisions made while building the data. Part 2 records data-quality
findings. Part 3 records analysis findings, organised by question.

---

# Part 1 — Modelling decisions

## Decision 1: Handling missing `category_code`

**Situation:** 32.44% of rows (21,898,171 of 67,501,979 in November) have a missing
`category_code`, even though `category_id` is never missing.

**Investigation:**
- `category_id` is 100% populated — 0 missing across all rows.
- Each `category_id` maps to exactly one `category_code`, no exceptions. Checked via
  `GROUP BY category_id HAVING COUNT(DISTINCT category_code) > 1`, which returned 0 rows.
- Tested whether missing codes could be recovered by looking up `category_id` against rows
  that do have a code. Result: 0 recoverable.
- Root cause: the missing codes are not random gaps. 409 distinct `category_id`s never have a
  code anywhere in the dataset; 275 always do. Zero overlap between the two groups.

**Conclusion:** missing `category_code` is a structural property of certain categories, not a
per-row data entry error. Those 409 categories were never assigned a readable name at source.

**Options considered:**
1. Drop the rows — rejected, would lose 32% of all data.
2. Recover via `category_id` lookup — rejected, proven impossible.
3. Fill with a single generic "Unknown" — rejected as too coarse; would merge 409 genuinely
   different categories into one indistinguishable bucket.
4. **Chosen:** fill using the `category_id` itself as a distinguishing label
   (`unknown_<category_id>`), preserving the fact that these are 409 separate real categories
   that simply lack a human-readable name.

**Verification:** row count unchanged at 109,950,743. `category_code` nulls reduced from
35,413,780 (32.2% across both months) to 0. Sample rows confirmed correct formatting.

**Impact:** any category-level reporting must treat the "unknown" group as a real and sizeable
share of activity, not filter it out.

---

## Decision 2: Combining October and November into a single working table

**Situation:** two monthly Parquet files (Oct 42,448,764 rows; Nov 67,501,979 rows) needed the
same cleaning logic. Writing that logic twice risked duplication and drift between versions.

**Decision:** combine both months into one file using `UNION ALL`, adding a `source_file`
column to each half beforehand for traceability back to the original monthly file.

**Verification:** combined count 109,950,743 reconciles exactly against 42,448,764 + 67,501,979.
No rows lost or duplicated.

**Impact:** all cleaning and modelling from this point operates on a single source of truth.

---

## Decision 3: Removing exact duplicate rows

**Situation:** investigated rows identical across every column.

**Investigation:**
- 75,652 distinct duplicate groups, containing 206,391 rows in total.
- Broken down by event type: cart 71,961 rows (95.1%), view 3,606 (4.8%), purchase 85 (0.1%).
- Sampled the largest groups: identical cart events — same user, product, session and timestamp
  to the second — repeating up to 78 times. A person does not click "add to cart" 78 times in
  one second. Consistent with a tracking artefact, such as a cart-state event re-firing on UI
  re-render.

**Decision:** remove exact duplicates, keeping the first occurrence of each group, via
`ROW_NUMBER() OVER (PARTITION BY <all columns>)`.

**Verification:**
- Before: 109,950,743. After: 109,820,004. Removed: 130,739.
- Matches the expected calculation exactly: 206,391 rows in duplicate groups minus 75,652
  groups kept = 130,739 removed.
- Confirmed 0 remaining exact duplicates.

**Process note:** an earlier verification used a flawed query that conflated "number of
duplicate groups" with "number of duplicate rows", making the result look wrong. Caught by
cross-checking the actual row-count difference against a proper group-size calculation before
accepting the figure.

**Impact:** 130,739 rows removed (0.12% of data), overwhelmingly cart events. Prevents inflated
cart counts in any downstream funnel work.

---

## Decision 4: Session definition — rebuilt session ID versus the supplied `user_session`

**Situation:** the dataset includes a `user_session` column, but its underlying rules are
undocumented and unverifiable. A reliable, explainable definition of "visit" was needed for
funnels, engagement metrics and cohort work.

**Investigation:**
- Validated the supplied column first (see the `user_session` integrity finding in Part 2):
  12 missing values, and 939 sessions shared across multiple users, traced to an apparent
  session-ID collision. Both negligible.
- Built an independent definition using a 30-minute inactivity rule: `LAG()` to measure the gap
  between a user's consecutive events, a flag where the gap exceeds 30 minutes, then
  `SUM() OVER (...)` as a running total to number the sessions.
- Compared totals: the rebuilt logic produced 18,776,366 sessions against 23,016,650 supplied —
  about 18% fewer.
- Investigated the divergence directly. Found the supplied `user_session` changing after gaps as
  short as 25 minutes, below our 30-minute threshold. Example: user 512365995 on 1 October, a
  gap of 25 minutes between 14:01:06 and 14:26:12, where the supplied session ID changed but our
  rule would not have split. The two methods agree on large gaps (a 60-minute gap split under
  both) and diverge on borderline ones.

**Conclusion:** the source system uses a shorter effective threshold, or resets sessions on
non-time-based triggers such as logins or page reloads. Its exact rule cannot be reproduced or
verified from this data.

**Decision:** use the rebuilt session ID (`rebuilt_session_id`, formatted `<user_id>_<n>`) as the
primary definition for all downstream analysis. Retain the original `user_session` column for
reference and traceability, but do not base session-level metrics on it.

**Reasoning:** the rebuilt definition follows a well-known, explainable convention, and every
session boundary can be justified from the data itself. The supplied column may carry genuine
extra signal, but that cannot be confirmed, and unexplainable boundaries would undermine trust
in every session-based metric built on top of them.

**Impact:** expect roughly 18% fewer, and correspondingly longer, sessions than the supplied
column would give.

---

## Decision 5: Order reconstruction and non-standard order flagging

**Situation:** no order ID exists in the raw data. Purchase events had to be grouped into orders
using a defensible rule.

**Investigation:**
- Tested the hypothesis "all purchases within one session = one order".
- Found a session with 89 purchase events, all computer monitors, spread over about four hours
  at a steady rate of one purchase every 1–4 minutes, with the same product appearing four
  times. This is inconsistent with human checkout behaviour and more consistent with automated
  activity or a test account.
- Checked the overall distribution: 97.8% of sessions containing any purchase have three or
  fewer purchase events — a normal basket. Only 174 sessions (0.013%) have 20 or more,
  accounting for 4,792 rows (0.29% of purchase events).

**Decision:**
1. Define `order_id` = `rebuilt_session_id` — all purchases within one session form one order.
2. Flag orders with 20 or more purchase events as `is_likely_non_standard_order`. These are
   retained rather than deleted, but excluded from standard order-size and order-value reporting.

The 20 threshold was chosen from the observed distribution, where the tail thins sharply after
10–20, rather than picked as a round number.

**Verification:** row counts reconcile exactly against source (1,659,703 purchase events in
both). Flagged counts cross-checked against the distribution query; an initial mismatch
(144 versus 174 orders) was traced to a boundary inconsistency between `>= 20` and `21+` in two
different queries, and resolved — the 30 sessions with exactly 20 purchases accounted for the
entire gap.

**Impact:** 1,290,934 standard orders; 174 flagged for separate handling.

---

## Decision 6: Repeated purchases of the same product within one order

**Situation:** 152,692 rows (9.2% of standard-order purchase events) represent the same product
purchased more than once within the same order. There is no quantity column, so it is ambiguous
whether these are genuine multiple units or duplicate tracking events.

**Investigation:**
- Sampled the worst cases: products purchased up to 18 times within a single order, at a steady
  rate of roughly one purchase every 1–4 minutes over 20–90 minutes. This matches the mechanical
  repeat pattern already identified in cart events (Decision 3) and in the bot-like session
  (Decision 5).
- Quantified both interpretations:
  - Every repeat counted as genuine quantity: **$503,079,668.26**
  - Only the first occurrence per (order, product) counted: **$451,263,778.53**
  - Difference: **$51,815,889.73, or 10.3% of total revenue.**

**Decision:** treat repeats as duplicate tracking events. Keep only the chronologically first
occurrence per `(order_id, product_id)`, using
`ROW_NUMBER() OVER (PARTITION BY order_id, product_id ORDER BY event_time)`.

**Reasoning:** the steady mechanical timing is inconsistent with human purchasing and mirrors
patterns already identified as tracking artefacts elsewhere in this dataset. The absence of any
quantity column in the source argues against reading repeats as deliberate multi-unit purchases.

**Limitation acknowledged:** this is an assumption, not a certainty. Some share of the 152,692
repeat rows may be genuine — buying two of a cheap consumable is ordinary behaviour. A 9.2%
repeat rate is arguably too high to be purely a glitch. The 10.3% revenue swing is disclosed
openly rather than hidden behind a silent default.

**Verification:** final purchase table 1,503,806 rows, revenue $451,807,232.56. Reconciled: the
1,587-row difference against the standard-orders-only figure was traced exactly to first
occurrences within the 174 non-standard orders.

**Impact:** all revenue and order-value figures are computed on the first-occurrence basis.
Any comparison against a naive full-count approach must account for the 10.3% difference.

---

## Decision 7: `dim_date` promotional period boundaries

**Situation:** a promotional flag was needed in `dim_date` for Q5. The initial boundary was
eyeballed from a chart as 15–19 November.

**Verification:** pulled exact daily view counts for 12–22 November. The true elevated period is
**14–17 November** — views rise from a ~1.9M baseline to 2.9M on the 14th and 5.7–6.0M across
15–17, returning to 1.9M on the 18th. The visual estimate was wrong by one day at each end.

**Decision:** set `is_promo_period = TRUE` for 14–17 November, based on measured counts rather
than visual estimation.

**Note:** this period is **not Black Friday**, which fell on 29 November 2019 and shows only a
modest lift in this data.

---

## Decision 8: `dim_category` primary key

**Situation:** needed to choose between `category_id` (numeric) and `category_code` (the readable
dotted path) as the key.

**Investigation:**
- Decision 1 established that `category_id` → `category_code` is one-to-one.
- The reverse is **not** true. `category_code` → `category_id` is one-to-many: `apparel.shoes`
  alone maps to 27 distinct `category_id`s, and 58 codes map to more than one id.

**Decision:** `category_id` is the primary key. `category_code` and its split levels are
descriptive attributes. Confirmed 691 distinct `category_id`s, all unique.

**Impact:** every join from a fact table to `dim_category` must use `category_id`. Joining on the
readable code would silently merge genuinely distinct categories.

---

## Decision 9: `dim_product` price-history granularity

**Situation:** product prices change over time, so a single price column would discard the
history. A slowly-changing dimension (type 2) was needed: one row per product per price period.

**Investigation:**
- Sampled product 1004856: prices changed 3–4 times daily within a ~$3 band, throughout the
  period. (Note: Finding 5 in Part 3 later established this is characteristic of a minority of
  products, not the catalogue.)
- Considered simplifying to one price per product per day. Rejected — it produces *more* rows
  (4,998,112) than change-point tracking, because it creates a row for every day a product has
  any activity, not only days when the price moved.
- Evaluated full-fidelity tracking: 777,246 rows across 206,876 products. Small relative to the
  100M+ row fact tables, so no performance argument for simplifying.

**Decision:** full fidelity. One row per genuine price change, using `LAG()` to detect
transitions and `LEAD()` to close each period with a `valid_to`.

**Verification:** 206,876 products, exactly 206,876 rows flagged `is_current` — one current
version each, no gaps or duplicates. Product 1004856's history matched manually inspected change
points exactly.

---

## Decision 10: `dim_user` cohort anchor

**Decision:** anchor acquisition cohorts on **first purchase**, not first activity.

**Reasoning:** Q4 and Q5 are both about customer value. Only 697,470 of 5,316,649 users ever
purchase — 13.1%. Anchoring on first activity would build cohorts in which 87% of members could
never be retained as customers, because they were never customers. The grid would measure
whether browsers return to browse, a different question.

**Limitation:** this excludes the browse-to-first-purchase journey from cohort analysis. Both
date columns are retained, so first-activity cohorts remain computable if needed.

**Verification:** 5,316,649 rows (exact match to distinct users in the event data), 697,470
purchasers (exact match), total revenue $451,807,232.56 (exact match to the purchase table).

---

## Decision 11: same-second price collisions in `dim_product`

**Situation:** validating `fact_order_item` revealed 1,792 purchases (0.12%) where the price on
the purchase event disagreed with the price `dim_product` said was in effect. Gaps ran up to
$10, so not rounding error.

**Investigation:** inspected product 1004792 around a mismatched purchase. Found four price
versions within six seconds, two starting at the identical timestamp (08:26:23), and one with
`valid_from` equal to `valid_to` — a version valid for zero seconds. Root cause: the source
records multiple different prices for the same product within the same second, and the window
function ordered only by `event_time`, leaving no deterministic tiebreaker. Scope: 2,579
zero-length windows across 777,246 versions (0.33%).

**Fix:** collapse the source to one price per (product, second) before detecting change points,
taking `MAX(price)` within each second. `MAX` is arbitrary — `MIN` would be equally defensible —
but it is deterministic, which is what the window function requires.

**Result:** zero-length windows eliminated entirely (2,579 → 0). Version count 777,246 → 772,840.
Product count unchanged.

**Honest limitation:** the fix did **not** reduce the price mismatches (1,792 → 1,802). It could
not. Where the source records two contradictory prices in the same second, choosing one means
purchases recorded at the other will disagree by construction. This is a limit of the source
data, not of the model. What the fix achieved is structural integrity — the table now guarantees
exactly one valid price at any instant, which it previously did not.

**Consequent decision:** `fact_order_item` uses `event_price` for all revenue, not the
price-history price. Measured difference: $12,674 on $451.8m, or 0.0028% — analytically
irrelevant. The choice is made for clarity: `event_price` needs no join and is the direct record
of what the customer paid. `dim_product`'s role is price-history analysis, not revenue.

---

## Decision 12: splitting `fact_session` to work within memory limits

**Situation:** building `fact_session` in a single query failed with an out-of-memory error at
DuckDB's 12.5 GB limit. Cause: two `COUNT(DISTINCT ...)` columns across 110M rows required
tracking every unique product and category for all 18.8M sessions simultaneously.

**Attempted first:** `preserve_insertion_order=false` and a reduced thread count. Neither was
sufficient — the distinct-count memory requirement is structural, not incidental.

**Solution:** split into three stages, each writing to disk.
1. Session activity (counts and timings) from the event table, without distinct counts.
2. Breadth counts from `fact_session_product`. Because that table is already one row per
   session-product, counting products becomes `COUNT(*)` rather than `COUNT(DISTINCT ...)` —
   dramatically cheaper.
3. A light three-way join of the summarised pieces.

**Lessons:** splitting a query into disk-backed stages is usually more effective than tuning
memory settings. And an existing table at the right grain can make an expensive operation cheap.

**Verification:** 18,776,366 rows (matches the session count), revenue $451,807,232.56 (matches
three other tables), 1,291,108 purchasing sessions (matches the order count from Decision 5).

---

# Part 2 — Data quality findings

## `user_session` integrity

- 12 rows of 109,820,004 have a missing `user_session` — 0.00001%, negligible.
- 939 session IDs (0.004% of 23,016,650) are shared across multiple users.
- Inspected a case in detail: users 576718698 and 576718665 — numerically adjacent — both
  interacted with the same product under the same session ID, 28 seconds apart. Consistent with
  a session-ID generation collision in the source system, not genuine session sharing.

**Conclusion:** the supplied column is 99.996% clean. No cleaning action taken; documented as a
known negligible limitation. This finding motivated the independent session logic in Decision 4.

---

## Price validity

Investigated and found clean. No action required.

- No negative prices.
- 188,088 zero-price rows (0.28%), but **zero of them are purchase events** — no revenue impact.
- Highest prices check out: $2,574.07 on `electronics.clocks` branded `rado`, a real luxury watch
  brand. Plausible, not an error.
- Lowest non-zero prices check out: $0.77 on brands including `farmstay`, `ekel` and `tramontina`
   — real budget skincare and kitchenware brands.
- 82,965 distinct prices across 205,415 products — roughly 2.5 products per price point. Normal
  retail pricing, with no sign of artificial clustering.

**Distribution is heavily right-skewed:** median $165.77 against a mean of $292.46. The 90th
percentile is $756.75 and the 99th is $1,661.12. Reporting the mean alone would overstate what a
typical item costs.

---

## Timestamp coverage

All 61 days from 1 October to 30 November are present, with no missing dates and no days showing
outage-like drops in total volume. The lowest day (3 October, 1,126,624 events) sits close to the
normal baseline.

**However — see the next finding. This check was not sufficient.**

---

## Purchase events missing for 15 November 2019

**Discovered during:** Q5, when the daily revenue series skipped from 14 November to 16 November.

**What is missing:** all purchase events for 15 November. Views and carts are present at record
levels.

| Date | Views | Carts | Purchases |
|---|---|---|---|
| 13 Nov | 1,924,916 | 69,247 | 22,548 |
| 14 Nov | 2,877,071 | 165,541 | 22,124 |
| **15 Nov** | **5,737,078** | **468,262** | **0** |
| 16 Nov | 6,027,799 | 392,878 | 68,247 |
| 17 Nov | 5,783,122 | 411,604 | 185,195 |

15 November was the highest cart-volume day in the entire dataset with zero recorded purchases.
Not plausible as real behaviour; treated as a tracking failure.

**Why the earlier coverage check missed it:** that check tested for missing dates and for days
with unusually low *total* event counts. 15 November had 6.2M total events, so it passed both
comfortably. The gap only appears when completeness is checked **per event type**.

**Generalisable lesson:** a completeness check on total volume can hide the complete absence of
one event category. Coverage checks should run per category, not only in aggregate.

**Consequences:**
- Any daily purchase or revenue figure for 15 November is zero and must not be read as a
  business result.
- Purchases on 16–17 November may include late-recorded transactions from the 15th. The 17th
  shows 185,195 purchases against a ~20,000 baseline (9x), consistent with an extraordinary
  promotional day, batched recovery, or both. These cannot be distinguished.
- Q5's impact analysis was therefore conducted at block level rather than daily.

---

## Join fan-out when attaching brand

Joining `fact_session_product` to `dim_product` on `product_id` alone inflated view counts by
8.2% — 73,608,979 against the known 68,007,734.

**Cause:** `dim_product` has one row per price period, and 62 products have more than one brand
value across those periods — typically a null brand in one period and a real brand in another.

**Fix:** build a one-row-per-product lookup taking the most frequent non-null brand.

**Lesson:** only 62 products were affected, but they were high-traffic enough to add 5.6 million
duplicate rows. Fan-out severity depends on *which* rows are affected, not how many. The tell was
that the total disagreed with a figure already known.

---

# Part 3 — Analysis findings

## Q1: Conversion funnel

**Unit of analysis:** one product per session. Chosen because session grain destroys the category
and brand detail the question requires — a session touching three electronics items and two
apparel items collapses to `cart_count = 5`, from which category conversion cannot be recovered.

**Data limitation:** this dataset has no checkout-stage events, only view, cart and purchase. So
*checkout* abandonment is not measurable. Only *cart* abandonment — "added to cart, not bought" —
can be reported. These are different metrics.

**Window chosen:** same session, because every session has a complete observable outcome even one
starting on the final day of data. The "never bought at any point" figure is reported alongside.

### The strict funnel

| Stage | Product-sessions | Conversion |
|---|---|---|
| Viewed | 68,007,734 | — |
| Carted | 2,534,696 | 3.73% of viewed |
| Carted and purchased | 996,217 | 39.30% of carted |

**Same-session cart abandonment: 60.70%.**

**Methodology note:** an earlier loose calculation (all purchases ÷ all carts) gave 59.33%
cart-to-purchase, which nearly inverts the finding — it made most carts look like they convert,
when in fact most are abandoned. The strict version is reported because a conversion rate must
count the same journeys in numerator and denominator.

### The missing-cart problem

507,589 purchases — **33.8% of all purchases** — have no cart event in the same session.

Tested whether these were multi-visit journeys: only 39,529 (7.8%) had that product carted by the
same user in any earlier session. So 92% have no cart event anywhere in the data for that
user-product pair.

**Implication:** a strict funnel covers only 66.2% of purchases. The cause cannot be determined
here — either a checkout path that bypasses the cart, or incomplete cart tracking.

**Technical note:** the earlier-cart test initially returned 549,166 rows against a known total of
507,589 — a join fan-out from users carting the same product in several earlier sessions.
Corrected using `EXISTS`, which cannot multiply rows.

### By category

| Category | Viewed | View→cart | Cart→purchase |
|---|---|---|---|
| electronics | 23.3M | 5.87% | 44.79% |
| appliances | 7.5M | 3.81% | 36.01% |
| computers | 3.9M | 2.54% | 34.22% |
| apparel | 3.6M | 1.06% | 27.75% |
| furniture | 2.3M | 1.33% | 29.22% |

**View-to-cart varies 5.5x across named categories; cart-to-purchase varies less than 2x.**
Categories differ mainly in whether people cart at all, not in follow-through once they do.

**Electronics is 34% of all views** and the only named category above the overall benchmark on
both stages. This means the "overall" rate is largely electronics' own rate — a benchmark every
other category is measured against but barely influences. A fairer comparison for, say, apparel
would be against other non-electronics categories.

**Apparel is the clearest underperformer:** 3.6M views, only 1.06% carted. Cause not determinable
from this data.

**Limitation:** roughly 45 of 57 category groups have no readable name, including one with 3M
views — larger than apparel. Category analysis is complete in behaviour, incomplete in labelling.

### By brand

| Brand | Viewed | View→cart | Cart→purchase |
|---|---|---|---|
| unknown | 10.1M | 1.80% | 31.13% |
| samsung | 7.4M | 7.36% | 48.47% |
| apple | 6.3M | 7.17% | 44.36% |
| xiaomi | 4.2M | 5.28% | 36.92% |
| huawei | 1.5M | 4.87% | 46.82% |
| oppo | 0.8M | 5.22% | 49.28% |

Samsung and Apple cart at roughly twice the overall rate and together account for 13.7M views —
a fifth of all activity. The top four brands are all phone manufacturers, reinforcing the
concentration seen in the category cut. The unknown group (14.9% of views) is the largest single
bucket, so brand analysis excludes the biggest group.

### Statistical test: appliances versus computers

Two-proportion z-test on same-session cart-to-purchase rate.

| Category | Purchased | Carted | Rate |
|---|---|---|---|
| Appliances | 102,912 | 285,809 | 36.01% |
| Computers | 33,605 | 98,196 | 34.22% |

Difference 1.78 percentage points, 95% confidence interval [1.44, 2.13], z = 10.08, p effectively
zero.

**Interpretation:** the difference is real — the interval excludes zero. But it is small, and the
extremely low p-value reflects the sample size (384,005 carts), not the importance of the effect.
At this scale almost any difference tests as significant.

**What it would be worth:** applying appliances' rate to computers' cart volume would produce
roughly 1,750 additional purchases over two months. Whether that justifies intervention is a
commercial judgement, not a statistical one.

**Categories chosen deliberately:** appliances and computers are similar in kind and close in
rate. Comparing electronics against apparel would have produced a large obvious difference that
tested nothing.

---

## Q2: Session behaviour

### Distributions are heavily skewed

| | Median | 90th | 99th | Max | Mean |
|---|---|---|---|---|---|
| Duration (min) | 1.73 | 18.72 | 58.52 | 729.07 | 6.45 |
| Events | 3 | 14 | — | 4,128 | 5.85 |

The mean duration is 3.7x the median. Reporting it alone would suggest a typical visit is 6–7
minutes when half of all visits are under two minutes.

### Purchasing versus non-purchasing sessions

| | No purchase (93.1%) | Purchase (6.9%) |
|---|---|---|
| Sessions | 17,485,258 | 1,291,108 |
| Median duration | 1.45 min | 6.63 min |
| Median events | 3 | 7 |
| Median distinct products | 2 | 2 |
| Median distinct categories | 1 | 1 |

**The surprise: breadth does not differ.** Buyers and non-buyers both look at a median of two
products in one category. What differs is time and activity on that same narrow set. Visitors
arrive knowing roughly what they want; they do not browse widely and then commit.

This reframes the Q1 funnel finding. The 3.73% view-to-cart rate is not people wandering a large
catalogue and rejecting things — it is people looking at two products and mostly leaving.

**Causation caution:** purchasing sessions last 4.6x longer, but the direction is ambiguous.
Checking out itself takes time and generates events, so purchases plausibly cause longer sessions
rather than the reverse. This data cannot separate the two.

### Time of day

Timestamps are UTC. The quietest hour is 23:00 UTC, which is almost certainly the middle of the
night for these shoppers, implying local time of roughly **UTC+3**. Reporting raw UTC would place
peak traffic at 14:00 — "early afternoon" — when it is actually around 17:00 local, an evening
peak. Materially different operational conclusion.

**Traffic peak is not the conversion peak.** Sessions peak at 17:00–18:00 local, but purchase
rate peaks at 12:00 local (8.64%) and has fallen to 6.35% by the traffic peak — a 36% relative
difference. Evening traffic is high-volume and low-intent; midday traffic is the reverse.

### Outliers investigated, no action taken

47 sessions look like catalogue crawlers: 100+ events, every event on a different product, zero
purchases. The largest had 4,128 events across 4,128 distinct products in four hours — no product
viewed twice, which is not human browsing. These are 0.02% of all events, too small to affect any
distribution, so they were documented but not excluded.

Note these are a *different* population from the bot-like purchasing accounts in Decision 5.
There appear to be at least two kinds of non-human traffic in this dataset.

---

## Q3: Price movement and demand

### How much prices move

- **52.7% of products never changed price** in the two months.
- Median product: one price period. 90th percentile: 7. Maximum: 3,229 — roughly 53 changes/day.

**Correction to an earlier assumption:** investigating a single product (1004856) suggested
prices change 3–4 times daily, and Decision 9 was framed around that. Measuring across the
catalogue shows this is true of a small minority. The catalogue is mostly static with a volatile
tail.

**Price volatility tracks traffic almost perfectly:**

| Price versions | Products | Median views per product |
|---|---|---|
| 1 | 109,107 | 25 |
| 2–5 | 69,853 | 69 |
| 6–20 | 21,433 | 207 |
| 21–100 | 6,300 | 629 |
| 100+ | 183 | 64,597 |

Every band is higher than the one before, and the top band is 2,500x the bottom. Repricing is
applied **selectively to high-traffic products**, consistent with how commercial repricing tools
operate. Direction of causation is not determinable: products may be repriced because they are
popular, or popular because they are competitively repriced.

### Drift or bounce?

Among products repriced 10 or more times (16,464 products, median 17 versions): the typical
product ends within **0.3% of its starting price** despite travelling a **~15% range**. Prices
oscillate within a band rather than trending. Mild downward tilt — 61% ended lower than they
started.

**Methodology note:** an initial version of this metric included all products that changed price
and returned a net-to-range ratio of exactly 1.0. That is degenerate: with only two price
versions the ratio is 1 by construction, because max and min *are* the first and last prices.
Caught by noticing it contradicted the 0% net-change figure from the same query. Restricting to
10+ versions made the metric meaningful.

### Does demand respond?

Restricted to price changes with 24 hours of price stability on both sides — 208,852 of 565,964
periods (36.9%) qualify — then to changes of 10% or more.

| Direction | Changes | Events 24h before | Events 24h after | Change |
|---|---|---|---|---|
| Big drop (median −19.5%) | 25,530 | 14.47 | 19.69 | **+36.0%** |
| Big rise (median +21.9%) | 25,337 | 16.66 | 17.61 | **+5.7%** |

Small changes were excluded: the "small drop" group had a median change of −0.06%, which is
repricing noise rather than a price move any shopper would notice.

### Why this is not a measurement of price sensitivity

The gap is consistent with price sensitivity but cannot establish it. Three reasons:

1. **Repricing targets high-traffic products.** Demand and price changes are entangled from the
   start — whatever determines which products get repriced is correlated with demand.
2. **Demand rose after price increases too.** Price sensitivity alone predicts a fall. Something
   else — promotion, ranking, featured placement — is lifting activity in both groups.
3. **The groups had different baselines before any price moved** (14.47 versus 16.66 events).
   They were not comparable to begin with.

If the repricing system reacts to demand, the causal arrow points backwards. Estimating price
sensitivity would require price variation generated independently of demand: a randomised
experiment, or a natural experiment driven by an external cost or currency shock. Neither exists
in this data.

---

## Q4: Do customers come back?

**Cohort anchor: first purchase** (see Decision 10). **"Came back" means placed another order** —
consistent with the anchor, keeping the analysis about buying rather than mixing purchase and
browsing behaviour. **Weekly cohorts** — two months yields only two monthly cohorts; weekly gives
nine, of which the last two carry almost no observation window.

### Repeat purchase behaviour

| Measure | Value |
|---|---|
| Purchasing customers | 697,470 |
| One order only | 464,424 (66.6%) |
| Repeat customers | 233,046 (33.4%) |
| Median orders | 1 |
| Mean orders | 1.85 |
| 90th percentile | 3 |
| Maximum | 191 |

### Time to second purchase

| Percentile | Days |
|---|---|
| 25th | 0.9 |
| Median | 4.3 |
| 75th | 13.8 |

Repurchase is fast: 28.9% of repeat customers place a second order within 24 hours. This shapes
how the grid reads — most returns land in the first week or two because the repurchase cycle is
short, not because retention collapses.

**Caveat checked:** our order definition splits on 30-minute session gaps, so a customer who
pauses and buys again generates two orders. Only 2.5% of second orders occur within an hour of
the first, so this artefact does not materially inflate the 33.4% repeat rate.

### Retention grid

Values are the percentage of each cohort placing an order in that week. Blank cells are time that
has not elapsed, not missing data.

| Cohort | Size | Wk 1 | Wk 2 | Wk 3 | Wk 4 | Wk 5 | Wk 6 | Wk 7 | Wk 8 |
|---|---|---|---|---|---|---|---|---|---|
| 2019-09-30 | 78,674 | 20.3 | 17.5 | 13.6 | 11.3 | 11.3 | 13.8 | 8.5 | 7.6 |
| 2019-10-07 | 85,245 | 16.1 | 11.1 | 8.9 | 8.9 | 12.1 | 6.7 | 6.3 | — |
| 2019-10-14 | 81,004 | 13.5 | 9.1 | 9.0 | 11.5 | 6.7 | 5.8 | — | — |
| 2019-10-21 | 69,837 | 11.0 | 8.9 | 11.3 | 6.3 | 5.6 | — | — | — |
| 2019-10-28 | 58,851 | 12.2 | 11.9 | 6.6 | 5.7 | — | — | — | — |
| 2019-11-04 | 68,252 | 14.7 | 7.6 | 6.1 | — | — | — | — | — |
| **2019-11-11** | **146,154** | **7.2** | **5.3** | — | — | — | — | — | — |
| 2019-11-18 | 57,783 | 11.4 | — | — | — | — | — | — | — |
| 2019-11-25 | 51,670 | — | — | — | — | — | — | — | — |

**The promotional cohort is the largest and the worst-retaining.** The week of 11 November
contains the promotional period. It acquired 146,154 first-time buyers — roughly double a normal
week, so about 80,000 additional customers — but only 7.2% returned the following week, against
11–20% for every other cohort. The following week returns to 11.4%, so this is specific to the
promotion rather than a general November decline.

**Interpretation:** the promotion attracted deal-seekers rather than customers. It expanded the
base substantially but at materially lower quality. This is a direct caveat to any "new customers
acquired" figure from Q5.

**Earlier cohorts appear to retain better**, but the comparison is not clean. Earlier cohorts have
longer observation windows, and customers still active weeks later are by construction more
engaged. The apparent decline down the grid should not be read as deteriorating acquisition
quality.

### Limitations

- Two months is a short horizon. Only the first cohort has eight weeks of observation.
- The censored region is structural. The final cohort has five days of data; no conclusion about
  recent cohorts can be drawn.
- Week boundaries snap to Monday, so the first cohort is labelled 2019-09-30 even though data
  begins 1 October, a Tuesday.

---

## Q5: Customer segmentation and promotional impact

### Part 1 — RFM segmentation

**Snapshot date: 2019-11-30**, the last day of data. Chosen over a pre-promotion date to retain
all data; the resulting distortion is reported rather than engineered away.

**Scoring:** recency and monetary use quantile bins (`NTILE(5)`) — both have enough distinct
values to split evenly, and each bin holds exactly 139,494 customers (20.0%).

**Frequency uses manual bins.** Quantile binning is impossible here: 66.6% of purchasing
customers have exactly one order, so the first three quantile boundaries would all fall on the
same value. `pd.qcut` would either error or, with `duplicates='drop'`, silently return three bins
while the analyst assumes five — breaking every segment definition expecting F=4 or F=5.

| F score | Orders | Customers | % |
|---|---|---|---|
| 1 | 1 | 464,424 | 66.6% |
| 2 | 2 | 125,652 | 18.0% |
| 3 | 3–4 | 68,478 | 9.8% |
| 4 | 5–9 | 29,081 | 4.2% |
| 5 | 10+ | 9,835 | 1.4% |

Deliberately unequal. Each bin corresponds to a distinguishable behaviour pattern, which is more
useful than five bins pretending to be equal. A reader should not assume each score covers 20%.

**Segments** are defined on R and F only; monetary is reported but not used for assignment.

| Segment | Customers | % customers | % revenue | Median spend | Median recency | Median orders |
|---|---|---|---|---|---|---|
| Champions | 25,617 | 3.7% | 23.7% | $2,344 | 4 days | 7 |
| Lapsed | 255,295 | 36.6% | 19.4% | $190 | 44 days | 1 |
| New / recent | 216,735 | 31.1% | 19.2% | $224 | 11 days | 1 |
| Loyal | 58,084 | 8.3% | 18.7% | $829 | 13 days | 3 |
| Occasional | 135,478 | 19.4% | 14.1% | $236 | 21 days | 1 |
| At risk (was loyal) | 6,261 | 0.9% | 5.0% | $2,106 | 36 days | 6 |

**Revenue is concentrated:** Champions are 3.7% of customers and 23.7% of revenue. Champions plus
Loyal are 12% of customers and 42% of revenue.

**Two recommended actions:**

*At risk (was loyal)* — 6,261 customers with median spend of $2,106, comparable to Champions, but
median recency of 36 days against Champions' 4. Proven high-value buyers who have gone quiet,
representing roughly $13m of at-risk revenue. A targeted win-back contact is economically
favourable: re-engaging known buyers is materially cheaper than acquiring new ones.

*New / recent* — 216,735 customers, median one order, median spend $224. Q4 established the median
time to a second purchase is 4.3 days, so the conversion window is short. A second-purchase prompt
within the first week is where this segment converts into Loyal.

**Caveats:**
- **Segment boundaries are judgement calls.** "Champions = R≥4 and F≥4" is a convention, not a
  standard. Different thresholds produce different segment sizes.
- **The promotion distorts the segmentation.** New/recent is 31% of the base, inflated by the
  146,154 first-time buyers from the promotional week. Q4 showed that cohort retains at 7.2%, so
  a substantial share are deal-seekers unlikely to convert regardless of prompting.
- **"Lapsed" is an artefact of the short window.** A last purchase 44 days ago means mid-October,
  barely into the dataset. With a year of history, 44 days would not be lapsed.
- **Extreme spenders distort the top monetary bin.** The highest-spending customer accounts for
  $274,440 across 191 orders and 476 items, averaging $577 per item. These accounts show normal
  basket sizes and plausible prices — they appear to be trade or reseller buyers rather than the
  bot-like accounts identified elsewhere, and are retained as genuine revenue.

### Part 2 — Promotional impact

**Period identified from the data, not assumed.** Daily views rise from ~1.9M to 2.9M on 14
November and 5.7–6.0M across 15–17, returning to 1.9M on the 18th. This is **not Black Friday**,
which fell on 29 November 2019 and produced only a modest lift (25,415 orders against a ~19,000
baseline).

**Method: block comparison.** The promotional block (Thu 14 – Sun 17 Nov) is compared against the
same Thursday-to-Sunday weekdays from the three preceding weeks. Weekday matching controls for
day-of-week effects. Block rather than daily comparison was necessary because purchase tracking
failed on 15 November, making daily attribution within the window unreliable.

**Baseline stability:** the three baseline blocks produced 74,108 / 69,246 / 77,012 orders —
within about 10% of each other, so the average is a fair representation of a normal period.

| Measure | Baseline avg | Promo (14–17 Nov) | Incremental | Lift |
|---|---|---|---|---|
| Orders | 73,455 | 202,230 | +128,775 | +175% |
| Revenue | $25,018,382 | $78,751,198 | **+$53.7m** | +215% |
| New customers | 36,885 | 119,932 | +83,047 | +225% |

**Average order value rose 14.3%** ($340.59 → $389.41) — counterintuitive for a discount event.
Decomposing:

| Component | Baseline | Promo | Change |
|---|---|---|---|
| Items per order | 1.14 | 1.24 | +8.8% |
| Average item price | $298.26 | $313.93 | +5.3% |

Both contributed, with larger baskets doing slightly more of the work. The rise in average *item*
price suggests a mix shift toward more expensive products — consistent with Q3's finding that
repricing targets high-traffic items, if the promotion discounted premium goods specifically.

**Pull-forward test: none detected within the observable window.**

| Period | Orders/day | vs baseline |
|---|---|---|
| Baseline average | 18,364 | — |
| Week 1 after (21–24 Nov) | 18,496 | +0.7% |
| Week 2 after (28–30 Nov) | 22,164 | +20.7% |

Demand returned immediately to baseline and did not dip. The week-2 figure is inflated by Black
Friday falling inside it.

**Uncertainty and limitations:**
- **This is an estimate, not proof.** There is no control group. The comparison assumes the
  baseline weeks represent what the promotional week would have been otherwise, which cannot be
  verified. A randomised holdout — a matched group excluded from promotional messaging — would
  give a causal answer.
- **Week-to-week variation** in the baselines (~10%) implies the estimate could shift by a few
  million either way from noise alone.
- **The 15 November tracking gap** means the true figure may be higher if purchases were lost
  rather than re-attributed. The two cases cannot be distinguished.
- **Pull-forward beyond two weeks is untestable.** Only 13 days of post-promotion data exist. For
  durable goods, which dominate this catalogue, substitution over months is plausible and
  entirely invisible here.

**Anomaly detection was considered and deliberately not run.** The question suggests a rolling
z-score or seasonal decomposition to surface unusual days. All anomalous dates had already been
identified directly — the promotional spike via daily aggregation, the Black Friday bump, and the
15 November gap via per-event-type coverage checks. A statistical detector would have flagged the
same three dates without adding information.

---

# Summary of headline numbers

| | |
|---|---|
| Events analysed | 109,820,004 |
| Total revenue | $451,807,232.56 |
| Users | 5,316,649 (13.1% ever purchase) |
| Sessions | 18,776,366 |
| Orders | 1,291,108 |
| Products | 206,876 |
| View → cart | 3.73% |
| Cart → purchase (same session) | 39.30% |
| Cart abandonment (same session) | 60.70% |
| Repeat purchase rate | 33.4% |
| Revenue from top 12% of customers | 42% |
| Promotional incremental revenue | ~$53.7m over 4 days |

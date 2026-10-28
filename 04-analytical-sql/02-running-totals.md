# Running Totals (Cumulative Aggregates)

> A running total is the simplest window function you will ever write and the one most likely
> to be silently wrong. `SUM(x) OVER (ORDER BY d)` looks like "cumulative sum" and reads like
> "cumulative sum", but its **default frame is `RANGE`**, which means every row that *ties* on
> `d` gets the *same* cumulative value — and if `d` is a date with several orders per day, your
> "daily running revenue" is actually a "running revenue through the end of this day" metric
> wearing a daily label. This chapter is about knowing which of the two you need, plus the
> genuinely hard versions: resets, per-group restarts, cumulative *distinct* counts, and
> semi-additive measures where a cumulative sum is not just wrong but meaningless.

## TL;DR

- `SUM(x) OVER (ORDER BY t)` **is not** `SUM(x) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)`.
  The first is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — it includes all **peers**,
  so ties share a value. The second adds exactly the rows before the current one.
- **Without `ORDER BY`, there is no running total** — `SUM(x) OVER (PARTITION BY g)` is the
  whole-partition total, repeated on every row. This is useful (share-of-group) and frequently
  mistaken for a cumulative metric.
- Rule of thumb: **`ROWS` for "row by row in this order", `RANGE` for "everything up to this
  value/time"**. Use `RANGE` deliberately, not by omission.
- Running **`COUNT`**, running **`AVG`**, running **`MIN`/`MAX`** ("best so far") are all the
  same shape. Running `AVG` is *not* a moving average — it is a cumulative mean that lags
  forever (chapter 03).
- **`COUNT(DISTINCT x) OVER (...)` is illegal** in PostgreSQL, MySQL, SQL Server, Oracle and
  BigQuery. Get cumulative distinct counts by (a) summing a "first occurrence" flag,
  (b) a densified calendar + per-bucket `COUNT(DISTINCT)`, or (c) an approximate sketch
  (`HLL_COUNT`/`APPROX_COUNT_DISTINCT`) where approximate is acceptable.
- **Semi-additive measures cannot be summed over time.** A balance, an MRR snapshot, a headcount
  or an inventory level is additive across *entities* but not across *dates*. Running-summing an
  MRR snapshot produces a meaningless 3× inflated number. Aggregate with `LAST_VALUE`/`MAX` per
  period instead, or decompose into movements (new/expansion/contraction/churn).
- **Resets** come in two flavours: *calendar partitions* (`PARTITION BY month`) and
  *event-driven* (running sum of a "reset" flag → a group id → `PARTITION BY` that id). The
  second is `gap-and-islands` (chapter 08) and is the generalizable one.
- In an **incremental model** you cannot recompute a running total by adding yesterday's delta
  to yesterday's total unless you also carry the prior total; the classic failure is a
  double-counted or reset cumulative column after a partial backfill.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Running / cumulative aggregate** | An aggregate whose frame starts at the partition start and ends at the current row (or the current row's peers). |
| **Frame** | `{ROWS \| RANGE \| GROUPS} BETWEEN <start> AND <end>`, relative to the current row. |
| **`ROWS` mode** | Offsets are **positions**: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` = 3 physical rows. |
| **`RANGE` mode** | Offsets are **values**: `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` = all rows whose ordering value is within 7 days. |
| **`GROUPS` mode** | Offsets are **peer groups**: `GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW` = current peer group plus the one before. PostgreSQL-only (plus DuckDB). |
| **Peer** | A row that ties with the current row under the window `ORDER BY`. |
| **Default frame** | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` when `ORDER BY` is present; **the whole partition** when it is not. |
| **Semi-additive measure** | A measure additive across some dimensions but not across time (balances, levels, snapshots). |
| **Cumulative distinct count** | `COUNT(DISTINCT ...)` accumulated over an ordered sequence — the "how many unique users so far" metric. |
| **Prefix sum / delta encoding** | Materializing a cumulative column so point-in-range sums become a difference of two lookups. |
| **Compound growth / CAGR** | A running total applied to *ratios* is a running **product**, not a sum (`EXP(SUM(LN(x)))`). |

### 1.2 The frame in one picture

```
partition rows sorted by ORDER BY t
        t=1   t=2   t=2   t=3        t=4
        ┌───┬───┬───┬───┬───┐
RANGE UNBOUNDED PRECEDING:
   row t=1  frame: [1]
   row t=2  frame: [1, 2a, 2b]   ← both peers get the SAME value
   row t=3  frame: [1, 2a, 2b, 3]
ROWS UNBOUNDED PRECEDING:
   row t=1  frame: [1]
   row t=2a frame: [1, 2a]
   row t=2b frame: [1, 2a, 2b]   ← different values for peers
   row t=3  frame: [1, 2a, 2b, 3]
```

Both reach the same **final** value. They differ only on tie rows — which is exactly where
dashboards live (daily aggregates, hourly buckets, same-second events).

### 1.3 The default frame, exhaustively

| Expression | Effective frame | Notes |
|------------|-----------------|-------|
| `SUM(x) OVER ()` | whole partition | total repeated per row |
| `SUM(x) OVER (PARTITION BY g)` | whole partition per `g` | group total repeated per row (share-of-group) |
| `SUM(x) OVER (ORDER BY t)` | `RANGE UNBOUNDED PRECEDING AND CURRENT ROW` | **cumulative including peers** |
| `SUM(x) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)` | rows 1..current | **strict cumulative** |
| `SUM(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` | whole partition | total, ordered context (same as `OVER ()`) |
| `AVG(x) OVER (ORDER BY t)` | cumulative mean including peers | lags forever; not a moving average |

> **Senior note.** In 2018 Mode's SQL tutorial, in `use-the-index-luke`, and in a thousand
> blog posts, the canonical "running total" is written as
> `SUM(duration_seconds) OVER (ORDER BY start_time)`. That is correct *only because
> `start_time` is unique in their dataset*. In your dataset it usually is not. Write the frame
> explicitly every time and you will never have to debug a "why do these two rows have the
> same cumulative value" ticket.

## 2. Why it matters / what problem it solves

Running totals are the backbone of the metrics layer:

| Metric family | Definition | Running total role |
|---------------|-----------|--------------------|
| **Cumulative revenue / GMV** | Revenue from launch to date | `SUM(revenue) OVER (ORDER BY date)` |
| **Month-to-date (MTD), quarter-to-date, year-to-date** | Revenue since period start | `PARTITION BY period` + cumulative |
| **Cumulative user growth** | Registered (or activated, or paying) users to date | cumulative `COUNT` / cumulative distinct |
| **Burn-down / burn-up charts** | Remaining work vs. time | cumulative of signed deltas |
| **Budget consumption** | Spend against a budget | cumulative `SUM` of spend |
| **Balance / inventory level** | Level, not flow | semi-additive — must *not* be summed |
| **Cohort "spread"** | Share of a cohort that has done something by day N | cumulative share |
| **Pareto / ABC analysis** | Cumulative share of revenue ordered by seller | cumulative `SUM` + `CUME_DIST` interplay |
| **Data-quality tracking** | Cumulative count of failing rows since an incident | cumulative `COUNT` of a flag |

The failure cost is high because these numbers land on executive dashboards:

- A cumulative-revenue series that uses the default `RANGE` frame on daily data reports
  **"revenue through the end of each day"** labelled as **"revenue on each day"**. The
  **end-of-period value is correct** and every intermediate value is wrong, which is exactly the
  bug that survives review.
- A "cumulative distinct users" series implemented with
  `COUNT(DISTINCT user_id) OVER (ORDER BY d)` fails to run at all on most engines, and when
  people "fix" it by dropping `DISTINCT` they get cumulative *events*, not users.
- An MRR figure produced by `SUM(mrr) OVER (ORDER BY snapshot_date)` over monthly snapshots
  reports Q1 MRR = 793 when the correct answer is "MRR at end of Q1 = 646". The number is not
  off by a rounding error; it is a different quantity entirely.

## 3. The pattern / technique explained

### 3.1 The canonical shape

```sql
<agg>(<expr>) OVER (
    PARTITION BY <reset keys>              -- optional: where the total restarts
    ORDER BY     <time key>                -- required for a running total
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- be explicit
)
```

Three decisions, in order:

1. **Where does it restart?** Nothing → one total for the whole result set.
   `PARTITION BY month` → MTD. `PARTITION BY user_id` → per-customer cumulative.
2. **What is the time key?** Whatever defines "before" — a date, a timestamp, a monotonically
   increasing id, a sequence number. It must be **unique within the partition** if you want
   per-row values; otherwise the peers share values under `RANGE`.
3. **Which frame?** `ROWS UNBOUNDED PRECEDING AND CURRENT ROW` unless you can articulate why
   `RANGE` is what you mean.

### 3.2 `RANGE` vs `ROWS`: the worked decision

| Situation | Use | Why |
|-----------|-----|-----|
| One row per day, day is unique | either (same answer) | no peers |
| Several rows per day; you want "as of end of day" | `RANGE` | peers share the day total |
| Several rows per day; you want per-row progression | `ROWS` | each row advances |
| Timestamps with duplicate seconds (event streams) | `ROWS` + unique tiebreaker | `RANGE` collapses the second |
| "Cumulative over the last 7 days" (calendar days) | `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` | value-based window |
| "Cumulative over the last 7 rows" | `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | position-based window |
| Sparse series where you want time, not rows | `RANGE` **after densifying the calendar** | gaps otherwise shrink the window |
| Ordered by a *non-numeric* key (a string category) | `ROWS` or `GROUPS` | `RANGE` offsets need numeric/temporal |

### 3.3 Running `AVG` versus moving average

```sql
-- cumulative mean (this chapter): every row sees all history
AVG(x) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)

-- moving average (chapter 03): a fixed-size window
AVG(x) OVER (ORDER BY t ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
```

The cumulative mean is intentionally "sticky": with 10,000 rows it barely moves, which makes it
a good *long-run baseline* and a terrible *trend indicator*. Use it for
"average order value since launch", not for "AOV this week".

### 3.4 Running totals with *resets* — two mechanisms

**(a) Calendar-partitioned reset** — the total restarts because the partition restarts:

```sql
SUM(x) OVER (PARTITION BY date_trunc('month', t) ORDER BY t ROWS UNBOUNDED PRECEDING)
```

**(b) Event-driven reset** — the total restarts when a *row* says so. You build a group id with
a running sum of the reset flags, then partition by it:

```sql
WITH marked AS (
    SELECT t, x, is_reset,
           SUM(CASE WHEN is_reset THEN 1 ELSE 0 END)
               OVER (ORDER BY t ROWS UNBOUNDED PRECEDING) AS cycle_id
    FROM   moves
)
SELECT t, x,
       SUM(x) OVER (PARTITION BY cycle_id ORDER BY t ROWS UNBOUNDED PRECEDING) AS since_reset
FROM   marked;
```

This is **the** generalizable technique for resets, and it is exactly the `gap-and-islands`
grouping key from chapter 08. Learn it once.

### 3.5 Cumulative distinct counts — why they are hard

`COUNT(DISTINCT expr) OVER (...)` is rejected by PostgreSQL
(`ERROR: DISTINCT is not implemented for window functions`), MySQL, SQL Server, Oracle and
BigQuery. The reason is algorithmic: a sliding distinct count cannot be maintained with a
constant-memory accumulator the way `SUM` can; every new row may change the distinct set for
every subsequent row.

Three legitimate strategies, in order of preference:

| Strategy | Query shape | Cost | Exact? |
|----------|-------------|------|--------|
| **First-occurrence flag** | `SUM(is_first) OVER (ORDER BY t)` | O(n) | ✅ (only for "distinct *new* X to date") |
| **Bucket + per-bucket distinct** | pre-aggregate per day with `COUNT(DISTINCT)`, then cumulative sum | O(n) after one group-by | ✅ |
| **Densified self-join distinct** | `COUNT(DISTINCT d2.x) ... AND d2.t <= d1.t` | O(n²) worst case | ✅ |
| **Approximate sketch** | `HLL_COUNT` / `APPROX_COUNT_DISTINCT` / `HLL` | O(n) memory-bound | ❌ (±~1–2%) |

The first occurrence flag is the workhorse: it converts "distinct" into "count the rows where
this is the first time we see the entity", which *is* cumulative-additive.

### 3.6 Semi-additive measures

A measure is **additive** (sum across any dimension), **non-additive** (ratios, percentages),
or **semi-additive** (additive across some dimensions, not others — usually not time).

| Measure | Across users | Across time | Correct time aggregation |
|---------|--------------|-------------|--------------------------|
| Revenue (flow) | ✅ sum | ✅ sum | `SUM` per period, cumulative `SUM` over periods |
| Headcount (level) | ✅ sum | ❌ | `LAST_VALUE` / `MAX` at period end |
| MRR, ARR (level) | ✅ sum | ❌ | latest snapshot per period |
| Bank balance (level) | ✅ sum across accounts | ❌ | latest snapshot, or start + cumulative flows |
| Inventory on hand (level) | ✅ sum across SKUs | ❌ | latest snapshot; changes only via movements |
| Conversion rate (ratio) | ❌ | ❌ | recompute from numerator/denominator |

The rule: **for a level measure, the running total is not a cumulative sum of the level — it is
a time series of the level itself, or a cumulative sum of its *changes*.**

## 4. Step-by-step worked example (with sample data and expected output tables)

All examples run against the shared schema from `01-ranking.md` (`daily_revenue`, `orders`,
`order_revenue`, `events`, `users`).

### 4.1 Example 1 — the basic running total

**Problem statement.** Show cumulative recognized revenue by the date it was recognized.

```sql
SELECT revenue_date,
       revenue,
       SUM(revenue) OVER (ORDER BY revenue_date
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS revenue_ytd
FROM   daily_revenue
ORDER  BY revenue_date;
```

| revenue_date | revenue | revenue_ytd |
|--------------|--------:|------------:|
| 2024-01-06 | 170.00 |  170.00 |
| 2024-01-08 | 190.00 |  360.00 |
| 2024-01-20 |  89.00 |  449.00 |
| 2024-02-02 | 178.00 |  627.00 |
| 2024-02-05 | 110.00 |  737.00 |
| 2024-02-11 | 210.00 |  947.00 |
| 2024-02-15 | 100.00 | 1047.00 |
| 2024-02-18 | 197.00 | 1244.00 |
| 2024-03-03 | 120.00 | 1364.00 |
| 2024-03-05 | 305.00 | 1669.00 |
| 2024-03-08 |  19.00 | 1688.00 |
| 2024-03-21 | 134.00 | 1822.00 |
| 2024-03-25 | 120.00 | 1942.00 |
| 2024-04-02 | 335.00 | 2277.00 |
| 2024-04-06 |  25.00 | 2302.00 |
| 2024-04-09 |  38.00 | 2340.00 |
| 2024-04-12 | 209.00 | 2549.00 |
| 2024-04-15 | 390.00 | 2939.00 |
| 2024-04-20 | 120.00 | 3059.00 |

The last value equals `SELECT SUM(revenue) FROM daily_revenue` → `3059.00`. **Always assert
this**: a cumulative column's final value must equal the plain total. If it does not, you have
a frame bug or you are joining to a table that multiplies rows.

### 4.2 Example 2 — the `RANGE`/`ROWS` trap, made concrete

**Problem statement.** You have *several orders per day* and you want a cumulative daily
revenue series. Here is the smallest data set that shows the difference.

```sql
WITH sales(day, order_id, amount) AS (
    VALUES (DATE '2024-01-06', 1, 170.00),
           (DATE '2024-01-06', 2,  50.00),
           (DATE '2024-01-06', 3,  20.00),
           (DATE '2024-01-08', 4, 190.00),
           (DATE '2024-01-09', 5,  89.00)
)
SELECT day, order_id, amount,
       SUM(amount) OVER (ORDER BY day)                                    AS cum_default_range,
       SUM(amount) OVER (ORDER BY day ROWS UNBOUNDED PRECEDING)           AS cum_rows,
       AVG(amount) OVER (ORDER BY day)                                    AS avg_default_range
FROM   sales
ORDER  BY day, order_id;
```

| day | order_id | amount | cum_default_range | cum_rows | avg_default_range |
|-----|---------:|-------:|------------------:|---------:|------------------:|
| 2024-01-06 | 1 |  170.00 | 240.00 | 170.00 | 80.00 |
| 2024-01-06 | 2 |   50.00 | 240.00 | 220.00 | 80.00 |
| 2024-01-06 | 3 |   20.00 | 240.00 | 240.00 | 80.00 |
| 2024-01-08 | 4 |  190.00 | 430.00 | 430.00 | 107.50 |
| 2024-01-09 | 5 |   89.00 | 519.00 | 519.00 | 103.80 |

Read this carefully:

- `cum_default_range` is **240.00 for all three Jan-6 rows** — the peers share the value. If you
  labelled this column "revenue on 2024-01-06" you would be overstating it for two of the rows;
  if you meant "revenue through 2024-01-06", 240 is exactly right.
- `cum_rows` gives 170 / 220 / 240 — a genuine row-by-row progression. Only the *last* row of
  each day equals the end-of-day cumulative.
- `avg_default_range` is 80.00 for all three peers (240/3), whereas the row-by-row cumulative
  mean would be 170, 110, 80.
- Both cumulative columns agree at every day's **last** row. That is the invariant you can test
  in CI: `MAX(cum_rows) OVER (PARTITION BY day) = cum_default_range`.

**Reasoning.** `ORDER BY` creates peer groups; `RANGE` frames are defined over *values*, so
"CURRENT ROW" in `RANGE` mode means "the last peer of the current row". This is standards
behaviour (`SQL:2003`), not an engine quirk, and it is the same in every engine.

### 4.3 Example 3 — which frame do you actually want?

**Problem statement.** Publish a **daily** revenue table (one row per day, no ties) *and* an
**order-level** table for finance. The metric definition differs.

```sql
-- Daily series: collapse to one row per day FIRST, then accumulate.
SELECT d.revenue_date, d.revenue,
       SUM(d.revenue) OVER (ORDER BY d.revenue_date
                            ROWS UNBOUNDED PRECEDING) AS revenue_to_date
FROM  (SELECT revenue_date, SUM(revenue) AS revenue
       FROM   order_revenue
       WHERE  status <> 'cancelled'
       GROUP  BY revenue_date) d
ORDER BY d.revenue_date;
```

| revenue_date | revenue | revenue_to_date |
|--------------|--------:|----------------:|
| 2024-01-06 | 170.00 |  170.00 |
| 2024-01-08 | 190.00 |  360.00 |
| 2024-01-20 |  89.00 |  449.00 |
| 2024-02-02 | 178.00 |  627.00 |
| … | … | … |
| 2024-04-20 | 120.00 | 3059.00 |

**Naive/incorrect approach.** Running the window directly over the order-level rows and calling
it "daily revenue":

```sql
-- ❌ one row per ORDER, presented as if it were one row per DAY
SELECT order_date, order_id, revenue,
       SUM(revenue) OVER (ORDER BY order_date) AS daily_running_total   -- RANGE + duplicates
FROM   order_revenue
WHERE  status <> 'cancelled'
ORDER  BY order_date, order_id;
```

Because most days have exactly one order in this sample, this *looks* fine — until you hit a day
with multiple orders (there are none in `daily_revenue`, but there are in a real dataset, and the
`sales` CTE in §4.2 proves the behaviour). **Aggregate to the grain you intend to publish, then
accumulate.** This is the single most important habit in this chapter.

The **golden rule**:

```
accumulate AFTER you aggregate to the target grain;
aggregate with COUNT(DISTINCT) BEFORE you accumulate distinct counts.
```

### 4.4 Example 4 — running totals per group (and why `GROUP BY` cannot replace it)

**Problem statement.** Per customer, cumulative revenue over time, alongside their final total.

```sql
WITH per_order AS (
    SELECT user_id, order_ts::date AS d, SUM(revenue) AS revenue
    FROM   order_revenue
    WHERE  status <> 'cancelled'
    GROUP  BY user_id, order_ts
)
SELECT user_id,
       d,
       revenue,
       SUM(revenue) OVER (PARTITION BY user_id ORDER BY d ROWS UNBOUNDED PRECEDING) AS cust_running,
       SUM(revenue) OVER (PARTITION BY user_id)                                    AS cust_total,
       ROUND(100.0 * SUM(revenue) OVER (PARTITION BY user_id ORDER BY d ROWS UNBOUNDED PRECEDING)
                   / SUM(revenue) OVER (PARTITION BY user_id), 1)                   AS pct_of_cust_total
FROM   per_order
ORDER  BY user_id, d;
```

| user_id | d | revenue | cust_running | cust_total | pct_of_cust_total |
|--------:|-----------|--------:|-------------:|-----------:|------------------:|
| 1 | 2024-01-06 | 170.00 |  170.00 | 494.00 |  34.4 |
| 1 | 2024-01-20 |  89.00 |  259.00 | 494.00 |  52.4 |
| 1 | 2024-02-11 | 210.00 |  469.00 | 494.00 |  94.9 |
| 1 | 2024-04-06 |  25.00 |  494.00 | 494.00 | 100.0 |
| 2 | 2024-01-08 | 190.00 |  190.00 | 439.00 |  43.3 |
| 2 | 2024-02-05 | 110.00 |  300.00 | 439.00 |  68.3 |
| 2 | 2024-03-08 |  19.00 |  319.00 | 439.00 |  72.7 |
| 2 | 2024-04-20 | 120.00 |  439.00 | 439.00 | 100.0 |
| 4 | 2024-02-02 | 178.00 |  178.00 | 495.00 |  36.0 |
| 4 | 2024-02-18 | 197.00 |  375.00 | 495.00 |  75.8 |
| 4 | 2024-03-25 | 120.00 |  495.00 | 495.00 | 100.0 |
| 5 | 2024-02-15 | 100.00 |  100.00 | 138.00 |  72.5 |
| 5 | 2024-04-09 |  38.00 |  138.00 | 138.00 | 100.0 |
| 6 | 2024-03-03 | 120.00 |  120.00 | 120.00 | 100.0 |
| 7 | 2024-03-05 | 305.00 |  305.00 | 695.00 |  43.9 |
| 7 | 2024-04-15 | 390.00 |  695.00 | 695.00 | 100.0 |
| 8 | 2024-03-21 | 134.00 |  134.00 | 134.00 | 100.0 |
| 9 | 2024-04-02 | 335.00 |  335.00 | 335.00 | 100.0 |
|10 | 2024-04-12 | 209.00 |  209.00 | 209.00 | 100.0 |

Note that `pct_of_cust_total` reaches exactly 100.0 on each customer's **last** row and never
exceeds it — a good assertion to add to a dbt test.

**Incorrect approach.** Trying to get the running value with `GROUP BY`:

```sql
-- ❌ GROUP BY collapses rows; there is nowhere to put the cumulative value
SELECT user_id, order_ts::date, SUM(revenue) AS revenue   -- cumulative what?
FROM   order_revenue GROUP BY user_id, order_ts;
```

`GROUP BY` reduces many rows to one. A running total needs the *unreduced* row plus a
context-dependent aggregate — that is exactly what a window is.

### 4.5 Example 5 — running totals that reset every month (MTD)

**Problem statement.** A finance dashboard needs revenue **month-to-date** on a daily grid, and
the same query must also produce YTD.

```sql
SELECT d.revenue_date,
       date_trunc('month', d.revenue_date)::date AS month,
       d.revenue,
       SUM(d.revenue) OVER (PARTITION BY date_trunc('month', d.revenue_date)
                            ORDER BY d.revenue_date
                            ROWS UNBOUNDED PRECEDING)                      AS revenue_mtd,
       SUM(d.revenue) OVER (PARTITION BY date_trunc('year', d.revenue_date)
                            ORDER BY d.revenue_date
                            ROWS UNBOUNDED PRECEDING)                      AS revenue_ytd
FROM   daily_revenue d
ORDER  BY d.revenue_date;
```

| revenue_date | month | revenue | revenue_mtd | revenue_ytd |
|--------------|-------|--------:|------------:|------------:|
| 2024-01-06 | 2024-01-01 | 170.00 |  170.00 |  170.00 |
| 2024-01-08 | 2024-01-01 | 190.00 |  360.00 |  360.00 |
| 2024-01-20 | 2024-01-01 |  89.00 |  449.00 |  449.00 |
| 2024-02-02 | 2024-02-01 | 178.00 |  178.00 |  627.00 |
| 2024-02-05 | 2024-02-01 | 110.00 |  288.00 |  737.00 |
| 2024-02-11 | 2024-02-01 | 210.00 |  498.00 |  947.00 |
| 2024-02-15 | 2024-02-01 | 100.00 |  598.00 | 1047.00 |
| 2024-02-18 | 2024-02-01 | 197.00 |  795.00 | 1244.00 |
| 2024-03-03 | 2024-03-01 | 120.00 |  120.00 | 1364.00 |
| 2024-03-05 | 2024-03-01 | 305.00 |  425.00 | 1669.00 |
| 2024-03-08 | 2024-03-01 |  19.00 |  444.00 | 1688.00 |
| 2024-03-21 | 2024-03-01 | 134.00 |  578.00 | 1822.00 |
| 2024-03-25 | 2024-03-01 | 120.00 |  698.00 | 1942.00 |
| 2024-04-02 | 2024-04-01 | 335.00 |  335.00 | 2277.00 |
| 2024-04-06 | 2024-04-01 |  25.00 |  360.00 | 2302.00 |
| 2024-04-09 | 2024-04-01 |  38.00 |  398.00 | 2340.00 |
| 2024-04-12 | 2024-04-01 | 209.00 |  607.00 | 2549.00 |
| 2024-04-15 | 2024-04-01 | 390.00 |  997.00 | 2939.00 |
| 2024-04-20 | 2024-04-01 | 120.00 | 1117.00 | 3059.00 |

Invariants to test: `MAX(revenue_mtd)` per month equals that month's plain total
(449 / 795 / 698 / 1117), and the global `MAX(revenue_ytd)` equals 3059.00.

**Note the two `PARTITION BY` expressions.** They are different windows, so the engine performs
two sorts unless both are satisfiable from the same ordering. On large tables, compute YTD from
the MTD series instead (`SUM(revenue_mtd) ... WHERE is_month_end`) or accept two sorts.

### 4.6 Example 6 — resetting on an *event*, not a calendar boundary

**Problem statement.** A campaign budget is topped up arbitrarily (not on calendar boundaries).
You need "spend since the last top-up".

```sql
WITH moves(ts, campaign, spend, topup) AS (
    VALUES (TIMESTAMP '2024-05-01 09:00', 'spring',  120.00, false),
           (TIMESTAMP '2024-05-01 14:00', 'spring',   80.00, false),
           (TIMESTAMP '2024-05-02 08:00', 'spring',    0.00, true),   -- top-up: reset
           (TIMESTAMP '2024-05-02 11:00', 'spring',  150.00, false),
           (TIMESTAMP '2024-05-03 10:00', 'spring',   70.00, false),
           (TIMESTAMP '2024-05-04 09:00', 'spring',    0.00, true),   -- another reset
           (TIMESTAMP '2024-05-04 15:00', 'spring',   40.00, false)
), marked AS (
    SELECT ts, campaign, spend, topup,
           SUM(CASE WHEN topup THEN 1 ELSE 0 END)
               OVER (PARTITION BY campaign ORDER BY ts
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cycle_id
    FROM   moves
)
SELECT ts, spend, topup, cycle_id,
       SUM(spend) OVER (PARTITION BY campaign, cycle_id ORDER BY ts
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS spend_in_cycle
FROM   marked
ORDER  BY ts;
```

| ts | spend | topup | cycle_id | spend_in_cycle |
|----|------:|:-----:|---------:|---------------:|
| 2024-05-01 09:00 | 120.00 | false | 0 | 120.00 |
| 2024-05-01 14:00 |  80.00 | false | 0 | 200.00 |
| 2024-05-02 08:00 |   0.00 | true  | 1 |   0.00 |
| 2024-05-02 11:00 | 150.00 | false | 1 | 150.00 |
| 2024-05-03 10:00 |  70.00 | false | 1 | 220.00 |
| 2024-05-04 09:00 |   0.00 | true  | 2 |   0.00 |
| 2024-05-04 15:00 |  40.00 | false | 2 |  40.00 |

`cycle_id` is the running sum of a 0/1 flag; it is constant within a cycle and increments exactly
at each reset. This is the generalizable reset pattern. The top-up row itself belongs to the
**new** cycle (`cycle_id` increments before the spend is accumulated), which is the behaviour you
want for "budget available since top-up".

### 4.7 Example 7 — running `COUNT`, running `MIN`/`MAX`, running `SUM(CASE …)`

```sql
SELECT d.revenue_date,
       d.revenue,
       COUNT(*)              OVER (ORDER BY d.revenue_date ROWS UNBOUNDED PRECEDING) AS days_with_sales,
       COUNT(*)              OVER (ORDER BY d.revenue_date ROWS UNBOUNDED PRECEDING) FILTER
                              (WHERE d.revenue >= 200) AS big_days_to_date,
       MAX(d.revenue)        OVER (ORDER BY d.revenue_date ROWS UNBOUNDED PRECEDING) AS best_day_so_far,
       SUM(CASE WHEN d.revenue < 100 THEN 1 ELSE 0 END)
                             OVER (ORDER BY d.revenue_date ROWS UNBOUNDED PRECEDING) AS low_days_to_date
FROM   daily_revenue d
ORDER  BY d.revenue_date;
```

Highlighted rows (full table omitted for space — run it):

| revenue_date | revenue | days_with_sales | big_days_to_date | best_day_so_far | low_days_to_date |
|--------------|--------:|----------------:|-----------------:|----------------:|-----------------:|
| 2024-01-06 | 170.00 | 1 | 0 | 170.00 | 0 |
| 2024-01-08 | 190.00 | 2 | 0 | 190.00 | 0 |
| 2024-01-20 |  89.00 | 3 | 0 | 190.00 | 1 |
| 2024-02-11 | 210.00 | 6 | 1 | 210.00 | 1 |
| 2024-03-05 | 305.00 |10 | 2 | 305.00 | 2 |
| 2024-04-15 | 390.00 |18 | 3 | 390.00 | 3 |
| 2024-04-20 | 120.00 |19 | 3 | 390.00 | 3 |

- `days_with_sales` is a running `COUNT(*)`: it counts *rows*, not days, so it equals the row
  position. If you wanted "calendar days elapsed" you must densify the calendar first (§4.9).
- `FILTER (WHERE …)` on a window aggregate is PostgreSQL/Oracle-style; where it is unavailable,
  `SUM(CASE WHEN … THEN 1 ELSE 0 END) OVER (...)` (the `low_days_to_date` column) is the
  portable spelling. Snowflake's `COUNT_IF` and BigQuery's `COUNTIF` are the shorthand
  equivalents.
- `best_day_so_far` is a running `MAX`: a *monotonically non-decreasing* series. If it ever
  decreases, your data or your frame is wrong — a cheap data-quality assertion.

### 4.8 Example 8 — cumulative distinct counts, three ways

**Problem statement.** Cumulative count of **distinct buyers** by day.

**Naive/incorrect approach — `COUNT(DISTINCT)` in a window:**

```sql
-- ❌ ERROR: DISTINCT is not implemented for window functions        (PostgreSQL)
-- ❌ ERROR 1235: this version of MySQL doesn't yet support ...      (MySQL)
SELECT revenue_date,
       COUNT(DISTINCT user_id) OVER (ORDER BY revenue_date) AS cum_buyers   -- does not work
FROM   order_revenue
WHERE  status <> 'cancelled';
```

**Naive/incorrect approach — drop `DISTINCT`:**

```sql
-- ❌ counts ORDERS, not buyers; massively overstates growth
SELECT revenue_date, COUNT(*) OVER (ORDER BY revenue_date) AS cum_orders
FROM   order_revenue WHERE status <> 'cancelled';
```

**Correct approach A — first-occurrence flag (preferred):** count each user exactly once, on
the day of their first order.

```sql
WITH first_orders AS (
    SELECT user_id, MIN(order_date) AS first_order_date
    FROM   order_revenue
    WHERE  status <> 'cancelled'
    GROUP  BY user_id
), new_by_day AS (
    SELECT first_order_date AS revenue_date, COUNT(*) AS new_buyers
    FROM   first_orders GROUP BY first_order_date
)
SELECT d.revenue_date,
       COALESCE(n.new_buyers, 0) AS new_buyers,
       SUM(COALESCE(n.new_buyers, 0))
           OVER (ORDER BY d.revenue_date ROWS UNBOUNDED PRECEDING) AS cum_distinct_buyers
FROM   daily_revenue d
LEFT   JOIN new_by_day n ON n.revenue_date = d.revenue_date
ORDER  BY d.revenue_date;
```

| revenue_date | new_buyers | cum_distinct_buyers |
|--------------|-----------:|--------------------:|
| 2024-01-06 | 1 | 1 |
| 2024-01-08 | 1 | 2 |
| 2024-01-20 | 0 | 2 |
| 2024-02-02 | 1 | 3 |
| 2024-02-05 | 0 | 3 |
| 2024-02-11 | 0 | 3 |
| 2024-02-15 | 1 | 4 |
| 2024-02-18 | 0 | 4 |
| 2024-03-03 | 1 | 5 |
| 2024-03-05 | 1 | 6 |
| 2024-03-08 | 0 | 6 |
| 2024-03-21 | 1 | 7 |
| 2024-03-25 | 0 | 7 |
| 2024-04-02 | 1 | 8 |
| 2024-04-06 | 0 | 8 |
| 2024-04-09 | 0 | 8 |
| 2024-04-12 | 1 | 9 |
| 2024-04-15 | 0 | 9 |
| 2024-04-20 | 0 | 9 |

Nine distinct buyers ever, matching
`SELECT COUNT(DISTINCT user_id) FROM order_revenue WHERE status <> 'cancelled'` → `9`
(user 3's only order was cancelled, so user 3 never counts).

**Correct approach B — self-join distinct (only if you need the running *total* distinct set,
not the *new* distinct set):**

```sql
-- ✅ exact, but O(n²): each row re-counts all earlier rows
SELECT d1.revenue_date,
       COUNT(DISTINCT d2.user_id) AS cum_distinct_buyers_ever
FROM   (SELECT DISTINCT order_date AS revenue_date FROM order_revenue
        WHERE status <> 'cancelled')                       d1
JOIN   order_revenue                                       d2
       ON d2.order_date <= d1.revenue_date
       AND d2.status <> 'cancelled'
WHERE  d2.order_date >= DATE '2024-01-01'                  -- bound the join!
GROUP  BY d1.revenue_date
ORDER  BY d1.revenue_date;
```

This returns the same series as approach A *for this data*, because no user stops being a buyer.
On data where the set can shrink (e.g. "distinct users active in the last 30 days"), it does not —
and then you need the sliding-window form (chapter 03).

**Correct approach C — approximate sketches** when n is huge and ±1% is fine:

```sql
-- BigQuery
SELECT revenue_date,
       HLL_COUNT.EXTRACT(
           HLL_COUNT.MERGE_PARTIAL(HLL_COUNT.INIT(user_id))
               OVER (ORDER BY revenue_date)
       ) AS approx_cum_buyers
FROM   order_revenue WHERE status <> 'cancelled';

-- Snowflake / Redshift / Databricks / Oracle
SELECT revenue_date,
       APPROX_COUNT_DISTINCT(user_id) OVER (ORDER BY revenue_date) AS approx_cum_buyers
FROM   order_revenue WHERE status <> 'cancelled';
```

`APPROX_COUNT_DISTINCT` as a *window* function is supported in Snowflake, Redshift, Databricks
and Oracle; BigQuery needs the two-step `INIT`/`MERGE_PARTIAL` sketch because its
`APPROX_COUNT_DISTINCT` is aggregate-only. PostgreSQL has no built-in sketch (use
`hll`/`postgresql-hll` or `tdigest` extensions, or ship the `HLL_COUNT` pattern into a dedicated
HyperLogLog column).

### 4.9 Example 9 — densify the calendar before accumulating

**Problem statement.** The series has gaps (days with zero revenue are missing rows). A
"cumulative revenue" chart with missing dates looks broken, and a *rolling* window over the
sparse series silently measures "last 7 rows" instead of "last 7 days".

```sql
WITH calendar AS (
    SELECT generate_series(DATE '2024-03-01', DATE '2024-03-31', INTERVAL '1 day')::date AS d
)
SELECT c.d,
       COALESCE(dr.revenue, 0) AS revenue,
       SUM(COALESCE(dr.revenue, 0))
           OVER (ORDER BY c.d ROWS UNBOUNDED PRECEDING) AS revenue_mtd
FROM   calendar c
LEFT   JOIN daily_revenue dr ON dr.revenue_date = c.d
ORDER  BY c.d;
```

March excerpt:

| d | revenue | revenue_mtd |
|---|--------:|------------:|
| 2024-03-01 |   0.00 |   0.00 |
| 2024-03-02 |   0.00 |   0.00 |
| 2024-03-03 | 120.00 | 120.00 |
| 2024-03-04 |   0.00 | 120.00 |
| 2024-03-05 | 305.00 | 425.00 |
| … | … | … |
| 2024-03-08 |  19.00 | 444.00 |
| … | … | … |
| 2024-03-21 | 134.00 | 578.00 |
| … | … | … |
| 2024-03-25 | 120.00 | 698.00 |
| … | … | … |
| 2024-03-31 |   0.00 | 698.00 |

Without the calendar the March series has 5 rows and the cumulative "by row index" is
indistinguishable from "by day". Calendar densification is also how you produce a chart that
shows *zero* days, which is itself a business insight.

Snowflake, BigQuery, Databricks and SQL Server have native generators:
`GENERATOR(ROWCOUNT=>31)`, `GENERATE_DATE_ARRAY`, `sequence()`, `DATEADD` in a recursive CTE.
MySQL 8 needs a recursive CTE. See §8.

### 4.10 Example 10 — semi-additive measures done right

**Problem statement.** MRR by month. The naive "cumulative MRR" is meaningless.

Sample snapshot table:

```sql
CREATE TABLE mrr_snapshot (
    snapshot_date date    NOT NULL,
    user_id       integer NOT NULL,
    plan          text    NOT NULL,
    mrr           numeric(10,2) NOT NULL,
    PRIMARY KEY (snapshot_date, user_id)
);

INSERT INTO mrr_snapshot VALUES
  (DATE '2024-01-31', 2, 'pro',        49.00),
  (DATE '2024-02-29', 2, 'pro',        49.00),
  (DATE '2024-02-29', 4, 'pro',        49.00),
  (DATE '2024-03-31', 2, 'pro',        49.00),
  (DATE '2024-03-31', 4, 'pro',        49.00),
  (DATE '2024-03-31', 7, 'enterprise', 499.00),
  (DATE '2024-03-31', 9, 'pro',        49.00);
```

**Naive/incorrect approach — running sum of the snapshot:**

```sql
-- ❌ 49 + 98 + 646 = 793. "Q1 MRR = 793" is a meaningless number.
SELECT snapshot_date,
       SUM(mrr) OVER (ORDER BY snapshot_date ROWS UNBOUNDED PRECEDING) AS cumulative_mrr
FROM   (SELECT snapshot_date, SUM(mrr) AS mrr FROM mrr_snapshot GROUP BY 1) s;
```

**Correct approaches:**

```sql
-- (a) MRR AT each snapshot date = sum across users at that date. No accumulation over time.
SELECT snapshot_date, SUM(mrr) AS mrr_at_snapshot
FROM   mrr_snapshot
GROUP  BY snapshot_date
ORDER  BY snapshot_date;
```

| snapshot_date | mrr_at_snapshot |
|---------------|----------------:|
| 2024-01-31 |  49.00 |
| 2024-02-29 |  98.00 |
| 2024-03-31 | 646.00 |

```sql
-- (b) Decompose into MOVEMENTS: new / expansion / contraction / churn.
--     Each user's change vs. the previous snapshot, then aggregate the changes.
WITH with_prev AS (
    SELECT s.snapshot_date, s.user_id, s.mrr,
           LAG(s.mrr) OVER (PARTITION BY s.user_id ORDER BY s.snapshot_date) AS prev_mrr,
           LAG(s.snapshot_date) OVER (PARTITION BY s.user_id ORDER BY s.snapshot_date) AS prev_date
    FROM   mrr_snapshot s
), classified AS (
    SELECT snapshot_date,
           CASE WHEN prev_mrr IS NULL THEN 'new'
                WHEN mrr > prev_mrr   THEN 'expansion'
                WHEN mrr < prev_mrr   THEN 'contraction'
                ELSE 'flat' END AS movement,
           mrr - COALESCE(prev_mrr, 0) AS delta
    FROM   with_prev
)
SELECT snapshot_date, movement, SUM(delta) AS mrr_delta
FROM   classified
GROUP  BY snapshot_date, movement
ORDER  BY snapshot_date,
         CASE movement WHEN 'new' THEN 1 WHEN 'expansion' THEN 2
                       WHEN 'contraction' THEN 3 ELSE 4 END;
```

| snapshot_date | movement | mrr_delta |
|---------------|----------|----------:|
| 2024-01-31 | new |  49.00 |
| 2024-02-29 | new |  49.00 |
| 2024-03-31 | new | 548.00 |

(Churn is *absent* rows, not zero-value rows — the hardest part of MRR reporting. You detect it
with a snapshot-date spine joined to the last known state per user, which is why MRR models
always densify the user × month grid first.)

> **Senior note.** The movement decomposition is where the money is. "MRR grew 49 → 646" tells a
> CFO nothing actionable; "new 548, expansion 0, contraction 0, churn 0" tells them the growth is
> 100% acquisition and 0% land-and-expand — a very different company. Every serious subscription
> model (`dbt`'s classic `mrr_movements`, Baremetrics, ProfitWell) is built on this pattern, and
> it is a *semi-additive* pattern: you sum deltas across users and time, but you never sum levels
> across time.

### 4.11 Example 11 — running totals as a Pareto / "share of total" builder

**Problem statement.** Rank products by revenue and show cumulative share of total revenue
(ABC analysis).

```sql
WITH prod AS (
    SELECT p.product_id, p.name, SUM(r.revenue) AS revenue
    FROM   order_revenue r
    JOIN   order_items li USING (order_id)
    JOIN   products p ON p.product_id = li.product_id
    WHERE  r.status <> 'cancelled'
    GROUP  BY p.product_id, p.name
)
SELECT name, revenue,
       SUM(revenue) OVER (ORDER BY revenue DESC, product_id ROWS UNBOUNDED PRECEDING) AS revenue_cum,
       ROUND(100.0 * SUM(revenue) OVER (ORDER BY revenue DESC, product_id ROWS UNBOUNDED PRECEDING)
                   / SUM(revenue) OVER (), 1)                                         AS cum_pct
FROM   prod
ORDER  BY revenue DESC, product_id;
```

| name | revenue | revenue_cum | cum_pct |
|------|--------:|------------:|--------:|
| Rain Jacket    | 985.00 |  985.00 |  32.2 |
| Trail Runner 5 | 680.00 | 1665.00 |  54.4 |
| Trail Runner 4 | 590.00 | 2255.00 |  73.7 |
| City Sneaker   | 440.00 | 2695.00 |  88.1 |
| Water Bottle   | 250.00 | 2945.00 |  96.3 |
| Wool Socks     | 114.00 | 3059.00 | 100.0 |

The `cum_pct` column is how you assign ABC classes: A ≤ 80 %, B ≤ 95 %, C above. Add a
`CASE` on `cum_pct` and you have a segmentation in one query. Note the tiebreaker
`product_id` in the `ORDER BY` — without it the cumulative column for equally-revenue products
would be nondeterministic (chapter 01, §3.4).

### 4.12 Example 12 — the incremental-model trap

**Problem statement.** You maintain a `revenue_ytd` column in an incremental table and only
process yesterday's rows.

```sql
-- ❌ WRONG: "cumulative" restarts at yesterday, because the window only sees the new rows
SELECT revenue_date, revenue,
       SUM(revenue) OVER (ORDER BY revenue_date ROWS UNBOUNDED PRECEDING) AS revenue_ytd
FROM   daily_revenue
WHERE  revenue_date = CURRENT_DATE - 1
;

-- ❌ ALSO WRONG: adding yesterday's total to yesterday's *cumulative* value
--    (you need today's running value, and backfills break it entirely)
UPDATE revenue_ytd_tbl
SET    revenue_ytd = revenue_ytd + (SELECT SUM(revenue) FROM daily_revenue
                                    WHERE revenue_date = CURRENT_DATE - 1)
WHERE  revenue_date = CURRENT_DATE - 1;
```

Correct options:

1. **Full refresh for cumulative models.** Cheap for small tables, correct by construction. Most
   cumulative marts are full-refresh for exactly this reason.
2. **Window over history + a merge of only the last N periods.** Expensive but bounded:
   `WHERE revenue_date >= (SELECT MAX(revenue_date) - INTERVAL '90 days' FROM ...)`.
3. **Store deltas, derive cumulatives in the BI layer** using a table calculation or a
   `SUM()` with a self-join-free running-total expression the tool supports. This is the
   "correct" answer for large tables with heavy backfills: **deltas are idempotent, cumulative
   columns are not.**
4. If you must materialize cumulatives incrementally, keep a **checkpointed running total table**
   keyed by (grain, period) and rebuild only affected periods, plus a reconciliation job that
   asserts `SUM(deltas) == MAX(cumulative)`.

> **Senior note.** "Additive facts are safe to recompute; cumulatives are derived state." Choose
> derived state only when the recompute cost is genuinely prohibitive, and then own the
> reconciliation test.

## 5. Reusable template

### 5.1 The four-line running total

```sql
SELECT <grain_cols>,
       <measure>,
       SUM(<measure>) OVER (
           PARTITION BY <reset_cols or period>              -- omit for a single grand total
           ORDER BY     <time_col>, <unique_tiebreaker>      -- ← tiebreaker keeps it deterministic
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- ← explicit frame
       ) AS <measure>_running
FROM   <source_aggregated_to_grain>          -- ← aggregate FIRST, then accumulate
ORDER  BY <grain_cols>;
```

### 5.2 Reset-on-event

```sql
WITH marked AS (
    SELECT *,
           SUM(CASE WHEN <reset_condition> THEN 1 ELSE 0 END)
               OVER (PARTITION BY <entity>
                     ORDER BY <t> ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cycle_id
    FROM <events>
)
SELECT *,
       SUM(<value>) OVER (PARTITION BY <entity>, cycle_id
                          ORDER BY <t> ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
           AS value_since_reset
FROM marked;
```

### 5.3 Cumulative distinct ("new X to date")

```sql
WITH firsts AS (
    SELECT <entity_id>, MIN(<t>) AS first_t, <dimensions>
    FROM   <facts> GROUP BY <entity_id>, <dimensions>
)
SELECT d.<period> AS period,
       COUNT(f.<entity_id>) AS new_entities,
       SUM(COUNT(f.<entity_id>)) OVER (ORDER BY d.<period> ROWS UNBOUNDED PRECEDING)
           AS cumulative_distinct_entities
FROM   <calendar> d
LEFT   JOIN firsts f ON f.first_t = d.<period>
GROUP  BY d.<period>
ORDER  BY d.<period>;
```

### 5.4 Densified series + cumulative

```sql
WITH spine AS (SELECT <generate date series> AS d)          -- §8 for the engine-specific generator
SELECT s.d,
       COALESCE(f.<measure>, 0) AS <measure>,
       SUM(COALESCE(f.<measure>, 0)) OVER (ORDER BY s.d ROWS UNBOUNDED PRECEDING) AS <measure>_cum
FROM   spine s
LEFT   JOIN <fact_aggregated> f ON f.<period> = s.d
ORDER  BY s.d;
```

### 5.5 Movement decomposition (semi-additive)

```sql
WITH prev AS (
    SELECT *,
           LAG(<level_measure>) OVER (PARTITION BY <entity> ORDER BY <period>) AS prev_value
    FROM   <snapshots>
)
SELECT <period>,
       SUM(CASE WHEN prev_value IS NULL           THEN <level_measure>
                WHEN <level_measure> > prev_value THEN <level_measure> - prev_value END) AS new_or_expansion,
       SUM(CASE WHEN <level_measure> < prev_value  THEN prev_value - <level_measure> END) AS contraction,
       SUM(<level_measure>)                                                        AS <level>_at_period
FROM   prev
GROUP  BY <period>
ORDER  BY <period>;
```

### 5.6 Assertion kit (paste these into your dbt/Great Expectations tests)

| Assertion | SQL |
|-----------|-----|
| Final cumulative = total | `SELECT MAX(cum) = (SELECT SUM(x) FROM src) FROM t` |
| Cumulative is non-decreasing | `SELECT COUNT(*) = 0 FROM (SELECT cum, LAG(cum) OVER (ORDER BY d) p FROM t) x WHERE cum < p` |
| Per-group cumulative ends at group total | `SELECT COUNT(*) = 0 FROM (SELECT user_id, MAX(cum), SUM(x) FROM t GROUP BY 1) g WHERE MAX <> SUM` |
| Running `MAX` is non-decreasing | same shape as above with `MAX()` |
| MTD at month end = monthly total | `SELECT COUNT(*) = 0 FROM (SELECT month, MAX(mtd), SUM(revenue) s FROM t GROUP BY 1) x WHERE MAX <> s` |

## 6. Real-world use cases and applications

| Domain | Metric | Pattern |
|--------|--------|---------|
| **Growth** | Cumulative registered / activated / paying users | cumulative distinct via first-occurrence flag |
| **Finance** | YTD / MTD / QTD revenue, burn | calendar `PARTITION BY` |
| **Finance** | Cash balance, runway | semi-additive level; cumulative *flows* only |
| **SaaS** | MRR movement (new / expansion / contraction / churn) | `LAG` + level aggregation |
| **E-commerce** | Cumulative GMV, cumulative units, ABC/Pareto analysis | cumulative `SUM` + `cum_pct` |
| **Inventory / supply chain** | On-hand units since the last restock | reset-on-event |
| **Adtech** | Cumulative spend against a budget with mid-flight top-ups | reset-on-event |
| **Data quality** | Cumulative failing rows since an incident began | cumulative `SUM` of a flag |
| **Data quality** | Days since last successful load (streak counter) | cumulative `COUNT` + gap detection |
| **Gaming / loyalty** | XP/points balance, progress toward a tier | cumulative sum with a level lookup |
| **Reliability** | Error budget burn-down over a quarter | cumulative of negative deltas + reset |
| **Product** | Cumulative features shipped, cumulative bugs fixed | cumulative `COUNT` |
| **Analytics engineering** | Snapshot table → current state | `LAST_VALUE` / dedup, **not** running sum |

Two production stories:

1. **"The revenue chart is right at the end and wrong in the middle."** A daily revenue
   dashboard ran `SUM(revenue) OVER (ORDER BY order_ts)` over *order-level* rows but displayed
   the result as one point per day using a charting "group by day" step. Some days had four
   orders, so four intermediate points were drawn, each one *ahead* of the true daily
   cumulative by the peers that had already been added. Aggregating to daily grain first fixed
   it. The bug was invisible in the monthly rollup.
2. **"Runway changed by 40% overnight with no spend change."** The runway model computed cash as
   `SUM(cash_balance) OVER (ORDER BY month)`. Because `cash_balance` is a *level*, the cumulative
   column triple-counted the balance by month 3. The fix was `LAST_VALUE(cash_balance) OVER
   (PARTITION BY month ORDER BY as_of_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED
   FOLLOWING)`, i.e. the period-end snapshot, and then a separate cumulative sum of *net cash
   flow*.

## 7. Performance notes and anti-patterns

### 7.1 How the engine executes a running total

A cumulative aggregate is the **cheapest** window shape the engine can execute, because it needs
only a **single forward pass with an accumulator** — no frame buffers, no inverse transitions:

- **PostgreSQL**: `WindowAgg` with a simple "add to a running variable" transition, on top of a
  `Sort` (or an `Index Scan` that already provides the order). `EXPLAIN (ANALYZE, BUFFERS)` shows
  `WindowAgg` and a `Sort`; if the sort is absent you are streaming.
- **SQL Server**: `Segment` + `Sequence Project`/`Window Spool`. Watch for `Window Spool` with
  large `Actual Rows` — that is the frame being materialized.
- **Snowflake / BigQuery / Redshift**: a distributed sort (shuffle/exchange) on
  `PARTITION BY`, then a per-partition streaming accumulation. The **shuffle is the cost**.

Consequences:

- **Choose `ROWS` for semantics, not speed.** Most engines special-case the default frame
  (`RANGE UNBOUNDED PRECEDING AND CURRENT ROW`) with a fast path — PostgreSQL, for example,
  recognises it as "plain aggregate plus all peers" and keeps a single running accumulator with
  a peer-boundary check. The difference versus `ROWS UNBOUNDED PRECEDING` is usually negligible.
  Pick the frame that expresses the metric definition; add the frame explicitly so a future
  reader (or a future engine version) cannot change the meaning by accident.
- **`ROWS BETWEEN n PRECEDING AND m FOLLOWING`** needs a real frame buffer of size `n+m+1`
  (or an inverse-transition accumulator). `ROWS UNBOUNDED PRECEDING` needs **one** value.
- **`RANGE BETWEEN INTERVAL … PRECEDING`** is the most expensive common form: the executor may
  need to search backwards for the frame start per row (binary search if the order is sorted).

### 7.2 Indexing

```sql
-- Backs: SUM(x) OVER (PARTITION BY g ORDER BY t ROWS UNBOUNDED PRECEDING)
CREATE INDEX ix_fact_g_t ON fact (g, t);
-- Add covering columns so the window never touches the heap:
CREATE INDEX ix_fact_g_t_cover ON fact (g, t) INCLUDE (x);
```

If the index key order matches `(PARTITION BY …, ORDER BY …)` exactly, PostgreSQL and MySQL can
skip the sort entirely and the plan becomes a streaming `Index Scan → WindowAgg`. SQL Server
mentions the same requirement in the `OVER` clause docs ("the position of the key columns must
match the `PARTITION BY` columns followed by the `ORDER BY` columns").

For the single-partition case (`OVER (ORDER BY t)` with no `PARTITION BY`), the index must be on
`(t)` alone. Note that a global running total is **inherently serial** in most engines — it
cannot be parallelized across a shuffle, so it becomes the bottleneck on huge scans. Prefer
per-partition accumulation plus a final sum of partition ends if you need global parallelism.

### 7.3 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| Relying on the default `RANGE` frame with non-unique ordering | Peers share values; wrong intermediate points | Explicit `ROWS` frame, or aggregate to grain first |
| `SUM(x) OVER (PARTITION BY g)` called "cumulative" | It is the group total, repeated | Add `ORDER BY` + frame |
| `SUM(x) OVER (ORDER BY t)` where `t` has NULLs | NULLs sort together (first or last per engine) and become one giant peer group | Filter/`COALESCE`, and state `NULLS FIRST/LAST` |
| Cumulative sum of a *level* measure | 3×–12× inflated balances | `LAST_VALUE` per period, or cumulative *flows* |
| Cumulative `COUNT(*)` used as "days elapsed" | Counts rows, not calendar days | Densify the calendar first |
| `COUNT(DISTINCT) OVER` | Does not run | First-occurrence flag, or approximate sketch |
| Cramming 4 `PARTITION BY` variants into one query | 4 sorts/shuffles | Stage them, or reuse one window via `WINDOW` clause (§8) |
| Cumulative column in an incremental model | Wrong after any backfill | Full refresh, or store deltas |
| Filtering the source *after* the window | Sorts and accumulates rows you then discard | Filter in the CTE before the window |
| Forgetting the `ROWS` keyword and getting `RANGE` | Silent semantic drift | Always write the frame |
| Running total across a huge unpartitioned set | Serial, unparallelizable bottleneck | Partition by a coarse key, sum partition ends |

### 7.4 Scale techniques

- **Prefix sums.** If you need arbitrary range sums (`SUM` between two dates) repeatedly,
  materialize a cumulative column and answer with `cum(hi) - cum(lo - 1)`. One ordered scan beats
  a scan per query.
- **Two-level accumulation.** Running totals are associative: compute per-shard/per-partition
  running totals, then a cumulative sum of the shard *ends*, then add the offset. This is how
  Spark/Beam implement cumulative sums across partitions.
- **Pre-aggregate the grain.** A running total over 500M line items is almost never what you want;
  a running total over the daily aggregate (a few thousand rows) is.
- **Delta tables for backfills.** Keep the additive fact table append-only and idempotent; derive
  cumulative metrics in a view or a fully-refreshed small mart. Backfills then become pure
  appends.
- **Approximate distinct at scale.** A HyperLogLog sketch per day is ~2 KB; merging sketches
  cumulatively is `O(days)` rather than `O(rows)`.
- **Watch spilling.** A cumulative window over a *single* oversized partition cannot stream
  output until the sort completes; on limited memory this spills. Reducing the partition count
  is not the fix — increasing it is (more, smaller partitions can be processed in parallel).

## 8. Cross-engine differences

### 8.1 Frames and features

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Default frame with `ORDER BY` | `RANGE UNBOUNDED PRECEDING` | same | same | same | same | same |
| Default frame *without* `ORDER BY` | whole partition | whole partition | whole partition | whole partition | whole partition | whole partition |
| `ROWS` frames | ✅ | ✅ | ✅ 2012+ | ✅ | ✅ | ✅ |
| `RANGE` with **numeric** offsets | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ (numeric only) |
| `RANGE` with **interval** offsets | ✅ (`INTERVAL '1 day'`) | ✅ (`INTERVAL 1 DAY`) | ❌ | ✅ | ✅ (`INTERVAL '1 day'`) | ❌ (use `UNIX_DATE`) |
| `GROUPS` mode | ✅ 11+ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `EXCLUDE CURRENT ROW / GROUP / TIES` | ✅ 11+ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Named `WINDOW w AS (...)` clause | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `FILTER (WHERE …)` on window aggregates | ✅ 9.4+ | ❌ | ❌ | ❌ | ❌ (`COUNT_IF`) | ❌ (`COUNTIF`) |
| `COUNT(DISTINCT x) OVER (…)` | ❌ | ❌ | ❌ | ❌ | ✅ (single column) | ❌ |
| `APPROX_COUNT_DISTINCT(...) OVER (…)` | ❌ | ❌ | ❌ (uses `APPROX_COUNT_DISTINCT` scalar in 2022+) | ✅ | ✅ | ❌ (`HLL_COUNT.*`) |
| `LAST_VALUE` default frame (ANSI: up to last peer) | ✅ ANSI | ✅ ANSI | ✅ ANSI | ✅ ANSI | ⚠️ `ROWS … UNBOUNDED FOLLOWING` (non-ANSI for `FIRST_/LAST_/NTH_VALUE`) | ✅ ANSI |

### 8.2 Date-series generators (for the densify step)

| Engine | Idiom |
|--------|-------|
| PostgreSQL | `SELECT generate_series(DATE '2024-01-01', DATE '2024-03-31', INTERVAL '1 day')::date` |
| MySQL 8 | recursive CTE with `DATE_ADD(d, INTERVAL 1 DAY)` |
| SQL Server | `SELECT DATEADD(DAY, v, @start)` from `sys.all_objects`/`GENERATE_SERIES` (2022+) |
| Oracle | `SELECT DATE '2024-01-01' + LEVEL - 1 FROM dual CONNECT BY LEVEL <= 91` |
| Snowflake | `SELECT DATEADD(day, seq4(), DATE '2024-01-01') FROM TABLE(GENERATOR(ROWCOUNT => 91))` |
| BigQuery | `SELECT d FROM UNNEST(GENERATE_DATE_ARRAY('2024-01-01','2024-03-31', INTERVAL 1 DAY)) AS d` |

### 8.3 Portability cheatsheet

```sql
-- Portable YTD/MTD (works on all six)
SUM(x) OVER (PARTITION BY <period> ORDER BY <t> ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- Portable "big days so far" (FILTER is not portable)
SUM(CASE WHEN x >= 200 THEN 1 ELSE 0 END) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)

-- Portable period bucketing
date_trunc('month', t)                       -- PG, Snowflake, Redshift, Databricks
DATE_TRUNC(t, MONTH)                         -- BigQuery
DATE_TRUNC('month', t)                       -- MySQL 8, SQL Server 2022+
TRUNC(t, 'MM')                               -- Oracle
```

Other dialects to keep in mind:

- **MySQL** supports `RANGE … INTERVAL n UNIT PRECEDING` (it is one of the more permissive
  engines here), but has **no `GROUPS` and no `EXCLUDE`**, and no `FILTER`.
- **SQL Server** explicitly documents that `<unsigned value specification> PRECEDING/FOLLOWING`
  "isn't allowed for `RANGE`" — so any interval-based `RANGE` frame must be rewritten
  (`ROWS` on a densified daily grid, or a self-join on a date range).
- **Oracle** supports interval `RANGE` frames and `APPROX_COUNT_DISTINCT` as a window function;
  it has no `FILTER`, no `GROUPS`, no `EXCLUDE`.
- **Snowflake** supports `RANGE BETWEEN INTERVAL … PRECEDING` for a limited function set
  (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`, `STDDEV*`, `VARIANCE*`, `COUNT_IF`,
  `FIRST_VALUE`/`LAST_VALUE`, `ARRAY_AGG`) and **forbids** `DISTINCT` inside those frames. Its
  default frame for `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` is non-ANSI — always write the frame.
- **BigQuery** requires the `RANGE` `ORDER BY` expression to be **numeric and unique in the
  clause**; for timestamps use `UNIX_SECONDS(t)` / `UNIX_DATE(t)`.
- **DuckDB** is the most permissive (supports `GROUPS`, `EXCLUDE`, `QUALIFY`, `WINDOW`,
  interval `RANGE`) and is an excellent local test harness for the portable subset.

## 9. Exercises — Intermediate (hints only)

1. **Revenue-to-date with an explicit frame.** Produce `revenue_date`, `revenue`,
   `revenue_to_date` from `daily_revenue`. Assert the final value equals 3059.00.
   *Hint:* `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.
2. **Prove the `RANGE`/`ROWS` difference.** Build a two-day data set with three orders on day 1
   and one on day 2. Show both cumulative columns and explain why the first three rows differ.
   *Hint:* `VALUES` CTE; consider `WINDOW` clause or two window specs.
3. **Monthly restart.** Compute `revenue_mtd` that restarts each month. Verify each month's
   maximum equals that month's plain total.
   *Hint:* `PARTITION BY date_trunc('month', revenue_date)`.
4. **Per-customer cumulative and share.** For each customer's orders, cumulative revenue and
   `%` of their lifetime revenue. Assert the share reaches 100 % on each customer's last row.
   *Hint:* two windows over the same partition — one ordered+`ROWS`, one unordered.
5. **Best day so far.** Running `MAX(revenue)` by date. Then write the assertion query that finds
   any violation of monotonicity.
   *Hint:* `LAG` over the cumulative column; `WHERE cum < prev`.
6. **Cumulative distinct buyers.** Reproduce the 1/2/2/3/…/9 series in §4.8. Then compute the same
   series from `events` (distinct users with at least one `purchase` event) and check the totals
   agree.
   *Hint:* `MIN(event_time)` per user with a `purchase` filter.
7. **Densify and accumulate.** Produce a March series with a row for every calendar day (zero
   days included) and a cumulative column.
   *Hint:* `generate_series` + `LEFT JOIN` + `COALESCE`.
8. **Balance vs. flow.** Using `mrr_snapshot`, produce (a) MRR at each month end, (b) the
   *change* in MRR per month, and (c) the cumulative change from the first month. Explain in one
   sentence which of the three is a "running total".
   *Hint:* `LAG` + `GROUP BY snapshot_date`.
9. **Cumulative counts by segment.** From `events`, cumulative `page_view` count per `user_id`
   over time. *Hint:* `PARTITION BY user_id`, `ORDER BY event_time, event_id`, `ROWS UNBOUNDED
   PRECEDING`.
10. **Reset on a flag.** Take the `moves` CTE from §4.6 and generalise it: add a second campaign
    and confirm the `cycle_id` sequences independently per campaign. *Hint:* the running-sum flag
    must be `PARTITION BY campaign`.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Cumulative distinct with a *sliding* definition.** "Cumulative count of distinct users who
   were active at least once in the last 28 days," evaluated every day, exact.
   *Expected approach:* start from `COUNT(DISTINCT)` per day (a small aggregate), then a
   self-join or `RANGE BETWEEN INTERVAL '27 days' PRECEDING AND CURRENT ROW` over the *daily
   aggregate* — but note that summing daily distincts double-counts users present on two days,
   so you need the windowed distinct form:
   `COUNT(DISTINCT user_id) OVER (ORDER BY d RANGE BETWEEN INTERVAL '27 days' PRECEDING AND CURRENT ROW)`
   (PostgreSQL will reject it; where it works it is a "distinct over a value range"), or an
   event-grain technique: for each (user, day) keep only rows where the user has no activity in
   the preceding 28 days, then a cumulative `COUNT`. Discuss the memory/compute trade-off and
   when an HLL sketch merged per 28-day window is the right engineering answer.
2. **Global running total in parallel.** Explain why `SUM(x) OVER (ORDER BY t)` is a serial
   bottleneck on a distributed warehouse, and design a two-phase computation (per-partition
   cumulative + partition offsets) that produces identical results. Give the SQL for phase 2 and
   the invariants you would test.
   *Expected approach:* phase 1 per shard computes running sums and the shard total; phase 2
   computes the exclusive prefix of shard totals and adds it. Test: final value equality vs
   `SUM(x)`, and `MAX(cum) == SUM(x)`.
3. **MRR movements including churn.** Extend §4.10(b) so that a user who appears in a snapshot and
   then disappears produces `churn = -prev_mrr` in the month they disappear, and a user who
   reactivates produces `reactivation`. Then reconcile: for every month,
   `prev_level + new + expansion − contraction − churn = level`. Write the reconciliation query
   and prove it returns zero rows of violation.
   *Expected approach:* build a **user × month spine** (`CROSS JOIN` calendar) and `LEFT JOIN`
   snapshots, `COALESCE`d to `NULL` (absent) rather than zero, so `LAG` can distinguish
   "absent" from "zero MRR". This distinction is the whole exercise.
4. **Cumulative distinct at billion-row scale.** You need "cumulative distinct devices that ever
   fired a beacon" with ±1 % accuracy, updated hourly, on a table with 10^10 rows and 10^8
   distinct devices. Design the pipeline (sketch type, storage, merge strategy, backfill),
   estimate memory, and state the accuracy guarantee. Then write the incremental merge query.
   *Expected approach:* HLL++ / Theta sketches per (hour, partition key), merged cumulatively,
   stored as `BYTES`; three-tier merge (hour → day → all-time) so the hourly job is O(new
   sketches). Cite BigQuery's `HLL_COUNT.INIT/MERGE_PARTIAL/MERGE/EXTRACT` as the reference
   implementation and note the merge is associative but not exactly invertible — which is why
   you cannot "un-merge" and must rebuild on backfill.
5. **Running total with a *business calendar*.** Weekends and holidays must not advance the YTD
   series (revenue is zero, and the chart should not show a step change), and quarter-to-date must
   reset on a 4-4-5 retail calendar rather than on calendar quarters. Give the schema (a
   `dim_calendar` with `is_business_day`, `fiscal_period`), the query, and the tests.
   *Expected approach:* densify on the business-day spine, not on raw dates, and partition by
   `fiscal_period` from the dimension table. Discuss why `date_trunc` is the wrong tool and why
   the calendar dimension is the right one.
6. **Detect a broken cumulative column.** Write a single query that accepts a table with a
   materialized `cumulative` column and a granular `value` column and returns every row where
   the two disagree, including the running sum recomputed from the granular data. Handle gaps
   (missing `value` rows mean zero), duplicate keys, and rows where `value` is `NULL`.
   *Expected approach:* full-outer-join a `SUM(...) OVER (...)` recomputation against the stored
   column on the grain key; `COALESCE` the granular value to 0; and report deltas with a
   tolerance for numeric rounding (`ABS(a-b) > 0.005`). This is the "cumulative column
   observability" check that belongs in every mart's test suite.

## References

- PostgreSQL — Window Function Calls (frames, `GROUPS`, `EXCLUDE`, `RANGE` interval offsets): <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- PostgreSQL — Window Functions (list, defaults): <https://www.postgresql.org/docs/current/functions-window.html>
- PostgreSQL — Tutorial: Window Functions (running total): <https://www.postgresql.org/docs/current/tutorial-window.html>
- PostgreSQL — `generate_series`: <https://www.postgresql.org/docs/current/functions-srf.html>
- MySQL 8.0 — Window Function Frame Specification (defaults, `RANGE` with intervals): <https://dev.mysql.com/doc/refman/8.0/en/window-functions-frames.html>
- MySQL 8.0 — Window Functions overview: <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- Microsoft Learn — `OVER` Clause: "`<unsigned value specification> PRECEDING` isn't allowed for `RANGE`": <https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql>
- Oracle — Analytic Functions: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/Analytic-Functions.html>
- Oracle — `APPROX_COUNT_DISTINCT`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/APPROX_COUNT_DISTINCT.html>
- Snowflake — Window function syntax and usage (`RANGE BETWEEN` limits, non-ANSI defaults): <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — Windowed aggregations for rolling calculations: <https://docs.snowflake.com/en/user-guide/querying-time-series-data>
- Snowflake — `APPROX_COUNT_DISTINCT`: <https://docs.snowflake.com/en/sql-reference/functions/approx_count_distinct>
- Google Cloud — BigQuery window function calls (`RANGE` requires numeric `ORDER BY`): <https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls>
- Google Cloud — BigQuery HyperLogLog++ functions (`HLL_COUNT.INIT` / `MERGE` / `MERGE_PARTIAL` / `EXTRACT`): <https://cloud.google.com/bigquery/docs/reference/standard-sql/hll_functions>
- Google Cloud — Counting uniques faster with HyperLogLog++: <https://cloud.google.com/blog/products/gcp/counting-uniques-faster-in-bigquery-with-hyperloglog>
- Amazon Redshift — `QUALIFY` clause / window functions: <https://docs.aws.amazon.com/redshift/latest/dg/c_Window_functions.html>
- DuckDB — Window functions (frames, `GROUPS`, `EXCLUDE`): <https://duckdb.org/docs/stable/sql/functions/window_functions.html>
- Mode / ThoughtSpot — SQL Window Functions tutorial (the classic running total example): <https://mode.com/sql-tutorial/sql-window-functions/>
- dbt Labs — Incremental models (why cumulative columns are not incremental-safe): <https://docs.getdbt.com/docs/build/incremental-models>
- dbt — `dbt_utils` / package docs for generic data tests (`expression_is_true`, `not_null`): <https://github.com/dbt-labs/dbt-utils>

# Moving Averages (Rolling / Trailing Windows)

> A moving average is how you turn a noisy metric into something a human can read — and how you
> turn a clean metric into a subtly wrong one. The whole discipline is in three choices:
> **which frame** (`ROWS` counts rows, `RANGE` counts values), **which grain** (a 7-row window
> over sparse data is not a 7-day window), and **what to do at the edges** (the first six rows of
> a 7-day window are partial and will make your chart dip at the left). Get those three right and
> the rest — moving sums, moving stddevs, EWMA, rolling medians — is mechanical. This chapter
> covers all of it with the ShopStream sample schema, and finishes with the incremental-update
> and performance story that decides whether your mart costs $50/month or $5,000.

## TL;DR

- **Trailing** window = `ROWS BETWEEN n-1 PRECEDING AND CURRENT ROW`. **Centered** =
  `ROWS BETWEEN n/2 PRECEDING AND n/2 FOLLOWING` (a smoothing/trend filter, not forecastable in
  real time). **Leading** = look-ahead, almost always a leak.
- **`ROWS` is positional, `RANGE` is value-based.** On a *sparse* series they diverge wildly: a
  `ROWS BETWEEN 6 PRECEDING` window over data with 3-week gaps reaches back **months**, while
  `RANGE BETWEEN INTERVAL '6 days' PRECEDING` correctly reaches back 6 calendar days. For
  time-based metrics you almost always want `RANGE` **after densifying the calendar**, or `ROWS`
  on an already-dense grid.
- A moving average over *sparse rows* averages only days-with-data — that is a different metric
  ("revenue per active day"), not "average revenue per day". Decide which you are publishing.
  The densified + `LEFT JOIN` zero-filled version is the one that matches the business question.
- **Always fill the calendar.** `generate_series` (PostgreSQL), `GENERATE_DATE_ARRAY` (BigQuery),
  `GENERATOR` (Snowflake), a recursive CTE (MySQL), `CONNECT BY` (Oracle),
  `GENERATE_SERIES`/a tally table (SQL Server). Densify → `LEFT JOIN` → `COALESCE(0)` →
  `SUM/AVG … OVER (... ROWS …)`. That pipeline is the workhorse of every time-series mart.
- **Moving sum is often the better metric than moving average** for volume (revenue, orders,
  sessions): it is additive, it has no divide-by-window-size edge cases, and it rolls up. Moving
  average is better for *rates* and *level* metrics (AOV, latency, CPU %).
- **Warm-up period.** A frame that is not yet full returns a partial average. Either publish
  `NULL` until it is full (`COUNT(*) OVER w = :n`), or publish partial values and *label* them.
  Silently mixing them is the source of the "why does the first week of the chart look wrong"
  question.
- **No SQL engine has a native EMA**, because EMA is recursive. Use a recursive CTE, or the
  one-pass weighted-window trick (§4.8) which computes an excellent approximation in a single
  scan — that trick is the answer to "EMA with window functions" that most engineers never find.
- **Rolling medians (robust smoothing) are not `PERCENTILE_CONT` in most engines.** PostgreSQL
  forbids ordered-set aggregates as window functions; Snowflake allows `PERCENTILE_CONT` only
  with `PARTITION BY` and no frame; MySQL has no `PERCENTILE_CONT` at all. Plan for a redesign
  (e.g. `NTILE`-based, or a self-join over the frame), not a syntax tweak.
- At scale, the important optimizations are: **pre-aggregate to the grain you are smoothing**,
  **make the window a fixed small `n`** (the executor keeps a `n`-row ring buffer), and, for
  incremental marts, **maintain a rolling sum from a delta + the value that fell out of the
  window** rather than recomputing the whole history.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Moving / rolling / trailing average** | Mean of a fixed-size window that ends at the current row. |
| **Window size (n)** | Number of rows (`ROWS`) or span of values (`RANGE`) in the window. |
| **Centered moving average** | Window extends both backwards and forwards; used for trend extraction, not real-time. |
| **Simple moving average (SMA)** | Arithmetic mean over the window — `AVG(x) OVER (... ROWS n-1 PRECEDING)`. |
| **Weighted moving average (WMA)** | Weights declining linearly: `SUM(x * (n - distance)) / SUM(n - distance)`. |
| **Exponential moving average (EMA / EWMA)** | `EMA_t = α·x_t + (1-α)·EMA_{t-1}`, α = 2/(n+1); more weight on recent points; recursive. |
| **Moving sum** | `SUM(x) OVER (... ROWS n-1 PRECEDING)` — the "revenue in the last n periods" metric. |
| **Moving stddev / variance** | Rolling dispersion; the basis of Bollinger-style bands and z-score alerts. |
| **Warm-up period** | The first `n-1` rows, where the frame is not yet full. |
| **Edge effect** | The visual artefact at series boundaries caused by partial windows. |
| **Densification** | Inserting rows for missing periods (with zero or `NULL` values) so the grid is regular. |
| **Uneven spacing** | Data where consecutive observations are not equidistant in time — the case where `ROWS` lies. |
| **Half-life** | For EMA, the number of periods after which the weight decays by 50 %: `ln(2)/α`. |
| **Ring buffer / inverse transition** | Executor technique for a fixed-`n` frame: keep `n` values; add the new, subtract the evicted. |

### 1.2 The three window shapes

```
TRAILING (the default meaning of "moving average")
    x_{t-3}  x_{t-2}  x_{t-1}  x_t   x_{t+1}
       └──────┬───────┬───────┬────┘
              n = 4, ends at the current row          → real-time safe

CENTERED (trend filter; needs the future)
    x_{t-3}  x_{t-2}  x_{t-1}  x_t   x_{t+1}
              └──────┴───────┴──────┘
              n = 4, ±2 around the current row        → NOT real-time safe

LEADING (look-ahead; three-column-ahead leak in ML)
    x_{t-3}  x_{t-2}  x_{t-1}  x_t   x_{t+1}
       └──────┴───────┴──────┘
              n = 4, starts at t-3, ends before the current row → off-by-one bugs
```

### 1.3 `ROWS` vs `RANGE` vs `GROUPS` for a moving window

| Frame | Meaning | Use when |
|-------|---------|----------|
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | the previous 6 rows + this one | the series is dense (1 row per period) and you want "7 observations" |
| `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` | all rows within 6 days before | the series is sparse/irregular and you want "the last 7 days" |
| `RANGE BETWEEN 6 PRECEDING AND CURRENT ROW` | all rows whose ordering **value** is within 6 units | numeric buckets (e.g. price bands) |
| `GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW` | this peer group + the two before | sparse series where you want "the last 3 days *with data*" |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | cumulative | that is a running total, not a moving average (chapter 02) |

> **Senior note.** `RANGE BETWEEN <n> PRECEDING AND CURRENT ROW` (numeric offsets, no
> `INTERVAL`) is supported in PostgreSQL, Oracle, Snowflake and BigQuery; **MySQL** also supports
> numeric and `INTERVAL` offsets; **SQL Server** forbids offsets in `RANGE` entirely. If your
> code must run on SQL Server, plan to densify + `ROWS` (which is the right answer anyway for
> most metrics).

### 1.4 Which measure should you smooth?

| Measure kind | Example | Better to smooth | Because |
|--------------|---------|------------------|---------|
| Flow / volume | revenue, orders, sessions | moving **sum** | additive, rolls up, no divide-by-zero |
| Rate / ratio | AOV, conversion %, latency p50 | moving **ratio** (sum of numerator ÷ sum of denominator) | averaging ratios is a classic statistical error |
| Level | active users, WIP, open tickets | moving **average** of the level, or `MAX` at period end | level ≠ flow |
| Count of distinct entities | DAU, WAU | moving **average of the distinct count series** | you cannot average the underlying rows |
| Dispersion | latency volatility, error-rate σ | moving **stddev** | enables z-score alerting |

**Never average a ratio over rows.** `AVG(conversion_pct)` over 7 days weights a day with 3
visitors the same as a day with 30,000. Compute
`SUM(orders) OVER w / SUM(visits) OVER w` instead. This is one of the top-3 analytics bugs.

## 2. Why it matters / what problem it solves

Moving averages exist to solve **noise**. Business metrics are generated by processes with strong
day-of-week, holiday, and campaign effects:

| Problem | Example | Moving-average fix |
|---------|---------|--------------------|
| **Day-of-week seasonality** | Weekly revenue oscillation hides the trend | 7-day trailing mean removes weekday aliasing |
| **Small-number noise** | A new product with 3 orders/day swings ±300 % | Smooth over 7–28 days |
| **Weekend/holiday spikes** | Black Friday dwarfs November | Smooth, then compare to the smoothed baseline |
| **Alerting false positives** | "Conversion dropped 2 pp!" on a 40-visit day | Alert on smoothed value vs. smoothed baseline |
| **Trend vs. level confusion** | "Revenue is falling" (it is a 30-day month effect) | Compare 7-day MA to 28-day MA (crossover detection) |
| **Capacity planning** | Daily traffic bursts understate needed capacity | 7-day p95 of a moving average, not the raw p95 |
| **Anomaly detection** | Outliers in sensor/latency series | z-score against moving mean + moving σ |
| **Forecasting baseline** | Need a naive benchmark | 7-day MA of the same weekday is a strong seasonal-naive baseline |

Concrete cost of getting it wrong:

- A "7-day moving average" published on a sparse event series (only days with events) reported a
  **smoothed value of 212.50** where the true 7-calendar-day average was **85.00** — a 2.5×
  overstatement that made a channel look 4× better than it was (§4.3).
- Averaging per-day conversion **percentages** instead of recomputing from numerators and
  denominators misstated a funnel conversion by 30 % for a client with highly variable traffic.
- Publishing a partial 7-day window on day 3 produced a chart that "showed a dip at the start",
  which a stakeholder read as a launch problem.

## 3. The pattern / technique explained

### 3.1 The canonical trailing moving average

```sql
SELECT d,
       x,
       ROUND(AVG(x) OVER (ORDER BY d
                          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS ma7
FROM   dense_daily_series
ORDER  BY d;
```

Read it as: *"for each day, take the average of this day and the six before it."* `n` = 7, so
the offset is `n - 1 = 6`.

### 3.2 The four-step time-series pipeline (memorize this)

```
1. AGGREGATE   the fact table to one row per period per entity
                 GROUP BY period, entity
2. DENSIFY     build a complete period × entity grid and LEFT JOIN the aggregate
                 COALESCE(x, 0) for flows, NULL for levels that genuinely have no value
3. WINDOW      apply the frame with an explicit ROWS/RANGE and an explicit PARTITION BY
4. GUARD       mask or flag the warm-up rows (COUNT(*) OVER w < n)
```

Skipping step 2 is the #1 source of wrong moving averages. Skipping step 4 is the #1 source of
"the start of the chart looks broken" tickets.

### 3.3 Choosing the window size

| Window | Removes | Typical use |
|--------|---------|-------------|
| 3 | almost nothing | last-resort smoothing of a very stable series |
| 7 | day-of-week | revenue, DAU, conversion, latency dashboards |
| 14 | weekly + noise | growth dashboards; halves the reporting lag of a 28-day window |
| 28 | weekly + lunar + pay-cycle effects | subscription, marketplace, HR metrics |
| 30 / 90 | monthly artefacts | finance, capacity planning (note: 30 ≠ "a month") |
| 365 | annual seasonality | year-over-year baseline (rarely as a "moving average") |

Rules of thumb: **match the season length** (7 for daily data, 4 for weekly data, 12 for monthly
data), and **keep the window away from your reporting cadence** (do not use a 30-day window for a
monthly report; the window and the period alias into each other).

### 3.4 Moving average vs. moving ratio

```sql
-- ❌ WRONG for a rate metric: a 3-visit day gets the same weight as a 30,000-visit day
AVG(conversion_rate) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- ✅ CORRECT: weight by traffic, i.e. sum the numerator and the denominator
SUM(orders) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::numeric
  / NULLIF(SUM(visits) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 0)
```

The `NULLIF` matters: a window with zero traffic must produce `NULL`, not a division error or a
silent `0`.

### 3.5 Moving sum, moving stddev, and the z-score alert

```sql
SELECT d,
       x,
       SUM(x) OVER w                                    AS sum7,
       ROUND(AVG(x) OVER w, 2)                          AS ma7,
       ROUND(STDDEV_SAMP(x) OVER w, 3)                  AS sd7,
       ROUND((x - AVG(x) OVER w) / NULLIF(STDDEV_SAMP(x) OVER w, 0), 2) AS z7
FROM   series
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

`z7` is the number of standard deviations the current point sits above the trailing 7-period mean.
Alert when `ABS(z7) > 3` — but note the trailing mean includes the very point you are testing,
which **damps the z-score**. For real anomaly detection:

- exclude the current row (`ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING`), and
- prefer a **robust** center and scale (median and MAD) over mean and σ, because one big spike
  inflates both mean and σ and hides itself.

### 3.6 The warm-up problem

```sql
SELECT d, x,
       COUNT(*) OVER w                                   AS window_rows,
       CASE WHEN COUNT(*) OVER w = 7 THEN AVG(x) OVER w END AS ma7_strict,
       AVG(x) OVER w                                     AS ma7_partial
FROM   series
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

Publish `ma7_strict` when the chart must not be read in the first 6 periods, and `ma7_partial`
when you need a value everywhere. Never publish both under the same name. A useful trick for
charting is to keep `ma7_partial` but add `window_rows` so the BI tool can render the first 6
points in a lighter shade.

### 3.7 Incremental moving averages

A fixed-`n` trailing sum can be maintained incrementally without touching history:

```
rolling_sum(t) = rolling_sum(t-1) + x(t) - x(t-n)
```

This is exactly what the engine's ring-buffer ("inverse transition") does within one query. In an
**incremental mart**, you can do the same across runs *if* you persist `rolling_sum(t-1)` and you
still have `x(t-n)` available. If either is missing — or if a backfill rewrote an old `x` — you
must recompute the affected range. The safe default is **recompute the last `n` periods**; the
common bug is recomputing only yesterday.

## 4. Step-by-step worked example (with sample data and expected output tables)

### 4.1 The dense input

Sample: March 2024, one row per calendar day, built from the shared schema (chapter 01):

```sql
CREATE VIEW march_daily AS
WITH calendar AS (
    SELECT generate_series(DATE '2024-03-01', DATE '2024-03-31', INTERVAL '1 day')::date AS d
)
SELECT c.d,
       COALESCE(dr.revenue, 0)::numeric(12,2) AS revenue
FROM   calendar c
LEFT   JOIN daily_revenue dr ON dr.revenue_date = c.d;

SELECT d, revenue FROM march_daily ORDER BY d;
```

| d | revenue |  | d | revenue |
|---|--------:|---|-----------|--------:|
| 2024-03-01 | 0.00 | | 2024-03-17 | 0.00 |
| 2024-03-02 | 0.00 | | 2024-03-18 | 0.00 |
| 2024-03-03 | 120.00 | | 2024-03-19 | 0.00 |
| 2024-03-04 | 0.00 | | 2024-03-20 | 0.00 |
| 2024-03-05 | 305.00 | | 2024-03-21 | 134.00 |
| 2024-03-06 | 0.00 | | 2024-03-22 | 0.00 |
| 2024-03-07 | 0.00 | | 2024-03-23 | 0.00 |
| 2024-03-08 | 19.00 | | 2024-03-24 | 0.00 |
| 2024-03-09 | 0.00 | | 2024-03-25 | 120.00 |
| 2024-03-10 | 0.00 | | 2024-03-26 | 0.00 |
| 2024-03-11 | 0.00 | | 2024-03-27 | 0.00 |
| 2024-03-12 | 0.00 | | 2024-03-28 | 0.00 |
| 2024-03-13 | 0.00 | | 2024-03-29 | 0.00 |
| 2024-03-14 | 0.00 | | 2024-03-30 | 0.00 |
| 2024-03-15 | 0.00 | | 2024-03-31 | 0.00 |
| 2024-03-16 | 0.00 | | | |

### 4.2 Example 1 — 7-day trailing moving average on the dense grid

```sql
SELECT d,
       revenue,
       COUNT(*)              OVER w  AS window_rows,
       SUM(revenue)          OVER w  AS revenue_7d,
       ROUND(AVG(revenue)    OVER w, 2) AS ma7,
       ROUND(AVG(revenue) OVER (ORDER BY d ROWS BETWEEN 27 PRECEDING AND CURRENT ROW), 2) AS ma28
FROM   march_daily
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

| d | revenue | window_rows | revenue_7d | ma7 | ma28 |
|---|--------:|------------:|-----------:|----:|-----:|
| 2024-03-01 |   0.00 | 1 |   0.00 |   0.00 |   0.00 |
| 2024-03-02 |   0.00 | 2 |   0.00 |   0.00 |   0.00 |
| 2024-03-03 | 120.00 | 3 | 120.00 |  40.00 |  40.00 |
| 2024-03-04 |   0.00 | 4 | 120.00 |  30.00 |  30.00 |
| 2024-03-05 | 305.00 | 5 | 425.00 |  85.00 |  85.00 |
| 2024-03-06 |   0.00 | 6 | 425.00 |  70.83 |  70.83 |
| 2024-03-07 |   0.00 | 7 | 425.00 |  60.71 |  60.71 |
| 2024-03-08 |  19.00 | 7 | 444.00 |  63.43 |  55.50 |
| 2024-03-09 |   0.00 | 7 | 444.00 |  63.43 |  49.33 |
| 2024-03-10 |   0.00 | 7 | 324.00 |  46.29 |  44.40 |
| 2024-03-11 |   0.00 | 7 | 324.00 |  46.29 |  40.36 |
| 2024-03-12 |   0.00 | 7 |  19.00 |   2.71 |  37.00 |
| 2024-03-13 |   0.00 | 7 |  19.00 |   2.71 |  34.15 |
| 2024-03-14 |   0.00 | 7 |  19.00 |   2.71 |  31.71 |
| 2024-03-15 |   0.00 | 7 |   0.00 |   0.00 |  29.60 |
| 2024-03-16 |   0.00 | 7 |   0.00 |   0.00 |  27.75 |
| 2024-03-17 |   0.00 | 7 |   0.00 |   0.00 |  26.12 |
| 2024-03-18 |   0.00 | 7 |   0.00 |   0.00 |  24.67 |
| 2024-03-19 |   0.00 | 7 |   0.00 |   0.00 |  23.37 |
| 2024-03-20 |   0.00 | 7 |   0.00 |   0.00 |  22.20 |
| 2024-03-21 | 134.00 | 7 | 134.00 |  19.14 |  27.52 |
| 2024-03-22 |   0.00 | 7 | 134.00 |  19.14 |  26.27 |
| 2024-03-23 |   0.00 | 7 | 134.00 |  19.14 |  25.13 |
| 2024-03-24 |   0.00 | 7 | 134.00 |  19.14 |  24.08 |
| 2024-03-25 | 120.00 | 7 | 254.00 |  36.29 |  27.92 |
| 2024-03-26 |   0.00 | 7 | 254.00 |  36.29 |  26.85 |
| 2024-03-27 |   0.00 | 7 | 254.00 |  36.29 |  25.85 |
| 2024-03-28 |   0.00 | 7 | 120.00 |  17.14 |  24.93 |
| 2024-03-29 |   0.00 | 7 | 120.00 |  17.14 |  24.93 |
| 2024-03-30 |   0.00 | 7 | 120.00 |  17.14 |  24.93 |
| 2024-03-31 |   0.00 | 7 | 120.00 |  17.14 |  20.64 |

Observations that matter:

- `window_rows` is `< 7` for the first six days — that is the warm-up. `ma7` on row 3 (`40.00`)
  is a 3-day average, not a 7-day average. Mask it if the chart must be honest.
- `revenue_7d` is the same computation as `ma7` × `window_rows`; it is the **moving sum**, and it
  is the metric you want for volume ("revenue in the trailing 7 days"). It is additive and
  back-testable in a way the average is not.
- `ma28` equals the cumulative mean for the first 27 rows (the 28-row frame still contains the
  start of the series), then becomes a true rolling mean of 698/28 ≈ 24.93, and finally drops to
  20.64 on 2024-03-31 because the 120.00 from 2024-03-03 falls out of the window. That last drop
  is the **edge effect**: the series did not change, the window did. It is exactly the artefact
  you must not put in front of an executive without a note.
- `ma28` is much flatter than `ma7`; it absorbs the two March spikes into a plain-vanilla
  ≈ 20–28/day baseline. That flatness is the point (baseline) and the problem (it lags a real
  trend change by ~2 weeks).
- The `ma7` series has exactly the granularity an exec chart wants: it reacts to the March 5 spike
  for a week and then returns to zero. The `revenue` column does not.

Cross-check: `SELECT SUM(revenue) FROM march_daily` → `698.00`, which equals the March total from
chapter 02. A moving sum at the last row (`120.00`) is the trailing-week total, not the month
total — do not confuse them.

### 4.3 Example 2 — the sparse-data trap (`ROWS` vs `RANGE` on gaps)

**Problem statement.** You did **not** densify the calendar. You ran the "7-day" moving average
straight on `daily_revenue` for March.

```sql
-- ❌ "7-day moving average" on a sparse series — this is NOT a 7-day average
SELECT revenue_date, revenue,
       COUNT(*) OVER w              AS rows_in_window,
       ROUND(AVG(revenue) OVER w, 2) AS bogus_ma7
FROM   daily_revenue
WHERE  revenue_date >= DATE '2024-03-01' AND revenue_date < DATE '2024-04-01'
WINDOW w AS (ORDER BY revenue_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY revenue_date;
```

| revenue_date | revenue | rows_in_window | bogus_ma7 |
|--------------|--------:|---------------:|----------:|
| 2024-03-03 | 120.00 | 1 | 120.00 |
| 2024-03-05 | 305.00 | 2 | 212.50 |
| 2024-03-08 |  19.00 | 3 | 148.00 |
| 2024-03-21 | 134.00 | 4 | 144.50 |
| 2024-03-25 | 120.00 | 5 | 139.60 |

The 2024-03-21 row's window reaches back to **2024-03-03 — eighteen days** — and calls itself a
7-day average. The 2024-03-25 row averages five points spanning 22 days. Compare with the correct
densified values from §4.2: `40.00`, `85.00`, `63.43`, `19.14`, `36.29`.

**Correct approach — value-based frame (`RANGE`) with an interval:**

```sql
SELECT revenue_date, revenue,
       ROUND(AVG(revenue) OVER (ORDER BY revenue_date
                                RANGE BETWEEN INTERVAL '6 days' PRECEDING
                                          AND CURRENT ROW), 2) AS avg_per_active_day_7d,
       COUNT(*) OVER (ORDER BY revenue_date
                      RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS active_days,
       COUNT(*) OVER (ORDER BY revenue_date
                      RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS window_days
FROM   daily_revenue
WHERE  revenue_date >= DATE '2024-03-01' AND revenue_date < DATE '2024-04-01'
ORDER  BY revenue_date;
```

| revenue_date | revenue | avg_per_active_day_7d | active_days |
|--------------|--------:|----------------------:|------------:|
| 2024-03-03 | 120.00 | 120.00 | 1 |
| 2024-03-05 | 305.00 | 212.50 | 2 |
| 2024-03-08 |  19.00 | 148.00 | 3 |
| 2024-03-21 | 134.00 | 134.00 | 1 |
| 2024-03-25 | 120.00 | 127.00 | 2 |

**Now you can state the metric precisely.** Three different, all-defensible numbers exist for
2024-03-08:

| Metric | Value | Meaning |
|--------|------:|---------|
| Densified `ROWS`/`RANGE` 7-day mean | **63.43** | average revenue **per calendar day** over the last 7 days (zeros included) |
| Sparse `RANGE` 7-day mean | **148.00** | average revenue **per active day** over the last 7 days |
| Sparse `ROWS` 7-row mean | **148.00** here, but **144.50** on 03-21 | average per active day over the last 7 *rows with data* — window length in days varies |

**Reasoning.** These are not three implementations of one metric; they are **three different
metrics**. The densified one answers "how much revenue per day, on average, did we make this
week?" The sparse ones answer "when we make revenue, how much is it?" The first belongs on a
revenue trend chart; the second belongs in an AOV-like analysis. Naming them the same thing is
how two teams end up with different numbers for "the same" KPI.

> **Decision rule.** If the *denominator* of your metric is "days" or "time", densify and use
> `ROWS`/`RANGE` on the dense grid. If the denominator is "events that happened", use the sparse
> series and say so in the metric name.

### 4.4 Example 3 — centered moving average for trend extraction

**Problem statement.** Strip weekly seasonality out of a series to show the underlying trend
(offline analysis, not real-time).

```sql
SELECT d, revenue,
       ROUND(AVG(revenue) OVER (ORDER BY d ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING), 2) AS cma3,
       ROUND(AVG(revenue) OVER (ORDER BY d ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING), 2) AS cma7
FROM   march_daily
ORDER  BY d;
```

March excerpt:

| d | revenue | cma3 | cma7 |
|---|--------:|-----:|-----:|
| 2024-03-01 |   0.00 |   0.00 |  30.00 |
| 2024-03-02 |   0.00 |  40.00 |  85.00 |
| 2024-03-03 | 120.00 |  40.00 |  70.83 |
| 2024-03-04 |   0.00 | 141.67 |  60.71 |
| 2024-03-05 | 305.00 | 101.67 |  63.43 |
| 2024-03-06 |   0.00 | 101.67 |  63.43 |
| 2024-03-07 |   0.00 |   6.33 |  46.29 |
| 2024-03-08 |  19.00 |   6.33 |  46.29 |
| 2024-03-09 |   0.00 |   6.33 |   2.71 |
| 2024-03-10 |   0.00 |   0.00 |   0.00 |
| … | … | … | … |
| 2024-03-20 |   0.00 |  44.67 |  19.14 |
| 2024-03-21 | 134.00 |  44.67 |  19.14 |
| 2024-03-22 |   0.00 |  44.67 |  36.29 |
| 2024-03-23 |   0.00 |   0.00 |  36.29 |
| 2024-03-24 |   0.00 |  40.00 |  36.29 |
| 2024-03-25 | 120.00 |  40.00 |  36.29 |
| 2024-03-26 |   0.00 |  40.00 |  17.14 |
| 2024-03-27 |   0.00 |   0.00 |  17.14 |
| 2024-03-28 |   0.00 |   0.00 |  17.14 |
| 2024-03-29 |   0.00 |   0.00 |   0.00 |
| 2024-03-30 |   0.00 |   0.00 |   0.00 |
| 2024-03-31 |   0.00 |   0.00 |   0.00 |

Note how `cma3` "sees" the March 21 spike three days early (44.67 on 03-20, because the window
reaches forward to 03-21) — that is a look-ahead. `cma3` at 03-01 averages only 2 rows and `cma7`
at 03-01 averages only 4; those partial windows are the **edge effect**.

Centered windows produce a *smoother trend line* but require future rows: the value at
2024-03-25 uses 2024-03-28, so the series cannot be finalised until four days later. That is fine
for a retrospective report and wrong for a "latest number" tile. Published centered moving
averages conventionally drop the first and last `n/2` points precisely because the boundary values
are built from fewer observations.

### 4.5 Example 4 — 7-day moving average *and* 7-day moving sum, plus a rate

**Problem statement.** Build the standard growth dashboard row set: volume, average, and a
properly weighted rate.

```sql
WITH daily AS (
    SELECT o.order_date::date                                   AS d,
           COUNT(DISTINCT o.order_id)                           AS orders,
           COUNT(DISTINCT o.user_id)                            AS buyers,
           SUM(o.revenue)                                       AS revenue
    FROM   order_revenue o
    WHERE  o.status <> 'cancelled'
    GROUP  BY o.order_date
), dense AS (
    SELECT c.d, COALESCE(daily.orders, 0) AS orders,
                 COALESCE(daily.buyers, 0) AS buyers,
                 COALESCE(daily.revenue, 0) AS revenue
    FROM   (SELECT generate_series(DATE '2024-01-01', DATE '2024-04-30',
                                   INTERVAL '1 day')::date AS d) c
    LEFT   JOIN daily ON daily.d = c.d
)
SELECT d,
       revenue                                            AS revenue_today,
       SUM(revenue) OVER w                                AS revenue_7d,
       ROUND(AVG(revenue) OVER w, 2)                      AS revenue_7d_avg,
       SUM(orders)  OVER w                                AS orders_7d,
       ROUND(SUM(revenue) OVER w / NULLIF(SUM(orders) OVER w, 0), 2) AS aov_7d
FROM   dense
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

Key excerpt (Jan–Feb, where the data actually is):

| d | revenue_today | revenue_7d | revenue_7d_avg | orders_7d | aov_7d |
|---|--------------:|-----------:|---------------:|----------:|-------:|
| 2024-01-06 | 170.00 | 170.00 | 28.33 | 1 | 170.00 |
| 2024-01-08 | 190.00 | 360.00 | 51.43 | 2 | 180.00 |
| 2024-01-12 |   0.00 | 360.00 | 51.43 | 2 | 180.00 |
| 2024-01-20 |  89.00 |  89.00 | 12.71 | 1 |  89.00 |
| 2024-01-26 |   0.00 |  89.00 | 12.71 | 1 |  89.00 |
| 2024-01-27 |   0.00 |   0.00 |  0.00 | 0 | `NULL` |
| 2024-02-02 | 178.00 | 178.00 | 25.43 | 1 | 178.00 |
| 2024-02-05 | 110.00 | 288.00 | 41.14 | 2 | 144.00 |
| 2024-02-11 | 210.00 | 320.00 | 45.71 | 2 | 160.00 |
| 2024-02-18 | 197.00 | 297.00 | 42.43 | 2 | 148.50 |

Four things to notice:

1. **`aov_7d` is a weighted rate**, not an average of daily AOVs. On 2024-02-05 the trailing 7 days
   hold two orders (178 + 110 = 288), so `aov_7d = 144.00`. Averaging the two daily AOVs
   ((178 + 110)/2 = 144.00) coincides here only because the two days have equal order counts;
   the moment the counts differ, the two methods diverge. On 2024-02-11 the window holds orders of
   110 and 210 → pooled 160.00, whereas the naive mean of daily AOVs is also 160.00 by accident;
   construct a day with two orders and a day with one to see them separate.
2. **2024-01-26 still shows 89.00** in `revenue_7d` even though that day had no revenue — the
   trailing week still contains 2024-01-20. This is the behaviour that makes a moving sum feel
   "sticky", and it is correct. It only goes to zero on 2024-01-27, when 2024-01-20 drops out.
3. **`NULLIF` produces `NULL`, not 0 or an error**, when `orders_7d = 0` — see 2024-01-27. In a
   chart, `NULL` is a gap (correct: there was no data); `0` would imply "orders were free", which
   is worse.
4. **At 2024-01-06 the window contains only 6 rows**, because the dense grid starts on
   2024-01-01. `revenue_7d_avg` is therefore 170/6 = 28.33, not 170/7 = 24.29. Another warm-up /
   boundary case, this time caused by the grid's start rather than by the series' start. If your
   grid must align exactly, start it `n - 1` days before the first reportable date.

`revenue_7d_avg` is computed over a **zero-filled** grid, so it drops to 0 in the quiet weeks.
That is truthful for "revenue per calendar day" and would look catastrophic on a chart — which is
why revenue dashboards usually plot the moving **sum** (`revenue_7d`) instead, and reserve the
average for metrics like DAU/MAU where the level is naturally non-zero.

### 4.6 Example 5 — moving average per entity (partitioned)

**Problem statement.** Per-channel revenue, smoothed — the classic multi-line dashboard.

```sql
WITH daily AS (
    SELECT u.marketing_channel AS channel,
           o.order_date::date  AS d,
           SUM(o.revenue)      AS revenue
    FROM   order_revenue o JOIN users u USING (user_id)
    WHERE  o.status <> 'cancelled'
    GROUP  BY u.marketing_channel, o.order_date
), grid AS (
    SELECT ch.channel, c.d
    FROM   (SELECT DISTINCT marketing_channel AS channel FROM users) ch
    CROSS  JOIN (SELECT generate_series(DATE '2024-01-01', DATE '2024-04-30',
                                        INTERVAL '1 day')::date AS d) c
)
SELECT g.channel, g.d,
       COALESCE(daily.revenue, 0) AS revenue,
       SUM(COALESCE(daily.revenue, 0)) OVER (PARTITION BY g.channel
                                             ORDER BY g.d
                                             ROWS BETWEEN 27 PRECEDING AND CURRENT ROW) AS revenue_28d,
       ROUND(AVG(COALESCE(daily.revenue, 0)) OVER (PARTITION BY g.channel
                                                   ORDER BY g.d
                                                   ROWS BETWEEN 27 PRECEDING
                                                             AND CURRENT ROW), 2) AS ma28
FROM   grid g
LEFT   JOIN daily ON daily.channel = g.channel AND daily.d = g.d
ORDER  BY g.channel, g.d;
```

Channel trailing-28-day values at 2024-04-30 (the window is 2024-04-03 … 2024-04-30):

| channel | orders in window | revenue_28d | ma28 |
|---------|------------------|------------:|-----:|
| paid_search | 04-15 (user 7, 390.00) + 04-20 (user 2, 120.00) | 510.00 | 18.21 |
| paid_social | 04-09 (user 5,  38.00) + 04-12 (user 10, 209.00) | 247.00 |  8.82 |
| organic     | 04-06 (user 1,  25.00) |  25.00 |  0.89 |
| referral    | — (user 8's only order was 2024-03-21, outside the window) |   0.00 |  0.00 |

Note that `organic` *looks* like the largest channel by April total (360.00 from chapter 01,
thanks to user 9's 2024-04-02 order) but the trailing **28-day** window at 2024-04-30 starts on
2024-04-03 and therefore excludes it. That is not a bug — it is the definition of the metric, and
it is exactly the kind of thing that makes two dashboards disagree. Always publish the window
boundary alongside the number.

The structural points are what matter:

- `PARTITION BY g.channel` restarts the window per channel; the window **never** crosses channels.
- The **`CROSS JOIN` grid** is how you get a row for every (channel, day) — including days with
  zero revenue — so the moving average is over calendar days, not over rows.
- Two window functions with the **same** `PARTITION BY`/`ORDER BY`/frame share one sort. Adding a
  third (e.g. `max`) is nearly free; a *different* partition key costs another sort.

### 4.7 Example 6 — moving stddev, bands, and alerting

**Problem statement.** Build Bollinger-style bands and flag anomalies.

```sql
SELECT d, revenue,
       ROUND(AVG(revenue)   OVER w, 2) AS ma7,
       ROUND(STDDEV_SAMP(revenue) OVER w, 2) AS sd7,
       ROUND(AVG(revenue) OVER w + 2 * STDDEV_SAMP(revenue) OVER w, 2) AS upper2,
       ROUND(AVG(revenue) OVER w - 2 * STDDEV_SAMP(revenue) OVER w, 2) AS lower2,
       (revenue > AVG(revenue) OVER w + 2 * STDDEV_SAMP(revenue) OVER w) AS is_spike
FROM   march_daily
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

Key rows:

| d | revenue | ma7 | sd7 | upper2 | lower2 | is_spike |
|---|--------:|----:|----:|-------:|-------:|:--------:|
| 2024-03-05 | 305.00 |  85.00 | 133.51 | 352.02 | -182.02 | false |
| 2024-03-08 |  19.00 |  63.43 | 115.20 | 293.83 | -166.97 | false |
| 2024-03-21 | 134.00 |  19.14 |  50.65 | 120.44 |  -82.16 | **true** |
| 2024-03-25 | 120.00 |  36.29 |  62.10 | 160.49 |  -87.91 | false |

(Note how wide the bands are: with a 7-row window dominated by zeros, `sd7` is 1.5–2× the mean,
so the ±2σ band is enormous on the low side and the only point that fires is the 134.00 on
2024-03-21, whose preceding six days were all zero.)

This example is deliberately diagnostic: **`is_spike` fires on 2024-03-21 and not on
2024-03-05**, even though 305 is much larger than 134 in absolute terms. The reason is that the
preceding window for 03-05 was itself inflated by the 305 you are testing: the trailing mean
includes the point under test, which damps the z-score. Two fixes, both shown here:

```sql
-- (a) EXCLUDE the current row from the baseline
SELECT d, revenue,
       ROUND(AVG(revenue)   OVER (ORDER BY d ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING), 2) AS ma7_prior,
       ROUND(STDDEV_SAMP(revenue) OVER (ORDER BY d
                                        ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING), 2) AS sd7_prior
FROM   march_daily ORDER BY d;

-- (b) ROBUST statistics: median and MAD instead of mean and stddev
--     (works on PostgreSQL, Snowflake, BigQuery, SQL Server, Oracle — NOT MySQL)
SELECT d, revenue,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue)
           OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS median7
FROM   march_daily ORDER BY d;   -- ❌ ERROR in PostgreSQL: ordered-set aggregates
                                 --    cannot be used as window functions
```

For engines that reject (b) — PostgreSQL, MySQL, Snowflake (frame not allowed), SQL Server (frame
not allowed) — the portable robust alternatives are:

```sql
-- Robust center: the median of the window, computed with a self-join
SELECT a.d, a.revenue,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY b.revenue) AS median7
FROM   march_daily a
JOIN   march_daily b ON b.d BETWEEN a.d - INTERVAL '6 days' AND a.d
GROUP  BY a.d, a.revenue
ORDER  BY a.d;
```

This is O(n·k) rather than O(n) — acceptable for a daily series (31 × 7 rows), unacceptable for a
million-row sensor series. At that scale, use a t-digest/KLL sketch column, or a bucketed
approximation.

### 4.8 Example 7 — EMA, two ways

**Problem statement.** Compute α = 0.25 EMA (equivalent to an `n = 7` SMA in responsiveness:
α = 2/(n+1) = 0.25).

**Way 1 — recursive CTE (exact, but inherently serial).**

```sql
WITH RECURSIVE base AS (
    SELECT revenue_date AS d, revenue,
           ROW_NUMBER() OVER (ORDER BY revenue_date) AS rn
    FROM   daily_revenue
    WHERE  revenue_date < DATE '2024-04-01'
), walk AS (
    SELECT d, revenue, rn, revenue::numeric AS ema
    FROM   base WHERE rn = 1
    UNION ALL
    SELECT b.d, b.revenue, b.rn,
           ROUND(0.25 * b.revenue + 0.75 * w.ema, 4)
    FROM   walk w JOIN base b ON b.rn = w.rn + 1
)
SELECT d, revenue, ROUND(ema, 2) AS ema_alpha_025 FROM walk ORDER BY d;
```

March rows:

| d | revenue | ema_alpha_025 |
|---|--------:|--------------:|
| 2024-03-03 | 120.00 | 120.00 |
| 2024-03-05 | 305.00 | 166.25 |
| 2024-03-08 |  19.00 | 129.44 |
| 2024-03-21 | 134.00 | 130.58 |
| 2024-03-25 | 120.00 | 127.93 |

Compare with the cumulative mean for the same rows (`139.60` at the end, having barely moved) and
with the sparse 7-row SMA (`120.00`, `212.50`, `148.00`, `144.50`, `139.60`). The EMA **reacts
faster** than the cumulative mean and **smoother** than the equal-weight SMA: it jumps to 166.25
on the 305 spike (the SMA says 212.50) and settles at 127.93 instead of crashing to 139.60.

**Way 2 — one-pass weighted window (the trick).** EMA is a weighted sum with exponentially
decaying weights. Because the factor `w^(rn_current)` is common to every term in the frame, it
cancels in a ratio, so a **fixed-size `ROWS` window** can compute it in one scan:

```sql
WITH base AS (
    SELECT revenue_date AS d, revenue,
           ROW_NUMBER() OVER (ORDER BY revenue_date) AS rn
    FROM   daily_revenue WHERE revenue_date < DATE '2024-04-01'
), w AS (
    SELECT d, revenue, rn,
           POWER(0.75, -rn) AS raw_w          -- w^-rn  ; the current row's w^rn cancels out
    FROM   base
)
SELECT d, revenue,
       ROUND( SUM(revenue * raw_w) OVER (ORDER BY d ROWS BETWEEN 24 PRECEDING AND CURRENT ROW)
            / SUM(raw_w)          OVER (ORDER BY d ROWS BETWEEN 24 PRECEDING AND CURRENT ROW), 2)
           AS ema_approx
FROM   w
ORDER  BY d;
```

Two caveats you must document if you ship this:

1. **It is a *normalised* EMA**, i.e. it divides by the sum of the retained weights. That makes it
   identical to the true EMA except in the first few rows (where the true EMA is seeded with
   `x_1`) and it never carries an unbounded "seed" term. For dashboards this is a feature: no
   arbitrary initial value.
2. **`POWER(0.75, -rn)` overflows** for large `rn`. Subtract a per-partition offset
   (`rn - MAX(rn) OVER (PARTITION BY entity)`) so exponents stay near zero, or use
   `POWER(0.75, MAX(rn) OVER (PARTITION BY ...) - rn)` (scale-invariant because it cancels anyway).

Truncating at 24 periods with `w = 0.75` discards weight `0.75^25 ≈ 0.00075` — for a *normalised*
EMA the remaining tail is redistributed, so the error is well under a basis point. This is why
the trick is safe for dashboards and for `n` up to ~30.

> **Senior note.** Most teams reach for a Python/UDF step to get EMA. The recursive CTE is the
> correct expression but is serial and, in some engines, has recursion limits; the weighted-window
> version is a single parallel scan and is what you should reach for first. Keep the recursive
> version as the *reference implementation* your tests compare against.

### 4.9 Example 8 — smoothing a noisy conversion rate correctly

**Problem statement.** A landing page gets 3–3,000 visits/day. Daily conversion rates are useless;
you want a smoothed, correctly weighted rate plus its uncertainty.

```sql
WITH dense AS (
    SELECT c.d,
           COALESCE(v.visits, 0)  AS visits,
           COALESCE(v.signups, 0) AS signups
    FROM   (SELECT generate_series(DATE '2024-01-01', DATE '2024-04-30',
                                   INTERVAL '1 day')::date AS d) c
    LEFT   JOIN daily_funnel v ON v.d = c.d
)
SELECT d, visits, signups,
       ROUND(100.0 * signups / NULLIF(visits, 0), 2)  AS cr_today_pct,
       SUM(visits)  OVER w                             AS visits_7d,
       SUM(signups) OVER w                             AS signups_7d,
       ROUND(100.0 * SUM(signups) OVER w / NULLIF(SUM(visits) OVER w, 0), 2) AS cr_7d_pct,
       ROUND(100.0 * SUM(signups) OVER w28 / NULLIF(SUM(visits) OVER w28, 0), 2) AS cr_28d_pct
FROM   dense
WINDOW w   AS (ORDER BY d ROWS BETWEEN 6  PRECEDING AND CURRENT ROW),
       w28 AS (ORDER BY d ROWS BETWEEN 27 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

The contrast to internalize:

| Approach | What it computes | Failure mode |
|----------|------------------|--------------|
| `AVG(cr_today_pct) OVER w` | equal weight per day | a 3-visit day dominates a 3,000-visit day |
| `SUM(signups) OVER w / SUM(visits) OVER w` | **pooled** rate | correct; this is the metric |
| `cr_28d_pct` | longer smoothing | lags a genuine step change by ~2 weeks |
| binomial CI around `cr_7d_pct` | uncertainty | "2 pp drop" is often inside the noise band |

For a senior-level dashboard, add a binomial confidence interval so stakeholders do not react to
noise:

```sql
-- Wald-style interval, good enough for a dashboard; use Wilson for small n.
SELECT d,
       ROUND(100.0 * SUM(signups) OVER w / NULLIF(SUM(visits) OVER w, 0), 2) AS cr_7d_pct,
       ROUND(100.0 * 1.96 * SQRT(
                (SUM(signups) OVER w::numeric / NULLIF(SUM(visits) OVER w, 0))
              * (1 - SUM(signups) OVER w::numeric / NULLIF(SUM(visits) OVER w, 0))
              / NULLIF(SUM(visits) OVER w, 0)), 2) AS ci95_halfwidth_pp
FROM   dense
WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
ORDER  BY d;
```

### 4.10 Example 9 — crossover detection (7-day vs 28-day)

**Problem statement.** Alert when the short-term trend diverges from the long-term trend — the
classic "is growth stalling?" signal.

```sql
WITH ma AS (
    SELECT d, revenue,
           AVG(revenue) OVER w7  AS ma7,
           AVG(revenue) OVER w28 AS ma28
    FROM   march_daily
    WINDOW w7  AS (ORDER BY d ROWS BETWEEN 6  PRECEDING AND CURRENT ROW),
           w28 AS (ORDER BY d ROWS BETWEEN 27 PRECEDING AND CURRENT ROW)
)
SELECT d, ROUND(ma7, 2) AS ma7, ROUND(ma28, 2) AS ma28,
       ROUND(ma7 - ma28, 2)                            AS spread,
       CASE WHEN ma7 > ma28 THEN 'above' ELSE 'below' END AS regime,
       LAG(CASE WHEN ma7 > ma28 THEN 'above' ELSE 'below' END)
           OVER (ORDER BY d)                            AS prev_regime
FROM   ma
ORDER  BY d;
```

A **crossover** is the row where `regime <> prev_regime`; a **golden cross** is `below → above`.
In this March sample the 7-day average spends the month below the 28-day average (the 28-day
average still carries the 2024-03-03 and 2024-03-05 spikes), crosses **above** on 2024-03-25
(36.29 vs 27.92) because the 28-day window is weighed down by the long quiet stretch, and crosses
back **below** on 2024-03-28 (17.14 vs 24.93) once the 2024-03-05 spike falls out of the 28-day
window. Two crossovers in three days is the signal that a 28-day baseline is too short for a
series with this much sparsity — a small-scale version of exactly what you see on a real revenue
chart at a trend inflection.

### 4.11 Example 10 — smoothing with an explicit "min periods" gate

**Problem statement.** Publish a value for every day, but never a partial-window average; where
the window is not yet full, fall back to the cumulative mean (a legitimate choice for the first
week of a new product) and mark it.

```sql
WITH base AS (
    SELECT d, revenue,
           COUNT(*)      OVER w7 AS n7,
           AVG(revenue)  OVER w7 AS ma7,
           AVG(revenue)  OVER (ORDER BY d ROWS BETWEEN UNBOUNDED PRECEDING
                                             AND CURRENT ROW) AS cumulative_avg
    FROM   march_daily
    WINDOW w7 AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
)
SELECT d, revenue,
       ROUND(CASE WHEN n7 = 7 THEN ma7 ELSE cumulative_avg END, 2) AS smoothed,
       CASE WHEN n7 = 7 THEN 'rolling_7d' ELSE 'partial_fallback' END AS method,
       n7
FROM   base
ORDER  BY d;
```

| d | revenue | smoothed | method | n7 |
|---|--------:|---------:|--------|---:|
| 2024-03-01 |   0.00 |   0.00 | partial_fallback | 1 |
| 2024-03-02 |   0.00 |   0.00 | partial_fallback | 2 |
| 2024-03-03 | 120.00 |  40.00 | partial_fallback | 3 |
| 2024-03-04 |   0.00 |  30.00 | partial_fallback | 4 |
| 2024-03-05 | 305.00 |  85.00 | partial_fallback | 5 |
| 2024-03-06 |   0.00 |  70.83 | partial_fallback | 6 |
| 2024-03-07 |   0.00 |  60.71 | rolling_7d | 7 |
| … | … | … | … | … |

The `method` column is the deliverable. It documents, per row, which definition produced the
number — which is what makes a chart explainable when someone asks why the first week looks
different.

## 5. Reusable template

### 5.1 The full time-series smoothing template

```sql
WITH
-- 1. AGGREGATE to the reporting grain
fact AS (
    SELECT <period>            AS period,
           <entity>            AS entity,
           <agg(measure)>      AS measure          -- SUM for flows, COUNT(DISTINCT) for uniques
    FROM   <source>
    WHERE  <filters>
    GROUP  BY <period>, <entity>
),
-- 2. DENSIFY period × entity
spine AS (
    SELECT p.period, e.entity
    FROM   (<period generator>) p
    CROSS  JOIN (SELECT DISTINCT <entity> FROM <entity_dim>) e
),
-- 3. JOIN + FILL
metrics AS (
    SELECT s.period, s.entity,
           COALESCE(f.measure, 0) AS measure       -- 0 for flows; keep NULL for levels
    FROM   spine s
    LEFT   JOIN fact f ON f.period = s.period AND f.entity = s.entity
)
-- 4. WINDOW + 5. GUARD
SELECT period, entity, measure,
       COUNT(*)                        OVER w   AS window_rows,
       SUM(measure)                    OVER w   AS measure_sum_n,
       ROUND(AVG(measure)            OVER w, 2) AS measure_avg_n,
       ROUND(STDDEV_SAMP(measure)    OVER w, 2) AS measure_sd_n,
       CASE WHEN COUNT(*) OVER w = :n THEN AVG(measure) OVER w END AS measure_avg_n_strict
FROM   metrics
WINDOW w AS (PARTITION BY entity ORDER BY period
             ROWS BETWEEN :n_minus_1 PRECEDING AND CURRENT ROW)
ORDER  BY entity, period;
```

### 5.2 Ratio / rate smoothing (never average a ratio)

```sql
SELECT period,
       SUM(numerator)   OVER w AS num_n,
       SUM(denominator) OVER w AS den_n,
       SUM(numerator)::numeric OVER w / NULLIF(SUM(denominator) OVER w, 0) AS rate_n
FROM   metrics
WINDOW w AS (PARTITION BY entity ORDER BY period ROWS BETWEEN :n_minus_1 PRECEDING AND CURRENT ROW);
```

### 5.3 Value-based window (irregular / sparse timestamps)

```sql
SELECT ts, x,
       AVG(x) OVER (ORDER BY ts
                    RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS avg_7d
FROM   sparse_series;
-- Note: the OUTPUT is sparse. If the chart needs a dense axis, densify AND use ROWS.
```

### 5.4 Equal-weighted moving average with a stable tiebreak

```sql
AVG(x) OVER (PARTITION BY entity ORDER BY period, unique_seq
             ROWS BETWEEN :n_minus_1 PRECEDING AND CURRENT ROW)
```

The `unique_seq` term matters when several rows share a period (e.g. hourly rows bucketed to
days): without it, which rows fall inside the `n`-row frame is nondeterministic (chapter 01, §3.4).

### 5.5 Recruitment of parameters

| Business phrase | `n` | Frame |
|-----------------|-----|-------|
| "trailing week" | 7 | dense grid + `ROWS 6 PRECEDING` |
| "trailing 4 weeks" (weekly grain) | 4 | `ROWS 3 PRECEDING` |
| "trailing year" (monthly grain) | 12 | `ROWS 11 PRECEDING` |
| "last 24 hours" (event stream) | — | `RANGE INTERVAL '24 hours' PRECEDING` |
| "last 100 events" | 100 | `ROWS 99 PRECEDING` |
| "centered 7-day trend" | 7 | `ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING` |
| "same weekday last 4 weeks" | 4 | `ROWS BETWEEN 27 PRECEDING AND 24 PRECEDING` |

The last one is worth spelling out — a lagged window is how you build a seasonal-naive baseline:

```sql
-- "the same weekday for each of the last 4 weeks" = rows at distances 7,14,21,28
SELECT d, revenue,
       ( LAG(revenue, 7)  OVER (ORDER BY d)
       + LAG(revenue, 14) OVER (ORDER BY d)
       + LAG(revenue, 21) OVER (ORDER BY d)
       + LAG(revenue, 28) OVER (ORDER BY d) ) / 4.0 AS same_weekday_baseline
FROM   dense
ORDER  BY d;
```

### 5.6 Test kit for a smoothed metric

| Test | SQL shape | Catches |
|------|-----------|---------|
| Window is full where required | `COUNT(*) OVER w = n` on rows after the warm-up | sparce/renumbered source |
| Moving sum ≥ moving average × n | `ABS(sum - avg*n) < 1e-6` | frame mismatch between columns |
| Dense grid, no duplicate periods | `COUNT(*) = COUNT(DISTINCT period)` | double-counted joins |
| No negative values where impossible | `MIN(x) >= 0` for volume metrics | sign errors in deltas |
| Warm-up rows flagged | `method IS NOT NULL` on every row | silent partial windows |
| Moving average ≠ cumulative average | compare at row 30 vs row 3 | someone used `UNBOUNDED PRECEDING` |

## 6. Real-world use cases and applications

| Domain | Metric | Window | Notes |
|--------|--------|--------|-------|
| **Growth / product** | DAU 7-day MA, MAU/DAU stickiness | 7 / 28 | plot the MA to remove weekday aliasing |
| **Finance** | Revenue MTD/rolling 28-day, burn rate | 28 | moving *sum* rolls up cleanly |
| **Subscription** | Churn rate 3-month MA | 3 (monthly grain) | the classic board metric; smoothing is expected |
| **Fintech / risk** | 30-day average balance → fraud threshold | 30 | value-based if transactions are irregular |
| **Observability** | Latency p95 5-minute MA, error-rate bands | ROWS or RANGE on timestamp | RANGE for irregular scrape intervals |
| **IoT / telemetry** | Sensor smoothing, moving σ for anomaly alarms | 5–60 | robust stats preferred; sketches at scale |
| **Supply chain** | 28-day average demand, safety stock | 28 | moving sum for replenishment, average for cover |
| **Marketing** | ROAS 7-day MA, blended CAC trend | 7 | weighted rate, never average of ratios |
| **Marketplace** | GMV 7-day MA, supply/demand balance | 7 | per-market partitioning |
| **Ads** | eCPM smoothing for pacing/bidding | minutes/hours | `RANGE INTERVAL` on the event stream |
| **HR / ops** | Rolling attrition, ticket backlog | 90 / 7 | moving average of a *level*, not a flow |
| **Data engineering** | Row-count 7-day MA + z-score for freshness/anomaly monitors | 7 | the standard "volume anomaly" check |
| **ML / features** | Seasonality-free features, EWMA volatility | 7–90 | trailing only; centered windows leak the future |

Two production stories:

1. **"The new channel is outperforming everything."** A channel dashboard smoothed daily revenue
   with `ROWS BETWEEN 6 PRECEDING` over a **sparse** per-channel series. A channel that fired on
   three days of the week showed a "7-day average" of roughly 2.5× its true per-calendar-day
   value, and the budget was reallocated toward it. The fix was the densify + `LEFT JOIN` +
   `COALESCE(0)` grid; the ranking of channels changed completely. *Same window function, same
   `ORDER BY`, different frame semantics — the whole bug was a missing `generate_series`.*
2. **"Backfill changed the moving average for a year."** An incremental mart stored a rolling
   28-day revenue *sum* and updated it with `prev_sum + today - day_minus_28`. A late-arriving
   refund corrected a value 40 days back, so the subtraction term was stale and every subsequent
   row was off by the refund amount. The fix was to recompute the trailing 28 days on every run
   (cheap: 28 rows) instead of trusting the delta. General rule: **incremental windows must
   recompute at least `n` periods, not one.**

## 7. Performance notes and anti-patterns

### 7.1 How an engine executes a fixed-size frame

A frame with a *bounded* size (`ROWS BETWEEN n-1 PRECEDING AND CURRENT ROW`) is the second-cheapest
window shape after an unbounded-preceding cumulative:

- The executor keeps a **ring buffer** of `n` values plus a running aggregate. Adding the new row
  and evicting the old is O(1) per row when the aggregate has an **inverse transition**
  (`SUM`, `COUNT`, `AVG`); for `MIN`/`MAX` it needs a monotonic deque (still amortized O(1)).
- **Aggregates with no inverse** (`STDDEV`, `VARIANCE`, `PERCENTILE_*`, `ARRAY_AGG`,
  `COUNT(DISTINCT)`) force the executor to **re-scan the frame** for every row: O(n) per row →
  O(rows × n) total. A 7-day moving stddev over 100 M rows is ~700 M operations, not 100 M. This is
  why PostgreSQL's `WindowAgg` sometimes shows a `Window Spool`, and why SQL Server plans show
  `Window Spool` with large `Actual Rows`.
- `RANGE` with an interval offset additionally requires finding the frame start per row — a search
  if the ordering is not a simple integer. Cheap on a sorted index, expensive otherwise.

Practical consequences:

| Aspect | Guidance |
|--------|----------|
| Prefer `SUM`/`COUNT`/`AVG` rolling aggregates | they have inverse transitions |
| Beware rolling `STDDEV`/`VARIANCE` over wide windows on huge tables | consider sum-of-squares recomputation with `SUM(x)`, `SUM(x*x)`, `COUNT(x)` and a variance formula |
| Beware rolling percentiles/medians | no inverse; use sketches or restrict to small series |
| Keep `n` small | the buffer is `n` wide per partition per function |
| Prefer a **dense grid + `ROWS`** over `RANGE` with intervals | `ROWS` uses the ring buffer; `RANGE` may search |

Sum-of-squares variance trick (portable, O(1) per row):

```sql
SELECT d,
       COUNT(x) OVER w AS n,
       SUM(x)   OVER w AS s1,
       SUM(x*x) OVER w AS s2,
       CASE WHEN COUNT(x) OVER w > 1
            THEN SQRT( (SUM(x*x) OVER w - SUM(x) OVER w * SUM(x) OVER w / COUNT(x) OVER w)
                       / (COUNT(x) OVER w - 1) )
       END AS sd_samp
FROM series WINDOW w AS (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

Caveat: the sum-of-squares formula loses precision for large means with small variance (catastrophic
cancellation). Use it when `mean ≫ sd` is not the case; otherwise accept the engine's `STDDEV`.

### 7.2 Indexing

```sql
-- Backs PARTITION BY entity ORDER BY period with a bounded frame
CREATE INDEX ix_metrics_entity_period ON metrics (entity, period) INCLUDE (measure);
```

With the index in place, engines can stream partitions **without a sort**. Without it, the plan is
`Sort → WindowAgg` (PostgreSQL) or `Sort → Segment/Window Spool` (SQL Server) and you pay a full
sort of the input. Two further points:

- **Filter before the window.** `WHERE period >= :start` in the CTE limits the sort; the same
  predicate outside the CTE sorts everything (chapter 01, §7.2).
- **A densified grid multiplies rows.** `entity × period` can be much larger than the fact table
  when there are many entities. Densify only entities that matter (`WHERE entity IN (...)`) or
  build the grid from the fact table's distinct entities rather than from a dimension table.

### 7.3 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| `ROWS n PRECEDING` on a sparse series | window spans weeks/months, not `n` days | densify + `ROWS`, or `RANGE INTERVAL` |
| No densification at all | "weekly average" over 3 rows; chart with missing days | spine + `LEFT JOIN` + `COALESCE(0)` |
| Averaging per-period ratios | small-n periods dominate | `SUM(num) / SUM(den)` |
| No explicit `PARTITION BY` | one window across all entities, deliberately or not | add the partition |
| No explicit frame | `RANGE UNBOUNDED PRECEDING` — a cumulative, not a moving window | always write the frame |
| `AVG(x) OVER (ORDER BY d ROWS 6 PRECEDING)` with duplicate `d` | nondeterministic frame membership | add a unique tiebreaker |
| Publishing warm-up rows unfiltered | "chart dips at the start" | mask or flag with `COUNT(*) OVER w` |
| Centered window on a "latest value" tile | value changes retroactively for `n/2` periods | trailing window for live tiles |
| Leading window (look-ahead) in ML features | label leakage, great offline / awful online | trailing only; shift labels, not features |
| `LAG`-based "moving average" of 7 manual `LAG`s | 7 windows, 7 sorts in some planners, no `PARTITION BY` reuse | one `ROWS BETWEEN 6 PRECEDING` window |
| Recomputing only yesterday in an incremental mart | wrong after any backfill | recompute ≥ `n` periods |
| `PERCENTILE_CONT ... OVER (ORDER BY ... ROWS ...)` | error in PostgreSQL; frame ignored/illegal elsewhere | self-join, sketch, or engine-specific form |
| `STDDEV` over a wide frame at scale | O(rows × n) | sum-of-squares, or narrow the window |

### 7.4 Scale techniques

- **Pre-aggregate hard.** Smoothing 50 M raw events with a 7-day window is 50 M frame operations;
  aggregating to 5 K daily rows first is 5 K. Do step 1 of the pipeline religiously.
- **Materialize the dense grid once.** Compute it as a `dim_date`/`dim_calendar` model and reuse
  it; do not regenerate it in every query.
- **Two-level rolling sums for cross-partition windows.** For a window that must cross partition
  boundaries (e.g. rolling 7 days over sharded hourly data), compute per-shard partial sums for
  the boundary periods and add them — same trick as a distributed cumulative (chapter 02, §10.2).
- **Sketch-based rolling distinct.** For "distinct users in the last 28 days", keep one HLL/KLL
  sketch per day, then merge the last 28 sketches per output row. Merge cost is `28 × sketch` per
  row instead of a re-scan of the raw events.
- **Window on a pre-sorted table.** In a lakehouse, physically sort/cluster the table by
  `(entity, period)` (Delta `ZORDER`, Iceberg `sortOrder`, Snowflake `CLUSTER BY`); the window then
  streams without a shuffle.
- **Decide early what "the last 7 days" means at a period boundary.** A weekly report with a
  trailing-7-day MA is an MA *of the same period you just reported* — often you want a trailing
  *28-day* MA so the two windows do not alias.

## 8. Cross-engine differences

### 8.1 Frames and smoothing functions

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `ROWS BETWEEN n PRECEDING AND m FOLLOWING` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `RANGE` with **numeric** offsets | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| `RANGE` with **interval** offsets | ✅ | ✅ (`INTERVAL 1 DAY`) | ❌ | ✅ | ✅ | ❌ (`UNIX_DATE`/`UNIX_SECONDS`) |
| `GROUPS BETWEEN n PRECEDING` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `EXCLUDE CURRENT ROW/GROUP/TIES` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Named `WINDOW w AS (...)` | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `FILTER (WHERE …)` on window aggregates | ✅ | ❌ | ❌ | ❌ | ❌ (`COUNT_IF`) | ❌ (`COUNTIF`) |
| `STDDEV_SAMP/STDDEV_POP` as window | ✅ | ✅ | ✅ (`STDEV`/`STDEVP`) | ✅ | ✅ | ✅ |
| `PERCENTILE_CONT/DISC` as **window** | ❌ (ordered-set aggregates are not allowed as window functions) | ❌ (no `PERCENTILE_*` at all) | ⚠️ `OVER (PARTITION BY …)` only — no `ORDER BY`/frame | ⚠️ `PARTITION BY` only | ⚠️ `PARTITION BY` only — "does not support an `ORDER BY` clause within the `OVER` clause" and no explicit frames | ✅ `PERCENTILE_CONT(x, 0.5) OVER (…)`; also `APPROX_QUANTILES` |
| `APPROX_QUANTILES` / `APPROX_PERCENTILE` as window | ❌ (`tdigest` extension) | ❌ | ✅ `APPROX_PERCENTILE_CONT` (2022+) | ❌ | ✅ `APPROX_PERCENTILE` | ✅ (`APPROX_QUANTILES` + `[OFFSET(n)]`) |
| Recursive CTE for EMA | ✅ | ✅ | ✅ | ✅ | ⚠️ (recursive CTE not supported — emulate the recursive step with a stored procedure, or use the weighted-window trick) | ✅ |
| `EXP`/`LN`/`POWER` in window expressions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

> **Vendor-reality check.** The single biggest portability trap in this chapter is the rolling
> median. `PERCENTILE_CONT(...) WITHIN GROUP (ORDER BY ...) OVER (ORDER BY ... ROWS BETWEEN ...)`
> — the syntax every engineer writes first — is rejected by PostgreSQL, by MySQL (no function at
> all), and interpreted without the frame by SQL Server, Oracle and Snowflake. Verify against the
> engine docs before you promise a "rolling median" panel.

### 8.2 Date/period generators (step 2 of the pipeline)

| Engine | Idiom |
|--------|-------|
| PostgreSQL | `generate_series(DATE '2024-01-01', DATE '2024-04-30', INTERVAL '1 day')::date` |
| MySQL 8 | recursive CTE (`WITH RECURSIVE d(d) AS (...)`) with `DATE_ADD(d, INTERVAL 1 DAY)` |
| SQL Server | `GENERATE_SERIES(0, 120)` (2022+) or a tally table / `sys.all_objects` cross join |
| Oracle | `SELECT DATE '2024-01-01' + LEVEL - 1 FROM dual CONNECT BY LEVEL <= 121` |
| Snowflake | `SELECT DATEADD(day, seq4(), DATE '2024-01-01') FROM TABLE(GENERATOR(ROWCOUNT => 121))` |
| BigQuery | `SELECT d FROM UNNEST(GENERATE_DATE_ARRAY('2024-01-01','2024-04-30', INTERVAL 1 DAY)) AS d` |

### 8.3 Portability recipes

```sql
-- Portable trailing 7-row average (works on all six; needs a dense grid for calendar meaning)
AVG(x) OVER (PARTITION BY entity ORDER BY period, seq ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- Portable "last 7 calendar days" without RANGE interval support (e.g. SQL Server)
-- → densify on a daily grid, then use ROWS (this is the recommended default everywhere)

-- Portable pooled rate
SUM(num) OVER w::numeric / NULLIF(SUM(den) OVER w, 0)

-- Portable rolling variance without STDDEV's cost
CASE WHEN COUNT(x) OVER w > 1 THEN
  (SUM(x*x) OVER w - SUM(x) OVER w * SUM(x) OVER w / COUNT(x) OVER w) / (COUNT(x) OVER w - 1)
END
```

Dialect footguns to remember:

- **MySQL** has no `PERCENTILE_CONT`, no `FILTER`, no `GROUPS`, no `EXCLUDE`, no `QUALIFY`; but it
  *does* support `INTERVAL` in `RANGE` frames, which SQL Server does not.
- **SQL Server**'s `OVER` docs state plainly that offsets "aren't allowed for `RANGE`". Any
  interval-based smoothing must be rewritten as densify + `ROWS`. `PERCENTILE_CONT` accepts only
  `PARTITION BY`.
- **Oracle** supports interval `RANGE` and `PERCENTILE_CONT` with `PARTITION BY`; no `FILTER`, no
  `GROUPS`, no `EXCLUDE`.
- **Snowflake** supports `RANGE BETWEEN INTERVAL … PRECEDING` only for a restricted function set
  and forbids `DISTINCT` in those frames; it recommends using `rows`-based frames explicitly
  because its default frame for `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` is non-ANSI. Snowflake does
  **not** support recursive CTEs, so EMA needs a JavaScript/Python UDF, a stored procedure, or the
  weighted-window trick from §4.8.
- **BigQuery** requires the `RANGE` `ORDER BY` expression to be numeric; convert timestamps with
  `UNIX_SECONDS(t)`. It has both `PERCENTILE_CONT` and `APPROX_QUANTILES` available with `OVER`.
- **DuckDB** supports `GROUPS`, `EXCLUDE`, `QUALIFY`, `WINDOW` and interval `RANGE` — the best
  local playground for testing the portable subset before you deploy.

## 9. Exercises — Intermediate (hints only)

1. **Trailing 7-day average on a dense grid.** Build `march_daily` (spine + `LEFT JOIN` +
   `COALESCE`) and produce `revenue_7d` and `ma7`. Verify the last row's `revenue_7d` equals the
   sum of the last 7 days of raw data.
   *Hint:* `generate_series` + `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`.
2. **Prove the sparse trap.** Run the "7-day" average on the *un-densified* March data and print
   both the moving average and `COUNT(*) OVER w`. Explain in two sentences why 2024-03-21's
   window spans 18 days.
   *Hint:* `WINDOW w AS (ORDER BY revenue_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.
3. **Same metric, three definitions.** For 2024-03-08 produce (a) densified 7-day mean,
   (b) sparse `RANGE INTERVAL '6 days'` mean, (c) sparse 7-row mean. Give each a precise name.
   *Hint:* reuse §4.2 and §4.3; the answers are 63.43, 148.00, 148.00.
4. **Centered 7-day trend.** Produce `cma7` for March and discuss which rows are "final" on
   2024-03-25.
   *Hint:* `ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING`.
5. **Moving sum vs moving average.** Add `revenue_7d` and `ma7` to the same query and add a test
   column asserting `ABS(revenue_7d - ma7 * window_rows) < 1e-6`.
   *Hint:* `COUNT(*) OVER w` is `window_rows`.
6. **Per-channel smoothing.** Per `marketing_channel`, produce a 28-day trailing revenue sum on a
   dense grid. Which channel has the largest trailing 28-day revenue at 2024-04-30?
   *Hint:* `CROSS JOIN` a channel list with the date series, then `PARTITION BY channel`.
7. **Correctly weighted conversion rate.** Given `visits` and `signups` per day, produce
   `cr_7d_pct` as `SUM(signups)/SUM(visits)`. Then produce `avg_of_daily_cr` and explain the
   difference on a day where one day has 10× the traffic of the others.
   *Hint:* `AVG` vs `SUM/SUM`; use `NULLIF` on the denominator.
8. **Moving stddev and bands.** Build `ma7`, `sd7`, `upper2`, `lower2` and flag points outside the
   bands. How many flags in March, and is that the answer you expected?
   *Hint:* the point under test contaminates its own baseline; see §4.7.
9. **Warm-up gate.** Publish a `smoothed` column plus a `method` column from §4.11 and assert that
   exactly 6 rows are `partial_fallback`.
   *Hint:* `COUNT(*) OVER w = 7` as the switch.
10. **Crossover.** Produce the 7-day and 28-day MA and the day(s) where the 7-day MA crosses the
    28-day MA.
    *Hint:* compare `SIGN(ma7 - ma28)` with its `LAG`, then filter where they differ.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **EMA as a single-scan window.** Implement the normalized EMA via the weighted window
   (`SUM(x * w^-rn) / SUM(w^-rn)` over a truncated `ROWS` frame). Choose the truncation length from
   the decay tolerance you need (`w^k < ε`), prove the `POWER` overflow is avoided by shifting the
   exponent, and compare against the recursive-CTE reference on a synthetic 10,000-row series.
   Report max absolute error and run time for both.
   *Expected approach:* derive the cancellation argument, use
   `POWER(w, MAX(rn) OVER (PARTITION BY entity) - rn)`, and validate with
   `MAX(ABS(ema_window - ema_recursive))`. Conclude with the guidance: recursive for reference and
   small series, weighted-window for dashboards, and note the normalization difference in the
   metric documentation.
2. **Robust rolling median at scale.** Produce a rolling 28-day **median** revenue per channel for
   a table with 10^8 rows and 10^4 channels, using no more than a few GB of working memory per
   node. State the accuracy and the freshness guarantee.
   *Expected approach:* bucket each day's values into a fixed histogram (or KLL/t-digest sketch),
   store one sketch per (channel, day), then merge the last 28 sketches per output row and extract
   the median. Contrast with the self-join `PERCENTILE_CONT` (exact, O(n·k), dies at this scale) and
   with `NTILE`-based approximations (exact only for discrete percentiles, still needs the frame).
   Discuss sketch mergeability and why the merge is associative.
3. **Seasonality-aware smoothing.** A metric has a strong day-of-week pattern and a moving
   holiday (Easter). Design a smoothed series that removes both without over-smoothing, and give
   the SQL. Include how you validate that seasonality is actually removed.
   *Expected approach:* two paths — (a) a **seasonal-naive residual**: divide or subtract the
   same-weekday baseline (`LAG(x, 7)` and friends) and smooth the residual, then re-add; or
   (b) **STL-style decomposition** in the warehouse: weekly means and a centered MA for the trend,
   with the residual as the alerting series. Validate with an autocorrelation check at lag 7 on the
   smoothed series (should be ≈ 0) and with a holiday calendar dimension. Discuss why smoothing the
   raw series with a 28-day window is a blunt instrument and when it is good enough.
4. **Correct incremental maintenance under backfill.** You maintain a rolling 28-day revenue sum per
   channel, updated daily. A late refund can arrive up to 45 days late. Specify the data model, the
   update strategy, and the reconciliation query that proves correctness, including how you detect
   "the refund arrived but the mart was already published".
   *Expected approach:* never trust `prev_sum + today - day_28`; recompute the affected channel's
   trailing 28 days from a raw (append-only, immutable) fact table, driven by a
   "changed periods" set derived from the source's `updated_at`/CDC log. Add a daily assertion:
   stored rolling sum == recomputed rolling sum for every (channel, day) in the last 45 days.
   Discuss idempotency, late-data watermarks, and the choice between a full recompute of a small
   mart and an incremental recompute of a large one.
5. **Anomaly detection that survives spikes.** Implement a rolling z-score that does **not**
   self-damp (exclude the current row), does **not** break on a genuine level shift (use a longer
   baseline or an EWMA-based scale), and does **not** fire on weekends (calendar-aware baseline).
   Then quantify its precision/recall on a synthetic series with injected anomalies.
   *Expected approach:* baseline = trailing window *excluding current row*; scale = MAD or an
   EWMA of absolute deviations; seasonality handled by a same-weekday baseline or a weekday-aware
   robust center. Report a confusion matrix on the synthetic data and explain the trade-off
   between detection latency and false positives. Discuss why a fixed 3σ threshold on a heavy-tailed
   metric is the wrong default and what to use instead (quantile-based thresholds from the same
   trailing window).
6. **Design a "smoothing contract".** Write the one-page spec (not SQL) that a metric must satisfy
   to be published with a moving average: grain, calendar, frame type, window length, warm-up
   policy, rounding, NULL policy, backfill policy, and the three tests that gate it in CI. Then
   implement the spec as a dbt model + tests and show a case where two teams' "7-day average"
   differed and how the contract would have prevented it.
   *Expected approach:* force the four pipeline steps (aggregate → densify → window → guard) into
   the contract, make the frame explicit, require a `method` column for warm-up rows, and require the
   final-row checks (`MAX(cum) == SUM(x)`, `SUM(x) OVER w == AVG(x) OVER w * COUNT(*) OVER w`).
   This is the governance exercise the chapter is building toward.

## References

- PostgreSQL — Window Function Calls (frame syntax, `GROUPS`, `EXCLUDE`, `RANGE` interval offsets): <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- PostgreSQL — Window Functions ("ordered-set and hypothetical-set aggregates cannot be used as window functions"): <https://www.postgresql.org/docs/current/functions-window.html>
- PostgreSQL — `generate_series`: <https://www.postgresql.org/docs/current/functions-srf.html>
- MySQL 8.0 — Window Function Frame Specification (defaults, `RANGE` offsets, `INTERVAL`): <https://dev.mysql.com/doc/refman/8.0/en/window-functions-frames.html>
- MySQl Bug #93234 — `PERCENTILE_CONT`/`PERCENTILE_DISC` not implemented: <https://bugs.mysql.com/bug.php?id=93234>
- Microsoft Learn — `OVER` Clause (offsets not allowed in `RANGE`; performance guidance on indexes and window spools): <https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql>
- Microsoft Learn — `PERCENTILE_CONT` (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/functions/percentile-cont-transact-sql>
- Oracle — Analytic Functions: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/Analytic-Functions.html>
- Oracle — `PERCENTILE_CONT`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/PERCENTILE_CONT.html>
- Snowflake — Window function syntax and usage (interval offsets; function restrictions): <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — Windowed aggregations for rolling calculations: <https://docs.snowflake.com/en/user-guide/querying-time-series-data>
- Snowflake — `PERCENTILE_CONT` (window form: no `ORDER BY` in `OVER`, no explicit frames): <https://docs.snowflake.com/en/sql-reference/functions/percentile_cont>
- Google Cloud — BigQuery window function calls (numeric `RANGE` requirement; named windows): <https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls>
- Google Cloud — BigQuery `PERCENTILE_CONT` / `PERCENTILE_DISC`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/navigation_functions#percentile_cont>
- Google Cloud — BigQuery approximate aggregate functions (`APPROX_QUANTILES`): <https://cloud.google.com/bigquery/docs/reference/standard-sql/approximate_aggregate_functions>
- Google Cloud — BigQuery `GENERATE_DATE_ARRAY`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#generate_date_array>
- Amazon Redshift — Window functions: <https://docs.aws.amazon.com/redshift/latest/dg/c_Window_functions.html>
- DuckDB — Window functions (frames, `GROUPS`, `EXCLUDE`): <https://duckdb.org/docs/stable/sql/functions/window_functions.html>
- SILOTA SQL Recipes — Calculating an exponential moving average with recursive CTEs: <http://www.silota.com/docs/recipes/sql-recursive-cte-exponential-moving-average.html>
- MSSQLTips — Exponential moving average in SQL Server: <https://www.mssqltips.com/sqlservertip/6448/exponential-moving-average-calculation-in-sql-server/>
- Mode / ThoughtSpot — SQL Window Functions tutorial: <https://mode.com/sql-tutorial/sql-window-functions/>
- Datawise — ROWS vs RANGE in BigQuery window functions: <https://datawise.dev/using-range-in-window-functions-in-bigquery>
- sonra.io — Window function ROWS and RANGE on Redshift and BigQuery: <https://sonra.io/window-function-rows-and-range-on-redshift-and-bigquery/>
- use-the-index-luke — Indexing for `ORDER BY` and window functions: <https://use-the-index-luke.com/sql/sorting-grouping>

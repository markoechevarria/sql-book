# Cohort Analysis

> Cohort analysis answers the one question aggregate metrics cannot: **"is the product getting
> better, or did we just acquire more people?"** A flat DAU number can hide a product whose new
> users churn in a week and whose old users are the only thing holding the line. The moment you
> split a population by *when it first showed up* and then track it forward in **aligned time**
> (week 0, week 1, week 2 — not calendar weeks), that hiding place disappears. Brutal, honest, and
> entirely mechanical: two `date_trunc`s, one subtraction, one `COUNT(DISTINCT)`, one cross-tab.

## TL;DR

- A **cohort** is a group of entities defined by a shared start event in a shared time bucket:
  "users who signed up in the week of 2024-01-01", "accounts created in Q3". The bucket is the
  cohort key.
- The **cohort index** (`cohort_age`, `period_number`) is *time aligned to each cohort's own
  start*: `FLOOR((activity_date - cohort_start) / interval)`. This is what makes a cohort matrix
  comparable across rows. Confusing it with a calendar period is the #1 cohort bug.
- The core is four lines: `date_trunc` the signup bucket, `date_trunc` the activity bucket, take
  the difference as an integer index, and `COUNT(DISTINCT entity)` per (cohort, index).
- **Normalise by cohort size** to compare rows: `active / cohort_size`. Always publish the size
  column next to the percentage — 100 % of 1 is not 100 % of 10,000.
- **Censor incomplete cells.** A cohort created last week has no week-8 data. Showing `0` (or
  `NULL`) in those cells and then averaging across the row is how you manufacture a fake
  declining trend. Truncate the matrix at `FLOOR((today - cohort_start) / interval)`.
- **Two missing-value semantics:** `0` means "the cohort existed and nobody came back";
  `NULL`/absent means "not yet observable". Never conflate them — use `NULL` for censored cells
  and `0` only for genuinely observed zero activity.
- **Cohort definition is a product decision.** Signup cohorts include users who never activated;
  first-action cohorts do not. "Days since signup" cohorts differ from "days since first
  purchase" cohorts. Name your cohort by its event, and document it.
- **Cross-tab with `CASE` is portable; `PIVOT` is not.** `SUM(CASE WHEN idx = 2 THEN 1 ELSE 0 END)
  AS w2` works on every engine; `PIVOT` exists on SQL Server, Oracle, Snowflake and BigQuery and
  changes shape when the bucket list grows.
- Read the matrix in three ways: **horizontal** (retention curve per cohort), **vertical**
  (same age across cohorts — the only comparison that is fair), and **cohort-size weighted**
  (the "average retention curve" that sits on top of the heatmap).

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Cohort** | A group of entities sharing a start event within a time bucket (usually calendar-aligned). |
| **Cohort key** | The bucket label, e.g. `2024-01-01` for the week starting that Monday. |
| **Cohort index / age / period** | Integer number of periods since the cohort start; `0` = the start period. |
| **Cohort size** | Number of entities in the cohort at index 0 (or at the start event). The denominator. |
| **Retention matrix / cohort table** | Rows = cohorts, columns = cohort index, cells = active entities or a percentage. |
| **Censoring** | A cohort that has not yet lived long enough to have a value at index `k`. |
| **Normalisation** | Dividing by cohort size so rows are comparable, and so the column total is a weighted average. |
| **Calendar alignment** | Making bucket boundaries regular (`date_trunc('week'|'month', …)`). |
| **Signup cohort** | Cohort defined by `users.signup_date`. |
| **First-action cohort** | Cohort defined by the first occurrence of a meaningful event (`MIN(purchase_time)`). |
| **Rolling / bracket cohort** | "Did the entity return in period `k` *or later*?" rather than "exactly in `k`". |
| **Engagement cohort** | Matrix where the cell is a *level* (events per active user, revenue per user), not just a headcount. |
| **Survivorship bias** | Reading a cohort matrix only for cohorts that survived; classic misinterpretation. |

### 1.2 Calendar cohort vs aligned cohort — the picture

```
                calendar week
            Jan 01   Jan 08   Jan 15   Jan 22   Jan 29
signup wk
Jan 01      ██████   ██████   ██████   ██████   ██████
Jan 15               ██████   ██████   ██████   ██████
Jan 29                        ██████   ██████   ██████

   ──────────  calendar view (a triangle)  ──────────

            cohort index (weeks since that cohort's start)
              W0       W1       W2       W3       W4
Jan 01      ██████   ██████   ██████   ██████   ██████
Jan 15      ██████   ██████   ██████   ██████   ░░░░░░   ← censored
Jan 29      ██████   ██████   ██████   ░░░░░░   ░░░░░░   ← censored

   ──────────  aligned view (a rectangle + a censored corner)  ──────────
```

Only the aligned view lets you compare "week 2" across cohorts. The calendar view compares
2024-01-15-in-the-Jan-01-cohort against 2024-01-15-in-the-Jan-15-cohort — different ages, so the
comparison is meaningless.

### 1.3 The five metric families you can put in a cohort matrix

| Cell content | Metric | Question answered |
|--------------|--------|-------------------|
| `COUNT(DISTINCT entity)` / size | **Retention** | what fraction came back |
| `SUM(events)` / active entities | **Engagement depth** | how much they did, per active user |
| `SUM(revenue)` / size | **Monetisation / LTV curve** | how much value per cohort member |
| `SUM(revenue)` cumulative / size | **Cumulative LTV** | total lifetime value per member by age `k` |
| `COUNT(DISTINCT entity) FILTER (WHERE event = 'purchase')` / size | **Feature / funnel adoption** | adoption of a specific behaviour by age |

### 1.4 Retention definitions that differ (and therefore disagree)

| Variant | Cell definition | Behaviour |
|---------|-----------------|-----------|
| **Classic / N-period** | active in exactly period `k` | can go up and down; "came back in week 2" |
| **Rolling / bracket** | active in period `k` or any later period | monotonically non-increasing; "was still around at week 2" |
| **Unbounded / "at least once since"** | active at any point from 0 to `k` | monotonic; equals rolling when you accumulate |
| **Cumulative active ever** | ever active up to `k` | monotonic, the same as the above if computed over distinct entities |

Chapter 05 (`05-retention.md`) is devoted to these; the point here is that a cohort matrix is
meaningful only when the cell definition is written down.

## 2. Why it matters / what problem it solves

Aggregate metrics mix **composition** and **behaviour**, and therefore cannot distinguish:

| Symptom | Aggregate says | Cohort says |
|---------|----------------|-------------|
| DAU flat, signups up | "growth is fine" | new cohorts are worse; the old cohort is carrying the average |
| Revenue up | "we're winning" | revenue up because of acquisition; retention/LTV is falling |
| Churn rate stable | "nothing changed" | churn moved from month-1 to month-3 (worse — you paid more to acquire) |
| Conversion up | "the funnel improved" | conversion up because the traffic mix shifted to a cheaper channel |
| MAU/DAU down | "engagement is down" | a one-off surge cohort aged out; the newest cohorts are stickier |
| Feature adoption "40 %" | "fine" | 40 % of *all* users, but 5 % of the newest cohort |

Cohort analysis is the standard tool for four business decisions:

1. **Product/market fit and PMF decay** — is week-4 retention of new cohorts better than it was six
   months ago? (The single most-used growth signal in consumer products.)
2. **Channel and campaign quality** — which acquisition channel produces cohorts that are still
   around in month 3? CAC means nothing without a retention curve to multiply it by.
3. **Unit economics** — a cohort-LTV curve told you payback period and whether a channel is
   profitable, which is impossible from a blended LTV.
4. **Onboarding / activation experiments** — did the new flow improve week-1 retention for the
   cohort that received it? (Cohort analysis is how A/B tests on retention are actually read.)

Costs of getting it wrong:

- A team reported "day-30 retention of 42 %" from a matrix where most cells were `NULL` because
  the cohorts were younger than 30 days. The `AVG` over the column ignored the `NULL`s (SQL
  `AVG` skips `NULL`), so the number was computed only over the few old cohorts — a textbook
  **survivorship bias** that reversed when the cohort set matured.
- A "retention" number built on calendar weeks rather than cohort-aligned weeks reported a spike at
  "week 2" that was entirely an artefact of two cohorts' calendar weeks overlapping.

## 3. The pattern / technique explained

### 3.1 The five-step cohort pipeline

```
1. DEFINE the cohort      cohort_key = date_trunc('week', <start_event_time>)
2. DEFINE the activity    activity_key = date_trunc('week', <activity_event_time>)
3. INDEX them             cohort_index = (activity_key - cohort_key) / 1 week   → integer
4. AGGREGATE              COUNT(DISTINCT entity_id) per (cohort_key, cohort_index)
5. NORMALISE & CENSOR     divide by cohort size; NULL out cells where index > max observable
```

Each step has exactly one classic failure mode:

| Step | Failure mode |
|------|--------------|
| 1 | cohort defined by the wrong event (signup vs activation) |
| 2 | "activity" defined by the same event that defined the cohort (so index 0 is always 100 %) |
| 3 | using `activity_key - cohort_key` as a *date* rather than dividing into an integer index |
| 4 | `COUNT(*)` instead of `COUNT(DISTINCT entity)` → counts events, not entities |
| 5 | averaging percentages across cohorts of different sizes (unweighted average) |

### 3.2 The cohort key and the index

```sql
-- Cohort key: the bucket containing the start event.
date_trunc('week', u.signup_date)::date                    -- PostgreSQL, Monday-start week
-- Interval-agnostic index, PostgreSQL/standard-ish:
((date_trunc('week', e.event_date) - date_trunc('week', u.signup_date)) / 7)::int
```

Portable index formulas:

| Engine | Weeks since | Days since | Months since |
|--------|-------------|------------|--------------|
| PostgreSQL | `((aw - cw) / 7)::int` on dates | `(ad - cd)` | `(EXTRACT(YEAR FROM m)*12 + EXTRACT(MONTH FROM m)) - (EXTRACT(YEAR FROM c)*12 + EXTRACT(MONTH FROM c))` |
| MySQL 8 | `TIMESTAMPDIFF(WEEK, cw, aw)` | `TIMESTAMPDIFF(DAY, …)` | `TIMESTAMPDIFF(MONTH, …)` |
| SQL Server | `DATEDIFF(WEEK, cw, aw)` | `DATEDIFF(DAY, …)` | `DATEDIFF(MONTH, …)` |
| Oracle | `TRUNC((aw - cw)/7)` | `aw - cw` | `MONTHS_BETWEEN(m, c)` |
| Snowflake | `DATEDIFF('week', cw, aw)` | `DATEDIFF('day', …)` | `DATEDIFF('month', …)` |
| BigQuery | `DATE_DIFF(aw, cw, WEEK)` | `DATE_DIFF(aw, cw, DAY)` | `DATE_DIFF(m, c, MONTH)` |

> **Caution, month arithmetic.** `DATEDIFF('month', '2024-01-31', '2024-02-29')` is `1` in most
> engines (they compare truncated periods), but `MONTHS_BETWEEN('2024-02-29','2024-01-31')` is
> `0.93` in Oracle. Always `date_trunc` first and use the **period-number difference**
> (`year*12 + month`) for month cohorts; it is unambiguous and portable.

### 3.3 The four window-function shortcuts you may not need

A cohort matrix is a `GROUP BY`, not a window function. But three window idioms appear around it:

```sql
-- (a) cohort rank of each user's first activity — an alternative cohort key
MIN(first_seen) OVER (PARTITION BY user_id)         -- trivially equal to the group-by value

-- (b) the max observable index per cohort — for censoring
FLOOR(EXTRACT(EPOCH FROM (DATE :as_of - MIN(cohort_key) OVER (PARTITION BY cohort_key)))
      / 604800)::int

-- (c) running/cumulative LTV inside the matrix
SUM(revenue) OVER (PARTITION BY cohort_key ORDER BY cohort_index
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- (d) the weighted average retention curve (top row of the heatmap)
SUM(active_users) * 1.0 / SUM(cohort_size)      -- per cohort_index, over cohorts that have that index
```

### 3.4 Censoring, precisely

For cohort `c` and as-of date `A` with bucket size `B`:

```
max_observable_index(c) = FLOOR( (A - cohort_key(c)) / B )
```

Any cell with `cohort_index > max_observable_index` is **censored**. In SQL:

```sql
CASE WHEN cohort_index > FLOOR(EXTRACT(EPOCH FROM (DATE :as_of - cohort_key)) / 604800)::int
     THEN NULL ELSE retention_pct END
```

Censoring is not cosmetic. Without it, `AVG(retention_pct)` over a column silently *omits* the
young cohorts (because `AVG` skips `NULL`) but *includes* zeros if you wrote `0` instead — two
different wrong answers depending on whether you used `NULL` or `0`. Choose `NULL`, and make the
BI layer render it as a gap.

### 3.5 Activity definition matters

For the retention matrix to be informative, the **activity event must not be identical to the cohort
event**. Two common mistakes:

- Cohort = "signup", activity = "any event including signup" → every cohort is 100 % at index 0 by
  construction, which is fine, but the index-0 column carries no information. That is acceptable and
  conventional (it anchors the 100 %).
- Cohort = "first purchase", activity = "first purchase" → every row is `100, 0, 0, 0, …` and the
  matrix is useless. Activity must be a *repeatable* event ("placed an order", "opened the app").

The strong version separates the axes explicitly:

```
cohort  = when the entity entered        (signup / first activation / first purchase)
index   = how many periods since then    (0, 1, 2, …)
activity= a repeatable event within that period (any event / session / order / purchase)
```

## 4. Step-by-step worked example (with sample data and expected output tables)

All examples use the shared schema from `01-ranking.md`.

### 4.1 The input data

```sql
SELECT user_id, signup_date, marketing_channel FROM users ORDER BY user_id;
```

| user_id | signup_date | marketing_channel |
|--------:|-------------|-------------------|
| 1 | 2024-01-05 | organic |
| 2 | 2024-01-07 | paid_search |
| 3 | 2024-01-15 | referral |
| 4 | 2024-02-01 | organic |
| 5 | 2024-02-14 | paid_social |
| 6 | 2024-03-02 | organic |
| 7 | 2024-03-03 | paid_search |
| 8 | 2024-03-20 | referral |
| 9 | 2024-04-01 | organic |
|10 | 2024-04-05 | paid_social |

Activity days per user (derived from `events`):

| user_id | activity dates |
|--------:|----------------|
| 1 | 2024-01-05, 01-06, 01-12, 01-19, 01-26, 02-02 |
| 2 | 2024-01-07, 01-08, 01-14, 01-21, 02-04 |
| 3 | 2024-01-15, 01-16, 01-22, 02-05 |
| 4 | 2024-02-01, 02-02, 02-09, 02-16, 03-01 |
| 5 | 2024-02-14, 02-15, 02-21, 02-28 |
| 6 | 2024-03-02, 03-03, 03-09 |
| 7 | 2024-03-04, 03-05, 03-06, 03-07, 03-09, 03-10, 03-11 |
| 8 | *(none — signed up, never activated)* |
| 9 | *(none)* |
|10 | *(none)* |

Users 8, 9 and 10 are **deliberate**: they are the "signed up but never activated" population that
a signup-cohort analysis must include in the denominator. Excluding them silently inflates
retention.

### 4.2 Example 1 — build the cohort matrix (counts)

**Problem statement.** Weekly cohorts by signup week; weekly retention; counts first, percentages
later.

```sql
WITH cohort AS (
    SELECT u.user_id,
           date_trunc('week', u.signup_date)::date AS cohort_week
    FROM   users u
), activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date
    FROM   events e
), joined AS (
    SELECT c.user_id,
           c.cohort_week,
           ((date_trunc('week', a.activity_date) - c.cohort_week) / 7)::int AS cohort_index
    FROM   cohort c
    LEFT   JOIN activity a USING (user_id)          -- LEFT: keep never-active users
)
SELECT cohort_week,
       cohort_index,
       COUNT(DISTINCT user_id) AS active_users
FROM   joined
WHERE  cohort_index IS NOT NULL                     -- drops the never-active rows from the cells
GROUP  BY cohort_week, cohort_index
ORDER  BY cohort_week, cohort_index;
```

| cohort_week | cohort_index | active_users |
|-------------|-------------:|-------------:|
| 2024-01-01 | 0 | 2 |
| 2024-01-01 | 1 | 2 |
| 2024-01-01 | 2 | 2 |
| 2024-01-01 | 3 | 1 |
| 2024-01-01 | 4 | 2 |
| 2024-01-15 | 0 | 1 |
| 2024-01-15 | 1 | 1 |
| 2024-01-15 | 3 | 1 |
| 2024-01-29 | 0 | 1 |
| 2024-01-29 | 1 | 1 |
| 2024-01-29 | 2 | 1 |
| 2024-01-29 | 4 | 1 |
| 2024-02-12 | 0 | 1 |
| 2024-02-12 | 1 | 1 |
| 2024-02-12 | 2 | 1 |
| 2024-02-26 | 0 | 1 |
| 2024-02-26 | 1 | 2 |
| 2024-02-26 | 2 | 1 |
| 2024-03-18 | 0 | 0 |
| 2024-04-01 | 0 | 0 |

(Note the two explicit zeros at the end: those cohorts exist and have zero active users. They are
*observed* zeros, not censored cells.)

### 4.3 Example 2 — the retention matrix as a cross-tab (`CASE`/`FILTER`)

**Problem statement.** Turn the long table into a matrix and normalise by cohort size.

```sql
WITH cohort AS (
    SELECT u.user_id, date_trunc('week', u.signup_date)::date AS cohort_week
    FROM   users u
), activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date FROM events e
), cell AS (
    SELECT c.cohort_week,
           ((date_trunc('week', a.activity_date) - c.cohort_week) / 7)::int AS idx,
           c.user_id
    FROM   cohort c
    JOIN   activity a USING (user_id)
), sizes AS (
    SELECT cohort_week, COUNT(*) AS cohort_size FROM cohort GROUP BY cohort_week
), matrix AS (
    SELECT cohort_week,
           COUNT(DISTINCT user_id) FILTER (WHERE idx = 0) AS w0,
           COUNT(DISTINCT user_id) FILTER (WHERE idx = 1) AS w1,
           COUNT(DISTINCT user_id) FILTER (WHERE idx = 2) AS w2,
           COUNT(DISTINCT user_id) FILTER (WHERE idx = 3) AS w3,
           COUNT(DISTINCT user_id) FILTER (WHERE idx = 4) AS w4
    FROM   cell GROUP BY cohort_week
)
SELECT s.cohort_week, s.cohort_size,
       COALESCE(m.w0, 0) AS w0, ROUND(100.0 * COALESCE(m.w0,0) / s.cohort_size, 1) AS w0_pct,
       COALESCE(m.w1, 0) AS w1, ROUND(100.0 * COALESCE(m.w1,0) / s.cohort_size, 1) AS w1_pct,
       COALESCE(m.w2, 0) AS w2, ROUND(100.0 * COALESCE(m.w2,0) / s.cohort_size, 1) AS w2_pct,
       COALESCE(m.w3, 0) AS w3, ROUND(100.0 * COALESCE(m.w3,0) / s.cohort_size, 1) AS w3_pct,
       COALESCE(m.w4, 0) AS w4, ROUND(100.0 * COALESCE(m.w4,0) / s.cohort_size, 1) AS w4_pct
FROM   sizes s LEFT JOIN matrix m USING (cohort_week)
ORDER  BY s.cohort_week;
```

| cohort_week | cohort_size | w0 | w0_pct | w1 | w1_pct | w2 | w2_pct | w3 | w3_pct | w4 | w4_pct |
|-------------|------------:|--:|------:|--:|------:|--:|------:|--:|------:|--:|------:|
| 2024-01-01 | 2 | 2 | 100.0 | 2 | 100.0 | 2 | 100.0 | 1 | 50.0 | 2 | 100.0 |
| 2024-01-15 | 1 | 1 | 100.0 | 1 | 100.0 | 0 |   0.0 | 1 | 100.0 | 0 |   0.0 |
| 2024-01-29 | 1 | 1 | 100.0 | 1 | 100.0 | 1 | 100.0 | 0 |   0.0 | 1 | 100.0 |
| 2024-02-12 | 1 | 1 | 100.0 | 1 | 100.0 | 1 | 100.0 | 0 |   0.0 | 0 |   0.0 |
| 2024-02-26 | 2 | 1 |  50.0 | 2 | 100.0 | 1 |  50.0 | 0 |   0.0 | 0 |   0.0 |
| 2024-03-18 | 1 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 |
| 2024-04-01 | 2 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 | 0 |   0.0 |

**Read this table and notice that it is wrong to read it as it stands.** The last two rows and most
of the right-hand columns are **censored** (see §4.4). The 2024-02-26 cohort shows 50 % at week 2
and the 2024-04-01 cohort shows 0 % at week 2 — but the second number does not exist yet.

### 4.4 Example 3 — censoring done right

**Problem statement.** Replace every not-yet-observable cell with `NULL`, and derive the matrix's
boundary from an explicit as-of date so the query is reproducible.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
cohort AS (
    SELECT u.user_id, date_trunc('week', u.signup_date)::date AS cohort_week
    FROM   users u
), activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date FROM events e
), cell AS (
    SELECT c.cohort_week,
           ((date_trunc('week', a.activity_date) - c.cohort_week) / 7)::int AS idx,
           c.user_id
    FROM   cohort c JOIN activity a USING (user_id)
), sizes AS (
    SELECT cohort_week, COUNT(*) AS cohort_size FROM cohort GROUP BY cohort_week
), merged AS (
    SELECT s.cohort_week,
           s.cohort_size,
           p.as_of,
           ((p.as_of - s.cohort_week) / 7)::int AS max_observable_index,
           COALESCE(m.w0, 0) AS w0, COALESCE(m.w1, 0) AS w1, COALESCE(m.w2, 0) AS w2,
           COALESCE(m.w3, 0) AS w3, COALESCE(m.w4, 0) AS w4, COALESCE(m.w5, 0) AS w5
    FROM   sizes s CROSS JOIN params p
    LEFT   JOIN (SELECT cohort_week,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 0) AS w0,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 1) AS w1,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 2) AS w2,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 3) AS w3,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 4) AS w4,
                        COUNT(DISTINCT user_id) FILTER (WHERE idx = 5) AS w5
                 FROM   cell GROUP BY cohort_week) m USING (cohort_week)
)
SELECT cohort_week, cohort_size, max_observable_index,
       CASE WHEN 0 <= max_observable_index THEN ROUND(100.0*w0/cohort_size,1) END AS w0_pct,
       CASE WHEN 1 <= max_observable_index THEN ROUND(100.0*w1/cohort_size,1) END AS w1_pct,
       CASE WHEN 2 <= max_observable_index THEN ROUND(100.0*w2/cohort_size,1) END AS w2_pct,
       CASE WHEN 3 <= max_observable_index THEN ROUND(100.0*w3/cohort_size,1) END AS w3_pct,
       CASE WHEN 4 <= max_observable_index THEN ROUND(100.0*w4/cohort_size,1) END AS w4_pct,
       CASE WHEN 5 <= max_observable_index THEN ROUND(100.0*w5/cohort_size,1) END AS w5_pct
FROM   merged
ORDER  BY cohort_week;
```

With `as_of = 2024-04-30`:

| cohort_week | cohort_size | max_observable_index | w0_pct | w1_pct | w2_pct | w3_pct | w4_pct | w5_pct |
|-------------|------------:|---------------------:|-------:|-------:|-------:|-------:|-------:|-------:|
| 2024-01-01 | 2 | 17 | 100.0 | 100.0 | 100.0 |  50.0 | 100.0 |   0.0 |
| 2024-01-15 | 1 | 15 | 100.0 | 100.0 |   0.0 | 100.0 |   0.0 |   0.0 |
| 2024-01-29 | 1 | 13 | 100.0 | 100.0 | 100.0 |   0.0 | 100.0 |   0.0 |
| 2024-02-12 | 1 | 11 | 100.0 | 100.0 | 100.0 |   0.0 |   0.0 |   0.0 |
| 2024-02-26 | 2 |  9 |  50.0 | 100.0 |  50.0 |   0.0 |   0.0 |   0.0 |
| 2024-03-18 | 1 |  6 |   0.0 |   0.0 |   0.0 |   0.0 |   0.0 |   0.0 |
| 2024-04-01 | 2 |  4 |   0.0 |   0.0 |   0.0 |   0.0 |   0.0 | `NULL` |

(`max_observable_index` is `FLOOR((2024-04-30 − cohort_week) / 7)`: 120/7 = 17, 106/7 = 15,
92/7 = 13, 78/7 = 11, 64/7 = 9, 43/7 = 6, 29/7 = 4.)

Three things this table demonstrates:

1. **The `NULL` shows up in exactly one cell** — the 2024-04-01 cohort at week 5, because that cohort
   is only 4 weeks old at the as-of date. Every other cell is *observed*, including the zeros.
2. **Zeros are information.** The two never-active cohorts (2024-03-18 and 2024-04-01) read as a full
   row of zeros. That is the correct and honest answer: the company acquired those users and none of
   them ever did anything. If you exclude them you report a retention matrix that flatters the
   product and hides an activation problem.
3. **Reading one row in isolation is misleading.** The 2024-01-29 cohort (a single user) shows
   `100, 100, 100, 0, 100` — a "cliff at week 3 and instant recovery at week 4", which is impossible
   for a real product and is pure `n = 1` noise. Read **columns**, not rows (see §4.5).

For a cleaner illustration, restrict to cohorts with `cohort_size >= 2 OR max_observable_index >= 4`,
which here means dropping only the two singleton low-information rows:

| cohort_week | size | W0 | W1 | W2 | W3 | W4 |
|-------------|-----:|---:|---:|---:|---:|---:|
| 2024-01-01 | 2 | 100.0 | 100.0 | 100.0 | 50.0 | 100.0 |
| 2024-02-26 | 2 |  50.0 | 100.0 |  50.0 |  0.0 |   0.0 |
| 2024-04-01 | 2 |   0.0 |   0.0 |   0.0 |  0.0 |   0.0 |

Even this is not publishable — two cohorts is an anecdote. The point of the sample is the mechanics,
not the marketing.

### 4.5 Example 4 — the cohort-size-weighted average retention curve

**Problem statement.** The "top row" of the heatmap: for each cohort index, what fraction of *all*
cohort members were active, weighting by cohort size.

```sql
-- Reuse the 'merged' CTE from §4.4 with a NULLIF-guarded denominator.
WITH long_form AS (
    SELECT cohort_week, cohort_size, max_observable_index, idx, active_users
    FROM   merged
    CROSS  JOIN LATERAL (VALUES
        (0, w0), (1, w1), (2, w2), (3, w3), (4, w4), (5, w5)
    ) AS v(idx, active_users)
    WHERE  idx <= max_observable_index          -- ← only observable cells
)
SELECT idx AS cohort_index,
       SUM(active_users)                                            AS active_users,
       SUM(cohort_size)                                             AS cohort_members,
       ROUND(100.0 * SUM(active_users) / NULLIF(SUM(cohort_size), 0), 1) AS weighted_retention_pct,
       COUNT(*)                                                     AS cohorts_observed
FROM   long_form
GROUP  BY idx
ORDER  BY idx;
```

| cohort_index | active_users | cohort_members | weighted_retention_pct | cohorts_observed |
|-------------:|-------------:|---------------:|-----------------------:|-----------------:|
| 0 | 6 | 10 | 60.0 | 7 |
| 1 | 7 | 10 | 70.0 | 7 |
| 2 | 5 | 10 | 50.0 | 7 |
| 3 | 2 | 10 | 20.0 | 7 |
| 4 | 3 | 10 | 30.0 | 7 |
| 5 | 0 |  8 |  0.0 | 6 |

Verify by hand. Index 0: all seven cohorts are observable, sizes `2+1+1+1+2+1+2 = 10`, and actives
are `2+1+1+1+1+0+0 = 6` → 60.0 %. Index 5: the 2024-04-01 cohort (size 2) is censored
(`max_observable_index = 4 < 5`), so the denominator is `10 − 2 = 8` and `cohorts_observed = 6`.
This is the *weighted* curve — it is **not** the unweighted mean of the row percentages:

| cohort_index | weighted (correct) | unweighted mean of cohort % (wrong) |
|-------------:|-------------------:|------------------------------------:|
| 0 | 60.0 | (100+100+100+100+50+0+0)/7 = 64.3 |
| 1 | 70.0 | (100+100+100+100+100+0+0)/7 = 71.4 |
| 2 | 50.0 | (100+0+100+100+50+0+0)/7 = 50.0 |
| 3 | 20.0 | (50+100+0+0+0+0+0)/7 = 21.4 |
| 4 | 30.0 | (100+0+100+0+0+0+0)/7 = 28.6 |

They diverge because the cohorts have different sizes (1, 1, 1, 1, 2, 1, 2) — and in a real
warehouse they diverge by a lot more, because one cohort will have 10,000 users and another 40. The
weighted form is the one that answers "what fraction of all the users we acquired were active at age
`k`".

**The non-monotonicity is real and is the most important thing to internalise.** Index 3 is 20 %
and index 4 is 30 %. Nothing about the product improved with age; the *cohort composition* observable
at each index is identical here, so the wobble is pure small-sample noise from `n ≤ 2` actives per
cell. The senior reading is: **cohort matrices with tiny N produce fake shapes**, and the correct
response is to roll up to monthly cohorts (or quarters) until every cell has thousands of entities.
Publishing this March sample as a retention chart would be malpractice — but the *query* is exactly
the query you ship.

One more diagnostic worth computing alongside the curve:

```sql
SELECT l.idx,
       ROUND(100.0 * SUM(l.active_users) / NULLIF(SUM(l.cohort_size), 0), 1)  AS retention_all_signups_pct,
       ROUND(100.0 * SUM(l.active_users)
                   / NULLIF(SUM(l.cohort_size_active), 0), 1)                 AS retention_activated_pct
FROM   long_form l
GROUP  BY l.idx ORDER BY l.idx;
```

where `cohort_size_active` counts only cohort members with at least one event ever. Reporting both is
standard practice at consumer companies: the first is the honest "did the cohort stay", the second is
the diagnostic "among those who ever activated, do they stay". The **gap between the two curves is
your onboarding problem** — here it is dramatic, because 3 of the 10 users never fired a single
event.

| cohort_index | retention_all_signups_pct | retention_activated_pct |
|-------------:|--------------------------:|------------------------:|
| 0 | 60.0 | 6/7 = 85.7 |
| 1 | 70.0 | 7/7 = 100.0 |
| 2 | 50.0 | 5/7 = 71.4 |
| 3 | 20.0 | 2/7 = 28.6 |
| 4 | 30.0 | 3/7 = 42.9 |
| 5 |  0.0 | 0/6 = 0.0 |

(The denominator here is the 7 users who ever activated; the 3 never-active users are excluded from
the second curve entirely and *included* in the first.)

### 4.6 Example 5 — the engagement cohort (levels, not just headcounts)

**Problem statement.** Retention tells you *whether* they came back. The engagement matrix tells you
*how much* they did. Same skeleton, different measure.

```sql
WITH cohort AS (
    SELECT u.user_id, date_trunc('week', u.signup_date)::date AS cohort_week FROM users u
), session_week AS (
    SELECT e.user_id,
           date_trunc('week', e.event_time)::date AS activity_week,
           COUNT(*) AS events_in_week
    FROM   events e GROUP BY 1, 2
), joined AS (
    SELECT c.cohort_week,
           ((s.activity_week - c.cohort_week) / 7)::int AS idx,
           s.user_id, s.events_in_week
    FROM   cohort c JOIN session_week s USING (user_id)
)
SELECT cohort_week, idx,
       COUNT(DISTINCT user_id)                            AS active_users,
       SUM(events_in_week)                                AS total_events,
       ROUND(AVG(events_in_week), 2)                      AS events_per_active_user,
       ROUND(SUM(events_in_week)::numeric
             / NULLIF(COUNT(DISTINCT user_id), 0), 2)     AS events_per_active_user_v2
FROM   joined
GROUP  BY cohort_week, idx
ORDER  BY cohort_week, idx;
```

Excerpt for the 2024-01-01 cohort (`W0 = 2024-01-01…01-07`, `W1 = 01-08…01-14`, and so on):

| cohort_week | idx | active_users | total_events | events_per_active_user |
|-------------|----:|-------------:|-------------:|----------------------:|
| 2024-01-01 | 0 | 2 | 16 | 8.00 |
| 2024-01-01 | 1 | 2 |  7 | 3.50 |
| 2024-01-01 | 2 | 2 |  9 | 4.50 |
| 2024-01-01 | 3 | 1 |  1 | 1.00 |
| 2024-01-01 | 4 | 2 |  2 | 1.00 |

Accounting, so you can verify it: in week 0, user 1 fired 11 events (5 on 2024-01-05 and 6 on
2024-01-06) and user 2 fired 5 (2024-01-07) → 16 events over 2 users → 8.00. In week 1, user 1's
2024-01-12 session is 4 events and user 2's 2024-01-08 plus 2024-01-14 sessions are 1 + 2 = 3 →
7 over 2 → 3.50. In week 2, user 1's 2024-01-19 session is 5 and user 2's 2024-01-21 session is 4 →
9 over 2 → 4.50. In week 3, only user 1 (2024-01-26, 1 event) → 1.00. In week 4, user 1 (2024-02-02)
and user 2 (2024-02-04) fire one event each → 2 over 2 → 1.00.

The structural point is what matters: with the same cohort/index skeleton you can emit *any* per-cell
measure, and the `events_per_active_user` column separates "fewer people came back" from "the people
who came back did less" — two very different product diagnoses. Notice that this cohort's *headcount*
retention never dropped below 100 % for the first two weeks, yet its *engagement per user* fell from
8.00 to 3.50 in one week. A headcount-only retention chart would have called that a success.

### 4.7 Example 6 — cohort LTV curve and payback

**Problem statement.** Cumulative revenue per cohort member by age, and how long until a cohort
repays a fixed CAC.

```sql
WITH cohort AS (
    SELECT u.user_id, date_trunc('month', u.signup_date)::date AS cohort_month,
           u.marketing_channel
    FROM   users u
), revenue AS (
    SELECT o.user_id, o.order_date,
           SUM(o.revenue) AS revenue
    FROM   order_revenue o WHERE o.status <> 'cancelled'
    GROUP  BY o.user_id, o.order_date
), joined AS (
    SELECT c.cohort_month, c.marketing_channel,
           ((date_trunc('month', r.order_date) - c.cohort_month) / 30)::int AS idx,
           r.revenue
    FROM   cohort c JOIN revenue r USING (user_id)
), sized AS (
    SELECT cohort_month, marketing_channel, COUNT(*) AS cohort_size
    FROM   cohort GROUP BY 1, 2
)
SELECT j.cohort_month, j.marketing_channel, s.cohort_size, j.idx,
       SUM(j.revenue)                                                AS revenue,
       ROUND(SUM(j.revenue) / s.cohort_size, 2)                      AS arpu_at_age,
       ROUND(SUM(SUM(j.revenue)) OVER (PARTITION BY j.cohort_month
                                       ORDER BY j.idx
                                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
             / s.cohort_size, 2)                                     AS cum_ltv_per_member
FROM   joined j JOIN sized s USING (cohort_month, marketing_channel)
GROUP  BY j.cohort_month, j.marketing_channel, s.cohort_size, j.idx
ORDER  BY j.cohort_month, j.marketing_channel, j.idx;
```

For the 2024-01-01 `organic`/`paid_search` split, remember user 1 is `organic` (494.00 lifetime) and
user 2 is `paid_search` (439.00 lifetime) — different cohorts' sizes therefore differ by channel, and
`cum_ltv_per_member` is the number you compare against CAC. The reusable shape is:

```
cum_ltv_per_member(cohort, k) = SUM over ages 0..k of (revenue in that age) / cohort_size
```

which is the cohort-level LTV curve. Two rules:

- **Denominator = cohort size at index 0** (all members), not "active members at age k" — otherwise
  you are computing ARPU-among-survivors and will overstate LTV.
- Use `30` for a month approximation only if your months are calendar months; otherwise use the
  period-number difference (year × 12 + month) to avoid double-counting a 31-day month.

### 4.8 Example 7 — cross-tab with `PIVOT` (SQL Server / Oracle / Snowflake / BigQuery)

```sql
-- SQL Server / Snowflake / BigQuery (GoogleSQL)
SELECT cohort_week, cohort_size, [0] AS w0, [1] AS w1, [2] AS w2, [3] AS w3
FROM (
    SELECT cohort_week, idx, COUNT(DISTINCT user_id) AS active_users, cohort_size
    FROM   cell_with_size
) src
PIVOT (SUM(active_users) FOR idx IN ([0], [1], [2], [3])) AS p
ORDER  BY cohort_week;

-- Oracle
SELECT * FROM (
    SELECT cohort_week, idx, active_users FROM cell_with_size
)
PIVOT (SUM(active_users) FOR idx IN (0 AS w0, 1 AS w1, 2 AS w2, 3 AS w3))
ORDER BY cohort_week;

-- PostgreSQL: tablefunc's crosstab, or just use CASE/FILTER (preferred)
CREATE EXTENSION IF NOT EXISTS tablefunc;
SELECT * FROM crosstab(
    'SELECT cohort_week, idx, COUNT(DISTINCT user_id)::int
       FROM cell_with_size GROUP BY 1, 2 ORDER BY 1, 2',
    'SELECT DISTINCT idx FROM cell_with_size ORDER BY 1'
) AS ct (cohort_week date, w0 int, w1 int, w2 int, w3 int, w4 int, w5 int);
```

The `PIVOT` trade-offs:

| Aspect | `CASE`/`FILTER` | `PIVOT` |
|--------|-----------------|---------|
| Portability | all six engines | SQL Server, Oracle, Snowflake, BigQuery (not PostgreSQL/MySQL) |
| Column list | hard-coded either way | hard-coded either way (except Snowflake/BigQuery `ANY`) |
| NULL handling | `COALESCE` explicit | engine-specific |
| Censoring | easy (`CASE WHEN idx <= max_idx`) | awkward — do it before the pivot |
| Nested percentages | free | requires an outer `SELECT` |
| Readability at 52 columns | poor | poor-ish, but `PIVOT` is marginally better |
| Dynamic bucket lists | requires dynamic SQL | Snowflake/BigQuery support `PIVOT (… FOR idx IN (ANY ORDER BY idx))` / `IN (…)` |

**Recommendation.** For a cohort matrix, keep the long form in the model and pivot in the BI layer.
The number of columns grows every week, so a hard-coded `PIVOT` in SQL is a maintenance liability;
`CASE` at least fails loudly when you add a bucket.

### 4.9 Example 8 — signup cohort vs first-action cohort

**Problem statement.** Show how the cohort definition changes the answer.

```sql
-- Signup cohorts: everyone who registered (includes never-activated users)
SELECT date_trunc('week', u.signup_date)::date AS cohort_week, COUNT(*) AS size
FROM   users u GROUP BY 1 ORDER BY 1;

-- First-action cohorts: only users who ever did something
SELECT date_trunc('week', MIN(e.event_time))::date AS cohort_week, COUNT(*) AS size
FROM   events e GROUP BY e.user_id;   -- one row per user, then re-aggregate
```

| cohort_week | signup cohort size | first-action cohort size |
|-------------|-------------------:|-------------------------:|
| 2024-01-01 | 2 | 2 |
| 2024-01-15 | 1 | 1 |
| 2024-01-29 | 1 | 1 |
| 2024-02-12 | 1 | 1 |
| 2024-02-26 | 2 | 2 |
| 2024-03-18 | 1 | **0 (user 8 never fired an event)** |
| 2024-04-01 | 2 | **0 (users 9 and 10 never fired an event)** |

So the two definitions differ by the **entire never-activated population**. This is not a technicality:

- **Signup cohorts** answer "of everyone we acquired in week X, how many are still with us?" — the
  number the business cares about.
- **First-action cohorts** answer "of everyone who actually started using the product in week X, how
  many stayed?" — the number that isolates product quality from onboarding failure.

Compute both and report the difference as the **activation gap**. A large gap means your acquisition
is fine and your onboarding is broken. This is the single most actionable cohort output.

### 4.10 Example 9 — rolling (bracket) retention vs classic

**Problem statement.** Two cells for the same cohort and index, using different definitions.

```sql
WITH cohort AS (
    SELECT u.user_id, date_trunc('week', u.signup_date)::date AS cohort_week FROM users u
), activity AS (
    SELECT DISTINCT e.user_id, date_trunc('week', e.event_time)::date AS activity_week
    FROM   events e
), idx AS (
    SELECT c.cohort_week, c.user_id,
           ((a.activity_week - c.cohort_week) / 7)::int AS idx
    FROM   cohort c JOIN activity a USING (user_id)
)
SELECT cohort_week, idx,
       COUNT(DISTINCT user_id) AS classic_active,
       (SELECT COUNT(DISTINCT i2.user_id) FROM idx i2
        WHERE i2.cohort_week = i.cohort_week AND i2.idx >= i.idx) AS rolling_active_or_later
FROM   idx i
GROUP  BY cohort_week, idx
ORDER  BY cohort_week, idx;
```

For the 2024-01-01 cohort (users 1 and 2, whose activity weeks are W0, W1, W2, W3, W4 for user 1
and W0, W1, W2, W4 for user 2):

| cohort_week | idx | classic_active | rolling_active_or_later |
|-------------|----:|---------------:|------------------------:|
| 2024-01-01 | 0 | 2 | 2 |
| 2024-01-01 | 1 | 2 | 2 |
| 2024-01-01 | 2 | 2 | 2 |
| 2024-01-01 | 3 | 1 | 2 |
| 2024-01-01 | 4 | 2 | 2 |

At index 3 the two definitions disagree, and that single cell is the whole distinction:

- **Classic** = "came back in *exactly* week 3" → only user 1 (2024-01-26) → 1.
- **Rolling** = "was around at week 3 *or later*" → user 1 (week 3) **and** user 2 (week 4) → 2.

Classic retention can **increase** with age: a user who skipped week 3 and returned in week 4 makes
the week-4 classic cell larger than the week-3 cell, which is exactly what happens here (1 → 2). That
is why product teams report the classic curve (it is informative about habit formation and reveals
which weeks users skip) and finance teams report the rolling curve (it is monotonically
non-increasing, so it is the only one you can put into a cohort-LTV model without the value going
up). Never mix them in one chart, and never describe a classic curve as "the retention rate" without
the word "exactly".

The `rolling` version is also the cheapest sanity check in the whole chapter: **rolling must be
non-increasing over `idx` for every cohort.** If it ever rises, you have a join or definition bug.

## 5. Reusable template

### 5.1 The cohort skeleton (copy, change four things)

```sql
WITH
-- 1. Define the cohort: one row per entity with its cohort key + cohort event date
cohort AS (
    SELECT <entity_id>,
           <cohort_start_event_time>                                  AS start_ts,
           date_trunc(:interval, <cohort_start_event_time>)::date      AS cohort_key
    FROM   <entity_source>
    WHERE  <cohort filters>              -- e.g. signup_date >= :start, is_test_account = false
),
-- 2. Define repeatable activity: one row per (entity, activity period)
activity AS (
    SELECT <entity_id>,
           date_trunc(:interval, <activity_event_time>)::date          AS activity_key
    FROM   <activity_source>
    WHERE  <activity filters>
    GROUP  BY 1, 2                       -- ← dedupe to one row per entity per period
),
-- 3. Align: integer period index since the cohort's own start
cell AS (
    SELECT c.entity_id,
           c.cohort_key,
           <period_index>(a.activity_key, c.cohort_key)                AS cohort_index
    FROM   cohort c
    JOIN   activity a USING (<entity_id>)
),
-- 4. Sizes and the censoring boundary
sizes AS (
    SELECT cohort_key, COUNT(*) AS cohort_size FROM cohort GROUP BY cohort_key
)
-- 5. Aggregate + normalise + censor
SELECT s.cohort_key,
       s.cohort_size,
       COUNT(DISTINCT CASE WHEN c.cohort_index <= :max_index THEN c.entity_id END) AS active_users,
       ROUND(100.0 * COUNT(DISTINCT CASE WHEN c.cohort_index <= :max_index THEN c.entity_id END)
                   / s.cohort_size, 1)                                             AS retention_pct
FROM   sizes s
LEFT   JOIN cell c ON c.cohort_key = s.cohort_key AND c.cohort_index = :k
GROUP  BY s.cohort_key, s.cohort_size
ORDER  BY s.cohort_key;
```

The four things you change: the **entity** source, the **cohort event**, the **activity event**, and
the **interval**.

### 5.2 The cross-tab macro (portable `CASE`/`FILTER`)

```sql
SELECT cohort_key, cohort_size,
       ROUND(100.0 * COUNT(DISTINCT CASE WHEN idx = 0 THEN entity_id END) / cohort_size, 1) AS w0_pct,
       ROUND(100.0 * COUNT(DISTINCT CASE WHEN idx = 1 THEN entity_id END) / cohort_size, 1) AS w1_pct,
       ROUND(100.0 * COUNT(DISTINCT CASE WHEN idx = 2 THEN entity_id END) / cohort_size, 1) AS w2_pct,
       ROUND(100.0 * COUNT(DISTINCT CASE WHEN idx = 3 THEN entity_id END) / cohort_size, 1) AS w3_pct
       -- …
FROM   base
GROUP  BY cohort_key, cohort_size
ORDER  BY cohort_key;
```

> `COUNT(DISTINCT CASE WHEN … THEN entity_id END)` is the fully portable form:
> `FILTER (WHERE …)` is PostgreSQL/Oracle/SQLite-only, `COUNTIF` is BigQuery-only, and `COUNT_IF`
> is Snowflake-only. The `CASE` inside the aggregate works everywhere and the `NULL` padding is
> exactly what `COUNT` ignores.

### 5.3 Censoring helper

```sql
-- One reusable expression; put it in a macro/CTE and never think about it again.
CASE WHEN idx <= FLOOR(EXTRACT(EPOCH FROM (DATE :as_of - cohort_key)) / :bucket_seconds)::int
     THEN retention_pct
     ELSE NULL
END AS retention_pct_censored
```

Bucket seconds: day 86,400; week 604,800; month — do **not** hard-code; use the period-number
difference for months.

### 5.4 Interpretation checklist

| Question | Where to look | What it means |
|----------|---------------|---------------|
| Is the product improving? | **vertical** comparison of the same index across cohorts | newer cohorts flatter curve = improving |
| Which cohort was best? | the whole row, over observable indices only | a strong week 0 and a weak week 3 is an onboarding win / habit loss |
| Which channel is best? | rows grouped by channel, vertical comparison | CAC × LTV only makes sense here |
| Is index 3 a cliff? | the column mean and its spread | a force-function/onboarding step ends at that age |
| Is the average lying? | cohort sizes | one 10-user cohort and one 10,000-user cohort cannot be averaged unweighted |
| Are the numbers trustworthy? | count of observable cohorts per column | < 10 cohorts per column ⇒ treat as anecdote, roll up the interval |

### 5.5 Assertions for CI

| Test | SQL |
|------|-----|
| Cohort sizes sum to the population | `SELECT SUM(cohort_size) = (SELECT COUNT(*) FROM users) FROM sizes` |
| Index-0 cell equals activated members | `w0 <= cohort_size` for every row |
| No cell exceeds cohort size | `SELECT COUNT(*) FROM base WHERE active_users > cohort_size` → 0 |
| No negative indices | `SELECT COUNT(*) FROM base WHERE idx < 0` → 0 |
| Censored cells are `NULL`, not `0` | `SELECT COUNT(*) FROM base WHERE idx > max_idx AND retention_pct IS NOT NULL` → 0 |
| Rolling ≥ classic at every cell | `SELECT COUNT(*) FROM base WHERE rolling < classic` → 0 |
| Weighted curve monotone (rolling version) | `SELECT COUNT(*) FROM (… LAG …) x WHERE rolling < prev_rolling` → 0 |

That last one is a genuinely useful canary: the **rolling** (bracket) retention curve is
mathematically non-increasing, so any increase is a definition or join bug.

## 6. Real-world use cases and applications

| Domain | Cohort definition | Index | Cell | Decision it drives |
|--------|-------------------|-------|------|--------------------|
| **Consumer app** | signup week | day / week | active | PMF, onboarding, week-1 cliff |
| **SaaS** | signup month | month | active, seats, MRR | net revenue retention, expansion |
| **E-commerce** | first purchase month | month | repeat purchase | repeat-rate, subscription attach |
| **Marketplace** | first listing / first order | week | supply or demand active | liquidity per cohort |
| **Fintech** | account-open month | month | transacting | activation and KYC funnel by cohort |
| **Subscription** | first-paid month | month | active subscriber, MRR | churn cohorts, payback period |
| **Gaming** | install day | day | session | D1/D7/D30 retention, level drop-off |
| **Marketing** | acquisition channel × signup week | week | active / revenue | CAC vs cohort LTV by channel |
| **EdTech** | enrolment term | week | lesson completed | course completion by intake |
| **Health / IoT** | device activation week | week | device transmitting | firmware/onboarding regressions |
| **B2B sales** | deal-close quarter | quarter | revenue, seats | NRR, gross/net retention |
| **Data platform** | dataset onboarding quarter | month | queries per dataset | adoption of a new warehouse/lakehouse |

Two production stories:

1. **"Retention is improving!"** A growth team rebuilt the onboarding flow and the week-1 retention
   of the new cohort rose from 31 % to 44 %. But the cohort changed too: the new cohort arrived via a
   partner integration whose users had *already* used a similar product, so their baseline intent was
   higher. Splitting the matrix by `marketing_channel` showed the improvement was 100 % mix shift and
   0 % product. The correct next step was a held-out channel comparison, not a launch. **Cohort
   analysis must be segmented, or it answers the wrong question confidently.**
2. **"The heatmap shows a cliff at month 3."** A subscription business saw every cohort drop from
   ~70 % to ~20 % between months 2 and 3. Investigation showed the "activity" event was
   `dashboard_view` and the product had a monthly usage cycle; the cliff was the natural billing
   cycle, not churn. Switching the index to *months since first payment* and the activity to
   *invoice paid* revealed smooth retention. **The choice of activity event changes the shape of the
   matrix more than anything else.**

## 7. Performance notes and anti-patterns

### 7.1 How the query executes

A cohort matrix is fundamentally:

```
scan(cohort source) ⋈ scan(activity source) → GROUP BY (cohort_key, index) → cross-tab
```

Cost drivers, in order:

1. **The activity scan.** Usually the largest table. Pre-aggregate it to
   `(entity_id, period)` with `GROUP BY` before the join — that reduces a 10^9-event table to
   `entities × periods` rows, which is typically 2–3 orders of magnitude smaller. **Always do this.**
2. **The join.** Hash join on `entity_id`, small side = cohort (one row per entity). Fine.
3. **The cross-tab.** Pure `GROUP BY` + aggregates; the `CASE`/`FILTER` form is a single pass over
   the reduced set.

Indexes that matter:

```sql
CREATE INDEX ix_events_user_time ON events (user_id, event_time);   -- helps the pre-aggregation
CREATE INDEX ix_users_signup    ON users  (signup_date);            -- helps cohort filters
```

For a warehouse with `events` partitioned by day, the activity pre-aggregation becomes a partition
scan with `GROUP BY`; columnar storage makes `COUNT(DISTINCT)` the expensive part, not the scan.

### 7.2 `COUNT(DISTINCT)` is the bottleneck

`COUNT(DISTINCT entity_id)` cannot be hash-aggregated with a small state the way `SUM` can, and per
group it needs a distinct set. On a large cohort matrix you have `cohorts × indices` groups, each
needing a distinct-set. Mitigations:

- **Shrink the input first** (step 1 above). One row per `(entity, period)` makes `COUNT(DISTINCT)`
  almost free — in many engines `COUNT(*)` after deduplication is faster than `COUNT(DISTINCT)` over
  duplicates.
- **Prefer `COUNT(*)` over a pre-deduplicated CTE.** `SELECT DISTINCT entity_id, period` then
  `COUNT(*)` is often faster than `COUNT(DISTINCT)` because the dedup can be a hash aggregate that
  the engine parallelises better.
- **Use approximate distinct when N is huge and the decision tolerates ±1 %.** `APPROX_COUNT_DISTINCT`
  (Snowflake, Redshift, Databricks, Oracle, SQL Server 2022) or `HLL_COUNT.*` (BigQuery). For a
  retention *ratio*, a 1 % error on a 30 % number is invisible.
- **For very wide matrices (52+ columns), pivot in the BI tool**, not in SQL.

### 7.3 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| Calendar weeks instead of cohort-aligned indices | a "triangle" you cannot compare across rows | compute the integer period index |
| `COUNT(*)` instead of `COUNT(DISTINCT entity)` | retention > 100 %; you counted events | dedupe activity to `(entity, period)` first |
| Dropping never-active users | inflated retention, hiding an onboarding problem | `LEFT JOIN` from the cohort source, keep zeros |
| `0` for censored cells | fake declining trend after averaging | `NULL` for censored; `0` only for observed zeroes |
| Averaging percentages across cohorts | small cohorts dominate | weighted: `SUM(active) / SUM(size)` |
| `AVG(retention_pct)` over a column with `NULL`s | silently skips young cohorts (survivorship bias) | censoring-aware denominator or explicit `WHERE idx <= max_idx` |
| Cohort by the same event as activity | every row is `100, 0, 0, …` | activity must be a repeatable event |
| Hard-coded `PIVOT` column list in a weekly model | breaks every week | `CASE` + BI pivot, or dynamic SQL |
| `date_trunc('month') + INTERVAL '1 month'` arithmetic for the index | off-by-one on 31-day months | period-number difference `year*12 + month` |
| Cohort by cohort, no segmentation | mix shift masquerades as product improvement | always slice by channel, platform, plan |
| Reading a matrix with < 10 cohorts per column | fake cliffs and hockey sticks | roll up the interval to month/quarter |
| Mixing classic and rolling retention in one chart | non-monotone series next to monotone ones | pick one; document it; publish both only side by side |

### 7.4 Scale techniques

- **Materialise the activity spine once.** A `fct_user_activity_period` table
  (`user_id, period, events, revenue`) built daily is the single most reused artefact in a product
  analytics warehouse. Every cohort, retention, DAU and engagement query then reads the same small
  table.
- **Pre-compute cohorts as a dimension.** `dim_user` carries `signup_cohort_week`,
  `signup_cohort_month`, `acquisition_channel`. Joins then need no `date_trunc` on the fact side.
- **Store the matrix long, not wide.** `(cohort_key, entity_id, cohort_index, active)` is a few
  million rows; the 52-column wide version is unmanageable in SQL and cheap in the BI layer.
- **Incremental cohort models.** Today's activity only touches `(cohort_key, index = today's index)`
  rows. A cohort mart is one of the few places where incremental materialisation is genuinely
  simple — as long as the cohort dimension itself is stable (see the anti-pattern: a late-arriving
  signup that changes a user's cohort must invalidate their whole row).
- **Approximate distinct for wide matrices.** 52 `COUNT(DISTINCT)` columns is 52 distinct sets over
  the same data; on Snowflake/BigQuery, `HLL`/`APPROX` collapses that dramatically.

## 8. Cross-engine differences

### 8.1 Building blocks

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `date_trunc` | ✅ `date_trunc('week', d)` | `DATE_FORMAT` / `DATE(d - INTERVAL WEEKDAY(d) DAY)` | `DATETRUNC('week', d)` (2022+) | `TRUNC(d,'IW')` | ✅ `DATE_TRUNC('week', d)` | ✅ `DATE_TRUNC(d, WEEK(MONDAY))` |
| Truncate to ISO Monday | ✅ (`week`, Monday) | emulate via `WEEKDAY` | `DATETRUNC(week)` (Sunday-based for `d` datepart) | `'IW'` | `'week'` (respects `WEEK_START`) | `WEEK(MONDAY)` explicit |
| Integer period index | `(aw - cw)/7` on dates | `TIMESTAMPDIFF(WEEK, cw, aw)` | `DATEDIFF(WEEK, cw, aw)` | `TRUNC((aw-cw)/7)` | `DATEDIFF('week', cw, aw)` | `DATE_DIFF(aw, cw, WEEK)` |
| Month period-number diff | `EXTRACT(YEAR…)*12 + EXTRACT(MONTH…)` | `YEAR()*12+MONTH()` | `DATEDIFF(MONTH, c, m)` | `MONTHS_BETWEEN` (fractional!) | `DATEDIFF('month',…)` | `DATE_DIFF(m, c, MONTH)` |
| `COUNT(DISTINCT x)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Conditional aggregate | `FILTER (WHERE …)`, `CASE` | `CASE` only | `CASE` only | `FILTER` is **not** supported → `CASE` | `COUNT_IF`, `CASE` | `COUNTIF`, `CASE` |
| `PIVOT` | ❌ (`tablefunc.crosstab`) | ❌ (emulate) | ✅ | ✅ | ✅ (`PIVOT … FOR … IN`) | ✅ (`PIVOT (… FOR … IN (…))`) |
| `PIVOT` with dynamic/`ANY` columns | ❌ | ❌ | ❌ | ❌ | ✅ `IN (ANY ORDER BY …)` | ❌ (must list values) |
| `APPROX_COUNT_DISTINCT` | ❌ (extension) | ❌ | ✅ (2022+) | ✅ | ✅ | ✅ (`APPROX_COUNT_DISTINCT` / `HLL_COUNT`) |
| `GENERATE_SERIES`-style period spine | ✅ | ❌ (recursive CTE) | ✅ `GENERATE_SERIES` (2022+) | ✅ `CONNECT BY` | ✅ `GENERATOR` | ✅ `GENERATE_DATE_ARRAY` |
| `QUALIFY` for the censoring filter | ❌ | ❌ | ❌ | ✅ 26ai+ | ✅ | ✅ |

### 8.2 Dialect notes that actually bite

- **`date_trunc('week')` is not universal Monday.** PostgreSQL uses the ISO week (Monday); Snowflake
  respects the session's `WEEK_START` parameter (default 0 = legacy Sunday); BigQuery requires
  `WEEK(MONDAY)` explicitly; SQL Server's `DATETRUNC(week)` is Sunday-based for the `date` type.
  **Cohort boundaries silently shift by a day between engines.** Set `WEEK_START=1` on Snowflake and
  always write `WEEK(MONDAY)` on BigQuery, or accept that your cohorts are Sunday-based and say so.
- **Oracle `MONTHS_BETWEEN` returns a fractional number.** `MONTHS_BETWEEN(DATE '2024-02-29',
  DATE '2024-01-31')` is `0.93` (31/28 is not 1). Truncate it *and* decide whether you want "one
  month later" to mean the same day-number (`ADD_MONTHS`) or the same calendar month; the
  period-number difference (`year*12+month`) is the safe choice.
- **MySQL has no `FILTER` and no `PIVOT`.** Use `SUM(CASE WHEN idx = 2 THEN 1 ELSE 0 END)` after
  deduplicating the activity spine, and pivot in the BI layer. Reaching for `PIVOT` will send you
  down a dynamic-SQL path unnecessarily.
- **Snowflake does not support recursive CTEs** (relevant if you generate a period spine that way) and
  its `PIVOT` is the friendliest of the six.
- **BigQuery's `PIVOT` requires the value list in the query text**, so a weekly cohort model must
  either be regenerated (dynamic SQL) or, far more commonly, emit long form and let the BI layer do
  the pivoting.
- **PostgreSQL's `crosstab`** requires the `tablefunc` extension, needs a consistent column list, and
  cannot express the percentage normalisation cleanly. Prefer `CASE`/`FILTER` unless you have a
  strong reason.

## 9. Exercises — Intermediate (hints only)

1. **Weekly signup cohorts.** Produce `cohort_week`, cohort size, and index-0 active users for the
   sample data. Which user is alone in their cohort? *Hint:* `date_trunc('week', signup_date)`.
2. **Retention matrix.** Produce a 6-column matrix of `retention_pct` by `(cohort_week,
   cohort_index)`. Verify the 2024-01-01 cohort's row is `100, 100, 100, 50, 100, 0`.
   *Hint:* `COUNT(DISTINCT user_id) FILTER (WHERE idx = k)` or `SUM(CASE WHEN idx = k THEN 1 ELSE 0 END)`
   after deduplication.
3. **Censor the matrix.** Add `max_observable_index` from an explicit `as_of` date and `NULL` out
   cells beyond it. How many cells change from `0` to `NULL` at `as_of = 2024-04-30`?
   *Hint:* `FLOOR(EXTRACT(EPOCH FROM (as_of - cohort_week)) / 604800)`.
4. **Weighted retention curve.** Produce the single-column series of `SUM(active)/SUM(size)` per
   index, restricted to observable cells. Explain why index 3 is lower than index 4.
   *Hint:* the cohort set observable at each index changes; count the cohorts per column.
5. **Monthly cohorts.** Redo the matrix on monthly signup cohorts with a monthly index. How many
   cohorts do you get, and how many are censored at index 1?
   *Hint:* use the period-number difference, not `date_trunc + INTERVAL`.
6. **Engagement matrix.** For each `(cohort_week, index)`, produce `active_users`,
   `total_events`, and `events_per_active_user`. Which cohort degrades fastest?
   *Hint:* build `(user_id, activity_week, COUNT(*))` once, then join.
7. **Channel segmentation.** Repeat exercise 2 sliced by `marketing_channel`. Which channel has the
   best index-1 retention, and how many users is that based on? (Careful: this is the "N too small"
   trap.)
   *Hint:* put `marketing_channel` in the cohort CTE and `GROUP BY` it.
8. **Signup vs first-action cohorts.** Produce both cohort sizes side by side and compute the
   "activation gap" per cohort. What is the gap for the 2024-04-01 cohort?
   *Hint:* left join the first-action cohort onto the signup cohort; `NULL` means zero activations.
9. **Classic vs rolling retention.** Produce both cells for every `(cohort, index)` and find all rows
   where they differ. Add a CI assertion that rolling ≥ classic everywhere.
   *Hint:* the rolling cell is `COUNT(DISTINCT user_id)` over all indices `>= k`.
10. **Pivot in two dialects.** Write the matrix once with `CASE`/`FILTER` (PostgreSQL) and once with
    `PIVOT` (Snowflake or BigQuery) and compare the maintenance burden when a 7th week is added.
    *Hint:* the `CASE` version fails loudly (missing column); the `PIVOT` version silently omits it.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Cohort LTV with payback against a channel-specific CAC.** Given `cohort_size`, a per-channel CAC
   from an ad-spend table, and cohort revenue, compute the **payback month** for each (cohort,
   channel) with censoring — i.e. do not report "never pays back" for a cohort that is only 2 months
   old and still climbing.
   *Expected approach:* build a bounded spine of `cohort × month`, `LEFT JOIN` revenue, `COALESCE`
   to 0, compute cumulative `cum_ltv_per_member`, then `MIN(index) FILTER (WHERE
   cum_ltv >= cac)`; for cohorts with no payback yet, emit `NULL` plus a "months observable" column
   and a lower bound on the payback month. Discuss why blending channels destroys this analysis and
   how to handle a channel with < 100 users (confidence interval on the payback month).
2. **Retention curve with confidence intervals.** For each cohort index, produce the weighted
   retention and a 95 % Wilson interval on the pooled proportion, plus the number of cohorts
   contributing. Then determine the minimum cohort size at which you would trust a 5 pp week-over-week
   move.
   *Expected approach:* pooled proportion = `SUM(active)/SUM(members)`; Wilson interval on that
   proportion treating members as trials; report `cohorts_observed` so readers see the small-N
   columns. Conclude with a "do not read this column below N members" annotation. Discuss why the
   Wilson interval is correct here and the naive Wald interval is not.
3. **Survivorship-safe "retention N days" metric.** Produce a "day-30 retention" number that is
   correct at any as-of date, does not silently drop young cohorts, and does not report a value for
   cohorts that cannot yet have one — including a machine-readable "insufficient data" state.
   *Expected approach:* compute per cohort `max_observable_index`; filter to
   `max_observable_index >= 30`; report `count_of_eligible_cohorts`; and when the count is below a
   threshold, return `NULL` with a reason code rather than a percentage. Then write the test that
   catches the classic bug (an `AVG` over a column with `NULL`s silently skipping young cohorts) by
   comparing against a cohort-count column.
4. **Cohort analysis on a late-arriving, mutable event stream.** Your `events` table receives events
   with up to 72 hours of latency and occasional corrections (a replayed batch can add events to a
   past day). Design a cohort model that is correct after a replay: which tables are incremental vs
   full-refresh, how do you detect that a cohort's past cells changed, and how do you avoid
   publishing a retention number that then silently revises?
   *Expected approach:* separate an immutable "as-received" landing layer from a curated activity
   model partitioned by event date; make the cohort mart incremental on `(cohort_key, index)` with a
   **recompute window** of `max_latency` buckets; emit a `published_at` and a `revision` column;
   compare the current cell against the previously published cell and alert on drift. Discuss why
   "restate silently" is unacceptable for a number in a board deck.
5. **Cohort analysis where the cohort is not a user.** Apply the same skeleton to a different entity:
   for example, "cohorts of *marketplaces* by launch month, tracking seller-side liquidity" or
   "cohorts of *warehouse tables* by onboarding quarter, tracking query volume". Give the schema
   changes, the metric definitions, the censoring rule, and the interpretation you would present.
   *Expected approach:* demonstrate that the wrapper (cohort key, aligned index, size, weighted
   average, censoring) is fully generic; the domain work is entirely in choosing the cohort event and
   the repeatable activity, and in deciding whether the denominator is entities or a modelled
   population. Include a discussion of when the "cohort" should instead be a *bucket of
   behaviour* (e.g. usage decile) rather than a time bucket — the two are often confused.
6. **A cohort matrix that survives a metric change.** You are told next quarter to add "purchase
   events" to the definition of "active". Explain exactly which cells of which cohorts would change,
   whether the historical series becomes non-comparable, and design the schema (a
   `metric_definition`/`semantic version` column on the cell) that lets a dashboard show both
   definitions without rewriting history.
   *Expected approach:* the cell is `(cohort_key, index, metric_definition_id, value)`; the cohort and
   index are definition-independent, and only the activity predicate changes; keep both series,
   label them, and never overwrite. Discuss the governance implication: a retention number is only
   meaningful with its definition attached, which is why "retention" in a metrics catalogue must be
   a fully-specified metric, not a word.

## References

- PostgreSQL — Date/Time functions and operators (`date_trunc`, `EXTRACT`, ISO weeks): <https://www.postgresql.org/docs/current/functions-datetime.html>
- PostgreSQL — Aggregate expressions (`FILTER`, `DISTINCT`) and window functions: <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES>
- PostgreSQL — `tablefunc` (`crosstab` and `crosstabN`): <https://www.postgresql.org/docs/current/tablefunc.html>
- MySQL 8.0 — Date and time functions (`TIMESTAMPDIFF`, `DATE_FORMAT`, `WEEKDAY`): <https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html>
- Microsoft Learn — `DATEDIFF` (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/functions/datediff-transact-sql>
- Microsoft Learn — `FROM` — `PIVOT` and `UNPIVOT`: <https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot>
- Oracle — `MONTHS_BETWEEN`, `TRUNC` (date), `ADD_MONTHS`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/MONTHS_BETWEEN.html>
- Oracle — `PIVOT` clause: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/SELECT.html>
- Snowflake — `DATE_TRUNC` / week-start behaviour: <https://docs.snowflake.com/en/sql-reference/functions/date_trunc>
- Snowflake — `PIVOT` and `DATEDIFF`: <https://docs.snowflake.com/en/sql-reference/constructs/pivot>
- Google Cloud — BigQuery `DATE_TRUNC` with `WEEK(MONDAY)` and `PIVOT`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/date_functions>
- Google Cloud — BigQuery `PIVOT` operator: <https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#pivot_operator>
- Amplitude — How time works in a retention analysis (N-day vs rolling vs bracket): <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-time>
- Amplitude — Interpret your retention analysis: <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-interpret>
- Amplitude — Mastering user retention playbook (cohort curves, D1/D7/D30): <https://info.amplitude.com/rs/138-CDN-550/images/product-analytics-playbook-vol1-amplitude.pdf>
- ga4bigquery — Sessionizing GA4 event data in BigQuery, part 1 (30-minute default): <https://www.ga4bigquery.com/how-to-sessionize-your-ga4-event-data-in-bigquery-part-1-default-session-definition/>
- dbt Labs — Incremental models (cohort marts as incremental): <https://docs.getdbt.com/docs/build/incremental-models>
- dbt Labs — Metrics and semantic layer (why a "retention" metric needs a definition): <https://docs.getdbt.com/docs/build/build-metrics-intro>
- Mode / ThoughtSpot — SQL tutorials (window functions and conditional aggregation used throughout): <https://mode.com/sql-tutorial/sql-window-functions/>

# Retention

> Retention is the single most important number in a consumer product and the one most often
> computed three different ways by three different teams. "Day-30 retention is 42 %" is not a
> metric; it is an ambiguous sentence. It becomes a metric when you say *which* 30 days, *which*
> population is the denominator, *which* activity counts, and whether "day 30" means *exactly*
> day 30 (classic), *day 30 or later* (rolling), or *any time up to day 30* (unbounded). Those four
> choices produce numbers that differ by 20–40 points on the same data. This chapter builds all of
> them, side by side, so you can name the one your business actually means — and defend it.

## TL;DR

- **Four definitions, four different numbers.** Learn them as a quadrant and never use the word
  "retention" without one of these qualifiers:

  | Definition | Cell formula | Monotone? | Answers |
  |------------|--------------|-----------|---------|
  | **Classic N-period** | active in *exactly* period `k` | no (can rise) | "did they form a habit in week `k`?" |
  | **Rolling / bracket N** | active in period `k` *or any later* | yes ↓ | "were they still with us at week `k`?" |
  | **Unbounded N** | active in *any* period from 0 to `k` | yes ↑ | "had they ever come back by week `k`?" |
  | **Cumulative active ever** | ever active up to `k` | yes ↑ | same as unbounded for distinct entities |

- **Two denominators, two meanings.** `active / all_signups` is the number the business cares
  about; `active / ever_activated` isolates product quality from onboarding failure. Report both,
  and report the gap as *activation rate*.
- **Denominator must be fixed at period 0.** Dividing by the number of users active in period `k`
  gives you ARPU-among-survivors and will overstate everything. Cohort size is captured once, at
  `k = 0`.
- **Censoring is not optional.** A cohort that is 20 days old has no day-30 number. Emit `NULL`, add
  a `max_observable_day` column, and never let an `AVG()` over a `NULL`-containing column silently
  become a survivorship-biased estimate.
- **DAU / WAU / MAU are retention metrics wearing different clothes.** They are "active in the last
  1 / 7 / 30 days" — rolling windows. `DAU/MAU` (**stickiness**) is a *level* ratio, and it must be
  computed as `AVG(DAU over the month) / MAU`, not as "DAU on the last day of the month / MAU".
- **Churn and retention are not complements** when returns exist. `churn = 1 − classic retention`
  only if "churned" means "inactive in this period", which is a *pause*, not a cancellation. For
  subscription businesses the number that matters is **revenue** retention (NRR/GRR), not logo
  retention.
- **N-day retention is not a curve.** It is a family of independent cells. Plot it and you will see
  it wobble; that wobble is the product working, not a bug.
- **Consolidate.** Pick one primary retention definition per entity per product, put it in the
  metrics catalogue with its SQL, and require every dashboard to reference it. Three definitions of
  "retention" in one company is a governance failure, not an analytics one.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Retention** | Fraction of a cohort that performs a qualifying activity in a given period (see the four definitions). |
| **Classic / N-period retention** | Active in *exactly* period `k`. Written `Dn` / `Wn` / `Mn` for day/week/month `n`. |
| **Rolling / bracket retention** | Active in period `k` **or later**. Non-increasing in `k`. |
| **Unbounded / cumulative retention** | Active in *any* period from `0` to `k`. Non-decreasing in `k`. |
| **Cohort size** | Denominator: the number of entities that entered in period 0. |
| **Churned** | No qualifying activity in the period (a *pause* for activity-based products; a *cancel* for subscriptions). |
| **Resurrected / returned** | Was inactive in period `k−1` and active in period `k`. |
| **Resurrection rate** | Resurrected entities ÷ inactive entities in the prior period. |
| **DAU / WAU / MAU** | Distinct active entities in the trailing 1 / 7 / 30 days. |
| **Stickiness** | `DAU / MAU`. The canonical product-engagement ratio. |
| **Quick ratio** | `(new + resurrected) / churned` — growth efficiency. `> 4` is the rule of thumb for healthy SaaS. |
| **Gross revenue retention (GRR)** | Revenue retained from a cohort, excluding expansion. ≤ 100 % by construction. |
| **Net revenue retention (NRR)** | Revenue retained **including** expansion. Can exceed 100 %. |
| **Survivorship bias** | Averaging a retention column that silently excludes cohorts too young to have that cell. |
| **Censoring** | A cell that is not yet observable because the cohort is too young. |
| **Activity event** | The repeatable event that defines "active" (session, order, purchase, sign-in). |
| **Entry event** | The one-time event that defines the cohort (signup, first purchase, activation). |

### 1.2 The four definitions, on one payload

Take a cohort of 10 users and their activity weeks, `W0 … W4`:

```
user   W0  W1  W2  W3  W4
 1      ●   ●   ●   ●   ●
 2      ●   ●   ●   ·   ●
 3      ●   ●   ·   ·   ·
 4      ·   ●   ·   ●   ·
 5      ·   ·   ·   ·   ·
 ─────────────────────────
active 3   4   2   2   2        (of 5 users)
```

| Definition | W0 | W1 | W2 | W3 | W4 |
|------------|----|----|----|----|----|
| **Classic** (`exactly Wk`) | 3/5 = 60 % | 4/5 = 80 % | 2/5 = 40 % | 2/5 = 40 % | 2/5 = 40 % |
| **Rolling** (`Wk` or later) | 4/5 = 80 % | 4/5 = 80 % | 3/5 = 60 % | 3/5 = 60 % | 2/5 = 40 % |
| **Unbounded** (`any W≤k`) | 3/5 = 60 % | 4/5 = 80 % | 4/5 = 80 % | 4/5 = 80 % | 4/5 = 80 % |

Read the three rows and note:

- **Classic** is the only one that can *rise* (W1 = 80 % > W0 = 60 %) — because user 4 skipped W0.
- **Rolling** is the only one that is monotone (non-increasing). That is why it is the one you put
  in a financial model.
- **Unbounded** is monotone the other way and asymptotes to the "ever retained" share — 80 % here,
  which is "4 of 5 users ever came back".
- All three agree at `k = 0` only if everybody did the entry event in period 0, which is rarely
  true (see §4.3).

> **Senior note.** `rolling` and `classic` differ by exactly the set of users who skipped period `k`
> and returned later. In consumer apps, a *classic* W2 dip with a flat *rolling* curve is the
> signature of a weekly-habit product (people return but not every week). A *rolling* curve that is
> flat while the classic curve oscillates is the signature of a monthly-billing product. Reading the
> two together tells you the cadence of your product, which no single number can.

### 1.3 The denominator question

```
                    all signups in period 0
                    ├──────────────┴──────────────┤
       never activated                     activated (≥1 event ever)
       ├──────────┤                        ├──────────────────────┤
    retained?                              retained in period k
    (always "no")                          ├───────────┤
                                        active in k
```

Two legitimate retention metrics:

```
retention_all_signups   = active_in_k / all_signups_in_0      ← "did the cohort stay?"
retention_activated     = active_in_k / activated_in_0       ← "do the users who started, stay?"
activation_rate         = activated_in_0 / all_signups_in_0   ← the diagnostic in between
```

For the ShopStream sample: 10 signups, 7 ever activated, so `activation_rate = 70 %`. Every
retention number you report should say which denominator it uses.

### 1.4 DAU / WAU / MAU and stickiness

These are **rolling-window distinct counts**, not cohort metrics:

```sql
-- As of a given date D:
DAU(D) = COUNT(DISTINCT entity) WHERE activity BETWEEN D      AND D
WAU(D) = COUNT(DISTINCT entity) WHERE activity BETWEEN D-6    AND D
MAU(D) = COUNT(DISTINCT entity) WHERE activity BETWEEN D-29   AND D
```

`DAU/MAU` is the ratio of two of them. Two conventions exist and they give different answers:

| Convention | Formula | When to use |
|------------|---------|-------------|
| **Daily ratio** | `DAU(D) / MAU(D)` for each day | a time series; spiky, cheap to compute |
| **Monthly stickiness** | `AVG(DAU over the month) / MAU(at month end)` | the single headline number |

Report the monthly convention as "stickiness"; report the daily ratio as a *chart*. Comparing them
without saying which is which is a classic source of "our stickiness is 25 %" / "no, it's 12 %"
arguments.

### 1.5 Retention versus churn versus return

```
period:        W0     W1     W2     W3
user 2:        ●      ●      ●      ·
user 4:        ·      ●      ·      ●         ← returned (resurrected) in W3

classic retention W1 = 80 %        (4 of 5 active)
churn              W1 = 20 %        (1 of 5 inactive)
resurrection       W3 = 1 of 2 inactive-at-W2 → 50 %
```

Three rules:

1. `churn = 1 − classic retention` **only** if activity is mandatory. For a subscription product,
   `churn` means cancelled, and there is a separate "paused/inactive" state.
2. **A return is not a new user.** Counting a resurrected user as "new" inflates growth twice.
3. **Quick ratio** is the summary that separates churn from growth:
   `(new + resurrected) / churned`. A ratio of 1 means you are running to stand still.

## 2. Why it matters / what problem it solves

Retention is the variable that determines everything downstream of it:

| Decision | Needs | Because |
|----------|-------|---------|
| How much can we spend on acquisition (CAC ceiling)? | cohort retention curve | `LTV = ARPU × Σ retention(k) × k`, so the curve *is* the LTV multiplier |
| Is the product good? | vertical cohort comparison | new cohorts flatter than old = improving product |
| Which channel to scale? | retention by acquisition channel | cheap traffic with 10 % D30 is more expensive than expensive traffic with 40 % |
| Which week to intervene? | the shape of the curve | a cliff at W2 means the "second visit" experience is broken |
| Are we growing? | quick ratio / NRR | revenue growth with NRR < 100 % is a leaky bucket |
| Is engagement healthy? | DAU/MAU | a low ratio means infrequent-use product; the target is not the same for a bank and a game |
| Did the release help? | cohort-aligned comparison around the release date | only aligned index comparison is fair |

Costs of ambiguity, from real teams:

- **The definition war.** Marketing reported "month-3 retention of 41 %" (rolling, including
  everyone who ever logged in later), Finance reported "22 %" (classic month-3 logins), and Product
  reported "63 %" (unbounded: "somebody came back at some point"). All three were correct for their
  definition, all three were in the same board deck, and the deck was wrong. Fixing the definition
  took one hour; fixing the trust took a quarter.
- **The survivorship average.** A team averaged a day-30 retention column at a moment when only 3 of
  14 cohorts were old enough. Because the young cohorts had `NULL` and SQL's `AVG` skips `NULL`, the
  reported number was computed over the 3 oldest (best) cohorts only, and it was 18 points too high.
- **The denominator drift.** A "retention" model divided by users *active in the prior period*
  instead of cohort size, so the curve stopped falling — mathematically guaranteed, behaviourally
  meaningless, and it took two weeks to notice because the shape looked "healthier".

## 3. The pattern / technique explained

### 3.1 Classic N-period retention — the template

```sql
WITH
-- 1. Entry: one row per entity, with its cohort day and cohort size contribution
entry AS (
    SELECT user_id, <entry_event_time>::date AS entry_date,
           (<entry_event_time>::date - DATE :cohort_start) AS entry_day_index
    FROM   <entry_source>
    WHERE  <filters>
),
-- 2. Activity: one row per (entity, activity day) — DEDUPLICATED
activity AS (
    SELECT DISTINCT user_id, <activity_event_time>::date AS activity_date FROM <activity_source>
),
-- 3. Align the two on the integer day index
aligned AS (
    SELECT e.user_id,
           (a.activity_date - e.entry_date) AS day_index
    FROM   entry e JOIN activity a USING (user_id)
)
-- 4. One row per day_index — this IS the retention curve
SELECT day_index,
       COUNT(DISTINCT user_id)                                                    AS active_users,
       (SELECT COUNT(*) FROM entry)                                               AS cohort_size,
       ROUND(100.0 * COUNT(DISTINCT user_id) / (SELECT COUNT(*) FROM entry), 1)   AS retention_pct
FROM   aligned
GROUP  BY day_index
ORDER  BY day_index;
```

Three things to note:

- `day_index = 0` is the entry day. Its retention is the **activation rate within day 0**, which is
  *not* 100 % unless the entry event is itself an activity event.
- The `SELECT COUNT(*) FROM entry` subquery is a cohort-size scalar. In a multi-cohort version,
  join it or use a window (`SUM(...) OVER ()`).
- **Deduplicate activity first.** If you skip step 2, a user with 5 events on day 3 contributes 5 to
  `COUNT(DISTINCT user_id)` — harmless — but 5 to any non-distinct cell, and it inflates the row
  count of everything downstream.

### 3.2 Rolling and unbounded retention

```sql
-- Rolling: they were around at day k or later
SELECT day_index, active_users,
       SUM(active_users) OVER (ORDER BY day_index DESC
                               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rolling_users
FROM   classic_curve
ORDER  BY day_index;
```

Unbounded is the same with `ASC`:

```sql
SELECT day_index, active_users,
       SUM(active_users) OVER (ORDER BY day_index
                               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS unbounded_users
FROM   classic_curve;
```

Two absolutely critical caveats:

1. **This only works if `classic_curve` contains a row for every day index**, including indices with
   zero active users and indices that are censored. `SUM ... OVER (ORDER BY day_index DESC)` over a
   *sparse* curve gives a rolling total of the wrong rows (chapter 03's sparse-window trap, in
   cumulative form). Densify the day index first: `generate_series(0, :max_day)`.
2. **Rolling must be computed as "ever active at ≥ k", which is a set union, not a sum.** Summing
   `active_users` is correct only because the day-index sets are disjoint by construction (a user
   active on day 3 and day 5 is counted in the day-3 cell and the day-5 cell, but the *cells* are
   per-day so the union size is not the sum). **This is why the base curve must be a per-day
   distinct count.** If a user appears in the day-3 cell and again in the day-5 cell, summing gives
   2 — correct for "user-days", wrong for "distinct users still retained". So:

```sql
-- ✅ The correct rolling retention: distinct entities active in ANY day >= k
WITH firsts AS (SELECT MIN(day_index) AS first_index FROM classic_curve WHERE active_users > 0),
     ...
-- Equivalently, and more robustly:
SELECT k AS day_index,
       (SELECT COUNT(DISTINCT user_id) FROM aligned WHERE day_index >= k) AS rolling_active,
       (SELECT COUNT(DISTINCT user_id) FROM entry)                        AS cohort_size
FROM   generate_series(0, :max_day) AS k;
```

   The window-sum version is a **performance shortcut that is only valid when no user can be active
   in two different periods** — which is true for "distinct users" only if you define the cell as
   "active in exactly day k" *and* you are summing distinct counts, in which case the sum counts a
   user once per active day and **overstates** rolling retention. For most products, users are active
   on multiple days, so summing daily distinct counts **does** double-count. Use the
   `COUNT(DISTINCT)`-per-`k` form (or a set-union / sketch merge) and reserve the window sum for
   *event* counts, which are additive by nature.

> **This is the single most common retention bug in production.** `SUM(daily_active_users) OVER
> (ORDER BY day DESC)` looks like rolling retention, is a one-liner, is fast, and is **wrong**
> whenever a user is active on more than one day of the tail. If your rolling curve ever exceeds
> 100 % of the cohort, this is why.

### 3.3 The retention curve vs. the point-in-time number

| Output | Shape | Used for |
|--------|-------|----------|
| The **curve** (`retention by day_index`, dense) | one row per day index | LTV modelling, diagnosing the product, comparing cohorts |
| A **point-in-time number** (`D1`, `D7`, `D30`) | one scalar per cohort | KPI tracking, board reporting |
| **DAU/WAU/MAU** | a time series of rolling windows | engagement health, capacity planning |
| **Stickiness** | one scalar | product categorisation |

A point-in-time number is a single cell of the curve. If the curve is wrong, every point-in-time
number derived from it is wrong in the same way.

### 3.4 Censoring, and how to get it wrong in three different ways

Given as-of date `A` and entry date `E`, the maximum observable day index is
`D_max(E) = A − E`. A cell at index `k` is censored if `k > D_max(E)`.

| Wrong handling | What happens |
|----------------|--------------|
| Emit `0` for censored cells | the cohort looks like it churned; averaging drags the curve down at high `k` |
| Omit the row entirely | `AVG` over a missing row ≠ `AVG` over a zero, and the cohort size in the denominator becomes wrong |
| Emit `NULL` but still `AVG` the column | `AVG` skips `NULL`, so the average is computed only over old cohorts → **survivorship bias** |
| Emit `NULL` and count cohorts | ✅ correct |

```sql
SELECT k,
       SUM(active_users)                                     AS active_users,
       SUM(cohort_size)                                       AS members,
       COUNT(*)                                               AS cohorts_eligible,   -- ← the antidote
       ROUND(100.0 * SUM(active_users) / NULLIF(SUM(cohort_size), 0), 1) AS retention_pct
FROM   cells
WHERE  k <= max_observable_index                              -- ← censoring filter
GROUP  BY k
ORDER  BY k;
```

Publish `cohorts_eligible` next to `retention_pct`. It is the single most useful column in a
retention table because it tells the reader how much to trust the number.

## 4. Step-by-step worked example (with sample data and expected output tables)

Using the shared schema from `01-ranking.md`: 10 users, 7 of whom ever fired an event.

### 4.1 Example 1 — DAU, WAU, MAU and stickiness from `events`

**Problem statement.** Produce a daily engagement panel: DAU, rolling WAU, rolling MAU, and
stickiness.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
spine AS (
    SELECT d::date AS d
    FROM   generate_series(DATE '2024-01-05', (SELECT as_of FROM params), INTERVAL '1 day') d
),
daily AS (
    SELECT d.d,
           COUNT(DISTINCT e.user_id) FILTER (WHERE e.event_time::date = d.d)                     AS dau,
           COUNT(DISTINCT e.user_id) FILTER (WHERE e.event_time::date BETWEEN d.d - 6  AND d.d)  AS wau,
           COUNT(DISTINCT e.user_id) FILTER (WHERE e.event_time::date BETWEEN d.d - 29 AND d.d)  AS mau
    FROM   spine d
    LEFT   JOIN events e ON e.event_time::date <= d.d AND e.event_time::date >= d.d - 29
    GROUP  BY d.d
)
SELECT d, dau, wau, mau,
       ROUND(100.0 * dau / NULLIF(mau, 0), 1) AS stickiness_pct,
       ROUND(100.0 * wau / NULLIF(mau, 0), 1) AS wau_mau_pct
FROM   daily
ORDER  BY d;
```

Selected rows:

| d | dau | wau | mau | stickiness_pct | wau_mau_pct |
|---|----:|----:|----:|---------------:|------------:|
| 2024-01-05 | 1 | 1 | 1 | 100.0 | 100.0 |
| 2024-01-07 | 1 | 2 | 2 |  50.0 | 100.0 |
| 2024-01-12 | 1 | 2 | 2 |  50.0 | 100.0 |
| 2024-01-16 | 1 | 3 | 3 |  33.3 | 100.0 |
| 2024-01-26 | 1 | 1 | 3 |  33.3 |  33.3 |
| 2024-02-02 | 2 | 2 | 4 |  50.0 |  50.0 |
| 2024-02-16 | 1 | 2 | 5 |  20.0 |  40.0 |
| 2024-03-11 | 1 | 2 | 4 |  25.0 |  50.0 |

Verify one row by hand. On **2024-01-16**: MAU window = 2023-12-18 … 2024-01-16, containing user 1
(01-05, 01-06, 01-12), user 2 (01-07, 01-08, 01-14), user 3 (01-15, 01-16) → `mau = 3`. WAU window =
2024-01-10 … 01-16: user 1 (01-12), user 2 (01-14), user 3 (01-15, 01-16) → `wau = 3`. DAU on 01-16 =
user 3 only → `dau = 1`. So `stickiness = 1/3 = 33.3 %`.

On **2024-02-16**: MAU window = 2024-01-18 … 02-16: user 1 (01-19, 01-26), user 2 (01-21, 02-04),
user 3 (01-22, 02-05), user 4 (02-01, 02-02, 02-09, 02-16), user 5 (02-14, 02-15) → `mau = 5`. WAU
window = 02-10 … 02-16: user 4 (02-16), user 5 (02-14, 02-15) → `wau = 2`. DAU on 2024-02-16 = user 4
only → `dau = 1`. So `stickiness = 1/5 = 20 %`.

**The monthly (headline) version:**

```sql
WITH months AS (
    SELECT date_trunc('month', e.event_time)::date AS m, e.user_id, e.event_time::date AS d
    FROM   events e
), per_day AS (
    SELECT m, d, COUNT(DISTINCT user_id) AS dau FROM months GROUP BY m, d
), per_month AS (
    SELECT m, COUNT(DISTINCT user_id) AS mau FROM months GROUP BY m
), month_days AS (
    SELECT m, COUNT(*) AS days_in_month
    FROM   (SELECT generate_series(DATE '2024-02-01', DATE '2024-03-31',
                                   INTERVAL '1 day')::date AS d) c
    JOIN   per_month pm ON pm.m = date_trunc('month', c.d)::date
    GROUP  BY m
)
SELECT p.m,
       ROUND(SUM(pd.dau)::numeric / md.days_in_month, 2)                  AS avg_dau,
       p.mau,
       ROUND(100.0 * SUM(pd.dau) / md.days_in_month / NULLIF(p.mau, 0), 1) AS stickiness_pct
FROM   per_month p
JOIN   month_days md USING (m)
LEFT   JOIN per_day pd USING (m)
GROUP  BY p.m, p.mau, md.days_in_month
ORDER  BY p.m;
```

| m | active days | sum(DAU) | days_in_month | avg_dau | mau | stickiness_pct |
|---|------------:|---------:|--------------:|--------:|----:|---------------:|
| 2024-02-01 | 10 | 11 | 29 | 0.38 | 5 | 7.6 |
| 2024-03-01 | 10 | 11 | 31 | 0.35 | 3 | 11.8 |

Those stickiness numbers are absurdly low — and that is the honest answer for a 7-user sample. The
*shape* is what you are learning: `avg_dau` is tiny because the product is a low-frequency shopping
companion, and `mau` falls in March because the February users went quiet. On real data, a
stickiness above 20 % puts you in "habit-forming" territory for consumer apps; above 50 % is
exceptional. Always report the ratio with its `mau` denominator, its `days_in_month` divisor and the
number of users behind it. Note also that **February's 29 days matter**: dividing a February sum by
31 understates avg_dau by 6 % and makes month-over-month comparisons wrong in a way that looks like a
trend.

### 4.2 Example 2 — the classic N-day curve, days since signup

**Problem statement.** Day-0/1/7/14/28 retention with signup as day 0, on all 10 signups.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
entry AS (
    SELECT u.user_id, u.signup_date,
           ((SELECT as_of FROM params) - u.signup_date) AS max_observable_day
    FROM   users u
),
activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date FROM events e
),
aligned AS (
    SELECT e.user_id,
           (a.activity_date - e.signup_date) AS day_index
    FROM   entry e JOIN activity a USING (user_id)
),
curve AS (
    SELECT g.k AS day_index,
           COUNT(DISTINCT a.user_id)  AS active_users,
           (SELECT COUNT(*) FROM entry) AS cohort_size
    FROM   generate_series(0, 35) AS g(k)
    LEFT   JOIN aligned a ON a.day_index = g.k
    GROUP  BY g.k
)
SELECT day_index, active_users, cohort_size,
       ROUND(100.0 * active_users / cohort_size, 1) AS retention_pct
FROM   curve
WHERE  day_index IN (0, 1, 2, 3, 4, 6, 7, 8, 14, 15, 21, 28, 29)
ORDER  BY day_index;
```

| day_index | active_users | cohort_size | retention_pct |
|----------:|-------------:|------------:|--------------:|
|  0 | 6 | 10 | 60.0 |
|  1 | 7 | 10 | 70.0 |
|  2 | 1 | 10 | 10.0 |
|  3 | 1 | 10 | 10.0 |
|  4 | 1 | 10 | 10.0 |
|  6 | 1 | 10 | 10.0 |
|  7 | 6 | 10 | 60.0 |
|  8 | 2 | 10 | 20.0 |
| 14 | 3 | 10 | 30.0 |
| 15 | 1 | 10 | 10.0 |
| 21 | 2 | 10 | 20.0 |
| 28 | 2 | 10 | 20.0 |
| 29 | 1 | 10 | 10.0 |

Let me walk the derivation, because every number here is a lesson:

| day_index | who is active on exactly that day after their own signup | why |
|-----------|----------------------------------------------------------|-----|
| 0 | users 1, 2, 3, 4, 5, 6 | they fired an event on their signup date; user 7 did not |
| 1 | users 1, 2, 3, 4, 5, 6, **7** | user 7 signed up 2024-03-03 (Sunday) and first acted 2024-03-04 |
| 2, 3, 4 | user 7 only | user 7's 2024-03-05, 03-06, 03-07 | 
| 6 | user 7 only | `03-09 − 03-03 = 6` |
| 7 | users 1, 2, 3, 5, 6, 7 | 01-12, 01-14, 01-22, 02-21, 03-09, 03-10 |
| 8 | users 4, 7 | `02-09 − 02-01 = 8` and `03-11 − 03-03 = 8` |
| 14 | users 1, 2, 5 | 01-19, 01-21, 02-28 (user 3's `01-15 + 14 = 01-29` has no activity) |
| 15 | user 4 | `02-16 − 02-01 = 15` |
| 21 | users 1, 3 | `01-26 − 01-05 = 21` and `02-05 − 01-15 = 21` |
| 28 | users 1, 2 | `02-02 − 01-05 = 28` and `02-04 − 01-07 = 28` |
| 29 | user 4 | `03-01 − 02-01 = 29` |

The shape of this curve — **60, 70, 10, 10, 10, 10, 60, 20, …, 30, …, 20, 20, 10** — is the most
important thing in the chapter:

- **Day 0 is only 60 %**, because user 7 did not act on signup day, and users 8, 9, 10 never acted at
  all. A "day-0 retention" of 60 % is really an **activation rate**, and it is the number to attack
  first. If you report day-1 retention as the headline you are hiding a 40 % activation failure.
- **Day 1 (70 %) is higher than day 0 (60 %)**, which is impossible for a monotone metric and
  perfectly normal for a classic N-day metric. It means people come back the next day more reliably
  than they act on the same day.
- **The curve is not smooth**, because the sample is 7 people with idiosyncratic shopping rhythms.
  Real products show this too: a *weekly* product has spikes at days 7, 14, 21, 28 and troughs in
  between; a *monthly* product has a single spike near day 30. **The spike pattern is the product's
  cadence.** If you deploy a release and the day-7 spike vanishes, the habit broke.
- **Days 2–6 are near zero.** For a shopping app this is normal (nobody shops daily); for a social
  app it would be catastrophic. There is no universal "good" retention curve — only a curve compared
  against *your own* previous cohorts.

### 4.3 Example 3 — days since *first activity* (activation cohorts)

**Problem statement.** The signup-based curve above is dragged down by never-activated users. Produce
the same curve for activation cohorts (day 0 = first event ever), then compare.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
first_event AS (
    SELECT e.user_id, MIN(e.event_time::date) AS first_date FROM events e GROUP BY e.user_id
),
activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date FROM events e
),
aligned AS (
    SELECT f.user_id, (a.activity_date - f.first_date) AS day_index
    FROM   first_event f JOIN activity a USING (user_id)
),
curve AS (
    SELECT g.k AS day_index,
           COUNT(DISTINCT a.user_id)                    AS active_users,
           (SELECT COUNT(*) FROM first_event)           AS cohort_size      -- 7, not 10
    FROM   generate_series(0, 35) AS g(k)
    LEFT   JOIN aligned a ON a.day_index = g.k
    GROUP  BY g.k
)
SELECT day_index, active_users, cohort_size,
       ROUND(100.0 * active_users / cohort_size, 1) AS retention_pct
FROM   curve
WHERE  day_index IN (0, 1, 2, 3, 4, 6, 7, 8, 14, 21, 28, 29)
ORDER  BY day_index;
```

| day_index | active_users | cohort_size | retention_activated_pct | retention_all_signups_pct (from §4.2) |
|----------:|-------------:|------------:|------------------------:|-------------------------------------:|
|  0 | 7 | 7 | 100.0 | 60.0 |
|  1 | 7 | 7 | 100.0 | 70.0 |
|  2 | 1 | 7 |  14.3 | 10.0 |
|  3 | 1 | 7 |  14.3 | 10.0 |
|  4 | 0 | 7 |   0.0 | 10.0 |
|  5 | 1 | 7 |  14.3 |  0.0 |
|  6 | 1 | 7 |  14.3 | 10.0 |
|  7 | 6 | 7 |  85.7 | 60.0 |
|  8 | 1 | 7 |  14.3 | 20.0 |
| 14 | 3 | 7 |  42.9 | 30.0 |
| 21 | 2 | 7 |  28.6 | 20.0 |
| 28 | 2 | 7 |  28.6 | 20.0 |
| 29 | 1 | 7 |  14.3 | 10.0 |

Note how the two curves **swap places at day 4 and day 5**. Day 4 in the activated cohort is 0 % and
day 5 is 14.3 %; in the signup cohort it is the reverse (10 % then 0 %), because user 7's 2024-03-07
activity is day 4 *after signup* but day 3 *after first activity*, and their 2024-03-09 activity is
day 6 after signup and day 5 after first activity. **Shifting the origin by two days reorders the
whole curve.** This is not a bug — it is why the entry event must be fixed before any comparison.

The interpretation is the **activation gap**:

```
retention_all_signups(D7) = 60.0 %
retention_activated(D7)   = 85.7 %
activation_rate           = 70.0 %   (7 of 10 signups ever fired an event)
gap                        = 25.7 percentage points   ← "we lose 30 % before they ever start"
```

The senior reading: *the product works for the people who start using it (85.7 % at week 1) and the
company fails to get 30 % of signups to start at all.* The intervention is onboarding, not product
quality. Reporting only the 60 % number would send the team to the wrong problem.

### 4.4 Example 4 — the three definitions side by side

**Problem statement.** For the signup cohorts, compute classic, rolling and unbounded retention at
`N = 7` and `N = 28`, with a censoring column.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
entry AS (
    SELECT u.user_id, u.signup_date,
           ((SELECT as_of FROM params) - u.signup_date) AS max_observable_day
    FROM   users u
),
activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS activity_date FROM events e
),
aligned AS (
    SELECT e.user_id, (a.activity_date - e.signup_date) AS day_index
    FROM   entry e JOIN activity a USING (user_id)
),
defs AS (
    SELECT n AS horizon,
           (SELECT COUNT(*) FROM entry) AS cohort_size,
           (SELECT COUNT(DISTINCT user_id) FROM aligned WHERE day_index =  n) AS classic_users,
           (SELECT COUNT(DISTINCT user_id) FROM aligned WHERE day_index <= n) AS unbounded_users,
           (SELECT COUNT(DISTINCT user_id) FROM aligned WHERE day_index >= n) AS rolling_users,
           (SELECT COUNT(*) FROM entry WHERE max_observable_day >= n)         AS cohorts_eligible
    FROM   (VALUES (7), (28)) AS t(n)
)
SELECT horizon, cohorts_eligible, cohort_size,
       classic_users,   ROUND(100.0 * classic_users   / cohort_size, 1) AS classic_pct,
       rolling_users,   ROUND(100.0 * rolling_users   / cohort_size, 1) AS rolling_pct,
       unbounded_users, ROUND(100.0 * unbounded_users / cohort_size, 1) AS unbounded_pct
FROM   defs
ORDER  BY horizon;
```

| horizon | cohorts_eligible | cohort_size | classic_users | classic_pct | rolling_users | rolling_pct | unbounded_users | unbounded_pct |
|--------:|-----------------:|------------:|--------------:|------------:|--------------:|------------:|----------------:|--------------:|
|  7 | 10 | 10 | 6 | 60.0 | 7 | 70.0 | 7 | 70.0 |
| 28 | 10 | 10 | 2 | 20.0 | 3 | 30.0 | 7 | 70.0 |

Read the `N = 28` row. The **same 10 users on the same day** produce three numbers:

- **classic 20 %** — 2 users (1 and 2) were active on *exactly* day 28 after their own signup.
- **rolling 30 %** — 3 users (1, 2 and 4) were active on day 28 *or later* (user 4's day 29 counts).
- **unbounded 70 %** — 7 users had been active at *some point* in their first 28 days.

A board deck that says "day-30 retention is 20 %" and an investor update that says "70 % of our users
return within 30 days" are both technically true and describe completely different businesses. This
is why the definition belongs in the metric name: `d30_classic_by_signup`,
`d30_rolling_by_signup`, `ever_active_within_30d_by_signup`.

**Which one should you use?**

| Use case | Definition |
|----------|-----------|
| Product habit formation, onboarding diagnosis | **classic** (per-day/per-week cells, plotted as a curve) |
| LTV / unit economics / payback models | **rolling** (monotone → sums and decays cleanly) |
| "Did we ever activate this user?" / activation rate | **unbounded** (or its complement, "never activated") |
| Renewal / subscription contracts | **rolling at the renewal boundary** (active at month 12 or later) |
| Board KPI | pick one and freeze it in the metrics catalogue |

### 4.5 Example 5 — retention by cohort (weekly matrix)

Chapter `04-cohort-analysis.md` builds this in full. The retention-specific point here is that the
retention metric you chose in §4.4 changes the *shape* of the matrix, and that the **rolling** matrix
is the only one you can average confidently.

```sql
WITH cohort AS (
    SELECT u.user_id, date_trunc('week', u.signup_date)::date AS cohort_week FROM users u
), activity AS (
    SELECT DISTINCT e.user_id, date_trunc('week', e.event_time)::date AS activity_week,
           e.event_time::date AS activity_date
    FROM   events e
), idx AS (
    SELECT c.cohort_week, c.user_id,
           ((a.activity_week - c.cohort_week) / 7)::int AS idx
    FROM   cohort c JOIN activity a USING (user_id)
)
SELECT cohort_week,
       COUNT(DISTINCT CASE WHEN idx = 0 THEN user_id END)                            AS w0_classic,
       COUNT(DISTINCT user_id)                                                       AS ever_in_window,
       COUNT(DISTINCT CASE WHEN idx <= 1 THEN user_id END)                           AS w1_unbounded,
       COUNT(DISTINCT CASE WHEN idx >= 1 THEN user_id END)                           AS w1_rolling
FROM   idx
GROUP  BY cohort_week
ORDER  BY cohort_week;
```

| cohort_week | w0_classic | ever_in_window | w1_unbounded | w1_rolling |
|-------------|-----------:|---------------:|-------------:|-----------:|
| 2024-01-01 | 2 | 2 | 2 | 2 |
| 2024-01-15 | 1 | 1 | 1 | 1 |
| 2024-01-29 | 1 | 1 | 1 | 1 |
| 2024-02-12 | 1 | 1 | 1 | 1 |
| 2024-02-26 | 1 | 2 | 2 | 2 |

The last row is the interesting one: the 2024-02-26 cohort has `w0_classic = 1` (only user 6 was
active in week 0) but `w1_unbounded = w1_rolling = 2`, because user 7 (who signed up on the Sunday of
that week and first acted on Monday) and user 6 both show up by week 1. In a **classic** world this
cohort "w0 = 50 %" looks like a bad cohort; in an **unbounded/rolling** world it looks perfect. The
cohort is the same; only the definition changed.

### 4.6 Example 6 — churn, returns, and the quick ratio

**Problem statement.** For weekly periods, produce new / churned / resurrected / active, and the
quick ratio.

```sql
WITH params AS (SELECT DATE '2024-04-30' AS as_of),
weeks AS (
    SELECT generate_series(DATE '2024-01-01', (SELECT as_of FROM params), INTERVAL '7 days')::date AS wk
),
user_week AS (
    SELECT DISTINCT e.user_id, date_trunc('week', e.event_time)::date AS wk FROM events e
),
states AS (
    SELECT u.user_id, w.wk,
           (uw.user_id IS NOT NULL)                          AS active,
           LAG(uw.user_id IS NOT NULL) OVER (PARTITION BY u.user_id ORDER BY w.wk) AS was_active,
           MIN(CASE WHEN uw.user_id IS NOT NULL THEN w.wk END)
               OVER (PARTITION BY u.user_id)                 AS first_active_wk
    FROM   (SELECT DISTINCT user_id FROM events) u
    CROSS  JOIN weeks w
    LEFT   JOIN user_week uw ON uw.user_id = u.user_id AND uw.wk = w.wk
)
SELECT wk,
       COUNT(*)           FILTER (WHERE active AND first_active_wk = wk)                       AS new,
       COUNT(*)           FILTER (WHERE active AND NOT was_active AND first_active_wk <> wk)   AS resurrected,
       COUNT(*)           FILTER (WHERE NOT active AND was_active)                             AS churned,
       COUNT(*)           FILTER (WHERE active)                                                AS active,
       ROUND( (COUNT(*) FILTER (WHERE active AND first_active_wk = wk)
               + COUNT(*) FILTER (WHERE active AND NOT was_active AND first_active_wk <> wk))::numeric
              / NULLIF(COUNT(*) FILTER (WHERE NOT active AND was_active), 0), 2)               AS quick_ratio
FROM   states
GROUP  BY wk
ORDER  BY wk;
```

| wk | new | resurrected | churned | active | quick_ratio |
|----|----:|------------:|--------:|-------:|------------:|
| 2024-01-01 | 2 | 0 | 0 | 2 | `NULL` |
| 2024-01-08 | 0 | 0 | 0 | 2 | `NULL` |
| 2024-01-15 | 1 | 0 | 0 | 3 | `NULL` |
| 2024-01-22 | 0 | 0 | 1 | 2 | 0.00 |
| 2024-01-29 | 1 | 1 | 1 | 3 | 2.00 |
| 2024-02-05 | 0 | 1 | 2 | 2 | 0.50 |
| 2024-02-12 | 1 | 0 | 1 | 2 | 1.00 |
| 2024-02-19 | 0 | 0 | 1 | 1 | 0.00 |
| 2024-02-26 | 1 | 1 | 0 | 3 | `NULL` |
| 2024-03-04 | 1 | 0 | 2 | 2 | 0.50 |
| 2024-03-11 | 0 | 0 | 1 | 1 | 0.00 |
| 2024-03-18 | 0 | 0 | 1 | 0 | 0.00 |

Verify three rows, because they contain the whole logic:

- **2024-01-15** (users 1, 2 and 3 are active): user 3's first-ever activity is 2024-01-15, so
  `new = 1`; nobody who was active in the week of 2024-01-08 is absent, so `churned = 0`; nobody
  was inactive-then-active, so `resurrected = 0`. `quick_ratio` is `NULL` (1/0).
- **2024-01-29** (users 1, 2, 4 active): user 4's first activity is 2024-02-01, which is in the week
  of 2024-01-29 → `new = 1`. User 2 was **not** active in the week of 2024-01-22 but is active now,
  and its first week was 2024-01-01 → `resurrected = 1`. User 3 was active in 2024-01-22 and is not
  active now → `churned = 1`. `quick_ratio = (1 + 1) / 1 = 2.00`.
- **2024-03-04** (users 6 and 7 active): user 7's first activity is 2024-03-04 → `new = 1`. Users 4
  and 5 were active in the week of 2024-02-26 and are not active now → `churned = 2`.
  `quick_ratio = 1 / 2 = 0.50`. This is a shrinking week: two laps for one new user.

The three ratios that matter:

- **new / active** — how much of the active base is brand new (acquisition dependence),
- **resurrected / active** — how much comes back from the dead (the cheapest growth there is),
- **churned / active** — the leak rate.

Two invariants worth testing (they catch almost every bug in this query):

```
active(this week) = active(prev week) − churned(this week) + resurrected(this week) + new(this week)
quick_ratio >= 4 is "healthy" for SaaS; between 2 and 4 is "growing but fragile"; < 1 is shrinking
```

Check the invariant on 2024-01-29: `active(prev) = 2` (2024-01-22), `churned = 1`,
`resurrected = 1`, `new = 1` → `2 − 1 + 1 + 1 = 3` = `active`. ✅

Note the `NULL` quick ratios: dividing by zero churn has no defined answer. Emitting `NULL` (rather
than `∞` or a giant number) is correct and keeps the chart readable. In this sample the product only
ever reached a quick ratio of 2.00 in its best week and 0.00 in its worst — a leaky bucket with
`n = 7`. That is the honest picture, and it is exactly the picture a raw DAU chart hides.

> **`was_active` for the first week is `NULL`, not `false`.** `LAG` returns `NULL` on the first row of
> each partition. Writing `NOT was_active` with a `NULL` yields `NULL`, which is neither true nor
> false — and in a `FILTER` clause, `NULL` behaves like `false`, which is what you want here. But in a
> `CASE WHEN NOT active THEN 'churned' ELSE 'active' END` you would silently label a first-week
> inactive user as "active". Always `COALESCE(was_active, false)` when you need a two-valued
> predicate.

### 4.7 Example 7 — weekly retention for a subscription-ish view (NRR/GRR sketch)

For a subscription business, entity retention is the wrong metric. Revenue retention is:

```sql
WITH base AS (
    SELECT date_trunc('month', o.order_date)::date AS m, o.user_id, SUM(o.revenue) AS revenue
    FROM   order_revenue o WHERE o.status <> 'cancelled'
    GROUP  BY 1, 2
), first_month AS (
    SELECT user_id, MIN(m) AS cohort_month FROM base GROUP BY user_id
), labelled AS (
    SELECT b.m, b.user_id, b.revenue, f.cohort_month,
           ((EXTRACT(YEAR FROM b.m) * 12 + EXTRACT(MONTH FROM b.m))
          - (EXTRACT(YEAR FROM f.cohort_month) * 12 + EXTRACT(MONTH FROM f.cohort_month)))::int AS age
    FROM   base b JOIN first_month f USING (user_id)
)
SELECT cohort_month, age,
       SUM(revenue)                                                 AS revenue,
       SUM(CASE WHEN age = 0 THEN revenue END)                       AS revenue_at_age_0,
       ROUND(100.0 * SUM(revenue) / NULLIF(SUM(CASE WHEN age = 0 THEN revenue END), 0), 1)
                                                                     AS nrr_pct_approx
FROM   labelled
GROUP  BY cohort_month, age
ORDER  BY cohort_month, age;
```

Two warnings that matter more than the numbers here:

1. **GRR uses the *floor*, NRR uses the total.** `GRR < 100 %` by construction (a cohort cannot
   retain more revenue than it started with *excluding expansion*). If your GRR is above 100 %, you
   included expansion — that is NRR.
2. **You cannot compute NRR from order data alone.** NRR needs *contracted* recurring revenue
   (seats, plan price, term), not order totals. On `orders`, expansion and renewal are
   indistinguishable. This is why every serious SaaS warehouse has a
   `subscription_events` / `mrr_movements` fact table (chapter 02, §4.10).

### 4.8 Example 8 — the retention-curve comparison that answers "is the product improving?"

**Problem statement.** Compare two adjacent cohorts on the same aligned index and decide whether the
release helped. This is the single most valuable retention query in a product company.

```sql
WITH first_event AS (
    SELECT e.user_id, MIN(e.event_time::date) AS first_date FROM events e GROUP BY e.user_id
), activity AS (
    SELECT DISTINCT e.user_id, e.event_time::date AS d FROM events e
), aligned AS (
    SELECT f.user_id, date_trunc('month', f.first_date)::date AS cohort_month,
           (a.d - f.first_date) AS day_index
    FROM   first_event f JOIN activity a USING (user_id)
), cells AS (
    SELECT cohort_month, day_index, COUNT(DISTINCT user_id) AS active_users
    FROM   aligned GROUP BY 1, 2
), sizes AS (
    SELECT date_trunc('month', first_date)::date AS cohort_month, COUNT(*) AS cohort_size
    FROM   first_event GROUP BY 1
)
SELECT c.day_index,
       MAX(CASE WHEN c.cohort_month = DATE '2024-01-01' THEN c.active_users END) AS jan_active,
       MAX(CASE WHEN c.cohort_month = DATE '2024-02-01' THEN c.active_users END) AS feb_active,
       MAX(CASE WHEN c.cohort_month = DATE '2024-03-01' THEN c.active_users END) AS mar_active,
       MAX(s1.cohort_size) AS jan_size, MAX(s2.cohort_size) AS feb_size, MAX(s3.cohort_size) AS mar_size,
       ROUND(100.0 * MAX(CASE WHEN c.cohort_month = DATE '2024-02-01' THEN c.active_users END)
                   / NULLIF(MAX(s2.cohort_size), 0), 1) AS feb_pct,
       ROUND(100.0 * MAX(CASE WHEN c.cohort_month = DATE '2024-03-01' THEN c.active_users END)
                   / NULLIF(MAX(s3.cohort_size), 0), 1) AS mar_pct
FROM   cells c
LEFT   JOIN sizes s1 ON s1.cohort_month = DATE '2024-01-01'
LEFT   JOIN sizes s2 ON s2.cohort_month = DATE '2024-02-01'
LEFT   JOIN sizes s3 ON s3.cohort_month = DATE '2024-03-01'
GROUP  BY c.day_index
ORDER  BY c.day_index;
```

The reusable shape is the **cohort × index comparison**, and the rule for reading it:

| Pattern | Interpretation |
|---------|----------------|
| newer cohort is **higher at every index** | genuine improvement; ship it |
| newer cohort is higher at index 0 only | better acquisition/onboarding, same product |
| newer cohort is higher after index `k` only | a specific intervention (a re-engagement email) |
| curves cross | mix shift — segment by channel/platform/geo before concluding anything |
| one curve is above because of a single spike | check the cohort size; probably noise |

On our sample, February (5 activated users) and March (2 activated users) are far too small for this
comparison — which is itself the lesson: **run the comparison monthly with thousands per cohort, or
run it weekly with hundreds, but never on cohorts of ten.**

## 5. Reusable template

### 5.1 Classic N-day retention (the deliverable)

```sql
WITH params AS (SELECT DATE :as_of AS as_of),                       -- freeze the as-of date!
entry AS (                                                          -- 1. one row per entity
    SELECT <entity_id>, <entry_event_time>::date AS entry_date,
           ((SELECT as_of FROM params) - <entry_event_time>::date) AS max_observable_day
    FROM   <entry_source> WHERE <filters>
),
activity AS (                                                       -- 2. dedupe
    SELECT DISTINCT <entity_id>, <activity_event_time>::date AS activity_date FROM <activity_source>
),
aligned AS (                                                        -- 3. align
    SELECT e.<entity_id>, (a.activity_date - e.entry_date) AS day_index
    FROM   entry e JOIN activity a USING (<entity_id>)
),
curve AS (                                                          -- 4. dense curve
    SELECT g.k AS day_index, COUNT(DISTINCT a.<entity_id>) AS active_users
    FROM   generate_series(0, :max_day) g(k)
    LEFT   JOIN aligned a ON a.day_index = g.k
    GROUP  BY g.k
)
SELECT cu.day_index,
       cu.active_users,
       (SELECT COUNT(*) FROM entry)                                                  AS cohort_size,
       ROUND(100.0 * cu.active_users / (SELECT COUNT(*) FROM entry), 1)              AS retention_pct,
       (SELECT COUNT(*) FROM entry WHERE max_observable_day >= cu.day_index)         AS cohorts_eligible
FROM   curve cu
ORDER  BY cu.day_index;
```

### 5.2 All three definitions in one pass

```sql
SELECT n AS horizon,
       (SELECT COUNT(*) FROM entry)                                 AS cohort_size,
       (SELECT COUNT(DISTINCT <entity_id>) FROM aligned WHERE day_index =  n) AS classic_users,
       (SELECT COUNT(DISTINCT <entity_id>) FROM aligned WHERE day_index >= n) AS rolling_users,
       (SELECT COUNT(DISTINCT <entity_id>) FROM aligned WHERE day_index <= n) AS unbounded_users,
       (SELECT COUNT(*) FROM entry WHERE max_observable_day >= n)     AS cohorts_eligible
FROM   (VALUES (:n1), (:n2), (:n3)) AS t(n);
```

### 5.3 DAU / WAU / MAU / stickiness panel

```sql
SELECT d,
       COUNT(DISTINCT <entity_id>) FILTER (WHERE activity_date = d)                        AS dau,
       COUNT(DISTINCT <entity_id>) FILTER (WHERE activity_date BETWEEN d - 6  AND d)       AS wau,
       COUNT(DISTINCT <entity_id>) FILTER (WHERE activity_date BETWEEN d - 29 AND d)       AS mau,
       ROUND(100.0 * COUNT(DISTINCT <entity_id>) FILTER (WHERE activity_date = d)
                   / NULLIF(COUNT(DISTINCT <entity_id>) FILTER (WHERE activity_date BETWEEN d - 29 AND d), 0),
             1)                                                                            AS stickiness_pct
FROM   <dense_calendar> c
LEFT   JOIN <activity_deduped> a ON a.activity_date BETWEEN c.d - 29 AND c.d
GROUP  BY c.d
ORDER  BY c.d;
```

The performance trick that matters: join the calendar to a **pre-deduplicated**
`(entity_id, activity_date)` table, not to the raw event table. On 10^9 events this is the difference
between 10 minutes and 10 seconds.

### 5.4 New / resurrected / churned / quick ratio

```sql
SELECT wk,
       COUNT(*) FILTER (WHERE active AND first_active = wk)                                AS new,
       COUNT(*) FILTER (WHERE active AND NOT COALESCE(was_active, false)
                                  AND first_active <> wk)                                  AS resurrected,
       COUNT(*) FILTER (WHERE NOT active AND COALESCE(was_active, false))                  AS churned,
       COUNT(*) FILTER (WHERE active)                                                      AS active,
       ROUND((COUNT(*) FILTER (WHERE active AND first_active = wk)
            + COUNT(*) FILTER (WHERE active AND NOT COALESCE(was_active, false)
                                        AND first_active <> wk))::numeric
             / NULLIF(COUNT(*) FILTER (WHERE NOT active AND COALESCE(was_active, false)), 0), 2)
                                                                                           AS quick_ratio
FROM   states
GROUP  BY wk;
```

### 5.5 Censoring helper (works with day, week or month indices)

```sql
-- Days since entry, capped by the as-of date
((SELECT as_of FROM params) - entry_date) AS max_observable_day

-- For a weekly index:
(((SELECT as_of FROM params) - date_trunc('week', entry_date)::date) / 7)::int AS max_observable_week

-- For a monthly index (period-number arithmetic — never INTERVAL arithmetic):
((EXTRACT(YEAR  FROM (SELECT as_of FROM params)) * 12 + EXTRACT(MONTH FROM (SELECT as_of FROM params)))
 - (EXTRACT(YEAR FROM entry_date) * 12 + EXTRACT(MONTH FROM entry_date)))::int AS max_observable_month
```

### 5.6 Test kit

| Test | SQL / assertion | Catches |
|------|-----------------|---------|
| `D0 ≤ 100 %` | `SELECT COUNT(*) FROM curve WHERE retention_pct > 100` → 0 | duplicate join fan-out |
| Rolling is non-increasing | `SELECT COUNT(*) FROM (SELECT rolling, LAG(rolling) OVER (ORDER BY k) p FROM t) x WHERE rolling > p` → 0 | wrong rolling computation (summing distinct counts) |
| Unbounded is non-decreasing | same, reversed | wrong `ORDER BY` direction |
| `unbounded ≥ classic` at every `k` | `WHERE unbounded < classic` → 0 | mis-defined sets |
| `rolling ≥ classic` at every `k` | `WHERE rolling < classic` → 0 | mis-defined sets |
| Cohort sizes sum to the population | `SUM(cohort_size) = (SELECT COUNT(*) FROM entry)` | missing cohorts (left join direction) |
| Censored cells are `NULL` | `WHERE k > max_observable_day AND retention_pct IS NOT NULL` → 0 | fake zeros |
| `cohorts_eligible` published | `WHERE retention_pct IS NOT NULL AND cohorts_eligible IS NULL` → 0 | survivorship bias in the dashboard |
| Curve is dense | `COUNT(*) = :max_day + 1` for a single cohort | sparse windows (chapter 03) |
| State flow balances | `active_now = active_prev − churned + resurrected + new` | churn/return logic |

## 6. Real-world use cases and applications

| Domain | Metric | Definition | Notes |
|--------|--------|-----------|-------|
| **Consumer app** | D1 / D7 / D30 | classic, by install date | the classic funnel; D1 is often the activation rate |
| **Subscription SaaS** | Logo retention by month | rolling (active at month end) | pairs with GRR/NRR |
| **Subscription SaaS** | GRR / NRR | revenue, expansion included/excluded | needs contracted MRR, not orders |
| **E-commerce** | Repeat-purchase rate, days-to-second-order | classic + rolling | "rolling within 90 days" is the standard |
| **Marketplace** | Supply retention (sellers active 90d) | rolling | liquidity depends on the supply side |
| **Fintech** | 30-day active accounts | classic | regulatory reporting often fixes the definition for you |
| **Media / streaming** | Monthly churn, "at-risk" cohorts | rolling + reactivation | win-back campaigns target resurrection |
| **Gaming** | D1/D7/D30 by acquisition channel | classic | channel quality is a retention question, not a CPI question |
| **Telecom** | Postpaid churn | rolling at renewal | contractual boundaries |
| **Health / fitness** | Weekly actives by goal-set cohort | classic | habit formation is a weekly phenomenon |
| **B2B / enterprise** | Seat-level NRR | revenue, seats | expansion via seat growth is the main lever |
| **Data platform** | Dashboard/table adoption retention | rolling | "is the new metric layer being used in week 4?" |
| **Internal tooling** | Engineer tool DAU/WAU | rolling windows | the same math, internalised |

Two production stories:

1. **"Retention fell 15 points after the redesign."** A team compared D7 *classic* retention before
   and after a release, and it dropped from 62 % to 47 %. Investigation showed the redesign changed
   the weekly email cadence: users were still returning, just on different days, so the *exactly
   day 7* cell lost users while the *rolling day 7* cell was unchanged. The right metric for the
   hypothesis ("do users stay?") was rolling; the team had been using classic because it is the
   default in their BI tool. Both numbers existed; nobody had written down which one the KPI used.
2. **"Our DAU/MAU is 45 %, that's great!"** A B2B tool charted `DAU(last day of month) / MAU`. The
   last day of the month was a reporting deadline, so the ratio spiked every month. Switching to
   `AVG(DAU) / MAU` put the true number at 18 % — a completely different product diagnosis
   (occasional-use tool, not daily-driver). The metrics catalogue entry now includes the exact SQL.

## 7. Performance notes and anti-patterns

### 7.1 Cost model

Retention queries are dominated by three operations:

| Operation | Cost | Mitigation |
|-----------|------|-----------|
| **`COUNT(DISTINCT entity_id)`** | the expensive part; cannot be hash-aggregated cheaply | dedupe to `(entity, period)` once and use `COUNT(*)`; or use `APPROX_COUNT_DISTINCT`/HLL |
| **The `BETWEEN d-29 AND d` join** for MAU | a range join per calendar day → `O(days × rows)` if done naively | pre-aggregate to `(entity, activity_date)` and use `RANGE` frames or a period-keyed join |
| **The cohort × index cross join** | `cohorts × indices` groups, each needing a distinct set | reduce the period count (weeks not days, months not weeks) |

The single highest-leverage optimisation in every retention query:

```sql
-- ✅ The spine that makes everything else fast: one row per (entity, period)
CREATE TABLE fct_entity_activity_day AS
SELECT user_id, event_time::date AS activity_date, COUNT(*) AS events
FROM   events
GROUP  BY user_id, event_time::date;
```

With that table, a DAU/WAU/MAU panel is a 3-column self-join over a table whose rows are bounded by
`entities × days_active` — typically 2–3 orders of magnitude smaller than the event table. Every
cohort, retention, funnel and session query in this section benefits from it.

### 7.2 Indexes

```sql
CREATE INDEX ix_fct_activity_entity_date ON fct_entity_activity_day (user_id, activity_date);
CREATE INDEX ix_fct_activity_date_entity ON fct_entity_activity_day (activity_date, user_id);
CREATE INDEX ix_users_signup            ON users (signup_date);
```

Two indexes because the two access patterns are opposite: "per entity, over time" (cohort alignment)
and "per date, across entities" (DAU/WAU/MAU). In a columnar warehouse, cluster/partition the table
by `activity_date` and let the micro-partitions do the DAU work; in a row store, you need both
indexes.

### 7.3 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| "Retention" with no qualifier | three teams, three numbers, one board deck | name the metric: classic/rolling/unbounded + denominator |
| `SUM(dau) OVER (ORDER BY d DESC)` as "rolling retention" | > 100 % when a user is active on multiple days | `COUNT(DISTINCT entity)` where `day_index >= k`, or sketch merge |
| `COUNT(*)` instead of `COUNT(DISTINCT entity)` | retention > 100 % | dedupe the activity spine |
| Denominator = users active last period | the curve can never fall; meaningless | denominator = cohort size at period 0 |
| No censoring | young cohorts look like churn | `max_observable_day` + `NULL` + `cohorts_eligible` |
| `AVG(retention_pct)` over `NULL`-containing rows | survivorship bias | weighted `SUM(active)/SUM(size)` over eligible cohorts |
| Comparing retention *across different cohort ages* | apples to oranges | compare the same index across cohorts |
| Using signup cohorts only | hides a 30 % activation failure | publish both denominators and the activation gap |
| `DAU(end of month)/MAU` called stickiness | spikes on month-end dates | `AVG(DAU)/MAU` |
| Treating resurrected users as new | double-counted growth | separate new / resurrected / churned |
| `NOT was_active` with a `NULL` first value | first-week users mislabelled | `COALESCE(was_active, false)` |
| Retention on a sparse calendar | gaps break `ROWS`-based rolling forms | densify (chapter 03) |
| A single D30 scalar for all use cases | wrong metric for LTV, wrong metric for product | curve for modelling; scalar for KPI |
| Retroactively changing the definition | historical numbers become non-comparable | version the metric; keep both series |

### 7.4 Scale techniques

- **Pre-aggregate to `(entity, period)`.** Non-negotiable. See §7.1.
- **Approximate distinct for DAU/MAU at scale.** `APPROX_COUNT_DISTINCT` (Snowflake, Redshift,
  Databricks, Oracle, SQL Server 2022) or `HLL_COUNT.INIT/MERGE_PARTIAL` (BigQuery). On 10^8+ users,
  exact distinct over 30 rolling windows is prohibitive; HLL sketches merge in `O(periods)`.
- **Materialise the retention curve, not the matrix.** In most warehouses the *matrix* is what the
  product team wants, but the *curve* (one row per cohort × index) is what everything else derives
  from. Store the curve at the finest grain you need and let the BI layer pivot.
- **Roll up periods when cohorts get small.** Daily cohorts with 50 users each produce a heatmap of
  noise. Roll to weekly when daily cells drop below a few hundred, and monthly below that.
- **Incremental retention models are subtly hard.** Today's activity only changes cells at
  `index = today − entry_date`, but a late-arriving *entry* event changes a user's entire row (their
  whole cohort). Run entry-side changes as a full refresh for the affected cohorts, and activity-side
  changes incrementally. Most teams get this backwards.
- **Bound the query with an as-of parameter.** `WHERE activity_date <= :as_of` makes the query
  reproducible and keeps the censoring logic honest. Without it, every run silently includes
  partial data from today.

## 8. Cross-engine differences

### 8.1 The primitives

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Date subtraction → day count | `d2 - d1` (integer) | `DATEDIFF(d2, d1)` / `TIMESTAMPDIFF(DAY,…)` | `DATEDIFF(DAY, d1, d2)` | `d2 - d1` | `DATEDIFF('day', d1, d2)` | `DATE_DIFF(d2, d1, DAY)` |
| Week bucket (Monday) | `date_trunc('week', d)` | `d - INTERVAL WEEKDAY(d) DAY` | `DATETRUNC(week, d)` (2022+, Sunday-based) | `TRUNC(d,'IW')` | `DATE_TRUNC('week', d)` (`WEEK_START` dependent) | `DATE_TRUNC(d, WEEK(MONDAY))` |
| Month bucket | `date_trunc('month', d)` | `DATE_FORMAT(d,'%Y-%m-01')` | `DATETRUNC(month, d)` (2022+) | `TRUNC(d,'MM')` | `DATE_TRUNC('month', d)` | `DATE_TRUNC(d, MONTH)` |
| Calendar spine | `generate_series(a,b,'1 day')` | recursive CTE | `GENERATE_SERIES` (2022+) | `CONNECT BY LEVEL` | `TABLE(GENERATOR(ROWCOUNT=>n))` | `GENERATE_DATE_ARRAY` |
| `FILTER (WHERE …)` aggregates | ✅ | ❌ (`CASE`) | ❌ (`CASE`) | ❌ (`CASE`) | `COUNT_IF` | `COUNTIF` |
| `COUNT(DISTINCT …) FILTER` | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Approximate distinct | ❌ (extension) | ❌ | ✅ `APPROX_COUNT_DISTINCT` (2022+) | ✅ | ✅ | ✅ `APPROX_COUNT_DISTINCT` / `HLL_COUNT.*` |
| `RANGE BETWEEN INTERVAL … PRECEDING` (for rolling MAU) | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ (`UNIX_DATE`) |
| Interval arithmetic on dates | `d + INTERVAL '7 days'` | `DATE_ADD(d, INTERVAL 7 DAY)` | `DATEADD(day, 7, d)` | `d + 7` | `DATEADD(day, 7, d)` | `DATE_ADD(d, INTERVAL 7 DAY)` |
| Boolean type in aggregates | ✅ | `TINYINT(1)` | `BIT` | ❌ (no boolean) | ✅ | ✅ |
| `LEFT JOIN LATERAL` | ✅ | ✅ 8.0.14+ | `CROSS APPLY` | `CROSS APPLY` | `LATERAL` | `LEFT JOIN UNNEST` |
| `QUALIFY` | ❌ | ❌ | ❌ | ✅ 26ai+ | ✅ | ✅ |

### 8.2 Dialect traps specific to retention

- **Boolean handling.** Oracle has no boolean type, so `COUNT(*) FILTER (WHERE active)` becomes
  `SUM(CASE WHEN active = 1 THEN 1 ELSE 0 END)`. MySQL's `TINYINT(1)` works with `SUM()` but
  `COUNT(active)` counts non-`NULL` values including `0` — a classic silent bug.
- **Week start.** This changes *every* weekly retention number by a day or more of content.
  PostgreSQL's `date_trunc('week')` is ISO (Monday); Snowflake's depends on the `WEEK_START`
  session/account parameter (default `0`, legacy Sunday); BigQuery requires `WEEK(MONDAY)`;
  SQL Server's `DATETRUNC(week)` is Sunday-based while `DATEPART(week)` is configurable via
  `SET DATEFIRST`. Pick a convention, set it explicitly, and test it.
- **Month index arithmetic.** Use `year × 12 + month` differences. Oracle's `MONTHS_BETWEEN` returns
  a fraction (`MONTHS_BETWEEN(DATE '2024-02-29', DATE '2024-01-31') = 0.93`), and adding
  `INTERVAL '1 month'` to 2024-01-31 gives 2024-02-29 in some engines and errors or clamps in others.
- **`RANGE BETWEEN INTERVAL` availability.** SQL Server forbids offsets in `RANGE` frames entirely,
  and BigQuery requires a numeric ordering column. For rolling MAU/WAU in those engines, use an
  explicit self-join over a deduplicated activity spine (`a.activity_date BETWEEN d - 29 AND d`) or
  densify and use `ROWS` (see §4.1, which uses the self-join form precisely because it ports
  everywhere).
- **Snowflake's non-ANSI frame defaults** for `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` do not affect the
  aggregates used here, but they bite the moment you add "the value at the start of the cohort"
  columns. Write the frame explicitly.
- **`FILTER` portability.** The `FILTER (WHERE …)` form used throughout this chapter is PostgreSQL,
  SQLite and Oracle-`CASE`-adjacent. For cross-engine SQL, replace every
  `COUNT(DISTINCT x) FILTER (WHERE cond)` with `COUNT(DISTINCT CASE WHEN cond THEN x END)` — it is
  equivalent (`COUNT` ignores the `NULL` padding) and runs on all six engines.

### 8.3 Portability recipe for the whole chapter

```sql
-- Portable classic N-day retention, all six engines
COUNT(DISTINCT CASE WHEN day_index = :n THEN entity_id END)               AS d_n_active
COUNT(DISTINCT CASE WHEN day_index <= :n THEN entity_id END)              AS d_n_unbounded
COUNT(DISTINCT CASE WHEN day_index >= :n THEN entity_id END)              AS d_n_rolling
SUM(CASE WHEN day_index = :n THEN 1 ELSE 0 END)                           -- after dedup: fast
```

Always `date_trunc` upstream, always compute day/week/month differences as integers, always
deduplicate the activity spine, and you never need engine-specific date functions again.

## 9. Exercises — Intermediate (hints only)

1. **DAU/WAU/MAU as of one date.** For 2024-02-16 produce `dau`, `wau`, `mau` and stickiness from
   `events`. Verify against §4.1 (`1`, `2`, `5`, `20 %`).
   *Hint:* three `COUNT(DISTINCT ...) FILTER` clauses over the same deduplicated table.
2. **The classic N-day curve.** Produce the `day_index`-to-`retention_pct` curve for signup cohorts and
   confirm the day-0/1/7/28 values are 60 %, 70 %, 60 %, 20 %.
   *Hint:* `generate_series(0, 35)` joined to `(activity_date − signup_date)`.
3. **Explain day 1 > day 0.** Using `users` and `events`, identify the user responsible and write the
   two-sentence explanation a stakeholder would understand.
   *Hint:* find users whose first event is not on their signup date.
4. **Three definitions, one payload.** Compute classic, rolling and unbounded at `N = 7` and `N = 28`
   and produce the table in §4.4. Which two are monotone and in which direction?
   *Hint:* `day_index = n`, `<= n`, `>= n`.
5. **The activation gap.** Compute `retention_all_signups(D7)`, `retention_activated(D7)` and
   `activation_rate`, and state in one sentence which problem they imply.
   *Hint:* the denominators are 10, 7 and 7 respectively.
6. **Dense the curve.** Why does the classic curve need a `generate_series` spine even though you can
   group the raw data directly? Construct a case where omitting it changes a rolling calculation.
   *Hint:* chapter 03's sparse-window trap; a `SUM ... OVER (ORDER BY k DESC)` over a sparse curve.
7. **Churn and returns.** For weekly periods, produce `new`, `resurrected`, `churned`, `active` and the
   quick ratio. Which week has the first resurrection?
   *Hint:* `LAG` over a months × users spine; `COALESCE(was_active, false)`.
8. **Censoring.** Add `max_observable_day` and `cohorts_eligible` to the curve. At `as_of = 2024-04-01`,
   how many days after signup can the 2024-03-20 signups be evaluated?
   *Hint:* `DATE '2024-04-01' − DATE '2024-03-20' = 12`.
9. **Stickiness, two conventions.** Compute stickiness for February with (a) `DAU(last day)/MAU` and
   (b) `AVG(DAU)/MAU`. Explain in one sentence why they differ.
   *Hint:* the seed data ends on 2024-03-11, so the last day of February has DAU 1 (user 5, 2024-02-28)
   — but the last day of the *month* with activity is 2024-02-28, not 2024-02-29.
10. **Portability pass.** Rewrite §4.4's three-definition query without any `FILTER` clause so it runs
    on MySQL 8 and SQL Server 2012+.
    *Hint:* `COUNT(DISTINCT CASE WHEN day_index = n THEN user_id END)`.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **A single retention metric definition, productionised.** Write the metric spec for exactly one
   retention number for your (hypothetical) product: entity, entry event, activity event, period,
   denominator, definition (classic/rolling/unbounded), horizon, censoring policy, rounding. Then
   implement it as a versioned model plus tests, and list the three dashboard changes it forces.
   *Expected approach:* the spec must be machine-checkable — encode it as a dbt-style model with
   columns `cohort_period, horizon, cohort_size, active_users, retention_pct, eligible_cohorts,
   metric_definition_id`. Show that a second definition (say, rolling) is a *separate*
   `metric_definition_id` in the same table, so both can be charted without rewriting history. This
   is the governance outcome the chapter argues for.
2. **Rolling retention computed correctly at scale.** Implement rolling `D30` retention for 5 × 10^8
   users × 10^9 daily activity rows with ≤ 2 % error and a 2-hour SLA. Your implementation must not
   produce `> 100 %` and must be idempotent under replay.
   *Expected approach:* pre-aggregate to `(user_id, activity_date)`; per (cohort_day, horizon) you
   need "distinct users active on ≥ 1 day in `[entry+k, as_of]`" — note this is a *union* over days,
   so you cannot sum daily distinct counts. Use a **set-union structure**: a bitmap (RoaringBitmap)
   per `(cohort_day, week)` of users active that week, then OR the bitmaps for all weeks ≥ k and
   `POPCOUNT`. Alternatively an HLL/sketch per week merged by union. State the memory bound, the
   incremental update (only the newest week's bitmap changes), and the backfill strategy. Explicitly
   discuss why the naive `SUM(dau) OVER (ORDER BY d DESC)` over-counts and by how much on realistic
   multi-day activity.
3. **DAU/MAU that is not fooled by a single heavy-user day.** Design a stickiness metric that is
   robust to (a) a month-end deadline spike, (b) a handful of power users active every day, and
   (c) a partner integration that fires one event per user per month. Give the SQL and the
   interpretation.
   *Expected approach:* prefer `MEDIAN(DAU over the month) / MAU` or the 25th-percentile DAU over
   `MAU`, publish the DAU distribution (p25/p50/p75) next to the ratio, and segment bot/partner
   traffic out of the activity event. Discuss why any single-number stickiness is fragile and what
   you would put on the dashboard instead (a DAU time series plus the ratio and its percentiles).
4. **Retention as a survival-analysis problem.** Classic N-day retention answers "active on exactly
   day k". Model the underlying question — "how long until a user stops being active?" — with a
   discrete-time survival / hazard approach, and show how to compute the hazard rate and the
   survival function in SQL. Explain what the survival curve gives you that the retention curve does
   not.
   *Expected approach:* build `(user, day_index, active)` rows, compute the per-index hazard as
   `1 − (active at k / at-risk at k)` where "at-risk" is "was active at k−1" (or "has not yet
   churned" under a churn definition), accumulate `S(k) = Π(1 − h(i))`, and compare cohorts by
   hazard. Discuss censoring (a user who has not yet been observably inactive is censored, not
   churned) and why survival analysis is the right frame for a cohort that is only 20 days old. Cite
   the Kaplan–Meier / discrete-hazard literature.
5. **Reconcile three retention numbers in one board deck.** Given three teams each reporting a
   different month-3 retention for the same cohort, write the queries that produce each number, the
   exact sentence that makes each one correct, and a one-page memo that replaces all three with one
   governed metric plus two diagnostics.
   *Expected approach:* enumerate the axes (denominator, definition, activity event, period
   alignment, censoring) and map each reported number to a point in that space; show the numbers are
   not in conflict, they are answering different questions; then propose the governed metric
   (classic month-3 by activation cohort, with the activation rate as a companion). Conclude with the
   process change (definition in the metrics catalogue; new metrics require a spec, not a query) —
   the exercise is about *measurement governance*, and the SQL is just the evidence.
6. **Late-arriving entries and the instability of the denominator.** A user can sign up today but
   have their *first event* backdated to yesterday by a mobile SDK that buffers offline events.
   Design the retention model so that (a) cohort assignment is stable once published, (b) a late
   entry that would move a user between cohorts is detected and handled, and (c) the historical
   retention series does not silently revise.
   *Expected approach:* freeze cohort assignment on first observation and store both
   `signup_ts_observed` and `first_event_ts_reported`; detect divergence (the reported first event
   precedes the cohort boundary) and route it to a restatement table rather than mutating history;
   publish a `metric_version` / `as_published` snapshot so dashboards can show "as first reported"
   vs "restated". Discuss the trade-off between freshness (assign immediately) and correctness
   (wait for a late-arrival window) and how a watermark makes that trade-off explicit.

## References

- PostgreSQL — Date/Time functions and operators (`date_trunc`, `EXTRACT`, ISO week): <https://www.postgresql.org/docs/current/functions-datetime.html>
- PostgreSQL — Aggregate expressions (`FILTER`, `DISTINCT`) and window function calls: <https://www.postgresql.org/docs/current/sql-expressions.html>
- PostgreSQL — `generate_series` (calendar spine): <https://www.postgresql.org/docs/current/functions-srf.html>
- MySQL 8.0 — Date and time functions (`TIMESTAMPDIFF`, `DATE_ADD`): <https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html>
- MySQL 8.0 — Window functions (no `FILTER`, `RANGE` offsets): <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- Microsoft Learn — `DATEDIFF` and `DATETRUNC`: <https://learn.microsoft.com/en-us/sql/t-sql/functions/datediff-transact-sql>
- Microsoft Learn — `OVER` clause (frames, indexing and performance): <https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql>
- Oracle — Date and time functions (`MONTHS_BETWEEN`, `TRUNC`): <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/MONTHS_BETWEEN.html>
- Snowflake — `DATE_TRUNC` and `WEEK_START` behaviour: <https://docs.snowflake.com/en/sql-reference/functions/date_trunc>
- Snowflake — Window function syntax and usage: <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — `APPROX_COUNT_DISTINCT`: <https://docs.snowflake.com/en/sql-reference/functions/approx_count_distinct>
- Google Cloud — BigQuery date functions (`DATE_DIFF`, `DATE_TRUNC`, `GENERATE_DATE_ARRAY`): <https://cloud.google.com/bigquery/docs/reference/standard-sql/date_functions>
- Google Cloud — BigQuery HyperLogLog++ functions (approximate distinct over rolling windows): <https://cloud.google.com/bigquery/docs/reference/standard-sql/hll_functions>
- Amplitude — How time works in a retention analysis (N-day, rolling/bracket, unbounded): <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-time>
- Amplitude — Interpret your retention analysis: <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-interpret>
- Amplitude — Mastering user retention playbook (retention curves and cohort interpretation): <https://info.amplitude.com/rs/138-CDN-550/images/product-analytics-playbook-vol1-amplitude.pdf>
- Amplitude Community — How is retention calculated? (N-day): <https://community.amplitude.com/discussion/144/retention-how-is-retention-calculated-n-day-retention>
- Google Analytics — Measure sessions and user engagement (session definitions that feed DAU/WAU/MAU): <https://developers.google.com/analytics/devguides/collection/ga4/sessions>
- ga4bigquery — Sessionizing GA4 event data (part 1, 30-minute default): <https://www.ga4bigquery.com/how-to-sessionize-your-ga4-event-data-in-bigquery-part-1-default-session-definition/>
- dbt Labs — Incremental models (retention marts and recompute windows): <https://docs.getdbt.com/docs/build/incremental-models>
- dbt Labs — Metrics and the semantic layer: <https://docs.getdbt.com/docs/build/build-metrics-intro>
- Mode / ThoughtSpot — SQL Window Functions tutorial: <https://mode.com/sql-tutorial/sql-window-functions/>

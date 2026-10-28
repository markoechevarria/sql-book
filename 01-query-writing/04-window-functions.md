# Window Functions

> Window functions are the difference between SQL as a filter language and SQL as a
> computation language. They let you compute a value *per row* using a *set of rows related
> to that row* — ranking, running totals, moving averages, deltas, percentiles, session
> boundaries — without collapsing the result set the way `GROUP BY` does. This chapter
> covers the whole surface: partitions, ordering, the three frame units (`ROWS`, `RANGE`,
> `GROUPS`), frame exclusion, the ranking families, `LEAD`/`LAG`, `NTILE`, named windows,
> and the evaluation-order rules that explain every "column does not exist" error you have
> ever gotten from them.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- `OVER ()` defines a **window**: `PARTITION BY` (grouping), `ORDER BY` (ordering within
  partition), and a **frame** (which rows within the partition participate per row).
- `GROUP BY` collapses rows; **window functions keep every row**. That is the entire reason
  they exist.
- **Evaluation order**: window functions run *after* `WHERE`/`GROUP BY`/`HAVING` and
  *before* `DISTINCT`, `ORDER BY`, `LIMIT`. Consequence: you **cannot** filter on a window
  function in `WHERE` — wrap it in a subquery/CTE, or use `QUALIFY` (Snowflake/BigQuery/DuckDB).
- **The default frame is the trap.** With `ORDER BY`, the default is
  `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — a *running* window, and a *peer-aware*
  one. `LAST_VALUE(...)` therefore returns the current row, not the partition's last value.
  Without `ORDER BY`, the frame is the whole partition.
- `ROWS` = physical rows. `RANGE` = logical, based on the ordering *value* (includes peers).
  `GROUPS` = peer groups. Same query, different answers, all "correct".
- Ranking family: `ROW_NUMBER` (unique, arbitrary on ties), `RANK` (gaps on ties),
  `DENSE_RANK` (no gaps). All three are needed; there is no universal one.
- `QUALIFY` + `ROW_NUMBER() = 1` is the modern dedup idiom where available; elsewhere use a
  subquery with a `rn` column.
- **Performance is dominated by sorting.** One `Sort` per distinct `(PARTITION BY, ORDER BY)`
  combination; matching an index to the window definition can remove the sort entirely.

---

## 1. Definitions and core concepts

### 1.1 The three parts of a window

```sql
function(args) OVER (
    PARTITION BY <cols>        -- 1. divide rows into independent groups
    ORDER BY <cols>            -- 2. order rows inside each partition
    <frame clause>             -- 3. choose the slice of the partition per row
)
```

| Element | Meaning | Default |
|---------|---------|---------|
| `PARTITION BY` | Rows are processed independently per distinct key. Omitted = one global partition. | one partition |
| `ORDER BY` | Defines a row's position (needed by ranking, `LEAD`/`LAG`, running aggregates). | no order (all rows are peers) |
| Frame | `{ROWS | RANGE | GROUPS} BETWEEN <start> AND <end> [EXCLUDE ...]` | with `ORDER BY`: `RANGE UNBOUNDED PRECEDING … CURRENT ROW`; without: whole partition |

Frame bounds vocabulary:

```
UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW | n FOLLOWING | UNBOUNDED FOLLOWING
```

- `UNBOUNDED PRECEDING` … `CURRENT ROW` → running total.
- `n PRECEDING` … `n FOLLOWING` → centered moving window.
- `CURRENT ROW` … `UNBOUNDED FOLLOWING` → reverse running.
- `5 PRECEDING` … `CURRENT ROW` → trailing window (moving average over 6 rows).

### 1.2 Families of window functions

| Family | Functions | Notes |
|--------|-----------|-------|
| **Ranking** | `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()`, `CUME_DIST()`, `NTILE(n)` | require `ORDER BY` (except `NTILE` still needs an order to be meaningful) |
| **Value/navigation** | `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()` | need `ORDER BY` to be deterministic |
| **Aggregate as window** | `SUM`, `AVG`, `MIN`, `MAX`, `COUNT`, `STRING_AGG`, `BOOL_AND`, … `OVER (...)` | most support the `FILTER (WHERE ...)` clause in PostgreSQL |
| **Statistical** | `STDDEV`, `VAR_SAMP`, `CORR`, `REGR_*` | engine coverage varies |
| **Hypothetical-set / inverse distribution** | `PERCENTILE_CONT`, `PERCENTILE_DISC`, `MODE()` | syntax without frame: `WITHIN GROUP (ORDER BY ...) OVER (PARTITION BY ...)` |

### 1.3 Terms

| Term | Meaning |
|------|---------|
| **Peer rows** | Rows with equal `ORDER BY` values; they form a *peer group*. Crucial for `RANGE` and `RANK`. |
| **Frame** | The multiset of rows the function sees for the *current* row. |
| **Named window** | `WINDOW w AS (...)` — defines a window once, referenced as `OVER w`. |
| **Frame exclusion** | Removing the current row / its peers from the frame — `EXCLUDE CURRENT ROW | GROUP | TIES | NO OTHERS`. |
| **Window operator** | The physical plan node (`WindowAgg` in Postgres, `Window`/`Segment`+`Sequence Project` in SQL Server). |

---

## 2. Why it matters / what problem it solves

Without window functions, these questions require self-joins or correlated subqueries, and
both are unpleasant:

| Question | Without windows | With windows |
|---|---|---|
| Rank products by price within a category | self-join + `COUNT(cheaper)` | `RANK() OVER (PARTITION BY category ORDER BY price DESC)` |
| Cumulative revenue by day | self-join on `date >=` | `SUM(x) OVER (ORDER BY d)` |
| Day-over-day growth | self-join on `d = d - 1` (breaks on gaps) | `LAG(x)` — gap-safe |
| Share of total (percent of category) | subquery per row | `x / SUM(x) OVER (PARTITION BY cat)` |
| Deduplicate CDC rows | correlated `MAX(updated_at)` | `ROW_NUMBER() OVER (PARTITION BY key ORDER BY ts DESC)` + `= 1` |
| 7-day moving average | 7 self-joins | `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` |
| Sessionization (gaps and islands) | procedural loop | `LAG` + cumulative `SUM` flags |
| "Top 3 per group" | `LATERAL`/self-join | `ROW_NUMBER() … <= 3` |

In analytics engineering, windows are the workhorse of every dbt intermediate model:
funnels, retention, cohorts, sessionization, ranking dashboards, and deduplication of messy
event streams.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Position in the pipeline

```
FROM/JOIN → WHERE → GROUP BY → HAVING → WINDOW (evaluate window functions)
          → SELECT (projection) → DISTINCT → ORDER BY → LIMIT/OFFSET
```

Three consequences that explain 90% of window-function bugs:

1. **You cannot use a window function in `WHERE`/`HAVING`/`GROUP BY`.**
   `WHERE ROW_NUMBER() OVER (...) = 1` is a syntax error everywhere. Use a subquery, a CTE,
   or `QUALIFY`.
2. **`DISTINCT` is applied *after* the window.** `SELECT DISTINCT ROW_NUMBER() OVER (…)`
   will produce unique row numbers, so `DISTINCT` cannot collapse anything — it only wastes
   a sort. (Some engines optimize it away.)
3. **`ORDER BY` can reference window results** (it runs after), and so can
   `ORDER BY` a `ROW_NUMBER()` for stable pagination.

### 3.2 Physically: sort, then scan

A window operator generally needs its input **sorted by `(PARTITION BY cols, ORDER BY cols)`**:

```
Sort (partition keys, order keys)  ← the expensive part
  └─ WindowAgg (or Sequence Project in SQL Server)
        └─ scan / join / aggregate
```

- If a **matching index** provides the order (e.g. `orders(customer_id, ordered_at)` for
  `PARTITION BY customer_id ORDER BY ordered_at`), the planner can use an index scan and skip
  the sort — this is the single biggest window-function optimization.
- Windows that share the same `PARTITION BY` **and** `ORDER BY` are computed in **one pass**
  over the same sort (`WindowAgg` with multiple functions).
- Different orderings force **multiple sorts**. Reordering your query so all windows share one
  ordering is a real, measurable win.
- PostgreSQL 13+ can use **incremental sort** when the input is partially ordered, cutting
  the sort cost substantially.

### 3.3 How the frame is computed

For each row, the engine holds a frame boundary based on the frame unit:

- `ROWS`: count physical rows. `1 PRECEDING` = literally the previous row in the sort order,
  even if it has the same `ORDER BY` value.
- `RANGE`: use the *value* difference. With `ORDER BY x RANGE BETWEEN 1 PRECEDING AND 1
  FOLLOWING`, all rows whose `x` is within ±1 of the current row's `x` are included — this is
  the only frame unit that can express "same value" semantics. On a `timestamptz` you can
  write `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` (Postgres, Oracle).
- `GROUPS`: count *peer groups*. `GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING` = the previous
  distinct value, the current value, the next distinct value — regardless of how many rows
  share each value.

```
ORDER BY day, values: 1,1,2,2,2,3

ROWS   BETWEEN 1 PRECEDING AND 1 FOLLOWING   → physical neighbours
RANGE  BETWEEN 1 PRECEDING AND 1 FOLLOWING   → all rows with value in [v-1, v+1]
GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING   → previous distinct value group … next group
```

### 3.4 Frame exclusion

```
EXCLUDE NO OTHERS      -- default
EXCLUDE CURRENT ROW    -- remove just this row
EXCLUDE GROUP          -- remove this row and its peers
EXCLUDE TIES           -- remove peers but KEEP the current row
```

The canonical use case is "compare this row to the average of the *others* in the group":

```sql
SELECT t.*,
       AVG(value) OVER (PARTITION BY grp
                        ORDER BY ts
                        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                        EXCLUDE CURRENT ROW) AS avg_of_others
FROM t;
```

Exclusion is independent of the frame unit: `CURRENT ROW` means the current row for `ROWS`,
`RANGE` and `GROUPS` alike.

---

## 4. Syntax and mechanics (with examples)

### 4.1 Minimal forms

```sql
SELECT customer_id, order_id, ordered_at,
       COUNT(*)      OVER ()                              AS total_orders,       -- global
       COUNT(*)      OVER (PARTITION BY customer_id)      AS orders_by_customer,
       ROW_NUMBER()  OVER (ORDER BY ordered_at)           AS global_seq,
       ROW_NUMBER()  OVER (PARTITION BY customer_id
                           ORDER BY ordered_at)           AS seq_per_customer
FROM orders;
```

### 4.2 Ranking families — the ties question

```sql
SELECT p.name, p.unit_price,
       ROW_NUMBER() OVER (ORDER BY p.unit_price DESC) AS rn,
       RANK()       OVER (ORDER BY p.unit_price DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY p.unit_price DESC) AS drnk,
       PERCENT_RANK() OVER (ORDER BY p.unit_price DESC) AS pr,
       CUME_DIST()    OVER (ORDER BY p.unit_price DESC) AS cd,
       NTILE(4)       OVER (ORDER BY p.unit_price DESC) AS quartile
FROM products p ORDER BY p.unit_price DESC;
```

| Price | rn | rnk | drnk |
|---|---|---|---|
| 1999.00 | 1 | 1 | 1 |
| 1499.00 | 2 | 2 | 2 |
| 999.00 | 3 | 3 | 3 |
| 899.00 | 4 | 4 | 4 |
| 799.00 | 5 | 5 | 5 |
| 89.00 | 6 | 6 | 6 |
| 59.00 | 7 | 7 | 7 |
| 45.00 | 8 | 8 | 8 |
| 39.99 | 9 | 9 | 9 |
| 29.99 | 10 | 10 | 10 |
| 19.99 | 11 | 11 | 11 |
| 12.99 | 12 | 12 | 12 |

The seed has no price ties — add one to see the difference:

```sql
UPDATE products SET unit_price = 19.99 WHERE product_id = 9;   -- now ties with product 6
-- rn: 11,12 | rnk: 11,11 | drnk: 11,11  (then the next row's rnk jumps to 13, drnk to 12)
```

Rule of thumb: **`ROW_NUMBER` for dedup/pagination (you need uniqueness), `RANK` for
"competition ranking" (ties share a rank, gaps follow), `DENSE_RANK` for buckets (no gaps).**

### 4.3 Running totals and moving averages

```sql
WITH daily AS (
    SELECT DATE_TRUNC('day', o.ordered_at)::date AS d,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status = 'delivered'
    GROUP BY 1
)
SELECT d, revenue,
       SUM(revenue) OVER (ORDER BY d)                                   AS running_total,
       AVG(revenue) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7,
       SUM(revenue) OVER (ORDER BY d ROWS BETWEEN UNBOUNDED PRECEDING
                                             AND UNBOUNDED FOLLOWING)   AS grand_total,
       100 * revenue / SUM(revenue) OVER ()                             AS pct_of_total,
       revenue - LAG(revenue) OVER (ORDER BY d)                         AS dod_change
FROM daily ORDER BY d;
```

`> Senior note`: `SUM(x) OVER (ORDER BY d)` is `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`
by default. If two rows share date `d`, both see the same running total (they are peers).
If you want strict physical accumulation, write `ROWS BETWEEN UNBOUNDED PRECEDING AND
CURRENT ROW` explicitly.

### 4.4 `LAST_VALUE` and the default-frame trap

```sql
-- WRONG: LAST_VALUE returns the current row, because the default frame ends at CURRENT ROW
SELECT order_id, ordered_at,
       LAST_VALUE(order_id) OVER (PARTITION BY customer_id ORDER BY ordered_at) AS last_order_BUG
FROM orders;

-- RIGHT: extend the frame to the end of the partition
SELECT order_id, ordered_at,
       LAST_VALUE(order_id) OVER (PARTITION BY customer_id ORDER BY ordered_at
                                  ROWS BETWEEN UNBOUNDED PRECEDING
                                           AND UNBOUNDED FOLLOWING) AS last_order_OK
FROM orders;
```

`FIRST_VALUE` does not have this problem (its default frame already starts at
`UNBOUNDED PRECEDING`) — but both are order-sensitive, so be explicit anyway.

### 4.5 `LEAD`/`LAG` with offsets and defaults

```sql
SELECT o.order_id, o.ordered_at,
       LAG(o.ordered_at)  OVER (PARTITION BY o.customer_id ORDER BY o.ordered_at) AS prev_order_at,
       LEAD(o.ordered_at) OVER (PARTITION BY o.customer_id ORDER BY o.ordered_at) AS next_order_at,
       LEAD(o.ordered_at, 2, o.ordered_at) OVER (PARTITION BY o.customer_id
                                                 ORDER BY o.ordered_at)           AS two_ahead_or_self,
       EXTRACT(day FROM o.ordered_at
                    - LAG(o.ordered_at) OVER (PARTITION BY o.customer_id
                                              ORDER BY o.ordered_at))             AS days_since_prev
FROM orders o
ORDER BY o.customer_id, o.ordered_at;
```

On the seed: Alice (customer 1) has orders on 2024-01-05, 2024-02-11 and 2024-03-30, so her
`days_since_prev` is `NULL`, `37`, `48`.

### 4.6 Named windows (`WINDOW` clause)

```sql
SELECT o.customer_id, o.order_id, o.ordered_at,
       ROW_NUMBER() OVER w AS seq,
       LAG(o.ordered_at) OVER w AS prev_at,
       SUM(1) OVER (w ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_cnt
FROM orders o
WINDOW w AS (PARTITION BY o.customer_id ORDER BY o.ordered_at)
ORDER BY o.customer_id, o.ordered_at;
```

Rules:

- A named window can add a frame, but **cannot add or override `PARTITION BY`/`ORDER BY`**
  once defined (Postgres, and the SQL standard).
- Windows can be defined in terms of other named windows (inheritance), which is how
  `WINDOW base AS (...), w AS (base ORDER BY x)` works in some engines.
- Named windows are **supported** in PostgreSQL, MySQL 8, and Oracle 21c; **not supported**
  in SQL Server, Snowflake, or BigQuery.

### 4.7 `QUALIFY`: filter on a window result (Snowflake, BigQuery, DuckDB, Teradata)

```sql
-- Portable form (works everywhere): subquery + WHERE
SELECT * FROM (
    SELECT product_id, category_id, unit_price,
           ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS rn
    FROM products WHERE is_active
) t
WHERE rn <= 2;

-- Snowflake / BigQuery / DuckDB only
SELECT product_id, category_id, unit_price
FROM products
WHERE is_active
QUALIFY ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY unit_price DESC) <= 2;
```

Postgres/MySQL/SQL Server/Oracle have no `QUALIFY`; the subquery form is the portable
answer.

### 4.8 Combining aggregate and window ("percent of group")

```sql
SELECT c.name AS category, p.name AS product,
       SUM(i.quantity * i.unit_price) AS revenue,
       ROUND(100.0 * SUM(i.quantity * i.unit_price)
             / SUM(SUM(i.quantity * i.unit_price)) OVER (PARTITION BY c.category_id), 1)
           AS pct_of_category
FROM order_items i
JOIN orders     o ON o.order_id = i.order_id AND o.status IN ('paid','shipped','delivered')
JOIN products   p ON p.product_id = i.product_id
JOIN categories c ON c.category_id = p.category_id
GROUP BY c.name, c.category_id, p.name
ORDER BY c.name, revenue DESC;
```

Note the **nested aggregate inside `OVER`**: `SUM(SUM(x)) OVER (...)` groups first (the inner
`SUM`), then windows the grouped result (the outer `SUM`) — a legal and extremely common
"aggregate of an aggregate" pattern enabled by the evaluation order.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: order sequence per customer

```sql
SELECT customer_id, order_id, ordered_at, status,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY ordered_at) AS order_seq,
       COUNT(*)     OVER (PARTITION BY customer_id)                     AS total_orders
FROM orders
ORDER BY customer_id, order_seq;
```

Expected: customer 1 → orders 1001 (seq 1), 1002 (2), 1014 (3), `total_orders = 3`;
customer 3 → 1005 (1), 1006 (2); each of customers 4, 5, 6, 7 → 1–2 orders.

### Example 2 — Basic: monthly revenue with running total and MoM delta

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', o.ordered_at)::date AS month,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY 1
)
SELECT month, revenue,
       SUM(revenue) OVER (ORDER BY month)                                AS running,
       revenue - LAG(revenue) OVER (ORDER BY month)                      AS mom_delta,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 1)         AS mom_pct
FROM monthly ORDER BY month;
```

Expected three rows: `2024-01`, `2024-02`, `2024-03`, with `mom_delta` `NULL` for January.

### Example 3 — Intermediate: top-N per group (the dedup/leaderboard idiom)

```sql
WITH ranked AS (
    SELECT p.category_id, p.name, p.unit_price,
           RANK() OVER (PARTITION BY p.category_id ORDER BY p.unit_price DESC) AS price_rank
    FROM products p
    WHERE p.is_active
)
SELECT c.name AS category, r.name AS product, r.unit_price, r.price_rank
FROM ranked r JOIN categories c USING (category_id)
WHERE r.price_rank <= 2
ORDER BY c.name, r.price_rank, r.name;
```

Expected: `Phones → iPhone 15 (1), Galaxy S24 (2)`; `Laptops → MacBook Air M3 (1),
ThinkPad X1 (2)`; `Cases → Laptop Sleeve (1), Phone Case (2)`; `Chargers → 65W Charger (1),
USB-C Cable 2m (2)`; `Kitchen → Coffee Maker (1), Blender (2)`. `Electronics` and `Home`
have no products directly, so they do not appear (inner join).

Using `RANK` rather than `ROW_NUMBER` means ties are all returned — often what a business
user actually wants from "top 2".

### Example 4 — Intermediate: deduplicating a CDC/event stream

```sql
WITH dedup AS (
    SELECT e.*,
           ROW_NUMBER() OVER (PARTITION BY e.event_id ORDER BY e.occurred_at DESC) AS rn
    FROM events e
)
SELECT event_id, customer_id, event_type, occurred_at
FROM dedup WHERE rn = 1;
```

Expected: the 13 seeded events (no duplicates), i.e. the query is a no-op on clean data —
which is exactly how you should test it. On a real stream, `PARTITION BY` the natural key
and order by the ingestion/`updated_at` timestamp.

### Example 5 — Intermediate: sessionization with `LAG` + cumulative flag

```sql
WITH ordered AS (
    SELECT customer_id, session_id, event_type, occurred_at,
           LAG(occurred_at) OVER (PARTITION BY session_id ORDER BY occurred_at) AS prev_at
    FROM events
), flagged AS (
    SELECT *,
           CASE WHEN prev_at IS NULL
                  OR occurred_at - prev_at > interval '30 minutes' THEN 1 ELSE 0 END AS is_new_session
    FROM ordered
)
SELECT session_id, customer_id,
       MIN(occurred_at) AS session_start,
       MAX(occurred_at) AS session_end,
       COUNT(*)         AS events,
       SUM(is_new_session) OVER (PARTITION BY customer_id ORDER BY session_start
                                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
FROM flagged
GROUP BY session_id, customer_id
ORDER BY customer_id, session_start;
```

This is the **gap-and-islands** skeleton: a `LAG` to detect the gap, a `CASE` to flag it, and
a running `SUM` of the flags to assign island IDs. Chapter [04 §Analytical SQL](../04-analytical-sql/07-sessionization.md)
develops it fully.

### Example 6 — Advanced: cohort retention matrix

```sql
WITH first_order AS (
    SELECT customer_id, MIN(DATE_TRUNC('month', ordered_at))::date AS cohort_month
    FROM orders WHERE status <> 'cancelled'
    GROUP BY customer_id
), activity AS (
    SELECT o.customer_id,
           DATE_TRUNC('month', o.ordered_at)::date AS activity_month,
           f.cohort_month
    FROM orders o JOIN first_order f USING (customer_id)
    WHERE o.status <> 'cancelled'
), cohort_grid AS (
    SELECT cohort_month, activity_month,
           (EXTRACT(year  FROM age(activity_month, cohort_month)) * 12
          + EXTRACT(month FROM age(activity_month, cohort_month)))::int AS month_index,
           COUNT(DISTINCT customer_id) AS active_customers
    FROM activity GROUP BY 1,2,3
)
SELECT cohort_month, month_index, active_customers,
       FIRST_VALUE(active_customers) OVER (PARTITION BY cohort_month
                                          ORDER BY month_index) AS cohort_size,
       ROUND(100.0 * active_customers
             / FIRST_VALUE(active_customers) OVER (PARTITION BY cohort_month
                                                   ORDER BY month_index), 1) AS retention_pct
FROM cohort_grid
ORDER BY cohort_month, month_index;
```

Expected (seed): the November 2023 cohort (Alice) has `cohort_size = 1`; `month_index`
values 0, 1, 2 … with `retention_pct = 100.0` each, because the cohort has one member.
The point is the **shape of the query** — `FIRST_VALUE(active_customers) OVER (PARTITION BY
cohort_month ORDER BY month_index)` gives the denominator without a self-join.

### Example 7 — Advanced: 30-day rolling metrics on an event stream

```sql
WITH daily AS (
    SELECT customer_id, occurred_at::date AS d, COUNT(*) AS events
    FROM events GROUP BY 1,2
)
SELECT customer_id, d, events,
       SUM(events) OVER (PARTITION BY customer_id ORDER BY d
                         RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW) AS events_30d,
       AVG(events) OVER (PARTITION BY customer_id ORDER BY d
                         ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)                    AS ma7,
       MAX(events) OVER (PARTITION BY customer_id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)    AS peak_day
FROM daily
ORDER BY customer_id, d;
```

This shows the practical difference between `RANGE INTERVAL` (calendar-window semantics —
correct for "last 30 days") and `ROWS` (row-count semantics — correct for "last 7 observed
days"). Using `ROWS 29 PRECEDING` for a calendar metric silently changes the meaning when
days are missing.

### Example 8 — Advanced: `NTILE` segmentation and percentiles

```sql
WITH lifetime AS (
    SELECT c.customer_id, c.full_name,
           COALESCE(SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)), 0) AS ltv
    FROM customers c
    LEFT JOIN orders o      ON o.customer_id = c.customer_id
                           AND o.status IN ('paid','shipped','delivered')
    LEFT JOIN order_items i ON i.order_id = o.order_id
    GROUP BY c.customer_id, c.full_name
)
SELECT full_name, ltv,
       NTILE(4) OVER (ORDER BY ltv DESC) AS value_quartile,
       PERCENT_RANK() OVER (ORDER BY ltv) AS pct_rank,
       CUME_DIST()    OVER (ORDER BY ltv) AS cum_dist
FROM lifetime ORDER BY ltv DESC;
```

Expected: 8 customers → each quartile has 2 rows. `Frank Miller` (1736.00) is `value_quartile
1`; `Heidi Novak` (0, no orders) is quartile 4 with `pct_rank = 0`. Note `NTILE` always
produces `n` buckets with sizes differing by at most 1, and the bucket sizes are assigned
deterministically only if the `ORDER BY` is a total order — otherwise the split is arbitrary.

### Example 9 — Advanced: exclusion frames and gap arithmetic

```sql
-- Compare each employee's salary to the team average *excluding* themselves
SELECT e.full_name, e.salary,
       AVG(e.salary) OVER (PARTITION BY e.manager_id) AS team_avg_incl_self,
       AVG(e.salary) OVER (PARTITION BY e.manager_id
                           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                           EXCLUDE CURRENT ROW)        AS team_avg_excl_self,
       e.salary - AVG(e.salary) OVER (PARTITION BY e.manager_id) AS delta_vs_team
FROM employees e
WHERE e.manager_id IS NOT NULL
ORDER BY e.manager_id, e.salary DESC;
```

Expected: for Barbara Liskov's team (`manager_id = 103`), salaries are 165000 (Dennis), 145000
(Edgar) and 140000 (Ken) → `team_avg_incl_self = 150000` and `team_avg_excl_self` differs per
row: 142500 (excluding Dennis), 152500 (excluding Edgar), 155000 (excluding Ken). Note that
`EXCLUDE CURRENT ROW` requires PostgreSQL 11+, Oracle 21c, or a manual
`(SUM(x) - x) / (COUNT(x) - 1)` rewrite elsewhere.

---

## 6. Real-world use cases and applications

| Use case | Window recipe |
|---|---|
| **Dashboards: top-N per group** | `ROW_NUMBER()/RANK()` + filter (or `QUALIFY`) |
| **Cumulative revenue / bookings (MTD, YTD)** | `SUM() OVER (PARTITION BY year ... ORDER BY month)` |
| **MoM / WoW / DoD growth** | `LAG()` with `NULLIF` guard |
| **7/28-day retention, rolling active users** | `RANGE BETWEEN INTERVAL 'n days' PRECEDING` |
| **Deduplication of CDC / late-arriving data** | `ROW_NUMBER() OVER (PARTITION BY key ORDER BY ts DESC) = 1` |
| **Sessionization / funnel steps** | `LAG` + `CASE` + running `SUM` (gap-and-islands) |
| **Cohort retention matrices** | `FIRST_VALUE` for the denominator + `COUNT(DISTINCT)` |
| **Percent of total / contribution analysis** | `x / SUM(x) OVER (...)` |
| **Customer value segmentation** | `NTILE(n)`, `PERCENT_RANK`, `CUME_DIST` |
| **First/last touch attribution** | `FIRST_VALUE`/`LAST_VALUE` with explicit frames |
| **Gap detection ("customers who stopped buying")** | `LAG(order_date)` + `CASE` + `HAVING` |
| **Stable pagination / keyset pagination ordering** | `ROW_NUMBER()` with a tie-broken `ORDER BY` |

---

## 7. Performance notes and anti-patterns

**Cost model.** A window pass = one sort (`O(n log n)`) + one linear scan
(`O(n × frame cost)`). Total ≈ sort + scan. Therefore:

1. **An index matching `(PARTITION BY cols, ORDER BY cols)` can eliminate the sort.** For
   `PARTITION BY customer_id ORDER BY ordered_at`, `orders(customer_id, ordered_at)` is
   exactly right (this index exists in the sample schema).
2. **Reuse one window definition.** Five different `(PARTITION BY, ORDER BY)` combinations =
   up to five sorts. Named windows (`WINDOW w AS (...)`) make reuse explicit and cheap.
3. **Frames with `RANGE` on timestamps are more expensive than `ROWS`** because the engine
   must handle peers and may need a rescan per row. `GROUPS` is cheap when the number of peer
   groups is small.
4. **Wide frames with `PRECEDING n`** cost `O(n)` per row in naive implementations; engines
   use a moving aggregate to keep it `O(1)` amortized — but `EXCLUDE` and non-additive
   functions (`MIN`, `MEDIAN`) may fall back to a rescan.
5. **Window functions are not parallelizable within a partition.** A single huge partition
   (one customer with 100M rows) is a single-threaded bottleneck. Salt it, or filter first.

**Anti-patterns**

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Filtering a window in `WHERE` | syntax error | subquery/CTE or `QUALIFY` |
| `DISTINCT` with a window function in the same `SELECT` | sort you cannot collapse | dedupe first, then compute windows |
| `LAST_VALUE` without a frame | returns the current row | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Implicit `RANGE` running total with tied `ORDER BY` | "running" totals are equal within ties | use `ROWS` if you want physical accumulation |
| `ROW_NUMBER()` without a deterministic tie-break | results change between runs | add a unique column to the `ORDER BY` |
| `SUM(x) OVER ()` where `x` is `NULL` | totals silently drop rows | `COALESCE(x,0)` |
| Window over an unbounded join result | sort of a fanned-out set | aggregate to grain first, then window |
| `ORDER BY` inside the window *and* the outer query with different keys | two sorts | align them |
| Partitioning by a high-cardinality column for a global metric | millions of tiny partitions (overhead) | drop `PARTITION BY` for global metrics |

**Reading the plan**

```
WindowAgg  (cost=…) (actual time=…)
  Filter/Cond: ...
  -> Sort  (or Index Scan if the index supplies the order)
```

In Postgres, `WindowAgg` with a `Sort` child means you paid for the sort. If the sort spills
(`Sort Method: external merge  Disk: …`), raise `work_mem` or add the index. In SQL Server,
look for `Segment` + `Sequence Project` + `Segment` operators; a `Sort` feeding a `Segment`
is the same story.

---

## 8. Cross-engine differences

| Feature | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Window functions | ✅ (8.4+ / full in 11) | ✅ 8.0+ | ✅ 2005+ | ✅ | ✅ | ✅ |
| `PARTITION BY` / `ORDER BY` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ROWS` frame | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `RANGE` frame (value-based) | ✅ (incl. `INTERVAL` on timestamps) | ✅ (numeric/temporal expressions; no interval arithmetic on `ORDER BY`) | ✅ (numeric only) | ✅ | ✅ (numeric) | ✅ (numeric) |
| `GROUPS` frame | ✅ 11+ | ❌ | ❌ | ✅ 21c | ❌ | ❌ |
| `EXCLUDE CURRENT ROW/GROUP/TIES` | ✅ 11+ | ❌ | ❌ | ✅ 21c | ❌ | ❌ |
| Named `WINDOW` clause | ✅ | ✅ | ❌ | ✅ 21c | ❌ | ❌ |
| `QUALIFY` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| `IGNORE NULLS` on `LAG`/`FIRST_VALUE` | ❌ (added in PostgreSQL 19) | ❌ | ❌ | ✅ | ✅ | ✅ |
| `FILTER (WHERE …)` on window aggregates | ✅ | ❌ | ❌ | ❌ | ✅ (`FILTER` supported) | ❌ |
| `PERCENTILE_CONT` / `PERCENTILE_DISC` | ✅ (ordered-set aggregate, also `OVER` in 14+ for `DISC`) | ❌ (emulate with `NTILE`/`CUME_DIST`) | ✅ (`PERCENTILE_CONT(...) WITHIN GROUP (...)` over a window via `OVER`) | ✅ | ✅ (`PERCENTILE_CONT OVER (PARTITION BY ...)`) | ✅ (`PERCENTILE_CONT(x, 0.5) OVER (...)`) |
| `MEDIAN` | ❌ (use `PERCENTILE_DISC(0.5) WITHIN GROUP`) | ❌ | ❌ | ✅ `MEDIAN` | ✅ `MEDIAN` | ❌ |
| Window over a `DISTINCT` query | allowed but pointless (`DISTINCT` runs after) | same | same | same | same | same |
| Plan node | `WindowAgg` | `Window aggregate` | `Segment` + `Sequence Project` | `WINDOW SORT` + `ANALYTIC VIEW` | `WindowFunction` | analytic stage |

Serious gotchas per engine:

- **MySQL 8** does not support `GROUPS`, `EXCLUDE`, or `QUALIFY`; the emulation for
  `EXCLUDE CURRENT ROW` is `(SUM(x) OVER w - x) / (COUNT(x) OVER w - 1)`.
- **SQL Server** has no `WINDOW` clause, and `RANGE` only supports `UNBOUNDED`/`CURRENT ROW`
  bounds (no `RANGE n PRECEDING`); it also does not support `FILTER`.
- **Oracle** gained `WINDOW`, `GROUPS` and `EXCLUDE` in 21c — 19c and earlier do not have
  them. `IGNORE NULLS` has been available since long before.
- **Snowflake and BigQuery** are the ones with `QUALIFY`, and both support `IGNORE NULLS`.
  BigQuery requires `RANGE` bounds to be numeric (no `INTERVAL` in `ORDER BY`-based range
  frames), so calendar windows need `ROWS` plus a generated date spine.
- **PostgreSQL** is the most standards-complete here (`GROUPS`, `EXCLUDE`, `WINDOW`,
  `FILTER`, `SEARCH`), except for `QUALIFY` and `IGNORE NULLS`.

---

## 9. Exercises — Intermediate (hints only)

1. **Number each customer's orders chronologically and return only the first.** *Hint:*
   `ROW_NUMBER()` in a subquery, filter `= 1` outside.
2. **Compute a running total of revenue by day for delivered orders.** *Hint:* pre-aggregate
   to daily grain, then `SUM(...) OVER (ORDER BY d)`.
3. **For each product, show its price rank inside its category.** *Hint:* `RANK()` or
   `DENSE_RANK()` depending on whether you want gaps.
4. **Add a 7-row moving average of daily order counts.** *Hint:* `ROWS BETWEEN 6 PRECEDING
   AND CURRENT ROW`.
5. **For every order, show the previous order date of the same customer and the gap in days.**
   *Hint:* `LAG()`; be careful with `NULL` on the first row.
6. **Compute each customer's share of total revenue as a percentage.** *Hint:*
   `SUM(x) / SUM(SUM(x)) OVER ()`.
7. **Split customers into 4 quartiles by lifetime value.** *Hint:* `NTILE(4)`.
8. **For each category, show the cheapest and most expensive product in one row per
   product.** *Hint:* `FIRST_VALUE`/`LAST_VALUE` with a full-partition frame, or `MIN`/`MAX`
   `OVER (PARTITION BY ...)`.
9. **Find, per customer, the order with the largest total.** *Hint:* aggregate per order
   first, then `ROW_NUMBER`.
10. **Show the percentage of orders that are `delivered` per country.** *Hint:*
    `AVG(CASE WHEN status='delivered' THEN 1 ELSE 0 END) OVER (PARTITION BY country_code)`.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Build a complete cohort retention matrix with `month_index` and retention percentage.**
   *Approach:* three CTEs (first order → activity → grid), then `FIRST_VALUE(...) OVER
   (PARTITION BY cohort ORDER BY month_index)` for the denominator. Then state the two ways
   to make it scale (materialized table, incremental refresh by cohort).

2. **Implement sessionization with a 30-minute inactivity timeout, producing
   `session_id`, `start`, `end`, `event_count`, `duration_s`.** *Approach:* `LAG` →
   `CASE` flag → `SUM(flag) OVER (ORDER BY ts)` as the island id → `GROUP BY`. Then rewrite
   using `RANGE` frames to justify the equivalence and explain why the flag approach is
   preferred (it works on engines with limited frames).

3. **Reproduce "gap-and-islands" for a `login_events` table.** *Approach:*
   `date - ROW_NUMBER() OVER (ORDER BY date)` is constant within an island; group by it.
   Compare with recursion (see [Chapter 03](./03-recursive-ctes.md) Exercise 7) and justify
   which you would ship.

4. **Design a deduplication query for a late-arriving, at-least-once event stream.**
   Requirements: keep the latest version per `event_id`, prefer rows with non-null payloads,
   and be deterministic when timestamps tie. *Approach:* `ROW_NUMBER() OVER (PARTITION BY
   event_id ORDER BY ingested_at DESC, payload IS NULL, seq DESC)` — discuss why
   `NULLS FIRST/LAST` and boolean ordering matter, and how you would prove idempotency by
   running the query twice.

5. **Rewrite `EXCLUDE CURRENT ROW` for an engine that does not support it.** Provide the
   arithmetic rewrite for `AVG`, `SUM`, `MIN`, `MAX` and state which ones are actually
   correct. *Approach:* `AVG` = `(SUM - x)/(COUNT - 1)`; `MIN`/`MAX` require a window over
   the "others", which is not expressible arithmetically — motivate why an engine that lacks
   `EXCLUDE` costs you a self-join or a `LATERAL`.

6. **Prove the default-frame semantics empirically.** Write one query that returns
   `SUM(x) OVER (ORDER BY k)` next to `SUM(x) OVER (ORDER BY k ROWS BETWEEN UNBOUNDED
   PRECEDING AND CURRENT ROW)` on a dataset with ties in `k`, and explain the difference.
   Then extend to `GROUPS` and `RANGE`.

7. **Measure the cost of multiple windows vs one named window.** Generate 5 million rows,
   run three metrics over three different `ORDER BY` combinations, then restructure them to
   share one ordering. *Approach:* `EXPLAIN (ANALYZE, BUFFERS)` and compare `Sort` node
   durations; state the algorithmic reason (one sort vs three).

8. **Attribution modeling.** Given `events` with `utm_source`, produce last-touch and
   first-touch attribution per converted session, handling sessions with no `utm_source`
   (carry-forward). *Approach:* `COUNT(utm) OVER (ORDER BY ts)` to build groups, then
   `FIRST_VALUE(utm) IGNORE NULLS` where available, or `MAX(grp)` grouping elsewhere — this
   is the exact problem `IGNORE NULLS` solves.

---

## References

**PostgreSQL**
- Window functions (tutorial): https://www.postgresql.org/docs/current/tutorial-window.html
- Window function calls (`OVER`, frames, `EXCLUDE`, `WINDOW`): https://www.postgresql.org/docs/current/sql-expressions.html
- Built-in window functions (`row_number`, `rank`, `lag`, …): https://www.postgresql.org/docs/current/functions-window.html
- `CREATE INDEX` (avoiding the sort): https://www.postgresql.org/docs/current/sql-createindex.html

**MySQL**
- Window functions: https://dev.mysql.com/doc/refman/8.0/en/window-functions.html
- Window function descriptions and frames: https://dev.mysql.com/doc/refman/8.0/en/window-function-descriptions.html

**SQL Server**
- `OVER` clause (frames, `ROWS`/`RANGE`): https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql
- `LEAD` (`LAG` reference sibling): https://learn.microsoft.com/en-us/sql/t-sql/functions/lead-transact-sql

**Oracle**
- Analytic functions (window syntax, `GROUPS`, `EXCLUDE`, `WINDOW`): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Analytic-Functions.html
- Analytic function syntax enhancements in 21c: https://oracle-base.com/articles/21c/analytic-function-syntax-enhancements-21c

**Snowflake / BigQuery**
- Snowflake analytic functions: https://docs.snowflake.com/en/sql-reference/functions-analytic
- Snowflake `QUALIFY`: https://docs.snowflake.com/en/sql-reference/constructs/qualify
- BigQuery window function calls: https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls

**Essays and practical guides**
- Modern SQL, *OVER (… EXCLUDE CURRENT ROW)* and its support matrix: https://modern-sql.com/caniuse/over_exclude_current_row
- Modern SQL, *OVER (… RANGE BETWEEN …) (datetime)*: https://modern-sql.com/caniuse/over_range_between_(datetime)
- Markus Winand's window-frames tutorial (PostgreSQL 11): https://modern-sql.com/blog/2019-02/postgresql-11
- *SQL Window Functions with Framing* (MSSQLTips): https://www.mssqltips.com/sqlservertip/11550/sql-window-functions-with-framing/

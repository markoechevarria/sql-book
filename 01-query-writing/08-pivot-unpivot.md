# PIVOT / UNPIVOT

> Analytical data is *tidy* (one row per observation) but human beings read *matrices* (one row
> per entity, one column per category). PIVOT and UNPIVOT are the two directions of that
> translation: rows → columns (cross-tabulate) and columns → rows (melt). Only some engines have
> native operators, and the manual versions built from `CASE` + aggregates are the ones you will
> actually use most — because they are portable, explainable, and easy to generate dynamically.
> This chapter covers both directions, native and manual, plus the traps (NULL-dropping
> unpivots, non-sargable pivots, exploding column counts) that turn a report into an incident.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- **PIVOT (cross-tabulate)**: aggregate a *long* table into a *wide* one, one output column per
  distinct value of a "pivot key". Implemented as **conditional aggregation**:
  `SUM(CASE WHEN k = 'x' THEN v END)` + `GROUP BY <all other columns>`.
- **UNPIVOT (melt)**: turn *wide* columns into *long* rows. Implemented as a `UNION ALL` of
  one projection per column (or `LATERAL`/`VALUES` + `CROSS JOIN`).
- **Native operators** exist in SQL Server, Oracle, Snowflake and BigQuery:
  `FROM … PIVOT (agg FOR col IN (v1, v2, …))` and `FROM … UNPIVOT (val FOR col IN (c1, c2, …))`.
- **PostgreSQL has no native `PIVOT`/`UNPIVOT`.** Use the `CASE` idiom, the `FILTER` clause,
  the `tablefunc` extension's `crosstab()`, or `jsonb_object_agg` for dynamic output.
- **Both native `UNPIVOT` implementations drop NULLs by default** (SQL Server always does;
  Oracle/Snowflake/BigQuery have explicit `INCLUDE NULLS`/`EXCLUDE NULLS`). Losing NULL rows
  silently is the #1 unpivot bug.
- **Native `PIVOT` implicitly groups by every column you do not mention** — the classic
  "why is my pivot producing more rows than expected" cause. Always select *only* the columns
  you want as row keys in the pre-pivot subquery.
- **Dynamic pivots** (unknown value list at write time) require either dynamic SQL
  (`EXECUTE IMMEDIATE` / `sp_executesql` / `FORMAT`), JSON aggregation, or a `crosstab`-style
  function. This is a real limitation of static SQL, not a bug.
- Wide pivots degrade: a pivot over a 5,000-value column produces 5,000 columns, blows up the
  plan and the client. Cross-tabs belong at the presentation layer.

---

## 1. Definitions and core concepts

### 1.1 The two directions

```
PIVOT                                    UNPIVOT
  long                                    wide
  country | status | n                    country | delivered | returned
  US      | delivered | 3        ─────►   US      | 3         | 1
  US      | returned  | 1                 DE      | 1         | 1
  DE      | delivered | 1
  DE      | returned  | 1
```

| Operation | Other names | Input grain | Output grain |
|---|---|---|---|
| PIVOT | cross-tab, crosstab, transpose (partially) | one row per (row-key, column-key) | one row per row-key, one column per column-key |
| UNPIVOT | melt, gather, normalise, stack | one row per entity, many numeric/measure columns | one row per (entity, measure-name) |

### 1.2 The aggregate that must exist

Pivoting is **always** an aggregation: for a given `(row-key, column-key)` pair there may be
zero, one, or many rows, so you must say what to do with them. That is why every pivot form
contains an aggregate function:

- `SUM(...)` — additive measures (revenue, quantity).
- `COUNT(...)` / `COUNT(CASE …)` — frequency counts (orders, events).
- `MAX(...)` / `MIN(...)` — a single value per cell (the "last status" pattern).
- `AVG(...)` — averages (usually a sign you should pivot on ratios instead).
- `STRING_AGG` / `LISTAGG` / `ARRAY_AGG` — multiple values per cell (list-valued cells).

Choosing `SUM` when you meant `MAX` (or vice versa) is the most common pivot correctness bug.

### 1.3 Vocabulary

| Term | Meaning |
|---|---|
| **Row key** | The columns that stay as rows (`GROUP BY` columns). |
| **Column key (pivot column)** | The column whose *values* become output columns. |
| **Value column** | The column that is aggregated into cells. |
| **Cell** | One `(row key, column key)` aggregate. `NULL` when no rows match — note the difference between "no rows" and "sum of zero". |
| **Static value list** | `IN ('pending','paid',…)` — required by native `PIVOT` in most engines. |
| **Dynamic pivot** | The value list is computed at runtime, requiring dynamic SQL or a helper function. |
| **Wide/long** | Wide = many columns; long/tidy = one measure column, one dimension column. |

---

## 2. Why it matters / what problem it solves

- **Reporting and dashboards**: "orders per month per channel" is a matrix; the natural SQL
  output is a matrix. Pivoting in SQL avoids shipping 100k rows to the client.
- **Data interchange**: spreadsheets, Google Sheets exports, and CSV extracts are wide by
  convention; unpivoting is the first step of any ingestion pipeline that receives such files.
- **Model requirements**: many statistical/ML models and plotting libraries want tidy data
  (one column per variable) — unpivot before you fit.
- **Comparison matrices**: A/B test results, cohort retention grids, confusion matrices.
- **Cross-engine migration**: knowing the `CASE` forms means your query runs unchanged on
  engines that lack `PIVOT`.
- **Schema change buffering**: a "one column per fiscal quarter" table is a pivot that a
  business user asked for; turning it back into rows is how you make it queryable.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Manual pivot = conditional aggregation

```sql
SELECT country_code,
       SUM(CASE WHEN status = 'delivered' THEN amount ELSE 0 END) AS delivered,
       SUM(CASE WHEN status = 'returned'  THEN amount ELSE 0 END) AS returned
FROM … GROUP BY country_code;
```

Evaluation:

1. The `FROM/JOIN/WHERE` pipeline produces one row per (row-key, column-key, value).
2. `GROUP BY country_code` partitions rows by the row key.
3. For each group and each output column, the `CASE` keeps only rows whose column key matches,
   and the aggregate folds them.

So the cost is **one scan + one hash/sort aggregation**, independent of how many output columns
you add — each extra column is one more `CASE` inside the same aggregate node. That is why
manual pivots scale linearly with the *number* of columns and not at all with the number of
rows. It is also why a pivot with 500 columns is slow to *plan* rather than slow to execute.

### 3.2 Native `PIVOT`

```sql
-- SQL Server / Oracle / Snowflake / BigQuery
SELECT country_code, [delivered], [returned]     -- SQL Server bracket syntax
FROM (
    SELECT o.country_code, o.status, i.quantity * i.unit_price AS amount
    FROM orders o JOIN order_items i ON i.order_id = o.order_id
) src
PIVOT (
    SUM(amount) FOR status IN ([delivered], [returned])
) pvt;
```

Semantics:

1. The inner subquery is the input relation.
2. Every column of the input that is **not** the pivot column and **not** the aggregated column
   becomes a **grouping column** — implicitly. (This is the "hidden `GROUP BY`".)
3. `FOR status IN (…)` creates one output column per listed value.
4. The `AS` alias names the resulting relation; output columns are named from the value list.

Consequences:

- **Filter/clean your pre-pivot subquery.** If you `SELECT *`, your pivot groups by every column
  and returns more rows than intended.
- **Ordering is not guaranteed.** Wrap the pivot in an outer `ORDER BY`.
- **The value list is static** (except Snowflake's `ANY`, Oracle's `XML`, and BigQuery's
  dynamic-SQL workaround). Adding a new status requires editing the query.
- **Non-constant values are rejected** — you cannot write `IN (SELECT DISTINCT status …)`.
- **`NULL` values in the pivot column** become their own column in some engines (Snowflake
  creates a column named after the NULL) or are silently dropped in others. Filter them out for
  deterministic output.

### 3.3 Native `UNPIVOT`

```sql
-- SQL Server
SELECT country_code, measure, value
FROM (SELECT country_code, delivered, returned FROM country_matrix) src
UNPIVOT (
    value FOR measure IN (delivered, returned)
) unpvt;
```

Semantics: for each input row, emit one output row per listed column:

1. Replicate the row once per column in the `IN (…)` list.
2. `measure` receives the *column name* (as a string).
3. `value` receives the column's value for that row.

Implementation is (conceptually) a `UNION ALL` of `N` projections, each selecting one column
aliased as a constant name. Because it is a union of projections, **each column reference is
read once** — but beware: some engines read the source once and fan out, others do not; check
the plan for repeated table scans.

**NULL handling — the big one:**

| Engine | Default | Control |
|---|---|---|
| SQL Server | drops NULL values | none (filter explicitly or `UNION ALL` manually) |
| Oracle | `EXCLUDE NULLS` (default) | `INCLUDE NULLS` / `EXCLUDE NULLS` keywords |
| Snowflake | `EXCLUDE NULLS` (default) | `INCLUDE NULLS` / `EXCLUDE NULLS` |
| BigQuery | `EXCLUDE NULLS` (default) | `INCLUDE NULLS` / `EXCLUDE NULLS` |
| PostgreSQL | n/a (no native operator) | your `UNION ALL` decides |

`> Senior note`: the manual form is the only one where NULL preservation is explicit and
obvious. If your pipeline distinguishes "this column was NULL" from "this column did not
exist", write the `UNION ALL` manually.

### 3.4 The manual unpivot: three equivalent forms

```sql
-- (1) UNION ALL of projections (universal, explicit, N scans)
SELECT country_code, 'delivered' AS measure, delivered AS value FROM country_matrix
UNION ALL
SELECT country_code, 'returned',  returned  FROM country_matrix;

-- (2) LATERAL + VALUES (Postgres/MySQL 8/Oracle/Snowflake; one scan)
SELECT m.country_code, v.measure, v.value
FROM country_matrix m
CROSS JOIN LATERAL (VALUES
    ('delivered', m.delivered),
    ('returned',  m.returned)
) AS v(measure, value);

-- (3) UNNEST of an array of structs (BigQuery/Snowflake/Postgres)
SELECT m.country_code, x.measure, x.value
FROM country_matrix m,
     UNNEST([STRUCT('delivered' AS measure, m.delivered AS value),
             STRUCT('returned', m.returned)]) AS x;
```

Form (2) is the sandbag: it reads the source **once** and is the modern portable idiom.

---

## 4. Syntax and mechanics (with examples)

### 4.1 Portable manual pivot (`CASE` + aggregate)

```sql
SELECT o.country_code,
       COUNT(CASE WHEN o.status = 'delivered' THEN 1 END) AS delivered,
       COUNT(CASE WHEN o.status = 'returned'  THEN 1 END) AS returned,
       COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancelled,
       COUNT(*)                                          AS total
FROM orders o
GROUP BY o.country_code
ORDER BY o.country_code;
```

### 4.2 PostgreSQL/Snowflake `FILTER` variant (cleaner, non-portable)

```sql
SELECT o.country_code,
       COUNT(*) FILTER (WHERE o.status = 'delivered') AS delivered,
       COUNT(*) FILTER (WHERE o.status = 'returned')  AS returned,
       COUNT(*)                                       AS total
FROM orders o
GROUP BY o.country_code
ORDER BY o.country_code;
```

### 4.3 PostgreSQL `crosstab()` (tablefunc extension) — dynamic-ish pivoting

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    $$ SELECT country_code, status, COUNT(*)::int
       FROM orders
       WHERE status IN ('delivered','returned','cancelled')
       GROUP BY 1, 2
       ORDER BY 1, 2 $$,
    $$ SELECT unnest(ARRAY['delivered','returned','cancelled']) ORDER BY 1 $$
) AS ct(country_code char(2), delivered int, returned int, cancelled int);
```

`crosstab` still needs the output column *types* declared up front — so it does not free you
from static SQL, only from writing the `CASE`s.

### 4.4 PostgreSQL dynamic pivot with `jsonb_object_agg` (no dynamic SQL)

```sql
SELECT o.country_code,
       jsonb_object_agg(o.status, o.n) AS status_counts
FROM (SELECT country_code, status, COUNT(*) AS n
      FROM orders GROUP BY country_code, status) o
GROUP BY o.country_code
ORDER BY o.country_code;
```

Output is a JSON object per country — a *semi-wide* result: schema-stable in SQL, dynamic in
content. Downstream tools can pivot the JSON themselves. This is the pragmatic answer when the
value list really is unknown at compile time.

### 4.5 Java/SQL Server dynamic pivot (`sp_executesql` / `EXECUTE IMMEDIATE`)

```sql
-- SQL Server pattern (abbreviated)
DECLARE @cols nvarchar(max), @sql nvarchar(max);
SELECT @cols = STRING_AGG(QUOTENAME(status), ',')
FROM (SELECT DISTINCT status FROM orders) s;

SET @sql = N'
SELECT country_code, ' + @cols + N'
FROM (SELECT o.country_code, o.status, 1 AS one FROM orders o) src
PIVOT (COUNT(one) FOR status IN (' + @cols + N')) p;';
EXEC sp_executesql @sql;
```

```sql
-- BigQuery pattern
EXECUTE IMMEDIATE FORMAT("""
  SELECT * FROM (SELECT country_code, status FROM orders)
  PIVOT (COUNT(*) FOR status IN (%s))
""", (SELECT STRING_AGG(FORMAT("'%s'", status), ',') FROM (SELECT DISTINCT status FROM orders)));
```

Snowflake is the outlier here — it has **no dynamic SQL inside a plain `SELECT`**, but its
`PIVOT` accepts `ANY` (and `ANY ORDER BY …`), which produces all values dynamically:

```sql
-- Snowflake
SELECT * FROM orders
PIVOT (COUNT(*) FOR status IN (ANY ORDER BY status)) AS p;
```

Oracle supports `<PIVOT … IN (ANY)>` as well (11g+), returning XML that you then shred with
`XMLTABLE`; 18c+ also allows a subquery in the `IN` list of `PIVOT`.

### 4.6 Native `UNPIVOT` in each engine

```sql
-- Oracle / Snowflake / BigQuery style
SELECT country_code, measure, value
FROM country_matrix
UNPIVOT INCLUDE NULLS (value FOR measure IN (delivered, returned));
```

Snowflake and BigQuery require explicit aliases for the generated columns in some positions;
SQL Server derives the column-name strings from the source columns.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: order counts per country × status (manual pivot)

```sql
SELECT o.country_code,
       COUNT(CASE WHEN o.status = 'pending'   THEN 1 END) AS pending,
       COUNT(CASE WHEN o.status = 'paid'      THEN 1 END) AS paid,
       COUNT(CASE WHEN o.status = 'shipped'   THEN 1 END) AS shipped,
       COUNT(CASE WHEN o.status = 'delivered' THEN 1 END) AS delivered,
       COUNT(CASE WHEN o.status = 'returned'  THEN 1 END) AS returned,
       COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancelled,
       COUNT(*)                                          AS total
FROM orders o
GROUP BY o.country_code
ORDER BY o.country_code;
```

Expected (from the seed):

| country | pending | paid | shipped | delivered | returned | cancelled | total |
|---|---|---|---|---|---|---|---|
| DE | 0 | 0 | 0 | 1 | 1 | 0 | 2 |
| FR | 1 | 1 | 0 | 0 | 0 | 0 | 2 |
| GB | 0 | 0 | 1 | 1 | 0 | 0 | 2 |
| JP | 0 | 1 | 0 | 0 | 0 | 0 | 1 |
| US | 2 | 0 | 0 | 3 | 0 | 1 | 6 |

(The US has orders 1001, 1002, 1005, 1006, 1011, 1012, 1014 — count them from the seed; the
structure is what matters.)

### Example 2 — Basic: revenue matrix — category × channel

```sql
WITH lines AS (
    SELECT cat.name AS category, o.channel,
           i.quantity * i.unit_price * (1 - i.discount_pct/100) AS amount
    FROM order_items i
    JOIN orders     o   ON o.order_id = i.order_id
    JOIN products   p   ON p.product_id = i.product_id
    JOIN categories cat ON cat.category_id = p.category_id
    WHERE o.status IN ('paid','shipped','delivered')
)
SELECT category,
       ROUND(SUM(CASE WHEN channel = 'web'         THEN amount ELSE 0 END), 2) AS web,
       ROUND(SUM(CASE WHEN channel = 'mobile'      THEN amount ELSE 0 END), 2) AS mobile,
       ROUND(SUM(CASE WHEN channel = 'marketplace' THEN amount ELSE 0 END), 2) AS marketplace,
       ROUND(SUM(CASE WHEN channel = 'store'       THEN amount ELSE 0 END), 2) AS store,
       ROUND(SUM(amount), 2)                                                  AS total
FROM lines
GROUP BY category
ORDER BY total DESC NULLS LAST;
```

Expected highlights: `Laptops → marketplace 2028.99, store 1499.00` (Dave's MacBook plus sleeve
via marketplace, Frank's ThinkPad in store); `Phones → web 999.00 (order 1001), mobile 899.00
(order 1003), mobile 799.00 (order 1009)` — all three are `paid`/`delivered` so they survive the
filter; `Cases → web 35.98` (order 1001's two discounted cases); `Chargers → web 79.98`
(order 1008's two chargers, `shipped`) and `mobile 32.98` (order 1013's case + cable, `paid`).
Note `ELSE 0` versus omitting `ELSE`: with `ELSE 0` the cell is `0.00`; without it the cell is
`NULL` and `SUM` ignores it — same *sum*, different *cell type*. Choose deliberately.

### Example 3 — Intermediate: three-way matrix with `ORDER BY` inside a pivot

```sql
WITH base AS (
    SELECT DATE_TRUNC('month', o.ordered_at)::date AS month,
           o.channel,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY 1, 2
)
SELECT month,
       COALESCE(SUM(CASE WHEN channel = 'web'         THEN revenue END), 0) AS web,
       COALESCE(SUM(CASE WHEN channel = 'mobile'      THEN revenue END), 0) AS mobile,
       COALESCE(SUM(CASE WHEN channel = 'marketplace' THEN revenue END), 0) AS marketplace,
       COALESCE(SUM(CASE WHEN channel = 'store'       THEN revenue END), 0) AS store
FROM base
GROUP BY month
ORDER BY month;
```

Expected: one row per month (`2024-01`, `2024-02`, `2024-03`) with zero-filled cells where a
channel had no revenue that month. This is the **date × channel matrix** that every BI tool
asks for; note the `COALESCE(...,0)` to turn "no rows" into a real zero for charting.

### Example 4 — Intermediate: unpivot a wide monthly table back to tidy

```sql
-- Pretend a spreadsheet delivered this wide table
CREATE TEMP TABLE monthly_wide AS
SELECT '2024-01'::text AS month, 2028.99::numeric AS marketplace, 1034.98::numeric AS web, 0::numeric AS mobile
UNION ALL SELECT '2024-02', 0, 38.97, 1499.00
UNION ALL SELECT '2024-03', 0, 39.99 + 799.00, 12.99 + 89.00;

-- Manual unpivot with LATERAL + VALUES: one scan, NULL-explicit
SELECT w.month, v.channel, v.revenue
FROM monthly_wide w
CROSS JOIN LATERAL (VALUES
    ('marketplace', w.marketplace),
    ('web',         w.web),
    ('mobile',      w.mobile)
) AS v(channel, revenue)
ORDER BY w.month, v.channel;

-- Equivalent UNION ALL form (3 scans of a temp table — fine here, not on 500M rows)
SELECT month, 'marketplace' AS channel, marketplace AS revenue FROM monthly_wide
UNION ALL SELECT month, 'web',    web    FROM monthly_wide
UNION ALL SELECT month, 'mobile', mobile FROM monthly_wide
ORDER BY month, channel;
```

Expected: 9 tidy rows (3 months × 3 channels). Both forms produce identical output; the
`LATERAL` form is the one to use on large inputs.

### Example 5 — Intermediate: pivot with `MAX` for "latest status per order"

A pivot is not always a sum. Here is the "one column per status, cell = the count of orders
that *ever* reached that status" using `MAX` of a flag:

```sql
WITH per_order AS (
    SELECT o.order_id,
           MAX(CASE WHEN o.status = 'delivered' THEN 1 ELSE 0 END) AS reached_delivered,
           MAX(CASE WHEN o.status = 'returned'  THEN 1 ELSE 0 END) AS reached_returned
    FROM orders o
    GROUP BY o.order_id
)
SELECT customer_id,
       MAX(reached_delivered) AS ever_delivered,   -- this is the pivot *for a customer*
       MAX(reached_returned)  AS ever_returned
FROM per_order p JOIN orders o USING (order_id)
GROUP BY customer_id
ORDER BY customer_id;
```

Expected: `Bob Smith` (2) has `ever_delivered = 1` and `ever_returned = 1`; `Carol White` (3)
has both `0`; `Alice` (1) has `ever_delivered = 1`, `ever_returned = 0`. This is the
"has-ever-done-X" pivot, ubiquitous in customer feature tables.

### Example 6 — Advanced: native `PIVOT` (SQL Server / Snowflake / BigQuery / Oracle)

```sql
-- Snowflake (bracket-free, double-quoted aliases)
SELECT *
FROM (SELECT country_code, status FROM orders)
PIVOT (COUNT(*) FOR status IN ('pending','paid','shipped','delivered','returned','cancelled'))
ORDER BY country_code;
```

```sql
-- SQL Server
SELECT country_code, [pending],[paid],[shipped],[delivered],[returned],[cancelled]
FROM (SELECT country_code, status FROM orders) AS src
PIVOT (COUNT(status) FOR status IN ([pending],[paid],[shipped],[delivered],[returned],[cancelled])) AS pvt
ORDER BY country_code;
```

```sql
-- BigQuery
SELECT * FROM (SELECT country_code, status FROM orders)
PIVOT (COUNT(*) AS n FOR status IN ('pending','paid','shipped','delivered','returned','cancelled'))
ORDER BY country_code;
```

```sql
-- Oracle
SELECT * FROM (SELECT country_code, status FROM orders)
PIVOT (COUNT(*) FOR status IN ('pending','paid','shipped','delivered','returned','cancelled'))
ORDER BY country_code;
```

All four produce the same matrix as Example 1 with far less typing. The **comparison of what
you give up** is the lesson:

| Trade-off | Manual `CASE` | Native `PIVOT` |
|---|---|---|
| Portability | everywhere | 4 engines only |
| Readability for 3 columns | fine | similar |
| Readability for 30 columns | terrible | good |
| Control over `NULL` vs `0` cells | explicit (`ELSE 0`, `COALESCE`) | engine default |
| Dynamic value list | via dynamic SQL/JSON | Snowflake `ANY`; others via dynamic SQL |
| Column ordering guarantee | yes (you write it) | not guaranteed; `ORDER BY` may not apply to columns |
| Debuggability | a normal aggregate | opaque grouping if you `SELECT *` |

### Example 7 — Advanced: native `UNPIVOT` and the NULL-dropping trap

```sql
-- Snowflake / Oracle / BigQuery: compare default vs INCLUDE NULLS
CREATE OR REPLACE TEMP TABLE w (id int, a numeric, b numeric, c numeric);
INSERT INTO w VALUES (1, 10, NULL, 30), (2, NULL, NULL, NULL);

SELECT id, measure, value FROM w
UNPIVOT (value FOR measure IN (a, b, c));                 -- 4 rows: (1,a,10),(1,c,30),(2,… none)

SELECT id, measure, value FROM w
UNPIVOT INCLUDE NULLS (value FOR measure IN (a, b, c));   -- 6 rows including NULLs
```

Expected: without `INCLUDE NULLS` you get **4 rows** (the two non-NULL cells for id 1, and
nothing for id 2 because all its cells are NULL); with `INCLUDE NULLS` you get **6 rows**.
PostgreSQL's manual equivalent:

```sql
SELECT w.id, v.measure, v.value
FROM w CROSS JOIN LATERAL (VALUES ('a', w.a), ('b', w.b), ('c', w.c)) AS v(measure, value)
ORDER BY w.id, v.measure;                                  -- 6 rows, NULLs preserved
```

This is the trap: a pipeline that unpivots, aggregates with `COUNT(*)`, and reports "3
observations" instead of "6" — because NULLs vanished.

### Example 8 — Advanced: dynamic pivot in PostgreSQL with `crosstab` and with JSON

```sql
-- Static types required, but the *rows* are data-driven
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    $$ SELECT country_code, status, COUNT(*)::int
       FROM orders GROUP BY 1,2 ORDER BY 1,2 $$,
    $$ SELECT s FROM unnest(ARRAY['pending','paid','shipped','delivered','returned','cancelled']) s ORDER BY 1 $$
) AS ct(country_code char(2), pending int, paid int, shipped int, delivered int,
        returned int, cancelled int)
ORDER BY country_code;

-- For genuinely dynamic shapes, emit JSON and let the consumer pivot
SELECT channel,
       jsonb_object_agg(status, cnt) AS by_status
FROM (SELECT channel, status, COUNT(*) AS cnt FROM orders GROUP BY 1,2) t
GROUP BY channel ORDER BY channel;
```

Expected: the `crosstab` result matches Example 1 exactly. The JSON result gives one row per
channel with an object like `{"delivered": 3, "cancelled": 1, "pending": 2}` — the pattern to
use when the set of statuses can change without notice.

### Example 9 — Advanced: pivot for ML feature engineering (the real use case)

```sql
WITH features AS (
    SELECT c.customer_id,
           COALESCE(SUM(CASE WHEN o.country_code = c.country_code THEN 1 ELSE 0 END), 0) AS orders_home,
           COALESCE(SUM(CASE WHEN o.channel = 'web'    THEN 1 ELSE 0 END), 0)            AS orders_web,
           COALESCE(SUM(CASE WHEN o.channel = 'mobile' THEN 1 ELSE 0 END), 0)            AS orders_mobile,
           COALESCE(SUM(CASE WHEN o.status  = 'returned' THEN 1 ELSE 0 END), 0)          AS orders_returned,
           COALESCE(SUM(CASE WHEN o.status  = 'delivered' THEN 1 ELSE 0 END), 0)         AS orders_delivered,
           COUNT(o.order_id)                                                             AS orders_total
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id
    GROUP BY c.customer_id
)
SELECT f.*, c.loyalty_tier, c.country_code
FROM features f JOIN customers c USING (customer_id)
ORDER BY f.customer_id;
```

Expected: one row per customer (8 rows) with six engineered features and no NULLs (thanks to
`COALESCE`). This is the canonical "one row per entity, one column per behavior" feature
table — and it is *exactly* a pivot with `SUM(CASE …)` — the same mechanics as every example
above, applied to model input rather than a report.

---

## 6. Real-world use cases and applications

| Use case | Direction | Notes |
|---|---|---|
| BI cross-tab / matrix reports | PIVOT | often better left to the BI layer if the column set is dynamic |
| Cohort retention grids | PIVOT | `month_index` becomes the column key |
| A/B test result matrices | PIVOT | variants × metrics |
| Excel/Sheets ingestion | UNPIVOT | wide files → tidy staging table |
| Feature engineering | PIVOT | behaviours → one-hot/feature columns |
| Confusion matrices | PIVOT | actual × predicted |
| Financial statements (periods as columns) | PIVOT | `period` becomes columns |
| Reconciliation reports | both | unpivot both sides, compare long, pivot for presentation |
| Event-to-state reconstruction | PIVOT with `MAX(CASE …)` | "current status" per entity |
| Spark/Databricks wide tables | PIVOT | `df.groupBy(...).pivot(...).agg(...)` — same idea in the DataFrame API |

---

## 7. Performance notes and anti-patterns

### 7.1 What pivoting costs

- **Planning, not execution.** Pivot nodes are aggregates; the execution cost is the underlying
  `GROUP BY`. The `PIVOT` operator's overhead is in *plan size* (hundreds of output columns)
  and in client serialization.
- **Column count explosion.** 1,000 output columns means a 1,000-column result; Postgres has a
  hard limit of 1,600 columns per result row, SQL Server 1,024, and most clients choke long
  before that.
- **Manual unpivot with `UNION ALL` = N scans** of the source. On a big table, use
  `LATERAL (VALUES …)` / `UNNEST` (one scan) or a native `UNPIVOT` and verify the plan shows a
  single scan.
- **Pre-aggregate before pivoting.** Pivoting a raw fact table with 100M rows into 50 columns
  is 50 `CASE`s over 100M rows. Aggregate to `(row key, column key)` first (a much smaller
  relation, typically), then pivot — the pivot then operates on `(#row keys × #column keys)`
  rows.
- **Add `NULL`-handling deliberately.** `SUM(CASE … END)` gives `NULL` for empty cells;
  `SUM(CASE … ELSE 0 END)` gives `0`. Pick one and apply it consistently, and remember that
  `AVG` differs drastically between the two.

### 7.2 Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Native `PIVOT` with `SELECT *` in the inner query | more rows than expected (hidden `GROUP BY`) | select only the row key, pivot key, value |
| Pivoting a high-cardinality column (`user_id`, `sku`) | thousands of columns, unreadable/unusable | aggregate first; report at a coarser grain |
| Using `PIVOT` as a storage model | schema changes on every new value | store tidily; pivot at presentation |
| `UNPIVOT` then `COUNT(*)` while relying on NULL rows | silently low counts | `INCLUDE NULLS` or manual `VALUES`/`UNION ALL` |
| `UNION ALL` unpivot of a 500-column table | 500 table scans | `LATERAL (VALUES …)` / `UNNEST`, or native `UNPIVOT` |
| Assuming `PIVOT` preserves column order | unstable output in some engines | name columns explicitly in a view/CTAS |
| Dynamic pivot via string concatenation without quoting | SQL injection / syntax errors on odd values | `QUOTENAME`/`FORMAT` helpers, or JSON instead |
| Pivoting in SQL *and* unpivoting in the BI tool | double work, double bugs | pick one layer |
| `AVG(CASE … ELSE 0 END)` | averages pulled toward zero | use `AVG(CASE … END)` (NULLs ignored) |
| Mixing types in pivot cells | implicit casts / errors | cast inside the `CASE` |

### 7.3 Deciding where the pivot belongs

| Situation | Put the pivot in |
|---|---|
| Fixed, small column set (≤ 20), needed by SQL consumers | SQL (manual `CASE`) |
| Fixed column set, engine supports `PIVOT`, query readability matters | native `PIVOT` |
| Dynamic column set, presentation-only | the BI tool / application |
| Dynamic column set, needed by SQL | JSON aggregation, or dynamic SQL with a quoted list |
| Very large input | pre-aggregate in SQL, pivot in the tool |

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Native `PIVOT` | ❌ (use `crosstab()` or `CASE`) | ❌ (use `CASE`) | ✅ | ✅ (11g+) | ✅ (`IN (ANY)` supported) | ✅ |
| Native `UNPIVOT` | ❌ | ❌ | ✅ | ✅ (`INCLUDE/EXCLUDE NULLS`) | ✅ (`INCLUDE/EXCLUDE NULLS`) | ✅ (`INCLUDE/EXCLUDE NULLS`) |
| `UNPIVOT` NULL default | n/a | n/a | **drops NULLs** (no option) | drops (`EXCLUDE NULLS`) | drops (`EXCLUDE NULLS`) | drops (`EXCLUDE NULLS`) |
| Alias column names in pivot output | n/a | n/a | must list `[x]` columns | derived, can alias | derived (quoted) | derived with `AS n` |
| Values must be constants | n/a | n/a | ✅ | ❌ (subquery allowed 18c+) | ❌ (`ANY`) | ✅ (constants; dynamic needs `EXECUTE IMMEDIATE`) |
| Dynamic value list | `crosstab` + declared types, or dynamic SQL | dynamic SQL (`PREPARE`) | `STRING_AGG` + `sp_executesql` | `XML` + `XMLTABLE`, or `ANY` | `ANY` / `ANY ORDER BY` | `EXECUTE IMMEDIATE FORMAT(...)` |
| Multiple aggregate functions per cell | one per `CASE` | one per `CASE` | ✅ (multiple `PIVOT` blocks) | ✅ | ✅ | ❌ (single aggregate per `PIVOT`) |
| `FILTER` clause (cleaner manual pivot) | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `LATERAL (VALUES …)` unpivot | ✅ | ✅ | ✅ (`CROSS APPLY`) | ✅ | ✅ | ✅ (`UNNEST`, `UNNEST`) |
| Column limit per result row | 1,600 | 4,096 | 1,024 | 1,000 (index) / 1000+ cols | 10,000 (table) | 10,000 |
| JSON aggregation for dynamic pivot | `jsonb_object_agg` | `JSON_OBJECTAGG` | `FOR JSON`, `STRING_AGG` | `JSON_OBJECTAGG` | `OBJECT_AGG` | `TO_JSON_STRING` |

Practical notes:

- **PostgreSQL**: `crosstab()` requires the `tablefunc` extension and a *declared* output type
  list. For dynamic shapes, `jsonb_object_agg` is usually the better answer, since SQL cannot
  return a variable column set without dynamic SQL.
- **MySQL 8**: no `PIVOT`; the `CASE` idiom is universal. MySQL does have
  `JSON_OBJECTAGG` for the semi-wide JSON approach.
- **SQL Server**: `PIVOT` requires an alias and explicitly named output columns; `UNPIVOT`
  always drops NULLs, so if NULLs matter you must hand-roll with `CROSS APPLY (VALUES …)`.
- **Oracle**: `PIVOT`/`UNPIVOT` since 11g, with `INCLUDE/EXCLUDE NULLS` and XML output for
  dynamic lists; `ANY` also works.
- **Snowflake**: the best-native tooling of the group (`ANY` inside `IN`, `INCLUDE NULLS` on
  `UNPIVOT`), plus `LATERAL FLATTEN` for semi-structured unpivoting.
- **BigQuery**: `PIVOT`/`UNPIVOT` operators in the `FROM` clause with constant value lists;
  dynamic pivoting requires `EXECUTE IMMEDIATE`. `UNNEST` of arrays of structs is the idiomatic
  unpivot.
- **Spark/Databricks**: `DataFrame.pivot()` exists; Spark SQL added a `PIVOT` clause in recent
  versions, and `stack()` / `UNPIVOT`-style functions cover the reverse.

---

## 9. Exercises — Intermediate (hints only)

1. **One row per `country_code` with a column per status (counts), using `CASE`.**
   *Hint:* `COUNT(CASE WHEN … THEN 1 END)`; add a `total` column and explain why `SUM` of the
   status columns equals `total`.
2. **One row per month with a column per channel (revenue).** *Hint:* pre-aggregate to
   `(month, channel)`, then pivot; `COALESCE` the empty cells.
3. **Unpivot a wide `(id, jan, feb, mar)` table into tidy rows.** *Hint:* `CROSS JOIN LATERAL
   (VALUES …)`; then compare with the `UNION ALL` version's plan.
4. **Pivot "has the customer ever ordered from channel X" per customer.** *Hint:*
   `MAX(CASE WHEN … THEN 1 ELSE 0 END)` per channel; then discuss the `0` versus `NULL` choice.
5. **Show the difference between `SUM(CASE … END)` and `SUM(CASE … ELSE 0 END)` for a
   category with no rows in a status.** *Hint:* `NULL` vs `0`, and the effect on a downstream
   `AVG`.
6. **Write the native `PIVOT` version of exercise 1 for SQL Server or Snowflake and list three
   things you lost (portability, NULL control, column order).** *Hint:* the comparison table in
   Example 6.
7. **Produce a two-metric cell: revenue *and* order count per (country, status).** *Hint:* two
   `CASE`s per status, or two `PIVOT` blocks in SQL Server/Oracle.
8. **Unpivot with NULL preservation and verify the row count is `rows × columns`.**
   *Hint:* manual `VALUES` form vs native `UNPIVOT` and its NULL default.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Build a cohort retention grid with a dynamic number of `month_index` columns.**
   Requirements: works on PostgreSQL, degrades gracefully as cohorts age, and does not require
   editing the query each month. *Approach:* compare three options — static `CASE` columns up to
   month 12 plus a `month_13_plus` bucket; JSON aggregation; dynamic SQL. Then state which you
   would ship in a dbt model and why (hint: a fixed window plus a roll-up bucket keeps the
   schema stable).

2. **Write a generic `pivot` helper for PostgreSQL.** A function taking `(source_sql text,
   row_key_cols text[], pivot_col text, value_sql text)` that returns a dynamically shaped
   result (or JSON), using `EXECUTE format(...)`. *Approach:* acknowledge that a function cannot
   change its declared row type; the idiomatic solution returns `jsonb` or `refcursor` /
   `SETOF record` with a caller-provided column definition list (as `crosstab` does).

3. **Reconcile two sources with unpivot → compare → pivot.** Given two wide tables with the
   same shape (e.g. warehouse revenue per month vs gateway revenue per month), produce a
   variance matrix. *Approach:* unpivot both to `(key, month, amount)`, `FULL OUTER JOIN` on
   `(key, month)`, compute variance, then pivot back for presentation; explain why the unpivot
   step is what makes the comparison correct.

4. **Prove the `UNPIVOT` NULL trap and its impact on a metric.** Show a query where dropping
   NULLs changes a `COUNT(*)`, an `AVG`, and a `SUM`; then fix it three ways (native
   `INCLUDE NULLS`, manual `VALUES`, and a sentinel value with a documented meaning).
   *Approach:* construct a small table with all-NULL rows and compare outputs; explain which
   fix preserves the ability to distinguish "0" from "unknown".

5. **Optimize a 40-column unpivot on a 200M-row table.** Compare: `UNION ALL` of 40
   projections, `CROSS JOIN LATERAL (VALUES …)`, native `UNPIVOT`, and a two-step approach
   (narrow CTAS then unpivot). *Approach:* measure scans and I/O; explain why the two-step
   approach often wins (materializing the narrow projection once, then fanning out).

6. **Design a feature table for a churn model.** 30 behaviour features, one row per customer,
   all numeric, no NULLs, refreshed daily. *Approach:* one `SUM(CASE …)` per feature over a
   pre-aggregated event/order CTE; discuss the cost of 30 conditional aggregates in one pass
   versus 30 models, and how to keep the column set versioned.

7. **Compare `crosstab` versus JSON versus dynamic SQL for a genuinely unknown pivot key set**
   on PostgreSQL. Give a decision rule with three criteria (consumer type, schema stability,
   governance). *Approach:* name the failure modes: fixed types, query-per-value-list, and
   injection/quoting risk in dynamic SQL.

8. **Explain why `PIVOT ... IN (SELECT DISTINCT …)` is not allowed in most engines** and design
   the workaround for SQL Server and BigQuery. *Approach:* static SQL requires the output arity
   to be known at parse time; dynamic SQL moves that decision to runtime. Then discuss why
   `ANY` (Snowflake/Oracle) is only a partial solution (output arity still varies, which breaks
   typed consumers).

---

## References

**PostgreSQL**
- `tablefunc` (`crosstab`, `crosstabN`) for static pivotting: https://www.postgresql.org/docs/current/tablefunc.html
- `SELECT` (`LATERAL`, `VALUES`, `GROUP BY` semantics): https://www.postgresql.org/docs/current/sql-select.html
- Aggregate expressions and the `FILTER` clause: https://www.postgresql.org/docs/current/sql-expressions.html

**MySQL**
- `WITH`/derived tables and `VALUES` (`LATERAL` derived tables for the unpivot idiom): https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html
- `JSON_OBJECTAGG` (dynamic/semi-wide output): https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html

**SQL Server**
- `FROM` – Using PIVOT and UNPIVOT (including the NULL-dropping behavior of `UNPIVOT`): https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot
- `FROM` clause (`CROSS APPLY`/`OUTER APPLY`, `PIVOT` syntax): https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql

**Oracle**
- `SELECT` (PIVOT/UNPIVOT, `INCLUDE NULLS`, XML output for dynamic lists): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake `PIVOT` (including `IN (ANY)`): https://docs.snowflake.com/en/sql-reference/constructs/pivot
- Snowflake `UNPIVOT` (`INCLUDE NULLS`): https://docs.snowflake.com/en/sql-reference/constructs/unpivot
- Snowflake `LATERAL`/`FLATTEN` (semi-structured unpivot): https://docs.snowflake.com/en/sql-reference/functions/flatten
- BigQuery query syntax (`PIVOT`, `UNPIVOT`, `UNNEST`): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Modern SQL, *Pivot* (the portable literal-`CASE` technique and engine support): https://modern-sql.com/use-case/pivot
- Modern SQL, *VALUES clause* (used by the `LATERAL (VALUES …)` unpivot): https://modern-sql.com/feature/values

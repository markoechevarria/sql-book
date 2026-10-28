# CASE Expressions

> `CASE` is the only conditional construct in the SQL standard, and it is far more powerful
> than the `if/else` it resembles: it is a *row-level* expression usable in the `SELECT`
> list, `WHERE`, `JOIN ... ON`, `GROUP BY`, `ORDER BY` and inside aggregates. With
> conditional aggregation it turns a single scan into a dozen metrics (pivot tables without
> `PIVOT`), and with `COALESCE`/`NULLIF` it is the standard tool for cleaning dirty data.
> This chapter covers the two syntaxes, the NULL traps, type-unification rules, and the
> performance characteristics that decide whether `CASE` is free or catastrophic.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- **Searched** `CASE WHEN <predicate> THEN … END` is the general form; **simple**
  `CASE <expr> WHEN <value> THEN … END` is shorthand for `CASE WHEN <expr> = <value>`.
- Simple `CASE` uses **equality**, so `CASE x WHEN NULL THEN …` **never matches** (`NULL = NULL`
  is `UNKNOWN`). Handle NULLs with a searched `CASE WHEN x IS NULL …`.
- A `CASE` **without `ELSE`** returns `NULL` for unmatched rows. That is the #1 source of
  silent NULLs in reports.
- **Conditional aggregation** (`SUM(CASE WHEN … THEN 1 ELSE 0 END)`, `COUNT(CASE WHEN … THEN 1 END)`)
  is how you pivot with portable SQL. `COUNT` ignores NULLs, so `COUNT(CASE WHEN … THEN 1 END)`
  needs no `ELSE`.
- The result type of a `CASE` is the **common supertype of all branches** — mixing `int` and
  `text`, or `date` and `numeric`, forces implicit casts (and can error). Always cast
  explicitly when branches disagree.
- `CASE` is evaluated **per row**, not short-circuited like a procedural `if`. Subexpressions
  that are *not* needed are not evaluated (per the SQL standard and PostgreSQL docs), but
  **constant (IMMUTABLE) subexpressions can be folded at plan time**, so a `CASE` does not
  reliably protect you from a plan-time error like `1/0`.
- In `WHERE`/`ON`, `CASE` around the *filtered column* disables index usage
  (`WHERE CASE WHEN … THEN col END = 5`). Rewrite as an `OR`/range so the optimizer can use
  the index. `CASE` around a *constant threshold*, or in the `SELECT` list, is free.
- `COALESCE` = first non-NULL; `NULLIF(a,b)` = NULL if `a = b` (the division-by-zero guard);
  `GREATEST`/`LEAST` NULL semantics **differ by engine** — see §8.

---

## 1. Definitions and core concepts

### 1.1 The two syntaxes

```sql
-- Simple CASE: compares one expression against a list of values (uses `=`)
CASE <expr>
     WHEN <value1> THEN <result1>
     WHEN <value2> THEN <result2>
     [ ELSE <default> ]
END

-- Searched CASE: evaluates arbitrary boolean predicates in order
CASE
     WHEN <predicate1> THEN <result1>
     WHEN <predicate2> THEN <result2>
     [ ELSE <default> ]
END
```

Semantics of the searched form: **the first `WHEN` whose predicate is `TRUE` wins**;
subsequent branches are not evaluated; if none is `TRUE` the result is the `ELSE` (or `NULL`).
A predicate that evaluates to `UNKNOWN` (because of `NULL`) is treated as *not TRUE*.

Simple form ≡ `CASE WHEN expr = value1 THEN … WHEN expr = value2 THEN …` — and if `expr`
is `NULL`, every comparison is `UNKNOWN`, so only `ELSE` can fire.

### 1.2 Conditional expressions and functions

| Construct | Semantics | Notes |
|---|---|---|
| `CASE … END` | general conditional | expressions and/or predicates |
| `COALESCE(a, b, c, …)` | first argument that `IS NOT NULL` | standard; variadic |
| `NULLIF(a, b)` | `NULL` if `a = b`, else `a` | standard; the divide-by-zero guard |
| `GREATEST(a, b, …)` | maximum (NULL rules vary) | non-standard; engine-specific NULL behavior |
| `LEAST(a, b, …)` | minimum (NULL rules vary) | same |
| `IF(cond, a, b)` | ternary | MySQL, BigQuery, Snowflake, Hive — **not** Postgres/SQL Server/Oracle |
| `IFNULL(a,b)` / `NVL(a,b)` / `ISNULL(a,b)` | two-arg `COALESCE` | MySQL / Oracle / SQL Server respectively |
| `DECODE(expr, v1, r1, …, default)` | Oracle's simple `CASE` (with `NULL`-equals-`NULL` semantics!) | Oracle; `DECODE` treats two NULLs as equal — a real difference from `CASE` |
| `IIF(cond,a,b)` | SQL Server ternary | SQL Server, BigQuery |

---

## 2. Why it matters / what problem it solves

1. **Deriving dimensions.** Bucket customers into cohorts, orders into value bands, sessions
   into channels — all `CASE`.
2. **Pivoting without `PIVOT`.** `SUM(CASE WHEN status='delivered' THEN amount END)` gives one
   column per status from one scan. This is the *portable* pivot (see
   [08-pivot-unpivot.md](./08-pivot-unpivot.md)).
3. **Data cleaning.** `NULLIF(TRIM(x), '')`, `COALESCE(a, b, 0)`, sentinel replacement
   (`NULLIF(x, -1)`), and deduping whitespace/casing.
4. **Safe division and arithmetic.** `x / NULLIF(y, 0)` instead of a runtime error.
5. **Conditional sorting.** `ORDER BY CASE WHEN priority='high' THEN 0 ELSE 1 END` gives a
   business-defined sort order that is not alphabetical.
6. **Conditional joins.** `ON t.id = CASE WHEN a.type='x' THEN a.id1 ELSE a.id2 END` (rare, and
   usually a modeling smell — but sometimes the only way).
7. **Guarding expensive expressions.** `CASE WHEN jsonb_typeof(x) = 'object' THEN x->>'k' END`
   avoids errors on non-object payloads.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Evaluation model

`CASE` is an **expression**, not a statement. It is evaluated once per row of the operator
that produces it, after the `FROM`/`WHERE`/`GROUP BY` stages that produced the row:

```
FROM/JOIN → WHERE → GROUP BY → HAVING → (window fns) → SELECT [CASE evaluated here]
          → DISTINCT → ORDER BY [CASE re-evaluated or reused] → LIMIT
```

Key consequences:

- In the `SELECT` list, `CASE` contributes no I/O and no sorts — it is very cheap.
- In `WHERE`, a `CASE` **wrapping a filtered column** makes the predicate non-sargable: the
  engine cannot use a B-tree index on that column, because the index is on the column, not on
  the `CASE` result.
- In `GROUP BY`, a `CASE` expression defines the grouping key; engines that allow positional
  references let you write `GROUP BY 1`, but the expression itself may be recomputed.
- In `ORDER BY`, `CASE` produces a sort key; a `CASE` that returns different types across
  branches will error (unlike `NULL` which coerces).

### 3.2 Short-circuiting: what you can and cannot rely on

The PostgreSQL documentation states:

> A `CASE` expression does not evaluate any subexpressions that are not needed to determine
> the result. For example, this is a possible way of avoiding a division-by-zero failure:
> `CASE WHEN x <> 0 THEN y/x ELSE 0 END`.

But there are two important caveats:

1. **Constant subexpressions can be folded at plan time.** An expression that is `IMMUTABLE`
   and involves only constants (e.g. `1/0` or `'x'::uuid`, `length(uuid_col)`) may be
   evaluated while planning, *before* the `CASE` is ever run — so the error fires anyway.
   This is the classic "the CASE was supposed to protect me" confusion.
2. **Aggregates in a `CASE` are computed before the `CASE` chooses.** `CASE WHEN COUNT(*) > 0
   THEN SUM(x)/COUNT(*) ELSE 0 END` computes `SUM` and `COUNT` regardless. Guard the
   *denominator* with `NULLIF`, not the numerator.

Rule of thumb: `CASE` short-circuits **row-level** conditional evaluation reliably, but you
cannot use it as a control-flow guard for plan-time errors or to avoid aggregate work.

### 3.3 Type unification

The result type of a `CASE` is determined by the *least common supertype* of all `THEN`/`ELSE`
branches. Practical implications:

| Branches | PostgreSQL | Note |
|---|---|---|
| `integer`, `numeric` | `numeric` | safe upcast |
| `date`, `timestamptz` | `timestamptz` | implicit conversion, timezone applies |
| `text`, `integer` | **error** (no implicit cast) | cast one branch: `THEN x::text` |
| `text`, `unknown literal` (`'n/a'`) | `text` | string literals adapt |
| `NULL`, anything | the other type | `THEN NULL` is fine |
| `boolean`, `integer` (MySQL) | `integer` | MySQL booleans are `TINYINT(1)` |

`> Senior note`: in a data pipeline, an implicit type change in a `CASE` branch can silently
alter a column's type in a downstream `CREATE TABLE AS SELECT` — and downstream BI tools then
treat a numeric as text. Always cast explicitly at branch boundaries.

---

## 4. Syntax and mechanics (with examples)

### 4.1 Simple vs searched

```sql
-- Simple: one expression, equality against values
SELECT order_id,
       CASE status
            WHEN 'delivered' THEN 'complete'
            WHEN 'returned'  THEN 'refund'
            ELSE 'in_progress'
       END AS state
FROM orders;

-- Searched: arbitrary predicates, first TRUE wins
SELECT order_id,
       CASE
            WHEN status IN ('delivered','shipped') THEN 'on_track'
            WHEN status = 'cancelled'              THEN 'lost'
            WHEN ordered_at < now() - interval '30 days' THEN 'stale'
            ELSE 'open'
       END AS state
FROM orders;
```

The searched form is a **superset** — anything the simple form does, it can do — so prefer it
when NULLs are possible or when the conditions are non-equality.

### 4.2 The NULL trap in simple `CASE`

```sql
-- WRONG: loyalty_tier is NULL for Heidi (customer 8) → falls through to ELSE
SELECT customer_id,
       CASE loyalty_tier WHEN NULL THEN 'none' ELSE loyalty_tier END AS tier_bug
FROM customers;

-- RIGHT: searched form with IS NULL
SELECT customer_id,
       CASE WHEN loyalty_tier IS NULL THEN 'none' ELSE loyalty_tier END AS tier_ok
FROM customers;
```

Expected: `tier_bug` returns `NULL` for customer 8 (not `'none'`), while `tier_ok` returns
`'none'`. Oracle's `DECODE` has the opposite behavior (NULL matches NULL), which is a real
portability trap.

### 4.3 `CASE` in every clause

```sql
-- SELECT: derive a column
SELECT product_id, unit_price,
       CASE WHEN unit_price >= 1000 THEN 'premium'
            WHEN unit_price >= 100 THEN 'mid'
            ELSE 'economy' END AS price_band
FROM products;

-- WHERE: filter on a derived condition (non-sargable — see §7)
SELECT * FROM orders
WHERE CASE WHEN channel IN ('web','mobile') THEN 'digital' ELSE 'physical' END = 'digital';

-- ORDER BY: business-defined ordering
SELECT order_id, status FROM orders
ORDER BY CASE status
              WHEN 'pending'   THEN 1
              WHEN 'paid'      THEN 2
              WHEN 'shipped'   THEN 3
              WHEN 'delivered' THEN 4
              WHEN 'returned'  THEN 5
              ELSE 6
         END, ordered_at DESC;

-- GROUP BY: define the bucket, then aggregate
SELECT CASE WHEN ordered_at::time < time '12:00' THEN 'morning' ELSE 'afternoon' END AS part_of_day,
       COUNT(*) AS orders
FROM orders GROUP BY 1 ORDER BY 1;

-- JOIN ... ON: conditional join key (use sparingly)
SELECT a.order_id, b.label
FROM orders a
JOIN dims b
  ON b.key = CASE WHEN a.channel IN ('web','mobile') THEN 'digital' ELSE a.channel END;
```

### 4.4 Conditional aggregation (the portable pivot)

```sql
SELECT cat.name AS category,
       COUNT(*)                                                        AS lines,
       COUNT(CASE WHEN o.status = 'delivered' THEN 1 END)              AS delivered_lines,
       SUM(CASE WHEN o.status = 'delivered'
                THEN i.quantity * i.unit_price * (1 - i.discount_pct/100) END) AS delivered_revenue,
       SUM(CASE WHEN o.status = 'returned'
                THEN i.quantity * i.unit_price * (1 - i.discount_pct/100) END) AS returned_revenue,
       ROUND(100.0 * COUNT(CASE WHEN o.status = 'returned' THEN 1 END) / COUNT(*), 1) AS return_rate_pct
FROM order_items i
JOIN orders     o   ON o.order_id = i.order_id
JOIN products   p   ON p.product_id = i.product_id
JOIN categories cat ON cat.category_id = p.category_id
GROUP BY cat.name
ORDER BY delivered_revenue DESC NULLS LAST;
```

Notes:
- `COUNT(CASE WHEN … THEN 1 END)` relies on `COUNT` ignoring NULLs → no `ELSE` needed.
- `SUM(CASE WHEN … THEN x END)` returns `NULL` when no rows match; wrap in `COALESCE(…, 0)`
  if a zero is required.
- PostgreSQL, Snowflake and DuckDB also support the **`FILTER` clause**, which is cleaner:
  `SUM(x) FILTER (WHERE status='delivered')`.

### 4.5 `COALESCE`, `NULLIF`, `GREATEST`, `LEAST`

```sql
SELECT c.full_name,
       COALESCE(c.loyalty_tier, 'none')                     AS tier,          -- fill defaults
       NULLIF(TRIM(c.email), '')                            AS clean_email,   -- '' → NULL
       GREATEST(SUM(i.quantity * i.unit_price), 0)          AS revenue_floor  -- clamp negatives
FROM customers c
LEFT JOIN orders o      ON o.customer_id = c.customer_id AND o.status IN ('paid','shipped','delivered')
LEFT JOIN order_items i ON i.order_id = o.order_id
GROUP BY c.customer_id, c.full_name, c.loyalty_tier, c.email
ORDER BY revenue_floor DESC;

-- Safe division ratio: conversion rate with a guaranteed non-null, non-error denominator
SELECT channel,
       COUNT(CASE WHEN status = 'delivered' THEN 1 END)                                        AS delivered,
       COUNT(*)                                                                                AS total,
       ROUND(100.0 * COUNT(CASE WHEN status = 'delivered' THEN 1 END) / NULLIF(COUNT(*), 0), 1) AS pct
FROM orders GROUP BY channel ORDER BY channel;
```

`COALESCE` is also the standard way to implement **carry-forward / last-known-value** when
`IGNORE NULLS` is unavailable:

```sql
-- Portable last-non-null-value: group NULLs with their preceding value, then take MAX per group
WITH g AS (
    SELECT event_id, properties->>'utm' AS utm,
           COUNT(properties->>'utm') OVER (ORDER BY occurred_at) AS grp
    FROM events
)
SELECT event_id,
       FIRST_VALUE(utm) OVER (PARTITION BY grp ORDER BY event_id) AS carried_utm
FROM g ORDER BY event_id;
```

### 4.6 Guards and validation (a data-quality pattern)

```sql
SELECT o.order_id,
       CASE
            WHEN o.shipped_at IS NULL AND o.status IN ('shipped','delivered')
                 THEN 'ERROR: shipped without ship date'
            WHEN o.shipped_at < o.ordered_at
                 THEN 'ERROR: ship before order'
            WHEN o.status = 'delivered' AND o.shipped_at IS NULL
                 THEN 'ERROR: delivery without shipment'
            ELSE 'ok'
       END AS dq_flags,
       CASE WHEN o.shipped_at IS NOT NULL
            THEN EXTRACT(epoch FROM o.shipped_at - o.ordered_at) / 86400.0 END AS days_to_ship
FROM orders o;
```

This is the shape of a **DQ audit query**: one row per record, one column per rule, and an
`ELSE 'ok'`. It is far cheaper than running each rule as a separate query.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: label order statuses for a business report

```sql
SELECT order_id, status, ordered_at, channel,
       CASE status
            WHEN 'pending'   THEN 'Not paid yet'
            WHEN 'paid'      THEN 'Awaiting fulfilment'
            WHEN 'shipped'   THEN 'In transit'
            WHEN 'delivered' THEN 'Completed'
            WHEN 'returned'  THEN 'Refunded'
            WHEN 'cancelled' THEN 'Cancelled'
            ELSE 'Unknown: ' || status
       END AS status_label
FROM orders ORDER BY order_id;
```

Expected: 14 rows, `ELSE` never fires here (the `CHECK` constraint prevents unknown statuses) —
which is itself a useful observation: a defensive `ELSE` documents the assumption.

### Example 2 — Basic: revenue by value band per customer

```sql
SELECT c.full_name,
       CASE
            WHEN COALESCE(SUM(i.quantity * i.unit_price), 0) = 0     THEN 'no_purchases'
            WHEN SUM(i.quantity * i.unit_price) < 100                THEN 'tiny'
            WHEN SUM(i.quantity * i.unit_price) < 1000               THEN 'small'
            WHEN SUM(i.quantity * i.unit_price) < 5000               THEN 'medium'
            ELSE 'large'
       END AS ltv_band,
       COALESCE(SUM(i.quantity * i.unit_price), 0) AS ltv
FROM customers c
LEFT JOIN orders o      ON o.customer_id = c.customer_id
                       AND o.status IN ('paid','shipped','delivered')
LEFT JOIN order_items i ON i.order_id = o.order_id
GROUP BY c.customer_id, c.full_name
ORDER BY ltv DESC;
```

Expected, ordered by `ltv` descending: `Dave Brown / medium / 2108.97` (2028.99 delivered +
79.98 shipped), `Frank Miller / medium / 1736.00`, `Alice Johnson / medium / 1077.95`
(order 1001's 999.00 + 2 × 19.99 + order 1002's 3 × 12.99), `Bob Smith / small / 899.00` (his
returned order is excluded), `Erin Davis / small / 799.00`, `Carol White / small / 134.00`
(only the `paid` order counts; 1006 is `cancelled`), `Grace Lee / tiny / 32.98`,
`Heidi Novak / no_purchases / 0`. Note the predicate placement in `ON` — filtered orders must
not eliminate the customer row (the trap from [Chapter 01](./01-complex-joins.md)). This query
does *not* apply `discount_pct`, which is why Alice's figure differs from the discounted
totals used in other chapters.

### Example 3 — Intermediate: pivot order counts by status

```sql
SELECT country_code,
       COUNT(*)                                                       AS total_orders,
       COUNT(CASE WHEN status = 'pending'   THEN 1 END)               AS pending,
       COUNT(CASE WHEN status = 'paid'      THEN 1 END)               AS paid,
       COUNT(CASE WHEN status = 'shipped'   THEN 1 END)               AS shipped,
       COUNT(CASE WHEN status = 'delivered' THEN 1 END)               AS delivered,
       COUNT(CASE WHEN status = 'returned'  THEN 1 END)               AS returned,
       COUNT(CASE WHEN status = 'cancelled' THEN 1 END)               AS cancelled
FROM orders
GROUP BY country_code
ORDER BY total_orders DESC, country_code;
```

Expected: `US` has 6 orders (1001, 1002, 1005, 1006, 1011, 1012, 1014 → 7 actually: count from
the seed), `DE` 2, `GB` 2, `FR` 2, `JP` 1. The pivot produces a **one-row-per-country
cross-tab** from a single scan — this is the manual pivot you will use on engines without
`PIVOT` (PostgreSQL, older MySQL, BigQuery before `PIVOT`).

### Example 4 — Intermediate: funnel with conditional aggregation

```sql
WITH session_flags AS (
    SELECT session_id, customer_id,
           MAX(CASE WHEN event_type = 'page_view'   THEN 1 ELSE 0 END) AS viewed,
           MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS carted,
           MAX(CASE WHEN event_type = 'checkout'    THEN 1 ELSE 0 END) AS checked_out,
           MAX(CASE WHEN event_type = 'purchase'    THEN 1 ELSE 0 END) AS purchased
    FROM events GROUP BY session_id, customer_id
)
SELECT COUNT(*)                                                        AS sessions,
       SUM(viewed)                                                     AS s1_viewed,
       SUM(carted)                                                     AS s2_carted,
       SUM(checked_out)                                                AS s3_checkout,
       SUM(purchased)                                                  AS s4_purchased,
       ROUND(100.0 * SUM(carted)      / NULLIF(SUM(viewed),0), 1)       AS pct_to_cart,
       ROUND(100.0 * SUM(checked_out) / NULLIF(SUM(carted),0), 1)       AS pct_to_checkout,
       ROUND(100.0 * SUM(purchased)   / NULLIF(SUM(checked_out),0), 1)  AS pct_to_purchase
FROM session_flags;
```

Expected (5 sessions): `s1_viewed = 5` (two Alice sessions, Bob, Heidi, one anonymous),
`s2_carted = 2` (Alice's two sessions), `s3_checkout = 2` (Alice's February session and Bob's),
`s4_purchased = 2` (Alice's two sessions) → `pct_to_cart = 40.0`, `pct_to_checkout = 100.0`,
`pct_to_purchase = 100.0`. Note that Bob's session checks out but never purchases, so a
"checkout → purchase" conversion below 100 % is the realistic case; the seed deliberately
contains both shapes.

### Example 5 — Intermediate: business-defined ordering and NULL-safe sorting

```sql
SELECT status, COUNT(*) AS orders
FROM orders
GROUP BY status
ORDER BY CASE status
              WHEN 'pending' THEN 1
              WHEN 'paid'    THEN 2
              WHEN 'shipped' THEN 3
              WHEN 'delivered' THEN 4
              WHEN 'returned'  THEN 5
              ELSE 99
         END;
```

Also, to put NULLs last *by business choice* rather than by engine defaults:

```sql
SELECT customer_id, loyalty_tier
FROM customers
ORDER BY CASE WHEN loyalty_tier IS NULL THEN 1 ELSE 0 END, loyalty_tier;
```

Engine defaults differ (`NULLS LAST` is the default for `ASC` in Postgres/Oracle but
`NULLS FIRST` is lowest in MySQL/SQL Server), so an explicit `CASE` makes the intent portable.

### Example 6 — Advanced: full pivot without `PIVOT` (channels × statuses)

```sql
SELECT channel,
       COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
       COUNT(*) FILTER (WHERE status = 'returned')  AS returned,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
       ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'returned') / NULLIF(COUNT(*),0), 1) AS return_pct
FROM orders
GROUP BY channel
ORDER BY channel;
```

Expected: `store → 2 delivered, 0 returned, 0 cancelled, 0.0`; `marketplace → 1 delivered`;
`mobile → 1 delivered?` (read the seed: mobile has 1002 delivered, 1005 paid, 1010 pending,
1013 paid); `web → 1001 delivered, 1003 delivered, 1004 returned, 1006 cancelled, 1008
shipped, 1009 paid, 1014 pending`. Because a `FILTER` clause is not universal, the
`CASE` form (`COUNT(CASE WHEN … THEN 1 END)`) is what you should write when portability
matters.

### Example 7 — Advanced: derive a sentinel-free column plus a "reason" code

```sql
WITH raw AS (
    SELECT o.order_id,
           o.status,
           o.ordered_at,
           o.shipped_at,
           COALESCE(o.shipped_at, o.ordered_at) AS effective_ship
    FROM orders o
)
SELECT r.order_id,
       -- ship latency in days, NULL when not shipped
       CASE WHEN r.shipped_at IS NULL THEN NULL
            ELSE ROUND(EXTRACT(epoch FROM r.shipped_at - r.ordered_at)/86400.0, 2) END AS days_to_ship,
       -- a compact reason code, using a nested CASE
       CASE
            WHEN r.shipped_at IS NULL AND r.status IN ('returned','cancelled') THEN NULL
            WHEN r.shipped_at IS NULL THEN 'NOT_SHIPPED'
            WHEN EXTRACT(epoch FROM r.shipped_at - r.ordered_at)/86400.0 > 3 THEN 'LATE'
            WHEN EXTRACT(epoch FROM r.shipped_at - r.ordered_at)/86400.0 > 1 THEN 'SLOW'
            ELSE 'OK'
       END AS ship_quality,
       -- nested CASE inside a CASE branch
       CASE WHEN r.status IN ('returned','cancelled')
                 THEN CASE WHEN r.shipped_at IS NULL THEN 'lost before ship' ELSE 'refunded' END
            ELSE 'active'
       END AS lifecycle
FROM raw r
ORDER BY r.order_id;
```

Expected: `1001` ≈ 1.95 days → `SLOW`; `1002` ≈ 1.74 → `SLOW`; `1003` ≈ 1.99 → `SLOW`;
`1004` ≈ 1.60 → `SLOW` / lifecycle `refunded` (returned after shipping); `1005` →
`NOT_SHIPPED` / `active`; `1006` → `NOT_SHIPPED` / lifecycle `lost before ship` (cancelled
before shipping); `1007` ≈ 2.10 → `SLOW`; `1008` ≈ 1.81 → `SLOW`; `1009`, `1010`, `1013`,
`1014` → `NOT_SHIPPED`; `1011` ≈ 0.83 → `OK`; `1012` ≈ 0.68 → `OK`. The lesson: a **nested
`CASE`** expresses multi-level decision tables without procedural code.

### Example 8 — Advanced: `CASE` versus a lookup table (maintainability)

```sql
-- Hard-coded bands: fast, but a business change requires a deploy
SELECT p.name, p.unit_price,
       CASE WHEN p.unit_price < 50   THEN 'budget'
            WHEN p.unit_price < 500  THEN 'standard'
            WHEN p.unit_price < 1500 THEN 'premium'
            ELSE 'flagship' END AS price_band
FROM products p;

-- Data-driven: the band table is editable without a deploy (and joins fine)
SELECT p.name, p.unit_price, b.label AS price_band
FROM products p
LEFT JOIN value_bands b
       ON p.unit_price >= b.low AND p.unit_price < b.high   -- non-equi join
ORDER BY p.unit_price DESC;
```

Use `CASE` when the rule is *structural* (e.g. `IS NULL` handling, funnel step flags) and a
lookup table when the rule is *business data* that changes without code review.

---

## 6. Real-world use cases and applications

| Use case | Pattern |
|---|---|
| **Cohort/bucket assignment** | `CASE WHEN x < a THEN 'A' … END` |
| **Pivot/cross-tab reports** | conditional aggregation with `COUNT/SUM(CASE …)` |
| **Funnel step flags** | `MAX(CASE WHEN event='x' THEN 1 ELSE 0 END)` per session |
| **Data quality audit** | one `CASE` per rule, `ELSE 'ok'` |
| **Sentinel/null cleaning** | `NULLIF(x, -1)`, `COALESCE(a,b,0)`, `TRIM`+`NULLIF` |
| **Safe math** | `x / NULLIF(y,0)`, `GREATEST(v, 0)` |
| **Business sort order** | `ORDER BY CASE status WHEN … THEN n END` |
| **Numeric → ordinal mapping** | `CASE WHEN n BETWEEN 1 AND 3 THEN 'low' … END` |
| **SCD classification** | `CASE WHEN changed_col IS DISTINCT FROM prev THEN 'new_version' ELSE 'same' END` |
| **Row-level access control** | `WHERE CASE WHEN current_user IN ('admin') THEN true ELSE owner_id = current_user END` |
| **Attribution fallback chains** | `COALESCE(last_click, session_source, 'direct')` |
| **Rate limiting / alert thresholds** | `CASE` to grade a metric as `ok/warn/critical` |

---

## 7. Performance notes and anti-patterns

### 7.1 Is `CASE` free?

`CASE` in the `SELECT` list is essentially free (a branch per row). It becomes expensive when:

1. **It is in `WHERE`/`ON` and wraps an indexed column** → the index cannot be used, forcing
   a sequential scan. Example:

   ```sql
   -- BAD: no index on the CASE result; seq scan on orders
   SELECT * FROM orders
   WHERE CASE WHEN channel IN ('web','mobile') THEN 'digital' ELSE 'physical' END = 'digital';

   -- GOOD: same semantics, sargable
   SELECT * FROM orders WHERE channel IN ('web','mobile');

   -- BAD: wraps an indexed column
   WHERE CASE WHEN status = 'delivered' THEN ordered_at END >= '2024-03-01'
   -- GOOD: rewrite as a compound predicate the index can use
   WHERE status = 'delivered' AND ordered_at >= '2024-03-01';
   ```

2. **It contains an expensive subquery or function per branch** → evaluated per row.
   Guard it (`CASE WHEN cond THEN expensive(x) END`) — that guard *does* short-circuit at row
   level, which is the legitimate use of `CASE` as a guard.
3. **It is used to emulate a partition key in `GROUP BY` on a huge table** → the engine must
   compute the expression for every row (cheap) but the grouping may lose index support.
   Prefer a generated/stored column with an index when the expression is stable.

### 7.2 Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| `CASE x WHEN NULL THEN …` | branch never fires | searched `CASE WHEN x IS NULL` |
| Missing `ELSE` | silent NULLs | add `ELSE` or `COALESCE` the result |
| `CASE` in `WHERE` on the indexed column | index not used | rewrite as `OR`/range/`IN` |
| Branch types differ (`int` vs `text`) | implicit cast surprises, downstream type drift | explicit `::type` on each branch |
| `CASE` to avoid a divide-by-zero in an *aggregate* | error still occurs (aggregate computed first) | guard the denominator with `NULLIF` |
| Repeating the same `CASE` five times in one query | drift between copies | one CTE / named expression |
| `CASE WHEN x IS NOT NULL THEN 'y' END` style — duplicated in metrics | unreadable | conditional aggregation + comments per metric |
| Deeply nested `CASE` (10+ levels) | unmaintainable | lookup table or a `VALUES`-based mapping joined in |
| `CASE` inside `ORDER BY` on a large result set | sort key computed per row (usually fine) | keep it, but check the plan if sorting spills |
| `CASE` for control flow that should be a `WHERE` | scans more rows than needed | push the predicate down |

### 7.3 The `VALUES`-list alternative

For a large, static mapping, a `VALUES` list joined in is often cleaner *and* faster to review:

```sql
WITH mapping(status, sort_order, label) AS (
    VALUES ('pending',1,'Not paid yet'),
           ('paid',2,'Awaiting fulfilment'),
           ('shipped',3,'In transit'),
           ('delivered',4,'Completed'),
           ('returned',5,'Refunded'),
           ('cancelled',6,'Cancelled')
)
SELECT o.order_id, m.label
FROM orders o LEFT JOIN mapping m USING (status)
ORDER BY m.sort_order, o.ordered_at DESC;
```

This gives you the same result with an *editable* mapping and a single `LEFT JOIN`
(note the `LEFT JOIN` so an unexpected status still returns a row).

---

## 8. Cross-engine differences

| Feature | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Simple `CASE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Searched `CASE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `IF(cond,a,b)` | ❌ | ✅ | ❌ (`IIF`) | ❌ (`DECODE`) | ✅ | ✅ |
| `IIF` | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ |
| `COALESCE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `NULLIF` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2-arg shorthands | — | `IFNULL` | `ISNULL` | `NVL` | `IFNULL`/`NVL` | `IFNULL` |
| `GREATEST`/`LEAST` | ✅, **ignores NULLs** | ✅, **returns NULL if any arg is NULL** | ✅ 2022+, ignores NULLs | ✅, returns NULL if any arg is NULL | ✅, returns NULL if any arg is NULL (use `GREATEST_IGNORE_NULLS`) | ✅, returns NULL if any arg is NULL |
| `DECODE` (NULL = NULL equality) | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Boolean type | ✅ real `boolean` | `TINYINT(1)` (booleans are integers) | `bit` | ❌ (use `NUMBER(1)`) | ✅ | ✅ |
| `CASE` returning `boolean` | ✅ | ✅ (integer) | ❌ (use `CASE WHEN … THEN 1 ELSE 0 END`) | ❌ | ✅ | ✅ |
| `FILTER (WHERE …)` on aggregates | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `IS DISTINCT FROM` (NULL-safe equality) | ✅ | ✅ (`<=>` operator) | ❌ (use `EXCEPT`/`INTERSECT` or explicit NULL checks) | ❌ (use `DECODE`) | ✅ (`IS DISTINCT FROM`) | ❌ (use `IS NOT DISTINCT FROM`) |
| `PIVOT` built-in | ❌ (use `CASE`) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Conditional function evaluation order guaranteed? | statements in `CASE` are not evaluated when unnecessary | similar | `IIF` is *not* guaranteed to short-circuit (documented) | `CASE`/`DECODE` short-circuit | yes | yes |

Notable traps:

- **`GREATEST`/`LEAST` NULL semantics are not portable.** In PostgreSQL they ignore NULLs; in
  MySQL, Oracle, BigQuery and Snowflake a single NULL argument makes the whole result NULL.
  Write `GREATEST(COALESCE(a,0), COALESCE(b,0))` or a `CASE` chain if you need one behavior
  everywhere.
- **SQL Server's `IIF`** is documented as *not* guaranteed to short-circuit (it may evaluate
  both branches). Use `CASE` when the guard matters.
- **Oracle's `DECODE`** treats `NULL` as equal to `NULL`, while `CASE` does not. Migrating
  `DECODE` to `CASE` can silently change results.
- **SQL Server has no `boolean` type**: `CASE WHEN … THEN 'yes' ELSE 'no' END` is fine, but
  `CASE WHEN … THEN TRUE ELSE FALSE END` is not valid T-SQL; use `1`/`0` or `CAST(... AS bit)`.
- **`FILTER`** (PostgreSQL/Snowflake/DuckDB) is the cleanest pivot primitive where available,
  but it is not portable — keep a `CASE` version in your toolbox.

---

## 9. Exercises — Intermediate (hints only)

1. **Label customers `new`, `active`, `at_risk`, `churned` based on their last order date.**
   *Hint:* `CASE` over `MAX(ordered_at)` compared with `now() - interval '90 days'`; be
   explicit about what "no orders" means.
2. **Produce a one-row-per-country pivot of order counts by status.** *Hint:* conditional
   aggregation with `COUNT(CASE WHEN … THEN 1 END)`.
3. **Show each product's margin with a `CASE`-derived margin band.** *Hint:*
   `unit_price - cost`, then band it; also compute the margin *percentage* with `NULLIF`.
4. **Fix this and explain: `CASE loyalty_tier WHEN NULL THEN 'none' END`.** *Hint:* equality
   with NULL; rewrite as a searched `CASE`.
5. **Sort orders by a business priority, then by date descending.** *Hint:* `CASE` in
   `ORDER BY`; discuss why `ORDER BY` can use an alias but `WHERE` cannot.
6. **Compute the return rate per channel using one scan.** *Hint:*
   `100.0 * COUNT(CASE WHEN status='returned' THEN 1 END) / NULLIF(COUNT(*),0)`.
7. **Replace a chain of four `IFNULL`/`NVL` calls with one portable expression.** *Hint:*
   `COALESCE` is variadic; the shorthands are not.
8. **Build a DQ flag column for `orders` with at least four rules.** *Hint:* the audit pattern
   from §4.6; make rules mutually exclusive and testable.
9. **Map `channel` to a reporting group using a `VALUES` list instead of `CASE`.** *Hint:*
   CTE + `LEFT JOIN ... USING` so unknown channels survive.
10. **Show the difference between `SUM(CASE … THEN x END)` and `SUM(CASE … THEN x ELSE 0 END)`
   for a status that appears in no rows.** *Hint:* `NULL` vs `0`, and how it affects a
   downstream `COALESCE`/`AVG`.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Build a full status × channel cross-tab with row and column totals, without `PIVOT`.**
   *Approach:* one CTE of conditional aggregates, `UNION ALL` a totals row built with a
   second aggregate pass (or `GROUPING SETS` in §4 of the analytical chapter), and keep the
   column order deterministic by generating the *label* from a `VALUES` mapping.

2. **Implement a 3-step funnel with step-to-step conversion and drop-off attribution.**
   Requirements: per-session flags, per-step counts, per-step conversion versus step 1 *and*
   versus the previous step, and a `NULLIF`-guarded division. *Approach:* session flags via
   `MAX(CASE …)`; conversion via a single-row aggregate over the flags; discuss how to extend
   it to time-bounded funnels (step must occur within N minutes).

3. **Design a DQ framework column set.** One query over `orders` producing
   `dq_status`, `dq_codes` (comma-separated), and `dq_severity` (ordinal), where a record can
   fail several rules and severity is the maximum. *Approach:* one `CASE` per rule producing
   a text or NULL, `CONCAT_WS`/`STRING_AGG` to join codes, and a `CASE`-based severity fold.
   Then explain why computing all rules in one pass beats N separate queries.

4. **Make a `CASE`-based rule externally configurable.** Move price bands into
   `value_bands`, prove the `CASE` and join versions return identical results with `EXCEPT`,
   then discuss the failure mode of overlapping bands and how to test for it. *Approach:*
   two-directional `EXCEPT` (symmetric difference) and a `HAVING COUNT(*) > 1` overlap check.

5. **Prove that a `CASE` guard did not protect you from a plan-time error.** Construct a
   query where an `IMMUTABLE` expression containing `1/0` (or an invalid cast) is folded at
   plan time despite being inside `CASE`, and explain the mechanism (`IMMUTABLE` constant
   folding during planning). Then show a version using a `CASE` around a *volatile* function
   (e.g. `random()`) where the guard does work at row level. *Approach:* read the PostgreSQL
   documentation note about `IMMUTABLE` functions being evaluated during planning.

6. **Rewrite a non-sargable `CASE` predicate into an index-usable form** and prove it with
   `EXPLAIN (ANALYZE, BUFFERS)`. *Approach:* produce both plans, identify `Seq Scan` vs
   `Index Scan`/`Bitmap Heap Scan`, and quantify the buffers difference.

7. **Implement carry-forward (last-known-value) without `IGNORE NULLS`.** Given `events` with
   sparse `utm_source`, fill every event with the last non-null value per customer.
   *Approach:* running `COUNT(col) OVER (ORDER BY ts)` to group, then `MAX`/`FIRST_VALUE` per
   group; verify against a `LATERAL`-based or `IGNORE NULLS`-based implementation where the
   engine supports it.

8. **Decide between `CASE`, a lookup table, and a generated column** for a rule that is
   evaluated in `WHERE` on a 500M-row table. *Approach:* discuss sargability, storage, and
   refresh semantics; the expected answer involves a stored generated column plus an index
   (PostgreSQL 12+) or a materialized mapping table joined by an equi-key.

---

## References

**PostgreSQL**
- Conditional expressions (`CASE`, `COALESCE`, `NULLIF`, `GREATEST`, `LEAST`, and the
  "subexpressions are not evaluated" note): https://www.postgresql.org/docs/current/functions-conditional.html
- `SELECT` (where `CASE` may appear): https://www.postgresql.org/docs/current/sql-select.html

**MySQL**
- Flow control functions (`CASE`, `IF`, `IFNULL`, `NULLIF`): https://dev.mysql.com/doc/refman/8.0/en/flow-control-functions.html
- Comparison functions (`COALESCE`, `GREATEST`, `LEAST`, `ISNULL`): https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html

**SQL Server**
- `CASE` (Transact-SQL), including the `IIF` short-circuit caveat: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/case-transact-sql
- `COALESCE`: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/coalesce-transact-sql

**Oracle**
- CASE expressions: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CASE-Expressions.html
- NULLIF: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/NULLIF.html

**Snowflake / BigQuery**
- Snowflake `CASE`: https://docs.snowflake.com/en/sql-reference/functions/case
- Snowflake conditional expression functions (`GREATEST`, `GREATEST_IGNORE_NULLS`, `LEAST`): https://docs.snowflake.com/en/sql-reference/functions/greatest_ignore_nulls
- BigQuery conditional expressions: https://cloud.google.com/bigquery/docs/reference/standard-sql/conditional_expressions

**Essays and practical guides**
- Modern SQL, *CASE expression* (portability and semantics): https://modern-sql.com/feature/case
- Modern SQL, *FILTER clause* (the portable alternative to conditional aggregation): https://modern-sql.com/feature/filter
- Modern SQL, *Three-valued logic* (why `CASE` and `NULL` interact the way they do): https://modern-sql.com/concept/three-valued-logic
- Oracle-Base, *Analytic function syntax enhancements* (useful contrast for pivot alternatives): https://oracle-base.com/articles/21c/analytic-function-syntax-enhancements-21c

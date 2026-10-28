# Correlated Subqueries

> A correlated subquery references a column from the *outer* query, which makes it logically
> re-evaluated for every outer row. That is either perfectly fine — an indexed single-row
> lookup — or the single biggest performance bug in your warehouse, depending on whether an
> index exists and whether the optimizer can decorrelate it. This chapter explains the
> execution model, shows the concrete plan evidence you should look for, and gives the
> rewrite catalogue: correlated aggregate → join or window; correlated `MAX` → `DISTINCT ON`;
> correlated top-N → `LATERAL`; correlated `EXISTS` with `OR` → two semi joins.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- A subquery is **correlated** when it references a column produced by the enclosing query.
  Semantically it is a **function of the outer row**: `f(outer_row) → value | rowset | boolean`.
- Execution is a **nested loop by construction**: for each outer row, evaluate the subquery
  with the outer bindings. The optimizer *may* rewrite that into a join, a hash semi join, or
  a window — this is called **decorrelation** — but it is not guaranteed.
- The plan signatures to know (Postgres): `SubPlan` (per-row execution), `InitPlan` (once per
  query), `Materialize` feeding a `SubPlan` (cached across consecutive equal outer keys),
  `Hash Semi Join` / `Hash Anti Join` (successfully decorrelated).
- **`CROSS JOIN LATERAL` / `CROSS APPLY` is the explicit form of a correlated subquery** and is
  usually more optimizable: it is a real join node, so `LIMIT` can be pushed into it and the
  planner can choose the join algorithm.
- The classic rewrites:
  1. **Correlated `MAX`/`MIN` per group** → `DISTINCT ON`, a window function, or `LATERAL`.
  2. **Correlated aggregate per row** → pre-aggregate then join (one scan instead of N).
  3. **Correlated top-N** → `ROW_NUMBER() … = n` or `LATERAL … LIMIT n`.
  4. **Correlated `EXISTS` with `OR`** → two `EXISTS` (or two semi joins) — an `OR` inside a
     correlation usually kills the semi-join rewrite.
  5. **Correlated "previous row"** → `LAG`.
- Correlated subqueries are **fine** (and often clearer) when the outer set is small and the
  inner lookup is unique-indexed. They are **fatal** when they aggregate an unindexed table
  for each of a million rows.
- Every correlated subquery you write should be justified by one of: (a) an index, (b) a tiny
  outer set, or (c) a `LIMIT 1`.

---

## 1. Definitions and core concepts

### 1.1 Correlated vs uncorrelated

```sql
-- UNCORRELATED: no reference to c → one evaluation for the whole query
SELECT c.full_name,
       (SELECT COUNT(*) FROM orders) AS all_orders
FROM customers c;

-- CORRELATED: references c.customer_id → one evaluation *per customer row*
SELECT c.full_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS my_orders
FROM customers c;
```

### 1.2 Where correlated subqueries can appear

| Position | Returns | Typical use |
|---|---|---|
| `SELECT` list | scalar | attach a per-row aggregate |
| `WHERE` / `HAVING` | scalar or boolean | compare against a per-row value |
| `WHERE EXISTS` / `NOT EXISTS` | boolean | per-row existence test |
| `WHERE ... IN` | boolean | per-row membership |
| `FROM (…)` | relation | a derived table that depends on the outer row — this is **not** allowed unless written as `LATERAL` |
| `JOIN … ON` | boolean | per-row join predicate (rare, risky) |

The `FROM` case is why `LATERAL` exists: SQL has no other legal syntax for a derived table
that references a sibling in the same `FROM`.

### 1.3 Vocabulary

| Term | Meaning |
|---|---|
| **Correlation variable / outer reference** | The outer column the subquery depends on. |
| **SubPlan** | Postgres plan node for a correlated subquery: executed for each outer row (or group of rows). |
| **InitPlan** | Postgres plan node for an uncorrelated subquery: executed once. |
| **Decorrelation** | The optimizer's transformation of a correlated subquery into a join/group-by/aggregate. |
| **LATERAL / APPLY** | The explicit, join-based syntax for a per-row derived table. |
| **N+1 problem** | The cost pattern of one query plus one lookup per row; here it happens inside a single SQL statement. |

---

## 2. Why it matters / what problem it solves

Correlated subqueries are the most **expressive** form of per-row computation short of a
window function, and sometimes the only readable way to say what you mean:

- "This customer's most recent order" (per-row `MAX`).
- "Whether this session converted" (per-row `EXISTS`).
- "This row's total" (per-row aggregate over a child table).
- "The largest order of this product's category" (per-row correlated aggregate over a
  different grain).

They became popular in the pre-window-function era (before ~2012 in most engines), so they
are all over legacy code and BI tools' generated SQL. Knowing how they execute is what lets
you decide, per case, whether to keep one or rewrite it.

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 The naive model: a nested loop

```
FOR EACH outer_row r:
    subquery_result := evaluate(subquery, bindings = r)
    emit f(r, subquery_result)
```

That is `O(N)` subquery executions for `N` outer rows, each of which may itself be a scan or
an index lookup. With an index, each execution is `O(log M)`; without one, `O(M)` — giving
`O(N·M)` overall: the same complexity as a cross join.

### 3.2 Postgres plan signatures

```sql
EXPLAIN (ANALYZE, COSTS)
SELECT c.full_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id)
FROM customers c;
```

You will see one of:

```
-- UNCORRELATED: computed once
InitPlan 1 (returns $0)
  ->  Aggregate …

-- CORRELATED, executed per row
Seq Scan on customers c
  SubPlan 1
    ->  Aggregate
          ->  Index Only Scan using idx_orders_customer on orders o
                Index Cond: (customer_id = c.customer_id)
```

In `EXPLAIN (ANALYZE)`, the `SubPlan` node's `actual rows` and `loops` counts tell the story:
`loops=N` means the subplan ran `N` times.

A useful optimization Postgres sometimes applies:

```
SubPlan 1
  ->  Materialize              -- caches the subquery result for repeated outer keys
        ->  Index Scan on orders
```

`Materialize` helps when consecutive outer rows share the correlation key (a sorted join
input); it does nothing when the keys are scattered.

### 3.3 Decorrelation (the optimizer fighting back)

Modern optimizers rewrite correlated subqueries when the shape permits:

| Written | Rewritten to | When |
|---|---|---|
| `SELECT (SELECT SUM(x) FROM t WHERE t.k = o.k)` | `LEFT JOIN (SELECT k, SUM(x) FROM t GROUP BY k)` | always, in Postgres/SQL Server/Oracle for simple shapes |
| `WHERE EXISTS (SELECT 1 FROM t WHERE t.k = o.k)` | `Hash Semi Join` | equi-correlation |
| `WHERE NOT EXISTS (…)` with a `NOT NULL` inner column | `Hash Anti Join` | equi-correlation, nullability proven |
| `WHERE o.x > (SELECT AVG(x) FROM t WHERE t.k = o.k)` | join to a grouped subquery | many engines; not all |
| `SELECT (SELECT y FROM t WHERE t.k = o.k ORDER BY ts DESC LIMIT 1)` | `LATERAL`/window (`ROW_NUMBER`) | Postgres 14+ for some shapes; **not** generally rewritten |

`> Senior note`: decorrelation is why "correlated subqueries are always slow" is wrong. It is
also why you must read the plan rather than reason from the syntax: the same text can be a
`Hash Semi Join` on one engine/version and a per-row `SubPlan` on another.

### 3.4 `LATERAL`: the explicit, plan-friendly form

```sql
SELECT c.customer_id, o.order_id, o.ordered_at
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.order_id, o.ordered_at
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.ordered_at DESC
    LIMIT 1
) o;
```

Differences from a correlated scalar subquery:

| Aspect | Correlated scalar subquery | `LATERAL` |
|---|---|---|
| Cardinality | exactly one value (or error) | any number of rows |
| Plan shape | `SubPlan` | real join node |
| `LIMIT` inside | allowed but not indexed (§3.5) | fully pushed down; index scan with `LIMIT 1` |
| Extra columns | no | yes (you can return several) |
| Left-join variant | implicit (`NULL` when no row) | `LEFT JOIN LATERAL (...) ON TRUE` |
| Engines | all | Postgres, MySQL 8.0.14+, Oracle, Snowflake; SQL Server via `APPLY` |

### 3.5 Why the "correlated LIMIT 1" idiom still works

Even though a correlated scalar subquery with `ORDER BY … LIMIT 1` is *not* a join, engines
that implement it as a `SubPlan` can still use an index scan with a limit, because the
subplan's plan is built with the correlation parameter available. In Postgres, an index on
`orders(customer_id, ordered_at DESC)` makes it a two-row `Index Scan` per customer. Without
that index it is a full sort of the customer's orders — or worse, a full scan.

**The index is the whole story.** Compare:

```sql
-- WITH idx_orders_customer (customer_id, ordered_at DESC): fast, O(log n) per customer
SELECT c.customer_id,
       (SELECT o.order_id FROM orders o
        WHERE o.customer_id = c.customer_id
        ORDER BY o.ordered_at DESC LIMIT 1) AS last_order_id
FROM customers c;
```

---

## 4. Syntax and mechanics (with examples)

### 4.1 Correlated scalar in `SELECT`

```sql
SELECT c.full_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id)  AS n_orders,
       (SELECT MAX(o.ordered_at) FROM orders o
        WHERE o.customer_id = c.customer_id AND o.status = 'delivered')     AS last_delivered_at
FROM customers c
ORDER BY n_orders DESC, c.full_name;
```

### 4.2 Correlated `EXISTS` / `NOT EXISTS`

```sql
SELECT c.full_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'returned');
```

### 4.3 Correlated comparison in `WHERE`

```sql
-- Products priced above the average price of their own category
SELECT p.name, p.category_id, p.unit_price
FROM products p
WHERE p.unit_price > (SELECT AVG(p2.unit_price) FROM products p2
                      WHERE p2.category_id = p.category_id)
ORDER BY p.category_id, p.unit_price DESC;
```

### 4.4 Correlated subquery in `HAVING`

```sql
SELECT o.customer_id, SUM(i.quantity * i.unit_price) AS lifetime_value
FROM orders o JOIN order_items i USING (order_id)
GROUP BY o.customer_id
HAVING SUM(i.quantity * i.unit_price) > (SELECT AVG(total) FROM (
           SELECT SUM(i2.quantity * i2.unit_price) AS total
           FROM orders o2 JOIN order_items i2 ON i2.order_id = o2.order_id
           GROUP BY o2.customer_id) x);
```

Note: this `HAVING` subquery is *uncorrelated* — it does not reference the grouped row. A
correlated `HAVING` would look like
`HAVING SUM(...) > (SELECT AVG(i3.unit_price) FROM order_items i3 WHERE i3.order_id = o.order_id)`,
which is legal but odd (it applies to one arbitrary `o.order_id` per group unless you also
group by `o.order_id`) — a classic source of undefined results. Avoid.

### 4.5 `LATERAL` / `APPLY` forms

```sql
-- PostgreSQL / MySQL 8.0.14+ / Oracle / Snowflake
SELECT c.full_name, x.order_id, x.total
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.order_id,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS total
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.customer_id = c.customer_id
    GROUP BY o.order_id
    ORDER BY total DESC
    LIMIT 1
) x ON TRUE
ORDER BY c.full_name;

-- SQL Server equivalent
-- FROM customers c
-- OUTER APPLY (SELECT TOP 1 … ORDER BY …) x
```

### 4.6 The `IN` variant

```sql
-- Correlated membership test
SELECT c.full_name FROM customers c
WHERE c.customer_id IN (SELECT o.customer_id FROM orders o
                        WHERE o.status = 'delivered' AND o.country_code = c.country_code);
```

Careful: the correlation here is on `country_code`, not on `customer_id`. This is legal but
easy to misread — it reads as "customers who have a delivered order in *their own* country".
Prefer `EXISTS` for correlated membership; it is unambiguous about intent.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: per-customer order count and last order

```sql
SELECT c.customer_id, c.full_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id)       AS n_orders,
       (SELECT MAX(o.ordered_at) FROM orders o WHERE o.customer_id = c.customer_id) AS last_order_at
FROM customers c
ORDER BY n_orders DESC, c.full_name;
```

Expected: `Alice Johnson → 3 orders, 2024-03-30`; then the four customers with two orders
(`Bob Smith → 2, 2024-03-02`, `Carol White → 2, 2024-03-18`, `Dave Brown → 2, 2024-03-20`,
`Erin Davis → 2, 2024-03-25`, `Frank Miller → 2, 2024-02-20` — alphabetical after the tie);
`Grace Lee → 1, 2024-03-28`; `Heidi Novak → 0, NULL`.

Already visible here: `Heidi` gets `0` and `NULL` — uncorrelated-aggregate semantics, no
`LEFT JOIN` needed. This is the main *readability* advantage of the correlated form.

### Example 2 — Basic: the same answer three ways (and their plans)

```sql
-- (a) correlated scalar subqueries
SELECT c.customer_id,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS n_orders
FROM customers c;

-- (b) pre-aggregated join
WITH cnt AS (SELECT customer_id, COUNT(*) AS n_orders FROM orders GROUP BY customer_id)
SELECT c.customer_id, COALESCE(cnt.n_orders, 0) AS n_orders
FROM customers c LEFT JOIN cnt USING (customer_id);

-- (c) window function over a left join
SELECT c.customer_id,
       COUNT(o.order_id) OVER (PARTITION BY c.customer_id) AS n_orders
FROM customers c LEFT JOIN orders o ON o.customer_id = c.customer_id;
```

All three return the same 8 rows. Form (c) is subtly different if a customer could have the
same order twice (it cannot, `order_id` is a PK) and it *duplicates* rows with multiple orders
— which is often not what you want for a per-customer report. Forms (a) and (b) are equivalent
in grain; (a) is shorter, (b) is one scan.

### Example 3 — Intermediate: "latest order per customer" — correlated `MAX` and its rewrites

```sql
-- (a) Correlated scalar: simple, index-friendly
SELECT c.customer_id, c.full_name,
       (SELECT o.order_id FROM orders o
        WHERE o.customer_id = c.customer_id
        ORDER BY o.ordered_at DESC LIMIT 1) AS last_order_id
FROM customers c
ORDER BY c.customer_id;

-- (b) DISTINCT ON (PostgreSQL): one sort of the whole orders table
SELECT DISTINCT ON (o.customer_id) o.customer_id, o.order_id, o.ordered_at
FROM orders o
ORDER BY o.customer_id, o.ordered_at DESC;

-- (c) Window function: portable, one pass
SELECT customer_id, order_id, ordered_at FROM (
    SELECT o.customer_id, o.order_id, o.ordered_at,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.ordered_at DESC) AS rn
    FROM orders o) t
WHERE rn = 1;

-- (d) LATERAL: index-driven, one lookup per customer
SELECT c.customer_id, x.order_id, x.ordered_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.order_id, o.ordered_at FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.ordered_at DESC LIMIT 1) x ON TRUE
ORDER BY c.customer_id;
```

Expected: `1 → 1014`, `2 → 1004`, `3 → 1006`, `4 → 1008`, `5 → 1010`, `6 → 1012`, `7 → 1013`,
`8 → NULL`. Four different plans for the same answer:

| Form | Best when | Worst when |
|---|---|---|
| (a)/(d) nested-loop lookup | small/recently-filtered outer set, index on `(fk, ts desc)` | large outer set, no index |
| (b) `DISTINCT ON` | scan the child table once; Postgres only | huge child table with tiny outer filter |
| (c) window | needs multiple columns from the row; portable | large child table |

That is the decision table you should carry in your head.

### Example 4 — Intermediate: correlated aggregate vs join (fan-out-safe totals)

```sql
-- (a) Correlated: per order, sum its lines. 1 SubPlan execution per order.
SELECT o.order_id, o.status,
       (SELECT SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100))
        FROM order_items i WHERE i.order_id = o.order_id) AS order_total
FROM orders o
ORDER BY o.order_id;

-- (b) Join + aggregate: one scan of order_items
SELECT o.order_id, o.status, COALESCE(SUM(x.amount), 0) AS order_total
FROM orders o
LEFT JOIN (
    SELECT i.order_id, SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS amount
    FROM order_items i GROUP BY i.order_id) x USING (order_id)
GROUP BY o.order_id, o.status
ORDER BY o.order_id;
```

Expected (from the seed): `1001 → 1034.98`, `1002 → 38.97`, `1003 → 899.00`, `1004 → 19.99`,
`1005 → 134.00`, `1006 → 39.99`, `1007 → 2028.99`, `1008 → 79.98`, `1009 → 799.00`,
`1010 → 12.99`, `1011 → 237.00`, `1012 → 1499.00`, `1013 → 32.98`, `1014 → 999.00`.
Form (a) is clearer; form (b) is `O(orders + items)` instead of `O(orders × log items)` — at
10M orders the difference is the difference between a minute and an hour.

### Example 5 — Intermediate: correlated `EXISTS` with `OR` (why it plans badly)

```sql
-- BAD: the OR prevents the semi-join rewrite in most engines
SELECT c.full_name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE (o.customer_id = c.customer_id AND o.status = 'returned')
       OR (o.customer_id = c.customer_id AND o.status = 'cancelled'));

-- GOOD: two independent semi joins, each hashable
SELECT c.full_name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'returned')
   OR EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'cancelled');
```

Expected: `Bob Smith` (returned), `Carol White` (cancelled). Read the plans: the first typically
degrades to a per-row `SubPlan` with a filter; the second becomes two `Hash Semi Join`s
combined by an `Append`/`BitmapOr`-style node.

### Example 6 — Advanced: correlated aggregate for a *different* grain (category average)

```sql
-- Products priced above their category average — correlated aggregate over the same table
SELECT p.name, cat.name AS category, p.unit_price,
       (SELECT ROUND(AVG(p2.unit_price), 2) FROM products p2
        WHERE p2.category_id = p.category_id) AS category_avg_price
FROM products p
JOIN categories cat ON cat.category_id = p.category_id
WHERE p.unit_price > (SELECT AVG(p2.unit_price) FROM products p2
                      WHERE p2.category_id = p.category_id)
ORDER BY cat.name, p.unit_price DESC;

-- Rewrite: window function (single pass, no per-row subquery)
SELECT name, category, unit_price, category_avg_price FROM (
    SELECT p.name, cat.name AS category, p.unit_price,
           ROUND(AVG(p.unit_price) OVER (PARTITION BY p.category_id), 2) AS category_avg_price
    FROM products p JOIN categories cat ON cat.category_id = p.category_id) t
WHERE unit_price > category_avg_price
ORDER BY category, unit_price DESC;
```

Expected (5 rows, one per category that has products): `Phones → iPhone 15 (999.00, avg
899.00)`, `Laptops → MacBook Air M3 (1999.00, avg 1749.00)`, `Cases → Laptop Sleeve (29.99,
avg 24.99)`, `Chargers → 65W Charger (39.99, avg 26.49)`, `Kitchen → Coffee Maker (89.00, avg
64.33)`. Each category contributes exactly one row, which is a useful sanity signal: an
average is usually beaten by the top item only.

The window version is provably one pass. The correlated version is one pass *per product* —
fine for 12 rows, catastrophic for 12 million.

### Example 7 — Advanced: top-N per group — correlated `LIMIT`, `LATERAL`, and window

```sql
-- (a) LATERAL: index-driven, stops at N rows per category
SELECT cat.name AS category, p.name AS product, p.unit_price
FROM categories cat
LEFT JOIN LATERAL (
    SELECT p.name, p.unit_price FROM products p
    WHERE p.category_id = cat.category_id AND p.is_active
    ORDER BY p.unit_price DESC LIMIT 2) p ON TRUE
ORDER BY cat.name, p.unit_price DESC NULLS LAST;

-- (b) Window: one pass, but must sort the whole product set
SELECT category, product, unit_price FROM (
    SELECT cat.name AS category, p.name AS product, p.unit_price,
           ROW_NUMBER() OVER (PARTITION BY p.category_id ORDER BY p.unit_price DESC) AS rn
    FROM products p JOIN categories cat ON cat.category_id = p.category_id
    WHERE p.is_active) t
WHERE rn <= 2
ORDER BY category, unit_price DESC;
```

Both return the same top-2 sets. (a) is `O(categories × log products)` with an index on
`products(category_id, unit_price DESC)`; (b) is `O(products log products)` regardless of how
many categories you need. With 10 categories, (a) wins; with 10 million categories, (b) wins.

### Example 8 — Advanced: rewriting a correlated "previous row" as `LAG`

```sql
-- (a) Correlated: for each event, find the latest earlier event in the same session
SELECT e.event_id, e.session_id, e.occurred_at,
       (SELECT MAX(e2.occurred_at) FROM events e2
        WHERE e2.session_id = e.session_id AND e2.occurred_at < e.occurred_at) AS prev_at
FROM events e
ORDER BY e.session_id, e.occurred_at;

-- (b) Window: one sort, no subquery
SELECT e.event_id, e.session_id, e.occurred_at,
       LAG(e.occurred_at) OVER (PARTITION BY e.session_id ORDER BY e.occurred_at) AS prev_at
FROM events e
ORDER BY e.session_id, e.occurred_at;
```

Expected: identical results. Form (a) is `O(n log n)` *per row* (or a scan per row) unless the
engine decorrelates it; form (b) is `O(n log n)` total. This is the most common correlated
subquery in legacy analytics code, and the easiest to convert.

### Example 9 — Advanced: the plan evidence (what to look for)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.customer_id,
       (SELECT MAX(o.ordered_at) FROM orders o WHERE o.customer_id = c.customer_id) AS last_at
FROM customers c;
```

Interpretation guide:

| Plan fragment | Meaning | Action |
|---|---|---|
| `InitPlan` | computed once | nothing to do |
| `SubPlan` with `loops = N` | executed per outer row | check for an index; consider a rewrite |
| `Materialize` above the subplan's scan | results cached for repeated keys | good; nothing to do |
| `Index Only Scan` / `Index Scan` inside the `SubPlan` | indexed lookup per row | good; ensure the index matches the correlation + order |
| `Seq Scan` inside the `SubPlan` | full scan per row | **fix now**: add an index or rewrite as a join |
| `Hash Semi Join` / `Hash Anti Join` at the top | decorrelated successfully | nothing to do |

`> Senior note`: in `EXPLAIN ANALYZE`, the top-level node's `actual time` is the *total* wall
time; use the per-node `actual time` and `loops` columns to attribute cost. A `SubPlan` with
`loops = 100000` and `actual time = 0.02 ms` is 2 seconds of your query.

---

## 6. Real-world use cases and applications

| Use case | Shape | Preferred form today |
|---|---|---|
| **Customer 360: last order, lifetime value** | per-row aggregate | `LATERAL` (index) or pre-aggregated join |
| **Session flags for funnels** | per-row `EXISTS` | `EXISTS` semi join (decorrelates well) |
| **Ranking within group** | per-row `COUNT(*)` of "better" rows | window function |
| **Previous/next event** | per-row `MAX(<)` | `LAG`/`LEAD` |
| **Reference-data lookups by date** | per-row `SELECT … WHERE valid_from <= t ORDER BY valid_from DESC LIMIT 1` | `LATERAL`, or the SCD2 non-equi join |
| **Deduplication of CDC rows** | per-row `MAX(updated_at)` | `ROW_NUMBER()` |
| **"Top N per group" dashboards** | per-row top-N | `LATERAL`/`ROW_NUMBER` |
| **Data-quality checks** | per-row existence against a reference set | `NOT EXISTS` anti join |
| **BI-tool generated SQL** | many correlated subqueries in the `SELECT` list | often the main reason a dashboard is slow — rewrite |

---

## 7. Performance notes and anti-patterns

### 7.1 The three conditions under which a correlated subquery is acceptable

1. **The outer set is small** (hundreds/thousands of rows) — the N factor is bounded.
2. **The inner lookup is unique-indexed** (or index-supported for the `ORDER BY … LIMIT 1`).
3. **The inner result set per row is tiny** (one row or one scalar).

If any of these fails, rewrite.

### 7.2 The rewrite catalogue

| Correlated pattern | Rewrite | Why it is better |
|---|---|---|
| `(SELECT SUM(x) FROM child c WHERE c.fk = p.id)` | `LEFT JOIN (SELECT fk, SUM(x) GROUP BY fk)` | one scan instead of N |
| `(SELECT MAX(ts) FROM child WHERE fk = p.id)` | `DISTINCT ON (fk) … ORDER BY fk, ts DESC` or `ROW_NUMBER() = 1` | one sort, one pass |
| `(SELECT y FROM child WHERE fk = p.id ORDER BY ts DESC LIMIT 1)` | `LATERAL` (index-driven) or `ROW_NUMBER` | plan-friendly join node |
| `WHERE EXISTS (… OR …)` | two `EXISTS` / two semi joins | enables hash semi joins |
| `WHERE x > (SELECT AVG(x) FROM t WHERE t.k = o.k)` | window function `AVG(x) OVER (PARTITION BY k)` | single pass |
| `(SELECT … )` for the previous row | `LAG` | single sort |
| `IN (SELECT … WHERE inner.k = outer.k)` | `EXISTS` with an explicit correlation | unambiguous, decorrelatable |
| `SELECT (SELECT COUNT(*) …)` | pre-aggregated `LEFT JOIN` + `COALESCE` | same result, one pass |

### 7.3 Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| Correlated aggregate over an unindexed column | `SubPlan` with `Seq Scan`, quadratic runtime | index or rewrite |
| Several correlated subqueries over the *same* table in one `SELECT` | N × k executions | one grouped join re-used k times |
| Correlated subquery in a `SELECT` that is then filtered by `WHERE` | the subquery runs for rows you discard | filter in the CTE *before* the subquery, or push into a join |
| Correlated `EXISTS` with a non-sargable inner predicate | per-row scan | expression index or normalized column |
| Correlated subquery depending on a column of a table two levels up | hard to reason about, harder to decorrelate | flatten with CTEs |
| Assuming decorrelation "just happens" | plan changes across versions | pin behavior with tests / verify the plan |
| Using a correlated subquery where `LATERAL` was meant | returns one value when you needed many columns | use `LATERAL` |
| Correlated subquery for existence, then `DISTINCT` outside | double work | `EXISTS` alone |
| `LATERAL` with no index on the correlated column | one full scan per outer row — the worst of both worlds | add the index the lateral subquery needs |

### 7.4 The index that makes or breaks it

```sql
-- The correlation + ordering pattern must be an index, in that order
CREATE INDEX idx_orders_customer_recent ON orders (customer_id, ordered_at DESC);

-- For a filtered lookup, put the equality first and the filter second
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status, ordered_at DESC);

-- For an EXISTS correlated on a nullable column, a plain index suffices
CREATE INDEX idx_events_customer ON events (customer_id);
```

Rule: **the index must lead with the correlation column**, because the subquery's predicate is
known only at execution time (it is a parameter), and a B-tree can only use leading columns
with equality.

### 7.5 Verification recipe

```sql
-- (1) Baseline plan and timing
EXPLAIN (ANALYZE, BUFFERS) <correlated version>;

-- (2) Rewritten plan and timing
EXPLAIN (ANALYZE, BUFFERS) <join/window/LATERAL version>;

-- (3) Result equivalence (must be empty)
WITH a AS (SELECT … ), b AS (SELECT … )
SELECT * FROM ((SELECT * FROM a EXCEPT SELECT * FROM b)
               UNION ALL
               (SELECT * FROM b EXCEPT SELECT * FROM a));
```

Do all three, or you will "optimize" a query into different numbers.

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Correlated subquery in `SELECT` | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ limited/decorrelated |
| Correlated `EXISTS`/`NOT EXISTS` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Explicit lateral derived table | `LATERAL` | `LATERAL` (8.0.14+) | `CROSS/OUTER APPLY` | `LATERAL` / `OUTER APPLY` (+ `CROSS APPLY` from 12c) | `LATERAL` | `LEFT JOIN UNNEST(...)` / correlated `ARRAY` |
| Plan node for per-row execution | `SubPlan` (+ `Materialize`) | `DEPENDENT SUBQUERY` (in `EXPLAIN`); optimizer may use semi-join (8.0.17+) | `Nested Loops` + `Filter` with a subquery | `FILTER` / `TABLE ACCESS … FILTER` | shown as part of the join graph | decorrelated before execution |
| Auto-decorrelation of scalar aggregates | ✅ (many shapes) | ✅ (some) | ✅ | ✅ | ✅ | ✅ (must be expressible as a join) |
| Auto semi-join rewrite | ✅ | ✅ 8.0.17+ (not in 5.7) | ✅ | ✅ | ✅ | ✅ |
| `DISTINCT ON` (latest-per-key) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `IGNORE NULLS` for last-known-value | ❌ (Postgres 19 adds it) | ❌ | ❌ | ✅ | ✅ | ✅ |
| Statement-level hints to force a plan | ✅ (planner `enable_*` GUCs) | ✅ (optimizer hints 8.0.20+) | ✅ (query hints, `FORCE ORDER`, `HASH JOIN`) | ✅ (`/*+ … */`) | ❌ | ❌ |
| Best practice for top-N per group | `LATERAL` or window | `LATERAL` (8.0.14+) or window | `OUTER APPLY` or window | `LATERAL`/window + `KEEP (DENSE_RANK FIRST)` | `QUALIFY` + `ROW_NUMBER` | `QUALIFY` or window |

Engine notes:

- **MySQL**: `EXPLAIN` labels correlated subqueries `DEPENDENT SUBQUERY` — if you see that, you
  are paying per-row. MySQL 5.7 lacks semi-join optimization for many shapes, so `EXISTS` is
  often dramatically faster than `IN (subquery)` there.
- **SQL Server** prefers `APPLY` for correlated derived tables; the optimizer is also good at
  decorrelating scalar subqueries into `LEFT JOIN` + `Stream Aggregate`.
- **Oracle** has a very mature decorrelation engine and additionally offers
  `KEEP (DENSE_RANK FIRST ORDER BY …)` as an aggregate-based "latest row" idiom, plus
  `MATCH_RECOGNIZE` for pattern queries.
- **Snowflake** exposes `QUALIFY`, which removes a large class of correlated rewrites.
- **BigQuery** is the strictest: correlated subqueries in the `SELECT` list are not supported
  the way they are elsewhere; use `LEFT JOIN`/`ARRAY` subqueries. Correlated `EXISTS` works.
- **PostgreSQL** gives you `DISTINCT ON` (the cleanest "one row per key" idiom anywhere) and
  explicit `MATERIALIZED`/`NOT MATERIALIZED` control over CTE fences, which interact with
  decorrelation decisions.

---

## 9. Exercises — Intermediate (hints only)

1. **For each customer, show the number of orders and the date of the most recent one.**
   *Hint:* two correlated scalar subqueries; then rewrite with a single pre-aggregated join.
2. **Products above their category average price.** *Hint:* correlated `AVG`, then the window
   rewrite; verify equality with a symmetric `EXCEPT`.
3. **For each order, the number of distinct products.** *Hint:* correlated `COUNT(DISTINCT …)`;
   note that `COUNT(DISTINCT)` in a correlated subquery is one of the harder shapes to
   decorrelate.
4. **The most recent event per session.** *Hint:* `LATERAL` + `ORDER BY … LIMIT 1`, or a
   window function.
5. **Customers who have more orders than the average customer.** *Hint:* a correlated count
   compared against an uncorrelated average over a grouped subquery.
6. **The cheapest product in each category, with its price.** *Hint:* correlated `MIN` in a
   `WHERE` (`p.unit_price = (SELECT MIN(…) WHERE …)`), then the `DISTINCT ON`/window rewrite.
7. **For every employee, the number of direct reports.** *Hint:* correlated `COUNT(*)` over
   `employees` on `manager_id`; watch the CEO row (0 reports, not `NULL`).
8. **Identify products that appear in more than one order for the same customer.** *Hint:*
   correlated `EXISTS` with a self-referencing condition on `order_items`/`orders`.
9. **Rewrite a correlated "previous event" query as `LAG` and prove equivalence.**
   *Hint:* compare row counts and a symmetric difference.
10. **Measure the effect of adding `orders(customer_id, ordered_at DESC)`.** *Hint:*
    `EXPLAIN (ANALYZE, BUFFERS)` before/after; report the change in `SubPlan` node time.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Take a query with five correlated scalar subqueries over `order_items` and make it one
   pass.** Requirements: identical results, one scan of `order_items`, and a documented
   before/after plan. *Approach:* aggregate once into a CTE keyed by `order_id` (or
   `customer_id`), `LEFT JOIN` it, and `COALESCE` the nulls; then compare `EXPLAIN (ANALYZE)`
   totals and prove equivalence with a symmetric `EXCEPT`.

2. **Design the "as-of" lookup (SCD2) three ways** — correlated scalar with `LIMIT 1`,
   `LATERAL`, and a non-equi join on `[valid_from, valid_to)` — and decide which to ship.
   *Approach:* correctness (half-open intervals, no double matches), plan shape, and what
   happens when `valid_to IS NULL`; add a uniqueness guard query.

3. **Prove decorrelation with plan evidence.** Write a scalar correlated `SUM` that Postgres
   decorrelates into a `LEFT JOIN` + `GroupAggregate`, and one that it does *not* (e.g. with a
   `LIMIT 1` inside a `CASE`), showing `SubPlan` in the second. *Approach:* `EXPLAIN` both and
   point at the specific nodes; explain the transformation limits.

4. **Optimize a "latest row per key" query on a 200M-row table with 5 key rows per key on
   average.** Compare `DISTINCT ON`, `ROW_NUMBER`, `LATERAL`, and a correlated `MAX`, with and
   without a matching index. *Approach:* measure I/O (`BUFFERS`) as well as time; derive the
   rule of thumb for when each wins.

5. **Correlated subquery inside a recursive CTE.** Compute, for every node in the category
   tree, the number of products in its subtree — with a correlated aggregate evaluated inside
   the recursion. Then optimize it with the closure-table approach from
   [Chapter 03](./03-recursive-ctes.md). *Approach:* measure the multiplicative cost
   (`iterations × correlated executions`) and explain why the closure table decouples them.

6. **Convert a correlated `EXISTS` with a compound `OR` into a set of hash semi joins** and
   show the plan change from `Filter + SubPlan` to `Hash Semi Join` nodes.
   *Approach:* one `EXISTS` per disjunct, `UNION`-style deduplication via `OR` of semi joins;
   verify the row count is unchanged.

7. **Make a BI tool's generated SQL fast without editing it.** Given N correlated subqueries in
   the `SELECT` list, propose the minimal set of indexes (and possibly a materialized view)
   that makes the query acceptable, and justify each with a plan. *Approach:* attribute time
   per `SubPlan`, decide index vs. rewrite, and explain the maintenance cost of the
   materialized alternative.

8. **Write a lint rule that flags risky correlated subqueries.** *Approach:* detect
   `SELECT`-list subqueries that reference an outer alias and aggregate a large table; combine
   with catalog statistics (`pg_class.reltuples`, index existence) to estimate `N × M` cost and
   warn above a threshold. Explain why a purely syntactic rule is not enough.

---

## References

**PostgreSQL**
- Subquery expressions: https://www.postgresql.org/docs/current/functions-subquery.html
- `LATERAL` in `FROM`: https://www.postgresql.org/docs/current/queries-table-expressions.html
- Value expressions (scalar subqueries, `DISTINCT ON` alternative forms): https://www.postgresql.org/docs/current/sql-expressions.html
- Planner method configuration and cost constants (why the estimates move): https://www.postgresql.org/docs/current/runtime-config-query.html

**MySQL**
- Subqueries and the semi-join/derived-table optimizations: https://dev.mysql.com/doc/refman/8.0/en/any-in-some-subqueries.html
- Lateral derived tables (8.0.14+): https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html

**SQL Server**
- `FROM` with `APPLY` (`CROSS APPLY` / `OUTER APPLY`): https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql
- `EXISTS`: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/exists-transact-sql

**Oracle**
- `SELECT` (`LATERAL`, `CROSS APPLY`, subquery factoring): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake `QUALIFY` (removes a whole class of correlated rewrites): https://docs.snowflake.com/en/sql-reference/constructs/qualify
- BigQuery query syntax (correlated subqueries, `UNNEST`): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Markus Winand, *Nested loops and the N+1 selects problem*: https://use-the-index-luke.com/sql/join/nested-loops-join-n1-problem
- Markus Winand, *Fetching the next page* (keyset vs offset, a related per-row lookup problem): https://use-the-index-luke.com/sql/partial-results/fetch-next-page

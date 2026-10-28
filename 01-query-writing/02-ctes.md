# Common Table Expressions (CTEs)

> CTEs are the single biggest readability upgrade in modern SQL — they let you write a query
> top-to-bottom the way you think about it: stage 1, stage 2, final select. But `WITH` is not
> free sugar: in some engines it is an *optimization fence* that stops predicate pushdown, and
> in others it is expanded inline and may execute twice. This chapter covers the syntax, the
> materialization semantics (which differ per engine and per version), when a CTE beats a
> subquery, when it beats a temp table, and how to read an execution plan that tells you
> which one you actually got.

All examples run on the shared Acme Commerce schema defined in
[01-complex-joins.md](./01-complex-joins.md) (or `00-sample-schema.sql`).

---

## TL;DR

- A CTE is a **named, scoped, non-persistent result set** defined with `WITH`. Scope = the
  single statement that follows it (plus further CTEs in the same `WITH` list, left to
  right).
- **Readability is the primary reason to use one.** A 300-line nested subquery becomes a
  5-stage named pipeline.
- **Materialization is the second question, and it is engine-specific.**
  - PostgreSQL ≥12: non-recursive, side-effect-free CTE referenced **once** is **inlined**
    by default; referenced **more than once** it is **materialized**. You can force either
    with `MATERIALIZED` / `NOT MATERIALIZED`.
  - SQL Server: CTEs are **always inlined** (no `MATERIALIZED` hint). A CTE referenced twice
    may be **executed twice** — the classic "my CTE runs 2×" surprise.
  - MySQL 8: a CTE is either merged or materialized into a temporary table; there is **no
    hint**. Check `EXPLAIN FORMAT=JSON`.
  - BigQuery: CTEs are effectively **inline subqueries**; a CTE used twice duplicates the
    work (billable bytes!). Snowflake behaves similarly (inlined where possible).
  - Oracle: supports the `/*+ MATERIALIZE */` and `/*+ INLINE */` hints for factored
    subqueries.
- A **materialized CTE is an optimization fence**: filters in the outer query do not push
  into it. This is sometimes exactly what you want (execute the expensive scan once) and
  sometimes a 100× regression (no predicate pushdown, no index use).
- CTEs can be **data-modifying** in PostgreSQL (`WITH x AS (INSERT ... RETURNING *)`) — the
  only way to do multi-statement DML atomically inside one statement. SQL Server, MySQL,
  and others do not support this.
- Rule of thumb: **CTE for readability, temp table for reuse at scale, subquery for a
  one-off expression, view for reuse across queries.**

---

## 1. Definitions and core concepts

### 1.1 What a CTE is

```sql
WITH cte_name (column_list) AS (   -- column_list optional
    <select statement>
)
SELECT ... FROM cte_name ...
```

| Property | Value |
|----------|-------|
| Scope | The statement immediately following; visible to later CTEs in the same `WITH` |
| Lifetime | The statement (cursors can keep a materialized result alive) |
| Naming | Must be unique within the `WITH` list; can shadow a table name (dangerous) |
| Dependency order | A CTE can reference CTEs defined **before** it, never after |
| Recursion | Separate rules — see [03-recursive-ctes.md](./03-recursive-ctes.md) |
| Storage | None by itself; it is a *name*, not a table |

### 1.2 The three execution modes (which is which matters)

1. **Inline / merge (subquery folding).** The CTE body is textually substituted into the
   consuming expression, then the planner optimizes the merged query. Predicate pushdown
   works, indexes can be used, join reordering is free.
2. **Materialized.** The engine executes the CTE body once, stores the rows (a temp
   structure / tuplestore / spool), and every reference reads that. **Cost:** the outer
   predicates cannot push in, and there is no index on the materialized set unless the
   engine builds one (SQL Server can auto-create an index on a temp table; a CTE spool
   cannot be indexed).
3. **Lazy/inline but re-evaluated per reference.** The worst of both worlds: the query is
   folded in but appears at every reference site, so it is *computed multiple times*. This
   is the SQL Server and BigQuery behavior you must actively watch for.

### 1.3 Vocabulary

| Term | Meaning |
|------|---------|
| **Optimization fence** | A construct the planner cannot see through, e.g. a materialized CTE, so predicates/limits cannot be pushed across it. |
| **Predicate pushdown** | Moving a filter from the outer query into the inner query to reduce rows early. |
| **CTE scan** | The plan node that reads a materialized CTE (`CTE Scan` in Postgres, `Table Spool`/`Worktable` in SQL Server). |
| **Lateral reuse** | In MySQL 8.0.x, several CTEs can share one materialization when the optimizer detects identical subqueries. |

---

## 2. Why it matters / what problem it solves

- **Cognitive load.** Analytical SQL is a DAG of transformations. `WITH` lets you name each
  node. Reviewers can verify stage by stage; debugging is "comment out the last CTE".
- **DRY.** A filter used by three downstream aggregates is written once. No copy-paste drift
  (the classic bug where two copies of "active customer" disagree).
- **Correctness of join grain.** Naming an aggregate ("`order_revenue` is one row per
  order") makes fan-out bugs obvious.
- **Engine-independent composition.** The same structure works in Postgres, Snowflake,
  BigQuery, Spark SQL — unlike temp tables, whose `CREATE TEMPORARY TABLE` syntax and
  session semantics vary.
- **Data-modifying CTEs (Postgres)** allow "insert the order, then insert its items,
  atomically, in one statement" — invaluable for idempotent pipelines.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 The pipeline

```
SQL text
  └─ parse: WITH list is a set of named subqueries (a DAG)
      └─ rewrite: inline eligible CTEs, mark others as "materialize"
          └─ plan: choose access paths / join algorithms per remaining subtree
              └─ execute: CTE Scan pulls from a materialized tuplestore, or the
                          inlined subtree is executed where it appears
```

### 3.2 PostgreSQL: the exact rule (this is worth memorizing)

From the Postgres documentation (*7.8.3. Common Table Expression Materialization*):

> A useful property of WITH queries is that they are **normally evaluated only once** per
> execution of the parent query, even if they are referred to more than once… However, the
> other side of this coin is that the optimizer is not able to push restrictions from the
> parent query down into a multiply-referenced WITH query…
>
> …if a WITH query is **non-recursive** and is **side-effect-free** (that is, it contains no
> volatile functions) … it **can be folded** into the parent query if it is used **only
> once**. By default this happens if the parent query references the WITH query just once,
> but not more.

Mechanically:

| CTE referenced | Contains volatile function? | Default behavior | Override |
|---|---|---|---|
| 1× | no | **inlined** | `AS MATERIALIZED (...)` |
| 2+× | — | **materialized** | `AS NOT MATERIALIZED (...)` |
| any | yes | materialized | `AS NOT MATERIALIZED` (risky) |
| recursive | — | always iterative, never inlined | — |

Consequences you will actually observe:

- Adding a *second* reference to a CTE changes the plan and can make the query **slower**
  (fence) or **faster** (compute once).
- `LIMIT` in the outer query does not reduce the work of a materialized CTE — the whole
  CTE is computed.

```sql
-- Force materialization (compute the expensive thing once)
WITH heavy AS MATERIALIZED (
    SELECT ... FROM events WHERE event_type = 'page_view'
)
SELECT COUNT(*) FROM heavy;

-- Force inlining (let the outer filter push in)
WITH base AS NOT MATERIALIZED (
    SELECT * FROM orders
)
SELECT COUNT(*) FROM base WHERE status = 'delivered';   -- index on (status, ordered_at) can be used
```

### 3.3 SQL Server: inlined, possibly twice

SQL Server inlines CTEs into the query tree. There is no `MATERIALIZED` keyword. If the
same CTE is referenced twice, the subtree may be evaluated twice (unless the optimizer
introduces a spool to protect from the Halloween problem, or an `Eager Spool` is added).
Practical impact: a CTE containing `GETDATE()`, `NEWID()`, or `ORDER BY ... OFFSET` may
return **inconsistent values between references** — a correctness bug, not just a
performance one. Fix: materialize into `#temp` when you need stability at scale.

### 3.4 MySQL 8: materialized or merged

MySQL treats CTEs as derived tables; the optimizer decides per CTE whether to merge into
the outer query (`derived_merge`) or materialize into an internal temporary table. There is
no hint, so you verify with:

```sql
EXPLAIN FORMAT=JSON
SELECT ...;   -- look for "materialized": { ... } vs a merged derived table
```

### 3.5 BigQuery and Snowflake: inline, billed per reference

BigQuery compiles CTEs into inline subqueries. Referencing a CTE 5 times potentially scans
the underlying tables 5 times and **you pay for all of it**. Snowflake inlines CTEs too
where it can (Snowflake does keep the CTE definition in the DAG and reuses results when the
subquery is identical, but you cannot rely on it). The idiom in both is: if a CTE is
expensive and multiply referenced, write it to a temp table (Snowflake), or restructure so
each referenced CTE is cheap.

### 3.6 Data-modifying CTEs (PostgreSQL only)

```sql
WITH ins_order AS (
    INSERT INTO orders (customer_id, status, ordered_at, country_code, channel)
    VALUES (1, 'pending', now(), 'US', 'web')
    RETURNING order_id
)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT order_id, 6, 1, 19.99 FROM ins_order;
```

All statements in the `WITH` execute **in the same snapshot** and are not visible to each
other except through `RETURNING`. They commit atomically with the outer statement. This is
the backbone of "idempotent single-statement ETL" patterns.

---

## 4. Syntax and mechanics (with examples)

### 4.1 Basic form and column aliases

```sql
WITH revenue AS (
    SELECT o.customer_id,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS amount
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY o.customer_id
)
SELECT c.full_name, r.amount
FROM customers c JOIN revenue r USING (customer_id)
ORDER BY r.amount DESC;

-- Explicit column list (renames + documents the contract)
WITH revenue (customer_id, amount) AS (
    SELECT o.customer_id, SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100))
    FROM orders o JOIN order_items i USING (order_id)
    GROUP BY o.customer_id
)
SELECT * FROM revenue;
```

### 4.2 Multiple CTEs, referencing earlier ones

```sql
WITH
base AS (
    SELECT o.order_id, o.customer_id, o.status, o.ordered_at,
           i.quantity * i.unit_price * (1 - i.discount_pct/100) AS line_amount
    FROM orders o JOIN order_items i USING (order_id)
),
order_totals AS (            -- may reference base
    SELECT order_id, customer_id, status, ordered_at, SUM(line_amount) AS total
    FROM base GROUP BY order_id, customer_id, status, ordered_at
),
per_customer AS (            -- may reference order_totals
    SELECT customer_id, SUM(total) AS lifetime_value, COUNT(*) AS orders
    FROM order_totals
    WHERE status = 'delivered'
    GROUP BY customer_id
)
SELECT c.full_name, p.lifetime_value, p.orders
FROM per_customer p JOIN customers c USING (customer_id)
ORDER BY p.lifetime_value DESC;
```

### 4.3 `WITH` anywhere a subquery is allowed

```sql
-- Inside INSERT
WITH new_customers AS (
    SELECT 'ivy@example.com'::text AS email, 'Ivy Chen'::text AS full_name, 'SG'::char(2) AS country_code
)
INSERT INTO customers (email, full_name, country_code)
SELECT email, full_name, country_code FROM new_customers;

-- Inside UPDATE
WITH flagged AS (SELECT customer_id FROM orders WHERE status = 'returned')
UPDATE customers SET loyalty_tier = 'bronze'
WHERE customer_id IN (SELECT customer_id FROM flagged);

-- Inside DELETE
WITH stale AS (SELECT order_id FROM orders WHERE status = 'cancelled' AND ordered_at < '2024-01-01')
DELETE FROM orders WHERE order_id IN (SELECT order_id FROM stale);

-- Nested scope: an inner WITH sees the outer one
WITH a AS (SELECT 1 AS x)
SELECT * FROM (WITH b AS (SELECT x + 1 AS y FROM a) SELECT * FROM b) t;
```

### 4.4 Forcing the mode

```sql
WITH heavy AS MATERIALIZED ( SELECT ... ),     -- guarantee: one execution, sealed
     base  AS NOT MATERIALIZED ( SELECT ... )  -- guarantee: fold into the parent
SELECT ...
```

### 4.5 What CTEs are **not**

- Not a temp table: you cannot index it, you cannot reference it from a second statement.
- Not a view: it does not persist and is not shared across sessions.
- Not a cursor: no row-by-row access.
- Not a transaction boundary (unless data-modifying).

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: revenue per customer

```sql
WITH order_totals AS (
    SELECT o.order_id, o.customer_id,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS total
    FROM orders o
    JOIN order_items i ON i.order_id = o.order_id
    WHERE o.status = 'delivered'
    GROUP BY o.order_id, o.customer_id
)
SELECT c.full_name, SUM(ot.total) AS revenue, COUNT(*) AS delivered_orders
FROM order_totals ot
JOIN customers c ON c.customer_id = ot.customer_id
GROUP BY c.full_name
ORDER BY revenue DESC;
```

Expected: `Frank Miller 1736.00 (2 orders)`, `Alice Johnson 1073.95 (2)`, `Dave Brown
2028.99 (1)`, `Bob Smith 919.00 (1)`. (Read the actual output — the point is that the CTE
names the grain.)

### Example 2 — Refactoring a nested subquery into CTEs (identical semantics, readable form)

```sql
-- BEFORE: four levels of nesting, right-to-left reading
SELECT c.country_code, SUM(x.total)
FROM customers c
JOIN (SELECT o.customer_id,
             i.quantity * i.unit_price * (1 - i.discount_pct/100) AS total
      FROM orders o
      JOIN order_items i USING (order_id)
      WHERE o.status = 'delivered'
        AND o.order_id IN (SELECT order_id FROM orders WHERE ordered_at >= '2024-01-01')) x
  ON x.customer_id = c.customer_id
GROUP BY c.country_code;

-- AFTER: same result, top-to-bottom reading
WITH delivered_2024 AS (
    SELECT order_id, customer_id
    FROM orders
    WHERE status = 'delivered' AND ordered_at >= '2024-01-01'
),
lines AS (
    SELECT d.customer_id,
           i.quantity * i.unit_price * (1 - i.discount_pct/100) AS amount
    FROM delivered_2024 d
    JOIN order_items i ON i.order_id = d.order_id
)
SELECT c.country_code, SUM(l.amount) AS revenue
FROM lines l
JOIN customers c USING (customer_id)
GROUP BY c.country_code
ORDER BY revenue DESC;
```

Both produce `GB 2028.99`, `US 2809.95`, `DE 899.00` (total `5737.94`). The CTE version is what
you would review and what you would extend.

### Example 3 — Intermediate: funnel conversion using two CTEs

```sql
WITH sessions AS (
    SELECT session_id,
           customer_id,
           MIN(occurred_at)                                   AS started_at,
           MAX(event_type = 'add_to_cart')::int               AS added_to_cart,
           MAX(event_type IN ('checkout','purchase'))::int    AS reached_checkout,
           MAX(event_type = 'purchase')::int                  AS purchased
    FROM events
    GROUP BY session_id, customer_id
)
SELECT COUNT(*)                                        AS sessions,
       SUM(added_to_cart)                              AS add_to_cart_sessions,
       SUM(reached_checkout)                           AS checkout_sessions,
       SUM(purchased)                                  AS purchase_sessions,
       ROUND(100.0 * SUM(added_to_cart)   / COUNT(*), 1) AS cart_rate_pct,
       ROUND(100.0 * SUM(reached_checkout)/ NULLIF(SUM(added_to_cart),0), 1) AS checkout_of_cart_pct
FROM sessions;
```

Expected on the seed (5 sessions: Alice ×2, Bob, Heidi, plus one anonymous session):
`sessions=5`, `add_to_cart_sessions=2`, `checkout_sessions=2`, `purchase_sessions=2`,
`cart_rate_pct=40.0`, `checkout_of_cart_pct=100.0`. Note that Bob's session reaches checkout
without purchasing, and the anonymous session only views — that is exactly the shape a funnel
report has to survive. The `NULLIF(...,0)` is the standard guard against division-by-zero in
conversion math.

### Example 4 — Intermediate: the same CTE referenced **twice** (watch materialization)

```sql
WITH per_country AS (
    SELECT o.country_code, SUM(i.quantity * i.unit_price) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY o.country_code
)
SELECT p.country_code, p.revenue,
       ROUND(100.0 * p.revenue / (SELECT SUM(revenue) FROM per_country), 1) AS pct_of_total
FROM per_country p
WHERE p.revenue > (SELECT AVG(revenue) FROM per_country)
ORDER BY p.revenue DESC;
```

Because `per_country` is referenced three times, Postgres **materializes** it. In
`EXPLAIN` you see a `CTE Scan` over a `CTE per_country`. That is good here (the aggregate is
computed once). Now run the same query with `per_country AS NOT MATERIALIZED (...)` and
compare `EXPLAIN (ANALYZE)` time — on small data the numbers are identical; on 100M rows the
difference is one aggregation versus three.

### Example 5 — Advanced: forcing materialization to fix a plan

```sql
-- Baseline: the planner inlines `recent` and picks a nested loop with 3 different
-- outer inputs. Adding MATERIALIZED makes it a single, reusable result.
WITH recent AS MATERIALIZED (
    SELECT order_id, customer_id, ordered_at
    FROM orders
    WHERE ordered_at >= '2024-03-01'
)
SELECT c.full_name,
       (SELECT COUNT(*) FROM recent r WHERE r.customer_id = c.customer_id) AS recent_orders,
       (SELECT MAX(r.ordered_at) FROM recent r WHERE r.customer_id = c.customer_id) AS last_order_at
FROM customers c
ORDER BY recent_orders DESC, c.full_name;
```

Expected (as-of the seed): `Carol White 2`, `Erin Davis 2`, `Alice Johnson 1`,
`Dave Brown 1`, `Grace Lee 1`, everyone else `0`. Two correlated subqueries over the same
`recent` set would execute twice if inlined; `MATERIALIZED` guarantees one scan.

`> Senior note`: `MATERIALIZED` is a *hint to stop optimizing*, not a performance fix by
itself. On Postgres 11 and older, **every** CTE was materialized (the "optimization fence"
era). Code written then often relies on it implicitly; after upgrading to 12+, a CTE that
was materialized may get inlined and the plan flips. That is the most common
"nothing changed but the query got slower" upgrade story for Postgres 12.

### Example 6 — Advanced: data-modifying CTE (atomic multi-table write)

```sql
-- Create an order and its first line item atomically; print the new order_id.
WITH new_order AS (
    INSERT INTO orders (customer_id, status, ordered_at, country_code, channel)
    VALUES (8, 'pending', now(), 'AT', 'web')
    RETURNING order_id
), new_items AS (
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    SELECT order_id, 10, 1, 89.00 FROM new_order
    RETURNING order_id, product_id
)
SELECT o.order_id, i.product_id
FROM new_order o JOIN new_items i USING (order_id);
```

Expected: one row with the freshly generated `order_id` and `product_id = 10`. Note the
ordering guarantee: `new_items` sees `new_order` only through `RETURNING`, because all
sub-statements in a `WITH` share one snapshot and **cannot see each other's writes**.
This is also how you can enforce "audit row written iff the insert succeeded" without a
client-side transaction.

### Example 7 — Advanced: CTE vs subquery vs temp table (same answer, different contracts)

```sql
-- (a) CTE: one statement, readable, engine decides materialization
WITH x AS (SELECT customer_id, SUM(quantity*unit_price) AS amt FROM order_items oi
           JOIN orders o USING (order_id) WHERE o.status='delivered'
           GROUP BY customer_id)
SELECT * FROM x WHERE amt > 500;

-- (b) Subquery: identical to a CTE in most engines, but you cannot reference it twice
SELECT * FROM (
  SELECT customer_id, SUM(quantity*unit_price) AS amt FROM order_items oi
  JOIN orders o USING (order_id) WHERE o.status='delivered'
  GROUP BY customer_id) x
WHERE amt > 500;

-- (c) Temp table: a real object. Indexable, reusable across statements,
--     pays the "time to first row" cost up front, breaks the query into two.
CREATE TEMP TABLE cust_revenue ON COMMIT DROP AS
SELECT customer_id, SUM(quantity*unit_price) AS amt
FROM order_items oi JOIN orders o USING (order_id)
WHERE o.status='delivered'
GROUP BY customer_id;
CREATE INDEX ON cust_revenue (amt);
ANALYZE cust_revenue;
SELECT c.full_name, r.amt FROM cust_revenue r JOIN customers c USING (customer_id) WHERE amt > 500;
```

Decision guide:

| Need | Use |
|------|-----|
| Readability inside one statement | CTE |
| Reference 2–3× inside one statement | CTE (accept the fence) or `MATERIALIZED` deliberately |
| Reference many times, or need an index, or need statistics | Temp table |
| Reused by many queries / users | View or materialized view |
| Recursive traversal | Recursive CTE (see next chapter) |
| One-off expression in a `SELECT` list | Scalar subquery |

### Example 8 — Advanced: finding the CTE that is actually the bottleneck

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
WITH lines AS (
    SELECT i.order_id, i.product_id,
           i.quantity * i.unit_price * (1 - i.discount_pct/100) AS amount
    FROM order_items i
),
by_cat AS (
    SELECT p.category_id, SUM(l.amount) AS revenue
    FROM lines l JOIN products p USING (product_id)
    GROUP BY p.category_id
)
SELECT cat.name, b.revenue
FROM by_cat b JOIN categories cat USING (category_id)
ORDER BY b.revenue DESC;
```

Read the plan bottom-up. If you see `CTE Scan on by_cat` with `actual rows` far below the
estimate, the join to `categories` was not pushed into the CTE; try removing the CTE layer
(`NOT MATERIALIZED`, or inline the join) and re-measure. If instead you see an enormous
`Seq Scan on order_items` repeated twice, you have an inline-mode double evaluation and
should name it once and materialize.

---

## 6. Real-world use cases and applications

| Use case | Why a CTE fits |
|----------|----------------|
| **dbt / SQLMesh model structure** | Every model is a chain of CTEs (`source → staging → intermediate → final`); the tool compiles the final CTE as the materialization. |
| **Funnel / step-by-step conversion** | Each step is a named set; the reader can verify the drop-off per stage. |
| **Cohort tables** | `cohort_users` CTE referenced by retention and revenue CTEs (2+ references → materialized, which is what you want). |
| **Idempotent loads (Postgres)** | Data-modifying CTEs write fact + dimension rows in a single atomic statement. |
| **Readability review in code review** | Diffing a CTE-based query shows the changed stage; diffing a nested query does not. |
| **Airflow/Spark SQL tasks** | The same `WITH` text runs on Postgres for dev and on a warehouse for prod, with minor dialect tweaks. |
| **Guard-railing expensive scans** | `AS MATERIALIZED` pins an expensive scan to one execution when it is used by three downstream aggregations. |

---

## 7. Performance notes and anti-patterns

**The materialization decision tree**

```
Is the CTE referenced more than once?  ── yes ──► materialize (or temp table if huge)
        │ no
Is it recursive?                        ── yes ──► always iterative (see ch. 03)
        │ no
Does it contain a volatile function?    ── yes ──► materialize (Postgres); beware in
        │                                            SQL Server (may differ per reference)
        │ no
Is the outer filter highly selective?   ── yes ──► ensure inlining (NOT MATERIALIZED),
        │                                            otherwise pushdown is lost
        └─ default: let the planner decide, then VERIFY with EXPLAIN
```

**Anti-patterns**

| Anti-pattern | Symptom | Fix |
|---|---|---|
| CTE used as a "temp table" in Postgres 11 style | Query got slower after upgrading to 12+ | add `MATERIALIZED`, or restructure |
| `WITH x AS (...) SELECT * FROM x` (single use) | Redundant fence | inline it or drop the CTE |
| CTE referenced twice in SQL Server / BigQuery | Work duplicated, cost doubled | `#temp` / temp table, or restructure to one reference |
| CTE that only renames columns | Noise | remove |
| Deep CTE chains (15+) | Unreadable, hard to debug, planner struggles | split into 2–3 materialized steps (dbt models) |
| `WITH RECURSIVE` when iteration is unnecessary | Slow convergence, `LIMIT` loops | use `generate_series` / window functions when possible |
| CTE shadowing a table name | Silent wrong data source | never name a CTE like a table |
| Relying on CTE `ORDER BY` | Order is not guaranteed to survive | order in the final `ORDER BY`; if you need ordered rows, add an ordinal column |
| Relying on a CTE as a barrier for correctness (SQL Server) with volatile functions | Two references give different values | materialize into `#temp` |

**Diagnostics**

```sql
-- Postgres: see whether you got a CTE Scan (materialized) or inlined subtree
EXPLAIN (ANALYZE, COSTS, BUFFERS, VERBOSE) <your query>;

-- Postgres: detect double execution of an inline CTE
EXPLAIN (ANALYZE) ...;   -- look for the same subplan appearing twice with similar actual time

-- MySQL: is the CTE materialized?
EXPLAIN FORMAT=JSON <your query>;     -- look for "materialized" / "materialized_from_subquery"
```

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `WITH name AS (...)` | ✅ | ✅ (8.0) | ✅ | ✅ | ✅ | ✅ |
| `WITH` in `INSERT/UPDATE/DELETE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Column list `WITH x(a,b) AS` | ✅ | ✅ (8.0.19+ for full support) | ✅ | ✅ | ✅ | ✅ |
| Default mode | inline if 1 reference, else materialize (12+) | merge or materialize (optimizer) | always inline | inline, hintable | inline where possible | inline |
| Force materialize | `AS MATERIALIZED (...)` | ❌ no hint | ❌ (use `#temp`) | `/*+ MATERIALIZE */` | ❌ (use temp table) | ❌ |
| Force inline | `AS NOT MATERIALIZED (...)` | ❌ | n/a (already inline) | `/*+ INLINE */` | n/a | n/a |
| Data-modifying CTE | ✅ (`RETURNING`) | ❌ | ❌ | ❌ | ❌ (use `INSERT ... RETURNING` separately; Snowflake supports `RETURNING` in some statements) | ❌ |
| Recursive `WITH` | ✅ `WITH RECURSIVE` | ✅ 8.0 | ✅ `WITH` (no `RECURSIVE` keyword) | ✅ `WITH` (+ `CONNECT BY`) | ✅ | ✅ `WITH RECURSIVE` |
| Multiple references evaluated once? | only if materialized | depends | ❌ usually twice | hintable | usually | ❌ usually twice |
| Temp-table alternative | `CREATE TEMP TABLE … ON COMMIT DROP` | `CREATE TEMPORARY TABLE` | `#temp` / `##temp` | global/local temp tables | `CREATE TEMPORARY TABLE` | `CREATE TEMP TABLE` |
| Scoped to session? | yes (temp tables) | yes | yes | yes | yes | yes (script/job) |

Dialect gotchas:

- **SQL Server** does not use the `RECURSIVE` keyword; `WITH x AS (...)` is recursive
  automatically when it self-references. It also requires a preceding statement to be
  terminated with `;` before `WITH`.
- **Oracle** requires `WITH` clauses to precede `SELECT` and does not allow `WITH` after
  `INSERT` without `SELECT`; it also has hierarchical `CONNECT BY` as an alternative.
- **BigQuery** forbids forward references and does not support `WITH` in `UPDATE`/`DELETE`
  DML statements (only `INSERT`, and `MERGE` via subqueries).
- **MySQL 8.0.19+** added proper support for `WITH`-in-DML and column-list renaming.
- **Snowflake** supports `WITH` before `INSERT` and `SELECT`, plus recursive CTEs, but does
  not support `MATERIALIZED`; write to a temp table instead.

---

## 9. Exercises — Intermediate (hints only)

1. **Convert the nested query from Example 2 into CTEs and verify byte-identical results.**
   *Hint:* compare with `EXCEPT` both ways (see [Chapter 09](./09-set-operations.md)) — the
   result of `A EXCEPT B UNION B EXCEPT A` must be empty.
2. **Compute each customer's first and last order date with two CTEs.** *Hint:* one CTE
   aggregates `MIN/MAX(ordered_at)`; the second joins to `customers`.
3. **Write a `WITH` chain that classifies customers as `never`, `one_order`,
   `repeat`.** *Hint:* aggregate counts, then `CASE`; this previews
   [Chapter 05](./05-case-expressions.md).
4. **Use one CTE and reference it twice: total revenue and each category's share of it.**
   *Hint:* cross join the CTE with its own `SUM`.
5. **Rewrite a `SELECT * FROM (SELECT ... ) t WHERE ...` into a CTE and explain the
   difference in how the planner treats it.** *Hint:* in Postgres, both may be inlined; check
   `EXPLAIN`.
6. **Add a CTE that produces a "calendar" of dates with `generate_series` and left join
   orders to it.** *Hint:* `GENERATE_SERIES(date '2024-01-01', date '2024-03-31', interval '1 day')`;
   then `LEFT JOIN orders ON ordered_at::date = d::date AND status = 'delivered'`.
7. **Given a CTE containing a volatile function (`random()`, `now()`), show that two
   references in Postgres return the same value while two references in SQL Server may
   not.** *Hint:* write the same query in each dialect and compare.
8. **Measure the difference between `MATERIALIZED` and `NOT MATERIALIZED` for a
   3-reference CTE** on your own dataset. *Hint:* `EXPLAIN (ANALYZE, BUFFERS)` and compare
   `actual time` and buffer counts.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Build a 4-stage pipeline for a cohort retention table** (cohort by signup month →
   activity per month → months-since-signup → retention matrix). *Approach:* stage each
   step, keep grains explicit in a comment per CTE, and decide which CTEs must be
   `MATERIALIZED` (the ones referenced more than once). Explain where you would switch to a
   temp table and why.

2. **Prove that a multiply referenced CTE is executed once in your engine.** *Approach:*
   put a `pg_sleep`-like or a volatile counter into the CTE (Postgres: `random()` combined
   with a row count is not enough — use `EXPLAIN ANALYZE` timing, or a temp table with a
   trigger counter), run with and without `MATERIALIZED`, and interpret the plan.

3. **Design an idempotent single-statement load using data-modifying CTEs.** Insert into
   `orders`, capture `RETURNING`, insert into `order_items` with a `ON CONFLICT DO NOTHING`
   guard, and return a status per row. *Approach:* `ins` CTE + `RETURNING`, then a second
   CTE that joins the returning set to a staging relation. Discuss the concurrency caveat
   that the CTEs share one snapshot.

4. **Write a query that is correct only if you name the CTE, not if you inline it**,
   because the outer predicate would change the semantic (e.g. `LIMIT` inside an inlined
   CTE is not pushed down). *Approach:* a "top-N per group then aggregate" query; inlining
   turns `LIMIT N` into a global `LIMIT` effect. Explain with `EXPLAIN` output.

5. **Refactor a 250-line nested query into CTEs and quantify the plan change.**
   *Approach:* capture `EXPLAIN (ANALYZE, BUFFERS)` before/after, tabulate the top 3 nodes
   by total time, and decide per node whether to keep the fence, drop it, or move the node
   into a temp table with an index.

6. **Simulate the Postgres 11→12 upgrade regression.** Take a query with a highly selective
   outer filter over a single-reference CTE, show the inlined plan (Postgres 12+) versus the
   forced-materialized plan, and explain the `Rows Removed by Filter` counts. *Approach:*
   `WITH x AS MATERIALIZED` vs `WITH x AS NOT MATERIALIZED`, then read the plan for the
   filter/sort node.

7. **Decide between CTE and temp table with a cost model.** For a CTE referenced `k` times
   with a materialization cost `C` and a fan-out of `N` rows, write down when the temp table
   wins (index selectivity, statistics availability, downstream joins). *Approach:* reason
   with `k × cost(inline)` vs `cost(materialize) + k × cost(scan)` and validate empirically
   with `EXPLAIN ANALYZE`.

---

## References

**PostgreSQL**
- 7.8 `WITH` Queries (CTEs), materialization, data-modifying CTEs: https://www.postgresql.org/docs/current/queries-with.html
- `SELECT` reference (including `MATERIALIZED` / `NOT MATERIALIZED`): https://www.postgresql.org/docs/current/sql-select.html

**MySQL**
- `WITH` (CTEs): https://dev.mysql.com/doc/refman/8.0/en/with.html
- Derived table merging vs materialization (optimizer): https://dev.mysql.com/doc/refman/8.0/en/derived-table-optimization.html

**SQL Server**
- `WITH common_table_expression`: https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql
- Recursive queries using CTEs: https://learn.microsoft.com/en-us/sql/t-sql/queries/recursive-common-table-expression-transact-sql

**Oracle**
- `SELECT` / subquery factoring clause: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake `WITH`: https://docs.snowflake.com/en/sql-reference/constructs/with
- BigQuery query syntax (`WITH` clause): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Markus Winand, *WITH — organize complex queries*: https://modern-sql.com/feature/with
- DuckDB `WITH` documentation (a useful cross-check for semantics): https://duckdb.org/docs/sql/query_syntax/with.html
- PostgreSQL Wiki, *Don't Do This*: https://wiki.postgresql.org/wiki/Don%27t_Do_This

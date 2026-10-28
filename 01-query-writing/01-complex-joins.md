# Complex JOINs

> A JOIN is the only operation in SQL that can silently multiply your numbers. Everything
> downstream — revenue, churn, retention, unit economics — is only as trustworthy as the
> join predicates you wrote. This chapter builds the mental model you need to write joins
> that are *correct first* and *fast second*: the four outer join variants, self-joins,
> multi-way join trees, non-equi (range) joins, `LATERAL` (a correlated for-each loop the
> optimizer can actually plan), and the semi/anti join patterns that show up in every
> real data pipeline.

This file also defines the **shared sample schema** used by every chapter in Part II.
Load it once, then run every example in this section of the book.

---

## TL;DR

- SQL has **five logical join flavors**: `INNER`, `LEFT [OUTER]`, `RIGHT [OUTER]`,
  `FULL [OUTER]`, `CROSS`. `RIGHT` is `LEFT` with inputs swapped; `FULL` is
  `LEFT UNION RIGHT` with deduplication of matched rows.
- An outer join = inner join **plus null-extension**. `ON` decides *which pairs match*;
  `WHERE` filters *after* extension. Putting a predicate on the null-extended side in
  `WHERE` silently turns your `LEFT JOIN` into an `INNER JOIN`. This is the #1 join bug.
- **Logical join type ≠ physical algorithm.** The engine picks *nested loop*, *hash join*,
  or *merge join* based on cardinality estimates, indexes, and data distribution.
- **Join order does not change the result of inner joins, but it changes the cost by
  orders of magnitude.** Filter/aggregate early, and reduce row counts before you join.
- **Fan-out (row multiplication)** is the silent data-quality killer. Verify every join
  with a cardinality probe (`COUNT(*)` before vs after) before you trust the number.
- `LATERAL` (Postgres/Oracle/MySQL 8.0.14+/SQL Server's `APPLY`) makes a correlated
  subquery a first-class join input — the canonical "top-N per group" and
  "most-recent-row" solution without window functions.
- `EXISTS` / `IN` are **semi joins**; `NOT EXISTS` / `LEFT JOIN ... IS NULL` are
  **anti joins**. They answer "has a match?" without fan-out. Prefer them over joins +
  `DISTINCT` when you only need existence.
- **Non-equi joins** (`BETWEEN`, `<`, `>=`, `<>`) cannot use a hash join and are the most
  common cause of accidental cross joins in production.

---

## 1. Definitions and core concepts

### 1.1 Vocabulary you must be precise about

| Term | Meaning |
|------|---------|
| **Left input / right input** | The two row sources of a binary join. Joins are strictly binary; three tables = two joins. |
| **Join tree** | The binary tree of join operators. Its *shape* (left-deep, right-deep, bushy) determines which intermediate results materialize. |
| **Join predicate** | The boolean expression that decides whether a pair of rows matches (`ON`, `USING`, or the `WHERE`-implicit form). |
| **Equi-join** | Predicate is `l.k = r.k`. Enables hash and merge joins. |
| **Non-equi / theta join** | Predicate uses `<`, `>`, `BETWEEN`, `<>`, or a function. Usually forces nested-loop-ish plans. |
| **Null-extension** | When an outer join finds no match, the engine emits the preserved row padded with `NULL`s. |
| **Semi join** | Returns left rows that *have at least one* match; does not duplicate left rows. |
| **Anti join** | Returns left rows that have *no* match. |
| **Fan-out / replication factor** | How many output rows one input row produces. `1:N` join → left fan-out of `N`. |
| **Join cardinality** | Estimated rows out of a join = `|L| × |R| × selectivity`. Wrong estimates → wrong algorithm → 100× slowdowns. |
| **Join order** | The sequence in which the `n` tables are combined. `n!` permutations exist; the optimizer searches a subset. |

### 1.2 Logical vs physical joins

This distinction is where most confusion lives.

- **Logical (you write):** `INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`, `LATERAL`, semi, anti.
- **Physical (the engine picks):** *nested loop*, *index nested loop*, *hash join*,
  *sort-merge join*, *block nested loop*, *adaptive/grace hash join*, *broadcast hash join*
  (distributed engines), *merge/partition-wise join*.

You write the logical join; the planner maps it to one or more physical algorithms. Two
queries with identical semantics can run in 40 ms and 40 minutes depending on that
mapping — which is why [Chapter 05 of the optimization section](../02-query-optimization/05-join-algorithms.md)
exists.

### 1.3 Cardinality vocabulary

For `A JOIN B`:

- `1:1` — at most one match per row (e.g. `orders JOIN invoices` if invoices is unique per order).
- `1:N` — e.g. `orders JOIN order_items`. This is **fan-out**: 1 order with 4 items → 4 rows.
- `N:M` — e.g. `products JOIN promotions` where a product matches several promos.
- `0:1` / `1:0` — the gap cases that `LEFT`/`FULL` joins expose.

`> Senior note`: before you write `SUM(x)` on a joined result set, ask "what is the grain
of the output?" If the grain changed because of a join, your aggregate is now weighted by
the fan-out factor. That is how "revenue doubled" bugs are born.

---

## 2. Why it matters / what problem it solves

Relational modeling normalizes data into many narrow tables (customers, orders, order
items, products, categories). Almost no business question is answerable from one table:

- *"Which customers never ordered?"* → anti join.
- *"Monthly revenue by product category."* → 5-table join.
- *"Top 3 products per category."* → join + window or `LATERAL`.
- *"Reconcile our warehouse revenue against the payment provider."* → `FULL OUTER JOIN`.
- *"Which employees report (directly or indirectly) to X?"* → self-join (or recursive CTE).

Joins are also the operation that **converts a normalized model into an analytical one**,
which is literally what every dbt model, Spark job, and warehouse view does. Getting joins
wrong does not produce an error — it produces a plausible-looking wrong number, which is
strictly worse.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Logical clause evaluation order

```
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT (window fns, CASE)
             →  DISTINCT  →  ORDER BY  →  LIMIT/OFFSET
```

Two consequences matter for joins:

1. `ON` is evaluated as part of `FROM`. `WHERE` runs *after* the join produced rows.
2. Column aliases defined in `SELECT` are not visible in `WHERE` (but are in `ORDER BY`).

### 3.2 Outer join = inner join + null-extension

Semantically, the engine:

1. Finds all **matching pairs** (inner join result).
2. For every left row with **zero matches**, emits the left row with all right columns `NULL`.
3. `FULL` additionally patches in right rows with zero matches, left columns `NULL`.

```
customers            orders
---------            ------
1  Alice   ├──match──┤ 1001 cust=1
2  Bob     ├──match──┤ 1003 cust=2
3  Carol   │  NO MATCH
             │
LEFT JOIN  =>  1 Alice 1001
               2 Bob   1003
               3 Carol NULL   <-- null-extension
```

This is why the classic bug happens:

```sql
-- Looks like a LEFT JOIN, behaves like an INNER JOIN:
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'delivered';   -- NULL = 'delivered' is UNKNOWN -> row dropped
```

The fix is to move the predicate into the `ON` clause (or to accept you wanted an inner
join):

```sql
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
      AND o.status = 'delivered'   -- participates in the match test only
```

### 3.3 The three physical join algorithms (one paragraph each)

- **Nested loop join**: for each left row, scan the right input (or probe an index). Cost
  `O(|L| × |R|)` without an index, `O(|L| × log|R|)` with one. Excellent when the left side
  is tiny and the right side is uniquely indexed (this is exactly the "N+1 query" the ORM
  does, except the database does it in one plan). Cannot handle `<>`.
- **Hash join**: build a hash table on the smaller input, probe with the larger. Requires an
  equi-join (`=`), costs `O(|L| + |R|)`, needs memory (`work_mem` in Postgres). In
  distributed engines it becomes a *shuffle hash join* (both sides repartitioned) or a
  *broadcast hash join* (small side replicated to every worker).
- **Sort-merge join**: sort both inputs on the join key, then walk them like a zipper.
  Great when inputs are already sorted (index order) and for range predicates; also
  handles `FULL` joins naturally because both streams are ordered.

You can *steer* the engine with planner knobs (`SET enable_hashjoin = off;` in Postgres) —
useful for diagnosis, never for production.

### 3.4 Join trees, join order, and the optimizer

For `n` tables the planner must choose an order and an algorithm per join. Postgres uses
dynamic programming for small `n` and *GEQO* (a genetic algorithm) for `n ≥ geqo_threshold`
(default 12). Two practical rules follow:

1. **Reduce before you join.** A `WHERE` filter that cuts `orders` from 10M to 50k rows
   before the join is worth more than any index.
2. **Pre-aggregate one side.** Joining `orders` (10M) to `order_items` (40M) then summing
   is 40M intermediate rows; aggregating `order_items` to order level first (10M) halves
   the work.

You can influence the shape by *nesting* joins with parentheses/subqueries, but in modern
engines (Postgres, Snowflake, BigQuery, Spark) the optimizer is free to reorder inner
joins. **Outer joins constrain reordering** — the optimizer cannot push a table across the
preserved side of an outer join — which is another reason to keep outer joins minimal.

### 3.5 `LATERAL` semantics: a legal correlated for-each

`LATERAL` means: *this subquery may reference columns of tables to its left in the same
`FROM`, and it is evaluated once per left row.* Conceptually:

```
FOR EACH row r IN left_input:
    output := r × evaluate(right_subquery, bindings=r)
```

`CROSS JOIN LATERAL` behaves like an inner join; `LEFT JOIN LATERAL ... ON TRUE` preserves
left rows that produce zero rows (like a `LEFT JOIN`). Because it is a real join node —
not a correlated scalar subquery — the engine can push a `LIMIT` into it, which is what
makes "top-3 per group" efficient.

### 3.6 Semi and anti joins

`EXISTS` and `IN` are semi-join predicates: they stop at the *first* match and never
duplicate the left row. `NOT EXISTS` and `LEFT JOIN ... IS NULL` are anti joins. A
well-behaved optimizer turns these into `Hash Semi Join` / `Hash Anti Join` plan nodes —
visible in `EXPLAIN`. If you see `Nested Loop` with a `Filter` for a `NOT EXISTS`, your
predicate is probably non-equi and the plan is quadratic.

`> Senior note`: `NOT IN` is **not** an anti join when the subquery can return `NULL` — the
predicate becomes `UNKNOWN` and the query returns *zero rows*. See
[Chapter 06](./06-exists-and-subqueries.md) for the full NULL-trap taxonomy.

---

## 4. Syntax and mechanics (with examples)

### 4.1 The five join forms

```sql
-- 1. INNER JOIN: only matching pairs
SELECT c.full_name, o.order_id
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;          -- INNER is the default

-- 2. LEFT OUTER JOIN: keep every left row
SELECT c.full_name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;

-- 3. RIGHT OUTER JOIN: keep every right row (== LEFT with inputs swapped)
SELECT c.full_name, o.order_id
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.customer_id;

-- 4. FULL OUTER JOIN: keep unmatched rows from both sides
SELECT c.full_name, o.order_id
FROM customers c
FULL JOIN orders o ON o.customer_id = c.customer_id;

-- 5. CROSS JOIN: cartesian product (no predicate)
SELECT c.full_name, p.name
FROM customers c
CROSS JOIN products p;                                   -- |customers| × |products| rows
```

### 4.2 `USING` and the `NATURAL JOIN` trap

`USING (col)` is shorthand for `ON l.col = r.col` and **merges the column** (it appears
once in `SELECT *`):

```sql
SELECT * FROM orders o JOIN order_items i USING (order_id);  -- one order_id column
```

`NATURAL JOIN` joins on *all* same-named columns. Never use it in production: adding a
column named `updated_at` to both tables silently changes the join predicate and the row
count. It is a footgun, not a feature.

### 4.3 Multi-way joins and join trees

```sql
-- Left-deep: ((A ⋈ B) ⋈ C) ⋈ D — the common, planner-friendly shape
SELECT c.country_code, cat.name, SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN orders    o   ON o.order_id    = oi.order_id
JOIN products  p   ON p.product_id  = oi.product_id
JOIN categories cat ON cat.category_id = p.category_id
JOIN customers c   ON c.customer_id = o.customer_id
WHERE o.status = 'delivered'
GROUP BY c.country_code, cat.name;
```

Force a shape explicitly when the optimizer guesses badly (rare, but useful in Oracle
where you can use hints, and in Postgres via CTEs with `MATERIALIZED`):

```sql
WITH big AS MATERIALIZED (SELECT ... FROM order_items WHERE ...)
SELECT ... FROM big JOIN ...;
```

### 4.4 Self-joins

A self-join is just the same table twice with **two different aliases**. Requires at least
one alias to be unambiguous:

```sql
-- Employee → manager name
SELECT e.full_name AS employee, m.full_name AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id;
```

Self-joins also express "compare a row with the previous row" without window functions, and
"find pairs of customers in the same country" (see exercises).

### 4.5 Non-equi (theta / range) joins

```sql
-- Assign an order to a price bucket using a range join against a literal table
SELECT o.order_id, o.total, b.label
FROM order_totals o
JOIN value_bands b
  ON o.total >= b.low AND o.total < b.high;
```

Range joins cannot use a hash join; they use nested loops or merge joins on the range
endpoints. They are also how "as-of joins" (join to the SCD2 dimension version that was
current at transaction time) are expressed:

```sql
SELECT f.sale_id, d.customer_name, d.tier
FROM fact_sales f
JOIN dim_customer d
  ON d.customer_id = f.customer_id
 AND f.sold_at >= d.valid_from
 AND f.sold_at <  COALESCE(d.valid_to, 'infinity');
```

This is the single most important join pattern in a dimensional warehouse.

### 4.6 `LATERAL` (Postgres/Oracle/MySQL 8.0.14+; `CROSS APPLY`/`OUTER APPLY` in SQL Server)

```sql
-- Top 3 most expensive products per category — one index-driven lookup per category
SELECT c.name AS category, p.name AS product, p.unit_price
FROM categories c
CROSS JOIN LATERAL (
    SELECT p.name, p.unit_price
    FROM products p
    WHERE p.category_id = c.category_id
    ORDER BY p.unit_price DESC
    LIMIT 3
) p;
```

`LEFT JOIN LATERAL (...) ON TRUE` keeps categories that have no products at all.

### 4.7 Semi / anti joins without fan-out

```sql
-- semi join: customers who have at least one delivered order
SELECT c.* FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'delivered');

-- anti join: customers who never ordered (three equivalent forms)
SELECT c.* FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

SELECT c.* FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;

SELECT c.* FROM customers c
WHERE c.customer_id NOT IN (SELECT customer_id FROM orders);  -- DANGEROUS if NULLs
```

### 4.8 Aggregation before joining (fan-out control)

```sql
-- WRONG: join first → each order row is duplicated by its item count
SELECT c.customer_id, SUM(o.total)
FROM customers c
JOIN orders o      ON o.customer_id = c.customer_id
JOIN order_items i ON i.order_id   = o.order_id
GROUP BY c.customer_id;    -- inflated by the number of items per order

-- RIGHT: aggregate order_items to the order grain, THEN join
SELECT c.customer_id, SUM(oi.line_total)
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN (SELECT order_id, SUM(quantity * unit_price * (1 - discount_pct/100)) AS line_total
      FROM order_items GROUP BY order_id) oi
  ON oi.order_id = o.order_id
GROUP BY c.customer_id;
```

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### 5.0 The shared sample schema (load this first)

Domain: a small multi-channel e-commerce ("Acme Commerce"). Everything in Part II runs on
this schema.

```sql
-- =====================================================================
-- Acme Commerce — shared sample schema (PostgreSQL 14+)
-- Drop in dependency order
-- =====================================================================
DROP TABLE IF EXISTS events, order_items, orders, products, categories,
                     customers, employees CASCADE;

CREATE TABLE customers (
    customer_id   bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    email         text        NOT NULL UNIQUE,
    full_name     text        NOT NULL,
    country_code  char(2)     NOT NULL,
    signup_ts     timestamptz NOT NULL DEFAULT now(),
    referred_by   bigint      REFERENCES customers(customer_id),
    loyalty_tier  text        CHECK (loyalty_tier IN ('bronze','silver','gold','platinum')),
    is_active     boolean     NOT NULL DEFAULT true
);

CREATE TABLE employees (
    employee_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    full_name   text   NOT NULL,
    manager_id  bigint REFERENCES employees(employee_id),
    title       text   NOT NULL,
    hire_date   date   NOT NULL,
    salary      numeric(12,2) NOT NULL
);

CREATE TABLE categories (
    category_id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name        text   NOT NULL,
    parent_id   bigint REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id  bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    sku         text   NOT NULL UNIQUE,
    name        text   NOT NULL,
    category_id bigint NOT NULL REFERENCES categories(category_id),
    unit_price  numeric(10,2) NOT NULL CHECK (unit_price >= 0),
    cost        numeric(10,2) NOT NULL,
    is_active   boolean NOT NULL DEFAULT true,
    attributes  jsonb
);

CREATE TABLE orders (
    order_id     bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id  bigint NOT NULL REFERENCES customers(customer_id),
    status       text   NOT NULL CHECK (status IN
                   ('pending','paid','shipped','delivered','returned','cancelled')),
    ordered_at   timestamptz NOT NULL,
    shipped_at   timestamptz,
    country_code char(2) NOT NULL,
    channel      text NOT NULL DEFAULT 'web'   -- web | mobile | marketplace | store
);

CREATE TABLE order_items (
    order_item_id bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id      bigint NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id    bigint NOT NULL REFERENCES products(product_id),
    quantity      int    NOT NULL CHECK (quantity > 0),
    unit_price    numeric(10,2) NOT NULL,       -- price captured at sale time
    discount_pct  numeric(5,2)  NOT NULL DEFAULT 0
);

CREATE TABLE events (
    event_id    bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id bigint REFERENCES customers(customer_id),
    session_id  uuid   NOT NULL,
    event_type  text   NOT NULL,   -- page_view | add_to_cart | checkout | purchase
    occurred_at timestamptz NOT NULL,
    properties  jsonb
);

-- Indexes that the join examples will rely on
CREATE INDEX idx_orders_customer   ON orders (customer_id, ordered_at DESC);
CREATE INDEX idx_orders_status     ON orders (status, ordered_at);
CREATE INDEX idx_items_order       ON order_items (order_id);
CREATE INDEX idx_items_product     ON order_items (product_id);
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_events_customer   ON events (customer_id, occurred_at DESC);
```

Seed data (deterministic IDs so the expected results below are reproducible):

```sql
INSERT INTO customers (customer_id, email, full_name, country_code, signup_ts,
                       referred_by, loyalty_tier, is_active) VALUES
 (1,'alice@example.com','Alice Johnson','US','2023-11-02', NULL,'gold',     true),
 (2,'bob@example.com',  'Bob Smith',    'DE','2023-12-15', 1,   'silver',   true),
 (3,'carol@example.com','Carol White',  'US','2024-01-08', 1,   'bronze',   true),
 (4,'dave@example.com', 'Dave Brown',   'GB','2023-10-21', NULL,'gold',     true),
 (5,'erin@example.com', 'Erin Davis',   'FR','2024-01-30', 2,   'silver',   true),
 (6,'frank@example.com','Frank Miller', 'US','2023-09-05', NULL,'platinum', true),
 (7,'grace@example.com','Grace Lee',    'JP','2024-02-14', 4,   'bronze',   true),
 (8,'heidi@example.com','Heidi Novak',  'AT','2024-03-01', NULL, NULL,      true);

INSERT INTO employees (employee_id, full_name, manager_id, title, hire_date, salary) VALUES
 (100,'Ada Lovelace',   NULL,'CEO',            '2015-01-05', 250000),
 (101,'Grace Hopper',    100,'VP Engineering', '2016-03-01', 210000),
 (102,'Alan Turing',     100,'VP Sales',       '2017-06-12', 205000),
 (103,'Barbara Liskov',  101,'Engineering Manager','2018-02-19', 180000),
 (104,'Dennis Ritchie',  103,'Staff Engineer', '2019-09-01', 165000),
 (105,'Ken Thompson',    103,'Engineer',       '2021-04-15', 140000),
 (106,'Margaret Hamilton',102,'Sales Manager','2019-11-11', 150000),
 (107,'Katherine Johnson',106,'Account Executive','2022-01-10', 120000),
 (108,'Edgar Codd',      103,'Engineer',       '2023-07-03', 145000);

INSERT INTO categories (category_id, name, parent_id) VALUES
 (1,'Electronics', NULL),
 (2,'Phones',      1),
 (3,'Laptops',     1),
 (4,'Accessories', 1),
 (5,'Cases',       4),
 (6,'Chargers',    4),
 (7,'Home',        NULL),
 (8,'Kitchen',     7);

INSERT INTO products (product_id, sku, name, category_id, unit_price, cost, is_active, attributes) VALUES
 (1,'SKU-PH-1','iPhone 15',        2,  999.00,  700.00, true, '{"color":"black","storage":128}'),
 (2,'SKU-PH-2','Galaxy S24',       2,  899.00,  620.00, true, '{"color":"white","storage":256}'),
 (3,'SKU-PH-3','Pixel 8',          2,  799.00,  560.00, true, '{"color":"blue","storage":128}'),
 (4,'SKU-LP-1','MacBook Air M3',   3, 1999.00, 1500.00, true, '{"ram_gb":16}'),
 (5,'SKU-LP-2','ThinkPad X1',      3, 1499.00, 1100.00, true, '{"ram_gb":32}'),
 (6,'SKU-CS-1','Phone Case',       5,   19.99,    4.00, true, '{"material":"silicone"}'),
 (7,'SKU-CS-2','Laptop Sleeve',    5,   29.99,    7.00, true, '{"material":"felt"}'),
 (8,'SKU-CH-1','65W Charger',      6,   39.99,   12.00, true, NULL),
 (9,'SKU-CH-2','USB-C Cable 2m',   6,   12.99,    3.00, true, NULL),
 (10,'SKU-KT-1','Coffee Maker',    8,   89.00,   40.00, true, NULL),
 (11,'SKU-KT-2','Blender',         8,   59.00,   25.00, true, NULL),
 (12,'SKU-KT-3','Electric Kettle', 8,   45.00,   18.00, false, NULL);

INSERT INTO orders (order_id, customer_id, status, ordered_at, shipped_at, country_code, channel) VALUES
 (1001,1,'delivered','2024-01-05 10:12:00+00','2024-01-07 09:00:00+00','US','web'),
 (1002,1,'delivered','2024-02-11 14:30:00+00','2024-02-13 08:15:00+00','US','mobile'),
 (1003,2,'delivered','2024-01-20 08:05:00+00','2024-01-22 07:45:00+00','DE','web'),
 (1004,2,'returned', '2024-03-02 19:40:00+00','2024-03-04 10:00:00+00','DE','web'),
 (1005,3,'paid',     '2024-03-15 11:20:00+00', NULL,                        'US','mobile'),
 (1006,3,'cancelled','2024-03-18 16:22:00+00', NULL,                        'US','web'),
 (1007,4,'delivered','2024-02-01 09:00:00+00','2024-02-03 11:30:00+00','GB','marketplace'),
 (1008,4,'shipped',  '2024-03-20 13:45:00+00','2024-03-22 09:10:00+00','GB','web'),
 (1009,5,'paid',     '2024-03-22 07:55:00+00', NULL,                        'FR','web'),
 (1010,5,'pending',  '2024-03-25 21:05:00+00', NULL,                        'FR','mobile'),
 (1011,6,'delivered','2024-01-15 12:00:00+00','2024-01-16 08:00:00+00','US','store'),
 (1012,6,'delivered','2024-02-20 15:10:00+00','2024-02-21 07:30:00+00','US','store'),
 (1013,7,'paid',     '2024-03-28 03:20:00+00', NULL,                        'JP','mobile'),
 (1014,1,'pending',  '2024-03-30 18:00:00+00', NULL,                        'US','web');

INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price, discount_pct) VALUES
 (1,1001, 1,1, 999.00, 0), (2,1001, 6,2,  19.99,10),
 (3,1002, 9,3,  12.99, 0),
 (4,1003, 2,1, 899.00, 0),
 (5,1004, 6,1,  19.99, 0),
 (6,1005,10,1,  89.00, 0), (7,1005,12,1,  45.00, 0),
 (8,1006, 8,1,  39.99, 0),
 (9,1007, 4,1,1999.00, 0), (10,1007,7,1,  29.99, 0),
 (11,1008, 8,2,  39.99, 0),
 (12,1009, 3,1, 799.00, 0),
 (13,1010, 9,1,  12.99, 0),
 (14,1011,10,2,  89.00, 0), (15,1011,11,1,  59.00, 0),
 (16,1012, 5,1,1499.00, 0),
 (17,1013, 6,1,  19.99, 0), (18,1013,9,1,  12.99, 0),
 (19,1014, 1,1, 999.00, 0);

INSERT INTO events (customer_id, session_id, event_type, occurred_at, properties) VALUES
 (1,'11111111-1111-1111-1111-111111111111','page_view',  '2024-01-05 09:50:00+00','{"path":"/"}'),
 (1,'11111111-1111-1111-1111-111111111111','add_to_cart','2024-01-05 10:05:00+00','{"sku":"SKU-PH-1"}'),
 (1,'11111111-1111-1111-1111-111111111111','purchase',   '2024-01-05 10:12:00+00','{"order_id":1001}'),
 (2,'22222222-2222-2222-2222-222222222222','page_view',  '2024-01-20 07:40:00+00','{"path":"/phones"}'),
 (2,'22222222-2222-2222-2222-222222222222','checkout',   '2024-01-20 08:02:00+00','{"order_id":1003}'),
 (1,'33333333-3333-3333-3333-333333333333','page_view',  '2024-02-11 14:00:00+00','{"path":"/accessories"}'),
 (1,'33333333-3333-3333-3333-333333333333','add_to_cart','2024-02-11 14:22:00+00','{"sku":"SKU-CH-2"}'),
 (1,'33333333-3333-3333-3333-333333333333','checkout',   '2024-02-11 14:28:00+00','{"order_id":1002}'),
 (1,'33333333-3333-3333-3333-333333333333','purchase',   '2024-02-11 14:30:00+00','{"order_id":1002}'),
 (8,'88888888-8888-8888-8888-888888888888','page_view',  '2024-03-05 12:00:00+00','{"path":"/"}'),
 (8,'88888888-8888-8888-8888-888888888888','page_view',  '2024-03-05 12:04:00+00','{"path":"/laptops"}'),
 -- Anonymous traffic: customer_id IS NULL. This is what makes `NOT IN (SELECT customer_id ...)`
 -- return zero rows — see 06-exists-and-subqueries.md §3.2.
 (NULL,'99999999-9999-9999-9999-999999999999','page_view','2024-03-06 08:10:00+00','{"path":"/"}'),
 (NULL,'99999999-9999-9999-9999-999999999999','page_view','2024-03-06 08:12:00+00','{"path":"/phones"}');

-- Keep identity sequences ahead of the seeded explicit IDs
SELECT setval(pg_get_serial_sequence('customers','customer_id'),   (SELECT max(customer_id)   FROM customers));
SELECT setval(pg_get_serial_sequence('employees','employee_id'),   (SELECT max(employee_id)   FROM employees));
SELECT setval(pg_get_serial_sequence('categories','category_id'),  (SELECT max(category_id)  FROM categories));
SELECT setval(pg_get_serial_sequence('products','product_id'),     (SELECT max(product_id)    FROM products));
SELECT setval(pg_get_serial_sequence('orders','order_id'),         (SELECT max(order_id)      FROM orders));
SELECT setval(pg_get_serial_sequence('order_items','order_item_id'),(SELECT max(order_item_id) FROM order_items));
```

**Schema at a glance** (this table is repeated in every chapter):

| Table | Grain | Key columns | Fan-out risk |
|-------|-------|-------------|--------------|
| `customers` | one row per customer | `customer_id`, `referred_by` (self-FK) | low |
| `employees` | one row per employee | `employee_id`, `manager_id` (self-FK) | low |
| `categories` | one row per category | `category_id`, `parent_id` (self-FK) | low |
| `products` | one row per SKU | `product_id`, `category_id` | low |
| `orders` | one row per order | `order_id`, `customer_id` | 1:N from customers |
| `order_items` | one row per line item | `order_item_id`, `order_id`, `product_id` | 1:N from orders **and** products |
| `events` | one row per behavioral event | `event_id`, `customer_id`, `session_id` | 1:N from customers |

### Example 1 — Basic INNER JOIN (inventory report)

```sql
SELECT p.name, p.unit_price, c.name AS category
FROM products p
JOIN categories c ON c.category_id = p.category_id
ORDER BY p.unit_price DESC;
```

Expected: 12 rows. `iPhone 15 / 999.00 / Phones`, `MacBook Air M3 / 1999.00 / Laptops`, …,
ending with `Phone Case / 19.99 / Cases`.

### Example 2 — LEFT JOIN + anti-join (customers who never ordered)

```sql
SELECT c.customer_id, c.full_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

Expected: **1 row** — `8 | Heidi Novak`. Every other customer has ≥1 order. Note that
Heidi *does* have a `page_view` event, so she is "engaged but never converted" — a classic
win-back segment.

### Example 3 — RIGHT JOIN (equivalent, inputs swapped)

```sql
SELECT c.full_name, o.order_id
FROM orders o
RIGHT JOIN customers c ON c.customer_id = o.customer_id
ORDER BY c.customer_id, o.order_id;
```

Expected: identical result to the equivalent `LEFT JOIN`. Rule of thumb: **never write
`RIGHT JOIN`** — swap the tables and write `LEFT JOIN` so the "preserved side" is always
on the left, which matches how humans read join chains.

### Example 4 — FULL OUTER JOIN (reconciling two sources)

Warehouse revenue vs payment-gateway revenue (simulated with aggregates):

```sql
WITH warehouse AS (
    SELECT order_id, SUM(quantity * unit_price * (1 - discount_pct/100)) AS amount
    FROM order_items GROUP BY order_id
), gateway AS (                 -- pretend the PSP missed order 1013 and invented 9999
    SELECT order_id, SUM(quantity * unit_price * (1 - discount_pct/100)) AS amount
    FROM order_items
    WHERE order_id <> 1013
    GROUP BY order_id
)
SELECT COALESCE(w.order_id, g.order_id) AS order_id,
       w.amount AS warehouse_amount,
       g.amount AS gateway_amount,
       CASE WHEN g.order_id IS NULL THEN 'missing_in_gateway'
            WHEN w.order_id IS NULL THEN 'missing_in_warehouse'
            WHEN w.amount <> g.amount THEN 'amount_mismatch'
            ELSE 'ok' END AS status
FROM warehouse w
FULL OUTER JOIN gateway g ON g.order_id = w.order_id
WHERE w.order_id IS NULL OR g.order_id IS NULL OR w.amount <> g.amount;
```

Expected: one row — `1013 | 32.98 | NULL | missing_in_gateway`. This is the canonical
financial reconciliation query: `FULL OUTER JOIN` is the only join type that surfaces
**both** sides of a discrepancy.

### Example 5 — Self-join (org chart with managers)

```sql
SELECT e.employee_id, e.full_name, e.title,
       COALESCE(m.full_name, '— none —') AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

Expected: 9 rows. `100 Ada Lovelace / CEO / — none —`, `101 Grace Hopper / VP Engineering /
Ada Lovelace`, `103 Barbara Liskov / Engineering Manager / Grace Hopper`, … The `LEFT JOIN`
is essential: Ada has `manager_id IS NULL`, and an inner join would hide the CEO.

### Example 6 — Multi-way join with pre-aggregation (revenue by category and country)

```sql
WITH order_revenue AS (      -- collapse 19 line items to 14 orders FIRST
    SELECT i.order_id,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM order_items i
    GROUP BY i.order_id
), item_revenue AS (         -- revenue per (order, product) for category attribution
    SELECT i.order_id, i.product_id,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM order_items i
    GROUP BY i.order_id, i.product_id
)
SELECT cat.name AS category, o.country_code, SUM(ir.revenue) AS revenue
FROM item_revenue ir
JOIN orders     o   ON o.order_id    = ir.order_id
JOIN products   p   ON p.product_id  = ir.product_id
JOIN categories cat ON cat.category_id = p.category_id
WHERE o.status IN ('paid','shipped','delivered')     -- exclude cancelled/returned
GROUP BY cat.name, o.country_code
ORDER BY revenue DESC;
```

Expected highlights: `Laptops | GB | 2028.99` (order 1007 — MacBook + sleeve),
`Laptops | US | 1499.00` (order 1012 — ThinkPad), `Phones | DE | 899.00` (order 1003).
Order 1004 (returned) and 1006 (cancelled) are excluded by the `WHERE` on the *inner*
side, correctly.

### Example 7 — Non-equi join (as-of / SCD2 pattern)

Join each order to the price band it falls into:

```sql
WITH value_bands(label, low, high) AS (
    VALUES ('small',  0.00,    100.00),
           ('medium', 100.00,  1000.00),
           ('large',  1000.00, 1e12)
), order_totals AS (
    SELECT o.order_id, SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS total
    FROM orders o JOIN order_items i USING (order_id)
    GROUP BY o.order_id
)
SELECT b.label, COUNT(*) AS orders, SUM(t.total) AS revenue
FROM order_totals t
JOIN value_bands b ON t.total >= b.low AND t.total < b.high
GROUP BY b.label ORDER BY b.low;
```

Expected: `large = 3` orders (1001 = 1034.98, 1007 = 2028.99, 1012 = 1499.00),
`medium = 5` (1003 = 899.00, 1005 = 134.00, 1009 = 799.00, 1011 = 237.00, 1014 = 999.00),
`small = 6` (1002, 1004, 1006, 1008, 1010, 1013 — all under 100.00). Revenue per band is the
sum of the order totals in that band; the mechanics (a theta join against a small reference
table) are the point.

`> Senior note`: `high` must be exclusive and `COALESCE(d.valid_to,'infinity')` must be
handled, or boundary rows appear in **two** bands. Overlapping bands are the range-join
equivalent of fan-out.

### Example 8 — `LATERAL` (top-2 products per category by price)

```sql
SELECT cat.name AS category, p.name AS product, p.unit_price
FROM categories cat
LEFT JOIN LATERAL (
    SELECT p.name, p.unit_price
    FROM products p
    WHERE p.category_id = cat.category_id
      AND p.is_active
    ORDER BY p.unit_price DESC
    LIMIT 2
) p ON TRUE
ORDER BY cat.category_id, p.unit_price DESC;
```

Expected: `Electronics` has no products directly → one row with `NULL` product (because we
used `LEFT JOIN LATERAL ... ON TRUE`). `Phones → iPhone 15, Galaxy S24`; `Laptops → MacBook
Air M3, ThinkPad X1`; `Cases → Laptop Sleeve, Phone Case`; `Chargers → 65W Charger, USB-C
Cable 2m`; `Kitchen → Coffee Maker, Blender` (Electric Kettle is inactive).

Without `LATERAL` you would need a window function plus a `WHERE rn <= 2` wrapper. Both are
valid; `LATERAL` usually wins when the outer side is small and the inner side is indexed.

### Example 9 — Semi vs anti vs join+distinct (three ways, three costs)

```sql
-- A) semi join: "customers with a delivered order" — no duplicates by construction
EXPLAIN ANALYZE
SELECT c.customer_id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'delivered');

-- B) DISTINCT over a join: correct but pays for a sort/hash-aggregate
SELECT DISTINCT c.customer_id
FROM customers c JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'delivered';

-- C) anti join: customers WITHOUT a delivered order
SELECT c.customer_id FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o
                  WHERE o.customer_id = c.customer_id AND o.status = 'delivered');
```

Expected: A and B return `{1,2,4,6}`; C returns `{3,5,7,8}` (Carol's orders are
paid|paid|cancelled, Erin's are paid|pending, Grace's is paid, Heidi has none). In Postgres
`EXPLAIN` shows `Hash Semi Join` / `Hash Anti Join` for A and C — a semantic hint that the
engine understood you do not want fan-out.

### Example 10 — The fan-out bug, demonstrated and fixed

```sql
-- BUG: revenue per customer, joined through order_items (fan-out on BOTH sides)
SELECT c.full_name, SUM(i.quantity * i.unit_price) AS revenue_BUGGY
FROM customers c
JOIN orders o      ON o.customer_id = c.customer_id
JOIN order_items i ON i.order_id    = o.order_id
GROUP BY c.full_name ORDER BY c.full_name;

-- FIX: aggregate at the order grain, then join to customers
SELECT c.full_name, SUM(x.order_revenue) AS revenue_FIXED
FROM customers c
JOIN (SELECT o.order_id, o.customer_id,
             SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS order_revenue
      FROM orders o JOIN order_items i USING (order_id)
      GROUP BY o.order_id, o.customer_id) x
  ON x.customer_id = c.customer_id
GROUP BY c.full_name ORDER BY c.full_name;
```

Detection recipe: compare `COUNT(*)` of the fact table to `COUNT(*)` of the joined result.

```sql
SELECT (SELECT COUNT(*) FROM order_items)                          AS items,
       (SELECT COUNT(*) FROM order_items i JOIN orders o USING (order_id)
        JOIN products p USING (product_id))                        AS after_join;
```

If `after_join > items`, a join in your chain is `N:M`.

### Example 11 — Controlling join order explicitly

```sql
-- If the planner underestimates the filter on orders, force the reduction first
WITH recent AS MATERIALIZED (
    SELECT order_id, customer_id
    FROM orders
    WHERE ordered_at >= '2024-03-01'    -- only 6 rows here; 10M in production
)
SELECT c.full_name, COUNT(*) AS orders_since_march
FROM recent r
JOIN customers c ON c.customer_id = r.customer_id
GROUP BY c.full_name
ORDER BY orders_since_march DESC, c.full_name;
```

Expected: `Carol White 2`, `Erin Davis 2`, then `Alice Johnson 1`, `Bob Smith 1`,
`Dave Brown 1`, `Grace Lee 1`, and `Frank Miller 0` / `Heidi Novak 0`. The lesson is that
`MATERIALIZED` prevented the planner from pushing the (selective and cheap) `recent` set back
into a bigger join.

---

## 6. Real-world use cases and applications

| Use case | Join pattern |
|----------|--------------|
| **Win-back / churn segments** | `LEFT JOIN orders ... WHERE o.order_id IS NULL` |
| **Funnel attribution** | semi join `events` against `orders`, then aggregate |
| **Revenue mart (star schema)** | multi-way join `fact → dim` with pre-aggregation |
| **SCD2 dimension lookup (as-of join)** | non-equi join on `valid_from/valid_to` |
| **Top-N per group** | `LATERAL` / window function |
| **Financial reconciliation (PSP vs ledger)** | `FULL OUTER JOIN` + `CASE` classification |
| **Org charts & reporting lines** | self-join (recursive CTE when depth > 1) |
| **Product recommendations ("customers who bought X also bought Y")** | self-join on `order_items` |
| **Deduplicating CDC streams** | `LEFT JOIN latest ON ... AND rn = 1` |
| **Slowly-changing route/price lookups** | `CROSS JOIN LATERAL` over a small reference table |

## 7. Performance notes and anti-patterns

**Predicate placement**
- Predicate on the *preserved* side of a `LEFT JOIN` in `WHERE` is fine (it filters left
  rows); predicate on the *nullable* side in `WHERE` converts to inner join. Put the
  condition in `ON` when you mean "match only if".
- `OR` across the join key (`ON a.x = b.x OR a.y = b.y`) prevents hash joins → nested loop
  → quadratic. UNION the two joins instead.

**Fan-out and correctness**
- Always verify the grain after joining. `COUNT(*)` before/after is a 2-second check that
  prevents weeks of wrong dashboards.
- `SELECT DISTINCT` is the smell of an accidental `N:M` join. Fix the join, do not mask it.

**Cardinality and statistics**
- Stale statistics are the top cause of catastrophic join plans. Postgres: `ANALYZE`
  (autovacuum normally handles it); Snowflake/BigQuery collect their own metadata.
- A join on a column with a **type mismatch** (`varchar` vs `int`, `text` vs `uuid`) can
  disable index usage or require a cast that kills hash partitioning. Keep FK types identical.

**Memory and spill**
- Hash joins need memory. Postgres: `work_mem` (per hash node). A below-estimate plan
  causes a *batch* hash join that spills to disk — watch for `Batches: > 1` in
  `EXPLAIN (ANALYZE)`.
- In Spark/Databricks, a `SortMergeJoin` where one side is small is a shuffle you did not
  need; consider a broadcast hint.

**Anti-pattern gallery**

| Anti-pattern | Why it hurts | Fix |
|--------------|--------------|-----|
| `LEFT JOIN ... WHERE right.col = x` | silently inner join | move to `ON`, or use inner join |
| `NOT IN (subquery)` with NULLable column | returns 0 rows on NULL | `NOT EXISTS` |
| `JOIN` then `DISTINCT` | unnecessary sort/hash of a fanned-out set | semi join (`EXISTS`) |
| `NATURAL JOIN` | breaks when columns are added | explicit `ON` |
| `CROSS JOIN` used to "unroll" rows | explodes row counts, then `LIMIT` | `LATERAL` with `LIMIT`, or `GENERATE_SERIES` |
| joining on a `CAST()`ed or `COALESCE()`d key | no index, no hash partitioning | normalize/stage the key |
| `SELECT *` in a join chain | extra I/O, ambiguity, blocks covering indexes | list columns |

**Reading the plan**

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT c.country_code, SUM(...)
FROM ... JOIN ... JOIN ...;
```

Look for: `Hash Join` vs `Nested Loop`, estimated vs actual rows (a 1000× gap means stale
stats or a correlated-predicate misestimate), `Rows Removed by Join Filter` (predicate not
pushed), and `Batches > 1` (spill).

## 8. Cross-engine differences

| Feature | PostgreSQL | MySQL 8.0 | SQL Server | Oracle | Snowflake | BigQuery |
|---------|-----------|-----------|------------|--------|-----------|----------|
| `INNER/LEFT/RIGHT` | ✅ (since forever) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `FULL OUTER JOIN` | ✅ | ❌ (emulate with `UNION ALL` of two `LEFT JOIN`s) | ✅ | ✅ | ✅ | ✅ |
| `CROSS JOIN` | ✅ | ✅ | ✅ (or comma) | ✅ | ✅ | ✅ |
| `USING (col)` | ✅ | ✅ | ❌ (use `ON`) | ✅ | ✅ | ✅ (but output keeps one column) |
| `NATURAL JOIN` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ (not supported) |
| Correlated lateral join | `LATERAL` | `LATERAL` (8.0.14+) | `CROSS/OUTER APPLY` | `LATERAL` / `OUTER APPLY` | `LATERAL` (`FLATTEN`, `... , LATERAL`) | `LEFT JOIN UNNEST(...)` / correlated `ARRAY` subqueries |
| Non-equi join | ✅ (merge/nested) | ✅ | ✅ | ✅ | ✅ | ✅ (**inner** non-equi ok; **outer** non-equi joins with `>=`/`<` were historically restricted) |
| Join hints | planner knobs (`enable_hashjoin`) | `STRAIGHT_JOIN`, optimizer hints (8.0.20+) | `INNER/LOOP/HASH/MERGE` hints | `/*+ USE_HASH */`, `LEADING` | none | none |
| `SEMI/ANTI` in `EXPLAIN` | `Hash Semi/Anti Join` | `semi join`/`antijoin` (8.0.17+) | `Left Semi Join`/`Left Anti Semi Join` | `HASH JOIN SEMI` | `Join` w/ hint | `Semi/Anti` in the execution graph |
| Default join algorithm for large–large | hash | hash (8.0.18+) | hash | hash | hash (or shuffle) | shuffle/broadcast hash |
| `TABLESAMPLE`-style join sampling | ✅ | ✅ | ✅ | ✅ | ✅ (`SAMPLE`) | ✅ (`TABLESAMPLE`) |

Notes worth knowing:

- **MySQL has no `FULL OUTER JOIN`.** The idiom is
  `SELECT ... FROM a LEFT JOIN b ... UNION SELECT ... FROM a RIGHT JOIN b ... WHERE a.k IS NULL`.
- **SQL Server uses `APPLY`**, and unlike `LATERAL` its `OUTER APPLY` needs no `ON TRUE`.
- **BigQuery's non-equi joins**: equality + inequality predicates are allowed on inner
  joins; for `LEFT JOIN`, inequality predicates in `ON` historically required care (use a
  correlated subquery or `UNNEST`). Always test on your dataset.
- **Oracle** has both the ANSI syntax and the legacy `(+)` outer-join operator. Never mix
  them in one query — mixing is an error, and `(+)` cannot express `FULL OUTER JOIN`.
- **Snowflake/BigQuery**: joins are distributed; "join order" is really *shuffle strategy*
  (broadcast vs repartition). Filtering a large table early is even more valuable there.

## 9. Exercises — Intermediate (hints only)

1. **List every customer with the number of orders they placed, including customers with
   zero orders.** *Hint:* `LEFT JOIN` + `COUNT(o.order_id)` (never `COUNT(*)`, which counts
   the null-extended row).
2. **Find products that have never been ordered.** *Hint:* anti join. Also write the
   `NOT IN` version and explain why it would be unsafe if `order_items.product_id` were
   nullable.
3. **For each order, show the customer name and the number of distinct products in it.**
   *Hint:* aggregate `order_items` first; `COUNT(DISTINCT product_id)`.
4. **Build a "customers who bought a Laptop and a Phone in different orders" list.**
   *Hint:* two semi joins (`EXISTS ... AND EXISTS ...`), not a single join (which would
   require the same order row to match twice).
5. **Show each employee with their manager's title, including the CEO.** *Hint:* self
   `LEFT JOIN`; `COALESCE` for the missing manager.
6. **Report revenue by country for delivered orders only, using a 4-table join.** *Hint:*
   `orders → order_items → products`, `orders → customers`; filter status in `WHERE`.
7. **Produce a matrix of channel × status order counts, one row per channel.** *Hint:*
   this is a preview of [`CASE` pivotting](./05-case-expressions.md); for now use one row
   per (channel, status) pair.
8. **List every category with its number of active products, including categories with
   none.** *Hint:* `LEFT JOIN` with the `is_active` filter in the `ON` clause.
9. **Find orders whose total is larger than the average order total.** *Hint:* aggregate in
   a CTE, then join back or compare against a scalar subquery.
10. **Using `LATERAL`, list the single most recent delivered order per customer.**
    *Hint:* `ORDER BY ordered_at DESC LIMIT 1` inside the lateral subquery; decide between
    `CROSS` and `LEFT` based on whether you want customers with no delivered orders.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Quantify join fan-out across the whole model.** Write a diagnostic query that, for
   each of `customers→orders`, `orders→order_items`, `products→order_items`, reports the
   source row count, the joined row count, and the max fan-out per key. Then propose the
   invariant test you would put in CI (dbt `unique` + `relationships` tests).
   *Approach:* `COUNT(*)` in a CTE vs a joined `COUNT(*)`, then group by key and take
   `MAX(cnt)`.

2. **Implement the SCD2 as-of join properly.** Given a `dim_product_price` table with
   `valid_from`, `valid_to` and `price`, rewrite "revenue at the price in effect when the
   order was placed". Then prove correctness by checking that no order matches two versions.
   *Approach:* non-equi join with a half-open interval `[valid_from, valid_to)`, plus a
   `HAVING COUNT(*) > 1` guard query.

3. **Rewrite a `N:M` join into a `1:N` join without changing the result.** Take
   `products ⋈ promotions` (a product can be in several promotions) and produce "total
   promotional discount per order" without inflating line-item sums.
   *Approach:* pre-aggregate promotions to product grain (or `AVG`/`MAX` semantics per
   business rule), then join; verify with before/after `SUM` equality on a sample.

4. **Give the three-way `FULL OUTER JOIN` reconciliation.** Sources: `warehouse_orders`,
   `psp_transactions`, `erp_invoices`, all keyed by `order_id`. Classify every order as
   matched-in-all-3, missing-in-exactly-one (name it), or amount-mismatch. Discuss how you
   would scale this to 20 sources without a 20-way `FULL JOIN`.
   *Approach:* build a `UNION ALL` "tall" fact of (source, order_id, amount) and pivot with
   conditional aggregation — the difference between a `FULL JOIN` fan and a row-oriented
   reconciliation.

5. **Diagnose and fix a bad join plan.** Produce an `EXPLAIN (ANALYZE, BUFFERS)` of
   Example 6 after removing the pre-aggregation CTE, find the node whose estimated rows
   differ from actual by >100×, and propose two independent fixes (statistics, rewrite).
   *Approach:* read the `rows=` estimates top-down, compare to `actual rows=`; correlate
   with `pg_stats` `n_distinct` / MCV lists.

6. **Implement "customers who bought X also bought Y"** with a self-join on `order_items`
   at order grain, exclude pairs within the same category, and rank the top 10 pairs.
   Then explain the complexity of the self-join and how you would bound it (min-support
   threshold, category buckets, approximate algorithms like Apriori/HashMap-based counting).
   *Approach:* `GROUP BY p1.product_id, p2.product_id` with `p1.product_id < p2.product_id`
   to deduplicate the symmetric pair; `COUNT(DISTINCT oi1.order_id)` for support.

7. **Non-equi join at scale.** Join 100M `events` to a 5k-row `campaigns` table where
   `event.occurred_at BETWEEN c.start_at AND c.end_at` and `event.properties->>'utm' = c.utm`.
   Explain why this cannot use a hash join, what a merge/interval join does, and how you
   would restructure the data (bucketing by day, `LATERAL`, or a broadcast join in Spark) to
   make it tractable.

8. **Prove the join is `1:N`, not `N:M`, with a repeatable test.** Write a query that fails
   loudly (returns a non-empty set) if any key in the join has more than one match on the
   "one" side. This is the assertion you should ship alongside every join in a data pipeline.

---

## References

**PostgreSQL**
- Joins between tables (tutorial): https://www.postgresql.org/docs/current/tutorial-join.html
- Table expressions / join syntax / `LATERAL`: https://www.postgresql.org/docs/current/queries-table-expressions.html
- `SELECT` (syntax, `USING`, join types): https://www.postgresql.org/docs/current/sql-select.html
- Planner method configuration (`enable_hashjoin`, …): https://www.postgresql.org/docs/current/runtime-config-query.html
- `CREATE INDEX`: https://www.postgresql.org/docs/current/sql-createindex.html

**MySQL**
- JOIN syntax: https://dev.mysql.com/doc/refman/8.0/en/join.html
- Lateral derived tables (8.0.14+): https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html
- Support for lateral derived tables (release blog): https://dev.mysql.com/blog-archive/support-for-lateral-derived-tables-added-to-mysql-8-0-14/

**SQL Server**
- `FROM` + JOIN + APPLY: https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql

**Oracle**
- `SELECT` (join syntax, `LATERAL`): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake JOIN: https://docs.snowflake.com/en/sql-reference/constructs/join
- BigQuery query syntax (joins, lateral/`UNNEST`): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Books, essays, and practical guides**
- Markus Winand, *The Join Operation* (Use The Index, Luke!): https://use-the-index-luke.com/sql/join
- Nested loops / the N+1 selects problem: https://use-the-index-luke.com/sql/join/nested-loops-join-n1-problem
- Lukas Eder, *Say NO to Venn diagrams when explaining JOINs*: https://blog.jooq.org/say-no-to-venn-diagrams-when-explaining-joins/
- PostgreSQL Wiki, *Don't Do This* (anti-pattern catalogue): https://wiki.postgresql.org/wiki/Don%27t_Do_This

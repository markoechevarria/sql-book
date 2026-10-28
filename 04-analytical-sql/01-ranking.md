# Ranking — ROW_NUMBER, RANK, DENSE_RANK, NTILE, PERCENT_RANK, CUME_DIST

> Every analytics dashboard, every "top N" API endpoint, every dedup job, and every
> "latest order per customer" table is a ranking problem in disguise. Ranking is the single
> most reused window-function family in production SQL, and it is also the one where
> engineers most often pick the wrong function: they reach for `ROW_NUMBER()` when they
> needed `RANK()`, silently dropping tied rows; or they use `RANK()` when they needed
> `ROW_NUMBER()`, silently emitting more rows than the `LIMIT` they promised. This chapter
> builds the mental model that makes the choice obvious, gives you the reusable templates,
> and introduces the **shared sample schema** that every chapter in this section reuses.

## TL;DR

- The six ranking functions answer **different questions**. Learn the questions, not the
  syntax:

  | Function | Question it answers | Ties |
  |----------|--------------------|------|
  | `ROW_NUMBER()` | "Give each row a distinct position." | Arbitrary unless you add a tiebreaker. Gaps none. |
  | `RANK()` | "What is this row's position, counting the rows above it?" | Tied rows share a rank; **gaps** follow (1,1,3). |
  | `DENSE_RANK()` | "How many *distinct* values are above this one?" | Tied rows share a rank; **no gaps** (1,1,2). |
  | `NTILE(n)` | "Split the partition into `n` roughly equal buckets." | Breaks ties arbitrarily; buckets fill greedily. |
  | `PERCENT_RANK()` | "What fraction of rows rank below me?" (0…1) | Uses `RANK()` semantics → gaps. |
  | `CUME_DIST()` | "What fraction of rows rank at or below me?" (0…1] | Includes all peers → never 0. |

- **Top-N per group** = `ROW_NUMBER()`/`RANK()` in a subquery (or `QUALIFY`) and filter
  `rn <= N`. Choose `RANK()` if you want ties to be included, `ROW_NUMBER()` if you want
  exactly N rows.
- **Deduplication keeping the latest row** = `ROW_NUMBER() OVER (PARTITION BY key ORDER BY
  updated_at DESC, <tiebreaker> DESC)`, then `= 1`. Always add a **deterministic
  tiebreaker** or the job is not reproducible.
- The whole family is **non-deterministic for ties by definition**; the engine is free to
  return any peer order. `RANK()`/`DENSE_RANK()`/`PERCENT_RANK()`/`CUME_DIST()` are the
  "stable" options; `ROW_NUMBER()`/`NTILE()` are the "unstable" ones.
- You **cannot filter on a window function in `WHERE`** — window functions are evaluated
  after `WHERE`/`GROUP BY`/`HAVING`. Use a CTE/subquery, or `QUALIFY` on engines that have
  it (Snowflake, BigQuery, DuckDB, Redshift, Databricks, Oracle 26ai+).
- Ranking is a **sort + scan**, not a join. It is cheap if the sort can be satisfied by an
  index whose key order is `(PARTITION BY cols..., ORDER BY cols...)`, and expensive if it
  spills to disk on a billion rows. On very large tables, restrict the partition set first
  (predicate pushdown / partition pruning) *before* the window runs.

## 1. Definitions and core concepts

### 1.1 The shared sample schema (used by all chapters in this section)

Everything you see from here on runs against this schema. It is an e-commerce + product
analytics domain — deliberately small (10 users, 6 products, 20 orders, ~60 events) so you
can trace every number by hand, but shaped exactly like the real thing.

```sql
-- ============================================================================
-- ShopStream: shared sample schema for the "Analytical SQL" section
-- Dialect: PostgreSQL 16+. Ports to other engines with the notes in §8.
-- ============================================================================

-- ---------------------------------------------------------------- users
CREATE TABLE users (
    user_id           integer PRIMARY KEY,
    email             text    NOT NULL UNIQUE,
    signup_date       date    NOT NULL,
    country           text    NOT NULL,
    plan              text    NOT NULL DEFAULT 'free',   -- free | pro | enterprise
    marketing_channel text    NOT NULL DEFAULT 'organic'
);

INSERT INTO users (user_id, email, signup_date, country, plan, marketing_channel) VALUES
  ( 1, 'alice@example.com', DATE '2024-01-05', 'US', 'free',       'organic'),
  ( 2, 'bob@example.com',   DATE '2024-01-07', 'DE', 'pro',        'paid_search'),
  ( 3, 'carol@example.com', DATE '2024-01-15', 'US', 'free',       'referral'),
  ( 4, 'dave@example.com',  DATE '2024-02-01', 'US', 'pro',        'organic'),
  ( 5, 'erin@example.com',  DATE '2024-02-14', 'FR', 'free',       'paid_social'),
  ( 6, 'frank@example.com', DATE '2024-03-02', 'US', 'free',       'organic'),
  ( 7, 'grace@example.com', DATE '2024-03-03', 'DE', 'enterprise', 'paid_search'),
  ( 8, 'heidi@example.com', DATE '2024-03-20', 'US', 'free',       'referral'),
  ( 9, 'ivan@example.com',  DATE '2024-04-01', 'US', 'pro',        'organic'),
  (10, 'judy@example.com',  DATE '2024-04-05', 'CA', 'free',       'paid_social');

-- ------------------------------------------------------------- products
CREATE TABLE products (
    product_id integer PRIMARY KEY,
    sku        text    NOT NULL UNIQUE,
    name       text    NOT NULL,
    category   text    NOT NULL,
    unit_price numeric(10,2) NOT NULL
);

INSERT INTO products (product_id, sku, name, category, unit_price) VALUES
  (101, 'SKU-101', 'Trail Runner 4',  'Shoes',       120.00),
  (102, 'SKU-102', 'City Sneaker',    'Shoes',        89.00),
  (103, 'SKU-103', 'Wool Socks',      'Apparel',      19.00),
  (104, 'SKU-104', 'Rain Jacket',     'Apparel',     210.00),
  (105, 'SKU-105', 'Water Bottle',    'Accessories',  25.00),
  (106, 'SKU-106', 'Trail Runner 5',  'Shoes',       140.00);

-- --------------------------------------------------------------- orders
CREATE TABLE orders (
    order_id bigint    PRIMARY KEY,
    user_id  integer   NOT NULL REFERENCES users(user_id),
    order_ts timestamp NOT NULL,
    status   text      NOT NULL   -- pending | paid | shipped | delivered | cancelled | refunded
);

INSERT INTO orders (order_id, user_id, order_ts, status) VALUES
  (1001,  1, TIMESTAMP '2024-01-06 10:00:00', 'delivered'),
  (1002,  2, TIMESTAMP '2024-01-08 11:30:00', 'delivered'),
  (1003,  1, TIMESTAMP '2024-01-20 09:15:00', 'delivered'),
  (1004,  3, TIMESTAMP '2024-01-22 14:00:00', 'cancelled'),
  (1005,  4, TIMESTAMP '2024-02-02 16:45:00', 'delivered'),
  (1006,  2, TIMESTAMP '2024-02-05 08:20:00', 'delivered'),
  (1007,  1, TIMESTAMP '2024-02-11 12:00:00', 'delivered'),
  (1008,  5, TIMESTAMP '2024-02-15 19:10:00', 'delivered'),
  (1009,  4, TIMESTAMP '2024-02-18 10:05:00', 'delivered'),
  (1010,  6, TIMESTAMP '2024-03-03 13:40:00', 'delivered'),
  (1011,  7, TIMESTAMP '2024-03-05 09:00:00', 'delivered'),
  (1012,  2, TIMESTAMP '2024-03-08 15:25:00', 'delivered'),
  (1013,  8, TIMESTAMP '2024-03-21 11:11:00', 'delivered'),
  (1014,  4, TIMESTAMP '2024-03-25 17:30:00', 'pending'),
  (1015,  9, TIMESTAMP '2024-04-02 08:45:00', 'delivered'),
  (1016,  1, TIMESTAMP '2024-04-06 14:20:00', 'delivered'),
  (1017,  5, TIMESTAMP '2024-04-09 10:35:00', 'delivered'),
  (1018, 10, TIMESTAMP '2024-04-12 18:05:00', 'delivered'),
  (1019,  7, TIMESTAMP '2024-04-15 09:50:00', 'delivered'),
  (1020,  2, TIMESTAMP '2024-04-20 12:15:00', 'delivered');

-- ---------------------------------------------------------- order_items
CREATE TABLE order_items (
    order_id   bigint  NOT NULL REFERENCES orders(order_id),
    line_no    integer NOT NULL,
    product_id integer NOT NULL REFERENCES products(product_id),
    quantity   integer NOT NULL CHECK (quantity > 0),
    unit_price numeric(10,2) NOT NULL,
    discount   numeric(10,2) NOT NULL DEFAULT 0,
    PRIMARY KEY (order_id, line_no)
);

INSERT INTO order_items (order_id, line_no, product_id, quantity, unit_price, discount) VALUES
  (1001, 1, 101, 1, 120.00,  0.00), (1001, 2, 105, 2,  25.00,  0.00),
  (1002, 1, 104, 1, 210.00, 20.00),
  (1003, 1, 102, 1,  89.00,  0.00),
  (1004, 1, 103, 3,  19.00,  0.00),
  (1005, 1, 106, 1, 140.00,  0.00), (1005, 2, 103, 2,  19.00,  0.00),
  (1006, 1, 101, 1, 120.00, 10.00),
  (1007, 1, 104, 1, 210.00,  0.00),
  (1008, 1, 105, 4,  25.00,  0.00),
  (1009, 1, 102, 2,  89.00,  0.00), (1009, 2, 103, 1,  19.00,  0.00),
  (1010, 1, 101, 1, 120.00,  0.00),
  (1011, 1, 106, 2, 140.00,  0.00), (1011, 2, 105, 1,  25.00,  0.00),
  (1012, 1, 103, 1,  19.00,  0.00),
  (1013, 1, 102, 1,  89.00,  5.00), (1013, 2, 105, 2,  25.00,  0.00),
  (1014, 1, 101, 1, 120.00,  0.00),
  (1015, 1, 106, 1, 140.00,  0.00), (1015, 2, 104, 1, 210.00, 15.00),
  (1016, 1, 105, 1,  25.00,  0.00),
  (1017, 1, 103, 2,  19.00,  0.00),
  (1018, 1, 101, 1, 120.00,  0.00), (1018, 2, 102, 1,  89.00,  0.00),
  (1019, 1, 104, 2, 210.00, 30.00),
  (1020, 1, 106, 1, 140.00, 20.00);

-- --------------------------------------------------------------- events
CREATE TABLE events (
    event_id   bigserial PRIMARY KEY,
    user_id    integer   NOT NULL REFERENCES users(user_id),
    session_id text,                 -- assigned by the app when available; NULL in raw exports
    event_type text      NOT NULL,   -- page_view | view_product | add_to_cart | checkout_start | purchase
    event_time timestamp NOT NULL,
    properties jsonb     NOT NULL DEFAULT '{}'::jsonb
);

INSERT INTO events (user_id, event_type, event_time) VALUES
  -- user 1 — three sessions on the same weekend, one incomplete checkout
  (1,'page_view',      TIMESTAMP '2024-01-05 10:00:00'),
  (1,'view_product',   TIMESTAMP '2024-01-05 10:02:00'),
  (1,'add_to_cart',    TIMESTAMP '2024-01-05 10:04:00'),
  (1,'checkout_start', TIMESTAMP '2024-01-05 10:06:00'),
  (1,'purchase',       TIMESTAMP '2024-01-05 10:09:00'),
  (1,'page_view',      TIMESTAMP '2024-01-06 09:00:00'),
  (1,'view_product',   TIMESTAMP '2024-01-06 09:05:00'),
  (1,'page_view',      TIMESTAMP '2024-01-06 14:00:00'),   -- 4h55m later -> new session
  (1,'view_product',   TIMESTAMP '2024-01-06 14:03:00'),
  (1,'add_to_cart',    TIMESTAMP '2024-01-06 14:05:00'),
  (1,'purchase',       TIMESTAMP '2024-01-06 14:07:00'),
  (1,'page_view',      TIMESTAMP '2024-01-12 11:00:00'),
  (1,'view_product',   TIMESTAMP '2024-01-12 11:04:00'),
  (1,'add_to_cart',    TIMESTAMP '2024-01-12 11:06:00'),
  (1,'checkout_start', TIMESTAMP '2024-01-12 11:08:00'),   -- abandons at checkout
  (1,'page_view',      TIMESTAMP '2024-01-19 08:30:00'),
  (1,'view_product',   TIMESTAMP '2024-01-19 08:33:00'),
  (1,'add_to_cart',    TIMESTAMP '2024-01-19 08:35:00'),
  (1,'checkout_start', TIMESTAMP '2024-01-19 08:36:00'),
  (1,'purchase',       TIMESTAMP '2024-01-19 08:40:00'),
  (1,'page_view',      TIMESTAMP '2024-01-26 20:15:00'),
  (1,'page_view',      TIMESTAMP '2024-02-02 12:00:00'),
  -- user 2
  (2,'page_view',      TIMESTAMP '2024-01-07 09:00:00'),
  (2,'view_product',   TIMESTAMP '2024-01-07 09:02:00'),
  (2,'add_to_cart',    TIMESTAMP '2024-01-07 09:04:00'),
  (2,'checkout_start', TIMESTAMP '2024-01-07 09:05:00'),
  (2,'purchase',       TIMESTAMP '2024-01-07 09:06:00'),
  (2,'page_view',      TIMESTAMP '2024-01-08 15:00:00'),
  (2,'page_view',      TIMESTAMP '2024-01-14 10:00:00'),
  (2,'view_product',   TIMESTAMP '2024-01-14 10:02:00'),
  (2,'page_view',      TIMESTAMP '2024-01-21 10:00:00'),
  (2,'view_product',   TIMESTAMP '2024-01-21 10:01:00'),
  (2,'add_to_cart',    TIMESTAMP '2024-01-21 10:03:00'),
  (2,'checkout_start', TIMESTAMP '2024-01-21 10:05:00'),   -- abandons at checkout
  (2,'page_view',      TIMESTAMP '2024-02-04 18:00:00'),
  -- user 3
  (3,'page_view',      TIMESTAMP '2024-01-15 12:00:00'),
  (3,'view_product',   TIMESTAMP '2024-01-15 12:03:00'),
  (3,'page_view',      TIMESTAMP '2024-01-16 12:30:00'),
  (3,'page_view',      TIMESTAMP '2024-01-22 09:00:00'),
  (3,'page_view',      TIMESTAMP '2024-02-05 09:00:00'),
  -- user 4
  (4,'page_view',      TIMESTAMP '2024-02-01 16:00:00'),
  (4,'view_product',   TIMESTAMP '2024-02-01 16:02:00'),
  (4,'add_to_cart',    TIMESTAMP '2024-02-01 16:04:00'),   -- abandons before checkout
  (4,'page_view',      TIMESTAMP '2024-02-02 16:00:00'),
  (4,'page_view',      TIMESTAMP '2024-02-09 10:00:00'),
  (4,'page_view',      TIMESTAMP '2024-02-16 10:00:00'),
  (4,'page_view',      TIMESTAMP '2024-03-01 10:00:00'),
  -- user 5
  (5,'page_view',      TIMESTAMP '2024-02-14 19:00:00'),
  (5,'view_product',   TIMESTAMP '2024-02-14 19:10:00'),
  (5,'page_view',      TIMESTAMP '2024-02-15 19:05:00'),
  (5,'page_view',      TIMESTAMP '2024-02-21 19:00:00'),
  (5,'page_view',      TIMESTAMP '2024-02-28 19:00:00'),
  -- user 6
  (6,'page_view',      TIMESTAMP '2024-03-02 13:30:00'),
  (6,'view_product',   TIMESTAMP '2024-03-02 13:40:00'),
  (6,'page_view',      TIMESTAMP '2024-03-03 13:45:00'),
  (6,'page_view',      TIMESTAMP '2024-03-09 13:50:00'),
  -- user 7 — seven consecutive-ish days, used by the streak examples
  (7,'page_view',      TIMESTAMP '2024-03-04 09:00:00'),
  (7,'page_view',      TIMESTAMP '2024-03-05 09:00:00'),
  (7,'page_view',      TIMESTAMP '2024-03-06 09:00:00'),
  (7,'page_view',      TIMESTAMP '2024-03-07 09:00:00'),
  (7,'page_view',      TIMESTAMP '2024-03-09 09:00:00'),   -- gap: no event on 03-08
  (7,'page_view',      TIMESTAMP '2024-03-10 09:00:00'),
  (7,'page_view',      TIMESTAMP '2024-03-11 09:00:00');

-- --------------------------------------------- materialized sessions
-- (what the sessionization query in 07-sessionization.md produces and writes back)
CREATE TABLE sessions (
    session_id    bigserial PRIMARY KEY,
    user_id       integer   NOT NULL REFERENCES users(user_id),
    session_start timestamp NOT NULL,
    session_end   timestamp NOT NULL,
    event_count   integer   NOT NULL
);

-- ------------------------------------ semi-additive balance snapshot
-- (used by 02-running-totals.md for the "semi-additive measure" discussion)
CREATE TABLE subscriptions (
    subscription_id bigint PRIMARY KEY,
    user_id         integer NOT NULL REFERENCES users(user_id),
    plan            text    NOT NULL,
    mrr             numeric(10,2) NOT NULL,
    started_on      date    NOT NULL,
    ended_on        date
);

-- ------------------------- canonical revenue view used everywhere below
CREATE VIEW order_revenue AS
SELECT o.order_id,
       o.user_id,
       o.order_ts::date                                   AS order_date,
       o.order_ts,
       o.status,
       SUM(li.quantity * li.unit_price - li.discount)     AS revenue
FROM   orders o
JOIN   order_items li USING (order_id)
GROUP  BY o.order_id, o.user_id, o.order_ts, o.status;
```

A convenience table for the time-series chapters (`02`–`03`), materialized from the view
above so both chapters show identical numbers:

```sql
-- Daily recognized revenue. Cancelled orders excluded; a day with no
-- qualifying order simply has no row — that gap is what makes the
-- ROWS-vs-RANGE discussion in 03-moving-averages.md concrete.
CREATE TABLE daily_revenue (revenue_date date PRIMARY KEY, revenue numeric(12,2) NOT NULL);

INSERT INTO daily_revenue (revenue_date, revenue) VALUES
  (DATE '2024-01-06', 170.00),
  (DATE '2024-01-08', 190.00),
  (DATE '2024-01-20',  89.00),
  (DATE '2024-02-02', 178.00),
  (DATE '2024-02-05', 110.00),
  (DATE '2024-02-11', 210.00),
  (DATE '2024-02-15', 100.00),
  (DATE '2024-02-18', 197.00),
  (DATE '2024-03-03', 120.00),
  (DATE '2024-03-05', 305.00),
  (DATE '2024-03-08',  19.00),
  (DATE '2024-03-21', 134.00),
  (DATE '2024-03-25', 120.00),
  (DATE '2024-04-02', 335.00),
  (DATE '2024-04-06',  25.00),
  (DATE '2024-04-09',  38.00),
  (DATE '2024-04-12', 209.00),
  (DATE '2024-04-15', 390.00),
  (DATE '2024-04-20', 120.00);
```

Sanity checks you can run to confirm your load is correct:

```sql
SELECT SUM(revenue) FROM daily_revenue;   -- 3059.00
SELECT COUNT(*) FROM users;               -- 10
SELECT COUNT(*) FROM events;              -- 63
SELECT SUM(revenue) FROM order_revenue WHERE status <> 'cancelled';  -- 3059.00
```

### 1.2 Vocabulary

| Term | Meaning |
|------|---------|
| **Window** | The subset of rows a window function "sees" for the current row: `PARTITION BY` → `ORDER BY` → frame. |
| **Partition** | The peer group the window is scoped to. Ranking **restarts at 1 in each partition**. |
| **Peer** | A row that compares *equal* to the current row under the window's `ORDER BY`. Ties are peer groups. |
| **Frame** | For ranking functions the frame is *ignored* — they always see the whole partition. Frames matter for aggregates (chapters 02–03). |
| **Rank gap** | A missing rank value caused by a tie under `RANK()` (e.g. `1, 1, 3`). |
| **Deterministic tie-break** | Adding a unique column to the window `ORDER BY` so tie ordering is reproducible. |

> **Senior note — the four "stable" and two "unstable" functions.** ANSI SQL requires
> `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()` and `CUME_DIST()` to give **the same answer to
> every row in a peer group**. It says nothing about *which* peer gets which position for
> `ROW_NUMBER()` or which bucket for `NTILE()`. In practice this means: if you backfill a
> table with `ROW_NUMBER()` and there are ties, two runs can produce **different row
> assignments** and your "idempotent" pipeline is not idempotent. Add the primary key to the
> `ORDER BY`. This is the single most common ranking bug in data engineering.

## 2. Why it matters / what problem it solves

Ranking solves four recurring production problems, and each one has a different "correct"
function:

**(a) Top-N per group.** "The three best-selling products per category." "The latest order
per customer." "The highest-paid employee per department." Naive approaches (correlated
subqueries per group, or `GROUP BY` + `MAX` joined back) are either O(groups × rows) or
produce wrong results when the tie-break key is not unique. One window function solves it.

**(b) Deduplication / "latest row wins".** Analytics tables get appended to by at-least-once
pipelines. You need exactly one row per business key, and it must be the newest. That is
`ROW_NUMBER() = 1`.

**(c) Distribution and bucketing.** Quartiles, deciles, percentiles, "which decile of
customers is this one in?" — `NTILE`, `PERCENT_RANK`, `CUME_DIST`.

**(d) Data-quality detection.** "Which invoices have a duplicate `(customer, invoice_no)`?"
"Is this sequence monotonically increasing?" "Are there two active subscriptions per user?"
Ranking exposes these as values, not as errors.

The cost of getting it wrong is not academic. A dedup job that keeps the *oldest* row
because the tiebreaker was missing serves stale prices to customers. A "top 10 customers"
dashboard that uses `RANK()` shows 12 rows when there is a 3-way tie for 10th and the
stakeholder says the chart is broken. A `ROW_NUMBER()`-based session ID that renumbers on
every run breaks downstream incremental models.

## 3. The pattern / technique explained

### 3.1 Anatomy of the `OVER` clause

```sql
<ranking_function>() OVER (
    [ PARTITION BY <expr> [, ...] ]   -- reset the ranking per group
    [ ORDER BY     <expr> [ASC|DESC] [NULLS FIRST|LAST] [, ...] ]  -- defines "better"
)
```

- **`PARTITION BY`** splits the result set into independent groups. Ranking restarts at 1 in
  each. Think of `PARTITION BY` as "the group I want a top-N *within*".
- **`ORDER BY`** defines the ranking direction. `DESC` means "rank 1 = biggest".
- **`NULLS FIRST / NULLS LAST`** is dialect-dependent by default (PostgreSQL: `NULLS LAST`
  for `ASC`, `NULLS FIRST` for `DESC`; MySQL and SQL Server treat `NULL` as lowest, i.e.
  first under `ASC`). **Always state it explicitly** when NULLs are possible, otherwise the
  same query produces different rankings on different engines.
- Ranking functions ignore `ROWS`/`RANGE` frames. Writing
  `RANK() OVER (ORDER BY x ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)` is legal in some
  engines and a syntax error in others; it never changes the answer.

### 3.2 Evaluation order (why `WHERE` cannot see a rank)

```
FROM/JOIN → WHERE → GROUP BY → HAVING → WINDOW FUNCTIONS → SELECT → DISTINCT → ORDER BY → LIMIT
```

Window functions are evaluated **after** `HAVING` and **before** `DISTINCT`/`ORDER BY`/
`LIMIT`. Therefore:

```sql
-- ❌ ERROR: window functions are not allowed in WHERE
SELECT user_id, ROW_NUMBER() OVER (ORDER BY signup_date) AS rn
FROM   users
WHERE  rn <= 5;
```

Three legal spellings:

```sql
-- ✅ 1. subquery / CTE (portable everywhere)
WITH ranked AS (
  SELECT user_id, signup_date, ROW_NUMBER() OVER (ORDER BY signup_date) AS rn
  FROM   users
)
SELECT * FROM ranked WHERE rn <= 5;

-- ✅ 2. QUALIFY (Snowflake, BigQuery, DuckDB, Redshift, Databricks, Oracle 26ai+)
SELECT user_id, signup_date
FROM   users
QUALIFY ROW_NUMBER() OVER (ORDER BY signup_date) <= 5;

-- ✅ 3. LATERAL + LIMIT (PostgreSQL / MySQL 8.0.14+; the "top-N per group" workhorse)
SELECT u.user_id, o.order_id, o.order_ts
FROM   users u
CROSS  JOIN LATERAL (
    SELECT order_id, order_ts FROM orders
    WHERE user_id = u.user_id ORDER BY order_ts DESC LIMIT 3
) o;
```

### 3.3 Choosing the function: a decision tree

```
Do you need exactly N rows, with ties broken arbitrarily?
├── yes → ROW_NUMBER()
└── no
    ├── Do ties need the same number AND compact ranks (1,1,2)?   → DENSE_RANK()
    ├── Do ties need the same number AND competition ranks (1,1,3)? → RANK()
    ├── Do you need equal-sized buckets (quartiles/deciles)?      → NTILE(n)
    └── Do you need a 0–1 score, not an integer position?
        ├── "fraction of rows below me"      → PERCENT_RANK()
        └── "fraction of rows at or below me" → CUME_DIST()
```

`NTILE(n)` ≠ percentile. `NTILE(100)` splits the partition into 100 buckets of size
`⌈rows/100⌉` or `⌊rows/100⌋`; with 10 rows it produces 10 non-empty buckets numbered 1–10,
which looks nothing like a percentile. `PERCENT_RANK()` is the 0–1 score you probably meant.

### 3.4 `ROW_NUMBER()` and ties: how engines break them

`ROW_NUMBER()` must assign distinct numbers, so the engine needs a total order. If
`ORDER BY` produces peers, the *relative* order within the peer group is
**implementation-defined** and may vary between executions (a parallel hash-scan can
re-order peers). Practical consequences:

1. Any dedup/keep-latest query **must** include a unique tiebreaker.
2. Any "top N" that must be stable across runs **must** include a unique tiebreaker.
3. Sorting on a non-unique column and `LIMIT 1` gives you *a* row, not *the* row. If your
   business rule is "the most recent", "most recent" must be a **unique** ordering, and
   often a monotonic `updated_at` is not enough — two micro-batch writes can share a
   timestamp. Add the surrogate key or a sequence/`ingested_at`.

## 4. Step-by-step worked example (with sample data and expected output tables)

### 4.1 The input

Per-user order counts, computed from `orders`:

```sql
SELECT u.user_id, u.email, COUNT(o.order_id) AS order_count
FROM   users u
LEFT JOIN orders o ON o.user_id = u.user_id
GROUP  BY u.user_id, u.email;
```

| user_id | email | order_count |
|--------:|-------|------------:|
| 1 | alice@example.com | 4 |
| 2 | bob@example.com   | 4 |
| 3 | carol@example.com | 1 |
| 4 | dave@example.com  | 3 |
| 5 | erin@example.com  | 2 |
| 6 | frank@example.com | 1 |
| 7 | grace@example.com | 2 |
| 8 | heidi@example.com | 1 |
| 9 | ivan@example.com  | 1 |
| 10| judy@example.com  | 1 |

There are plenty of ties (four users with 1 order, two with 4, two with 2). That is exactly
what we want to see the functions diverge.

### 4.2 All six functions side by side

```sql
WITH per_user AS (
    SELECT u.user_id, u.email, COUNT(o.order_id) AS order_count
    FROM   users u
    LEFT JOIN orders o ON o.user_id = u.user_id
    GROUP  BY u.user_id, u.email
)
SELECT user_id,
       order_count,
       ROW_NUMBER()    OVER w AS rn,
       RANK()          OVER w AS rnk,
       DENSE_RANK()    OVER w AS dense_rnk,
       NTILE(4)        OVER w AS quartile,
       ROUND(PERCENT_RANK() OVER w, 4) AS pct_rank,
       ROUND(CUME_DIST()    OVER w, 4) AS cume_dist
FROM   per_user
WINDOW w AS (ORDER BY order_count DESC, user_id)   -- ← deterministic tiebreaker!
ORDER  BY order_count DESC, user_id;
```

| user_id | order_count | rn | rnk | dense_rnk | quartile | pct_rank | cume_dist |
|--------:|------------:|---:|----:|----------:|---------:|---------:|----------:|
| 1 | 4 | 1 | 1 | 1 | 1 | 0.0000 | 0.2 |
| 2 | 4 | 2 | 1 | 1 | 1 | 0.0000 | 0.2 |
| 4 | 3 | 3 | 3 | 2 | 1 | 0.2222 | 0.3 |
| 5 | 2 | 4 | 4 | 3 | 2 | 0.3333 | 0.5 |
| 7 | 2 | 5 | 4 | 3 | 2 | 0.3333 | 0.5 |
| 3 | 1 | 6 | 6 | 4 | 2 | 0.5556 | 1.0 |
| 6 | 1 | 7 | 6 | 4 | 3 | 0.5556 | 1.0 |
| 8 | 1 | 8 | 6 | 4 | 3 | 0.5556 | 1.0 |
| 9 | 1 | 9 | 6 | 4 | 4 | 0.5556 | 1.0 |
|10 | 1 |10 | 6 | 4 | 4 | 0.5556 | 1.0 |

Read this table carefully — it contains the whole chapter:

- **`rn`** gives 10 distinct numbers. Users 1 and 2 tie on `order_count`; `user_id` broke
  the tie. Without the `user_id` term, the 1-versus-2 assignment would be arbitrary.
- **`rnk`** gives 1, 1, 3, 6 — **competition ranking** with gaps. Four users share rank 6.
- **`dense_rnk`** gives 1, 1, 2, 3, 4 — **no gaps**. Note that `max(dense_rnk) = 4`, the
  number of *distinct* values, whereas `max(rnk) = 10`.
- **`quartile`** = `NTILE(4)`. 10 rows / 4 buckets → sizes 3, 3, 2, 2 (greedy fill:
  the first `10 mod 4 = 2` buckets get the extra row). Ties are split across buckets; there
  is no "bucket boundary" guarantee that equal values land together.
- **`pct_rank`** = `(RANK() - 1) / (n - 1)` = `(rnk - 1) / 9`. Users 3–10 all have `rnk = 6`,
  hence `5/9 = 0.5556`. **No row has `pct_rank = 1`** when there are ties at the bottom;
  that is correct behaviour, not a bug.
- **`cume_dist`** = `(# rows ≤ current in window order) / n`. The two leaders share `0.2`;
  the four tail users share `1.0`. `cume_dist` never equals 0.

> **Senior note.** `CUME_DIST()` is the correct function for "what fraction of customers
> spend **at most** as much as this one?" — that is the definition used in percentile
> *plots* and in "top 10% of customers account for X% of revenue" statements.
> `PERCENT_RANK()` answers "what fraction spend strictly less?". If two customers tie at the
> mean, `CUME_DIST` includes both, `PERCENT_RANK` excludes both peers. Pick deliberately.

### 4.3 Pattern A — Top-N per group

**Problem statement.** For each user, return their 2 largest orders by revenue.

**Naive/incorrect approach #1 — `GROUP BY` + `MAX`.** It returns the top *amount*, not the
top *rows*, and it cannot return a second row per group:

```sql
-- ❌ only gives the max order per user; the order_id is not even well-defined
SELECT user_id, MAX(revenue) FROM order_revenue GROUP BY user_id;
```

**Naive/incorrect approach #2 — global `LIMIT`.** `ORDER BY revenue DESC LIMIT 20` gives the
global top 20, not the top 2 *per user*:

```sql
-- ❌ global top-N, silently returns 20 orders for the highest-spending user
SELECT user_id, order_id, revenue
FROM   order_revenue ORDER BY revenue DESC LIMIT 20;
```

**Naive/incorrect approach #3 — correlated subquery per row.** Correct but O(rows × groups):

```sql
-- ✅ but a nested loop over every row; dies on a fact table
SELECT o.* FROM order_revenue o
WHERE (SELECT COUNT(*) FROM order_revenue o2
       WHERE o2.user_id = o.user_id AND o2.revenue > o.revenue) < 2;
```

**Correct approach — window function + filter.** The reusable shape is
*compute a rank in a CTE, filter the rank in the outer query*:

```sql
WITH ranked AS (
    SELECT user_id, order_id, order_ts, revenue,
           ROW_NUMBER() OVER (PARTITION BY user_id
                              ORDER BY revenue DESC, order_id DESC) AS rn
    FROM   order_revenue
    WHERE  status <> 'cancelled'
)
SELECT user_id, order_id, order_ts::date AS order_date, revenue, rn
FROM   ranked
WHERE  rn <= 2
ORDER  BY user_id, rn;
```

Expected output:

| user_id | order_id | order_date | revenue | rn |
|--------:|---------:|-----------|--------:|---:|
| 1 | 1007 | 2024-02-11 | 210.00 | 1 |
| 1 | 1001 | 2024-01-06 | 170.00 | 2 |
| 2 | 1002 | 2024-01-08 | 190.00 | 1 |
| 2 | 1020 | 2024-04-20 | 120.00 | 2 |
| 4 | 1009 | 2024-02-18 | 197.00 | 1 |
| 4 | 1005 | 2024-02-02 | 178.00 | 2 |
| 5 | 1008 | 2024-02-15 | 100.00 | 1 |
| 5 | 1017 | 2024-04-09 |  38.00 | 2 |
| 6 | 1010 | 2024-03-03 | 120.00 | 1 |
| 7 | 1019 | 2024-04-15 | 390.00 | 1 |
| 7 | 1011 | 2024-03-05 | 305.00 | 2 |
| 8 | 1013 | 2024-03-21 | 134.00 | 1 |
| 9 | 1015 | 2024-04-02 | 335.00 | 1 |
|10 | 1018 | 2024-04-12 | 209.00 | 1 |

`ROW_NUMBER()` is right here because the business rule is "at most 2 rows per user" (a
dashboard panel with fixed height). If the rule were "all orders within the top-2 revenue
levels", you would use `RANK() <= 2` and could legitimately get 3+ rows.

**Reasoning.** The window function computes a per-partition counter with one sort of the
(already filtered) result set; the outer filter is a cheap streaming predicate. There is
**no join** and **no per-group rescans**. This is why the window version scales where the
correlated subquery does not.

### 4.4 Pattern B — Top-N with ties (`RANK` vs `ROW_NUMBER`)

**Problem statement.** "Show the top 5 customers by revenue, and if there is a tie for 5th,
show all tied customers." This is exactly what `FETCH FIRST 5 ROWS WITH TIES` does, and
`RANK() <= 5` reproduces it portably.

Let us make a tie deliberately. Users 1 and 4 spend 494.00 and 495.00 — change one order to
create the tie and the difference becomes visible:

```sql
WITH per_user AS (
    SELECT user_id, SUM(revenue) AS revenue
    FROM   order_revenue WHERE status <> 'cancelled'
    GROUP  BY user_id
)
SELECT user_id, revenue,
       ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rn,
       RANK()       OVER (ORDER BY revenue DESC) AS rnk
FROM   per_user;
```

| user_id | revenue | rn | rnk |
|--------:|--------:|---:|----:|
| 7 | 695.00 | 1 | 1 |
| 4 | 495.00 | 2 | 2 |
| 1 | 494.00 | 3 | 3 |
| 2 | 439.00 | 4 | 4 |
| 9 | 335.00 | 5 | 5 |
|10 | 209.00 | 6 | 6 |
| 5 | 138.00 | 7 | 7 |
| 8 | 134.00 | 8 | 8 |
| 6 | 120.00 | 9 | 9 |

Now `ROW_NUMBER() ... <= 5` returns exactly 5 rows; `RANK() ... <= 5` returns 5 rows too,
*until a tie appears at the boundary*. The rule to internalize:

| Business rule | Function | Result size |
|---------------|----------|-------------|
| "Show at most N rows" | `ROW_NUMBER() <= N` | exactly ≤ N |
| "Show everyone in the top N positions" | `RANK() <= N` | ≥ N when ties |
| "Show everyone whose value is among the top N distinct values" | `DENSE_RANK() <= N` | ≥ N |

### 4.5 Pattern C — Deduplication keeping the latest row

**Problem statement.** A CDC/upsert feed writes a full snapshot per change. You need one
row per `user_id`: the current one (highest `updated_at`).

Sample data (note the **duplicate timestamp** for user 7 — the classic trap):

| user_id | updated_at | plan | country |
|--------:|------------|------|---------|
| 1 | 2024-01-05 08:00 | free | US |
| 1 | 2024-02-10 09:30 | pro | US |
| 1 | 2024-03-01 07:00 | enterprise | US |
| 2 | 2024-01-07 10:00 | free | DE |
| 2 | 2024-01-20 10:00 | pro | DE |
| 5 | 2024-02-14 11:00 | free | FR |
| 7 | 2024-03-03 12:00 | free | DE |
| 7 | 2024-03-03 12:00 | enterprise | DE |  ← same timestamp, different plan!

**Naive/incorrect approach #1 — `DISTINCT`.** Wrong: it keeps *all* distinct rows, not the
latest one per key.

```sql
-- ❌ three rows for user 1 and two for user 7
SELECT DISTINCT user_id, plan, country FROM user_profile_cdc;
```

**Naive/incorrect approach #2 — `GROUP BY` + `MAX`.** If you also select `plan`, the engine
must pick *some* value, and standard SQL says that is illegal / undefined:

```sql
-- ❌ error in strict engines; silently arbitrary in permissive ones
SELECT user_id, MAX(updated_at), plan FROM user_profile_cdc GROUP BY user_id;
```

**Naive/incorrect approach #3 — `MAX(updated_at)` join without a tiebreaker.** For user 7
this returns **two rows**, because both rows share the max timestamp:

```sql
-- ❌ "latest row" is not unique
SELECT c.* FROM user_profile_cdc c
WHERE c.updated_at = (SELECT MAX(updated_at) FROM user_profile_cdc c2
                      WHERE c2.user_id = c.user_id);
```

**Correct approach — `ROW_NUMBER()` with a full deterministic ordering:**

```sql
WITH ranked AS (
    SELECT user_id, updated_at, plan, country,
           ROW_NUMBER() OVER (PARTITION BY user_id
                              ORDER BY updated_at DESC, plan DESC) AS rn
    FROM   user_profile_cdc
)
SELECT user_id, updated_at, plan, country FROM ranked WHERE rn = 1;
```

| user_id | updated_at | plan | country |
|--------:|------------|------|---------|
| 1 | 2024-03-01 07:00 | enterprise | US |
| 2 | 2024-01-20 10:00 | pro | DE |
| 5 | 2024-02-14 11:00 | free | FR |
| 7 | 2024-03-03 12:00 | enterprise | DE |

You must choose a **second sort term** whose meaning you can defend. Options, best first:

1. A monotonic ingestion sequence (`ingested_at`, `ingest_seq`, Kafka offset) — the only
   truly correct answer for "which write landed last".
2. A business preference encoded in data (`plan` rank: enterprise > pro > free) — you are
   now *defining* the tie rule, which is fine as long as it is explicit and documented.
3. The natural key (`user_id`) — deterministic, arbitrary, and honest: it says "I don't know
   which is newer, so I picked one, reproducibly."

> **Senior note.** The "one row per key, latest wins" pattern is the heart of almost every
> dimension-table build (`dbt` models named `dim_users__current`), every Kafka-to-warehouse
> landing job, and every "snapshot → current state" materialization. If you get the
> tiebreaker wrong, the failure mode is *nondeterministic row counts across backfills*, which
> is very hard to debug because the wrong query usually looks right.

**PostgreSQL shortcut:** `DISTINCT ON` does the same thing with less ceremony, but the
`ORDER BY` **must** start with the `DISTINCT ON` columns:

```sql
SELECT DISTINCT ON (user_id) user_id, updated_at, plan, country
FROM   user_profile_cdc
ORDER  BY user_id, updated_at DESC, plan DESC;
```

MySQL, SQL Server, Oracle, Snowflake and BigQuery have no `DISTINCT ON`; the
`ROW_NUMBER() = 1` form is the portable one.

### 4.6 Pattern D — Top-N per category, with the "remaining" bucket

Dashboards usually want "top 3 categories by revenue" as a stable chart. `RANK()` produces
variable row counts when ties exist, `ROW_NUMBER()` produces a stable count:

```sql
WITH cat AS (
    SELECT p.category, SUM(r.revenue) AS revenue
    FROM   order_revenue r JOIN order_items li USING (order_id)
                           JOIN products p ON p.product_id = li.product_id
    WHERE  r.status <> 'cancelled'
    GROUP  BY p.category
)
SELECT category, revenue,
       ROW_NUMBER() OVER (ORDER BY revenue DESC) AS rn,
       ROUND(100.0 * revenue / SUM(revenue) OVER (), 2) AS pct_of_total
FROM   cat
ORDER  BY rn;
```

| category | revenue | rn | pct_of_total |
|----------|--------:|---:|-------------:|
| Shoes | 1710.00 | 1 | 55.90 |
| Apparel | 1099.00 | 2 | 35.93 |
| Accessories | 250.00 | 3 | 8.17 |

`SUM(revenue) OVER ()` (an empty `OVER`) is an aggregate window over the whole result set —
useful for share-of-total columns without a self-join. Verify against §4.4: `SUM` over
categories must equal the grand total `3059.00` (the small rounding differences in
`pct_of_total` are display rounding).

### 4.7 Pattern E — Detecting duplicates (data quality)

```sql
-- "Which (user_id, order_ts) pairs appear more than once?"
SELECT user_id, order_ts, COUNT(*) AS copies,
       ROW_NUMBER() OVER (PARTITION BY user_id, order_ts ORDER BY order_id) AS copy_no
FROM   orders
GROUP  BY user_id, order_ts      -- (this collapses duplicates; see below)
```

That version collapses. The real check needs the duplicates *individually investigated*, so
count in one CTE and join back:

```sql
WITH dup_keys AS (
    SELECT user_id, order_ts
    FROM   orders
    GROUP  BY user_id, order_ts
    HAVING COUNT(*) > 1
)
SELECT o.*
FROM   orders o
JOIN   dup_keys d USING (user_id, order_ts)
ORDER  BY o.user_id, o.order_ts, o.order_id;
```

The same idea for events, which is where duplicates actually hurt: at-least-once producers
re-emit the same `(user_id, event_type, event_time)`:

```sql
SELECT user_id, event_type, event_time,
       COUNT(*)                                       AS copies,
       ROW_NUMBER() OVER (PARTITION BY user_id, event_type, event_time
                          ORDER BY event_id)          AS copy_no
FROM   events
GROUP  BY user_id, event_type, event_time
HAVING COUNT(*) > 1;
```

Expected (empty on the clean sample — but if you re-run the `INSERT INTO events` you will
get exactly this, which is the point):

| user_id | event_type | event_time | copies |
|--------:|-----------|------------|-------:|
| (no rows on a clean load) |

### 4.8 Pattern F — Bucketing customers into deciles (and doing it *right*)

**Problem statement.** Assign every customer to a spend decile for a marketing campaign.
The buckets must be roughly equal in size and the assignment must be stable.

**Incorrect approach — integer division on revenue.** Buckets are wildly unequal and the
boundary is arbitrary:

```sql
-- ❌ decile boundaries depend on the currency scale; buckets are not equal-sized
SELECT user_id, revenue, (revenue / 50)::int AS bogus_decile FROM per_user;
```

**Incorrect approach — `NTILE(100)` on 10 rows.** You get 10 buckets of 1 row each; it looks
like a ranking, not a decile:

```sql
-- ❌ with 10 rows this is just ROW_NUMBER()
SELECT user_id, NTILE(100) OVER (ORDER BY revenue DESC) FROM per_user;
```

**Correct approach — `NTILE(n)` with n = the number of buckets you actually want, on a
population large enough to fill them, plus an explicit tiebreaker:**

```sql
WITH per_user AS (
    SELECT user_id, SUM(revenue) AS revenue
    FROM   order_revenue WHERE status <> 'cancelled'
    GROUP  BY user_id
)
SELECT user_id, revenue,
       NTILE(4)         OVER (ORDER BY revenue DESC, user_id) AS quartile,
       ROW_NUMBER()     OVER (ORDER BY revenue DESC, user_id) AS rn
FROM   per_user
ORDER  BY rn;
```

| user_id | revenue | quartile |
|--------:|--------:|---------:|
| 7 | 695.00 | 1 |
| 4 | 495.00 | 1 |
| 1 | 494.00 | 1 |
| 2 | 439.00 | 2 |
| 9 | 335.00 | 2 |
|10 | 209.00 | 3 |
| 5 | 138.00 | 3 |
| 8 | 134.00 | 4 |
| 6 | 120.00 | 4 |

(Nine rows, `NTILE(4)` → bucket sizes 3, 2, 2, 2: `9 = 4·2 + 1`, so the first bucket gets the
extra row.)

**Reasoning.** `NTILE` guarantees *equal-sized* buckets, not *equal-valued* ones. That is
usually what you want for a campaign split (you want 25% of customers in each cell). If you
want equal *value* ranges (e.g. spend bands), use `CASE` with explicit thresholds or
`WIDTH_BUCKET` (PostgreSQL / Oracle), **not** `NTILE`.

### 4.9 Pattern G — "Rank within the current period" (dashboards)

**Problem statement.** A monthly leaderboard that must **restart every month**:

```sql
WITH monthly AS (
    SELECT user_id,
           date_trunc('month', order_ts)::date AS month,
           SUM(revenue)                        AS revenue
    FROM   order_revenue WHERE status <> 'cancelled'
    GROUP  BY user_id, date_trunc('month', order_ts)
)
SELECT month, user_id, revenue,
       RANK() OVER (PARTITION BY month ORDER BY revenue DESC) AS rank_in_month
FROM   monthly
ORDER  BY month, rank_in_month, user_id;
```

Run it and paste the output — never reason about it in a slide. The actual result is:

| month | user_id | revenue | rank_in_month |
|-------|--------:|--------:|--------------:|
| 2024-01-01 | 1 | 259.00 | 1 |
| 2024-01-01 | 2 | 190.00 | 2 |
| 2024-02-01 | 4 | 375.00 | 1 |
| 2024-02-01 | 1 | 210.00 | 2 |
| 2024-02-01 | 2 | 110.00 | 3 |
| 2024-02-01 | 5 | 100.00 | 4 |
| 2024-03-01 | 7 | 305.00 | 1 |
| 2024-03-01 | 8 | 134.00 | 2 |
| 2024-03-01 | 4 | 120.00 | 3 |
| 2024-03-01 | 6 | 120.00 | 3 |
| 2024-03-01 | 2 |  19.00 | 5 |
| 2024-04-01 | 7 | 390.00 | 1 |
| 2024-04-01 | 9 | 335.00 | 2 |
| 2024-04-01 |10 | 209.00 | 3 |
| 2024-04-01 | 2 | 120.00 | 4 |
| 2024-04-01 | 5 |  38.00 | 5 |
| 2024-04-01 | 1 |  25.00 | 6 |

March shows a genuine tie (users 4 and 6 at 120.00) → both get rank 3 and the next rank in
that partition jumps to **5**. That is correct competition ranking with a gap, and it is
exactly the kind of thing that makes a leaderboard show two "#3" rows and no "#4". Notice
also that March has 5 rows, February 4, and April 6 — the partition sizes differ, which is
fine because `PARTITION BY` resets the window, but it means you must not compare raw ranks
across months.

### 4.10 Pattern H — "Latest non-null value" (SCD-style carry-forward)

**Problem statement.** A dimension receives full snapshots where a column can be `NULL`
meaning "no change". You need to carry the last *known* value forward. The trick:
`COUNT(col) OVER (ORDER BY ...)` counts only non-null values, so it acts as a "group id" for
the segments of known values.

```sql
WITH s AS (
    SELECT user_id, updated_at, plan,
           COUNT(plan) OVER (PARTITION BY user_id ORDER BY updated_at) AS grp
    FROM   user_profile_cdc
)
SELECT user_id, updated_at,
       FIRST_VALUE(plan) OVER (PARTITION BY user_id, grp
                               ORDER BY updated_at
                               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS plan_filled
FROM   s
ORDER  BY user_id, updated_at;
```

This is `gap-and-islands` in miniature (see `08-gap-and-islands.md`), and the same
`COUNT(x) OVER (...)` idiom is the reusable trick for "fill down".

## 5. Reusable template

### 5.1 Top-N per group (portable)

```sql
WITH ranked AS (
    SELECT <partition_cols>,
           <payload_cols>,
           ROW_NUMBER() OVER (
               PARTITION BY <partition_cols>
               ORDER BY     <metric> DESC,
                            <unique_tiebreaker> DESC     -- ← never omit this
           ) AS rn
    FROM   <source>
    WHERE  <pre-filter>                                   -- ← push filters BEFORE the window
)
SELECT *
FROM   ranked
WHERE  rn <= :n
ORDER  BY <partition_cols>, rn;
```

### 5.2 "Latest row per key" (idempotent dedup)

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY <business_key>
               ORDER BY     <event_time> DESC,
                            <ingest_sequence> DESC        -- monotonic surrogate
           ) AS rn
    FROM   <append_only_table>
)
SELECT *
FROM   ranked
WHERE  rn = 1;
```

### 5.3 "Detect duplicates" (data quality gate)

```sql
SELECT <business_key>, COUNT(*) AS copies,
       MIN(<ingest_sequence>) AS first_seen_seq,
       MAX(<ingest_sequence>) AS last_seen_seq
FROM   <table>
GROUP  BY <business_key>
HAVING COUNT(*) > 1
ORDER  BY copies DESC;
```

### 5.4 "Decile / quartile assignment"

```sql
WITH metrics AS ( SELECT <entity_id>, <metric> FROM <source> GROUP BY 1, 2 )
SELECT entity_id, metric,
       NTILE(:buckets) OVER (ORDER BY metric DESC, entity_id) AS bucket
FROM   metrics;
```

### 5.5 Cheat sheet: which function for which job

| Job | Function | Filter |
|-----|----------|--------|
| Latest row per key | `ROW_NUMBER()` | `= 1` |
| At most N rows per group | `ROW_NUMBER()` | `<= N` |
| Everyone in the top N positions | `RANK()` | `<= N` |
| Everyone with a top-N distinct value | `DENSE_RANK()` | `<= N` |
| N equal-sized buckets | `NTILE(N)` | assign as a column |
| 0–1 percentile score, exclusive | `PERCENT_RANK()` | `< 0.9` |
| 0–1 percentile score, inclusive | `CUME_DIST()` | `<= 0.9` |
| Nth row of a group | `NTH_VALUE(x, N)` | — |
| First/last by some order | `FIRST_VALUE` / `LAST_VALUE` | add `ROWS BETWEEN ... AND UNBOUNDED FOLLOWING` for `LAST_VALUE` |
| Single non-null value per group | `MAX(x) OVER (PARTITION BY g)` | — |

> **Trap — `LAST_VALUE` without a frame.** `LAST_VALUE(x) OVER (ORDER BY t)` returns
> `x` **of the current row**, because the default frame ends at `CURRENT ROW`. You almost
> always want
> `LAST_VALUE(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`.
> This is a top-3 "SQL surprises" interview question and a top-3 production bug.

## 6. Real-world use cases and applications

| Domain | Use case | Pattern |
|--------|----------|---------|
| **Product analytics** | "Top 5 features by weekly active users, per platform" | Top-N per group |
| **Growth** | "Cohort leaderboard restarted each week" | `RANK() PARTITION BY week` |
| **E-commerce** | "Two most recent orders per customer for the account page" | `ROW_NUMBER() <= 2` + `LATERAL` for the API path |
| **Marketing** | "Decile customers by LTV; export top decile to ads" | `NTILE(10)` / `CUME_DIST` |
| **Data engineering** | "Idempotent `dim_users` build: one row per user, latest wins" | Dedup with tiebreaker |
| **Data quality** | "Duplicate invoice detector feeding a quarantine table" | Duplicate detection |
| **Finance** | "Largest exposure per counterparty" | Top-N per group with `RANK()` ties |
| **Support** | "Highest-severity open ticket per account" | `RANK()` with severity ordering |
| **Streaming/CDC** | "Collapse upserts into current state" | `ROW_NUMBER() = 1` in a microbatch |
| **Dashboards** | "Share of total" column without a self-join | `SUM(x) OVER ()` |
| **Pagination** | Stable keyset pagination over a ranked set | `rn > :offset` |

Two concrete stories:

1. **"Why does the top-10 chart show 13 bars?"** A growth dashboard used
   `RANK() <= 10` on a metric with many ties (referrals per user). When a campaign created a
   3-way tie at rank 10, the chart grew to 13 bars overnight and the exec deck was wrong.
   Fix: `ROW_NUMBER() <= 10` with an explicit tiebreaker, and a footnote saying ties are
   broken by user id.
2. **"Why did the nightly dedup job produce different row counts on two identical
   reruns?"** `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC)` on data
   with same-second timestamps. The engine's sort was not stable under parallelism, so
   different rows survived each run, and the downstream `MERGE` reported different
   `rows_updated`. Fix: add the ingestion sequence to the `ORDER BY`.

## 7. Performance notes and anti-patterns

### 7.1 What the engine actually does

Ranking is implemented as a **WindowAgg over a sorted stream** (PostgreSQL: `WindowAgg` on
top of a `Sort` or an `Index Scan` that already provides the order; SQL Server: a `Segment`
+ `Sequence Project`/`Window Spool` operator on top of a `Sort`; Snowflake/BigQuery: a
distributed sort-and-window stage with an exchange/shuffle on `PARTITION BY`).

Two consequences:

- **One sort (or one shuffle) per distinct window definition.** Multiple windows sharing the
  same `PARTITION BY ... ORDER BY ...` should be expressed with one `WINDOW w AS (...)` clause
  so the engine can reuse the sort. Different windows ⇒ additional sorts.
- **The sort is the cost.** If an index provides the order, the plan degrades to a streaming
  scan and the cost approaches O(rows).

### 7.2 Indexing for ranking

The index that helps a window function has key order
`(PARTITION BY columns..., ORDER BY columns...)`, with the payload columns included
(`INCLUDE` on SQL Server / `INCLUDE` on PostgreSQL 11+):

```sql
CREATE INDEX ix_orders_user_ts_desc ON orders (user_id, order_ts DESC) INCLUDE (status);
-- Backs: ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_ts DESC)
```

For the top-N-per-group API pattern (`LATERAL ... LIMIT 1` per key), this index turns each
subquery into a 1-row index scan — the classic "loose index scan" shape.

Filtering **before** the window is mandatory. `WHERE status <> 'cancelled'` in the CTE
reduces the sorted row count; the same predicate *outside* the CTE sorts everything first and
throws most of it away:

```sql
-- ❌ sorts the entire table, then discards 99% of it
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_ts DESC) rn
               FROM orders) x WHERE rn = 1 AND status <> 'cancelled';

-- ✅ filters first (the engine can use the index on status too)
WITH r AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_ts DESC) rn
           FROM orders WHERE status <> 'cancelled')
SELECT * FROM r WHERE rn = 1;
```

### 7.3 Anti-patterns

| Anti-pattern | Why it hurts | Fix |
|--------------|--------------|-----|
| Filtering a window function in `WHERE` | Not allowed; people then write correlated subqueries | CTE + outer filter, or `QUALIFY` |
| `ROW_NUMBER()` without a tiebreaker on non-unique keys | Nondeterministic results across runs/engines | Add a unique column to `ORDER BY` |
| `RANK()` when you promised exactly N rows | Variable row counts | `ROW_NUMBER()` |
| `ROW_NUMBER()` when ties must be kept | Drops legitimate tied rows | `RANK()`/`DENSE_RANK()` |
| `NTILE(100)` on small partitions | Buckets ≈ ranks; misleading percentiles | `PERCENT_RANK()`/`CUME_DIST()` |
| `LAST_VALUE` without the `UNBOUNDED FOLLOWING` frame | Returns the current row | Explicit frame, or `MAX(x) OVER (PARTITION BY ...)` |
| Many distinct windows in one query | One extra sort/shuffle each | `WINDOW` clause, or compute in stages |
| Ranking entire fact tables | Billion-row sorts | Aggregate to a coarser grain first, then rank |
| Using ranking where an aggregate suffices | Extra sort for nothing | `MAX(x) OVER (PARTITION BY k)` avoids `ORDER BY` and thus the sort in some engines |
| Assuming `NULLS LAST` | Engine-dependent defaults | Say it explicitly |

### 7.4 Scale considerations

- **Rank the smallest population that answers the question.** "Top 10 products by revenue"
  should rank ~thousands of aggregated rows, not hundreds of millions of line items.
  Aggregate (or pre-aggregate into a rollup/summary table or a dbt mart) first.
- **Partition pruning beats windowing.** In `orders` partitioned by month, always constrain
  `order_ts` so the window runs on one partition.
- **Beware high-cardinality `PARTITION BY`.** `PARTITION BY user_id` over 500M rows means a
  shuffle on ~hundreds of millions of groups. If you only need the top 100 users, aggregate
  to user level first (with a `GROUP BY`, which can be hash-aggregated and spilled
  efficiently).
- **`QUALIFY` does not make it faster**; it is syntax sugar over the same plan. It does make
  reviews faster.
- **Watch the single-partition case.** With no `PARTITION BY`, the whole result set is one
  partition and the window must see every row — the engine cannot stream results before the
  sort completes (memory/spill pressure). This shows up as "disk spill" in
  `EXPLAIN (ANALYZE, BUFFERS)`.

## 8. Cross-engine differences

### 8.1 Ranking functions

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `ROW_NUMBER / RANK / DENSE_RANK / PERCENT_RANK / CUME_DIST / NTILE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `NTILE` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `NTH_VALUE` | ✅ 11+ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Named `WINDOW w AS (...)` clause | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `QUALIFY` | ❌ | ❌ | ❌ | ✅ 26ai+ | ✅ | ✅ |
| `DISTINCT ON` (PG shortcut for dedup) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `TOP n WITH TIES` | via `FETCH FIRST n ROWS WITH TIES` (13+) | ❌ | ✅ `TOP n WITH TIES` | ✅ `FETCH FIRST n ROWS WITH TIES` | ❌ | ❌ |
| `KEEP (DENSE_RANK FIRST/LAST …)` aggregate | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `WIDTH_BUCKET` | ✅ | ❌ | ❌ | ✅ | ✅ `WIDTH_BUCKET` | ❌ (use `RANGE_BUCKET`) |
| `LATERAL` for top-N-per-key | ✅ | ✅ 8.0.14+ (derived tables only) | ✅ `CROSS APPLY` | ✅ `CROSS APPLY` | ✅ `LATERAL` | ✅ `CROSS JOIN UNNEST(...)` / correlated `ARRAY` subquery |
| Window functions in `WHERE` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Window functions in `HAVING`/`QUALIFY` | ❌ | ❌ | ❌ | `QUALIFY` only | `QUALIFY` only | `QUALIFY` only |

Notes that matter in practice:

- **MySQL 8.0+** is the floor for window functions on MySQL (5.7 has none). MySQL also
  **prohibits `DISTINCT` inside a window function** (`COUNT(DISTINCT x) OVER (...)` is an
  error) and cannot filter on a window function without a derived table.
- **PostgreSQL** requires the `OVER` clause always; it errors with
  `ERROR:  window functions are not allowed in WHERE`. PostgreSQL's `WINDOW` clause is
  standards SQL and is the cleanest way to share a window.
- **SQL Server** requires `CROSS APPLY` instead of `LATERAL`, and `TOP n WITH TIES` requires
  an accompanying `ORDER BY`. Window functions have been available since **2005**, but the
  `ROWS`/`RANGE` frame with `PRECEDING`/`FOLLOWING` offsets only since **2012**.
- **Oracle** has the richest extra toolkit: `KEEP (DENSE_RANK FIRST ORDER BY ...)`,
  `STATS_MODE`, `RATIO_TO_REPORT`, `WIDTH_BUCKET`. Oracle 26ai added `QUALIFY`.
- **Snowflake** defaults the window frame for `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` to
  `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, which **does not comply with
  ANSI** and differs from PostgreSQL. Always write the frame explicitly when porting.
- **BigQuery** supports the `WINDOW` clause but `RANGE` frames require the `ORDER BY`
  expression to be **numeric** (and exactly one expression), so date-based `RANGE` frames
  must be rewritten with `UNIX_DATE()`/`UNIX_SECONDS()`.
- **Snowflake / BigQuery / Redshift / Databricks / DuckDB** support `QUALIFY`, which removes
  the CTE wrapper and is the shortest readable form of every pattern in this chapter.

### 8.2 Portability recipe

To keep ranking SQL portable across all six engines:

1. Never use `QUALIFY` in shared code — wrap in a CTE instead.
2. Always spell out `NULLS FIRST`/`NULLS LAST`.
3. Always include a unique tiebreaker in `ORDER BY`.
4. Use `FETCH FIRST n ROWS ONLY` (standards) rather than `LIMIT`/`TOP` when portability
   matters; use `LIMIT`/`TOP` in engine-specific hot paths for clarity.
5. Prefer `ROW_NUMBER() = 1` over `DISTINCT ON`.
6. For top-N per key, prefer the window version over `LATERAL`/`CROSS APPLY` unless the
   source is huge and the key set is small (then `LATERAL` with an index wins).

## 9. Exercises — Intermediate (hints only)

1. **Top 3 orders per user.** Return the three largest orders (`revenue`) for every user,
   with the order date. *Hint:* CTE with `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY
   revenue DESC, order_id DESC)`, then `rn <= 3`. Justify the tiebreaker in a comment.
2. **Latest event per user.** One row per user: their most recent event and its type.
   *Hint:* partition by `user_id`, order by `event_time DESC, event_id DESC`. Why is
   `event_id` needed? Construct a case where `event_time` is not unique.
3. **Second-highest revenue per category.** For each product category, the product with the
   second-highest revenue. *Hint:* `DENSE_RANK()` if ties should skip nothing, `ROW_NUMBER()`
   if you want exactly one row. Decide and defend.
4. **Find the ties.** List every (user, order date) pair where two or more orders share the
   same timestamp. *Hint:* `GROUP BY ... HAVING COUNT(*) > 1`.
5. **Rank users by number of distinct products purchased.** Which users bought the most
   distinct products? *Hint:* you need a `COUNT(DISTINCT product_id)` per user *before*
   ranking — you cannot `COUNT(DISTINCT)` inside a window in many engines.
6. **Quartile assignment with a stable tiebreak.** Split users into 4 quartiles by total
   revenue; produce a table with `user_id`, `revenue`, `quartile`. *Hint:* `NTILE(4)` with
   `ORDER BY revenue DESC, user_id`.
7. **Percentile score.** For each user, compute the fraction of users spending **at most**
   as much. *Hint:* `CUME_DIST()`. Explain why no user can have 0.
8. **Deduplicate the events table.** Assume `events` is appended by an at-least-once job that
   can write the same `(user_id, event_type, event_time)` twice. Write the dedup query.
   *Hint:* `ROW_NUMBER()` over the natural key, `ORDER BY event_id`.
9. **Top-N per group with `LATERAL`.** Rewrite exercise 1 using `CROSS JOIN LATERAL ... LIMIT 3`.
   *Hint:* index on `(user_id, revenue DESC)` — but note `revenue` is computed, so you may need
   a materialized view; discuss.
10. **Rolling "best so far".** For each order in time order (per user), show the largest
    order amount *seen so far*. *Hint:* `MAX(revenue) OVER (PARTITION BY user_id ORDER BY
    order_ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Deterministic dedup under late-arriving correction events.** Your `orders` feed contains
   both the original row and later *correction* rows with the same `order_id` but a
   `_op` column (`insert` / `update` / `delete`) and an `ingest_seq`. Write a query that
   produces the *current* state (deleted orders excluded) as of a given `ingest_seq`, and
   prove it is idempotent by running it twice.
   *Expected approach:* `ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY ingest_seq DESC)`
   with `rn = 1`, then filter out `_op = 'delete'` **after** the dedup (filtering before
   would resurrect a deleted row if an older `insert` existed). Discuss why the filter
   placement is the whole exercise.
2. **Top-N per group with a *minimum-quality* constraint.** "Top 3 products per category by
   revenue, but only categories with at least 20 distinct buyers, and only products whose
   return rate is below 5%." Give the plan order (aggregate → rank → filter, or rank →
   filter → rank?) and explain which intermediate results must be materialized to avoid
   ranking rows you are about to discard.
3. **Stable pagination over a ranked set.** Provide keyset pagination for "users ordered by
   revenue DESC, user_id ASC" that is stable under concurrent inserts. Explain why
   `OFFSET` is wrong here, what the cursor key is, and how the query changes when `revenue`
   is itself a window function result.
4. **`RANK()` vs `DENSE_RANK()` in a metric definition.** A "Top-N products" KPI is defined
   as "products in the top 10 revenue ranks". Two analysts disagree on the number of
   products in the KPI because of ties. Write the two SQL variants, quantify the difference
   on a realistic distribution (Zipf-like), and write the one-paragraph definition you would
   put in the metric's documentation.
   *Expected approach:* build a synthetic distribution with `generate_series` and
   `random()`, compare `COUNT(*) FILTER (WHERE rnk <= 10)` vs
   `COUNT(*) FILTER (WHERE dense_rnk <= 10)`, and conclude that the metric must specify
   *rank semantics* explicitly. This is a metric-governance problem dressed as a SQL problem.
5. **Incremental dedup without re-reading history.** You must maintain a `dim_users__current`
   table incrementally (only yesterday's file). Explain why plain `ROW_NUMBER()` over
   yesterday's file is insufficient (the "latest row may arrive later" problem), and write
   the `MERGE`/`upsert` that handles both insert-only and out-of-order arrivals.
   *Expected approach:* dedup within the incoming partition first, then upsert with a
   *watermark comparison* (`ON target.user_id = src.user_id AND src.updated_at >= target.updated_at`),
   plus a tie rule for equal timestamps. Discuss idempotency and replays.
6. **Ranking with `NULL`s in the ordering column.** You must rank products by
   `return_rate DESC NULLS FIRST` (unknown return rates sort worst). Write the query, then
   explain how the result changes if the engine's default NULL ordering is used instead.
   *Expected approach:* explicit `NULLS FIRST` plus an assertion query that fails loudly if
   any engine-dependent behaviour is observed (e.g. compare against a
   `COALESCE(return_rate, -1)` rewrite that is NULL-order-independent).

## References

- PostgreSQL — Window Functions: <https://www.postgresql.org/docs/current/functions-window.html>
- PostgreSQL — Window Function Calls (syntax, `WINDOW` clause, frame defaults): <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- PostgreSQL — Tutorial: Window Functions: <https://www.postgresql.org/docs/current/tutorial-window.html>
- PostgreSQL — `SELECT` (`DISTINCT ON`, `FETCH FIRST … WITH TIES`): <https://www.postgresql.org/docs/current/sql-select.html>
- MySQL 8.0 Reference Manual — Window Functions: <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- MySQL 8.0 Reference Manual — Window Function Concepts and Syntax: <https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html>
- MySQL 8.0 Reference Manual — Window Function Restrictions: <https://dev.mysql.com/doc/refman/8.0/en/window-function-restrictions.html>
- Microsoft Learn — `OVER` Clause (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql>
- Microsoft Learn — Ranking Functions (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/functions/ranking-functions-transact-sql>
- Oracle Database — Analytic Functions: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/Analytic-Functions.html>
- Oracle Database — `RANK`, `DENSE_RANK`, `CUME_DIST`, `PERCENT_RANK`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/RANK.html>
- ORACLE-BASE — `QUALIFY` clause in Oracle 26ai: <https://oracle-base.com/articles/26/qualify-clause-26>
- Snowflake — Window function syntax and usage: <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — Analyzing data with window functions: <https://docs.snowflake.com/en/user-guide/functions-window-using>
- Google Cloud — BigQuery window function calls (frames, `QUALIFY`): <https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls>
- Google Cloud — BigQuery window functions list: <https://cloud.google.com/bigquery/docs/reference/standard-sql/window-functions>
- Amazon Redshift — `QUALIFY` clause: <https://docs.aws.amazon.com/redshift/latest/dg/r_QUALIFY_clause.html>
- DuckDB — Window functions: <https://duckdb.org/docs/stable/sql/functions/window_functions.html>
- Modern SQL (Markus Winand) — window function support matrix and standards notes: <https://modern-sql.com/>
- Mode / ThoughtSpot — SQL Window Functions tutorial (`ROW_NUMBER`, `RANK`, `NTILE`, `LAG`): <https://mode.com/sql-tutorial/sql-window-functions/>
- Itzik Ben-Gan — "How to use Microsoft SQL Server 2012's window functions": <https://www.itprotoday.com/innovations-of-the-2010s/how-to-use-microsoft-sql-server-2012-s-window-functions-part-1>
- use-the-index-luke — indexing for `ORDER BY` / index key order: <https://use-the-index-luke.com/sql/sorting-grouping>

# Set Operations

> `UNION`, `INTERSECT`, `EXCEPT` are the only SQL operators that treat an entire *result set*
> as a value. They stack relations vertically (instead of horizontally like a `JOIN`), they are
> the natural tool for appending partitions, comparing two populations, and testing data
> pipelines — and they are the source of two very common mistakes: using `UNION` (which
> de-duplicates) when you meant `UNION ALL` (which does not), and assuming `NULL = NULL` is
> false when the set operators treat `NULL` as an ordinary value. This chapter covers the
> semantics precisely, including the `ALL` variants that most engineers have never used and
> that answer "multiset difference" questions exactly.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- `UNION ALL` = concatenation (**bag/multiset** semantics, no de-duplication, parallelizable).
  `UNION` = `UNION ALL` + de-duplication (**set** semantics, requires a sort or hash aggregate).
  `UNION` is the default when you omit the modifier.
- `INTERSECT` = rows present in **both** inputs (deduplicated). `EXCEPT` (a.k.a. `MINUS`) = rows
  in the first input **not** in the second (deduplicated).
- **`ALL` variants count copies** and are the exact multiset operations:
  `INTERSECT ALL` emits `min(count_left, count_right)` copies;
  `EXCEPT ALL` emits `max(0, count_left − count_right)` copies.
- **Set operators compare `NULL` as equal to `NULL`** (unlike `=`). So `INTERSECT` finds NULLs
  that appear on both sides, and `UNION` de-duplicates multiple NULL rows into one.
- Rows are matched **by position**, not by name: column names come from the **first** query.
  Two tables with the same columns in a different order produce silently wrong results —
  hence Snowflake's `UNION BY NAME` extension.
- **Precedence:** `INTERSECT` binds tighter than `UNION`/`EXCEPT`; `UNION` and `EXCEPT` are
  left-associative with equal precedence. Parenthesize whenever you mix them.
- A single `ORDER BY` / `LIMIT` applies to the **whole** set operation, not to a branch. To
  limit a branch, wrap it in a subquery.
- **Performance:** `UNION ALL` is a cheap append; `UNION`/`INTERSECT`/`EXCEPT` force a global
  de-duplication (hash aggregate or sort) — in distributed engines, that is a **shuffle**.
- **`EXCEPT` is a great data-quality primitive**: "expected minus actual" and "actual minus
  expected" is the standard dbt test for `SELECT`-based assertions.

---

## 1. Definitions and core concepts

### 1.1 The six operators

| Operator | Result | Duplicates | Cost driver |
|---|---|---|---|
| `UNION ALL` | all rows from both inputs | preserved | append only |
| `UNION` (`UNION DISTINCT`) | all distinct rows from both | removed | hash/sort dedup of the whole result |
| `INTERSECT` | distinct rows in both | removed | hash/sort dedup of both + join |
| `INTERSECT ALL` | `min(m,n)` copies of each value | counted | group-by counting join |
| `EXCEPT` / `MINUS` | distinct rows in the first, not the second | removed | hash/sort anti join |
| `EXCEPT ALL` / `MINUS ALL` | `max(0, m-n)` copies of each value | counted | group-by counting anti join |

### 1.2 Set vs bag semantics, precisely

Let `A` and `B` be multisets, and `cA(x)`, `cB(x)` the number of copies of row value `x`:

| Expression | Multiplicity of `x` in the result |
|---|---|
| `A UNION ALL B` | `cA(x) + cB(x)` |
| `A UNION B` | `1` if `cA(x) + cB(x) > 0`, else nothing |
| `A INTERSECT B` | `1` if `cA(x) > 0` and `cB(x) > 0` |
| `A INTERSECT ALL B` | `min(cA(x), cB(x))` |
| `A EXCEPT B` | `1` if `cA(x) > 0` and `cB(x) = 0` |
| `A EXCEPT ALL B` | `max(0, cA(x) − cB(x))` |

```
A = [1, 1, 2, 3]        B = [1, 2, 2, 4]

A UNION ALL B       = [1,1,2,3,1,2,2,4]            (8 rows)
A UNION B           = [1,2,3,4]                    (4 rows)
A INTERSECT B       = [1,2]                        (2 rows)
A INTERSECT ALL B   = [1,2]                        (min(2,1)=1 for 1; min(1,2)=1 for 2)
A EXCEPT B          = [3]                          (1 row)
A EXCEPT ALL B      = [1,3]                        (max(0,2-1)=1 for 1; max(0,1-2)=0 for 2)
```

### 1.3 Compatibility rules

1. **Same number of columns** in every branch (except Snowflake's `UNION [ALL] BY NAME`).
2. **Compatible types per position.** The engine picks a common type; if it cannot
   (`text` vs `integer`), it errors; if it can (`integer` vs `numeric`), it silently casts.
3. **Names come from the first branch.** `SELECT a AS x UNION ALL SELECT b AS y` produces a
   column named `x`.
4. **Matching is positional.** The second branch's `y` is matched against `x` regardless of
   names.
5. `ORDER BY` may reference output column names/positions, applies to the final result, and
   must appear only once, at the end.

### 1.4 Vocabulary

| Term | Meaning |
|---|---|
| **Set semantics** | duplicates removed (`UNION`, `INTERSECT`, `EXCEPT`) |
| **Bag / multiset semantics** | duplicates counted (`UNION ALL`, `INTERSECT ALL`, `EXCEPT ALL`) |
| **Set difference (symmetric)** | `(A EXCEPT B) UNION ALL (B EXCEPT A)` — a.k.a. XOR; the standard way to detect mismatches |
| **Append / stack** | the physical implementation of `UNION ALL` in column stores (an `Append`/`Union` node with no de-duplication) |
| **Global dedup** | the sort/hash-aggregate step `UNION`/`INTERSECT`/`EXCEPT` require |

---

## 2. Why it matters / what problem it solves

- **Appending sources.** Same shape, different origin: web orders from the transactional DB,
  marketplace orders from a partner feed, legacy orders from an archive. `UNION ALL` is *the*
  ingestion primitive, and it is what partitioned tables/views do under the hood.
- **Comparing populations.** "Which customers are in the paid set but not the trial set?"
  (`EXCEPT`), "which users appear in both experiments?" (`INTERSECT`), "what is the exact
  multiset difference between the staging and target tables?" (`EXCEPT ALL`).
- **Testing.** The canonical SQL data test is `expected EXCEPT actual` ∪ `actual EXCEPT
  expected` = empty. dbt's `custom` tests are literally this.
- **Migration validation.** Run the same query against the old and new engine, `EXCEPT` the
  results, and check for zero rows — the standard engine-migration acceptance criterion.
- **Reconciliation without joins.** Set difference is immune to the fan-out and NULL pitfalls
  of joins; for whole-row comparison it is the simplest correct tool.
- **Deduplication with a preference.** `UNION ALL` + `ROW_NUMBER()`/`DISTINCT ON` gives
  "dedup but keep the row I want" — impossible with plain `UNION`.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Logical model

All six operators are functions over *bags of rows*. Conceptually:

```
UNION ALL      → Append(left, right)                       (no comparison between rows)
UNION          → Distinct(Append(left, right))             (compare all columns)
INTERSECT      → Distinct(SemiJoin(left, right))           (compare all columns)
EXCEPT         → Distinct(AntiJoin(left, right))           (compare all columns)
* ALL variants → GroupBy(all columns) + count arithmetic    (min / difference)
```

The comparison is over **all output columns**, using NULL-safe equality (`IS NOT DISTINCT FROM`
semantics), not `=`.

### 3.2 Physical plans

| Operator | Postgres plan | Notes |
|---|---|---|
| `UNION ALL` | `Append` | can run branches in parallel |
| `UNION` | `HashAggregate` or `Sort` + `Unique` over an `Append` | memory or disk |
| `INTERSECT` | `HashSetOp Intersect` | builds a hash of one side |
| `EXCEPT` | `HashSetOp Except` | anti-join shaped |
| `* ALL` | `GroupAggregate` + counting | more expensive than the distinct forms, not cheaper |

Distributed engines (Snowflake, BigQuery, Spark) add a nuance: `UNION ALL` across branches is a
**local** operation unless the branches come from different shuffles, whereas `UNION` forces a
**global** de-duplication → a full shuffle/repartition. On a 1 TB table that is the difference
between seconds and many minutes.

### 3.3 `ORDER BY`, `LIMIT`, and scope

```sql
-- LEGAL: one ORDER BY at the end, applying to the whole result
SELECT id FROM a UNION ALL SELECT id FROM b ORDER BY id;

-- ILLEGAL: a per-branch ORDER BY (syntax error everywhere)
SELECT id FROM a ORDER BY id
UNION ALL
SELECT id FROM b;                      -- ERROR

-- LEGAL: per-branch ordering requires a subquery/parenthesized branch with LIMIT
(SELECT id FROM a ORDER BY id DESC LIMIT 5)
UNION ALL
(SELECT id FROM b ORDER BY id DESC LIMIT 5)
ORDER BY id DESC LIMIT 10;             -- the final ORDER BY/LIMIT still applies to the union

-- ILLEGAL: per-branch LIMIT as a bare clause in some engines (Postgres requires parens)
SELECT id FROM a LIMIT 5 UNION ALL SELECT id FROM b;   -- ERROR in Postgres
```

`> Senior note`: a `UNION` cannot guarantee *which* duplicate survives de-duplication — there is
no "first row wins" semantics. If you need a preference (latest, most complete, non-NULL),
use `UNION ALL` plus `ROW_NUMBER() OVER (PARTITION BY key ORDER BY preference)` or
`DISTINCT ON`.

### 3.4 Precedence

Following the ANSI/ISO standard (and implemented by PostgreSQL, MySQL, Snowflake):

```
INTERSECT  >  UNION | EXCEPT     (INTERSECT binds tighter)
UNION, EXCEPT: equal precedence, left-associative
```

```sql
-- Without parens: INTERSECT first, then UNION
SELECT 1 UNION SELECT 2 INTERSECT SELECT 2;      -- {1, 2}

-- Same thing explicitly (recommended style)
(SELECT 1 UNION SELECT 2) INTERSECT SELECT 2;    -- {2}  ← different query!
(SELECT 1 UNION (SELECT 2 INTERSECT SELECT 2));  -- {1, 2}
```

Follow Snowflake's own documentation advice: **always parenthesize** when mixing operators, or
your query's meaning changes when ported to an engine that differs.

---

## 4. Syntax and mechanics (with examples)

### 4.1 The six forms

```sql
-- 1. Append, keep duplicates, cheapest
SELECT customer_id, 'web' AS source FROM orders WHERE channel = 'web'
UNION ALL
SELECT customer_id, 'store' AS source FROM orders WHERE channel = 'store';

-- 2. Append and de-duplicate (set union)
SELECT customer_id FROM orders WHERE status = 'delivered'
UNION
SELECT customer_id FROM orders WHERE status = 'returned';

-- 3. Rows in both (distinct)
SELECT customer_id FROM orders WHERE channel = 'web'
INTERSECT
SELECT customer_id FROM orders WHERE channel = 'mobile';

-- 4. Multiset intersection
SELECT customer_id FROM orders WHERE channel = 'web'
INTERSECT ALL
SELECT customer_id FROM orders WHERE channel = 'mobile';

-- 5. Difference (set)
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;                 -- customers with no orders

-- 6. Multiset difference
SELECT customer_id FROM orders WHERE status = 'delivered'
EXCEPT ALL
SELECT customer_id FROM orders WHERE status = 'returned';
```

### 4.2 Type unification

```sql
-- FAILS: no common type for text and integer in Postgres
SELECT 'total' AS label UNION ALL SELECT COUNT(*) FROM orders;   -- ERROR

-- WORKS: cast explicitly
SELECT 'total' AS label UNION ALL SELECT COUNT(*)::text FROM orders;

-- WORKS silently: int + numeric unifies to numeric; the "total" row is 0
SELECT 0 AS n FROM (SELECT 1) t
UNION ALL
SELECT COUNT(*) FROM orders;
```

Silent unification across 'integer' and 'bigint' or 'text' and 'varchar' is fine; text+number
is a hard error in Postgres and Snowflake, and a silent conversion (or data corruption) in
looser engines. Cast on the branch boundary, deliberately.

### 4.3 The `UNION ALL` + preference dedup pattern

```sql
-- Keep the most recent row per key across two sources
WITH combined AS (
    SELECT customer_id, email, 'crm'    AS src, updated_at FROM crm_customers
    UNION ALL
    SELECT customer_id, email, 'billing' AS src, updated_at FROM billing_customers
), ranked AS (
    SELECT c.*, ROW_NUMBER() OVER (PARTITION BY customer_id
                                   ORDER BY updated_at DESC, src) AS rn
    FROM combined c
)
SELECT customer_id, email, src FROM ranked WHERE rn = 1;
```

This is *the* pattern for merging two versions of the truth, and it is only expressible with
`UNION ALL` (plain `UNION` would have discarded the information needed to choose).

### 4.4 Symmetric difference (the test helper)

```sql
-- "A and B agree" ⇔ this returns zero rows
WITH a AS (SELECT …), b AS (SELECT …)
SELECT 'only_in_a' AS diff, * FROM (SELECT * FROM a EXCEPT SELECT * FROM b) x
UNION ALL
SELECT 'only_in_b', * FROM (SELECT * FROM b EXCEPT SELECT * FROM a) y;
```

Zero rows is the assertion; any row is a failure with enough information to debug (which side).

### 4.5 Per-branch limiting and ordering

```sql
-- Top 3 orders per channel, then one combined answer
(SELECT order_id, channel, ordered_at FROM orders
  WHERE channel = 'web'  ORDER BY ordered_at DESC LIMIT 3)
UNION ALL
(SELECT order_id, channel, ordered_at FROM orders
  WHERE channel = 'store' ORDER BY ordered_at DESC LIMIT 3)
ORDER BY channel, ordered_at DESC;
```

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: append two "sources" of orders

```sql
SELECT order_id, channel, 'online' AS segment, ordered_at
FROM orders WHERE channel IN ('web','mobile')
UNION ALL
SELECT order_id, channel, 'offline' AS segment, ordered_at
FROM orders WHERE channel IN ('store','marketplace')
ORDER BY ordered_at;
```

Expected: 14 rows (all orders), with 11 `online` (web + mobile) and 3 `offline` (store ×2,
marketplace ×1). Nothing is de-duplicated — the branches are disjoint here, which is the
ideal case for `UNION ALL` (no overlap means no accidental duplication).

### Example 2 — Basic: `UNION` vs `UNION ALL` row counts

```sql
-- Union ALL over two overlapping sets
SELECT customer_id, 'has delivered' AS fact FROM orders WHERE status = 'delivered'
UNION ALL
SELECT customer_id, 'has returned'  AS fact FROM orders WHERE status = 'returned';

-- Set union over the same overlapping sets
SELECT customer_id FROM orders WHERE status = 'delivered'
UNION
SELECT customer_id FROM orders WHERE status = 'returned';

-- Set intersection
SELECT customer_id FROM orders WHERE status = 'delivered'
INTERSECT
SELECT customer_id FROM orders WHERE status = 'returned';
```

Expected: the first returns 7 rows (delivered: customers 1,1,2,4,6,6 → 6 rows; returned:
customer 2 → 1 row). The `UNION` returns 4 distinct customers (`1, 2, 4, 6`). The `INTERSECT`
returns 1 row (`2`, Bob — who both received and returned an order). This tiny example shows the
three multiplicities at once.

### Example 3 — Intermediate: `EXCEPT` as an anti join (customers who never ordered)

```sql
SELECT c.customer_id, c.full_name FROM customers c
EXCEPT
SELECT c.customer_id, c.full_name FROM customers c JOIN orders o ON o.customer_id = c.customer_id;
```

Expected: 1 row — `8 | Heidi Novak`. Equivalent to the `NOT EXISTS` version from
[Chapter 06](./06-exists-and-subqueries.md), and immune to the `NOT IN` NULL trap because set
difference uses NULL-safe equality.

Comparison:

| Form | Typical plan | When it wins |
|---|---|---|
| `NOT EXISTS` | hash anti join | indexed right side, needs to keep extra columns |
| `LEFT JOIN … IS NULL` | hash join + filter | when you also need right-side columns |
| `EXCEPT` | hash set-op | whole-row comparison, no join keys to think about, testing |

`> Senior note`: `EXCEPT` requires the *whole row* to match, including every selected column.
That is a feature for tests and a trap for differences in derived columns: if you select a
`now()`-based column, you can never get a match. Select only stable, comparable columns.

### Example 4 — Intermediate: `INTERSECT` for cross-segment customers

```sql
SELECT customer_id FROM orders
WHERE ordered_at >= '2024-01-01' AND ordered_at < '2024-02-01'      -- January buyers
INTERSECT
SELECT customer_id FROM orders
WHERE ordered_at >= '2024-03-01';                                   -- March buyers
ORDER BY customer_id;
```

Expected: buyers in January were customers 1 (1001), 2 (1003), 4 (1007), 6 (1011). March
buyers: 2 (1004), 3 (1005, 1006), 4 (1008), 5 (1009, 1010), 7 (1013), 1 (1014). Intersection =
`1, 2, 4`? Alice's March order is 1014 (2024-03-30) → yes. So `1, 2, 4` — repeat buyers across
a two-month gap. This is the "retention by month, without a cohort table" quick answer.

### Example 5 — Intermediate: `EXCEPT ALL` for multiset difference (inventory reconciliation)

```sql
-- Two independent counts of the same events: which copies appear more than once on one side?
SELECT customer_id, event_type FROM events
EXCEPT ALL
SELECT customer_id, event_type FROM events WHERE event_type <> 'page_view';
```

Expected: for each `(customer_id, event_type)` pair, the result has
`max(0, count_all − count_non_page_view)` copies — i.e. exactly the page views that the second
set removed. The page-view pairs are `(1,'page_view')` → 3 copies, `(2,'page_view')` → 1,
`(8,'page_view')` → 2, and `(NULL,'page_view')` → 2 (the two anonymous events). Total: **8
rows**. With `EXCEPT` (distinct) you would get only **4 rows** — one per distinct
`(customer_id, event_type)` pair — losing all multiplicity information. Bonus lesson: set
operators treat the `NULL` customer as an ordinary value, so all anonymous events share one
group.

### Example 6 — Advanced: precedence, demonstrated

```sql
-- (a) Default precedence: INTERSECT binds tighter → differs from the naive reading
SELECT customer_id FROM orders WHERE channel = 'web'         -- web buyers
UNION
SELECT customer_id FROM orders WHERE channel = 'mobile'      -- mobile buyers
INTERSECT
SELECT customer_id FROM orders WHERE status = 'delivered';   -- delivered buyers

-- (b) What you probably meant
(SELECT customer_id FROM orders WHERE channel = 'web')
UNION
(SELECT customer_id FROM orders WHERE channel = 'mobile')
INTERSECT
(SELECT customer_id FROM orders WHERE status = 'delivered');

-- (c) Explicit and unambiguous: all customers using web-or-mobile
SELECT DISTINCT customer_id FROM orders
WHERE channel IN ('web','mobile')
INTERSECT
SELECT customer_id FROM orders WHERE status = 'delivered';
```

Form (a) is parsed as `web UNION (mobile INTERSECT delivered)`. Forms (b) and (c) are also
parenthesized differently from each other — which is the point: **write the parentheses you
mean.** Run all three and count rows to see the difference; on the seed the differences are
small, but on real data they are not.

### Example 7 — Advanced: the migration/output test

```sql
-- Assert: the "wide" query and the "long" query agree on revenue per channel.
WITH wide AS (
    SELECT 'web' AS channel, ROUND(SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)), 2) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.channel = 'web' AND o.status IN ('paid','shipped','delivered')
    UNION ALL
    SELECT 'mobile',
           ROUND(SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)), 2)
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.channel = 'mobile' AND o.status IN ('paid','shipped','delivered')
), long AS (
    SELECT o.channel,
           ROUND(SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)), 2) AS revenue
    FROM orders o JOIN order_items i USING (order_id)
    WHERE o.status IN ('paid','shipped','delivered')
    GROUP BY o.channel
)
SELECT 'only_in_wide' AS mismatch, * FROM (SELECT * FROM wide EXCEPT SELECT * FROM long) a
UNION ALL
SELECT 'only_in_long', * FROM (SELECT * FROM long EXCEPT SELECT * FROM wide) b;
```

Expected: zero rows (the two formulations agree on web and mobile; other channels appear only
in `long`, so add them to `wide` or filter `long` to `channel IN ('web','mobile')` to make the
assertion exact). Note the lesson: **`EXCEPT` compares the full row, so the two sides must have
the same grain and the same filters.** This is the shape of a pipeline unit test.

### Example 8 — Advanced: dedup with preference (the `UNION ALL` upgrade)

```sql
-- Two systems report the customer's loyalty tier; keep the freshest, prefer non-NULL tier
WITH combined AS (
    SELECT c.customer_id, c.email, c.loyalty_tier,
           'orders_db'  AS src, MAX(o.ordered_at) AS observed_at
    FROM customers c LEFT JOIN orders o ON o.customer_id = c.customer_id
    GROUP BY c.customer_id, c.email, c.loyalty_tier
    UNION ALL
    SELECT c.customer_id, c.email, c.loyalty_tier,
           'crm_db' AS src, c.signup_ts AS observed_at
    FROM customers c
), ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id
               ORDER BY (loyalty_tier IS NULL), observed_at DESC, src) AS rn
    FROM combined
)
SELECT customer_id, email, loyalty_tier, src FROM ranked WHERE rn = 1
ORDER BY customer_id;
```

Expected: 8 rows, one per customer (note customer 8 keeps a row with `loyalty_tier = NULL`
because both sources agree it is NULL — the `(loyalty_tier IS NULL)` ordering expression puts
non-NULL tiers first). Plain `UNION` cannot express this because it has no notion of priority.

### Example 9 — Advanced: `UNION` vs `UNION ALL` in recursive CTEs (the semantics link)

```sql
-- UNION ALL: keeps duplicates → a cyclic edge table never terminates
WITH RECURSIVE walk AS (
    SELECT category_id, parent_id, 1 AS depth FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id, w.depth + 1
    FROM categories c JOIN walk w ON c.parent_id = w.category_id
)
SELECT COUNT(*) FROM walk;

-- UNION (distinct): removes already-seen rows → terminates even on cycles
WITH RECURSIVE walk AS (
    SELECT category_id, parent_id FROM categories WHERE parent_id IS NULL
    UNION
    SELECT c.category_id, c.parent_id
    FROM categories c JOIN walk w ON c.parent_id = w.category_id
)
SELECT COUNT(*) FROM walk;
```

Both return 8 rows for the healthy seed. The difference is what happens with a cycle — see
[Chapter 03 §3.5](./03-recursive-ctes.md). The `UNION` in a recursive CTE is not just a
de-duplication switch; it is a *termination strategy* (paid for with a global hash of the whole
result).

---

## 6. Real-world use cases and applications

| Use case | Operator | Note |
|---|---|---|
| **Appending partitions / multi-source ingestion** | `UNION ALL` | the only safe choice (no accidental dedup) |
| **Merging CDC snapshots** | `UNION ALL` + `ROW_NUMBER` | preference-based dedup |
| **Anti-join without join syntax** | `EXCEPT` | whole-row; immune to NULL traps |
| **Audience overlap** | `INTERSECT` | "users in both experiments" |
| **Data-quality tests (dbt, Great Expectations)** | `EXCEPT` both directions | zero rows = pass |
| **Engine-migration acceptance** | `EXCEPT` both directions | standard validation |
| **Multiset reconciliation (counts differ)** | `EXCEPT ALL` / `INTERSECT ALL` | copy-level differences |
| **Schema-change buffering (old vs new columns)** | `UNION ALL` with `NULL` padding | make the union shape-compatible |
| **Building a "spine" of dimensions** | `UNION ALL` of `VALUES` lists | a poor-man's `VALUES` table |
| **Gap tenancy (calendar ∪ spikes)** | `UNION` | de-duplicate dates |
| **Snapshot comparison (SCD-free "diff")** | `EXCEPT` | daily change detection |
| **Testing `NULL` handling** | `INTERSECT` | NULLs match, unlike `=` |

---

## 7. Performance notes and anti-patterns

### 7.1 Cost model

| Operator | Cost | Parallelism |
|---|---|---|
| `UNION ALL` | `O(n)` append; no comparison | full (branches independent) |
| `UNION` | `O(n)` + hash/sort dedup; memory ∝ distinct rows | dedup is a barrier |
| `INTERSECT` | `O(n + m)` + hash build | barrier |
| `EXCEPT` | `O(n + m)` + hash build | barrier |
| `* ALL` variants | grouping + counting | barrier |

Rules of thumb:

- Use `UNION ALL` unless you *need* de-duplication. It is almost always faster, and using
  `UNION` "just in case" is the most common silent performance problem in BI SQL.
- If you need de-duplication but only on a subset of columns (e.g. one row per customer, latest
  wins), use `UNION ALL` + a ranking window — it is more code but far more predictable.
- In column stores and distributed engines, the de-duplication step is a **shuffle**. `UNION`
  across a 100-partition table is a full data movement.
- `UNION ALL` of two branches that both scan the same table is two scans. If you can express
  the set as a `CASE`/`WHERE` union *within* one scan, do that instead.

### 7.2 `UNION ALL` of two "shapes" — the column-order trap

```sql
-- DANGEROUS: relies on column order matching between two schemas
SELECT * FROM orders_2023
UNION ALL
SELECT * FROM orders_2024;

-- SAFE: name and order the columns explicitly
SELECT order_id, customer_id, status, ordered_at FROM orders_2023
UNION ALL
SELECT order_id, customer_id, status, ordered_at FROM orders_2024;
```

Using `SELECT *` with `UNION ALL` is the classic silent corruption: adding a column to one table
(or reordering) changes the meaning of the query without an error. Snowflake's
`UNION [ALL] BY NAME` exists precisely for schema-evolution-heavy pipelines; in portable code,
**always list columns.**

### 7.3 Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| `UNION` where `UNION ALL` was meant | hidden sort/hash, memory spikes, slow queries | `UNION ALL` |
| `UNION` to "fix" duplicates caused by a bad join | masking the real bug (and still slow) | fix the join grain |
| `SELECT *` in a `UNION ALL` | silent misalignment after schema change | explicit column list |
| Assuming names control matching | wrong results | remember: positional matching |
| `ORDER BY` inside a branch | syntax error | subquery/parenthesized branch |
| Per-branch `LIMIT` without parens | syntax error | wrap each branch |
| `UNION` of queries with different filters expecting one row | unexpected multiplicity | count rows in both branches |
| `EXCEPT` over a select list including a timestamp/`now()` | never matches | select only stable columns |
| Deep `UNION ALL` chains (50+ branches) | plan bloat, long planning time | a partitioned table or a `VALUES` join |
| Mixing types implicitly | silent casts, wrong sort order (`'10' < '9'`) | cast explicitly |
| Mixing `UNION`/`INTERSECT` without parens | meaning changes on port | parenthesize |

### 7.4 Deduplication recipes (ranked by preference)

```sql
-- (1) Keep the best row per key (most flexible, most explicit)
SELECT * FROM (
    SELECT t.*, ROW_NUMBER() OVER (PARTITION BY key ORDER BY preference_1, preference_2) rn
    FROM combined t) x
WHERE rn = 1;

-- (2) PostgreSQL DISTINCT ON (concise, Postgres-only)
SELECT DISTINCT ON (key) * FROM combined ORDER BY key, preference_1, preference_2;

-- (3) Plain UNION (only when "any copy" is acceptable)
SELECT DISTINCT * FROM combined;
```

---

## 8. Cross-engine differences

| Feature | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `UNION` / `UNION ALL` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `INTERSECT` | ✅ | ✅ 8.0.31+ | ✅ | ✅ | ✅ | ✅ (`INTERSECT DISTINCT` — the keyword is required) |
| `EXCEPT` | ✅ | ✅ 8.0.31+ | ✅ | ✅ **21c** (before: `MINUS`) | ✅ (alias `MINUS`) | ✅ (`EXCEPT DISTINCT`) |
| `MINUS` alias | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `INTERSECT ALL` | ✅ | ✅ 8.0.31+ | ❌ | ✅ 21c | ❌ | ❌ |
| `EXCEPT ALL` / `MINUS ALL` | ✅ | ✅ 8.0.31+ | ❌ | ✅ 21c (`EXCEPT ALL`, `MINUS ALL`) | ❌ | ❌ |
| Precedence (`INTERSECT` tighter) | ✅ ANSI | ✅ ANSI | ⚠️ SQL Server evaluates strictly left-to-right (unlike the standard) | ✅ ANSI | ✅ ANSI (documented) | ✅ ANSI |
| `UNION BY NAME` | ❌ | ❌ | ❌ | ❌ | ✅ (`UNION [ALL] BY NAME`) | ❌ (`UNION ALL BY NAME` exists in BigQuery!) |
| Per-branch `ORDER BY/LIMIT` | requires parens | requires parens | requires `TOP` in a branch + parens | requires parens/subqueries | requires parens | requires parens |
| One `ORDER BY` per statement | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| NULLs compare equal in set ops | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Plan node | `Append`, `HashAggregate`, `HashSetOp` | `UNION RESULT`/temp table | `Concatenation`, `Hash Match (Union/Aggregate)` | `UNION-ALL`, `HASH UNIQUE`, `MINUS` | `Union`, `Intersect`, `Except` | `UNION ALL` stages, `DISTINCT` stages |

Engine notes:

- **MySQL 8.0.31** finally added `INTERSECT` and `EXCEPT` (with `ALL` modifiers). Before that,
  the workarounds were `INNER JOIN`/`DISTINCT` for intersection and `NOT EXISTS`/`LEFT JOIN …
  IS NULL` for difference. If you maintain MySQL 5.7 code, you will see those idioms.
- **SQL Server** has `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT` — **no `ALL` variants** of
  `INTERSECT`/`EXCEPT`. To get multiset semantics, `GROUP BY` the columns and `HAVING` the
  counts.
- **Oracle** historically used `MINUS` for set difference; 21c added `EXCEPT` (standard) plus
  `MINUS ALL`, `INTERSECT ALL`, `EXCEPT ALL`. Code targeting 19c must use `MINUS`.
- **Snowflake** documents the ANSI precedence explicitly and recommends parentheses. It also
  supports `UNION [ALL] BY NAME`, which matches columns by *name* — extremely useful for
  schema-evolving ingestion.
- **BigQuery** requires the `DISTINCT` keyword (`INTERSECT DISTINCT`, `EXCEPT DISTINCT`) and has
  no `ALL` variants; it also requires the same number of columns and compatible types. `UNION
  ALL BY NAME` is supported.
- **PostgreSQL** is the most complete: all six operators, ANSI precedence, and `UNION` in
  recursive CTEs for cycle-safe traversal.

---

## 9. Exercises — Intermediate (hints only)

1. **Append `orders` from two channels with a `segment` label, and verify the row count equals
   the two branch counts summed.** *Hint:* `UNION ALL` then compare with `COUNT(*)` per branch.
2. **List customers who ordered in January but not in March.** *Hint:* `EXCEPT`; then write the
   `NOT EXISTS` equivalent and compare row counts.
3. **List customers who ordered in both January and March.** *Hint:* `INTERSECT`.
4. **Compute the multiset difference between two event subsets.** *Hint:* `EXCEPT ALL`, and
   explain the `max(0, m−n)` rule using a row-count check.
5. **Deduplicate a `UNION ALL` of two customer sources, keeping the row with the newest
   `updated_at`.** *Hint:* `ROW_NUMBER()` over the combined CTE.
6. **Show that `(SELECT 1 UNION SELECT 2) INTERSECT SELECT 2` differs from
   `SELECT 1 UNION SELECT 2 INTERSECT SELECT 2`.** *Hint:* parse trees; parenthesize both.
7. **Write a per-branch top-2 query with `UNION ALL` and a final `ORDER BY`.** *Hint:*
   parenthesized branches with `LIMIT`, then an outer `ORDER BY`.
8. **Add a `total` row to a grouped result using `UNION ALL`.** *Hint:* one branch is the
   `GROUP BY`, the other is the grand total with the group column set to `'ALL'`; watch types.
9. **Find rows in a `staging` table that are missing from `target` and vice versa, and identify
   which side each missing row came from.** *Hint:* the symmetric-difference helper with a
   literal `'only_in_x'` label column — remember the label must be in *both* branches.
10. **Explain why `UNION` over a select list that includes `now()` can never de-duplicate
    two otherwise identical rows.** *Hint:* the row value includes the timestamp.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Design a full-reconciliation framework for two systems of record.** Requirements: per-key
   status (`only_left`, `only_right`, `value_mismatch`, `matched`), scalable to 100M keys, and
   cheap to run nightly. *Approach:* compare three designs — `FULL OUTER JOIN` on the key
   (column-wise comparison, fan-out risk), `EXCEPT` both directions (whole-row, no
   column attribution), and a `UNION ALL` "tall" fact with a source column plus conditional
   aggregation (needs a pre-aggregation step). Explain which gives per-column attribution.

2. **Implement multiset reconciliation for two event loaders.** Given two tables with the same
   `(event_id, type, ts)` shape that should be identical, detect (a) missing copies, (b)
   extra copies, (c) differing multiplicity. *Approach:* `GROUP BY all columns` + `FULL JOIN`
   the counts, or `EXCEPT ALL`/`INTERSECT ALL` with count guards. State why the distinct
   operators are insufficient.

3. **Explain the performance cliff between `UNION` and `UNION ALL` at scale, with a
   measurement.** Generate 50M rows in two branches, compare wall time and memory for both,
   and identify the plan node responsible. *Approach:* `EXPLAIN (ANALYZE, BUFFERS)`, look for
   `HashAggregate`/`Unique` above the `Append`; report spill-to-disk if `work_mem` is
   insufficient. Then discuss the equivalent shuffle in a distributed engine.

4. **Rewrite an `EXCEPT`-based anti join as an anti join and decide which to ship.**
   *Approach:* measure both on indexed data; `EXCEPT` is usually fine and much more readable,
   but a hash anti join on an indexed key can use indexes that a whole-row hash cannot. Give a
   decision rule based on column count and index availability.

5. **Use `INTERSECT` to implement a "same set" assertion and show it is stronger than a count
   comparison.** *Approach:* `A INTERSECT B` = `A` and `B` are identical as sets; compare with
   `COUNT(*)` equality, which can hide substitutions. Then extend to multisets with
   `INTERSECT ALL` + count comparison.

6. **Build a schema-evolution-safe ingestion union.** Two sources where one has an extra
   column, and the column order differs. *Approach:* explicit column lists with `NULL::type`
   padding, or Snowflake/BigQuery `UNION ALL BY NAME`; explain why `SELECT *` is unsafe and how
   you would test it in CI (a column-order assertion on both sides).

7. **Port a set-operation query from Oracle 19c to PostgreSQL.** Identify every place where
   `MINUS` must become `EXCEPT`, where `ALL` variants become available, and where precedence
   differences could change results. *Approach:* diff the operator support table in §8, then
   parenthesize everything and back the port with symmetric-difference tests.

8. **Design a "no-op detection" check for a nightly load**: prove that re-running the pipeline
   produces zero row changes, using set operations only. *Approach:* snapshot the derived
   table, re-run, and compare with `EXCEPT`/`EXCEPT ALL` both ways; discuss cost (full table
   comparison) and cheaper alternatives (row counts per partition, checksums).

---

## References

**PostgreSQL**
- Combining queries (`UNION`, `INTERSECT`, `EXCEPT`, precedence, `ALL` variants): https://www.postgresql.org/docs/current/queries-union.html
- `SELECT` (set-operation syntax): https://www.postgresql.org/docs/current/sql-select.html

**MySQL**
- Set operations with `UNION`, `INTERSECT`, `EXCEPT` (8.0.31+): https://dev.mysql.com/doc/refman/8.0/en/set-operations.html
- MySQL 8.0.31 release notes (introducing `INTERSECT`/`EXCEPT`): https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-31.html

**SQL Server**
- Set operators `EXCEPT` and `INTERSECT`: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/set-operators-except-and-intersect-transact-sql
- `UNION` (Transact-SQL): https://learn.microsoft.com/en-us/sql/t-sql/language-elements/set-operators-union-transact-sql

**Oracle**
- Set operators (`UNION [ALL]`, `INTERSECT`, `MINUS`): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Set-Operators.html
- Set operator enhancements in 21c (`EXCEPT`, `EXCEPT ALL`, `MINUS ALL`, `INTERSECT ALL`): https://oracle-base.com/articles/21c/sql-set-operator-enhancements-21c

**Snowflake / BigQuery**
- Snowflake set operators (`INTERSECT`, `MINUS`/`EXCEPT`, `UNION [ALL] [BY NAME]`, precedence): https://docs.snowflake.com/en/sql-reference/operators-query
- BigQuery query syntax (`UNION`, `INTERSECT DISTINCT`, `EXCEPT DISTINCT`): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Oracle Blog, *Intersect and Except in MySQL 8.0* (the practical workarounds for older versions): https://blogs.oracle.com/mysql/intersect-and-except-in-mysql-80
- Modern SQL, *Three-valued logic* (why NULL-safe comparison in set operators looks inconsistent with `=`): https://modern-sql.com/concept/three-valued-logic

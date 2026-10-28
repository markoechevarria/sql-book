# EXISTS, IN, ANY, ALL and Subqueries

> Almost every "the query returned nothing and I don't know why" incident in SQL is either a
> `NULL` comparison or a `NOT IN`. This chapter covers subquery semantics end to end: scalar,
> row and table subqueries; correlated vs uncorrelated; the *semi* and *anti* join reading of
> `IN`/`EXISTS`; the three-valued-logic trap that makes `NOT IN` return an empty set; and
> `ANY`/`ALL` quantification. You will leave knowing exactly which of the four ways to ask
> "customers without orders" to use, and why.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)).

---

## TL;DR

- Subqueries come in three shapes: **scalar** (one value), **row** (one row of values) and
  **table** (a derived table in `FROM`). They can be **uncorrelated** (evaluated once) or
  **correlated** (evaluated per outer row — see [Chapter 07](./07-correlated-subqueries.md)).
- `IN (subquery)` and `EXISTS (subquery)` are both **semi joins**: "does at least one match
  exist?" — they never duplicate the left row and they stop at the first match.
- `NOT EXISTS` and `LEFT JOIN ... WHERE right.key IS NULL` are **anti joins**.
- **`NOT IN (subquery)` is the classic trap.** If the subquery returns even one `NULL`, the
  predicate becomes `UNKNOWN` for every non-matching row, and the query returns **zero rows**.
  Use `NOT EXISTS` (or `NOT IN` over a provably `NOT NULL` column).
- `x = ANY (subquery)` ≡ `x IN (subquery)`. `x <> ALL (subquery)` ≡ `x NOT IN (subquery)`.
  `ANY`/`ALL` are the general quantification forms and are also how you compare against an
  array (`x = ANY(ARRAY[1,2,3])`).
- **Empty-set algebra:** `x = ANY (∅)` is `FALSE`; `x <> ALL (∅)` (`NOT IN ∅`) is `TRUE`.
  `EXISTS (∅)` is `FALSE`; `NOT EXISTS (∅)` is `TRUE`.
- A **scalar subquery returning more than one row is a runtime error** in every engine. Guard
  with `LIMIT 1`, `MAX`, or an aggregate.
- A **correlated scalar subquery in the `SELECT` list executes once per outer row.** That is
  sometimes fine (indexed, tiny) and sometimes the single biggest performance bug in the
  query. `LATERAL`/joins or window functions are the usual fix.
- Prefer **`EXISTS` over `IN`** when the subquery is large: existence is a stop-at-first-hit
  test, while `IN` semantics conceptually require the full set (optimizers usually unify
  them, but `EXISTS` is more robust across plans and handles NULLs correctly by construction).

---

## 1. Definitions and core concepts

### 1.1 Taxonomy of subqueries

| Shape | Appears in | Returns | Example |
|---|---|---|---|
| **Scalar** | anywhere an expression is allowed | exactly 0 or 1 value (0 → `NULL`) | `(SELECT MAX(unit_price) FROM products)` |
| **Row** | comparisons | 0 or 1 row, multiple columns | `(a, b) = (SELECT x, y FROM t)` |
| **Table (derived)** | `FROM`/`JOIN` | a relation | `FROM (SELECT …) t` |
| **Semi** | `WHERE EXISTS` / `IN` | existence | `WHERE EXISTS (SELECT 1 …)` |
| **Anti** | `WHERE NOT EXISTS` / `NOT IN` | non-existence | `WHERE NOT EXISTS (SELECT 1 …)` |
| **Quantified** | `WHERE x op ANY/ALL (…)` | boolean | `WHERE price > ALL (SELECT …)` |

### 1.2 Correlated vs uncorrelated

```sql
-- UNCORRELATED: independent of the outer row → conceptually evaluated once
SELECT (SELECT COUNT(*) FROM products) AS product_count, c.full_name FROM customers c;

-- CORRELATED: references c.customer_id → evaluated per outer row
SELECT c.full_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS n_orders
FROM customers c;
```

The distinction matters for cost, not for correctness: uncorrelated subqueries are candidates
for one-time materialization (or a `InitPlan` in Postgres), correlated ones are candidates for
a nested-loop-shaped execution (or, if the optimizer is smart, **decorrelation** into a join
or a window).

### 1.3 Semi and anti joins

```
orders:     1001 c=1, 1002 c=1, 1003 c=2, …
customers:  1 Alice, 2 Bob, 3 Carol, …

SEMI  (EXISTS/IN)      → Alice, Bob, Carol?, Frank, …   (each customer AT MOST once)
ANTI  (NOT EXISTS)     → customers with no matching order
```

The defining property: **the output has the same cardinality as (a subset of) the left input.**
That is what makes semi/anti joins the correct tools for existence questions — unlike
`JOIN ... DISTINCT`, they cannot fan out and cannot require a de-duplication sort.

### 1.4 Vocabulary

| Term | Meaning |
|---|---|
| **`InitPlan`** (Postgres) | An uncorrelated subplan computed once per query execution. |
| **`SubPlan`** (Postgres) | A correlated subplan, typically executed per outer row. |
| **`Hash Semi Join` / `Hash Anti Join`** | Physical plan for semi/anti joins with an equi-correlation. |
| **Decorrelation** | The optimizer's rewrite of a correlated subquery into a join/group-by. |
| **`Hashed SubPlan`** | Postgres materializes an `IN`/`ANY` list into a hash for fast membership tests. |
| **Three-valued logic (3VL)** | `TRUE` / `FALSE` / `UNKNOWN`; `UNKNOWN` propagates and behaves like "not TRUE" in `WHERE`. |

---

## 2. Why it matters / what problem it solves

- **Existence questions dominate analytics**: "customers who bought X", "sessions that
  converted", "products never returned", "orders missing an invoice".
- **`NOT IN` is the most common silent-wrong-answer bug in SQL.** It is worth internalizing
  the three-valued-logic explanation once, permanently.
- **`ANY`/`ALL`** are the readable way to express "any of these" / "greater than all of
  these", and the only way to compare against arrays in Postgres.
- **Scalar subqueries** are the natural way to attach a single aggregate to a row set without
  a join (`(SELECT SUM(...))`), and the natural way to introduce accidental N+1 execution.
- **Set-based thinking**: `NOT EXISTS` is not just "safer than `NOT IN`" — it is the
  idiomatic anti join, and it tells the optimizer exactly what you mean.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 `EXISTS` semantics

`EXISTS (subquery)` is `TRUE` if the subquery produces **at least one row**. It:

- ignores the *values* of the projected columns entirely (`SELECT 1`, `SELECT *`, `SELECT
  NULL` are all equivalent — optimize for clarity with `SELECT 1`);
- short-circuits: the engine may stop at the first matching row;
- never returns `NULL`: `EXISTS` is always `TRUE` or `FALSE`;
- can correlate on nullable columns safely, because it never tests equality for a result —
  it tests row *production*.

### 3.2 `IN` semantics and the NULL trap

`x IN (a, b, c)` ≡ `x = a OR x = b OR x = c`, evaluated with three-valued logic.

| Expression | Result |
|---|---|
| `3 IN (1, 2, 3)` | `TRUE` |
| `3 IN (1, 2)` | `FALSE` |
| `3 IN (1, 2, NULL)` | `UNKNOWN` (3 ≠ 1, 3 ≠ 2, 3 = NULL is UNKNOWN → OR yields UNKNOWN) |
| `NULL IN (1, 2)` | `UNKNOWN` |
| `3 NOT IN (1, 2, NULL)` | `UNKNOWN` → **row filtered out** |
| `1 NOT IN (1, 2, NULL)` | `FALSE` |
| `3 NOT IN (SELECT y FROM t)` where `y` contains any NULL | `UNKNOWN` for every non-matching row → the query returns **only** rows that *do* match something in the list |

That last row is the bug. Concretely:

```sql
-- order_items.product_id is NOT NULL in our schema, so this is safe.
SELECT c.full_name FROM customers c
WHERE c.customer_id NOT IN (SELECT o.customer_id FROM orders o);

-- But if the correlated column were nullable, this returns ZERO rows:
SELECT COUNT(*) FROM customers c
WHERE c.customer_id NOT IN (SELECT e.customer_id FROM events e);  -- events.customer_id IS nullable!
```

`events.customer_id` is nullable in the sample schema, so run it and observe: **0 rows**, even
though customers 3, 4, 5, 6 and 7 have no events. The fix is one of:

```sql
-- 1) NOT EXISTS (recommended)
SELECT c.full_name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM events e WHERE e.customer_id = c.customer_id);

-- 2) NOT IN with an explicit NULL filter (works, but easy to forget)
SELECT c.full_name FROM customers c
WHERE c.customer_id NOT IN (SELECT e.customer_id FROM events e WHERE e.customer_id IS NOT NULL);

-- 3) LEFT JOIN ... IS NULL (works, but only if the join key is NOT NULL on the right side —
--    actually it works even with NULL keys, because a NULL key never matches)
SELECT c.full_name FROM customers c
LEFT JOIN events e ON e.customer_id = c.customer_id
WHERE e.event_id IS NULL;
```

`> Senior note`: some optimizers (PostgreSQL, for instance) will rewrite `NOT IN` into a hash
anti join **only when the subquery output column is provably non-nullable** (e.g. it comes from
a `NOT NULL` column with no outer joins). When it cannot prove that, the plan stays a
row-by-row `SubPlan` — so `NOT IN` can be both wrong *and* slow. `NOT EXISTS` has neither
problem.

### 3.3 `ANY` / `SOME` / `ALL` semantics

| Form | Equivalent | Meaning |
|---|---|---|
| `x = ANY (subquery)` | `x IN (subquery)` | true if any element equals `x` |
| `x <> ALL (subquery)` | `x NOT IN (subquery)` | true if every element differs from `x` |
| `x > ALL (subquery)` | — | `x` greater than the maximum |
| `x > ANY (subquery)` | `x > MIN(subquery)` | `x` greater than the minimum |
| `x < ALL (subquery)` | `x < MIN(subquery)` | below the minimum |
| `x < ANY (subquery)` | `x < MAX(subquery)` | below the maximum |

Empty-set algebra (worth memorizing):

- `x = ANY (∅)` → `FALSE`
- `x <> ALL (∅)` → `TRUE`  (this is why `NOT IN (empty subquery)` returns everything)
- `x > ALL (∅)` → `TRUE`
- `x > ANY (∅)` → `FALSE`

`SOME` is a synonym for `ANY`. Provably-nullable elements make any of these `UNKNOWN` rather
than `FALSE`, exactly as with `IN`.

### 3.4 How the planner executes these (Postgres-centric, but generic)

| Written | Typical plan |
|---|---|
| `WHERE x IN (1,2,3)` | `Filter: x = ANY('{1,2,3}')`, index-usable |
| `WHERE x IN (SELECT y FROM t)` | `Hash Semi Join` (or `Nested Loop Semi Join`) |
| `WHERE EXISTS (SELECT 1 FROM t WHERE t.y = x)` | `Hash Semi Join` / `Nested Loop Semi Join` |
| `WHERE NOT EXISTS (…)` | `Hash Anti Join` / `Nested Loop Anti Join` |
| `WHERE x NOT IN (SELECT y FROM t)` with nullable `y` | `SubPlan` + `Filter` (row-by-row, and semantics may be wrong) |
| `(SELECT SUM(…) FROM t WHERE t.y = x)` in `SELECT` | correlated `SubPlan` **or** decorrelated join+group by |

Reading `EXPLAIN` for "did it understand me?":

```
-- Good: the engine knows this is existence, not a join with fan-out
Hash Anti Join  (cost=… rows=…)
  Hash Cond: (o.customer_id = c.customer_id)

-- Suspicious: row-by-row evaluation
Filter: (NOT (SubPlan 1))
  SubPlan 1
    ->  Seq Scan on events e
```

### 3.5 Scalar subqueries

- Must return **at most one row**, at most one column. More than one row → runtime error
  (Postgres: `ERROR: more than one row returned by a subquery used as an expression`).
- Zero rows → `NULL` (not an error, and not `0`).
- **Nullable by construction**: always wrap arithmetic in `COALESCE` when you need a number.
- As a `WHERE` operand, they are evaluated once if uncorrelated.

Row subqueries (`(a, b) = (SELECT x, y FROM t)`) behave like scalar subqueries with tuple
comparison and support `IN` over tuples:

```sql
SELECT * FROM orders
WHERE (customer_id, status) IN (SELECT customer_id, 'delivered' FROM customers WHERE country_code = 'US');
```

---

## 4. Syntax and mechanics (with examples)

### 4.1 The four ways to ask the same existence question

```sql
-- 1) IN (semi join)
SELECT c.full_name FROM customers c
WHERE c.customer_id IN (SELECT o.customer_id FROM orders o WHERE o.status = 'delivered');

-- 2) EXISTS (semi join, recommended)
SELECT c.full_name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'delivered');

-- 3) ANY (equivalent to IN)
SELECT c.full_name FROM customers c
WHERE c.customer_id = ANY (SELECT o.customer_id FROM orders o WHERE o.status = 'delivered');

-- 4) JOIN + DISTINCT (works, but pays for de-duplication)
SELECT DISTINCT c.full_name
FROM customers c JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'delivered';
```

All four return the same rows here. Prefer (2): it is NULL-safe, intent-revealing, and the
optimizer can use it as a semi join without a `DISTINCT`.

### 4.2 The four ways to ask the negation — and which one is correct

```sql
-- A) NOT EXISTS — always correct, never fans out (use this)
SELECT c.full_name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- B) LEFT JOIN ... IS NULL — correct, requires choosing a NOT NULL column of the right table
SELECT c.full_name FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;

-- C) NOT IN — correct ONLY if the subquery column is NOT NULL everywhere
SELECT c.full_name FROM customers c
WHERE c.customer_id NOT IN (SELECT o.customer_id FROM orders o);

-- D) EXCEPT (set difference) — correct and readable, materializes both sides
SELECT c.full_name FROM customers c
EXCEPT
SELECT c.full_name FROM customers c JOIN orders o ON o.customer_id = c.customer_id;
```

Expected result for all four on the seed: exactly one row — `Heidi Novak`.

**Do not** use `WHERE NOT EXISTS` with an `OR` inside the correlation unless you understand
the plan; and **never** compare `COUNT(*) = 0` from a left join as a proxy for `NOT EXISTS`
unless you must (correct, but it forces the join to complete).

### 4.3 `ANY`/`ALL` beyond `IN`

```sql
-- Products priced above EVERY product in the Cases category
SELECT p.name, p.unit_price FROM products p
WHERE p.unit_price > ALL (SELECT c.unit_price FROM products c
                          JOIN categories cat ON cat.category_id = c.category_id
                          WHERE cat.name = 'Cases');

-- Products priced above AT LEAST ONE product in Cases (== above the minimum)
SELECT p.name, p.unit_price FROM products p
WHERE p.unit_price > ANY (SELECT c.unit_price FROM products c
                          JOIN categories cat ON cat.category_id = c.category_id
                          WHERE cat.name = 'Cases');

-- Compare against an array (PostgreSQL)
SELECT * FROM products WHERE product_id = ANY (ARRAY[1, 4, 12]);
SELECT * FROM products WHERE product_id <> ALL (ARRAY[1, 4, 12]);   -- "none of these"
```

Expected: the first returns all products priced above 29.99 (i.e. essentially all phones,
laptops, and the 89.00/59.00/45.00 kitchen items); the second returns everything above 19.99.

### 4.4 Scalar subqueries: attaching a single aggregate

```sql
SELECT o.order_id,
       (SELECT SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100))
        FROM order_items i WHERE i.order_id = o.order_id)  AS order_total,
       (SELECT MAX(o2.ordered_at) FROM orders o2
        WHERE o2.customer_id = o.customer_id)              AS last_order_at
FROM orders o
ORDER BY o.order_id;
```

Expected: `1001 → 1034.98` (999.00 + 2 × 19.99 × 0.9 = 999.00 + 35.98), `1002 → 38.97`,
`1003 → 899.00`, `1004 → 19.99`, `1005 → 134.00`, and so on.

Compare with the join+aggregate version (see §7 for when each is preferable):

```sql
SELECT o.order_id, SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS order_total
FROM orders o LEFT JOIN order_items i ON i.order_id = o.order_id
GROUP BY o.order_id ORDER BY o.order_id;
```

### 4.5 Guaranteeing scalar cardinality

```sql
-- FAILS at runtime if two products share the max price:
SELECT p.name FROM products p WHERE p.unit_price = (SELECT MAX(unit_price) FROM products);

-- SAFE: pick one deterministically
SELECT (SELECT p.name FROM products p ORDER BY p.unit_price DESC, p.product_id LIMIT 1) AS priciest;

-- SAFE: aggregate to a scalar
SELECT (SELECT MAX(unit_price) FROM products) AS max_price;
```

The `ORDER BY … LIMIT 1` form inside a scalar subquery is the correct idiom when the business
question means "a representative row".

### 4.6 `IN` with `VALUES` lists and tuples

```sql
-- A derived, typed constant set without a temp table
SELECT o.* FROM orders o
WHERE (o.country_code, o.channel) IN (
    VALUES ('US','web'), ('DE','web'), ('GB','marketplace')
);
```

This is the portable way to write multi-column `IN` tests without a lookup table.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: customers with at least one delivered order (`EXISTS`)

```sql
SELECT c.customer_id, c.full_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.customer_id AND o.status = 'delivered')
ORDER BY c.customer_id;
```

Expected: `1 Alice Johnson`, `2 Bob Smith`, `4 Dave Brown`, `6 Frank Miller`.
Note Bob appears once even though he has two orders — the semi join cannot fan out.

### Example 2 — Basic: products never sold *in a delivered order* (anti join)

Every SKU in the seed has been ordered at least once, so the naive "never sold at all" anti
join returns **zero rows**. The business question that actually has an answer is "which SKUs
have never shipped in a completed order?":

```sql
SELECT p.product_id, p.name
FROM products p
WHERE NOT EXISTS (
        SELECT 1
        FROM order_items i
        JOIN orders o ON o.order_id = i.order_id
        WHERE i.product_id = p.product_id
          AND o.status = 'delivered')
ORDER BY p.product_id;
```

Expected: **3 rows** — `3 Pixel 8` (only in order 1009, status `paid`),
`8 65W Charger` (orders 1006 `cancelled`, 1008 `shipped`),
`12 Electric Kettle` (order 1005 `paid`). This is the "not shipping" / assortment report.
Note that the correlated column here (`order_items.product_id`) is `NOT NULL`, so `NOT IN`
would also be correct — but the habit should be `NOT EXISTS` (see Example 3 for why).

### Example 3 — Intermediate: the NULL trap, demonstrated

```sql
-- events.customer_id is NULLABLE, and the seed contains two anonymous events
-- (customer_id IS NULL). This query is therefore WRONG: it returns 0 rows.
SELECT COUNT(*) AS customers_without_events_BUGGY
FROM customers c
WHERE c.customer_id NOT IN (SELECT e.customer_id FROM events e);

-- The same question asked safely
SELECT COUNT(*) AS customers_without_events_OK
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM events e WHERE e.customer_id = c.customer_id);
```

Expected: `BUGGY = 0`, `OK = 5` (customers 3, 4, 5, 6, 7 have no rows in `events`; customer 8
has page views but no orders — a nice reminder that "no events" and "no orders" are different
segments).

Also worth running, to see the third form behave:

```sql
SELECT COUNT(*) AS via_not_in_with_null_filter
FROM customers c
WHERE c.customer_id NOT IN (SELECT e.customer_id FROM events e WHERE e.customer_id IS NOT NULL);
```

Expected: `5` (same as `NOT EXISTS`).

### Example 4 — Intermediate: "bought a phone AND a laptop" (two semi joins)

```sql
SELECT c.customer_id, c.full_name
FROM customers c
WHERE EXISTS (
        SELECT 1 FROM orders o JOIN order_items i ON i.order_id = o.order_id
        JOIN products p ON p.product_id = i.product_id
        JOIN categories cat ON cat.category_id = p.category_id
        WHERE o.customer_id = c.customer_id AND cat.name = 'Phones')
  AND EXISTS (
        SELECT 1 FROM orders o JOIN order_items i ON i.order_id = o.order_id
        JOIN products p ON p.product_id = i.product_id
        JOIN categories cat ON cat.category_id = p.category_id
        WHERE o.customer_id = c.customer_id AND cat.name = 'Laptops');
```

Expected: customers 1 (Alice: iPhone in 1001, no laptop) → no; 4 (Dave: MacBook in 1007 → yes
for Laptops, but no Phones) → no; 6 (Frank: ThinkPad in 1012, no phone) → no. So the answer is
**zero rows** — which is a fine result for the seed but demonstrates the pattern. (Add an order
for customer 1 with a laptop to see a row.)

**Key point:** you cannot express this with a single join — a single join would require one
row to match both categories at once. Two semi joins is the correct formulation, and each one
is independent and index-friendly.

### Example 5 — Intermediate: `ANY`/`ALL` for outlier detection

```sql
-- Orders bigger than every delivered order from Germany (i.e. bigger than the DE max)
WITH order_totals AS (
    SELECT o.order_id, o.country_code,
           SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS total
    FROM orders o JOIN order_items i USING (order_id)
    GROUP BY o.order_id, o.country_code
)
SELECT order_id, country_code, total
FROM order_totals
WHERE total > ALL (SELECT total FROM (
        SELECT o2.order_id, SUM(i2.quantity * i2.unit_price) AS total
        FROM orders o2 JOIN order_items i2 ON i2.order_id = o2.order_id
        WHERE o2.country_code = 'DE' GROUP BY o2.order_id) de)
ORDER BY total DESC;
```

Expected (DE orders are 1003 = 899.00 and 1004 = 19.99 → the DE maximum is 899.00): every
order whose total exceeds 899.00, i.e. `1001` (1034.98), `1007` (2028.99), `1012` (1499.00)
and `1014` (999.00) — four rows. `1009` (799.00) is *not* returned, which is the point:
`> ALL` is "greater than the maximum of the set", not "greater than some of it".

### Example 6 — Advanced: semi join + scalar subquery + `ALL` in one report

```sql
SELECT c.full_name,
       c.country_code,
       (SELECT COUNT(DISTINCT o.order_id)
        FROM orders o
        WHERE o.customer_id = c.customer_id
          AND o.status IN ('paid','shipped','delivered'))        AS live_orders,
       (SELECT COUNT(DISTINCT i.product_id)
        FROM orders o JOIN order_items i ON i.order_id = o.order_id
        WHERE o.customer_id = c.customer_id)                     AS distinct_products
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)   -- has ordered
  AND NOT EXISTS (SELECT 1 FROM orders o                                    -- has no returns
                  WHERE o.customer_id = c.customer_id AND o.status = 'returned')
ORDER BY live_orders DESC, c.full_name;
```

Expected: Bob (id 2) is excluded because order 1004 was returned. Carol appears with
`live_orders = 1` (order 1005 is `paid`; 1006 `cancelled` is not in the live set). Frank has 2
live orders and 3 distinct products. Alice has 2 live orders (1001 and 1002 delivered; 1014 is
`pending`). Note the two independent semi/anti joins: the first keeps customers who ordered at
all, the second removes customers who returned anything.

### Example 7 — Advanced: rewriting `IN` semantics with a `NOT NULL` guarantee

```sql
-- The business wants "customers whose loyalty tier is one of the tiers used by a US customer"
-- Naive version — WRONG when any tier is NULL (customer 8 has a NULL tier)
SELECT DISTINCT c2.full_name, c2.loyalty_tier
FROM customers c2
WHERE c2.loyalty_tier IN (SELECT c1.loyalty_tier FROM customers c1 WHERE c1.country_code = 'US');

-- Correct: exclude NULLs explicitly, then decide policy for NULL-tier customers
SELECT DISTINCT c2.full_name, c2.loyalty_tier
FROM customers c2
WHERE EXISTS (SELECT 1 FROM customers c1
              WHERE c1.country_code = 'US'
                AND c1.loyalty_tier IS NOT DISTINCT FROM c2.loyalty_tier);
```

The first version silently drops customer 8 from the *result* even if you wanted "NULL tier
matches NULL tier". `IS NOT DISTINCT FROM` (Postgres/Snowflake) or MySQL's `<=>`
NULL-safe-equals makes the intent explicit. SQL Server has neither — use
`(a = b OR (a IS NULL AND b IS NULL))`, or `INTERSECT`.

### Example 8 — Advanced: `EXISTS` vs `IN` vs join — plan reading

```sql
EXPLAIN (ANALYZE, COSTS)
SELECT c.full_name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

EXPLAIN (ANALYZE, COSTS)
SELECT c.full_name FROM customers c
WHERE c.customer_id IN (SELECT o.customer_id FROM orders o);

EXPLAIN (ANALYZE, COSTS)
SELECT DISTINCT c.full_name FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;
```

On the seed all three are instant. On production data the third one is the only one that
*must* materialize the join plus a `HashAggregate`/`Unique` to remove duplicates — that is the
structural difference, independent of dataset size: **semi joins are bounded by the left input;
joins are bounded by matched pairs.**

---

## 6. Real-world use cases and applications

| Use case | Idiom |
|---|---|
| **Churn/win-back segments** | `NOT EXISTS (orders in last 90 days)` |
| **Funnel step completion** | `EXISTS (events of type X in session)` |
| **Never-sold inventory / dead SKUs** | `NOT EXISTS (order_items)` |
| **Referential integrity checks** | `NOT EXISTS` from child to parent (orphan detection) |
| **Incremental loads ("only new rows")** | `NOT EXISTS` against the target table |
| **De-duplicating staging data** | `NOT EXISTS` on a natural key with a tie-break |
| **Access control** | `EXISTS (SELECT 1 FROM acl WHERE acl.user = me AND acl.obj = t.id)` |
| **Outlier/benchmark comparisons** | `> ALL (…)`, `< ANY (…)` |
| **Feature flags / membership** | `= ANY (ARRAY[...])` in Postgres |
| **"Diff two populations"** | `EXCEPT` (set difference) — see [Chapter 09](./09-set-operations.md) |
| **Data-quality: missing FK parents** | `LEFT JOIN … IS NULL` or `NOT EXISTS` |

---

## 7. Performance notes and anti-patterns

### 7.1 Semi/anti joins are cheap by construction

A semi join can stop at the first match per left row; an anti join can stop when it finds one
(and otherwise must prove absence, which requires a full scan of the relevant partition — this
is why anti joins are inherently more expensive than semi joins). Both need an index on the
*correlated* column of the right table for the nested-loop form, or a hash build for the
hashed form.

| Right-side size | Best plan | What to ensure |
|---|---|---|
| tiny (a few hundred rows) | hashed subplan / hash semi join | nothing |
| indexed and filtered | nested-loop semi join | index on the correlation column |
| huge | hash semi/anti join | adequate `work_mem`, statistics |

### 7.2 Correlated scalar subqueries in `SELECT`

```sql
-- Executes the subquery once per outer row: 1 + N lookups
SELECT c.customer_id,
       (SELECT SUM(i.quantity * i.unit_price) FROM orders o
        JOIN order_items i ON i.order_id = o.order_id
        WHERE o.customer_id = c.customer_id) AS ltv
FROM customers c;

-- Usually better: one grouped scan joined in
WITH ltv AS (
    SELECT o.customer_id, SUM(i.quantity * i.unit_price) AS amount
    FROM orders o JOIN order_items i ON i.order_id = o.order_id
    GROUP BY o.customer_id
)
SELECT c.customer_id, COALESCE(l.amount, 0) AS ltv
FROM customers c LEFT JOIN ltv l USING (customer_id);
```

The first version is *fine* when the subquery is an indexed single-row lookup; it is
*catastrophic* when it is an unindexed aggregate. Modern optimizers decorrelate some of these
into a `LEFT JOIN` + `GROUP BY` (Postgres does this for many cases), but do not rely on it
blindly — inspect the plan. See [Chapter 07](./07-correlated-subqueries.md) for the full
treatment.

### 7.3 Anti-patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| `NOT IN (subquery)` on a nullable column | zero rows | `NOT EXISTS` |
| `NOT IN (1,2,NULL)` literal list | all rows filtered | remove NULLs from the list |
| `x IN (SELECT …)` where you need non-existence | wrong direction | `NOT IN`/`NOT EXISTS` — and mind the above |
| `COUNT(*) = 0` from a `LEFT JOIN` as existence | forces full join completion | `NOT EXISTS` |
| Scalar subquery expected to return one row but the data has duplicates | runtime error in production | `LIMIT 1` / aggregate |
| Scalar subquery returning `NULL` used in arithmetic | whole expression becomes `NULL` | `COALESCE` |
| `EXISTS (SELECT * FROM big)` | noise (no cost impact, but hurts review) | `SELECT 1` |
| `IN` with a 10,000-element literal list | parse/plan time blowup | a temp table or `VALUES` join |
| `EXISTS` with a non-sargable inner predicate (`WHERE lower(x) = y`) | index not used | expression index or normalized column |
| Correlated `EXISTS` on an unindexed column | quadratic plan | index the correlation column |

### 7.4 A verification recipe

Before shipping any existence query, run the symmetric-difference check with a second
formulation and confirm it is empty:

```sql
WITH a AS (
    SELECT c.customer_id FROM customers c
    WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id)),
     b AS (
    SELECT c.customer_id FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.customer_id WHERE o.order_id IS NULL)
SELECT * FROM ((SELECT * FROM a EXCEPT SELECT * FROM b)
               UNION ALL
               (SELECT * FROM b EXCEPT SELECT * FROM a));
```

Expected: zero rows. Make this a habit — two independent formulations agreeing is the cheapest
correctness evidence you will ever get.

---

## 8. Cross-engine differences

| Feature | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `EXISTS` / `NOT EXISTS` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `IN (subquery)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `SELECT`-list subqueries | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `= ANY` / `= SOME` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `<> ALL` / `> ALL` / `< ANY` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `x = ANY(array)` | ✅ native arrays | ❌ (use `JSON_CONTAINS`/`IN`) | ❌ | ❌ (use `MEMBER OF` / collection) | ✅ (`ARRAY_CONTAINS`) | ✅ (`x IN UNNEST(arr)`) |
| `NOT IN` NULL trap | present | present | present | present | present | present |
| NULL-safe equality | `IS NOT DISTINCT FROM` | `<=>` | ❌ (manual) | ❌ (`DECODE`) | `IS [NOT] DISTINCT FROM` | `IS NOT DISTINCT FROM` |
| Row/`VALUES`-tuple `IN` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Empty-set `NOT IN` → all rows | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Auto-rewrite `IN` → semi join | ✅ | ✅ (8.0.17+) | ✅ | ✅ | ✅ | ✅ |
| `MERGE`-friendly anti join for incremental loads | ✅ 15+ `MERGE` | ✅ 8.0.x `INSERT … ON DUPLICATE KEY` / `REPLACE` | ✅ `MERGE` | ✅ `MERGE` | ✅ `MERGE` | ✅ `MERGE` |
| Interval/temporal `EXISTS` performance | index-dependent | index-dependent | index-dependent | index-dependent | clustering helps | partition pruning helps |

Notes:

- The **`NOT IN` NULL trap exists in every engine** — it is a consequence of the SQL standard's
  three-valued logic, not an implementation quirk. Do not look for a workaround; use
  `NOT EXISTS`.
- **MySQL** historically could not rewrite `IN (subquery)` into a semi join; 8.0.17+ supports
  semi-join and anti-join optimizations explicitly. On MySQL 5.7, `EXISTS` is often
  dramatically faster than `IN (subquery)`.
- **BigQuery** is strict about correlated subqueries: they must be decorrelated by the
  optimizer or expressible as a join/`ARRAY` function. For per-row lookups, `LATERAL`/`UNNEST`
  or a pre-aggregated join is the idiom.
- **Snowflake** supports `IS [NOT] DISTINCT FROM`, which is the cleanest way to write
  anti joins on nullable keys.
- **Oracle**'s `DECODE` (with NULL = NULL semantics) is sometimes used to emulate NULL-safe
  comparison — remember that `CASE` does *not* behave that way.

---

## 9. Exercises — Intermediate (hints only)

1. **Customers who have never ordered** — write it three ways (`NOT EXISTS`, `LEFT JOIN …
   IS NULL`, `EXCEPT`) and confirm identical results. *Hint:* symmetric-difference check from
   §7.4.
2. **Products that have been sold exactly once.** *Hint:* compare a per-product count against
   `1` using a semi join (`EXISTS` on an aggregated subquery) — or a join with a `HAVING`.
3. **Orders that contain a product from more than one category.** *Hint:* aggregate per order
   with `COUNT(DISTINCT category_id) > 1` in a `HAVING` (not a semi join).
4. **Customers whose lifetime value is above every customer in `FR`.** *Hint:* `> ALL`.
5. **Find rows in `orders` whose `customer_id` has no matching customer** (orphan detection).
   *Hint:* `NOT EXISTS` against `customers`; then explain why you cannot rely on the FK
   constraint in a staging table loaded without constraints.
6. **Rewrite a query using `NOT IN (SELECT ...)` over nullable `events.customer_id` so it
   returns the same rows as `NOT EXISTS`.** *Hint:* add `WHERE e.customer_id IS NOT NULL`
   inside the subquery and explain what you changed semantically.
7. **List sessions that never converted** (no `purchase` event). *Hint:* per-session `EXISTS`
   on `purchase`; use `NOT EXISTS` at the session level.
8. **Use a scalar subquery to attach the overall average order total to every order row.**
   *Hint:* uncorrelated scalar subquery + `WHERE` filter comparing it.
9. **Return products whose price is greater than any two of the prices in `value_bands`.**
   *Hint:* `> ANY` — and note that `> ANY` is a minimum-based test, not "greater than two of".
   Re-read §3.3 and explain why the phrasing matters.
10. **Convert a 20-`OR` predicate into an `IN` over a `VALUES` list.** *Hint:* typed
    `VALUES` with a `WHERE` on a CTE; compare plans.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Prove the `NOT IN` trap with a minimal reproducer and a formal explanation.**
   *Approach:* construct a 3-row table with one NULL, run `x NOT IN` and `x <> ALL`, and derive
   the truth table showing `UNKNOWN` propagation. Then state the two conditions under which
   `NOT IN` is safe (column is `NOT NULL`, and no `NULL` can enter via an outer join).

2. **Incremental load: insert only new rows, then prove idempotency.**
   *Approach:* `INSERT … SELECT … WHERE NOT EXISTS (…)` on the natural key; run it twice and
   show the second run inserts zero rows. Then compare with `MERGE`/`INSERT … ON CONFLICT`
   and state the concurrency differences (the `NOT EXISTS` version can race).

3. **Optimize a quadratic correlated `EXISTS`.** Given an unindexed correlation column,
   measure the nested-loop plan, add the index, re-measure, then produce an alternative
   formulation with a pre-aggregated join and compare all three.
   *Approach:* `EXPLAIN (ANALYZE, BUFFERS)` before/after; discuss when a hash anti join beats
   a nested loop anti join.

4. **Deduplicate a stream with `EXISTS` and then with `ROW_NUMBER`, and compare contracts.**
   *Approach:* `NOT EXISTS` keeps the first row in insertion order and can race under
   concurrency; `ROW_NUMBER()` gives a deterministic winner per key. State which you would
   put in a dbt model and why.

5. **Express a "product affinity" question (A and B) that no single join can answer.** Then
   generalize to "bought 3 of the top 5 products". *Approach:* N independent semi joins, or a
   `HAVING COUNT(DISTINCT bucket) = N` over a bucketed join; discuss plan cost as N grows and
   how you would parameterize it (dynamic SQL or a bitmask aggregate).

6. **Reconcile two populations with `EXCEPT` in both directions and explain why it is not the
   same as a `FULL OUTER JOIN`.** *Approach:* symmetric difference; then extend to comparing
   *amounts* (where `EXCEPT` compares entire rows and a `FULL JOIN` can compare columns
   independently).

7. **Decide between `IN`, `EXISTS` and a `JOIN` for a 100M-row anti join on MySQL 5.7 vs
   MySQL 8.0.17+.** *Approach:* describe the pre- and post-semi-join-optimization plans and
   give the migration implication for the query you would write once and run on both.

8. **Design a CI test that catches `NOT IN` usage over nullable columns.** *Approach:* parse
   the SQL (sqlfluff/`pg_query` AST), detect `NOT IN (subquery)`, look up nullability from the
   catalog (`information_schema.columns.is_nullable` + join analysis), and fail the build.
   Explain why this is worth automating.

---

## References

**PostgreSQL**
- Subquery expressions (`EXISTS`, `IN`, `ANY`, `ALL`, `SOME`): https://www.postgresql.org/docs/current/functions-subquery.html
- `SELECT` (subqueries in `FROM`, `LATERAL`): https://www.postgresql.org/docs/current/sql-select.html
- Value expressions (scalar/row subqueries): https://www.postgresql.org/docs/current/sql-expressions.html

**MySQL**
- Subqueries (`IN`, `ANY`, `SOME`, `ALL`): https://dev.mysql.com/doc/refman/8.0/en/any-in-some-subqueries.html
- Comparison operators (including `<=>`): https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html

**SQL Server**
- `EXISTS`: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/exists-transact-sql
- `ANY`/`SOME`/`ALL`: https://learn.microsoft.com/en-us/sql/t-sql/language-elements/any-transact-sql

**Oracle**
- `SELECT` (subquery syntax and quantified predicates): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake `IS [NOT] DISTINCT FROM` and subquery support: https://docs.snowflake.com/en/sql-reference/operators-query
- BigQuery query syntax (subqueries, `IN`, `EXISTS`): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Modern SQL, *Three-valued logic* (the `NULL` semantics behind the `NOT IN` trap): https://modern-sql.com/concept/three-valued-logic
- Modern SQL, *IS DISTINCT FROM* (NULL-safe comparison) and its support matrix: https://modern-sql.com/feature/is-distinct-from
- Markus Winand, *The N+1 select problem* (why per-row lookups hurt): https://use-the-index-luke.com/sql/join/nested-loops-join-n1-problem

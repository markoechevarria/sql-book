# Recursive CTEs

> Real data is full of trees and graphs: category hierarchies, org charts, bill-of-materials,
> referral chains, dependency graphs, shortest paths. SQL's answer is the **recursive CTE** —
> a CTE that references itself and is evaluated by repeated iteration until no new rows are
> produced. It is the one place in SQL where you write an algorithm rather than a set
> operation, and the one place where a missing termination condition turns a report into an
> outage. This chapter covers the anchor/recursive-member contract, termination, path
> building, `SEARCH`/`CYCLE`, and how to debug the plan when recursion gets slow.

All examples run on the shared Acme Commerce schema
([01-complex-joins.md](./01-complex-joins.md)) — `categories.parent_id`, `employees.manager_id`
and `customers.referred_by` are three different hierarchies in the same schema.

---

## TL;DR

- A recursive CTE has two parts joined by `UNION [ALL]`: the **anchor** (non-recursive seed)
  and the **recursive member** (which references the CTE once).
- Semantics are **iterative, not magical**: the engine maintains a *working table* and a
  *result table*; each iteration computes the recursive member against the previous working
  table, appends new rows, and stops when the working table is empty.
- `UNION ALL` keeps duplicates and **never terminates on a cycle**. `UNION` (distinct)
  removes already-seen rows and therefore terminates on cycles — at the cost of hashing
  every row.
- `WITH RECURSIVE` is required in PostgreSQL, MySQL, BigQuery; SQL Server and Oracle use
  plain `WITH` (Oracle also offers `CONNECT BY`).
- **Every engine has an iteration limit or a loop error**: MySQL `cte_max_recursion_depth`
  (default 1000), SQL Server `MAXRECURSION` (default 100), BigQuery 500 iterations,
  PostgreSQL no default limit (you must impose your own), Oracle raises an error on a loop.
- Cycle-safe path building: carry a `path` array/string and test membership
  (`NOT (child_id = ANY(path))`), or use PostgreSQL 14+ / Db2 `CYCLE` clause.
- `SEARCH DEPTH FIRST | BREADTH FIRST BY ... SET ordercol` (PostgreSQL 14+, Db2) gives you
  deterministic traversal order without breaking the recursive structure with `ORDER BY`.
- **You cannot use aggregates, window functions, `DISTINCT`, `GROUP BY`, `ORDER BY` or
  `LIMIT` inside the recursive member** (PostgreSQL is explicit about this; SQL Server has a
  similar list). Aggregate in the *outer* query.
- Performance rule #1: **every FK used as the recursion edge needs an index**; rule #2:
  recursion is not parallelizable and does not use hash joins between iterations — keep the
  frontier small.

---

## 1. Definitions and core concepts

### 1.1 Anatomy

```sql
WITH RECURSIVE tree AS (
    ---------------- anchor member ----------------
    SELECT category_id, name, parent_id, 1 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    ---------------- recursive member -------------
    SELECT c.category_id, c.name, c.parent_id, t.depth + 1
    FROM categories c
    JOIN tree t ON c.parent_id = t.category_id
)
SELECT * FROM tree ORDER BY depth, name;
```

| Term | Meaning |
|------|---------|
| **Anchor (seed) member** | A normal query; must not reference the CTE. Can be `UNION`ed multiple times. |
| **Recursive member** | References the CTE exactly once in its `FROM`; is evaluated once per iteration. |
| **Working table** | The rows produced by the *previous* iteration — the input to the next iteration. |
| **Result table** | Accumulates all rows produced so far; this is what the outer query reads. |
| **Iteration** | Anchor produces working table `W0`; iteration `i` computes the recursive member against `W(i-1)` and sets `W(i)` to the new rows. |
| **Termination** | Stops when the recursive member produces zero new rows. |
| **Cycle** | A path that revisits a node; with `UNION ALL` and no guard it runs forever. |

### 1.2 Iteration semantics, precisely (PostgreSQL)

1. Evaluate the anchor. Result table `R := anchor`, working table `W := anchor`.
2. While `W` is not empty:
   a. Evaluate the recursive member with the CTE bound to `W`. Call the output `N`.
   b. With `UNION ALL`: `R := R ∪ N`. With `UNION`: `R := R ∪ (N \ R)` and duplicates inside
      `N` are removed too.
   c. `W := N'` where `N'` is the newly added rows (after deduplication if `UNION`).
3. Return `R` (then the outer query's `SELECT`, filters, joins, aggregates, ordering apply).

Two crucial consequences:

- **The outer query's `WHERE` does not constrain the recursion.** Filters on `depth` in the
  outer query prune the *output*, not the *work*. To prune the work you must filter inside
  the recursive member (e.g. `WHERE t.depth < 5`).
- **`ORDER BY` cannot appear inside the recursive member** (Postgres), so the returned rows
  are in an unspecified order. Either sort in the outer query (losing path order), or use
  `SEARCH ... SET ordercol` and order by that column.

### 1.3 Recursion vs `CONNECT BY` vs application-side loops

| Approach | Where | Pros | Cons |
|---|---|---|---|
| Recursive CTE | all major engines | standard, portable, composable with joins/`CASE`/windows in the outer query | one statement, iterative plan, hard to debug |
| `CONNECT BY PRIOR` | Oracle | concise, has `LEVEL`, `SYS_CONNECT_BY_PATH`, `NOCYCLE`, `ORDER SIBLINGS BY` | Oracle-only, no joins in the recursive condition shape you expect |
| Iterative loop (Python/SQL) | app / stored proc | easy to instrument | N round-trips, no set semantics, no transactional snapshot guarantees |
| Graph engine (Neo4j, PG `pgRouting`) | specialised | shortest path, weighted edges | another system to operate |

---

## 2. Why it matters / what problem it solves

1. **Hierarchies are everywhere.** Category trees (this schema), org charts (`manager_id`),
   referral trees (`referred_by`), comment threads, folder structures, dependency graphs.
   A `parent_id` column is a self-referencing FK and cannot be flattened with an ordinary
   join of fixed depth. You either know the depth in advance (fixed self-joins, brittle) or
   you recurse.
2. **Bill of materials.** Manufacturing and supply chain: a product = components = their
   components. Recursion gives exploded quantity and rolled-up cost.
3. **Graph reachability.** "Which users are downstream of this account?", "which services
   are transitively impacted by this outage?", "find the shortest hop count between A and B".
4. **Gap filling and date spine replacement.** Recursive CTEs can enumerate numbers/dates
   (though `generate_series` and modern `GENERATOR` functions are better).
5. **Access-control closure.** Flattening a role/permission hierarchy into a materialized
   closure table — a nightly job that is essentially one recursive CTE plus an `INSERT`.

---

## 3. How it works (execution semantics: how the engine evaluates it)

### 3.1 Visual

```
anchor ──► W0 ──► recursive(W0) ──► W1 ──► recursive(W1) ──► W2 ──► … ──► ∅
   │                                     │                    │
   └─────────────► R (accumulated result table) ◄──────────────┘
```

Each iteration is a separate scan of the base table joined to the previous frontier. That
is why:

- an index on the join column (`categories(parent_id)`) is mandatory;
- the number of iterations equals the depth (or the longest cycle-free path);
- the total work is roughly the number of *edges* in the traversed subgraph, times the cost
  of one lookup.

### 3.2 What the planner does (PostgreSQL)

`EXPLAIN` shows a **`Recursive Union`** node with two children: the anchor (top) and the
recursive term (bottom). Key details:

- `Recursive Union` cannot be executed in parallel.
- The working table is held in a tuplestore; with `UNION` the engine also hashes the whole
  result so far to deduplicate (memory growth, spill risk).
- The optimizer estimates the recursive term from the anchor's estimate, often badly — do
  not trust the row estimate on a `Recursive Union`.
- `EXPLAIN (ANALYZE)` prints the *total* time per node, i.e. the recursion's whole cost, not
  per-iteration.

### 3.3 Restrictions in the recursive member

PostgreSQL explicitly forbids, in the recursive term:

- aggregate functions (`SUM`, `COUNT`, …) and `GROUP BY`;
- window functions;
- `DISTINCT` / `UNION` inside the term (the `UNION`/`UNION ALL` *between* anchor and term is
  the only allowed one);
- `ORDER BY`, `LIMIT`, `OFFSET`;
- more than one reference to the recursive CTE, and any reference inside a subquery;
- placing the recursive reference on the nullable side of an outer join.

SQL Server forbids: aggregates, `TOP`, `DISTINCT`, `GROUP BY`, `HAVING`, scalar subqueries,
outer joins (in the recursive member), subqueries, and CTE self-reference in a subquery.

### 3.4 Termination limits per engine

| Engine | Default limit | How to change | Behavior when exceeded |
|---|---|---|---|
| PostgreSQL | none | add `WHERE depth < n` yourself | runs until completion (or OOM) |
| MySQL 8 | `cte_max_recursion_depth = 1000` | `SET SESSION cte_max_recursion_depth = 100000;` | `ERROR 3636 (HY000): Recursive query aborted after 1001 iterations` |
| SQL Server | `MAXRECURSION 100` | `SELECT ... FROM tree OPTION (MAXRECURSION 0)` | `Msg 530: The statement terminated. The maximum recursion 100 has been exhausted` |
| Oracle (recursive `WITH`) | none | self-imposed via `LEVEL`/depth predicate | loop raises ORA-32044 |
| Oracle (`CONNECT BY`) | none | `NOCYCLE`, `CONNECT_BY_ISCYCLE` | `ORA-01436: CONNECT BY loop in user data` |
| BigQuery | 500 iterations | not configurable | query fails |
| Snowflake | documented traversal limits apply | not configurable | query fails |

`> Senior note`: always add an explicit `WHERE depth < 1000` style guard **and** a cycle
test, even on engines that have a limit. When the recursion limit fires, the transaction
aborts — in a batch pipeline that can leave you with a partially loaded table and an
incident.

### 3.5 Cycle detection patterns

**Pattern A — path membership (portable).**

```sql
WITH RECURSIVE walk AS (
    SELECT c.category_id, c.parent_id, ARRAY[c.category_id] AS path, 1 AS depth
    FROM categories c WHERE c.parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id, w.path || c.category_id, w.depth + 1
    FROM categories c
    JOIN walk w ON c.parent_id = w.category_id
    WHERE NOT (c.category_id = ANY(w.path))     -- cycle guard
)
SELECT * FROM walk;
```

**Pattern B — `UNION` instead of `UNION ALL`.** Deduplicates visited rows globally, so a
cycle terminates. It also de-duplicates diamond-shaped DAGs (a node reachable by two paths
appears once) — which is either exactly what you want (reachability) or exactly what you do
not want (path enumeration).

**Pattern C — the standard `CYCLE` clause (PostgreSQL 14+, Db2).**

```sql
WITH RECURSIVE walk AS (
    SELECT category_id, parent_id FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id
    FROM categories c JOIN walk w ON c.parent_id = w.category_id
)
CYCLE category_id SET is_cycle USING path
SELECT * FROM walk WHERE NOT is_cycle;
```

`CYCLE` creates a boolean column (`is_cycle`) and a path column you can inspect; the engine
stops expanding a branch when the cycle column would repeat. The full form:

```
CYCLE <columns> SET <cycle_marker> USING <path_column>
      [ TO <true_value> DEFAULT <false_value> ]
```

### 3.6 `SEARCH` clause (PostgreSQL 14+, Db2)

```sql
WITH RECURSIVE walk AS (
    SELECT category_id, name, parent_id FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.name, c.parent_id
    FROM categories c JOIN walk w ON c.parent_id = w.category_id
)
SEARCH DEPTH FIRST BY name SET ordercol        -- or: SEARCH BREADTH FIRST BY name
SELECT * FROM walk ORDER BY ordercol;
```

- `SEARCH DEPTH FIRST BY <cols> SET <col>` builds an **array** you can `ORDER BY` to get
  depth-first, lexicographic-by-`<cols>` traversal.
- `SEARCH BREADTH FIRST BY <cols> SET <col>` builds a **record/row** (depth + the BY
  columns); ordering by it yields level-by-level order.

This is the standards-compliant way to get a deterministic traversal order — the engine
knows the ordering column and can keep the recursion itself untouched.

---

## 4. Syntax and mechanics (with examples)

### 4.1 Minimal template

```sql
WITH RECURSIVE <name> (<cols>) AS (
    <anchor>
    UNION [ALL]
    <recursive member referencing <name> once>
)
SELECT ... FROM <name> ...;
```

### 4.2 Depth/level counter and root attribution

```sql
WITH RECURSIVE d AS (
    SELECT category_id, name, parent_id, 1 AS depth, name::text AS root
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.name, c.parent_id, d.depth + 1, d.root
    FROM categories c JOIN d ON c.parent_id = d.category_id
)
SELECT depth, root, name FROM d ORDER BY root, depth, name;
```

### 4.3 Path as an array (PostgreSQL) or as a string (portable)

```sql
-- PostgreSQL: array path, then join to text for display
WITH RECURSIVE d AS (
    SELECT category_id, name, ARRAY[category_id] AS path
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.name, d.path || c.category_id
    FROM categories c JOIN d ON c.parent_id = d.category_id
)
SELECT array_to_string(path, ' > ') AS category_path, name FROM d ORDER BY path;

-- Portable-ish string path (works in MySQL/SQL Server/Snowflake/BigQuery)
-- … d.path || '/' || c.name   (Postgres/SQL Server 2012+/Snowflake/BigQuery)
-- … CONCAT(d.path, '/', c.name)   (MySQL)
```

### 4.4 Aggregating the label path (PostgreSQL `string_agg`, portable `GROUP_CONCAT`)

```sql
WITH RECURSIVE d AS (
    SELECT category_id, parent_id, name, 1 AS depth FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id, d.name || ' > ' || c.name, d.depth + 1
    FROM categories c JOIN d ON c.parent_id = d.category_id
)
SELECT category_id, depth, name AS full_path FROM d ORDER BY category_id;
```

### 4.5 Reverse traversal (leaf → root)

```sql
-- Ancestors of a given node: swap the join direction
WITH RECURSIVE ancestors AS (
    SELECT category_id, name, parent_id, 1 AS steps_up
    FROM categories WHERE category_id = 6          -- 'Chargers'
    UNION ALL
    SELECT c.category_id, c.name, c.parent_id, a.steps_up + 1
    FROM categories c JOIN ancestors a ON c.category_id = a.parent_id
)
SELECT * FROM ancestors ORDER BY steps_up;
```

Expected: `Chargers → Accessories → Electronics` (3 rows).

### 4.6 Multiple recursive references / mutually recursive CTEs

SQL standard allows several `WITH RECURSIVE` items where item *n* references item *n-1*;
PostgreSQL limits each recursive term to **one** self-reference. Mutual recursion
(A calls B, B calls A) is generally not supported in one statement — model it as a single
edge table with a shared recursion instead.

---

## 5. Worked examples (basic → intermediate → advanced) on the shared sample schema

### Example 1 — Basic: flatten the category tree

```sql
WITH RECURSIVE cat_tree AS (
    -- anchor: roots
    SELECT category_id, name, parent_id, 1 AS depth,
           name AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- recursive member: children of the current frontier
    SELECT c.category_id, c.name, c.parent_id, t.depth + 1,
           t.path || ' > ' || c.name
    FROM categories c
    JOIN cat_tree t ON c.parent_id = t.category_id
)
SELECT category_id, depth, path
FROM cat_tree
ORDER BY path;
```

Expected (8 rows):

```
1 | 1 | Electronics
4 | 2 | Electronics > Accessories
5 | 3 | Electronics > Accessories > Cases
6 | 3 | Electronics > Accessories > Chargers
3 | 2 | Electronics > Laptops
2 | 2 | Electronics > Phones
7 | 1 | Home
8 | 2 | Home > Kitchen
```

### Example 2 — Basic: org chart with indentation and level

```sql
WITH RECURSIVE org AS (
    SELECT employee_id, full_name, manager_id, title, 1 AS lvl,
           LPAD('', (1-1)*4) || full_name AS pretty
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.full_name, e.manager_id, e.title, o.lvl + 1,
           LPAD('', o.lvl*4) || e.full_name
    FROM employees e JOIN org o ON e.manager_id = o.employee_id
)
SELECT lvl, pretty, title FROM org ORDER BY pretty;
```

Expected (9 rows): `Ada Lovelace` (CEO) at level 1; `Grace Hopper` and `Alan Turing` at 2;
`Barbara Liskov`, `Margaret Hamilton` at 3; `Dennis Ritchie`, `Ken Thompson`, `Edgar Codd`,
`Katherine Johnson` at 4.

### Example 3 — Intermediate: subtree aggregation (revenue rolled up a category tree)

Roll the revenue of *all descendants* up to every ancestor:

```sql
WITH RECURSIVE
-- 1. leaf-level revenue per product's immediate category
prod_rev AS (
    SELECT p.category_id, SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)) AS revenue
    FROM order_items i
    JOIN orders  o ON o.order_id = i.order_id AND o.status IN ('paid','shipped','delivered')
    JOIN products p ON p.product_id = i.product_id
    GROUP BY p.category_id
),
-- 2. closure: every (ancestor, descendant) pair including self
closure AS (
    SELECT category_id AS ancestor_id, category_id AS descendant_id, 0 AS hops
    FROM categories
    UNION ALL
    SELECT cl.ancestor_id, c.category_id, cl.hops + 1
    FROM closure cl
    JOIN categories c ON c.parent_id = cl.descendant_id
)
SELECT a.name AS category,
       SUM(pr.revenue)                                    AS subtree_revenue,
       COUNT(DISTINCT cl.descendant_id)                   AS descendant_categories
FROM closure cl
JOIN categories a ON a.category_id = cl.ancestor_id
LEFT JOIN prod_rev pr ON pr.category_id = cl.descendant_id
GROUP BY a.name
ORDER BY subtree_revenue DESC NULLS LAST;
```

Expected: `Electronics` has the largest subtree revenue (it contains Phones, Laptops,
Accessories → Cases, Chargers), then `Home`, then `Accessories`, `Phones`, `Laptops`,
`Cases`, `Chargers`, `Kitchen`. This **closure-table** pattern is the standard way to answer
"subtree aggregates" and is exactly what a nightly job materializes into a
`category_closure` table for O(1) joins at query time.

### Example 4 — Intermediate: referral chain (multi-level marketing tree)

`customers.referred_by` is a self-FK. Build each customer's referral lineage:

```sql
WITH RECURSIVE referral AS (
    SELECT customer_id, full_name, referred_by, 1 AS depth,
           full_name AS lineage
    FROM customers WHERE referred_by IS NULL

    UNION ALL

    SELECT c.customer_id, c.full_name, c.referred_by, r.depth + 1,
           r.lineage || ' → ' || c.full_name
    FROM customers c JOIN referral r ON c.referred_by = r.customer_id
)
SELECT depth, lineage FROM referral ORDER BY lineage;
```

Expected lineage examples: `Alice Johnson → Bob Smith → Erin Davis` (depth 3),
`Alice Johnson → Carol White` (depth 2), `Dave Brown → Grace Lee` (depth 2).

Then answer "how many customers did Alice's network produce?":

```sql
WITH RECURSIVE referral AS (
    SELECT customer_id, referred_by, 1 AS depth FROM customers WHERE customer_id = 1
    UNION ALL
    SELECT c.customer_id, c.referred_by, r.depth + 1
    FROM customers c JOIN referral r ON c.referred_by = r.customer_id
)
SELECT COUNT(*) - 1 AS downstream_customers, MAX(depth) - 1 AS max_chain_depth FROM referral;
```

### Example 5 — Advanced: bill of materials (explosion with multiplied quantities)

The e-commerce schema has bundles in spirit: a "Kit" product composed of components. Model
it with a `bom(parent_product_id, component_product_id, qty)` table:

```sql
CREATE TABLE bom (
    parent_product_id  bigint NOT NULL REFERENCES products(product_id),
    component_product_id bigint NOT NULL REFERENCES products(product_id),
    qty numeric(10,3) NOT NULL,
    PRIMARY KEY (parent_product_id, component_product_id)
);
INSERT INTO bom VALUES (4, 5, 1), (4, 8, 1), (5, 9, 2), (4, 6, 1);  -- MacBook kit example
```

Explode it, multiplying quantities along the path and flagging cycles:

```sql
WITH RECURSIVE explosion AS (
    SELECT b.parent_product_id AS root_id,
           b.component_product_id AS component_id,
           b.qty                  AS qty_required,
           ARRAY[b.parent_product_id, b.component_product_id] AS path,
           1 AS level
    FROM bom b
    WHERE b.parent_product_id = 4                     -- the product being exploded

    UNION ALL

    SELECT e.root_id,
           b.component_product_id,
           e.qty_required * b.qty,                    -- multiply down the tree
           e.path || b.component_product_id,
           e.level + 1
    FROM explosion e
    JOIN bom b ON b.parent_product_id = e.component_id
    WHERE NOT (b.component_product_id = ANY(e.path))  -- cycle guard
      AND e.level < 20                                -- depth guard
)
SELECT p.name AS component, e.qty_required, e.level,
       array_to_string(e.path, '->') AS path
FROM explosion e
JOIN products p ON p.product_id = e.component_id
ORDER BY e.level, p.name;
```

Expected: level 1 = `ThinkPad X1` (qty 1), `65W Charger` (1), `Phone Case` (1);
level 2 = `USB-C Cable 2m` (2, through ThinkPad). Note the **multiplication of quantities**
is the whole point — this is the classic MRP (material requirements planning) query, and the
reason it is recursive is that quantity depends on the accumulated path, not just the edge.

**Rolled-up cost** with `SUM` in the *outer* query (never inside the recursive member):

```sql
WITH RECURSIVE explosion AS (
    SELECT b.parent_product_id AS root_id, b.component_product_id AS component_id,
           b.qty AS qty_required, 1 AS level
    FROM bom b WHERE b.parent_product_id = 4
    UNION ALL
    SELECT e.root_id, b.component_product_id, e.qty_required * b.qty, e.level + 1
    FROM explosion e JOIN bom b ON b.parent_product_id = e.component_id
    WHERE e.level < 20
)
SELECT p.name AS root_product,
       SUM(e.qty_required * c.cost) AS rolled_up_cost
FROM explosion e
JOIN products p ON p.product_id = e.root_id
JOIN products c ON c.product_id = e.component_id
GROUP BY p.name;
```

### Example 6 — Advanced: shortest path / reachability over a graph

Model a directed graph of "customer viewed product" transitions and find, per product, every
other product within 3 hops:

```sql
CREATE TABLE product_graph (src bigint, dst bigint, weight numeric);
INSERT INTO product_graph VALUES (1,6,0.5),(1,8,0.4),(6,8,0.9),(8,9,0.7),(9,3,0.2);

-- BFS with a visited set to avoid revisiting (UNION distinct dedupes globally)
WITH RECURSIVE reach AS (
    SELECT src AS origin, dst AS node, ARRAY[src,dst] AS path, 1 AS hops, weight AS cost
    FROM product_graph WHERE src = 1
    UNION ALL
    SELECT r.origin, g.dst, r.path || g.dst, r.hops + 1, r.cost + g.weight
    FROM reach r
    JOIN product_graph g ON g.src = r.node
    WHERE r.hops < 3
      AND NOT (g.dst = ANY(r.path))          -- simple paths only
)
SELECT node, MIN(hops) AS min_hops, MIN(cost) AS min_cost
FROM reach GROUP BY node ORDER BY min_hops, min_cost;
```

Note the pattern: **`MIN(...)` in the outer query**, because you cannot aggregate inside the
recursive member. For true shortest paths with weights, Dijkstra is expressed as a recursive
CTE with a `HAVING`-free, priority-less iteration and a final `MIN` — correct but
`O(V·E)`; use a graph engine or `pgRouting` when it matters.

### Example 7 — Advanced: cycle detection — three ways, one dataset

```sql
-- A corrupted hierarchy: 5's parent is 6 AND 6's parent is 5  (a 2-cycle)
UPDATE categories SET parent_id = 6 WHERE category_id = 5;
UPDATE categories SET parent_id = 5 WHERE category_id = 6;

-- (1) UNION ALL + depth guard: SAFE but includes duplicate cycle rows
WITH RECURSIVE t AS (
    SELECT category_id, parent_id, 1 AS depth FROM categories WHERE category_id = 5
    UNION ALL
    SELECT c.category_id, c.parent_id, t.depth + 1
    FROM categories c JOIN t ON c.parent_id = t.category_id
    WHERE t.depth < 10
)
SELECT * FROM t;                              -- returns 10 rows, alternating 5,6,5,6…

-- (2) path membership guard: terminates immediately, no duplicates
WITH RECURSIVE t AS (
    SELECT category_id, parent_id, ARRAY[category_id] AS path, 1 AS depth
    FROM categories WHERE category_id = 5
    UNION ALL
    SELECT c.category_id, c.parent_id, t.path || c.category_id, t.depth + 1
    FROM categories c JOIN t ON c.parent_id = t.category_id
    WHERE NOT (c.category_id = ANY(t.path))
)
SELECT * FROM t;                              -- returns 2 rows: 5 then 6, stops

-- (3) SEARCH/CYCLE (PostgreSQL 14+): declarative, inspectable
WITH RECURSIVE t AS (
    SELECT category_id, parent_id FROM categories WHERE category_id = 5
    UNION ALL
    SELECT c.category_id, c.parent_id
    FROM categories c JOIN t ON c.parent_id = t.category_id
)
CYCLE category_id SET is_cycle USING cycle_path
SELECT category_id, is_cycle, cycle_path FROM t;

-- Restore the good hierarchy afterwards
UPDATE categories SET parent_id = 4 WHERE category_id = 5;
UPDATE categories SET parent_id = 4 WHERE category_id = 6;
```

Run (1) and (2) and compare row counts — that gap is exactly the incident you would have had
in production.

### Example 8 — Advanced: date spine for gap filling (and why you should not do this)

```sql
-- Works everywhere, but generate_series / GENERATOR / a calendar table is better.
WITH RECURSIVE months AS (
    SELECT date '2024-01-01' AS month_start
    UNION ALL
    SELECT (month_start + interval '1 month')::date
    FROM months
    WHERE month_start < date '2024-03-01'
)
SELECT m.month_start,
       COALESCE(SUM(i.quantity * i.unit_price * (1 - i.discount_pct/100)), 0) AS revenue
FROM months m
LEFT JOIN orders o ON date_trunc('month', o.ordered_at)::date = m.month_start
                  AND o.status = 'delivered'
LEFT JOIN order_items i ON i.order_id = o.order_id
GROUP BY m.month_start
ORDER BY m.month_start;
```

Expected: three rows (January, February, March 2024) with non-zero revenue for each month
and `0` where a month had no delivered orders. Note the `LEFT JOIN` placement: the date
predicate must live in the `ON` clause, or you silently lose the zero-revenue months
(see [Chapter 01](./01-complex-joins.md) §3.2).

---

## 6. Real-world use cases and applications

| Use case | Recursion edge | Notes |
|---|---|---|
| **Category / taxonomy navigation** | `categories.parent_id` | Materialize the closure nightly for query-time joins. |
| **Org chart, span of control** | `employees.manager_id` | Include the root (`WHERE manager_id IS NULL`) or filter to a subtree. |
| **Referral / affiliate trees** | `customers.referred_by` | Guard with depth limits; referral fraud often has cycles. |
| **Bill of materials / MRP** | `bom.parent → component` | Multiply quantities down the path; cycles = data error. |
| **Where-used analysis** | reverse BOM | Same query, swapped direction. |
| **Dependency & impact analysis** | `service_depends_on` | "What breaks if `payments-api` is down?" |
| **Permission/RBAC closure** | `role_parent` | Flatten to a closure table for fast `EXISTS` checks. |
| **Comment threads** | `comments.parent_id` | Build indentation and order with `SEARCH DEPTH FIRST`. |
| **Route/path finding** | `edges(src,dst)` | BFS/DFS; watch out for exponential path enumeration. |
| **Session stitching without window functions** | `events.session_id` | Older pattern superseded by `SESSIONIZE`/gap-and-islands. |

---

## 7. Performance notes and anti-patterns

**The three rules**

1. **Index the edge column.** `CREATE INDEX ON categories (parent_id);`
   `CREATE INDEX ON employees (manager_id);` Without it, every iteration is a sequential
   scan; with it, every iteration is an index lookup per frontier row.
2. **Keep the frontier small.** Push filters inside the recursive member
   (`WHERE t.depth < 5`, `WHERE c.is_active`). A `WHERE` in the outer query only trims output.
3. **Do not re-scan the big table.** Join the recursive CTE to a *narrowed* base
   (a CTE with the needed columns only). The recursive member is executed once per level, so
   any wasted column/row is multiplied by depth.

**Cost model intuition**

```
cost ≈ Σ_over_iterations ( cost(scan base, restricted to frontier) )
     ≈ depth × cost(one indexed lookup) × |frontier|
```

Recursion is inherently serial: each level depends on the previous. That means:

- no parallelism across iterations (a 40-core warehouse gives you one core here);
- no hash join between levels (it is always a nested-loop-shaped lookup);
- memory grows with the frontier; with `UNION` it also grows with the whole result.

**Anti-patterns**

| Anti-pattern | Symptom | Fix |
|---|---|---|
| `UNION ALL` on cyclic data without a guard | query never finishes / aborts at the engine limit | path membership test, or `CYCLE` clause |
| Filtering depth only in the outer query | full traversal computed, then discarded | filter inside the recursive member |
| Aggregating inside the recursive member | syntax error (PG/SQL Server) or wrong results (some engines allow it with surprising semantics) | aggregate in the final `SELECT` |
| Recursing over a huge edge table to get a "reachability" answer | full traversal | precompute closure/bitsets, or use a graph store |
| Recursion for date/number generation | slower than `generate_series` | `generate_series`, `GENERATOR`, or a calendar dim |
| Recursion where one self-join suffices (known depth) | overhead of the iterative machinery | plain self-joins for depth ≤ 2–3 |
| Missing `ORDER BY` on `SEARCH` output | unstable ordering between runs | order by the `SEARCH`/`CYCLE` column |
| Re-running a closure CTE on every dashboard refresh | slow queries | materialize the closure table, refresh incrementally |

**Debugging a slow recursive CTE**

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH RECURSIVE ... ;
```

Look for: `Recursive Union` total time, the number of loops (PostgreSQL prints counts of
`actual rows` per node), `Rows Removed by Join Filter`, and whether the anchor is
over-broad (a huge anchor means a huge first frontier). Also check for a `Sort`/`HashAggregate`
from a `UNION` you did not need — switching to `UNION ALL` plus a path guard often halves
the cost.

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL 8 | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| Keyword | `WITH RECURSIVE` | `WITH RECURSIVE` | `WITH` (implicit) | `WITH` (implicit) | `WITH RECURSIVE` | `WITH RECURSIVE` |
| Anchor/recursive separator | `UNION [ALL]` | `UNION [ALL]` | `UNION ALL` (mostly) | `UNION ALL` | `UNION ALL` | `UNION ALL` |
| Iteration limit | none (self-imposed) | `cte_max_recursion_depth` (1000) | `MAXRECURSION` (100) | none; loop → error | engine traversal limit | 500 iterations |
| Raise the limit | add `WHERE depth < n` | `SET SESSION cte_max_recursion_depth` | `OPTION (MAXRECURSION 0)` | n/a | n/a | n/a |
| Cycle clause | ✅ `CYCLE ... SET ... USING ...` (14+) | ❌ | ❌ | ❌ (`CONNECT BY NOCYCLE` instead) | ❌ | ❌ |
| Search clause | ✅ `SEARCH DEPTH/BREADTH FIRST` (14+) | ❌ | ❌ | ✅ `CONNECT BY` + `ORDER SIBLINGS BY` | ❌ | ❌ |
| Array type for paths | ✅ `int[]`, `text[]` | ❌ (use JSON / `GROUP_CONCAT`) | ❌ (use `varchar(max)`) | ❌ (`SYS_CONNECT_BY_PATH`) | ✅ `ARRAY` / `VARIANT` | ✅ `ARRAY` |
| Aggregates in recursive member | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Alternate hierarchy syntax | — | — | — | ✅ `CONNECT BY PRIOR`, `LEVEL`, `SYS_CONNECT_BY_PATH`, `CONNECT_BY_ROOT`, `CONNECT_BY_ISCYCLE` | — | — |
| Recursion in a view | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Plan node name | `Recursive Union` | `Recursive` (CTE) | `Table Spool` / `Recursive CTE` | `RECURSIVE WITH` / `CONNECT BY` | `Recursive CTE` | iteration stages |

Notes:

- **Oracle** is the odd one out: besides recursive `WITH`, it has the legacy hierarchical
  query (`START WITH ... CONNECT BY PRIOR`), which is often faster and more concise for
  simple trees, with `LEVEL`, `SYS_CONNECT_BY_PATH`, `CONNECT_BY_ISCYCLE` and
  `ORDER SIBLINGS BY` built in.
- **MySQL** does not allow `LIMIT` inside the recursive member and does not support
  `SEARCH`/`CYCLE`; the portable idiom is a path column plus a `depth` guard.
- **BigQuery** requires the recursive CTE to be declared `WITH RECURSIVE` and limits the
  number of iterations; for very deep graphs, materialize the closure.
- **Snowflake** supports recursive CTEs but not `MATERIALIZED`/`SEARCH`/`CYCLE`; write
  results into a table for reuse.
- **SQL Server** requires statements that precede the recursive CTE to be terminated with a
  semicolon, and does not accept `MAXRECURSION` on a CTE that is part of a view/module.

---

## 9. Exercises — Intermediate (hints only)

1. **Produce a sorted, indented category tree using `SEARCH DEPTH FIRST`.** *Hint:* add
   `SEARCH DEPTH FIRST BY name SET ord` and `ORDER BY ord`.
2. **For each employee, list all ancestors' names in a single string.** *Hint:* start from
   each employee (not from the root) and prepend the parent each iteration.
3. **Count how many categories exist at each depth.** *Hint:* recursion to compute depth,
   then `GROUP BY depth` in the outer query.
4. **Find the deepest path in the category tree.** *Hint:* `ORDER BY depth DESC LIMIT 1` on
   the result — or an aggregate.
5. **List all customers whose referral chain reaches Alice (id 1), with depth ≤ 3.**
   *Hint:* reverse traversal from 1; filter `depth <= 4` inside the recursive member.
6. **Return the org subtree under `Barbara Liskov` (id 103) with indentation.** *Hint:*
   anchor = the specific employee, not the root; recursion goes downward.
7. **Detect and report cycles in `customers.referred_by`.** *Hint:* path array + `is_cycle`
   flag, or `UNION` (distinct) followed by a comparison of row counts.
8. **Build a numbered sequence from 1 to 100 with a recursive CTE, then with
   `generate_series`.** *Hint:* compare plans and times; explain the difference.

## 10. Exercises — Advanced (hints / expected approach, no full inline answers)

1. **Design a materialized closure table.** Write the DDL for `category_closure(ancestor_id,
   descendant_id, hops, is_self)` and the recursive `INSERT ... SELECT` that populates it
   from `categories`. Then (a) rewrite Example 3 as a plain join against the closure, and
   (b) describe the incremental refresh strategy when a category is reparented.
   *Approach:* self-inclusive anchor, one-hop recursive member; incremental = delete the
   affected subtree's closure rows and re-insert.

2. **Weighted shortest path.** Given `edges(src, dst, weight)` with positive weights, compute
   the cheapest path from a source to every node. Explain why the naive recursive CTE is
   `O(V·E)` and how you would keep it from enumerating exponentially many paths.
   *Approach:* carry accumulated cost + a path array for cycle avoidance; keep only the best
   cost per node per hop frontier and prune with `cost < best_so_far` (Dijkstra-ish
   relaxation); discuss vertex-centric (Pregel) alternatives.

3. **Cycle-safe BOM with quantity roll-up and cost aggregation in one statement.** Extend
   Example 5 so the output has one row per component with total required quantity (summed
   across all paths) and the rolled-up cost. *Approach:* explode with path + qty, then group
   by component in the outer query; add a `HAVING` cycle report or a second query to list
   detected cycles.

4. **Time-travel recursion.** Add `valid_from`/`valid_to` to `employees.manager_id` (SCD2)
   and write a recursive CTE that answers "who was X's manager chain on 2024-02-01?".
   *Approach:* join the recursion edge with a half-open interval predicate
   (`as_of >= valid_from AND as_of < COALESCE(valid_to,'infinity')`).

5. **Mutual recursion / two-hop expansion.** Show that a "A depends on B, B depends on A"
   schema can be traversed with a single edge table plus path guard, and prove that the
   naive two-CTE mutual recursion is not supported in PostgreSQL.
   *Approach:* build a union view of both adjacency directions conceptually, then recurse;
   cite the "one self-reference per recursive term" restriction.

6. **Optimize a depth-20 hierarchy at 100M rows.** Given a `parent_id` FK, propose and
   measure three strategies: (a) plain recursive CTE with an index on the FK, (b) BFS with
   batched working tables into temp tables, (c) precomputed closure. *Approach:* compare
   `EXPLAIN (ANALYZE, BUFFERS)`, and reason about `depth × frontier` work; state the
   crossover point in terms of read/write ratio.

7. **Gap-and-islands with recursion.** Given a table of `(id, date)` with gaps, produce
   contiguous date ranges using a recursive CTE, then solve the same problem with window
   functions (`date - ROW_NUMBER()`). Compare the two and explain which one a senior
   engineer would ship.
   *Approach:* recursion for the "expand each island" form; window functions for the
   grouping form — the window version is `O(n log n)` and usually preferred.

8. **Prove termination for every recursive CTE in your codebase.** Propose a lint/CI rule
   that requires either a `CYCLE` clause, an explicit `path` guard, or a documented
   `depth < N` predicate in every recursive CTE. Write the regex or sqlfluff rule sketch.

---

## References

**PostgreSQL**
- 7.8.2 Recursive Queries (`WITH RECURSIVE`, `SEARCH`, `CYCLE`): https://www.postgresql.org/docs/current/queries-with.html
- `SELECT` reference (syntax of `SEARCH` / `CYCLE`): https://www.postgresql.org/docs/current/sql-select.html
- `CREATE INDEX` (indexing the recursion edge): https://www.postgresql.org/docs/current/sql-createindex.html

**MySQL**
- `WITH` / recursive CTEs and `cte_max_recursion_depth`: https://dev.mysql.com/doc/refman/8.0/en/with.html

**SQL Server**
- Recursive queries using CTEs (with `MAXRECURSION`): https://learn.microsoft.com/en-us/sql/t-sql/queries/recursive-common-table-expression-transact-sql
- `WITH common_table_expression`: https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql

**Oracle**
- Hierarchical Queries (`CONNECT BY`, `LEVEL`, `NOCYCLE`, `SYS_CONNECT_BY_PATH`): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Hierarchical-Queries.html
- `SELECT` (recursive subquery factoring): https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html

**Snowflake / BigQuery**
- Snowflake `WITH`: https://docs.snowflake.com/en/sql-reference/constructs/with
- BigQuery `WITH RECURSIVE` (recursive CTEs, iteration limits): https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

**Essays and practical guides**
- Markus Winand, *CYCLE (recursion)* — cross-engine support matrix: https://modern-sql.com/caniuse/cycle_(recursion)
- Aiven, *Explore the new SEARCH and CYCLE features in PostgreSQL 14*: https://aiven.io/blog/explore-the-new-search-and-cycle-features-in-postgresql-14

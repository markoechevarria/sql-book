# Clustered vs Non-Clustered Indexes

> "Add an index" sounds like one idea, but it is actually two fundamentally different physical
> arrangements of the same rows. A **clustered index** *is* the table: the rows live inside the
> index's leaf pages, sorted by the clustering key, so the table has exactly one true order. A
> **non-clustered index** is a *side structure*: a small sorted map from key values to the place
> where the real row lives. Everything that follows — why a secondary index lookup costs two B-tree
> traversals instead of one, why `SELECT *` prunes the value of a covering index, why random
> primary keys fragment MySQL inserts, why SQL Server has "included" columns and Oracle has
> "index-organized tables" — is a direct consequence of which of these two you are looking at. If
> you internalize one idea from this chapter, make it this: **an index either contains the row or
> points to it, and the engine's cost model is built entirely around which case applies.**

## TL;DR

- A **heap** is an unordered bag of pages; rows are found via an index or a scan. A **clustered
  table / index-organized table (IOT)** stores rows inside the index leaves in key order.
- **PostgreSQL and Oracle heap tables** have *no* clustered index in the physical sense — only an
  optional one-time `CLUSTER`/`IOT` behavior. Their "primary key" is just a normal *unique* index.
- **MySQL InnoDB and SQL Server** *always* store the table in a clustered index (the primary key,
  or an auto-created key if none exists). Secondary indexes in both engines store the **clustered
  key**, not a physical row address.
- A **secondary (non-clustered) index** leaf entry → clustered key (InnoDB/SQL Server) or → `ctid`
  / `ROWID` (PostgreSQL/Oracle heap). When the query needs columns not in the index, the engine must
  do a **bookmark lookup / heap fetch** per matching row — the single biggest reason a plan that
  "uses an index" is still slow.
- **Clustered key choice is the highest-leverage schema decision in InnoDB and SQL Server.** Wide,
  random, or frequently-updated clustering keys bloat *every* secondary index and cause page splits.
- The cheap, portable fix for most heap-fetch pain is a **covering index** (see
  [Covering indexes](./03-covering-indexes.md)); the structural fix is the right clustering key.

---

## 1. Definitions and core concepts

### 1.1 The two things an index can be

Let's build the concept from zero. A relational table is a *logical* set of rows. Physically it can
be stored in two ways:

| Concept | What the leaf level of the index page holds | Who owns the "row" |
|---|---|---|
| **Heap + secondary indexes** | the *key* + a pointer (`ctid`, `ROWID`, or clustered key) | the heap owns the row |
| **Clustered index (index-organized)** | the *key* + **all remaining columns of the row** | the index owns the row |

That is the whole distinction. A table can have:

- **Zero or one** clustered arrangement (you cannot store the *same* rows sorted two ways at once —
  that would duplicate the data). SQL Server allows exactly one clustered index per table; InnoDB
  calls it "the clustered index"; Oracle calls the alternative an IOT.
- **Many** non-clustered (secondary) indexes.

### 1.2 Vocabulary

| Term | Engine(s) | Meaning |
|---|---|---|
| **Heap** | PostgreSQL, Oracle, SQL Server | A table stored unordered; rows identified by physical location. |
| **Clustered index** | SQL Server, InnoDB | The index whose leaves *are* the row data. |
| **Clustered table** | SQL Server, InnoDB, Oracle | A table physically stored in a clustered index. |
| **Index-organized table (IOT)** | Oracle | The Oracle name for a clustered table keyed by the primary key. |
| **Bookmark lookup / Key lookup** | SQL Server | Non-clustered index seek + clustered-key lookup to fetch missing columns. |
| **RID lookup (Row ID lookup)** | SQL Server (heaps) | Non-clustered index seek + heap row fetch by file:page:slot. |
| **`ctid`** | PostgreSQL | Physical tuple id `(block, offset)` — the "bookmark". |
| **`ROWID`** | Oracle | Physical row address (or logical for IOT). |
| **Clustering key** | all | The column(s) the rows are physically ordered by. |
| **Fill factor** | SQL Server, InnoDB (`FILLFACTOR`) | Free space left in leaf pages to absorb inserts without splitting. |

> **Senior note:** In PostgreSQL the `PRIMARY KEY` creates a **unique B-tree index**, *not* a
> clustered table. Many engineers coming from SQL Server/MySQL assume "primary key = clustered" —
> in Postgres it is not. `CLUSTER table USING index` reorders the heap *once*; it is not maintained
> on later inserts, so the clustering slowly decays. Plan accordingly.

### 1.3 How each maps keys to rows — the core diagram

**Heap layout (PostgreSQL / Oracle / SQL Server heap).** Rows land wherever there is free space;
the primary-key index is a separate structure that points into the heap.

```
  PRIMARY KEY INDEX (b-tree)                 HEAP (unordered pages)
  ┌───────────────────────────┐              ┌───────────────────────────────┐
  │ key 45 → ctid (7,3)       │  ───────────▶│ page 7                        │
  │ key 46 → ctid (2,11)      │      ┌──────▶│  (2,11) id=46, ...            │
  │ key 47 → ctid (402,1)     │      │    ┌─▶│ page 402: (402,1) id=47 ...   │
  │ key 48 → ctid (7,4)       │──┐   │    │  ├───────────────────────────────┤
  └───────────────────────────┘  └───┘    │  │ page 9: (9,1) id=312 ...      │
                                          │  └───────────────────────────────┘
     row order on disk: 46, 312, 45, 47 ... (insertion order, NOT key order)
```

**Clustered layout (InnoDB / SQL Server clustered / Oracle IOT).** The table *is* the b-tree; there
is no separate heap to fetch from.

```
  CLUSTERED INDEX = THE TABLE
  internal node:  [ 46 | 900 | 4500 ]
                   /      |       \
     leaf pages (row data lives here, in key order):
     ┌──────────────────────────┐ ┌──────────────────────────┐ ┌──────────────────────────┐
     │ id=45 qty=2 ...          │ │ id=46 qty=7 ...          │ │ id=900 qty=1 ...         │
     │ id=45? no (continues)    │ │ id=48 qty=3 ...          │ │ id=1002 ...              │
     └──────────────────────────┘ └──────────────────────────┘ └──────────────────────────┘
        ◀── leaf pages doubly linked ─────────────────────────────────────────────────▶
```

A **secondary index** in a clustered table points *not* to a page but to the clustered key:

```
  SECONDARY INDEX on (email)                   CLUSTERED INDEX (the table)
  ┌───────────────────────────────┐            ┌───────────────────────────────┐
  │ 'a@x.com'  → customer_id=45   │ ─────────▶ │ customer_id=45 → full row     │
  │ 'b@x.com'  → customer_id=71   │ ─────────▶ │ customer_id=71 → full row     │
  └───────────────────────────────┘            └───────────────────────────────┘
        "bookmark" = the primary key          one more b-tree descent per row
```

This is why a secondary lookup in InnoDB/SQL Server is **two traversals** (secondary, then
clustered), and why the clustered key must be *narrow* — its bytes are duplicated into every
secondary index entry.

---

## 2. Why it matters / what problem it solves

### 2.1 The problem: random I/O on row lookups

Without a clustered arrangement, a query like "give me the 100 most recent orders for customer 7"
that filters on a secondary index must perform, per matching row, a *random* page read into the
heap. At 100 rows that's 100 random I/Os; at 1,000,000 rows it's 1,000,000 — the classic
**non-clustered-index-and-row-lookup** cliff that dominates OLTP latency.

| The query wants… | Heap: cost | Clustered: cost |
|---|---|---|
| Row by primary key | 1 index descent + 1 heap fetch = **2 I/O** | 1 index descent = **1 I/O** |
| Range by primary key | index range + N heap fetches (scattered) | index range where **leaves are the rows** (sequential) |
| Row by secondary key | 1 secondary descent + 1 heap fetch | 1 secondary descent + 1 clustered descent |
| Top-N by clustering key | requires sort or index + fetch | already ordered, **zero extra sort** |

Clustering buys three things:
1. **One fewer step** for primary-key access (no heap indirection).
2. **Sequential range scans** at the leaf level — perfect locality on disk and in cache.
3. **Natural order** for `ORDER BY` / `BETWEEN` / `LIMIT` on the clustering key.

### 2.2 The cost of clustering

Nothing is free. Clustering imposes:

- **One order only.** You can cluster by `order_id` *or* by `(customer_id, created_at)` — not both.
- **Secondary index inflation.** Every secondary index repeats the clustered key.
- **Random-key page splits.** Inserting into the middle of a clustered index causes splits (file
  described in [B-tree indexes](./02-b-tree-indexes.md)).
- **Hot-page contention.** Concurrent inserts into the same leaf page serialize.

---

## 3. How it works internally

### 3.1 PostgreSQL: heap + `ctid`, no real clustering

A PostgreSQL primary key is a plain unique index. The leaf entry is `(key, ctid)` where `ctid` is
`(block number, line pointer)`.

```
idx_customers_pkey (b-tree leaf)
┌──────────────────────────────────────────┐
│ 45 → (block 7,  line ptr 3)               │
│ 46 → (block 2,  line ptr 11)              │
│ 47 → (block 402, line ptr 1)              │
└──────────────────────────────────────────┘
```

Because `ctid` is a *physical* address, an `UPDATE` that cannot be done in place creates a **new
tuple version elsewhere** and the index entries for old visible versions must be maintained (Postgres
uses HOT — Heap-Only Tuples — to avoid re-indexing when no indexed column changed; see the free-space
map reasoning in the MVCC chapter). `CLUSTER` reads the table in index order and rewrites it, after
which new inserts break the order again.

**Consequence:** In Postgres, a range scan ordered by primary key still costs one heap fetch per
row (unless the pages happen to be cached and roughly ordered). This is a real performance gap that
`pg_repack`/`CLUSTER` only partly closes.

### 3.2 MySQL InnoDB: the table *is* the clustered index

InnoDB always stores data in a clustered index. Rules:

1. If a `PRIMARY KEY` exists → that is the clustered index.
2. Else, the first `UNIQUE NOT NULL` index becomes the clustered index.
3. Else, InnoDB creates a hidden 6-byte `GEN_CLUST_ACCESS` row id.

A secondary index leaf entry is `(secondary_key…, <primary key columns>)`. When the query needs a
column not in the secondary index, InnoDB traverses the clustered index by that primary-key value —
the **bookmark lookup**. Autoincrement (`bigint`) primary keys give monotonically increasing inserts
(always append at the right edge) whereas `UUIDv4` primary keys insert into random leaves → splits
and a ~2–5× throughput loss on insert-heavy workloads (measured in the well-known **UUID as PK**
benchmarks, e.g. Peter Zaitsev's *"To UUID or not to UUID?"* and Karthik Appigatla's *"Store UUID in
an Optimized Way"*).

### 3.3 SQL Server: clustered or heap, plus `INCLUDE`

SQL Server lets you choose: a **heap** (no clustered index), or a **clustered index** — its leaves
are the rows. Its non-clustered indexes always store the **clustering key** as the row locator
(a RID `file:page:slot` for heaps). Since SQL 2005 you can add `INCLUDE (col…)` to a non-clustered
index: columns appended to *leaf* pages only, not used for ordering — the direct antidote to the
bookmark lookup. That is the subject of the covering-index chapter.

```sql
-- SQL Server: clustered PK, then a nonclustered index that pays for a key lookup
CREATE TABLE orders (
    order_id    BIGINT IDENTITY PRIMARY KEY CLUSTERED,
    customer_id BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    created_at  DATETIME2 NOT NULL,
    total_cents INT NOT NULL
);
CREATE NONCLUSTERED INDEX ix_orders_customer ON orders (customer_id);
-- SELECT status... WHERE customer_id=7  → seek ix_orders_customer + Key Lookup (order rows)
```

### 3.4 Oracle: heap tables vs IOTs

Oracle heap tables identify rows by `ROWID` (physical). An **IOT** stores rows in a primary-key
b-tree; secondary indexes on an IOT use a *logical* rowid that points to the PK, so they do *not*
break when the row migrates — Oracle even supports `ON OVERFLOW` for rows that don't fit. Oracle uses
IOTs for small "code"/lookup tables and for tables with fast PK access patterns; it warns that
inserts are costlier and multiple indexes less attractive than on heaps.

```sql
CREATE TABLE order_items (
    order_id   NUMBER NOT NULL,
    line_no    NUMBER NOT NULL,
    product_id NUMBER NOT NULL,
    quantity   NUMBER NOT NULL,
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, line_no)
) ORGANIZATION INDEX;
```

### 3.5 Choosing the clustering / clustered key

Ranked by *worst safe choice* first:

| Property | Good clustering key | Bad clustering key |
|---|---|---|
| Monotonic | ✅ autoincrement, time-ordered | ❌ random UUIDv4 |
| Width | ✅ 4–8 bytes (`int`/`bigint`) | ❌ long string, composite of many cols |
| Stability | ✅ immutable | ❌ email, username, `updated_at` |
| Uniqueness | ✅ unique | ❌ non-unique forces an internal uniquifier |
| Cardinality | ✅ high | ❌ low (e.g., `status`) |

> **Senior note (SQL Server uniquifier):** If the clustered key is *not unique*, SQL Server appends
> a hidden 4-byte **uniquifier** to every duplicate key, swelling *every* non-clustered index. This
> is a classic hidden bloat source in tables clustered on `created_at`.

### 3.6 Concrete numbers: why width matters

Consider `orders` with a `bigint` PK (8 bytes) vs a `uuid` (16 bytes) clustered key, and four
secondary indexes.

| | bigint PK | UUID PK |
|---|---|---|
| Clustered key bytes | 8 | 16 |
| Per secondary entry added | +8 B | +16 B |
| Rows per 8 KiB leaf (secondary leaf holding ~12 B) | ~680 | ~290 |
| Total secondary storage for 100M orders × 4 idx | ~5 GB | ~12 GB (+140%) |
| Insert pattern | append at right edge | split random leaves |

Widening the clustered key is not a local cost — it is multiplied by the number of secondary indexes
and the row count.

---

## 4. How to observe / inspect it

### 4.1 PostgreSQL

```sql
-- Is this table clustered, and by which index? (relkind 'r' always means heap)
SELECT c.relname, c.relkind, i.relname AS clustered_index
FROM   pg_class c
LEFT   JOIN pg_index x ON x.indrelid = c.oid AND x.indisclustered
LEFT   JOIN pg_class i ON i.oid = x.indexrelid
WHERE  c.relname = 'orders';

-- Prove the heap is unordered relative to the PK: sample ctids
SELECT ctid, order_id FROM orders WHERE order_id < 1000 LIMIT 5;
-- If ctids are scattered (e.g. (7,3),(402,1),(2,11)), the heap is not key-ordered.

-- Index vs table sizes (bloat of secondary indexes)
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
FROM   pg_stat_user_indexes WHERE relname = 'orders' ORDER BY idx_size DESC;

-- Reorder the heap once (takes an ACCESS EXCLUSIVE lock)
CLUSTER orders USING orders_pkey;
```

`pgstatindex` (from the `pgstattuple` contrib module) shows internal structure:

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('orders_pkey');
-- avg_leaf_density, leaf_fragmentation, level (tree height), root/internal/leaf pages
```

### 4.2 MySQL InnoDB

```sql
-- Which index is clustered? The PK (or first unique NOT NULL, or implicit GEN_CLUST_ACCESS)
SHOW INDEX FROM orders;        -- Key_name='PRIMARY', Index_type=BTREE

-- Is the PK a GUID? Check for the implicit clustered key
SELECT NAME, TYPE FROM INFORMATION_SCHEMA.INNODB_TABLES WHERE NAME='shop/orders';

-- Watch a bookmark lookup happen (secondary seek + clustered fetch)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;   -- rows examined >> rows returned

-- InnoDB counters about the clustered index
SHOW ENGINE INNODB STATUS\G
-- look for "Rows inserted", "rows updated", and history list length
```

### 4.3 SQL Server

```sql
-- Clustered vs heap, and where the clustered key lives
SELECT i.name, i.type_desc, i.is_primary_key
FROM   sys.indexes i JOIN sys.tables t ON t.object_id = i.object_id
WHERE  t.name = 'orders';
-- type_desc = 'CLUSTERED' or 'HEAP'

-- All indexes, with the clustering key repeated in each non-clustered index
SELECT i.name, i.type_desc,
       STRING_AGG(c.name, ', ') WITHIN GROUP (ORDER BY ic.key_ordinal) AS cols
FROM   sys.indexes i
JOIN   sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
JOIN   sys.columns c ON c.object_id = ic.object_id AND c.column_id = ic.column_id
WHERE  i.object_id = OBJECT_ID('orders')
GROUP BY i.name, i.type_desc;

-- Prove the bookmark lookup (Key Lookup operator in the plan)
SET SHOWPLAN_TEXT ON;
SELECT status, total_cents FROM orders WHERE customer_id = 42;
```

### 4.4 Oracle

```sql
-- Heap vs IOT
SELECT table_name, iot_type FROM user_tables WHERE table_name IN ('ORDERS','ORDER_ITEMS');

-- ROWID tells you if the address is physical (heap) or logical (IOT)
SELECT ROWID, order_id FROM orders WHERE ROWNUM < 5;

-- Index height and clustering factor (KEY: see §6.2)
SELECT index_name, blevel, leaf_blocks, clustering_factor
FROM   user_indexes WHERE table_name = 'ORDERS';
```

`clustering_factor` (Oracle, also observable via `pg_stats.correlation` in Postgres) is the number
the optimizer uses to decide whether a range scan via an index is worth it: near the number of
*blocks* → rows are clustered (cheap); near the number of *rows* → rows are scattered (do a full
scan instead).

---

## 5. Worked examples and demos on the shared sample schema

### 5.1 The shared e-commerce schema (PostgreSQL dialect)

```sql
CREATE TABLE customers (
    customer_id  bigserial PRIMARY KEY,
    email        text NOT NULL UNIQUE,
    full_name    text NOT NULL,
    country_code char(2) NOT NULL,
    status       text NOT NULL DEFAULT 'active',
    created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE categories (
    category_id int PRIMARY KEY,
    name        text NOT NULL,
    parent_id   int REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id  bigserial PRIMARY KEY,
    sku         text NOT NULL UNIQUE,
    name        text NOT NULL,
    category_id int NOT NULL REFERENCES categories(category_id),
    price_cents integer NOT NULL,
    stock_qty   integer NOT NULL DEFAULT 0,
    is_active   boolean NOT NULL DEFAULT true
);

CREATE TABLE orders (
    order_id    bigserial PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES customers(customer_id),
    status      text NOT NULL DEFAULT 'pending',
    created_at  timestamptz NOT NULL DEFAULT now(),
    total_cents integer NOT NULL DEFAULT 0
);

CREATE TABLE order_items (
    order_id   bigint NOT NULL REFERENCES orders(order_id),
    line_no    smallint NOT NULL,
    product_id bigint NOT NULL REFERENCES products(product_id),
    quantity   integer NOT NULL,
    unit_price_cents integer NOT NULL,
    PRIMARY KEY (order_id, line_no)
);
```

### 5.2 Demo: "the index is used, but the query is slow" (heap fetch)

```sql
-- Only a customer_id index exists (added by the FK, conceptually)
CREATE INDEX ix_orders_customer ON orders (customer_id);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 4242;
```

```
 Index Scan using ix_orders_customer on orders
   Index Cond: (customer_id = 4242)
   Buffers: shared hit=118 read=41     ← ~1 index page + ~159 heap pages
 Execution Time: 12.4 ms
```

The index seek found, say, 160 rows in 1–2 index pages, but `SELECT *` needed columns not in the
index, so the engine performed ~160 scattered heap fetches. **`Buffers: read=41`** is the tell.

### 5.3 Demo: the covering fix (preview — full treatment in ch. 03)

```sql
CREATE INDEX ix_orders_customer_created
    ON orders (customer_id, created_at) INCLUDE (status, total_cents);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, status, total_cents
FROM   orders WHERE customer_id = 4242;
```

```
 Index Only Scan using ix_orders_customer_created on orders
   Heap Fetches: 0        ← no bookmark lookup at all
   Buffers: shared hit=3
 Execution Time: 0.31 ms
```

Same query shape; 40× fewer buffers purely because the index *contains* the needed columns.

### 5.4 Demo: pgstattuple shows heap disorder

```sql
CREATE EXTENSION pgstattuple;
SELECT table_len, tuple_count, dead_tuple_count, free_percent
FROM   pgstattuple('orders');
-- Compare the ctid ordering of a primary-key range vs. insertion order:
SELECT ctid, order_id FROM orders ORDER BY order_id LIMIT 5;
--  (0,1) (0,2) (0,3)...  → ordered only if inserts were sequential AND no updates moved rows
```

### 5.5 Demo (InnoDB): random PK vs. sequential PK insert throughput

```sql
-- Sequential PK
CREATE TABLE o_seq (id BIGINT AUTO_INCREMENT PRIMARY KEY, payload VARBINARY(200));
-- Random PK (UUID)
CREATE TABLE o_rand (id CHAR(36) PRIMARY KEY, payload VARBINARY(200));
-- Insert 2M rows into each; compare wall-clock and ibd file fragmentation.
-- Typical result: o_seq ~2–4× faster and less fragmented.
```

---

## 6. Real-world use cases and applications

### 6.1 Time-series / append-only tables clustered by `(entity_id, ts)`

A `events` or `metrics` table almost always benefits from clustering on `(tenant_id, occurred_at)`:
range scans per tenant become sequential, and `ORDER BY ... LIMIT 100` needs no sort. In a heap engine
(Postgres), you get this with partitioning + `CLUSTER`, or by using a **BRIN** index if the data is
naturally append-ordered.

### 6.2 Lookup / reference tables as IOTs

Oracle IOTs shine for narrow code tables (`countries`, `currencies`, `states`) where every access is
by primary key. Round-trip becomes a single index descent.

### 6.3 The classic incident: UUID primary keys on InnoDB

Teams migrating a MySQL service to `UUID v4` primary keys (chosen for distributed ID generation)
suddenly see:
- Insert latency spikes (page splits + double-writes).
- 2–4× larger secondary indexes.
- Autoincrement sequences abandoned and re-architected to UUIDv7/ULID (time-ordered).

Amazon published the well-known result that **UUIDv7 / ULID** recover most of the sequential
performance. In SQL Server, the same story is told with `NEWID()` vs `NEWSEQUENTIALID()`.

### 6.4 The other classic incident: clustered key updated at scale

A table clustered on `created_at` with an `UPDATE ... SET created_at = now()` habit causes the row
to migrate to the end of the clustered index, turning a "cheap" update into a delete+insert plus
fragmentation. Immutability of the clustering key is a silent but load-bearing requirement.

### 6.5 Unindexed foreign keys

In Postgres, a foreign key does **not** automatically create an index on the referencing column.
The parent `DELETE` issues a `SELECT ... FOR KEY SHARE` on the child table, which without an index
becomes a **sequential scan per deleted parent row** — a notorious production incident ("deleting one
customer locks the DB"). InnoDB *does* auto-create an index on the FK column; SQL Server does not
(you must add it yourself). This is the single most common cause of a mysterious `DELETE`-time
table lock.

```sql
-- Find FK columns with no supporting index (PostgreSQL)
SELECT conrelid::regclass AS child_table, conname
FROM   pg_constraint c
WHERE  contype = 'f'
AND    NOT EXISTS (
         SELECT 1 FROM pg_index i
         WHERE i.indrelid = c.conrelid
           AND i.indkey::int[] @> c.conkey::int[]);  -- simplified containment check
```

---

## 7. Patterns and anti-patterns

### Patterns

| Pattern | Why it works |
|---|---|
| Narrow, monotonic clustering key (int/bigint/snowflake-id/time-UUID) | append-only inserts, small secondary indexes |
| Cover hot secondary indexes with `INCLUDE` | kills the bookmark lookup without widening the key |
| Cluster heap tables on the dominant range predicate (`CLUSTER`/`pg_repack`) | sequential range scans |
| Index every FK column | avoids scan-per-delete on the parent |
| Put the *selective* column first in a composite secondary index | narrower index range before the lookup |

### Anti-patterns

| Anti-pattern | What breaks |
|---|---|
| Random UUIDv4 clustered key on InnoDB/SQL Server | page splits, huge secondary indexes |
| Mutable clustering key | row migration, fragmentation |
| Non-unique clustered key on SQL Server | 4-byte uniquifier on every row + every non-clustered index |
| Assuming Postgres `PRIMARY KEY` = clustered | range scans still fetch the heap |
| `CLUSTER` and forget | order decays after the next writes |
| Loading `SELECT *` from a non-clustered index plan | millions of bookmark lookups |

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL InnoDB | SQL Server | Oracle |
|---|---|---|---|---|
| Table storage default | Heap (unordered) | **Clustered index always** | Heap or clustered (your choice) | Heap |
| "Clustered table" feature | none (only one-time `CLUSTER`) | the table itself | `CREATE CLUSTERED INDEX` | `ORGANIZATION INDEX` (IOT) |
| Primary key = clustered? | ❌ (unique index) | ✅ (or first unique NOT NULL; else hidden 6-byte id) | ❌ unless declared `CLUSTERED` | ❌ unless IOT |
| Row locator in secondary index | `ctid` (block, offset) | primary key columns | clustering key (RID for heaps) | `ROWID` (logical for IOT) |
| Bookmark lookup cost | 1 heap fetch per row | 1 clustered descent per row | Key Lookup / RID Lookup | table access by ROWID |
| `INCLUDE` (non-key payload) | ✅ since 11 (`INCLUDE`) | ❌ (must add to key) | ✅ since 2005 (`INCLUDE`) | ❌ (add to key) |
| Free-space control | `fillfactor` per index/table | `FILLFACTOR` (InnoDB: `MERGE_THRESHOLD`) | `FILLFACTOR` | `PCTFREE` |
| Ideal clustering key | narrow + monotonic | narrow + monotonic (auto-inc PK) | narrow, unique, monotonic | PK of an IOT: narrow, stable |
| Auto FK index | ❌ | ✅ | ❌ | ❌ |

---

## 9. Exercises — Intermediate (hints only)

1. **Back-of-envelope width.** Given a `bigint` clustered key and a 16-byte UUID clustered key,
   estimate the extra bytes per secondary index entry and the % storage growth for a table with four
   secondary indexes. *Hint: total = bytes_per_entry × entries_per_index × num_indexes.*

2. **Read the plan.** Run `EXPLAIN (ANALYZE, BUFFERS)` on
   `SELECT * FROM orders WHERE customer_id = ?`. If `Buffers: read` is ≫ the number of index pages,
   what operator is doing the extra work, and what are two ways to remove it?

3. **Proof by ctid.** Show that a Postgres heap is unordered. *Hint: `SELECT ctid, order_id FROM
   orders WHERE order_id BETWEEN 100 AND 120;` — do the ctids ascend with the ids?*

4. **IOT decision.** For `categories (category_id, name, parent_id)`, arg for and against making it
   an Oracle IOT. *Hint: access patterns, row width, update frequency.*

5. **Unindexed FK.** Delete one `customers` row with `EXPLAIN` and identify the scan on
   `order_items`. Add the FK index and re-measure.

---

## 10. Exercises — Advanced (hints / expected approach)

1. **Design a clustering key for a multi-tenant events table** (`tenant_id`, `occurred_at`,
   `event_type`, `payload jsonb`) on InnoDB and on SQL Server. Expected approach: discuss whether to
   cluster on `(tenant_id, occurred_at)` vs a surrogate monotonic PK + secondary index; quantify the
   secondary-index inflation of the composite clustered key versus the locality win.

2. **Simulate UUIDv4 vs UUIDv7 insert behavior.** Measure `Pages written`, `pages split`, and file
   fragmentation for 1M inserts of each key type on InnoDB (`innodb_metrics` /
   `innodb_metrics` counters `index_page_splits`). Expected: relate split count to leaf occupancy.

3. **Reason about the `CLUSTER` decay.** After `CLUSTER orders USING orders_pkey`, insert 5% new
   rows. Measure `pg_stats.correlation` for `order_id` before and after. Expected: explain the decay
   rate in terms of inserted fraction and free space.

4. **When a bookmark lookup is cheaper than a scan.** Construct a query where a non-clustered index
   seek + 100k key lookups is *slower* than a full table scan, and prove it with `BUFFERS`. Expected:
   show the crossover (rows returned vs pages in the table) and state the rule of thumb.

5. **Cross-engine equivalence.** Write the DDL that reproduces the same physical layout
   (rows sorted by `(customer_id, created_at)`) in all four engines. Expected approach: for Postgres
   discuss partitioned + clustered or a covering index; for InnoDB a clustered PK; for SQL Server a
   clustered index; for Oracle an IOT or a cluster.

---

## References

- PostgreSQL: [B-Tree Indexes](https://www.postgresql.org/docs/current/btree.html)
- PostgreSQL: [Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
- PostgreSQL: [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
- PostgreSQL: [`CLUSTER`](https://www.postgresql.org/docs/current/sql-cluster.html)
- MySQL: [Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- MySQL: [The Physical Structure of an InnoDB Index](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)
- Microsoft: [Clustered and Nonclustered Indexes Described](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- Microsoft: [SQL Server Index Architecture and Design Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- Oracle: [Indexes and Index-Organized Tables](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/indexes-and-index-organized-tables.html)
- Markus Winand: [Use The Index, Luke! — Clustering](https://use-the-index-luke.com/sql/clustering)
- Peter Zaitsev: [To UUID or not to UUID?](https://www.percona.com/blog/2007/05/09/to-uuid-or-not-to-uuid/) (Percona)
- Karthik Appigatla: [Store UUID in an Optimized Way](https://www.percona.com/blog/2018/09/12/store-uuid-in-an-optimized-way/) (Percona)
- Bayer & McCreight, *Organization and Maintenance of Large Ordered Indices* (1972): https://dl.acm.org/doi/10.1145/1734663.1734671

# SQL

SQL is the *lingua franca* of data. Every engine you touch (Postgres, Snowflake, BigQuery,
Spark SQL, DuckDB, ClickHouse) speaks a dialect of it, and it is the one skill that survives
every framework migration. But "knowing SQL" splits into four very different competencies, and
this part is organized around exactly those:

1. **Query writing** — expressing intent correctly and readably (joins, CTEs, windows, sets).
2. **Query optimization** — understanding *why* the engine picks a plan and how to make it fast.
3. **Database internals** — what is actually happening on disk and in memory (B-trees, MVCC, locks, ACID).
4. **Analytical SQL** — the patterns analysts and data engineers reinvent daily (cohorts, funnels, gaps-and-islands).

The philosophy is: **definitions → how it works inside → how to implement →
where to implement → real use cases → trade-offs → exercises**. Every chapter uses a **shared
sample schema** so the examples are runnable, and ends with **references** to authoritative
sources (PostgreSQL/MySQL/SQL Server/Oracle/Snowflake/BigQuery docs, *Use The Index, Luke!*,
*Modern SQL*, papers, and engineering blogs).

> **Primary dialect:** PostgreSQL (the most standards-compliant and the one SQL-fluent engineers
> should reason in). Each chapter adds a **cross-engine differences** table for MySQL, SQL
> Server, Oracle, Snowflake and BigQuery.

---

## Suggested learning path

| Order | Section | Why |
|------|---------|-----|
| 1 | Query writing | Get fluent at expressing intent before optimizing it. |
| 2 | Database internals | You cannot optimize what you do not understand (B-trees, MVCC). |
| 3 | Query optimization | Turn internals knowledge into faster queries. |
| 4 | Analytical SQL | Apply everything to the patterns that show up in real analytics work. |

---

## Table of contents

### [01 · Query Writing](./01-query-writing/)
- [Complex JOINs](./01-query-writing/01-complex-joins.md)
- [Common Table Expressions (CTEs)](./01-query-writing/02-ctes.md)
- [Recursive CTEs](./01-query-writing/03-recursive-ctes.md)
- [Window functions](./01-query-writing/04-window-functions.md)
- [CASE expressions](./01-query-writing/05-case-expressions.md)
- [EXISTS, IN and subqueries](./01-query-writing/06-exists-and-subqueries.md)
- [Correlated subqueries](./01-query-writing/07-correlated-subqueries.md)
- [PIVOT / UNPIVOT](./01-query-writing/08-pivot-unpivot.md)
- [Set operations](./01-query-writing/09-set-operations.md)

### [02 · Query Optimization](./02-query-optimization/)
- [Why a query is slow](./02-query-optimization/01-why-a-query-is-slow.md)
- [How indexes help](./02-query-optimization/02-how-indexes-help.md)
- [Why an index is ignored](./02-query-optimization/03-why-an-index-is-ignored.md)
- [Reading an execution plan](./02-query-optimization/04-reading-an-execution-plan.md)
- [Join algorithms (nested loop, hash, merge)](./02-query-optimization/05-join-algorithms.md)
- [Predicate pushdown](./02-query-optimization/06-predicate-pushdown.md)
- [Cardinality estimation](./02-query-optimization/07-cardinality-estimation.md)
- [Partition pruning](./02-query-optimization/08-partition-pruning.md)

### [03 · Database Internals](./03-database-internals/)
- [Clustered vs non-clustered indexes](./03-database-internals/01-clustered-vs-nonclustered-indexes.md)
- [B-tree indexes (from the ground up)](./03-database-internals/02-b-tree-indexes.md)
- [Covering indexes](./03-database-internals/03-covering-indexes.md)
- [Transactions](./03-database-internals/04-transactions.md)
- [Isolation levels](./03-database-internals/05-isolation-levels.md)
- [Locks](./03-database-internals/06-locks.md)
- [MVCC](./03-database-internals/07-mvcc.md)
- [ACID](./03-database-internals/08-acid.md)

### [04 · Analytical SQL](./04-analytical-sql/)
- [Ranking](./04-analytical-sql/01-ranking.md)
- [Running totals](./04-analytical-sql/02-running-totals.md)
- [Moving averages](./04-analytical-sql/03-moving-averages.md)
- [Cohort analysis](./04-analytical-sql/04-cohort-analysis.md)
- [Retention](./04-analytical-sql/05-retention.md)
- [Funnels](./04-analytical-sql/06-funnels.md)
- [Sessionization](./04-analytical-sql/07-sessionization.md)
- [Gap-and-islands problems](./04-analytical-sql/08-gap-and-islands.md)

---

## Conventions

- **Sample schema**: a small e-commerce domain (`customers`, `orders`, `order_items`,
  `products`, `categories`, `employees`/managers, `events`) is introduced in
  [Complex JOINs](./01-query-writing/01-complex-joins.md) and reused throughout.
  A runnable copy of the DDL + deterministic seed data lives in
  [`01-query-writing/00-sample-schema.sql`](./01-query-writing/00-sample-schema.sql) —
  load it once (`psql -f 00-sample-schema.sql`) and every example in this part runs as written.
- **Dialect**: PostgreSQL unless stated otherwise; cross-engine notes per chapter.
- **Exercises**: every chapter ends with an **Intermediate** and an **Advanced** set (hints and
  expected approach, not pasted answers).
- **`> Senior note`** blocks flag nuances, traps, and production lessons.

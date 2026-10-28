# Isolation Levels

> Isolation is the ACID property people get wrong most often, because the standard that defines it
> is *famous for being ambiguous*. SQL-92 describes four levels in terms of three "phenomena," and
> then almost every real engine implements something that does not fit those definitions: PostgreSQL
> promotes `READ UNCOMMITTED` to `READ COMMITTED` and implements `REPEATABLE READ` as **snapshot
> isolation** (so it has no phantoms but still permits **write skew**); MySQL's `REPEATABLE READ` is
> a snapshot plus gap locks; Oracle calls plain snapshot isolation `SERIALIZABLE`. The practical
> consequence is that you cannot reason about concurrency from the level *names* — you must know the
> **phenomena each engine actually permits**. This chapter gives you the phenomena, the mechanisms,
> and a decision procedure you can apply to your own schema.

## TL;DR

- Four standard levels: **READ UNCOMMITTED < READ COMMITTED < REPEATABLE READ < SERIALIZABLE**.
  Stronger = fewer anomalies, more blocking/aborts, lower concurrency.
- Site-of-truth phenomena (Berenson et al. additions included): **dirty write, dirty read, read skew,
  non-repeatable read, lost update, phantom, write skew**.
- **Snapshot isolation (SI)** is the reality behind most "REPEATABLE READ" (and Oracle's
  "SERIALIZABLE"). SI prevents dirty/read-skew/non-repeatable/phantom/lost-update, but **allows write
  skew**.
- **PostgreSQL**: `READ UNCOMMITTED` → behaves as `READ COMMITTED`; **default `READ COMMITTED`**;
  `REPEATABLE READ` = SI; `SERIALIZABLE` = **SSI** (Serializable Snapshot Isolation, the first
  production implementation) — aborts with `40001 serialization_failure`, so apps **must retry**.
- **MySQL InnoDB**: **default `REPEATABLE READ`**; consistent (non-locking) reads use a snapshot;
  `SELECT ... FOR UPDATE`/writes use a *current read* with **next-key (gap) locks** to prevent phantoms.
- **SQL Server**: **default `READ COMMITTED`** (locking); with `READ_COMMITTED_SNAPSHOT ON` it becomes
  statement-level snapshot (RCSI); `SNAPSHOT` gives transaction-level SI; `SERIALIZABLE` uses range
  locks.
- **Oracle**: **default `READ COMMITTED`** (statement-level snapshot); `SERIALIZABLE` = **snapshot
  isolation** (not fully serializable → write skew allowed); `READ UNCOMMITTED` maps to `READ COMMITTED`.
- Choose the **weakest level that preserves your invariants**, and note that at SI you often need to
  make invariants enforceable by rewriting the transaction (take explicit locks, or use a
  `SELECT ... FOR UPDATE`, or use a constraint).

---

## 1. Definitions and core concepts (from the ground up)

### 1.1 What "isolation" means

Isolation answers exactly one question: **when multiple transactions run concurrently, can their
effects be interleaved in a way that produces a result no serial (one-at-a-time) order could
produce?** A level names *which interleavings you forbid*.

### 1.2 The phenomena (build each one concretely)

Each example uses the e-commerce schema (`orders`, `products`, `customers`).

**Dirty write (P0).** T1 overwrites a value T2 wrote but has not committed. Never allowed in any
engine you will meet — it would make rollback impossible.

**Dirty read (P1).** T2 reads data T1 wrote but has not committed (and T1 later rolls back), so T2
acts on a value that never existed.

```
 T1: UPDATE products SET stock_qty = 0 WHERE product_id = 101;   -- not committed
 T2: SELECT stock_qty FROM products WHERE product_id = 101;      -- reads 0  ← dirty
 T1: ROLLBACK;                                                   -- 0 never existed
```

**Read skew / non-repeatable read.** T1 reads `X`, T2 updates `X` and commits, T1 reads `X` again
and gets a *different* answer — within the same logical operation. (Read skew is the multi-row
version; non-repeatable read is the two-reads-of-one-row version.)

```
 T1: SELECT total_cents FROM orders WHERE order_id = 42;  -- 1000
 T2: UPDATE orders SET total_cents = 2000 WHERE order_id = 42; COMMIT;
 T1: SELECT total_cents FROM orders WHERE order_id = 42;  -- 2000   ← changed mid-txn
```

**Phantom read.** T1 runs `SELECT ... WHERE status='pending'` (returns 3 rows), T2 inserts a new
`pending` row and commits, T1 re-runs the same query and now sees **4 rows** — a "phantom."

**Lost update.** Two transactions read the same value and both write back a value derived from their
read, so one update is silently lost. The classic inventory oversell: two buyers each read `stock=1`,
both decrement to `0`, but two orders were placed.

```
 T1: SELECT stock_qty ... → 1
 T2: SELECT stock_qty ... → 1
 T1: UPDATE products SET stock_qty = stock_qty - 1 → 0   COMMIT
 T2: UPDATE products SET stock_qty = stock_qty - 1 → 0   COMMIT   ← 2 sold, stock never went negative
```

**Write skew.** Two transactions each read a set, each makes a change that is *individually* valid,
but the *combination* violates an invariant. This is the anomaly snapshot isolation **does not**
prevent, and the reason it matters that PostgreSQL's `REPEATABLE READ` is not `SERIALIZABLE`.

Classic example (on-call doctors): invariant "at least one doctor on call."

```
 T1: SELECT COUNT(*) FROM on_call WHERE shift='night';   -- 2
 T2: SELECT COUNT(*) FROM on_call WHERE shift='night';   -- 2
 T1: UPDATE on_call SET on_call=false WHERE doctor_id=1; COMMIT   -- 1 remains ✓ individually
 T2: UPDATE on_call SET on_call=false WHERE doctor_id=2; COMMIT   -- 0 remain ✗ invariant broken
```

E-commerce analogue: a "maximum total discount per order" rule where each line item is discounted by
two concurrent transactions that only look at the current discount total.

```sql
-- Write-skew invariant: an order's total discount must stay under 20%
-- T1 and T2 each read 15% and each add 5% → 25% total, neither saw the other.
```

**Summary of what each level forbids:**

| Phenomenon | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ (locking) | SNAPSHOT (SI) | SERIALIZABLE |
|---|---|---|---|---|---|
| Dirty write | ❌ | ❌ | ❌ | ❌ | ❌ |
| Dirty read | **allowed** | ❌ | ❌ | ❌ | ❌ |
| Read skew / non-repeatable read | allowed | **allowed** | ❌ | ❌ | ❌ |
| Lost update | allowed | allowed | allowed* | ❌ | ❌ |
| Phantom | allowed | allowed | ❌ (range locks) | ❌ | ❌ |
| Write skew | allowed | allowed | allowed | **allowed** | ❌ |

\* Repeatable read with **range locks** prevents phantoms but can still allow lost updates unless the
read is a locking read.

> **Senior note:** The table above is the *mechanism* view. The standard's phenomena table
> (dirty/non-repeatable/phantom only) is misleading and was critiqued as such by Berenson et al. in
> 1995. "REPEATABLE READ" in the standard and "REPEATABLE READ" in PostgreSQL are **different
> promises**.

### 1.3 Vocabulary

| Term | Meaning |
|---|---|
| **Anomaly** | A concurrency outcome no serial order could produce. |
| **Snapshot** | A consistent view of the database as of a point in time (MVCC read view). |
| **SI** | Snapshot Isolation: txn sees one snapshot; conflicting writes to the same row abort. |
| **SSI** | Serializable Snapshot Isolation: SI + detection of rw-antidependency cycles. |
| **Next-key lock** | InnoDB gap+record lock preventing inserts into a range (kills phantoms). |
| **RCSI** | SQL Server Read Committed Snapshot Isolation (versioned READ COMMITTED). |
| **Read view** | InnoDB's clone of the active-transaction list for a snapshot. |
| **Serialization failure** | `40001`/deadlock: retry the transaction. |

---

## 2. Why it matters / what problem it solves

Isolation is a correctness dial, not a speed dial. Getting it wrong produces bugs that are
**impossible to reproduce**: an oversold item on Black Friday, a duplicate coupon, a double-booked
appointment — all from anomaly windows that exist for microseconds under load. Getting it *too*
strong produces throughput collapse from lock contention and aborts. The senior skill is mapping each
business invariant to the *weakest* mechanism that enforces it:

| Invariant type | Enforcement |
|---|---|
| Single-row value must not be lost | atomic `SET x = x - 1` or `FOR UPDATE` or SI |
| Row must exist/not exist | `UNIQUE` constraint (cheapest, concurrency-friendly) |
| Aggregate bound (sum, count) | serializable, or explicit predicate locking, or materialize the counter row |
| Cross-row rule | serializable SSI, or redesign to a single hot row |

---

## 3. How it works internally

### 3.1 Lock-based isolation (SQL Server default, Oracle/InnoDB for writes)

Locks are acquired as needed; the level controls **when locks are released** and **what is locked**:

- `READ UNCOMMITTED` → no shared locks (dirty reads possible).
- `READ COMMITTED` (locking) → shared locks released at *statement* end → non-repeatable reads and
  phantoms possible.
- `REPEATABLE READ` → shared locks held to *transaction* end → no non-repeatable reads, but inserts
  can still slip in → phantoms possible.
- `SERIALIZABLE` → range locks held to transaction end → phantoms impossible.

```
  Lock held until:   statement end ─────────────── txn end
  READ COMMITTED:    [S locks on read rows released]
  REPEATABLE READ:                        [S locks on read rows held]
  SERIALIZABLE:                           [S locks on read RANGES held → blocks inserts]
```

### 3.2 Snapshot isolation (MVCC) — how it really works

Under MVCC (see [MVCC](./07-mvcc.md)) a transaction reads a **snapshot**: a set of row versions
visible as of a point in time. Readers never block writers.

```
  T1 commits at t=5         T2 starts at t=6 (its snapshot = "everything with commit_ts <= 6")
  ─────┬───────────────────────┬─────────────────────────────────▶ time
       │                       │
     row v1 → v2             T2 reads v2 forever, even if
                             someone writes v3 at t=7
```

Snapshot isolation additionally checks **write-write conflicts**: if T2 tries to write a row whose
latest committed version was written *after* T2's snapshot, T2 must **abort** (or the write fails).
That rule is what prevents **lost updates** under SI.

```
  Rule (SI, "first committer wins"): two concurrent txns cannot both commit writes to the same row.
  PostgreSQL REPEATABLE READ  → the second writer gets:
     ERROR: could not serialize access due to concurrent update
  InnoDB REPEATABLE READ (current read) → uses locks, blocks instead of aborting
```

### 3.3 SSI (PostgreSQL SERIALIZABLE) — how write skew is caught

SSI tracks **serialization graph** edges cheaply. It looks for the "dangerous structure": two
consecutive **rw-antidependencies** (T1 read something T2 later wrote, and T3 later wrote something
T1 read) forming a potential cycle. If a cycle might form, one transaction is aborted with
`40001`. Under the doctor example, both T1 and T2 read the *set* of on-call rows and then wrote; SSI
notices the pivot row and aborts one.

```
 rw-dependency: T1 read row r; T2 wrote row r after →  T1 →rw T2
 cycle: T1 →rw T2 →rw T1  ⇒ abort one
```

> **Senior note:** PostgreSQL's `SERIALIZABLE` is **optimistic**: it does not block; it aborts and
> asks *you* to retry. Code that is not retry-safe will fail in production under contention. This is
> the number one reason teams "tried SERIALIZABLE once and it broke things."

### 3.4 InnoDB's REPEATABLE READ: snapshot + current reads

- **Consistent read** (`SELECT` without `FOR UPDATE`): uses the read view/undo chain → sees the
  snapshot, no locks.
- **Locking read / write** (`SELECT ... FOR UPDATE`, `UPDATE`, `DELETE`): reads the **latest**
  committed version and locks it, plus **next-key locks** (record + gap) to prevent phantoms.
- Therefore `READ COMMITTED` in InnoDB also disables gap locking (except for FK/unique checks),
  which is why many high-throughput apps lower it to `READ COMMITTED` to reduce deadlocks.

### 3.5 SQL Server: RCSI vs SNAPSHOT

- `ALTER DATABASE … SET READ_COMMITTED_SNAPSHOT ON` makes the **default** `READ COMMITTED` versioned:
  readers don't take shared locks, avoiding writer-vs-reader blocking. It is **statement-level**
  (each statement gets a fresh snapshot) → still allows non-repeatable reads.
- `SET TRANSACTION ISOLATION LEVEL SNAPSHOT` (requires `ALLOW_SNAPSHOT_ISOLATION ON`) is
  **transaction-level** SI, with update conflicts producing error `3960`.
- Both store row versions in **`tempdb`** → version store growth is an operational concern.

### 3.6 Oracle: two levels in practice

Oracle has `READ COMMITTED` and `SERIALIZABLE`, but the latter is **snapshot isolation** (Oracle's
docs note it is not fully serializable). Its `READ COMMITTED` is a *statement-level* snapshot, so
each statement sees the latest committed data — which is why Oracle apps rarely see the "long
transaction blocks everything" problem but do see read skew across statements.

---

## 4. How to observe / inspect it

### 4.1 Check the level

| Engine | Command |
|---|---|
| PostgreSQL | `SHOW transaction_isolation; SELECT current_setting('transaction_isolation');` |
| MySQL | `SELECT @@transaction_isolation; -- and @@transaction_read_only` |
| SQL Server | `DBCC USEROPTIONS; SELECT ... FROM sys.dm_exec_sessions` |
| Oracle | `SELECT * FROM v$transaction; -- no per-session level; check app/ALTER SESSION` |

### 4.2 Watch anomalies live

```sql
-- PostgreSQL: two psql sessions, default READ COMMITTED
-- Session A
BEGIN;
SELECT total_cents FROM orders WHERE order_id = 42;   -- 1000
-- Session B
UPDATE orders SET total_cents = 2000 WHERE order_id = 42;
COMMIT;
-- Session A (still open)
SELECT total_cents FROM orders WHERE order_id = 42;   -- 2000  ← non-repeatable read (RC)
COMMIT;
-- Rerun the same as:  SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; → second read returns 1000
```

```sql
-- PostgreSQL: prove SERIALIZABLE aborts on write skew
-- A: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
--    SELECT count(*) FROM orders WHERE status='pending';
-- B: SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
--    SELECT count(*) FROM orders WHERE status='pending';
-- A: INSERT INTO orders (customer_id,status) VALUES (1,'pending'); COMMIT;   -- ok
-- B: INSERT INTO orders (customer_id,status) VALUES (2,'pending'); COMMIT;
--    → ERROR: could not serialize access due to read/write dependencies among transactions
```

### 4.3 Inspect the snapshot machinery

```sql
-- PostgreSQL: which XIDs are in-flight (defines the snapshot)?
SELECT pid, backend_xid, backend_xmin, state FROM pg_stat_activity;

-- MySQL: the read view / active transaction list
SELECT trx_id, trx_state, trx_started FROM information_schema.innodb_trx;

-- SQL Server: version store size in tempdb
SELECT * FROM sys.dm_tran_version_store_space_usage;
SELECT * FROM sys.dm_tran_active_snapshot_database_transactions;
```

---

## 5. Worked examples and demos on the shared sample schema

### 5.1 The oversell bug, three ways

```sql
-- BROKEN under READ COMMITTED (lost update): read-modify-write in the app
SELECT stock_qty FROM products WHERE product_id = 101;   -- returns 1
-- app computes 1-1=0, later writes:
UPDATE products SET stock_qty = 0 WHERE product_id = 101;
-- two concurrent copies → two orders sold, stock never negative-facing.
```

```sql
-- FIX 1 — atomic read-modify-write (no isolation help needed)
UPDATE products
SET    stock_qty = stock_qty - 1
WHERE  product_id = 101 AND stock_qty >= 1;
-- Check the rowcount/affected rows: 0 → out of stock. Single statement, no anomaly window.
```

```sql
-- FIX 2 — explicit row lock (pessimistic; correct under READ COMMITTED)
BEGIN;
SELECT stock_qty FROM products WHERE product_id = 101 FOR UPDATE;  -- serializes checkouts
UPDATE products SET stock_qty = stock_qty - 1 WHERE product_id = 101;
COMMIT;
```

```sql
-- FIX 3 — snapshot isolation with retry
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE products SET stock_qty = stock_qty - 1 WHERE product_id = 101;
COMMIT;
-- Concurrent writer to the same row → one gets:
--   ERROR: could not serialize access due to concurrent update
-- → the app must retry.
```

### 5.2 The "at least one active admin" invariant (write skew demo)

```sql
-- Invariant: every tenant must keep at least one active admin.
-- BROKEN under REPEATABLE READ (SI) in PostgreSQL and MySQL, and under Oracle SERIALIZABLE:
-- A: SELECT count(*) FROM tenant_admins WHERE tenant_id=7 AND active;   -- 2
-- B: SELECT count(*) FROM tenant_admins WHERE tenant_id=7 AND active;   -- 2
-- A: UPDATE tenant_admins SET active=false WHERE tenant_id=7 AND user_id=1; COMMIT;
-- B: UPDATE tenant_admins SET active=false WHERE tenant_id=7 AND user_id=2; COMMIT;
-- → 0 active admins. Both transactions were individually valid.

-- FIX under SERIALIZABLE: PostgreSQL SSI aborts one with 40001 → retry.
-- FIX under SI: materialize the invariant into a single row you can lock:
BEGIN;
UPDATE tenants SET active_admin_count = active_admin_count - 1
WHERE  tenant_id = 7 AND active_admin_count > 1;   -- 0 rows affected → reject
COMMIT;
```

### 5.3 Phantom demo (READ COMMITTED vs REPEATABLE READ)

```sql
-- A (READ COMMITTED):
BEGIN;
SELECT count(*) FROM orders WHERE status='pending';   -- 3
-- B: INSERT INTO orders (customer_id,status) VALUES (1,'pending'); COMMIT;
SELECT count(*) FROM orders WHERE status='pending';   -- 4   ← phantom
COMMIT;

-- A (REPEATABLE READ): the second SELECT still returns 3 (snapshot holds). No phantom.
-- Under InnoDB REPEATABLE READ with FOR UPDATE, B would BLOCK on the gap lock instead.
```

### 5.4 Demoing default levels

```sql
-- PostgreSQL
SHOW transaction_isolation;          -- read committed (default)
BEGIN ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;          -- read uncommitted  (accepted but behaves as RC)
```

```sql
-- MySQL
SELECT @@transaction_isolation;      -- REPEATABLE-READ (default)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

```sql
-- SQL Server
DBCC USEROPTIONS;                    -- isolation level: read committed (default)
SELECT snapshot_isolation_state_desc, is_read_committed_snapshot_on FROM sys.databases WHERE name='shop';
```

---

## 6. Real-world use cases and applications

### 6.1 Black-Friday oversell (lost update)

The archetypal e-commerce incident: flash sale, thousands of concurrent checkouts, "sold 3,000 units
of a 2,000-unit allocation." Root cause is always a read-modify-write across an application boundary
or a non-locking read under `READ COMMITTED`. The fix is always: **push the check into the write**
(`WHERE stock >= n`) or take a lock.

### 6.2 Double-booking (phantom)

Booking systems (`appointments`, `seats`, `slots`) hit phantom reads: two transactions check "is this
slot free?" and both insert. The right fix is usually a **unique constraint** on
`(resource_id, slot_start)` — a constraint enforces the invariant regardless of isolation level and
costs nothing when it is satisfied.

### 6.3 Counter / quota invariants (write skew)

"At most N items discounted per order," "at most one primary contact per account," "sum of
allocations ≤ budget" are all write-skew-shaped. Under SI they need either `SERIALIZABLE` + retry, or
a redesign into a single lockable row.

### 6.4 Reporting on a busy OLTP database

Analysts want "a consistent snapshot of the whole database." PostgreSQL `REPEATABLE READ` and InnoDB
snapshot reads give exactly that — but a long snapshot blocks vacuum/purge (see
[Transactions](./04-transactions.md)). Dedicated read replicas or `pg_dump`-style export avoid the
problem.

### 6.5 Migrating from Oracle to PostgreSQL: the isolation surprise

Applications that relied on Oracle's `READ COMMITTED` (statement-level) semantics can break subtly
under PostgreSQL when they use `REPEATABLE READ` for "consistency," because PostgreSQL's RR is a
transaction-level snapshot — statements in the same transaction see stale data relative to each
other's point in time. Similarly, teams discover that Oracle's `SERIALIZABLE` did **not** actually
give them serializability.

---

## 7. Patterns and anti-patterns

### Patterns

| Pattern | Why |
|---|---|
| Encode invariants as **constraints** (unique/FK/check) | enforced regardless of isolation, no aborts |
| Atomic single-statement updates (`SET x = x - n WHERE x >= n`) | no read-modify-write window |
| `SELECT ... FOR UPDATE` / `FOR NO KEY UPDATE` for genuine critical sections | deterministic blocking |
| Use `SERIALIZABLE` + retry for aggregate invariants | only level that forbids write skew |
| Consistent lock ordering | fewer deadlocks |
| Read snapshots on replicas for reports | isolate the primary |

### Anti-patterns

| Anti-pattern | Symptom |
|---|---|
| Read-modify-write in application code under `READ COMMITTED` | lost updates |
| Assuming `REPEATABLE READ` = serializable | write-skew corruption |
| `SERIALIZABLE` without retry logic | bursts of `40001` errors under load |
| Raising isolation "to be safe" without measuring | throughput collapse |
| Using `SELECT ... FOR UPDATE` on a whole table | convoy/queueing |
| Long SI transactions for reporting on the primary | horizon blocks vacuum |

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL InnoDB | SQL Server | Oracle |
|---|---|---|---|---|
| Default level | **READ COMMITTED** | **REPEATABLE READ** | **READ COMMITTED** | **READ COMMITTED** |
| `READ UNCOMMITTED` | promoted to RC | real (dirty reads possible) | real | promoted to RC |
| `REPEATABLE READ` means | **snapshot isolation** | snapshot + locks (gap locks) | locking RR | (no such level) |
| Snapshot mechanism | MVCC heap + `xmin/xmax` | undo log + read view | `tempdb` version store (RCSI/SNAPSHOT) | undo segments |
| RCSI support | n/a (always MVCC) | n/a | ✅ `READ_COMMITTED_SNAPSHOT ON` | n/a |
| True `SERIALIZABLE`? | ✅ **SSI** (aborts, retry) | ✅ (locking, `SELECT` on `innodb_autoinc` quirks) | ✅ (range locks) | ❌ (SI; write skew allowed) |
| Phantoms at RR | ❌ none | ❌ none (consistent reads) | ❌ none | n/a |
| Lost updates at RR | ❌ (second writer aborts) | ⚠️ prevented via locks on writes | ❌ (locking RR) | ❌ (SI rule) |
| Write skew stopped only at | SERIALIZABLE | SERIALIZABLE | SERIALIZABLE | (needs app-level locks) |
| Errors to retry | `40001`, `40P01` | `1213` deadlock, `40001` | `1205` deadlock, `3960` update conflict | `ORA-08177` |
| Reduce deadlocks by | consistent ordering | lowering to RC (fewer gap locks) | RCSI | shorter txns |

---

## 9. Exercises — Intermediate (hints only)

1. **Name the anomaly.** Two transactions each read the same `stock_qty`, both decrement, both
   commit, and one decrement is lost. Which phenomenon? At which levels does it occur? *Hint: lost
   update; SI prevents it.*

2. **Phantom vs non-repeatable.** Explain the difference using two queries against `orders`. *Hint:
   one involves new rows, the other changed rows.*

3. **Atomic fix.** Rewrite a check-then-decrement so it cannot oversell under `READ COMMITTED`.
   *Hint: single `UPDATE ... WHERE stock >= n` + affected-rows check.*

4. **Default levels.** List the default isolation level for the four engines. *Hint: PostgreSQL RC,
   InnoDB RR, SQL Server RC, Oracle RC.*

5. **Constraint first.** For a booking table, why is `UNIQUE (resource_id, slot_start)` better than
   raising the isolation level? *Hint: cost when the invariant holds.*

---

## 10. Exercises — Advanced (hints / expected approach)

1. **Construct a write-skew test.** Build a schema and two concurrent transactions that violate an
   aggregate invariant *successfully* under PostgreSQL `REPEATABLE READ`, then show `SERIALIZABLE`
   aborting one. Expected approach: cite the `40001` error, and explain the rw-antidependency cycle.

2. **Design a retry-safe job.** A worker runs at `SERIALIZABLE` and must be exactly-once. Expected
   approach: idempotency key + retry with jitter + backoff + a detection of `40001`; discuss why
   retry *outside* the DB transaction is required.

3. **Model the concurrency cost.** For a hot-row counter (`total_cents` on one order), compare
   throughput of (a) `SELECT ... FOR UPDATE` + update, (b) atomic update, (c) `SERIALIZABLE`.
   Expected approach: reason about serialization of the hot row and the abort rate.

4. **InnoDB gap-lock reasoning.** Explain why `READ COMMITTED` reduces deadlocks in InnoDB, and what
   you give up. Expected approach: next-key locks vs. record locks; phantoms; FK checks.

5. **Cross-engine semantics.** Take a transaction written for Oracle `SERIALIZABLE` semantics and
   port it to PostgreSQL `SERIALIZABLE`. Expected approach: identify the write-skew window Oracle
   allowed, add retry handling, and note behavioural differences in snapshot timing.

---

## References

- PostgreSQL: [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- PostgreSQL: [Serializable Snapshot Isolation](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-SERIALIZABLE)
- MySQL: [Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- MySQL: [Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- Microsoft: [SET TRANSACTION ISOLATION LEVEL](https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql)
- Microsoft: [Transaction Locking and Row Versioning Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide)
- Oracle: [Data Concurrency and Consistency](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/data-concurrency-and-consistency.html)
- Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil — *A Critique of ANSI SQL Isolation Levels* (1995): https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
- Ports & Grittner — *Serializable Snapshot Isolation in PostgreSQL* (VLDB 2012): https://arxiv.org/abs/1208.4179
- Cahill, Röhm, Fekete — *Serializable Isolation for Snapshot Databases* (2009): https://dl.acm.org/doi/10.1145/1376616.1376690

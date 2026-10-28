# ACID

> ACID is the most quoted acronym in databases and the least precisely understood. Ask five
> engineers where **Atomicity** comes from and you will hear "rollback," "the WAL," "undo logs," and
> "transactions" — all partly right, all incomplete. This chapter defines each letter precisely, then
> names the exact internal mechanism that implements it in each of the four major engines (WAL and
> redo, undo segments, MVCC and locks, constraints, `fsync`), shows the experiments that let you
> *observe* each guarantee, and finally explores the axes along which real systems deliberately
> relax it — because "we are ACID" is never a binary claim, it is a point on several spectra:
> durability-vs-latency, isolation-vs-concurrency, and consistency-vs-availability.

## TL;DR

- **Atomicity** — all-or-nothing. Implemented by a **write-ahead log** (to redo committed work) plus
  **undo** (to erase uncommitted work). PostgreSQL: WAL + tuple visibility. InnoDB/Oracle: redo + undo.
- **Consistency** — the database moves from one valid state to another; **the DB enforces it, not the
  transaction** (constraints, FKs, triggers, domains, NOT NULL, UNIQUE, CHECK). The "C" in ACID is
  *not* the "C" in CAP.
- **Isolation** — concurrent effects are as if serial. Implemented by **MVCC + a read view** (for
  reads) and **locks** (for writes), tuned by the isolation level (see
  [Isolation levels](./05-isolation-levels.md)).
- **Durability** — a committed transaction survives a crash. Implemented by **fsync'ing the WAL/redo
  record before acknowledging the commit**, with `synchronous_commit` / `innodb_flush_log_at_trx_commit`
  / `DELAYED_DURABILITY` as the explicit dials.
- **Group commit** lets many transactions share one `fsync` — durability and throughput are not
  necessarily in conflict.
- **ACID vs BASE**: BASE (Basically Available, Soft state, Eventually consistent) trades the I and the
  instant-D for availability and scale. Many "NoSQL became ACID" stories are really "a single-node RDBMS
  with tunable durability."
- **CAP**: during a network partition you choose C (linearizability) or A. **PACELC** adds: else
  (no partition), you trade **Latency** vs **Consistency**. Real durability across failures is a
  *replication* property, not a single-node property — async replication silently gives you
  **⁠"ACID-1"**: durable on one node, which may be the node you lose.
- Where strictness is relaxed: `synchronous_commit = off`, `innodb_flush_log_at_trx_commit = 2`,
  SQL Server delayed durability, `READ UNCOMMITTED`, asynchronous replicas, bounded-staleness reads,
  and storing audit logs in a non-durable buffer.

---

## 1. Definitions and core concepts (build from the ground up)

### 1.1 Define each letter precisely

**Atomicity.** A transaction's writes are all eventually applied or none are — even if the process
crashes mid-write. It says nothing about *when* (that is isolation) or *surviving a disk failure*
(that is durability).

**Consistency.** Every committed transaction leaves the database satisfying all declared
**invariants**: constraints, foreign keys, uniqueness, check conditions, triggers. Note carefully: the
*transaction author* is responsible for logical invariants the DB cannot express; the *DB* is
responsible for the ones it can. The classic textbook definition ("the DB transitions between valid
states") is circular; the operational definition is: **the declared constraints hold after commit**.

**Isolation.** The observable effect of concurrently executing transactions is *equivalent to some
serial execution*. Which equivalences are required depends on the level (see
[Isolation levels](./05-isolation-levels.md)); true serializability is the strongest and most people
never use it.

**Durability.** Once the engine has acknowledged `COMMIT`, the effect survives crashes, power loss,
and OS restarts — subject to the honest caveat that "survives" means "is on a medium whose failure
model you accept" (a single disk can still fail permanently; only replication addresses that).

### 1.2 The four properties and their mechanisms

| Property | Question it answers | Mechanism(s) |
|---|---|---|
| **A**tomicity | "all or nothing?" | WAL/redo + undo; visibility on abort |
| **C**onsistency | "are constraints preserved?" | constraints, FKs, triggers, unique indexes |
| **I**solation | "can concurrent txns corrupt the view?" | MVCC read views + locks + isolation level |
| **D**urability | "survives a crash?" | fsync of WAL/redo before ack; group commit |

Each mechanism is a hard engineering artifact described in its own chapter
([B-trees](./02-b-tree-indexes.md), [Transactions](./04-transactions.md),
[MVCC](./07-mvcc.md), [Locks](./06-locks.md)).

---

## 2. Why it matters / what problem it solves

ACID is the contract that lets application teams write straightforward code ("debit, credit, commit")
without hand-building crash recovery and concurrency control. The trade-off is real: the strongest
ACID settings cost latency (`fsync`), throughput (blocking, aborts), and availability (synchronous
replication). Every production system lives somewhere on these spectra; the senior skill is choosing
the point consciously and documenting it.

```
 The ACID budget:

  Atomicity  ───────────────────────  you almost always want the whole thing
  Consistency ───────────────────────  constraints are cheap; logical invariants are on you
  Isolation  ── READ COMMITTED ──RR── SERIALIZABLE ──▶ more correctness, more aborts/blocking
  Durability ── off ── remote_write ── on ── remote_apply ──▶ more latency, less data loss
```

---

## 3. How it works internally

### 3.1 Atomicity in practice

The mechanism is a two-step dance that all four engines implement similarly, with different internal
names:

```
 1. LOG the intent (write-ahead, on disk or in a fast buffer)
 2. APPLY to data pages LAZILY (checkpointer / page flusher)

 On commit:
    - write the COMMIT record, fsync the log → txn is now durable
 On crash:
    - REDO everything after the last checkpoint (committed work is replayed)
    - UNDO anything uncommitted (or never make it visible)
```

| Engine | Redo (roll-forward) | Undo (roll-back) |
|---|---|---|
| PostgreSQL | WAL (`pg_wal`) | **not needed for user data** — aborted tuples are invisible via XID |
| MySQL InnoDB | redo log | undo log (rollback segments) |
| SQL Server | transaction log | rollback via log records (undo) + `tempdb` versions |
| Oracle | redo log files | undo segments |

> **Senior note:** PostgreSQL's lack of a user-data undo log is why it needs `VACUUM`
> ([MVCC](./07-mvcc.md)) and why an aborted transaction's tuples linger as dead versions. InnoDB and
> Oracle undo *in place*, at the cost of maintaining undo segments. Neither is "better"; each moves the
> cost.

### 3.2 The write-ahead rule (the spine of A and D)

> Before any modified data page is written to its permanent location, all log describing that change —
> including the commit record — must be on stable storage.

This one rule gives both atomicity and durability:

```
 Commit timeline (durable, group committed):
   t0  buffer log records in memory
   t1  append commit record to WAL
   t2  fsync WAL  ◀── the ack is sent ONLY when this returns
   t3  later: checkpointer writes data pages (already protected by the log)
```

```mermaid
sequenceDiagram
    participant App
    participant DB as DB engine
    participant WAL
    participant Disk
    App->>DB: UPDATE orders SET ...; COMMIT
    DB->>WAL: append [begin][update][commit]
    WAL->>Disk: fsync()
    Disk-->>WAL: ok
    WAL-->>DB: flush complete
    DB-->>App: COMMIT acknowledged  (now durable)
    Note over DB,WAL: data pages flushed lazily afterwards (safe: log is ahead)
```

### 3.3 Consistency: who enforces what

```
 DB-enforced (ACID's capital C):
   • PRIMARY KEY / UNIQUE      → uniqueness
   • FOREIGN KEY               → referential integrity
   • NOT NULL, CHECK, domains  → value validity
   • EXCLUSION constraints     → range/uniqueness over ranges (PostgreSQL)
   • triggers / materialized constraints

 App-enforced (NOT the DB's job):
   • "an order's total equals the sum of its items"
   • "the sum of allocations ≤ budget"
   • "at least one active admin per tenant"
   • business/state-machine rules (paid → shipped → …)
```

The table above is one of the most useful things to internalize: **the capital C in ACID only covers
constraints**. The rest of your invariants need application logic, explicit locking, or
`SERIALIZABLE` isolation.

> **Senior note:** Prefer a constraint over stronger isolation whenever possible. A `UNIQUE` index
> enforces an invariant at *zero* abort cost when satisfied, whereas `SERIALIZABLE` pays for every
> transaction touched by the predicate, whether or not it conflicts. "Push invariants down into
> constraints" is one of the highest-leverage schema moves you can make.

### 3.4 Isolation: MVCC for reads, locks for writes

```
 Reads  ──▶ read view / snapshot (no locks) ──▶ consistent, non-blocking, versioned
 Writes ──▶ row locks (X) + MVCC versioning ──▶ serialized per row/range
 Conflicts at higher levels ──▶ abort (40001) or block until timeout
```

Isolation is therefore a *dial* over the same machinery (see [Isolation levels](./05-isolation-levels.md)):
the level decides when the snapshot is taken and whether writes check for cross-transaction conflicts.

### 3.5 Durability: the physical floor

Everything above is a lie if the storage stack lies. Durability is only as good as the weakest link in
this chain:

```
 app → DB log buffer → OS page cache → device cache → platters/flash
          │                │                 │
          │                │                 └─ must be non-volatile (or battery-backed)
          │                └─ fsync/fdatasync must actually be honored
          └─ synchronous_commit / flush_log_at_trx_commit decide when we fsync
```

Failure modes people actually hit:
- **Volatile write cache that ignores `fsync`** → "committed" transactions vanish on power loss.
- **`data=writeback` filesystems** → reordering loses recently committed data.
- **Virtualization/qcow2 without cache flushing** → the same, one layer down.
- **The 2018 `fsync`-error "fsyncgate"**: PostgreSQL treated failing `fsync` as success in some
  edge cases, so it could `fsync` the directory before the file. Mitigated by `fsync`-retry logic.

```
  Durability ladder (fastest → slowest), each step adds fsync work:

  volatile cache ──▶ innodb_flush_log_at_trx_commit=0 ──▶ =2 / synchronous_commit=off
      (≈1s loss)         (≈1s loss, no fsync)               (≈ up to 3×wal_writer_delay)
                                                        ──▶ =1 / synchronous_commit=on (0 loss)
                                                        ──▶ + synchronous replication (survives node loss)
```

### 3.6 Atomicity + consistency on crash: recovery in one picture

```
  last checkpoint ────┬─────────────────────────────────────────┬──── crash
                      │  logged, committed work                  │
                      │  (REDO these)                            │
                      │              logged, uncommitted work     │
                      │              (UNDO these ← abort)         │
                      ▼                                          ▼
  recovery replays the log forward from the checkpoint, then rolls back uncommitted txns.
```

### 3.7 Where the light gets dimmed (relaxations, deliberately)

| Relaxation | Engine / setting | What you actually give up |
|---|---|---|
| Deferred commit flush | PG `synchronous_commit=off`; InnoDB `flush_log_at_trx_commit=2`; SQL Server `DELAYED_DURABILITY` | recent committed txns (last ~1 s) on crash |
| Dirty reads | MySQL `READ UNCOMMITTED` | isolation (dirty read) |
| Lower isolation | `READ COMMITTED` everywhere | serializability (write skew, phantoms possible) |
| Async replication | default of most managed replicas | durability across node loss (a commit is on one node) |
| Bounded staleness reads | `MAX_STALENESS` in some DBs / read replicas | freshness |
| Coordination-free writes | NoSQL / BASE stores | immediate consistency |

---

## 4. How to observe / inspect it

### 4.1 PostgreSQL

```sql
-- Durability configuration
SHOW synchronous_commit;
SHOW wal_sync_method;          -- fdatasync / open_sync / ...
SHOW fsync;                    -- must be on in production!
SHOW full_page_writes;         -- torn-page protection after crashes

-- WAL activity (commits, fsyncs, time spent flushing)
SELECT * FROM pg_stat_wal;     -- wal_records, wal_fpi, wal_bytes, wal_sync, wal_sync_time

-- Consistency: constraints in place
SELECT conname, contype, conrelid::regclass
FROM   pg_constraint WHERE conrelid = 'orders'::regclass ORDER BY contype;
```

### 4.2 InnoDB

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'innodb_doublewrite';       -- torn-page protection
SHOW ENGINE INNODB STATUS\G                      -- log sequence numbers, pending writes
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_fsyncs';
```

### 4.3 SQL Server

```sql
SELECT name, recovery_model_desc, delayed_durability_desc FROM sys.databases;
SELECT * FROM sys.dm_io_virtual_file_stats(NULL, NULL);   -- per-file I/O + stalls
-- Log flush waits
SELECT wait_type, wait_time_ms FROM sys.dm_os_wait_stats WHERE wait_type LIKE 'WRITELOG%';
```

### 4.4 Oracle

```sql
SELECT name, value FROM v$sysstat WHERE name IN ('redo writes','redo sync writes','redo blocks written');
SELECT * FROM v$log;                    -- redo groups and status
SELECT begin_time, undoblks, ssolderrcnt FROM v$undostat;
```

### 4.5 The crash test (the experiment that proves durability)

```bash
# Do NOT do this on anything you care about; it belongs in a lab.
# 1) Configure synchronous_commit=on, fsync=on
# 2) Write a known row and COMMIT
# 3) Hard-kill the postmaster
pg_ctl -D $PGDATA stop -m immediate     # or: kill -9 <postmaster pid>
# 4) Restart and verify the row is present
pg_ctl -D $PGDATA start
psql -c "SELECT count(*) FROM durable_test WHERE id=1;"    # must be 1
# 5) Repeat with synchronous_commit=off in a loop → observe that trailing commits can vanish
```

---

## 5. Worked examples and demos on the shared sample schema

Using the e-commerce schema from [chapter 01](./01-clustered-vs-nonclustered-indexes.md).

### 5.1 Atomicity in a checkout, demonstrated

```sql
BEGIN;
INSERT INTO orders (customer_id, status, created_at, total_cents)
VALUES (42, 'pending', now(), 0) RETURNING order_id;    -- say 900123
INSERT INTO order_items (order_id, line_no, product_id, quantity, unit_price_cents)
VALUES (900123, 1, 101, 2, 2599);
-- Simulate a failure before commit
DO $$ BEGIN RAISE EXCEPTION 'payment declined'; END $$;
-- or simply: ROLLBACK;
```

```sql
-- Prove nothing was written (atomicity)
SELECT count(*) FROM orders WHERE order_id = 900123;        -- 0
SELECT count(*) FROM order_items WHERE order_id = 900123;   -- 0
```

### 5.2 Consistency enforced by constraints (not by the app)

```sql
-- These are DB-enforced invariants:
ALTER TABLE order_items
  ADD CONSTRAINT ck_qty_positive CHECK (quantity > 0);
ALTER TABLE products
  ADD CONSTRAINT ck_stock_nonneg CHECK (stock_qty >= 0);

-- The check now rejects the oversell outright — regardless of isolation level:
UPDATE products SET stock_qty = stock_qty - 5 WHERE product_id = 101;  -- errors if stock < 5
```

This is *the* ACID-C win: **you cannot oversell no matter what concurrency does**, because the
invariant is a constraint. Contrast with a "check then update" in application code (see
[Isolation levels](./05-isolation-levels.md) §5.1).

### 5.3 Isolation vs. consistency: the same business rule, three implementations

```sql
-- (a) app-level check (weak): relies on luck/under READ COMMITTED it can fail
SELECT stock_qty FROM products WHERE product_id = 101;
-- (b) explicit lock: correct, blocks
BEGIN;
SELECT stock_qty FROM products WHERE product_id = 101 FOR UPDATE;
UPDATE products SET stock_qty = stock_qty - 1 WHERE product_id = 101;
COMMIT;
-- (c) constraint + atomic write: correct, non-blocking, cheapest
UPDATE products SET stock_qty = stock_qty - 1
WHERE  product_id = 101 AND stock_qty >= 1;   -- + CHECK (stock_qty >= 0) as a backstop
```

### 5.4 Durability: the latency cost of `on`

```bash
# Measure commit throughput with durability on/off on the same box:
pgbench -c 16 -j 4 -T 30 -f commit_only.sql shop
PGOPTIONS='-c synchronous_commit=off' pgbench -c 16 -j 4 -T 30 -f commit_only.sql shop
# Typical: 'on' is fsync-bound (hundreds–thousands TPS on SSD);
#          'off' is much faster but can lose the last fraction of a second of commits on crash.
```

### 5.5 Durability is not a single-node property

```sql
-- Even with synchronous_commit=on, an async replica creates a window:
SELECT application_name, sync_state, replay_lsn
FROM   pg_stat_replication;
--  sync_state = 'async'  →  a commit on the primary may be lost if the primary dies before
--                           the standby has replayed it. Durability across node loss requires
--                           synchronous replication (sync_state='sync' / quorum) or a shared-storage /
--                           consensus-backed engine.
```

```sql
-- Make one standby synchronous (durability across node loss):
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (standby1)';
-- Cost: every commit now waits for the standby to acknowledge → latency.
```

### 5.6 Consistency vs. availability (CAP) on the sample schema

```
 During a network partition between two replicas:

   CP choice: refuse writes on the minority side (or block). Customer sees an error.
              "No order can be created until the network heals."  → consistent, less available.

   AP choice: accept the order on both sides; reconcile later (last-writer-wins / CRDT).
              "The order exists in two places with different totals for a while."
              → available, eventually consistent.

 The e-commerce answer for orders is almost always CP (money is involved); for product
 view counts, likes, or telemetry, AP is fine.
```

---

## 6. Real-world use cases and applications

### 6.1 Payments and ledgers: full ACID, durability on, synchronous replication

A ledger chooses `synchronous_commit=on` plus synchronous replication and `SERIALIZABLE`-or-constraint
invariants, and accepts the latency. Anything less is a business decision about how much money may be
lost in a crash.

### 6.2 Telemetry/metrics ingestion: durability relaxed aggressively

A metrics pipeline chooses `synchronous_commit=off` or `innodb_flush_log_at_trx_commit=2` and batching,
because losing 500 ms of counter increments is acceptable while losing throughput is not. Same engine,
different point on the spectrum — which is why "is our database ACID?" is the wrong question and
"which tables need which guarantees?" is the right one.

### 6.3 The BASE/NoSQL era and the pendulum back

Early NoSQL (Dynamo, Cassandra, MongoDB pre-4.x) embraced **BASE** for horizontal scale: no joins, no
multi-row transactions, eventual consistency. The industry then spent a decade re-adding transactions
(MongoDB multi-document txs in 4.0, Spanner's TrueTime-based external consistency, CockroachDB's
serializable default, DynamoDB Transactions) — evidence that **application teams strongly prefer the
ACID contract when it is available at scale**.

### 6.4 The "we have replicas, so we are durable" fallacy

A team runs RDS with an asynchronous read replica, loses the primary's AZ, and discovers that the last
few seconds of committed orders are gone (they were never replicated). Fix: multi-AZ *synchronous*
replication, and monitoring replication lag as a durability SLO, not just a performance metric.

### 6.5 CAP in practice: the "CP vs AP" table

| System | Nano-consistency | Practical CAP/PACELC posture |
|---|---|---|
| PostgreSQL/MySQL single node | full ACID | CP-ish (blocks when primary unavailable) |
| Async-replicated RDBMS | ACID locally, eventual globally | favors A; loses recent writes on failover |
| Synchronous replication / quorum RDBMS | ACID + node-durable | CP; higher write latency |
| Google Spanner | external consistency via TrueTime | CP; hides partition with global clock |
| DynamoDB | tunable | AP by default; strongly consistent reads by opt-in |
| Cassandra (QUORUM) | tunable | AP(+L) with tunable consistency |

---

## 7. Patterns and anti-patterns

### Patterns

| Pattern | Why |
|---|---|
| Encode invariants as constraints (UNIQUE/FK/CHECK) | zero-abort enforcement of ACID-C |
| Keep transactions short; no network/user I/O | protects isolation and cleanup |
| Choose durability per table, not per database | telemetry and ledgers differ |
| Synchronous replication for money; async for derived data | matches cost to value |
| Retry on `40001`/`40P01`/`1213`/`1205` | isolation failures are normal |
| Verify `fsync=on`, non-volatile caches, correct FS mode | durability's real floor |
| Monitor replication lag as a durability metric | catches the durable-1 window |

### Anti-patterns

| Anti-pattern | Damage |
|---|---|
| "It's ACID, so we're safe" without checking durability settings | silent data loss |
| `fsync=off` / volatile device cache in production | ack'd commits lost on power loss |
| Assuming the DB enforces business invariants | inconsistent data |
| Long transactions + relaxed settings | bloat + lost writes (worst of both) |
| Using async replicas as "backups" | lag/loss window; not a backup |
| Turning on `SERIALIZABLE` without retries | user-visible failures |
| Treating CAP as "pick two" and ignoring latency | misses PACELC |

---

## 8. Cross-engine differences

| Aspect | PostgreSQL | MySQL InnoDB | SQL Server | Oracle |
|---|---|---|---|---|
| Redo (durability) | WAL (`pg_wal`) | redo log | transaction log | redo log files |
| Undo (atomicity) | tuple visibility (no user undo) | undo log | log-based undo + `tempdb` versions | undo segments |
| Default durability | `synchronous_commit=on` | `innodb_flush_log_at_trx_commit=1` | full (delayed durability off) | `COMMIT` waits for redo |
| Durability relaxation | `synchronous_commit=off` | `=2` / `=0` | `DELAYED_DURABILITY=ON/FORCED` | `COMMIT WRITE NOWAIT/BATCH` |
| Crash-recovery unit | WAL replay from checkpoint | redo replay + undo rollback | log replay (redo/undo) | redo replay + undo |
| Consistency primitives | constraints, EXCLUDE, triggers | constraints, triggers | constraints, triggers | constraints, triggers |
| Isolation default | READ COMMITTED | REPEATABLE READ | READ COMMITTED | READ COMMITTED |
| True serializable | ✅ SSI | ✅ locking | ✅ range locks | ❌ (SI only) |
| Sync replication | `synchronous_standby_names` | `semi-sync` plugin / group replication | Always On synchronous commit | Data Guard SYNC |
| Torn-page protection | `full_page_writes` | doublewrite buffer | page checksums + log | block checksums / RMAN |
| Garbage collection | `VACUUM` | purge threads | — | undo retention |

---

## 9. Exercises — Intermediate (hints only)

1. **Map each letter to a mechanism.** For Atomicity, name the two logs involved and which one lets you
   *finish* and which lets you *erase*. *Hint: redo/WAL vs. undo/visibility.*

2. **Which C?** Explain why the "C" in ACID is different from the "C" in CAP. *Hint: constraints vs.
   linearizability.*

3. **Constraint or isolation?** For "an order's total must equal the sum of its items," can you express
   it as a constraint? If not, what are your options? *Hint: consider triggers / deferred constraints /
   materialized totals.*

4. **Durability dial.** Name the settings that relax durability in the four engines and roughly how
   much data each can lose on a crash. *Hint: ~1 s windows for MySQL/SQL Server; 3× `wal_writer_delay`
   for PostgreSQL.*

5. **Replication.** Why does `synchronous_commit=on` not protect you if your only replica is async?
   *Hint: durability across node loss.*

---

## 10. Exercises — Advanced (hints / expected approach)

1. **Design a per-table durability policy.** Given a schema with `orders`, `payments`, `page_views`,
   and `audit_log`, choose `synchronous_commit` (or equivalent) per table and justify each. Expected
   approach: use per-transaction `SET LOCAL synchronous_commit`, classify by business value and
   acceptable loss window.

2. **Measure the durability–latency curve.** Benchmark commit latency with `synchronous_commit` at
   `off`, `local`, `on`, `remote_write`, `remote_apply`. Expected approach: plot p50/p99; identify the
   fsync and network components; relate to SLOs.

3. **Prove the ACID-C boundary.** Construct an invariant that constraints *cannot* enforce (e.g., a
   sum bound), show it failing under `READ COMMITTED` and `REPEATABLE READ`, and fix it three ways
   (SERIALIZABLE + retry, materialized counter row, trigger + serialization). Expected approach:
   analyse cost/throughput of each.

4. **CAP/PACELC analysis of your stack.** Take your production topology (primary + replica + cache)
   and describe the behaviour during a partition for reads and writes. Expected: name which operations
   are CP vs AP, and where staleness is visible to users.

5. **Crash-consistency test design.** Design a test that verifies durability end to end (app → DB →
   device), including a power-loss simulation and an `fsync`-lying device. Expected approach: describe
   the failure you are trying to detect at each layer and the tool you'd use.

---

## References

- PostgreSQL: [Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- PostgreSQL: [Write-Ahead Logging](https://www.postgresql.org/docs/current/wal-intro.html)
- PostgreSQL: [`synchronous_commit` / WAL configuration](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- PostgreSQL: [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- MySQL: [InnoDB Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html) and [Undo Logs](https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-logs.html)
- Microsoft: [Transaction Log Architecture and Management Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide)
- Microsoft: [Control Transaction Durability](https://learn.microsoft.com/en-us/sql/relational-databases/logs/control-transaction-durability)
- Oracle: [Transactions](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/transactions.html)
- Gilbert & Lynch — *Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services* (2002): https://dl.acm.org/doi/10.1145/564585.564601
- Abadi — *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC, 2012): https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf
- PostgreSQL wiki: [Fsync Errors](https://wiki.postgresql.org/wiki/Fsync_Errors)
- Brewer — [CAP Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)

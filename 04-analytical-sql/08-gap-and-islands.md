# Gap-and-Islands Problems

> Gap-and-islands is the name for a whole family of questions that all reduce to the same shape:
> *group consecutive rows into runs, then tell me something about the runs.* Consecutive-day login
> streaks. Missing invoice numbers. How long an account stayed in `paused`. Whether a sensor reported
> continuously. Which services were up for more than an hour. The technique is famously short — a
> `LAG`, a flag, and a running `SUM` — but the *reason* it works, the two different formulations, and
> the places where the naive version silently returns wrong answers are what separate an engineer who
> has memorised a pattern from one who can apply it to a problem nobody has seen before.

## TL;DR

- Two formulations, and you must know which one your problem needs:

  | Formulation | Grouping key | Requires |
  |-------------|--------------|----------|
  | **A. LAG-flag + running SUM** (universal) | `SUM(is_new_run) OVER (ORDER BY t ROWS UNBOUNDED PRECEDING)` | nothing special |
  | **B. Value − `ROW_NUMBER()`** (dense sequences) | `value − ROW_NUMBER() OVER (ORDER BY value)` | the value must be **arithmetic** (integer, date) and **dense in the sense of consecutive-ness** |
  | **C. `ROW_NUMBER()` − `ROW_NUMBER()`** (run-length encoding) | `ROW_NUMBER() OVER (ORDER BY t) − ROW_NUMBER() OVER (PARTITION BY value ORDER BY t)` | a unique ordering column |

  A is the general answer; B is a neat shortcut for dates and integers; C is the cleanest way to
  run-length-encode a categorical column.
- **The `LAG` flag is "does this row start a new run?"** A new run starts when the row is the first
  one, or when the gap to the previous row exceeds the tolerance, or when the value changes. Three
  conditions, all expressible in one `CASE`.
- **The running `SUM` of that flag is a run identifier** because the flag is `1` exactly at each run
  start: the cumulative total is constant inside a run and increments at each boundary. That is the
  entire trick.
- **A wide tolerance IS a different problem.** "Consecutive days" (tolerance 0) and "consecutive days
  allowing weekends and holidays" (tolerance = the number of calendar days between business days) are
  not the same query. Handle tolerance either by comparing to `1 + tolerance`, or — better for
  holidays — by joining to a calendar dimension with a dense **business-day sequence number** and
  applying formulation B to that.
- **Overlapping intervals are a *different* problem** that looks similar. Merging them requires
  `MAX(end) OVER (… ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)` — a **frame that excludes the
  current row** — because the test is "does this interval start before *any* previous interval has
  ended?" That is the one place in this chapter where the frame clause is load-bearing rather than
  decorative.
- **`DISTINCT` before you count.** Events are not days. "Consecutive days" must be computed on the
  deduplicated `(entity, day)` set, or a user with five events on Monday and one on Tuesday appears
  as a six-row run whose "length" is meaningless.
- **Gaps and islands are two halves of the same answer.** Every islands query has a companion: *where
  are the missing values?* `LEAD(value) − value − 1` produces the gaps; the island boundaries come
  from the same ordered scan.
- **The template is domain-independent.** Streaks, uptime, SCD-2 validity intervals, retention runs,
  inventory level periods, incident windows, sensor quality — same four steps every time.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Island / run / streak** | A maximal set of consecutive rows (by some ordering) satisfying a continuity condition. |
| **Gap** | A break in a sequence — missing values, missing dates, or a discontinuity in a state. |
| **Continuity condition** | What "consecutive" means: gap ≤ tolerance (time), value increments by 1 (integers), value is unchanged (state). |
| **Tolerance** | The maximum gap still considered part of the same run. `0` for strict consecutiveness. |
| **Run identifier (group key)** | A value constant within a run, used in the `GROUP BY`. |
| **Run-length encoding (RLE)** | Collapsing a run of identical values into one row: `(value, start, end, length)`. |
| **Validity interval** | `[valid_from, valid_to)` describing a period during which a state held (SCD-2). |
| **Island boundaries** | The min and max of the ordering column within a run. |
| **Streak** | A count-based island (usually days). |
| **Current streak** | The streak ending at the most recent observation. |
| **Longest streak** | `MAX(run_length)` over entities. |
| **Overlapping-interval merge** | A distinct problem: union of intervals that may overlap or touch. |
| **Peer group** | Rows equal under the window's ordering (relevant to formulation B's correctness). |

### 1.2 The two shapes

```
FORMULATION A — flag + running sum (works for anything ordered)

  t      value   prev    changed?  flag   grp = SUM(flag)
  ─────────────────────────────────────────────────────────
  Mon    up      NULL    yes       1      1     ┐
  Tue    up      up      no        0      1     │ island 1 (up, 2 rows)
  Wed    down    up      yes       1      2     ┐
  Thu    down    down    no        0      2     │ island 2 (down, 2 rows)
  Fri    down    down    no        0      2     │
  Sat    up      down    yes       1      3     ┘ island 3 (up, 1 row)

FORMULATION B — value − ROW_NUMBER() on a dense arithmetic sequence

  value   rn    value - rn
  ────────────────────────
  101     1     100   ┐
  102     2     100   │ island 1 (101..102, length 2)
  105     3     102   ┐   ← gap at 103, 104
  106     4     102   │ island 2 (105..106, length 2)
  107     5     102   ┘
  110     6     104   ┘ island 3 (110, length 1)
```

Both produce one constant per run. A is universal; B is shorter but needs arithmetic.

### 1.3 Choosing the formulation

| Situation | Use | Why |
|-----------|-----|-----|
| Ordered by time, run defined by a gap threshold | **A** | no arithmetic on the ordering column |
| Ordered by a dense integer or date sequence | **B** | shortest, and inherently handles "missing values" |
| Categorical state, run = consecutive identical values | **A** (running sum) or **C** (two row numbers) | C avoids the running sum |
| Ordered by a string or a non-arithmetic type | **A** or **C** | B cannot subtract |
| Overlapping intervals to merge | A **plus a frame that looks backwards** | the test needs all previous ends |
| Gaps (missing values) rather than islands | `LEAD`/`LAG` difference | the complement of the island boundaries |
| Tolerant consecutiveness (weekends/holidays) | A with a larger tolerance, or a **calendar sequence** + B | a calendar sequence is exact; a tolerance is a heuristic |

## 2. Why it matters / what problem it solves

Gap-and-islands is the highest-leverage pattern in analytics after the basic aggregate, because a
surprising number of business questions are secretly about runs:

| Domain | The island | The gap |
|--------|-----------|---------|
| **Product / growth** | consecutive-day login streak | days the user was absent |
| **Subscriptions** | months the account stayed on a paid plan | the cancellation month |
| **SCD-2 dimensions** | the period a row version was valid | the version's start/end boundaries |
| **IoT / telemetry** | contiguous periods a device reported | data outages |
| **Reliability** | contiguous periods a service was healthy | outage windows |
| **Finance / audit** | contiguous ranges of invoice numbers | missing invoice numbers |
| **Inventory** | periods a SKU stayed at a stock level | stock-out windows |
| **Marketing** | weeks a user was an active subscriber | lapsed periods |
| **Security** | contiguous periods a host talked to one peer | new connections (lateral movement) |
| **Data quality** | consecutive days of completed pipeline runs | broken runs, "days since last success" |
| **HR** | tenure periods, promotions | employment breaks |
| **Ads** | consecutive days a campaign hit its budget | pacing failures |
| **Gaming** | win/loss runs, "tilt" detection | — |

Costs of getting it wrong:

- **The `DISTINCT` omission.** A "consecutive-day streak" computed over the raw event table counts
  *events*, not days, so a power user with 40 events in one day gets a "40-day streak". This is
  extremely common and produces absurd leaderboard metrics.
- **The tolerance confusion.** A "streak" that ignores weekends without saying so produced a
  leaderboard where every user had a streak long enough to be meaningless. The fix was a business-day
  calendar, and it changed the top-10 completely.
- **The overlapping-interval bug.** A validity-interval query that used
  `MAX(end) OVER (ORDER BY s)` (with the default frame, which *includes the current row*) merged
  intervals incorrectly whenever a row was contained inside another. The fix was
  `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING`.

## 3. The pattern / technique explained

### 3.1 Formulation A — the universal four steps

```
1. DEDUPE      collapse the facts to the grain you mean (e.g. one row per entity per day)
2. ORDER       define the ordering, with a deterministic tiebreak
3. FLAG        is_new_run = 1 when this row starts a new run
4. NUMBER      run_id = SUM(is_new_run) OVER (… ROWS UNBOUNDED PRECEDING)
5. AGGREGATE   GROUP BY run_id
```

The `FLAG` condition is a `CASE` with up to three arms, depending on the problem:

```sql
CASE
    WHEN LAG(<order_key>) OVER w IS NULL                              -- first row
      OR LAG(<order_key>) OVER w <> <order_key> - <step>              -- strict consecutiveness
      OR <value> IS DISTINCT FROM LAG(<value>) OVER w                 -- state change
    THEN 1 ELSE 0
END AS is_new_run
```

Note that the "gap" condition and the "value changed" condition are alternatives, not conjuncts: you
use the one that matches your definition of continuity. For a *state* RLE you use the value change;
for a *streak of days* you use the gap.

### 3.2 Formulation B — `value − ROW_NUMBER()`

If the ordering key is an arithmetic sequence and "consecutive" means "steps by exactly 1", then:

```
For a row in a run of consecutive values v, v+1, v+2, …,
  v_i − i  is CONSTANT  ⟺  v_i = v_1 + (i − 1)
```

The difference is a run identifier. It works because `ROW_NUMBER()` increases by exactly 1 each row,
and so does a dense consecutive sequence. Wherever the sequence *skips*, the difference jumps.

```sql
SELECT <entity>,
       MIN(<value_key>) AS run_start, MAX(<value_key>) AS run_end,
       COUNT(*)         AS run_length,
       MIN(<value_key>) - <step> * 0 /* or a date arithmetic expression */ AS run_id
FROM (
    SELECT *, <value_key> - ROW_NUMBER() OVER (PARTITION BY <entity> ORDER BY <value_key>) AS grp
    FROM   <deduped_rows>
) x
GROUP BY <entity>, grp
ORDER BY <entity>, run_start;
```

For types where subtraction is awkward (SQL Server's `date − int` is not portable, Oracle's returns
days), use `ROW_NUMBER()` on a **dense sequence table** instead: join to a calendar/sequence
dimension that assigns `seq = 1, 2, 3, …` to every valid value, then subtract `seq` (an integer) from
`ROW_NUMBER()` (an integer). That is fully portable and also the exact way to implement
"tolerate weekends and holidays".

### 3.3 Formulation C — two `ROW_NUMBER()`s (cleanest RLE)

```sql
SELECT <value>,
       MIN(<seq>) AS run_start, MAX(<seq>) AS run_end, COUNT(*) AS run_length
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY <seq>)
         - ROW_NUMBER() OVER (PARTITION BY <value> ORDER BY <seq>) AS grp
    FROM   <rows>
) x
GROUP BY <value>, grp
ORDER BY run_start;
```

Why it works: `ROW_NUMBER() OVER (ORDER BY seq)` counts *all* rows; `ROW_NUMBER() OVER (PARTITION BY
value ORDER BY seq)` counts only rows with the same value. Inside a run of identical values both
counters advance together, so the difference is constant; when the value changes, only the first
counter keeps advancing, so the difference jumps. No running sum required, and it is exact for
arbitrary (even non-arithmetic) ordering keys.

Verify on `A, A, B, B, B, A` (`seq` = 1..6): the differences are `0, 0, 2, 2, 2, 3` — constant in each
run, distinct across runs. ✅

### 3.4 Gaps: the complement

Given an ordered set of values, the gaps are adjacent-pair differences greater than the step:

```sql
SELECT <value> AS gap_after,
       LEAD(<value>) OVER (ORDER BY <value>)                          AS gap_before,
       LEAD(<value>) OVER (ORDER BY <value>) - <value> - 1             AS missing_count
FROM   <ordered_values>
WHERE  ... -- filter in an outer query; window functions are not allowed in WHERE
```

Because window functions cannot appear in `WHERE`, the filter goes one level out:

```sql
WITH diffs AS (
    SELECT v, LEAD(v) OVER (ORDER BY v) AS next_v FROM values WHERE <filters>
)
SELECT v AS gap_after, next_v AS gap_before, next_v - v - 1 AS missing_count
FROM   diffs
WHERE  next_v IS NOT NULL AND next_v - v > 1
ORDER  BY v;
```

**You cannot find every gap from the data alone.** A gap *before* the first observed value and a gap
*after* the last one are invisible — the data does not know what should have been there. Those need an
expected sequence: a calendar, a `generate_series`, or a range table. Always ask "what is the complete
domain?" before claiming a gap list is exhaustive.

### 3.5 Overlapping intervals — the frame-critical case

Merging overlapping intervals is the one gap-and-islands variant where the **frame clause is not
optional**:

```sql
SELECT grp, MIN(s) AS merged_start, MAX(e) AS merged_end
FROM (
    SELECT *,
           SUM(CASE WHEN s <= MAX(e) OVER (ORDER BY s, e
                                           ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)
                    THEN 0 ELSE 1 END)
               OVER (ORDER BY s, e
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
    FROM intervals
) x
GROUP BY grp;
```

The inner `MAX(e) … AND 1 PRECEDING` deliberately **excludes the current row**: the question is "does
this interval start at or before the farthest end seen *so far*?", and if you include the current row
the answer is trivially yes for every row (since `e ≥ s`), so nothing ever merges. With the default
`RANGE UNBOUNDED PRECEDING` frame, the current row *and its peers* are included — a silent bug.

That is the deepest lesson in the chapter: **frames are semantics, not performance tuning.**

### 3.6 The generalisable template

```
DEFINE:
  entity      — what may have runs (user, account, device, service, SKU, host)
  order_key   — what orders the rows (date, timestamp, sequence, id)
  continuity  — what makes two adjacent rows part of the same run
                 • gap <= tolerance           (time streaks)
                 • value increments by step   (dense sequences) → formulation B
                 • value unchanged            (state RLE)        → formulation C
  dedupe      — the grain to which you must collapse before measuring runs

IMPLEMENT:
  dedupe → order (with tiebreak) → flag (CASE) → number (SUM of flag) → aggregate by run id
```

## 4. Step-by-step worked example (with sample data and expected output tables)

Using the shared schema from `01-ranking.md`.

### 4.1 Example 1 — consecutive-day streaks per user

**Problem statement.** For each user, find every run of consecutive days on which they were active,
and their longest run.

**Naive/incorrect approach.** Counting events instead of days:

```sql
-- ❌ "consecutive days" over raw events: user 1's 2024-01-05 with 5 events
--    becomes a 5-row "run"
SELECT user_id, COUNT(*) FROM events GROUP BY user_id;
```

**Correct approach — dedupe to `(user_id, day)` first, then use formulation B:**

```sql
WITH active_days AS (
    SELECT DISTINCT user_id, event_time::date AS d FROM events
), numbered AS (
    SELECT user_id, d,
           d - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY d) AS grp
    FROM   active_days
)
SELECT user_id,
       MIN(d)     AS streak_start,
       MAX(d)     AS streak_end,
       COUNT(*)   AS streak_days
FROM   numbered
GROUP  BY user_id, grp
ORDER  BY user_id, streak_start;
```

| user_id | streak_start | streak_end | streak_days |
|--------:|--------------|-----------|------------:|
| 1 | 2024-01-05 | 2024-01-06 | 2 |
| 1 | 2024-01-12 | 2024-01-12 | 1 |
| 1 | 2024-01-19 | 2024-01-19 | 1 |
| 1 | 2024-01-26 | 2024-01-26 | 1 |
| 1 | 2024-02-02 | 2024-02-02 | 1 |
| 2 | 2024-01-07 | 2024-01-08 | 2 |
| 2 | 2024-01-14 | 2024-01-14 | 1 |
| 2 | 2024-01-21 | 2024-01-21 | 1 |
| 2 | 2024-02-04 | 2024-02-04 | 1 |
| 3 | 2024-01-15 | 2024-01-16 | 2 |
| 3 | 2024-01-22 | 2024-01-22 | 1 |
| 3 | 2024-02-05 | 2024-02-05 | 1 |
| 4 | 2024-02-01 | 2024-02-02 | 2 |
| 4 | 2024-02-09 | 2024-02-09 | 1 |
| 4 | 2024-02-16 | 2024-02-16 | 1 |
| 4 | 2024-03-01 | 2024-03-01 | 1 |
| 5 | 2024-02-14 | 2024-02-15 | 2 |
| 5 | 2024-02-21 | 2024-02-21 | 1 |
| 5 | 2024-02-28 | 2024-02-28 | 1 |
| 6 | 2024-03-02 | 2024-03-03 | 2 |
| 6 | 2024-03-09 | 2024-03-09 | 1 |
| **7** | **2024-03-04** | **2024-03-07** | **4** |
| **7** | **2024-03-09** | **2024-03-11** | **3** |

Total runs: **23**, covering all **34** active user-days. Verify with
`SELECT user_id, COUNT(DISTINCT event_time::date) FROM events GROUP BY 1` (6 + 5 + 4 + 5 + 4 + 3 + 7 =
34). The run table breaks down as 15 runs of length 1, 6 of length 2, one of length 3 and one of
length 4: `15×1 + 6×2 + 3 + 4 = 34` rows across `15 + 6 + 1 + 1 = 23` runs. ✅ Run that check yourself —
**a run table whose lengths do not sum back to the input row count is the first thing to test**, and
the second thing is that runs of the same entity do not overlap.

What matters is the shape: **user 7 is the only user with a run longer than 2 days**, and their two
runs (4 and 3 days) are separated by a single missing day (2024-03-08).

The `grp` column is worth printing once to convince yourself of the mechanism:

| user_id | d | rn | d − rn (= grp) |
|--------:|---:|----:|----------------:|
| 7 | 2024-03-04 | 1 | 2024-03-03 |
| 7 | 2024-03-05 | 2 | 2024-03-03 |
| 7 | 2024-03-06 | 3 | 2024-03-03 |
| 7 | 2024-03-07 | 4 | 2024-03-03 |
| 7 | 2024-03-09 | 5 | 2024-03-04 |
| 7 | 2024-03-10 | 6 | 2024-03-04 |
| 7 | 2024-03-11 | 7 | 2024-03-04 |

`grp` is constant inside each run and different across runs. That is the island identifier.

### 4.2 Example 2 — longest streak, current streak, and a streak leaderboard

**Problem statement.** Two different questions that people call "the streak":
**(a)** the longest run ever, and **(b)** the run that is still alive as of an as-of date.

```sql
WITH streaks AS (
    /* … the streak table from §4.1 … */
)
-- (a) longest streak per user, and the global leaderboard
SELECT user_id, MAX(streak_days) AS longest_streak
FROM   streaks
GROUP  BY user_id
ORDER  BY longest_streak DESC, user_id;
```

| user_id | longest_streak |
|--------:|---------------:|
| 7 | 4 |
| 1 | 2 |
| 2 | 2 |
| 3 | 2 |
| 4 | 2 |
| 5 | 2 |
| 6 | 2 |

```sql
-- (b) current streak as of a fixed as-of date (only runs that touch it)
WITH params AS (SELECT DATE '2024-03-11' AS as_of)
SELECT s.user_id, s.streak_start, s.streak_end, s.streak_days AS current_streak
FROM   streaks s CROSS JOIN params p
WHERE  s.streak_end = p.as_of
ORDER  BY s.streak_days DESC, s.user_id;
```

| user_id | streak_start | streak_end | current_streak |
|--------:|--------------|-----------|---------------:|
| 7 | 2024-03-09 | 2024-03-11 | 3 |

Only user 7 has a live streak at 2024-03-11 — everyone else's last activity is older. This is the
"as-of" discipline again (chapter 05): the question "what is the current streak?" is meaningless
without a reference date, and a dashboard that omits it will disagree with another dashboard that
chooses differently.

> **The `as_of` must be a parameter, not `CURRENT_DATE`.** A pinned as-of date makes the query
> reproducible and testable; `CURRENT_DATE` makes yesterday's answer unreproducible today. This is the
> single most common reason two "current streak" dashboards disagree.

**The gap-tolerant variant.** If "consecutive" should allow weekends, join to a business-day calendar
and apply formulation B to *its* dense sequence, not to the raw date:

```sql
WITH biz AS (
    SELECT d, ROW_NUMBER() OVER (ORDER BY d) AS biz_seq
    FROM   dim_calendar
    WHERE  is_business_day
), joined AS (
    SELECT a.user_id, a.d, b.biz_seq
    FROM   (SELECT DISTINCT user_id, event_time::date AS d FROM events) a
    JOIN   biz b ON b.d = a.d
), numbered AS (
    SELECT *, biz_seq - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY d) AS grp
    FROM   joined
)
SELECT user_id, MIN(d) AS streak_start, MAX(d) AS streak_end, COUNT(*) AS streak_days
FROM   numbered GROUP BY user_id, grp ORDER BY user_id, streak_start;
```

The difference between this and §4.1 is **one join**: `d − ROW_NUMBER()` becomes
`biz_seq − ROW_NUMBER()`. A holiday-calendar rule becomes a data change, not a SQL change — which is
exactly as it should be.

### 4.3 Example 3 — finding gaps (the complement)

**Problem statement.** Invoice numbers should run continuously. Find the missing ones.

```sql
WITH invoices(n) AS (
    VALUES (1001), (1002), (1005), (1006), (1007), (1008), (1010)
), diffs AS (
    SELECT n, LEAD(n) OVER (ORDER BY n) AS next_n FROM invoices
)
SELECT n AS gap_after, next_n AS gap_before,
       next_n - n - 1 AS missing_count,
       (n + 1)::text || '..' || (next_n - 1)::text AS missing_range
FROM   diffs
WHERE  next_n IS NOT NULL AND next_n > n + 1
ORDER  BY n;
```

| gap_after | gap_before | missing_count | missing_range |
|----------:|-----------:|--------------:|---------------|
| 1002 | 1005 | 2 | `1003..1004` |
| 1008 | 1010 | 1 | `1009..1009` |

Two gaps found. **And here is the trap**: what about `1000` and `1009+1 = 1010+1`? If the invoice
sequence should have started at 1000, that gap is invisible. If it should have continued to 1099,
that gap is invisible too. The query answers "gaps *between* observed values" — nothing more. To find
boundary gaps you need the expected domain:

```sql
WITH expected AS (
    SELECT generate_series(1001, 1010) AS n
)
SELECT e.n
FROM   expected e
LEFT   JOIN (SELECT n FROM invoices) i USING (n)
WHERE  i.n IS NULL
ORDER  BY e.n;
```

| n |
|---:|
| 1003 |
| 1004 |
| 1009 |

This is the more useful form for a data-quality gate, because **it produces a row per missing value**
(joinable to a quarantine table) rather than a range. It also makes the expected domain *explicit*,
which is the actual deliverable: a `generate_series` from a business rule, not from the data.

### 4.4 Example 4 — islands of a continuous state (SCD-2 validity intervals)

**Problem statement.** Given a status log with one row per change (and possibly repeated identical
statuses), produce the **validity intervals**.

```sql
WITH status_log(t, status) AS (
    VALUES (TIMESTAMP '2024-01-01 00:00', 'active'),
           (TIMESTAMP '2024-01-05 00:00', 'active'),
           (TIMESTAMP '2024-01-10 00:00', 'paused'),
           (TIMESTAMP '2024-01-15 00:00', 'paused'),
           (TIMESTAMP '2024-01-20 00:00', 'active'),
           (TIMESTAMP '2024-01-25 00:00', 'active'),
           (TIMESTAMP '2024-01-30 00:00', 'churned')
), flagged AS (
    SELECT t, status,
           CASE WHEN status IS DISTINCT FROM LAG(status) OVER (ORDER BY t)
                THEN 1 ELSE 0 END AS is_new_run
    FROM   status_log
), grouped AS (
    SELECT t, status, is_new_run,
           SUM(is_new_run) OVER (ORDER BY t
                                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
    FROM   flagged
)
SELECT status,
       MIN(t) AS valid_from,
       -- the interval ends where the NEXT island begins
       LEAD(MIN(t)) OVER (ORDER BY MIN(t)) AS valid_to,
       COUNT(*) AS observations
FROM   grouped
GROUP  BY grp, status
ORDER  BY valid_from;
```

| status | valid_from | valid_to | observations |
|--------|------------|----------|-------------:|
| active | 2024-01-01 00:00 | 2024-01-10 00:00 | 2 |
| paused | 2024-01-10 00:00 | 2024-01-20 00:00 | 2 |
| active | 2024-01-20 00:00 | 2024-01-30 00:00 | 2 |
| churned | 2024-01-30 00:00 | `NULL` (still current) | 1 |

Three things to notice:

1. **Two separate `active` islands.** The account was active, paused, active again, churned. The
   `grp` key is what keeps them apart — a `GROUP BY status` would have merged them into one row with
   a 29-day "validity", which is simply wrong. This is the classic RLE-vs-GROUP-BY failure.
2. **`valid_to` comes from `LEAD(MIN(t))`.** The interval ends where the next island starts. This is
   the SCD-2 validity-interval idiom, and it must be applied *after* the `GROUP BY` (which is why the
   window function is over `MIN(t)`, not over `t`).
3. **The last interval is open.** `NULL` means "still current as of the data's end". Emitting a
   sentinel like `9999-12-31` is a common alternative; what matters is that the choice is explicit and
   consistent, because a `BETWEEN` join against a sentinel works but a `<` join against `NULL` does
   not.

The portable form (no `LEAD` over an aggregate) uses two levels of CTE:

```sql
WITH islands AS (
    SELECT status, MIN(t) AS valid_from FROM grouped GROUP BY grp, status
)
SELECT status, valid_from,
       LEAD(valid_from) OVER (ORDER BY valid_from) AS valid_to
FROM   islands
ORDER  BY valid_from;
```

### 4.5 Example 5 — run-length encoding a categorical column (formulation C)

```sql
WITH readings(seq, state) AS (
    VALUES (1,'up'), (2,'up'), (3,'up'),
           (4,'down'), (5,'down'),
           (6,'up'),
           (7,'degraded'), (8,'degraded'), (9,'degraded'), (10,'degraded')
)
SELECT state, MIN(seq) AS run_start, MAX(seq) AS run_end, COUNT(*) AS run_length
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY seq)
         - ROW_NUMBER() OVER (PARTITION BY state ORDER BY seq) AS grp
    FROM   readings
) x
GROUP BY state, grp
ORDER BY run_start;
```

| state | run_start | run_end | run_length |
|-------|----------:|--------:|-----------:|
| up | 1 | 3 | 3 |
| down | 4 | 5 | 2 |
| up | 6 | 6 | 1 |
| degraded | 7 | 10 | 4 |

Two observations:

- **`up` appears twice**, and formulation C keeps them separate — the two `ROW_NUMBER()`s jump every
  time the state changes, which is exactly what a run identifier must do.
- **This is the same output as formulation A** on the same data, but with no running sum. That is
  useful on engines where a running `SUM` over a very wide partition is the bottleneck, and it is
  arguably more elegant.

### 4.6 Example 6 — merging overlapping intervals (the frame-critical case)

**Problem statement.** A customer had four service contracts, some overlapping. Produce the merged
coverage periods.

```sql
WITH contracts(id, s, e) AS (
    VALUES (1, DATE '2024-01-01', DATE '2024-01-10'),
           (2, DATE '2024-01-05', DATE '2024-01-15'),
           (3, DATE '2024-01-20', DATE '2024-01-25'),
           (4, DATE '2024-01-22', DATE '2024-01-30')
), marked AS (
    SELECT id, s, e,
           MAX(e) OVER (ORDER BY s, e
                        ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS max_end_so_far
    FROM   contracts
), flagged AS (
    SELECT *,
           CASE WHEN max_end_so_far IS NULL OR s > max_end_so_far THEN 1 ELSE 0 END AS is_new_run
    FROM   marked
), grouped AS (
    SELECT *, SUM(is_new_run) OVER (ORDER BY s, e
                                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS grp
    FROM   flagged
)
SELECT grp, MIN(s) AS coverage_start, MAX(e) AS coverage_end, COUNT(*) AS contracts
FROM   grouped GROUP BY grp ORDER BY grp;
```

| grp | coverage_start | coverage_end | contracts |
|----:|----------------|--------------|----------:|
| 1 | 2024-01-01 | 2024-01-15 | 2 |
| 2 | 2024-01-20 | 2024-01-30 | 2 |

**Now demonstrate the bug.** Change the frame to the default (which includes the current row):

| variant | max_end_so_far at row 2 | is_new_run | groups produced | correct? |
|---------|------------------------|:----------:|----------------:|:--------:|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING` | 2024-01-10 | 0 (merges) | **2** | ✅ |
| default `RANGE UNBOUNDED PRECEDING AND CURRENT ROW` | 2024-01-15 (includes row 2's own `e`) | always 0 | **1** | ❌ |

With the default frame, `max_end_so_far` at row 2 is `MAX(2024-01-10, 2024-01-15) = 2024-01-15`, and
`s = 2024-01-05 ≤ 2024-01-15`, so `is_new_run` is `0` — which *happens* to be right here — but at row
3 the frame includes rows 1..3, so `max_end_so_far = 2024-01-25` and `s = 2024-01-20 ≤ 2024-01-25`
gives `is_new_run = 0`, wrongly merging the second block into the first. **The result is 1 group
covering 2024-01-01 … 2024-01-30 — a coverage claim the customer did not have.** That is the whole
reason this chapter insists on explicit frames.

The same technique with a `>=` instead of `>` merges *touching* intervals (`2024-01-01..01-10` and
`2024-01-10..01-15` become one), which is often what you want for coverage and never what you want for
SCD-2 validity intervals.

### 4.7 Example 7 — "days since the last success" (a data-quality gate)

**Problem statement.** For each pipeline, how many days since its last successful run, and what was
the longest run of consecutive successful days?

```sql
WITH daily AS (
    SELECT pipeline, run_date, MAX(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS ok
    FROM   pipeline_runs WHERE run_date <= :as_of
    GROUP  BY pipeline, run_date
), streaks AS (
    SELECT pipeline, run_date, ok,
           run_date - ROW_NUMBER() OVER (PARTITION BY pipeline, ok ORDER BY run_date) AS grp
    FROM   daily
    WHERE  ok = 1
), success_streaks AS (
    SELECT pipeline, MIN(run_date) AS streak_start, MAX(run_date) AS streak_end,
           COUNT(*) AS streak_days
    FROM   streaks GROUP BY pipeline, grp
)
SELECT pipeline,
       MAX(streak_days)                   AS longest_success_streak,
       MAX(streak_end)                    AS last_success_date,
       (:as_of - MAX(streak_end))         AS days_since_last_success,
       CASE WHEN (:as_of - MAX(streak_end)) = 0
            THEN 'healthy' ELSE 'stale' END AS health
FROM   success_streaks
GROUP  BY pipeline
ORDER  BY days_since_last_success DESC;
```

Note the `PARTITION BY pipeline, ok` in the run key: partitioning by `ok` (after filtering to
successes, that is constant) is unnecessary here, but it is *required* if you compute the
success/failure runs together in one pass — otherwise a run of successes and a run of failures can
receive the same group key.

This query is the shape of a real freshness/streak monitor, and the two outputs it produces
(`days_since_last_success` and `longest_success_streak`) are different alerts:

| Signal | Meaning | Action |
|--------|---------|--------|
| `days_since_last_success > 1` | currently broken | page on-call now |
| `longest_success_streak` fell | intermittent flakiness | fix the root cause, not the incident |
| `days_since_last_success` spiking but recovering | a transient | investigate the pattern over a month |

## 5. Reusable template

### 5.1 Formulation A — universal gap-and-islands

```sql
WITH deduped AS (                       -- 1. COLLAPSE to the grain you mean
    SELECT <entity_id>, <order_key>, <value_cols>
    FROM   <source>
    WHERE  <filters>
    GROUP  BY <entity_id>, <order_key>, <value_cols>
), flagged AS (                         -- 2. FLAG the start of each run
    SELECT *,
           CASE
               WHEN LAG(<order_key>) OVER w IS NULL
                 OR <order_key> - LAG(<order_key>) OVER w > :step_plus_tolerance  -- gap
                 OR <value_col> IS DISTINCT FROM LAG(<value_col>) OVER w            -- state change
               THEN 1 ELSE 0
           END AS is_new_run
    FROM   deduped
    WINDOW w AS (PARTITION BY <entity_id> ORDER BY <order_key>, <tiebreaker>)
), numbered AS (                        -- 3. NUMBER the runs
    SELECT *,
           SUM(is_new_run) OVER (PARTITION BY <entity_id>
                                 ORDER BY <order_key>, <tiebreaker>
                                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_id
    FROM   flagged
)
SELECT <entity_id>, run_id,             -- 4. AGGREGATE per run
       MIN(<order_key>) AS run_start,
       MAX(<order_key>) AS run_end,
       COUNT(*)         AS run_length
FROM   numbered
GROUP  BY <entity_id>, run_id
ORDER  BY <entity_id>, run_start;
```

### 5.2 Formulation B — dense-sequence islands

```sql
WITH numbered AS (
    SELECT <entity_id>, <value_key>,
           <value_key> - ROW_NUMBER() OVER (PARTITION BY <entity_id>
                                            ORDER BY <value_key>) AS run_id
    FROM   <deduped_dense_values>
)
SELECT <entity_id>, run_id,
       MIN(<value_key>) AS run_start, MAX(<value_key>) AS run_end, COUNT(*) AS run_length
FROM   numbered GROUP BY <entity_id>, run_id;
```

Portable variant when arithmetic on the key is awkward — subtract a **calendar/sequence number**
(always an integer) from `ROW_NUMBER()`:

```sql
SELECT d.<entity_id>, MIN(d.<value_key>) AS run_start, MAX(d.<value_key>) AS run_end, COUNT(*) AS len
FROM (
    SELECT d.*, c.seq - ROW_NUMBER() OVER (PARTITION BY d.<entity_id> ORDER BY d.<value_key>) AS run_id
    FROM   <deduped_dense_values> d
    JOIN   <sequence_dimension>   c ON c.value = d.<value_key>     -- e.g. a business-day calendar
) d
GROUP BY d.<entity_id>, d.run_id;
```

### 5.3 Formulation C — run-length encoding

```sql
SELECT <value_col>, MIN(<order_key>) AS run_start, MAX(<order_key>) AS run_end, COUNT(*) AS run_length
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY <order_key>)
         - ROW_NUMBER() OVER (PARTITION BY <value_col> ORDER BY <order_key>) AS run_id
    FROM   <rows>
) x
GROUP BY <value_col>, run_id
ORDER BY run_start;
```

### 5.4 Gaps (the complement)

```sql
WITH diffs AS (
    SELECT <value_key>, LEAD(<value_key>) OVER (ORDER BY <value_key>) AS next_value
    FROM   <observed_values>
)
SELECT <value_key> AS gap_after, next_value AS gap_before,
       next_value - <value_key> - 1 AS missing_count
FROM   diffs
WHERE  next_value IS NOT NULL AND next_value > <value_key> + 1
ORDER  BY <value_key>;
```

…and the **exhaustive** version, which needs an expected domain:

```sql
WITH expected AS (SELECT <generate the full expected sequence> AS v)
SELECT e.v
FROM   expected e
LEFT   JOIN <observed_values> o ON o.<value_key> = e.v
WHERE  o.<value_key> IS NULL
ORDER  BY e.v;
```

### 5.5 Interval merge

```sql
SELECT run_id, MIN(s) AS merged_start, MAX(e) AS merged_end
FROM (
    SELECT *,
           SUM(CASE WHEN s <= MAX(e) OVER (ORDER BY s, e
                                           ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)
                    THEN 0 ELSE 1 END)
               OVER (ORDER BY s, e ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_id
    FROM   intervals
) x
GROUP BY run_id
ORDER BY merged_start;
```

### 5.6 Decision table

| The question sounds like | Formulation | Continuity clause |
|--------------------------|-------------|-------------------|
| "consecutive days with activity" | B | `d − ROW_NUMBER()` |
| "consecutive days, weekends allowed" | B + calendar | `biz_seq − ROW_NUMBER()` |
| "gap of at most N days counts as continuous" | A | `<order_key> − LAG(...) > 1 + N` |
| "how long did the status stay the same" | A or C | `value IS DISTINCT FROM LAG(value)` |
| "missing invoice numbers" | gaps | `LEAD(v) − v − 1` |
| "which invoice numbers are missing (exhaustive)" | expected-domain anti-join | `LEFT JOIN … IS NULL` |
| "merge overlapping contracts" | A + backwards frame | `s <= MAX(e) OVER (… AND 1 PRECEDING)` |
| "longest run of identical values" | A or C, then `MAX(len)` | — |
| "runs of at least N rows" | A or C, then `HAVING COUNT(*) >= N` | — |
| "how many distinct runs per entity" | A or C, then `COUNT(*)` | — |
| "the run that is live as of a date" | A or B, then filter `run_end = :as_of` | — |

### 5.7 Test kit

| Test | Assertion | Catches |
|------|-----------|---------|
| Runs partition the rows | `SUM(run_length) = COUNT(*)` over the deduped input | dropped rows, fan-out |
| Runs within an entity do not overlap | `COUNT(*) FILTER (WHERE run_start <= LAG(run_end)) = 0` per entity | ordering bug |
| Runs are maximal | consecutive runs of the same entity satisfy `next_start − prev_end > tolerance` | merging/splitting bug |
| Run ids are dense per entity | `MAX(run_id) = COUNT(DISTINCT run_id)` | gaps in the flag |
| Gap list is consistent with the run list | the number of gaps equals the number of runs minus 1, per entity | off-by-one |
| Merged intervals are disjoint | `MAX(end) OVER (… 1 PRECEDING) < start` at every row | frame bug (§4.6) |
| Streak length ≤ distinct days | `run_length <= COUNT(DISTINCT day)` per entity | missing `DISTINCT` |
| Deterministic re-run | two executions produce identical run ids | missing tiebreaker |

## 6. Real-world use cases and applications

| Domain | Question | Formulation | Notes |
|--------|---------|-------------|-------|
| **Product / growth** | longest daily streak, current streak | B | dedupe to `(user, day)` first |
| **Habits / gamification** | "did not break the chain" | B | needs a day-boundary rule and a timezone |
| **Subscriptions** | consecutive paid months | A | billing months, not calendar months |
| **SCD-2 dimensions** | validity intervals per attribute value | A (+ `LEAD`) | the backbone of every `dim_*_history` table |
| **IoT** | contiguous reporting periods | A | tolerance-based; the tolerance is the sampling interval |
| **Reliability** | uptime runs, outage windows | A | combine with a health predicate |
| **Finance** | missing cheque/invoice numbers | gaps | exhaustive version needs an expected range |
| **Inventory** | periods at a stock level (level-runs) | A / C | "how long was it out of stock" |
| **Data quality** | days since last successful load; success streaks | A | the freshness monitor in §4.7 |
| **Security** | contiguous periods a host used a port | A | run-length anomalies are the signal |
| **HR** | employment tenure periods, gaps in employment | A | same shape as SCD-2 |
| **Adtech** | consecutive days a campaign hit budget | A | pacing and delivery health |
| **Logistics** | vehicle dwell periods at a location | A | from geofence entry/exit events |
| **Energy / utilities** | meter-on/meter-off periods | A / C | RLE on a boolean |
| **Genomics / sequences** | runs of the same base | C | RLE is literally the goal |

Two production stories:

1. **"Consecutive-day streaks of 340 days."** A gamification feature computed streaks over the raw
   event table without `DISTINCT`. A user with 340 events on one day had a "340-day streak". The fix
   was one `DISTINCT` — and the leaderboard changed completely. **This is the single most common
   gap-and-islands bug in the wild**, and it is invisible until someone eyeballs the top of the
   leaderboard.
2. **"The customer was covered for the whole month."** A coverage query merged overlapping contracts
   using `MAX(e) OVER (ORDER BY s)` with the default frame. Because the frame includes the current
   row, "the farthest end so far" was really "the farthest end seen *including this contract*", which
   is always ≥ the current start, so every contract merged into one. The screenshot went to the
   customer. The fix was `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING` — five words.

## 7. Performance notes and anti-patterns

### 7.1 Cost model

Gap-and-islands is **one ordered pass**:

| Step | Cost | Note |
|------|------|------|
| `DISTINCT`/`GROUP BY` (dedupe) | `O(F log F)` or `O(F)` hash | often the dominant cost when the source is event-level |
| `LAG` + `SUM ... OVER (ORDER BY t)` | one sort `O(R log R)`, or `O(R)` if the input is already sorted | `R` = deduped rows, usually ≪ `F` |
| The final `GROUP BY run_id` | `O(R)` hash or `O(R log R)` sort | small |
| Formulation C | one extra sort (the second `ROW_NUMBER()` is a different window) | may be slower than A on some engines |

Optimisations, in order of impact:

1. **Deduplicate first.** `R` (rows per entity per period) is typically 2–3 orders of magnitude
   smaller than `F` (events). All the expensive work happens on `R`.
2. **Provide an index / clustering on `(entity_id, value_key)`.** With it, every step streams and the
   sort disappears. `CREATE INDEX ix ON t (entity_id, value_key) INCLUDE (value_col);`
3. **Do not compute islands you will discard.** For "longest streak", you need all runs; for "current
   streak", you can filter to runs touching `:as_of` — but you must still compute the runs to know
   which touch it, so this saves only the aggregation.
4. **Compare formulations A and C empirically.** On engines with an efficient inverse-transition
   running sum, A is faster; on engines where the second `ROW_NUMBER()` partitions a small cardinality
   column, C can be faster. Measure.
5. **Merge intervals is the expensive variant.** The backwards-`MAX` frame is a real frame buffer, so
   it is `O(R × w)` in the worst case for a naive implementation; PostgreSQL and Snowflake optimise
   monotonic aggregations, but on a million intervals expect to feel it. Sorting by `(s, e)` first is
   essential.

### 7.2 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| No `DISTINCT`/dedupe before measuring runs | "streaks" of hundreds of days; meaningless run lengths | collapse to `(entity, period)` first |
| `MAX(e) OVER (ORDER BY s)` with the default frame for interval merging | everything merges into one interval | `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING` |
| `GROUP BY value` instead of `GROUP BY value, run_id` | separate runs of the same value merged | include the run id in the grouping |
| No tiebreak in the `ORDER BY` | nondeterministic runs under equal order keys | add a unique column |
| `ORDER BY` inside `OVER` differs from the outer `ORDER BY` | results look shuffled; run ids appear non-monotonic | keep them consistent |
| Assuming `ROW_NUMBER()` over a sparse set gives dense runs | formulation B silently mis-groups when peers exist | ensure the value key is unique per entity |
| A window function in `WHERE` | syntax error, then a correlated subquery | filter in an outer query |
| Claiming the gap list is exhaustive | boundary gaps missed | use an expected domain (calendar/series) |
| Tolerance implemented as `>` vs `>=` without thought | off-by-one across the whole dataset | decide, document, and test |
| `CURRENT_DATE` in a "current streak" query | not reproducible; disagrees with other dashboards | pin `:as_of` |
| Timezone-naive "day" keys | streaks break at the wrong boundary for some users | derive a `local_date` column once |
| Computing islands per query instead of materialising them | slow, and definitional drift | materialise `mat_*_runs`, join to it |

### 7.3 Scale techniques

- **Materialise the run table.** A `mat_user_streaks`, `mat_subscription_periods`, or
  `dim_customer_history` table of `(entity, run_id, run_start, run_end, run_length, attributes)` is
  small (one row per run) and serves many queries. Recompute it on a schedule.
- **Incrementally extend only the open run.** Runs that ended before `:as_of − tolerance` are frozen.
  Recompute the tail and append. Reset the "current run" row each run.
- **Use a calendar/sequence dimension.** It turns the tolerance rule into a join and lets you keep
  formulation B (the cheapest). It also makes the rule auditable by a non-engineer, which is worth a
  lot in a regulated environment.
- **Bitmaps for wide entities.** For "runs of a boolean per day" over millions of entities, a
  `BITMAP`/`VARBINARY` column with one bit per day makes run detection arithmetic (find the
  transitions with XOR against a shifted copy) rather than a row-wise window. This is the technique
  behind high-scale streak and compliance products.
- **Push the dedupe into the model.** If every query starts with
  `SELECT DISTINCT entity_id, day FROM events`, put that in a `fct_entity_activity_day` model
  (chapter 05, §7.1) and stop paying for it.
- **Sort once, reuse everywhere.** Several gap-and-islands metrics (streaks, gaps, run counts) can be
  computed from the same deduped, sorted set in a single pipeline stage; only the flag expression
  changes.

## 8. Cross-engine differences

### 8.1 Primitives

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `LAG` / `LEAD` | ✅ 8.4+ | ✅ | ✅ 2012+ | ✅ | ✅ | ✅ |
| `date − date` → integer days | ✅ | `DATEDIFF` | `DATEDIFF` | ✅ (days) | `DATEDIFF` | `DATE_DIFF` |
| `date − integer` → date | ✅ | `DATE_SUB` | `DATEADD` | ✅ | `DATEADD` | `DATE_SUB` |
| Frame `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING` | ✅ | ✅ | ✅ 2012+ | ✅ | ✅ | ✅ |
| `IS DISTINCT FROM` | ✅ | `<=>` (spaceship; `IS NOT DISTINCT FROM` not available) | `INTERSECT`/`EXCEPT` trick; `IS NULL` handling | ❌ (use `DECODE`) | ✅ | ❌ (use `IS DISTINCT FROM` — supported in GoogleSQL) |
| `GROUPING SETS`/`CUBE` for run rollups | ✅ | ✅ 8.0+ | ✅ | ✅ | ✅ | ✅ |
| `MATCH_RECOGNIZE` (alternative for RLE) | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| `LISTAGG` for path/RLE strings | `STRING_AGG` | `GROUP_CONCAT` | `STRING_AGG` | `LISTAGG` | `LISTAGG` | `STRING_AGG` |
| `BIT_XOR`/`BIT_COUNT` for bitmap RLE | ✅ 14+ | ✅ | ❌ | `BIT_XOR_AGG` | `BIT_XOR` | `BIT_XOR` (in `BIT_XOR` agg) |
| `generate_series` for expected domains | ✅ | ❌ (recursive CTE) | ✅ 2022+ | `CONNECT BY` | `GENERATOR` | `GENERATE_DATE_ARRAY` |
| `EXCLUDE CURRENT ROW` (alternative to `1 PRECEDING`) | ✅ 11+ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 8.2 Dialect traps that change results

- **`date − integer` is portable; `date − date` is not always.** PostgreSQL and Oracle return an
  integer number of days from `d2 - d1`. MySQL, SQL Server, Snowflake and BigQuery need
  `DATEDIFF`/`DATE_DIFF` with an explicit unit. Formulation B is therefore cleanest when written as
  `DATEDIFF('day', <start>, <value>)` or by subtracting a **calendar sequence number** (an integer)
  rather than a date.
- **SQL Server's `DATEDIFF(day, …)` counts boundary crossings, not elapsed days.** For "days between
  two dates" this is what you want; for "elapsed time in days" it is not. Use `DATEDIFF_BIG` with a
  finer unit when the distinction matters.
- **`IS DISTINCT FROM` is not universal.** MySQL uses `<=>`; Oracle requires `DECODE(a, b, 1, 0)` or a
  `MINUS`-based check; BigQuery supports `IS DISTINCT FROM` in GoogleSQL. For a portable state-change
  flag, use `COALESCE`-based null-safe comparisons or `a IS NULL <> b IS NULL OR a <> b`.
- **`EXCLUDE CURRENT ROW` is PostgreSQL-only** (and DuckDB). Elsewhere, express it as
  `ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING`. Both mean "everything strictly before the
  current row", but `EXCLUDE` is clearer when the frame end is `CURRENT ROW` for other reasons.
- **Oracle and Snowflake `MATCH_RECOGNIZE`** can express run-length encoding declaratively
  (`PATTERN (A+ B+ A+)`), which for complex state machines is far more readable than chained flags.
  They are the only two engines (plus BigQuery and Databricks) where this is available.
- **Bitwise aggregates differ.** PostgreSQL 14+ has `bit_xor`; Oracle has `BIT_XOR_AGG`; Snowflake has
  `BIT_XOR`. For the bitmap-RLE technique, verify the exact function name — there is no standard.
- **`generate_series` for the expected domain:** PostgreSQL and SQL Server 2022+ have it; MySQL needs a
  recursive CTE; Oracle uses `CONNECT BY LEVEL`; Snowflake uses
  `TABLE(GENERATOR(ROWCOUNT => n))`; BigQuery uses `GENERATE_DATE_ARRAY`/`UNNEST(GENERATE_ARRAY(...))`.

### 8.3 Portability recipe

```sql
-- Portable formulation A: no date arithmetic, no IS DISTINCT FROM
CASE WHEN LAG(seq)   OVER w IS NULL
      OR seq - LAG(seq) OVER w > :step_plus_tolerance
      OR (state IS NULL) <> (LAG(state) OVER w IS NULL)
      OR state <> LAG(state) OVER w
     THEN 1 ELSE 0 END AS is_new_run

-- Portable formulation B: subtract a SEQUENCE INTEGER, never a date
seq - ROW_NUMBER() OVER (PARTITION BY entity ORDER BY seq) AS run_id

-- Portable interval merge
SUM(CASE WHEN s <= MAX(e) OVER (ORDER BY s, e
                                ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)
         THEN 0 ELSE 1 END)
    OVER (ORDER BY s, e ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_id
```

If you follow those three, the query runs unchanged on PostgreSQL, MySQL 8, SQL Server 2012+, Oracle,
Snowflake and BigQuery. The only engine-specific piece is generating the expected domain for the
exhaustive gap query.

## 9. Exercises — Intermediate (hints only)

1. **Consecutive-day streaks.** Produce the 23 runs from §4.1 and identify the longest. Which user,
   and how many days?
   *Hint:* user 7, four days (2024-03-04 … 2024-03-07). `DISTINCT user_id, event_time::date` first!
2. **Events per streak.** For each run, report the number of events, not just the number of days.
   Why can this exceed `run_length × something`?
   *Hint:* join the run table back to `events` on `user_id` and `d BETWEEN run_start AND run_end`.
3. **Longest streak and current streak.** Compute both, with an explicit as-of date. How many users
   have a live streak at 2024-03-11?
   *Hint:* `MAX(streak_days)` vs `streak_end = :as_of`; exactly one user.
4. **Missing invoice numbers.** Using the `invoices` CTE in §4.3, produce the gap list and then the
   exhaustive list of missing numbers.
   *Hint:* `LEAD(n) − n − 1` for ranges; `generate_series` + anti-join for the exhaustive list.
5. **Status validity intervals.** Produce the four intervals from §4.4 and explain why `active`
   appears twice.
   *Hint:* `GROUP BY grp, status` — the `grp` is what keeps the two `active` periods apart.
6. **Run-length encode a boolean.** For a `daily.healthy` flag over 30 generated days, produce the
   runs. Which formulation is shortest?
   *Hint:* formulation C (two `ROW_NUMBER()`s) needs no running sum.
7. **Merge overlapping intervals.** Reproduce §4.6 and then produce the *buggy* version, showing that
   it collapses everything into one interval.
   *Hint:* change `AND 1 PRECEDING` to `AND CURRENT ROW` and compare the group counts.
8. **Tolerance.** Re-run exercise 1 with a tolerance of 1 day (a gap of one missing day still counts).
   Which runs merge, and what is the new longest streak?
   *Hint:* user 7's two runs (4 and 3 days) merge into one 8-day run, since only 2024-03-08 is missing.
9. **Gap-and-islands as a data-quality test.** Write the assertion that fails if any run in your
   streak table is not maximal (two adjacent runs with no missing day between them).
   *Hint:* `LAG(run_end)` per entity and compare with `run_start`; expect `run_start − prev_run_end > tolerance`.
10. **The live streak.** Write the query that gives "the current streak" for every user as of a
    parameterised as-of date, returning 0 for users with no live streak.
    *Hint:* `LEFT JOIN` the run table to `users` and `COALESCE(streak_days, 0)`.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Gap-and-islands on a *streaming* system.** Implement consecutive-day streaks per user in a stream
   processor, with a mutable "current streak" per user and correct behaviour under late events that
   can extend or *retroactively insert* a day into an existing streak.
   *Expected approach:* maintain per-user state `(last_day, current_streak_len, longest_streak_len)`;
   on an event for `day d`: if `d = last_day + 1`, increment; if `d = last_day`, ignore (dedupe); if
   `d > last_day + 1`, reset to 1 and update the max; if `d < last_day`, the event is *late* and may
   require re-deriving the run — which means either a bounded out-of-order window (recompute the last
   `k` days) or an event-sourced design where the run is derived in batch and the stream only
   maintains a provisional value. Discuss the correctness/latency trade-off and state the reconciliation
   test (batch vs stream streak equality within the lateness bound).
2. **Merging intervals with a cap and a priority.** Given overlapping contracts with a `max_coverage`
   attribute, produce non-overlapping merged periods where the coverage limit applies only while at
   least one contract is active, and a *priority* rule decides which coverage applies when contracts
   overlap.
   *Expected approach:* this is a sweep-line / "segmented max" problem rather than a simple merge.
   Build the event points (`+coverage` at `s`, `−coverage` at `e`), compute a running sum of coverage
   and a running max of priority with a **backwards frame**, and emit segments where the running set
   is non-empty. Note that the running `MAX` must *release* a contract's coverage when it ends, which
   a plain `MAX` cannot do — you need either a monotone-deque-style solution in SQL (hard) or a
   segmented approach over the event points (recommended). Discuss why the naive interval-merge
   template breaks and what the correct algorithm class is.
3. **Exhaustive gap detection without a full calendar table.** You must find all missing days for
   10^5 entities over 5 years (about `1.8 × 10^8` expected `(entity, day)` pairs) without materialising
   that grid. Give a query and its complexity, and state the accuracy/coverage trade-off.
   *Expected approach:* you do not need to materialise the grid — the *gaps* are exactly the
   adjacent-value differences greater than 1 in the observed set (formulation B's complement), and the
   only values you cannot discover are the boundary gaps, which need one anchor per entity (the
   entity's expected start). So: compute the internal gaps from `LEAD`, and the boundary gaps from a
   per-entity expected `first_day`/`last_day` in a small dimension table. Prove the coverage claim
   (internal + boundary = exhaustive) and give the complexity, which is now `O(observed)` rather than
   `O(expected)`. Discuss when you genuinely need the grid (when the domain is not a dense calendar,
   e.g. valid SKU codes).
4. **Run-based retention.** Define "a user is *on a habit* for a given week if their current daily
   streak is at least 5 days" and compute weekly habit prevalence per cohort. Then show how the
   number changes if the streak threshold is 3 or 7.
   *Expected approach:* compute daily streaks (§4.1) → for each (user, day) compute the running streak
   length at that day (a window over the streak table, `day − streak_start + 1`) → join to a
   week spine and flag days where the streak ≥ threshold → aggregate per week and cohort (chapter 04).
   Report the sensitivity table (3/5/7 days) and argue that the threshold is a *product definition*,
   not a parameter, because it changes the cohort comparison ranking. This is the metric-governance
   theme again.
5. **SCD-2 with attribute-level (Type 4 / "mini-dimension") history.** Instead of one validity
   interval per row version, produce an interval per *attribute* per value, so you can answer "when
   was this customer's `plan` = 'pro' and their `country` = 'DE'" with a single join.
   *Expected approach:* unpivot the attributes to `(entity, attribute, value, effective_from)`, apply
   the RLE formulation per `(entity, attribute)` to collapse repeated values, produce validity
   intervals per attribute, and then show how an as-of join against *two* attribute intervals answers
   the query. Discuss the cost (one row per attribute-value change, which can be much larger than one
   row per SCD-2 version) and when it is worth it versus a classic Type-2 dimension.
6. **Prove the interval-merge frame bug with a property test.** Write a test that generates random
   interval sets, merges them with both the correct and the default frame, and asserts that (a) the
   correct version's merged intervals are pairwise disjoint and (b) the default-frame version's output
   is a superset of the union. Then explain, in one paragraph, the general principle the bug
   illustrates about window frames.
   *Expected approach:* generate intervals with `generate_series` and `random()`, run both variants,
   and compare `COUNT(DISTINCT grp)`. Property (a) is `MAX(end) OVER (… 1 PRECEDING) < start` at every
   row; property (b) is `merged_correct ⊆ merged_default` as a set of ranges. The principle: a frame is
   a *definition of the set of rows the aggregate sees*, so including the current row changes the
   meaning of "so far" — the same class of error as `RANGE` vs `ROWS` (chapter 02) and `LAST_VALUE`
   without an explicit frame (chapter 01). Those three bugs are the same bug wearing three hats, and
   recognising that is the senior-level takeaway of this section.

## References

- PostgreSQL — Window function calls (frames, `EXCLUDE CURRENT ROW`, `GROUPS`): <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- PostgreSQL — Window functions (`LAG`, `LEAD`, `ROW_NUMBER`): <https://www.postgresql.org/docs/current/functions-window.html>
- PostgreSQL — `SELECT` (`DISTINCT ON`, `FETCH FIRST … WITH TIES`): <https://www.postgresql.org/docs/current/sql-select.html>
- PostgreSQL — `generate_series` (expected domains for the exhaustive gap query): <https://www.postgresql.org/docs/current/functions-srf.html>
- MySQL 8.0 — Window functions (`LAG`/`LEAD`, frames): <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- MySQL 8.0 — `TIMESTAMPDIFF` and date arithmetic: <https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html>
- Microsoft Learn — `LAG` / `LEAD` (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/functions/lag-transact-sql>
- Microsoft Learn — `DATEDIFF` / `DATEDIFF_BIG` (boundary-crossing semantics): <https://learn.microsoft.com/en-us/sql/t-sql/functions/datediff-transact-sql>
- Oracle — Analytic functions and `MATCH_RECOGNIZE` (row pattern matching for RLE): <https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/sql-pattern-matching-data-warehouses.html>
- Oracle — `BIT_XOR_AGG` (bitmap run-length techniques): <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/BIT_XOR_AGG.html>
- Snowflake — `MATCH_RECOGNIZE`: <https://docs.snowflake.com/en/sql-reference/constructs/match_recognize>
- Snowflake — Window function syntax and usage: <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — `DATEDIFF`: <https://docs.snowflake.com/en/sql-reference/functions/datediff>
- Google Cloud — BigQuery `DATE_DIFF` / `GENERATE_DATE_ARRAY`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/date_functions>
- Google Cloud — BigQuery `MATCH_RECOGNIZE`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#match_recognize_clause>
- DuckDB — Window functions (`EXCLUDE`, `GROUPS`, local testing): <https://duckdb.org/docs/stable/sql/functions/window_functions.html>
- SQLPad — How to solve gaps-and-islands interview questions with `LAG` and `ROW_NUMBER`: <https://sqlpad.io/tutorial/how-to-solve-gaps-and-islands-sql-interview-questions-with-lag-and-row-number/>
- sqlism — Identify gaps and islands in SQL (the classic technique): <https://sqlism.com/gaps-and-islands-in-sql>
- pipecode — SQL gaps & islands: sessionization, streaks and run-length patterns: <https://pipecode.ai/blogs/sql-gaps-and-islands-sessionization-streaks>
- dbhelm — Top-N per group, `DISTINCT ON`, and gaps and islands: <https://dbhelm.com/learn/postgresql/lessons/top-n-per-group>
- Mode / ThoughtSpot — SQL Window Functions tutorial (`LAG`/`LEAD` used throughout): <https://mode.com/sql-tutorial/sql-window-functions/>
- dbt Labs — Incremental models (freezing closed runs, extending the open one): <https://docs.getdbt.com/docs/build/incremental-models>

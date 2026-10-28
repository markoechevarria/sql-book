# Sessionization

> Before you can say anything about a user's *visit*, you have to decide where one visit ends and the
> next begins — and that decision is not in the data. Web and mobile telemetry arrives as an
> undifferentiated stream of timestamped events; the concept of a "session" is a **modelling choice**
> (an inactivity timeout, usually 30 minutes) that you impose on it. Sessionization is the canonical
> `gap-and-islands` problem, it is the single highest-leverage preprocessing step in product
> analytics, and it is the one where the definitional choices (timeout, entity, cap, day boundary)
> move every downstream metric — bounce rate, session duration, pages per session, funnel conversion —
> by 10–40 %.

## TL;DR

- A session is **a run of consecutive events with gaps below a threshold**. Formally: partition by
  entity, order by time, start a new session whenever `event_time − LAG(event_time) > timeout`.
- **The three-line pattern** (memorize it, it is the whole chapter):

  ```sql
  CASE WHEN event_time - LAG(event_time) OVER w > INTERVAL '30 minutes'
            OR LAG(event_time) OVER w IS NULL
       THEN 1 ELSE 0 END                                AS is_new_session
  SUM(is_new_session) OVER w_rows                       AS session_seq
  ```

  Then `session_id = entity_id || '-' || session_seq`. That is a sessionization.
- **Batch vs streaming is the real engineering split.** Batch: one pass over history, deterministic,
  trivially replayable. Streaming: sessions are *open* (they can still receive events), so you must
  buffer them until a **watermark** passes `last_event + timeout`. Out-of-order events can force two
  sessions to **merge**, which is why a streaming sessionizer is a stateful, bounded-retention
  operator and not a `GROUP BY`.
- **Choose the timeout at the middle of a plateau.** Compute sessions at 1, 5, 30, 60, 240 minutes
  and plot the session count. There is almost always a flat region; pick its centre. On the
  ShopStream sample the plateau runs from 5 to 240 minutes (35–37 sessions), which is why the industry
  default of 30 minutes generalises so well. A 1-minute timeout produces 59 sessions and a 95 %
  bounce rate — pure noise.
- **Order by `(event_time, unique_tiebreaker)`.** Events with identical timestamps are extremely
  common (batch uploads, SDK flushes). Without a tiebreaker, session boundaries are nondeterministic
  and your backfills do not reproduce (chapter 01, §3.4).
- **Sessions need more than a timeout in the real world**: a maximum duration cap (a 2-hour cap stops
  a "session" from lasting all day), a day boundary (so daily reports align), and a "hard break on
  context change" (new campaign, new device, new app version). All four are expressible as
  additional `is_new_session` conditions.
- **Materialise `session_id` once.** Sessionization inside every query is slow, and — worse —
  sessionization implemented slightly differently in two queries produces two sets of sessions and
  therefore two sets of every downstream number.
- **Sessions are what make DAU/WAU/MAU, bounce rate, funnels, and session-duration metrics
  computable.** They are also what makes the *definition* of those metrics contentious. Write the
  timeout into the metric catalogue.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Event stream** | Ordered `(entity_id, event_time, event_type, …)` records. |
| **Session** | A maximal run of events for one entity where consecutive gaps are below the timeout. |
| **Inactivity timeout** | The gap threshold (commonly 30 min) above which a new session begins. |
| **Session boundary** | The event that starts a new session. |
| **Session id** | A stable identifier for the run; usually `entity_id || '-' || running_count`. |
| **Session duration** | `MAX(event_time) − MIN(event_time)` within the session. Zero for single-event sessions. |
| **Bounce session** | A session with exactly one event (or, in web analytics, one page view). |
| **Engaged session** | A session above the bounce threshold (≥ 2 events, or ≥ 10 s, depending on definition). |
| **Session timeout cap** | A maximum session length that force-closes a session even without a gap. |
| **Day boundary break** | Starting a new session at local midnight (or at a reporting-day boundary). |
| **Gap-and-islands** | The general technique of grouping consecutive rows into runs; sessionization is one instance (chapter 08). |
| **Watermark** | In streaming, a monotone time bound meaning "no events older than this will arrive". |
| **Open session** | A streaming session that can still receive events and therefore cannot be emitted yet. |
| **Session merge** | When a late event bridges two previously-separate sessions, they must be combined. |
| **Sessionization key** | The entity plus any context that must not be crossed (device, app version, campaign). |
| **Late/out-of-order event** | An event whose `event_time` is earlier than events already processed. |

### 1.2 The picture

```
events for user 1, sorted by time

10:00 ─┐
10:02  │  gap 2m
10:04  │  gap 2m        ── one session (gap < 30m everywhere)
10:06  │  gap 2m
10:09 ─┘  gap 3m
                 gap 4h55m  ← > 30m  → SESSION BOUNDARY
14:00 ─┐
14:03  │  gap 3m
14:05  │  gap 2m        ── second session
14:07 ─┘  gap 2m
                 gap 19h    ← > 30m  → SESSION BOUNDARY
(next day 09:00) ──── third session
```

Sessionization is therefore **thresholding a lag**, then **numbering the resulting runs**. Two
operations, both window functions.

### 1.3 The four parameters that define a session

| Parameter | Typical value | What it changes |
|-----------|---------------|-----------------|
| **Entity** | `user_id`, `device_id`, `account_id`, `cookie_id` | who may share a session; changing it changes everything |
| **Timeout** | 30 min (web), 30–60 min (mobile), 5 min (API keys) | session count, bounce rate, duration |
| **Cap** | 2–4 h | truncates marathon sessions; affects "time on site" |
| **Break conditions** | day boundary, campaign change, app version, device | splits sessions that a pure gap rule would join |

Industry references for the defaults: Google Analytics 4 uses a **30-minute** default session
timeout and **also** starts a new session at local midnight and on a campaign change; a GA session is
capped at 24 h. Adobe uses 30 minutes. Most mobile SDKs default to 30 minutes of background time.
**Every one of those is a default, not a law** — a banking app (weekly usage) and a messaging app
(constant usage) need different timeouts, and both need the choice documented.

### 1.4 Session metrics you can only compute after sessionizing

| Metric | Definition | Why sessions are required |
|--------|-----------|---------------------------|
| **Sessions** | count of runs | the unit of a "visit" |
| **Bounce rate** | single-event sessions ÷ sessions | requires the run boundary |
| **Pages / screens per session** | events ÷ sessions | same |
| **Session duration** | per-session max − min | same |
| **Session-scoped funnel conversion** | steps within one session | the boundary defines the funnel (chapter 06) |
| **Engaged time** | time between first and last *engaged* event | needs the run |
| **Sessions per user** | sessions ÷ distinct users | the "visit frequency" metric |
| **Time between sessions** | `session_start − LAG(session_end)` | the re-engagement cadence |

## 2. Why it matters / what problem it solves

Sessionization is upstream of almost everything in product analytics, so **its errors propagate
everywhere**:

| Downstream metric | How it breaks without correct sessionization |
|-------------------|---------------------------------------------|
| Bounce rate | over-counts if the timeout is too short; under-counts if too long |
| Session duration | becomes "time on site" (which can be 20 hours) without a cap |
| Pages per session | inflated by merged sessions across days |
| Funnel conversion | over-counted (a user who abandons and returns later may appear to convert) |
| DAU/WAU/MAU | unaffected in count, but *session*-based engagement is |
| Retention | "active day" is fine without sessions; "engaged session" is not |
| Ad/attribution analysis | a campaign change should start a new session |
| Capacity planning | concurrent-session counts come from session boundaries |
| Cost (streaming state) | open-session state is the dominant memory cost of a sessionizer |

Three concrete failure modes:

1. **The 24-hour "session".** A mobile app without a cap produced sessions lasting up to 26 hours,
   because the SDK kept the session alive across background/foreground transitions. "Average session
   duration" was 47 minutes; with a 30-minute timeout plus a 4-hour cap it was 3.2 minutes. Both
   numbers were labelled "session duration".
2. **The nondeterministic backfill.** A sessionizer ordered only by `event_time`, and a batch upload
   gave dozens of events the same second. Two runs of the same pipeline produced different session
   counts (and therefore different bounce rates) because the tie order differed. Adding
   `ORDER BY event_time, event_id` fixed it — the entire fix was one sort key.
3. **The streaming session that never closed.** A streaming sessionizer emitted sessions at
   `last_event + 30 min`, but with no allowed lateness it dropped late events that belonged to an
   already-emitted session, so mobile users whose events arrived from a queued upload were
   under-counted. The fix was a watermark with allowed lateness plus a bounded state store.

## 3. The pattern / technique explained

### 3.1 The canonical batch sessionization

```sql
WITH ordered AS (
    SELECT user_id,
           event_id,
           event_type,
           event_time,
           LAG(event_time) OVER (PARTITION BY user_id
                                 ORDER BY event_time, event_id) AS prev_event_time
    FROM   events
), flagged AS (
    SELECT *,
           CASE WHEN prev_event_time IS NULL
                     OR event_time - prev_event_time > INTERVAL '30 minutes'
                THEN 1 ELSE 0 END AS is_new_session
    FROM   ordered
), numbered AS (
    SELECT *,
           SUM(is_new_session) OVER (PARTITION BY user_id
                                     ORDER BY event_time, event_id
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
    FROM   flagged
)
SELECT user_id || '-' || session_seq AS session_id,
       user_id, event_id, event_type, event_time, is_new_session, session_seq
FROM   numbered
ORDER  BY user_id, event_time, event_id;
```

Three lines of substance; five CTEs of clarity. The pieces:

| Piece | Why it is there |
|-------|-----------------|
| `PARTITION BY user_id` | sessions never cross entities |
| `ORDER BY event_time, event_id` | deterministic ordering (the `event_id` is the tiebreaker) |
| `LAG(event_time)` | the previous event *in this entity's ordered stream* |
| `IS NULL` in the condition | the first event of each entity is always a boundary |
| `> timeout` | strictly greater; an exact-30-minute gap is the *same* session |
| `SUM(flag) OVER (… ROWS UNBOUNDED PRECEDING)` | converts flags into run numbers (gap-and-islands) |
| `ROWS UNBOUNDED PRECEDING` | explicit frame so the cumulative sum cannot drift (chapter 02) |

The `SUM` of flags is the heart of it: `is_new_session` is `1` exactly at the start of a run, so its
running total is constant inside a run and increments at each boundary. That is the general
**gap-and-islands grouping key** (chapter 08).

### 3.2 `session_id` formats

| Format | Example | Pros | Cons |
|--------|---------|------|------|
| **Composite text** | `'42-3'` | trivially reproducible | collation-sortable, ugly, wide |
| **Hash** | `md5(user_id || ':' || session_seq)`, `to_hex(…)` | fixed width, joinable | not human-readable, not orderable |
| **`user_id` + `session_seq` tuple** | `(user_id, session_seq)` | keeps key parts | composite keys everywhere |
| **Surrogate from a dimension table** | `dim_session.session_key` | compact, carries metadata (start, end, device) | an extra model and a join |
| **`ROW_NUMBER() OVER ()` on the deduped session grain** | `mat_session.session_id` | stable if the sort is stable | must not be recomputed |

**Recommendation:** in the lake/warehouse, publish a `dim_sessions`/`mat_sessions` table keyed by
`(user_id, session_seq)` (or a hash of it) containing `session_start`, `session_end`, `event_count`,
`duration_seconds`, `is_bounce`, and the entity's context. Persist it. Every downstream query then
joins to sessions instead of recomputing them.

### 3.3 Ordering, ties, and determinism

The `ORDER BY` inside `OVER` must be a **total order**, or the answer is not reproducible. Timestamps
from a batch upload are frequently identical to the second, and some sources give
second-or-worse resolution while events arrive at millisecond granularity.

Options for the tiebreaker, best first:

1. The source's monotonic sequence (`event_id`, Kafka `offset`, an `ingest_seq`) — the only choice
   that reflects true arrival order.
2. A per-event unique id (UUID) — deterministic but arbitrary.
3. The event type — deterministic and occasionally *semantically* right (e.g. `page_view` before
   `page_load`).

Never leave it out. The cost of an ambiguous sort is that two runs of the same pipeline produce
different session counts, and there is no way to tell which run was right.

### 3.4 Adding a maximum duration cap

The rule "a session also ends if it has already lasted more than 2 hours" cannot be expressed with
`LAG` alone, because the cap depends on *where the current session started*, which is itself the
output of the run-numbering step. Do it in two passes:

```sql
WITH /* … flagged, numbered as in §3.1 … */ numbered AS (…),
with_session_start AS (
    SELECT *,
           MIN(event_time) OVER (PARTITION BY user_id, session_seq) AS session_start
    FROM   numbered
), recapped AS (
    SELECT *,
           CASE WHEN event_time - session_start > INTERVAL '2 hours' THEN 1 ELSE 0 END AS cap_break
    FROM   with_session_start
), renumbered AS (
    SELECT *,
           SUM(cap_break) OVER (PARTITION BY user_id
                                ORDER BY event_time, event_id
                                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq_v2
    FROM   recapped
)
SELECT user_id || '-' || session_seq || '_' || session_seq_v2 AS session_id, *
FROM   renumbered;
```

Two passes of the same run-numbering primitive. This generalises: every additional break condition
is another flag added to the same running sum, in the right order.

> **Senior note.** Note the subtlety: after capping, the *new* session's start moves forward, which can
> theoretically make an earlier event no longer exceed the cap. For a single extra pass and realistic
> timeouts this does not matter; for pathological data it does. The rigorous fix is a recursive walk
> (`WHERE NOT EXISTS a prior un-capped break`), which is what a streaming engine's session operator
> does natively. If your cap is 2 h and your timeout is 30 min, one pass is exact; if you are capping
> at 35 minutes with a 30-minute timeout, be careful.

### 3.5 Adding a day boundary

For daily reports, a session must not span midnight:

```sql
CASE WHEN prev_event_time IS NULL
          OR event_time - prev_event_time > INTERVAL '30 minutes'
          OR date_trunc('day', event_time) <> date_trunc('day', prev_event_time)
     THEN 1 ELSE 0 END AS is_new_session
```

Use the *local* day, not UTC, if the business reports locally (a session at 23:50 UTC is 16:50 in
Los Angeles and belongs to that day's report). Store a `local_date` column rather than recomputing
timezone offsets in every query.

### 3.6 Streaming sessionization

In streaming (Flink `EventTimeSessionWindows`, Kafka Streams `SessionWindows`, Spark Structured
Streaming `session_window`), the mechanics are:

```
1. KeyBy(entity)                                    ← the partition key is the sessionization key
2. Session window with a gap of :timeout            ← the window closes :timeout after the last event
3. Watermark with allowed lateness                  ← "no events older than (max_seen − lateness)"
4. For each key, keep OPEN sessions in state         ← bounded by timeout + lateness + cap
5. On watermark > last_event + timeout → EMIT + clear state
```

Three properties you must design for:

| Property | Consequence if ignored |
|----------|------------------------|
| **Session merging** | A late event that bridges two sessions must merge them; if you already emitted, you cannot. Buffer until the watermark passes. |
| **Allowed lateness** | Without it, late mobile events are dropped and you under-count. With a large lateness, state grows. |
| **State retention per key** | `keys × open_sessions × events`; an unbounded key space (cookie ids) blows up memory. Use a bounded state store with TTL. |

The **exact-match guarantee** you should aim for: a streaming sessionizer and a batch sessionizer
over the same data, with a watermark ≥ the maximum lateness, produce **identical** `session_id`s. If
they do not, the difference is either the tiebreaker (streaming orders by arrival, batch by
`event_time`) or the lateness window. Test this explicitly with a reconciliation job — it is the only
way to keep a real-time and a batch funnel dashboard from disagreeing.

### 3.7 Recompute vs. incremental

Sessions are **append-only within a session** and **frozen once the timeout has elapsed**:

```
open window = (now − timeout − lateness, now]

→ incremental strategy:
   • fully recompute sessions whose session_start >= (now − timeout − lateness)
   • never touch older sessions
```

That is a clean, bounded incremental model — one of the few in analytics where "recompute a short
tail, freeze the rest" is both correct and cheap. The failure to avoid: recomputing only *today's*
events, which misses sessions that started yesterday and continued past midnight.

## 4. Step-by-step worked example (with sample data and expected output tables)

Using the shared schema from `01-ranking.md` (`events`, 63 rows across 7 users).

### 4.1 Example 1 — the full sessionization

```sql
WITH ordered AS (
    SELECT user_id, event_id, event_type, event_time,
           LAG(event_time) OVER (PARTITION BY user_id
                                 ORDER BY event_time, event_id) AS prev_event_time
    FROM   events
), flagged AS (
    SELECT *,
           CASE WHEN prev_event_time IS NULL
                     OR event_time - prev_event_time > INTERVAL '30 minutes'
                THEN 1 ELSE 0 END AS is_new_session
    FROM   ordered
), numbered AS (
    SELECT *,
           SUM(is_new_session) OVER (PARTITION BY user_id
                                     ORDER BY event_time, event_id
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
    FROM   flagged
)
SELECT user_id || '-' || session_seq AS session_id,
       user_id, event_type, event_time, is_new_session, session_seq
FROM   numbered
ORDER  BY user_id, event_time, event_id;
```

Excerpt (user 1), showing the mechanism explicitly:

| session_id | user_id | event_type | event_time | prev_event_time | gap | is_new_session | session_seq |
|------------|--------:|-----------|------------|-----------------|-----|:--------------:|------------:|
| 1-1 | 1 | page_view | 2024-01-05 10:00:00 | `NULL` | — | **1** | 1 |
| 1-1 | 1 | view_product | 2024-01-05 10:02:00 | 10:00:00 | 2 min | 0 | 1 |
| 1-1 | 1 | add_to_cart | 2024-01-05 10:04:00 | 10:02:00 | 2 min | 0 | 1 |
| 1-1 | 1 | checkout_start | 2024-01-05 10:06:00 | 10:04:00 | 2 min | 0 | 1 |
| 1-1 | 1 | purchase | 2024-01-05 10:09:00 | 10:06:00 | 3 min | 0 | 1 |
| 1-2 | 1 | page_view | 2024-01-06 09:00:00 | 10:09:00 | **22 h 51 min** | **1** | 2 |
| 1-2 | 1 | view_product | 2024-01-06 09:05:00 | 09:00:00 | 5 min | 0 | 2 |
| 1-3 | 1 | page_view | 2024-01-06 14:00:00 | 09:05:00 | **4 h 55 min** | **1** | 3 |
| 1-3 | 1 | view_product | 2024-01-06 14:03:00 | 14:00:00 | 3 min | 0 | 3 |
| 1-3 | 1 | add_to_cart | 2024-01-06 14:05:00 | 14:03:00 | 2 min | 0 | 3 |
| 1-3 | 1 | purchase | 2024-01-06 14:07:00 | 14:05:00 | 2 min | 0 | 3 |
| 1-4 | 1 | page_view | 2024-01-12 11:00:00 | 14:07:00 | **5 d 20 h** | **1** | 4 |
| 1-4 | 1 | view_product | 2024-01-12 11:04:00 | 11:00:00 | 4 min | 0 | 4 |
| 1-4 | 1 | add_to_cart | 2024-01-12 11:06:00 | 11:04:00 | 2 min | 0 | 4 |
| 1-4 | 1 | checkout_start | 2024-01-12 11:08:00 | 11:06:00 | 2 min | 0 | 4 |
| 1-5 | 1 | page_view | 2024-01-19 08:30:00 | 11:08:00 | **6 d 21 h** | **1** | 5 |
| 1-5 | 1 | view_product | 2024-01-19 08:33:00 | 08:30:00 | 3 min | 0 | 5 |
| 1-5 | 1 | add_to_cart | 2024-01-19 08:35:00 | 08:33:00 | 2 min | 0 | 5 |
| 1-5 | 1 | checkout_start | 2024-01-19 08:36:00 | 08:35:00 | 1 min | 0 | 5 |
| 1-5 | 1 | purchase | 2024-01-19 08:40:00 | 08:36:00 | 4 min | 0 | 5 |
| 1-6 | 1 | page_view | 2024-01-26 20:15:00 | 08:40:00 | **7 d 11 h** | **1** | 6 |
| 1-7 | 1 | page_view | 2024-02-02 12:00:00 | 20:15:00 | **7 d 15 h** | **1** | 7 |

`is_new_session` is 1 exactly at each boundary, and `session_seq` is constant inside each run — that
is the whole technique. Note session **1-3**: it starts at 14:00 on 2024-01-06, 4 h 55 min after the
previous session ended at 09:05. **If the timeout had been 6 hours, 1-2 and 1-3 would have merged**,
and user 1 would have 6 sessions instead of 7 — with a completely different session-duration and
bounce picture. That is the sensitivity to watch.

### 4.2 Example 2 — the session table

```sql
WITH /* … numbered as above … */ numbered
SELECT user_id || '-' || session_seq                       AS session_id,
       user_id,
       session_seq,
       MIN(event_time)                                     AS session_start,
       MAX(event_time)                                     AS session_end,
       COUNT(*)                                            AS event_count,
       (EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))))::int AS duration_seconds,
       COUNT(*) = 1                                        AS is_bounce,
       COUNT(DISTINCT event_type)                          AS distinct_event_types,
       BOOL_OR(event_type = 'purchase')                    AS has_purchase
FROM   numbered
GROUP  BY user_id, session_seq
ORDER  BY user_id, session_seq;
```

| session_id | user_id | seq | session_start | session_end | events | duration_s | is_bounce | distinct_types | has_purchase |
|------------|--------:|----:|---------------|-------------|-------:|-----------:|:---------:|---------------:|:------------:|
| 1-1 | 1 | 1 | 2024-01-05 10:00:00 | 2024-01-05 10:09:00 | 5 | 540 | false | 5 | true |
| 1-2 | 1 | 2 | 2024-01-06 09:00:00 | 2024-01-06 09:05:00 | 2 | 300 | false | 2 | false |
| 1-3 | 1 | 3 | 2024-01-06 14:00:00 | 2024-01-06 14:07:00 | 4 | 420 | false | 4 | true |
| 1-4 | 1 | 4 | 2024-01-12 11:00:00 | 2024-01-12 11:08:00 | 4 | 480 | false | 4 | false |
| 1-5 | 1 | 5 | 2024-01-19 08:30:00 | 2024-01-19 08:40:00 | 5 | 600 | false | 5 | true |
| 1-6 | 1 | 6 | 2024-01-26 20:15:00 | 2024-01-26 20:15:00 | 1 | 0 | **true** | 1 | false |
| 1-7 | 1 | 7 | 2024-02-02 12:00:00 | 2024-02-02 12:00:00 | 1 | 0 | **true** | 1 | false |
| 2-1 | 2 | 1 | 2024-01-07 09:00:00 | 2024-01-07 09:06:00 | 5 | 360 | false | 5 | true |
| 2-2 | 2 | 2 | 2024-01-08 15:00:00 | 2024-01-08 15:00:00 | 1 | 0 | true | 1 | false |
| 2-3 | 2 | 3 | 2024-01-14 10:00:00 | 2024-01-14 10:02:00 | 2 | 120 | false | 2 | false |
| 2-4 | 2 | 4 | 2024-01-21 10:00:00 | 2024-01-21 10:05:00 | 4 | 300 | false | 4 | false |
| 2-5 | 2 | 5 | 2024-02-04 18:00:00 | 2024-02-04 18:00:00 | 1 | 0 | true | 1 | false |

Totals you can assert against:

```sql
SELECT COUNT(*) AS sessions, SUM(event_count) AS events,
       AVG(event_count) AS events_per_session,
       COUNT(*) FILTER (WHERE is_bounce) AS bounces,
       ROUND(100.0 * COUNT(*) FILTER (WHERE is_bounce) / COUNT(*), 1) AS bounce_rate_pct,
       SUM(duration_seconds) AS total_duration_s,
       ROUND(AVG(duration_seconds), 1) AS mean_duration_s,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY duration_seconds) AS median_duration_s,
       PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY duration_seconds) AS p90_duration_s
FROM   sessions;
```

| metric | value | comment |
|--------|------:|---------|
| sessions | 35 | 63 events ÷ 35 = 1.80 events/session |
| events | 63 | matches `SELECT COUNT(*) FROM events` |
| bounces | 23 | |
| bounce rate | 65.7 % | user 7 alone contributes 7 bounces |
| total duration | 4,740 s | |
| mean duration | 135.4 s | dominated by the 12 non-bounce sessions |
| **median duration** | **0 s** | **the median session is a single-event bounce** |
| p90 duration | 540 s | |

**The median session duration is zero.** That is not a bug — it is what a 65.7 % bounce rate means,
and it is the single most important reason never to publish a mean session duration. A
"mean of 135 seconds" sounds like a product with engaged users; "the median visit is one event and
lasts zero seconds" describes the same data and tells the truth.

### 4.3 Example 3 — timeout sensitivity (choose the plateau, not the default)

```sql
-- One query, five timeouts, using a VALUES list of candidate timeouts.
WITH timeouts(t_label, t_sec) AS (
    VALUES ('1 min', 60), ('5 min', 300), ('30 min', 1800), ('60 min', 3600), ('240 min', 14400)
),
ordered AS (
    SELECT user_id, event_id, event_time,
           LAG(event_time) OVER (PARTITION BY user_id
                                 ORDER BY event_time, event_id) AS prev_event_time
    FROM   events
), flagged AS (
    SELECT o.*, t.t_label,
           CASE WHEN prev_event_time IS NULL
                     OR EXTRACT(EPOCH FROM (event_time - prev_event_time)) > t.t_sec
                THEN 1 ELSE 0 END AS is_new_session
    FROM   ordered o CROSS JOIN timeouts t
), numbered AS (
    SELECT *,
           SUM(is_new_session) OVER (PARTITION BY user_id, t_label
                                     ORDER BY event_time, event_id
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
    FROM   flagged
)
SELECT t_label,
       COUNT(DISTINCT (user_id, session_seq))                     AS sessions,
       ROUND(63.0 / COUNT(DISTINCT (user_id, session_seq)), 2)    AS events_per_session
FROM   numbered
GROUP  BY t_label;
```

| timeout | sessions | events/session | single-event sessions | bounce rate |
|---------|---------:|---------------:|----------------------:|------------:|
| **1 min** | 59 | 1.07 | 56 | **94.9 %** |
| **5 min** | 37 | 1.70 | 27 | **73.0 %** |
| **30 min** (default) | 35 | 1.80 | 23 | **65.7 %** |
| **60 min** | 35 | 1.80 | 23 | **65.7 %** |
| **240 min (4 h)** | 35 | 1.80 | 23 | **65.7 %** |

**Read the plateau.** Between 5 minutes and 4 hours, the session count does not move: 37 → 35 → 35 →
35. The entire sample has no gap between 5 and 295 minutes, so every timeout in that range gives
almost the same answer. That flat region is where you want to be, and it is exactly why the industry
settled on 30 minutes.

The two regimes on either side:

- **Below the plateau (1 min):** sessions explode (59 ≈ the event count), the bounce rate approaches
  95 %, and you are measuring the SDK's flush interval, not user behaviour. Every event becomes its
  own "visit".
- **Above the plateau (not reachable here):** sessions start to merge across days, durations become
  absurd (a "session" of 26 hours), and page-view-per-session numbers become fiction.

**Practical procedure**, which you should run on every new dataset and re-run after any SDK change:

1. Compute session counts for a logarithmic range of timeouts (1 s, 10 s, 1 m, 5 m, 30 m, 1 h, 4 h, 24 h).
2. Plot (or tabulate) sessions vs timeout; find the plateau.
3. Pick the centre of the plateau.
4. Check that it is consistent with the product's usage cadence (a daily-use app's plateau center is
   minutes; a weekly-use app's can be hours).
5. Freeze the value in the metric catalogue and in the `dim_session` model.

### 4.4 Example 4 — adding a maximum duration cap

**Problem statement.** Force-close any session longer than 2 hours, on data where a cap actually
bites. Here is a constructed "always-on dashboard" stream:

```sql
WITH raw(user_id, event_time) AS (
    VALUES (1, TIMESTAMP '2024-06-01 08:00:00'),
           (1, TIMESTAMP '2024-06-01 08:20:00'),
           (1, TIMESTAMP '2024-06-01 08:40:00'),
           (1, TIMESTAMP '2024-06-01 09:00:00'),
           (1, TIMESTAMP '2024-06-01 09:20:00'),
           (1, TIMESTAMP '2024-06-01 09:40:00'),
           (1, TIMESTAMP '2024-06-01 10:00:00'),
           (1, TIMESTAMP '2024-06-01 10:20:00'),
           (1, TIMESTAMP '2024-06-01 10:40:00'),
           (1, TIMESTAMP '2024-06-01 11:00:00'),
           (1, TIMESTAMP '2024-06-01 11:20:00')
), flagged AS (
    SELECT *, CASE WHEN event_time - LAG(event_time) OVER (ORDER BY event_time)
                            > INTERVAL '30 minutes'
                     OR LAG(event_time) OVER (ORDER BY event_time) IS NULL
                THEN 1 ELSE 0 END AS is_new_session
    FROM raw
), numbered AS (
    SELECT *, SUM(is_new_session) OVER (ORDER BY event_time
                                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS seq
    FROM flagged
), started AS (
    SELECT *, MIN(event_time) OVER (PARTITION BY seq) AS seq_start FROM numbered
), capped AS (
    SELECT *, CASE WHEN event_time - seq_start > INTERVAL '2 hours' THEN 1 ELSE 0 END AS cap_break
    FROM started
), final AS (
    SELECT *, SUM(cap_break) OVER (ORDER BY event_time
                                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cap_seq
    FROM capped
)
SELECT event_time, seq AS naive_session, cap_break, cap_seq AS capped_session
FROM   final ORDER BY event_time;
```

| event_time | naive_session | cap_break | capped_session | session length |
|------------|--------------:|:---------:|---------------:|----------------|
| 08:00 | 1 | 0 | 0 | 0 s |
| 08:20 | 1 | 0 | 0 | 20 min |
| 08:40 | 1 | 0 | 0 | 40 min |
| 09:00 | 1 | 0 | 0 | 60 min |
| 09:20 | 1 | 0 | 0 | 80 min |
| 09:40 | 1 | 0 | 0 | 100 min |
| 10:00 | 1 | 0 | 0 | 120 min (exactly at the cap — not exceeded) |
| 10:20 | 1 | **1** | **1** | 140 min → **cap fires** |
| 10:40 | 1 | 0 | 1 | |
| 11:00 | 1 | 0 | 1 | |
| 11:20 | 1 | 0 | 1 | |

The naive session runs 08:00 → 11:20 (**3 h 20 min**) because every gap is 20 minutes and the timeout
never fires. With a 2-hour cap, the same data becomes two sessions: 08:00–10:00 (120 min) and
10:20–11:20 (60 min). Without the cap, "session duration" for this user is 200 minutes; with it, the
mean is 90 minutes. **The cap is not cosmetic — it is the difference between measuring a visit and
measuring uptime.**

### 4.5 Example 5 — day-boundary breaks

**Problem statement.** Add a local-day boundary so a session cannot span midnight. Take an event at
23:50 and one at 00:10 (a 20-minute gap — under any normal timeout):

```sql
WITH raw(user_id, event_time) AS (
    VALUES (1, TIMESTAMP '2024-06-01 23:50:00'),
           (1, TIMESTAMP '2024-06-02 00:10:00'),
           (1, TIMESTAMP '2024-06-02 00:25:00')
), ordered AS (
    SELECT *, LAG(event_time) OVER (ORDER BY event_time) AS prev_time FROM raw
)
SELECT event_time,
       CASE WHEN prev_time IS NULL
                 OR event_time - prev_time > INTERVAL '30 minutes'
                 OR event_time::date <> prev_time::date        -- ← the day boundary rule
            THEN 1 ELSE 0 END AS is_new_session
FROM ordered ORDER BY event_time;
```

| event_time | prev_time | is_new_session | why |
|------------|-----------|:--------------:|-----|
| 2024-06-01 23:50:00 | `NULL` | 1 | first event |
| 2024-06-02 00:10:00 | 23:50 | **1** | gap is only 20 min, but the **date changed** |
| 2024-06-02 00:25:00 | 00:10 | 0 | same day, 15 min gap |

Without the `::date <> ::date` condition this would be **one** session spanning midnight, and both
days' daily reports would disagree about whether it happened. With it, each day gets its own session
and each daily report is self-contained. The cost: a user who is genuinely active at midnight appears
as two sessions in "sessions per day" metrics. That is a deliberate trade-off — Google Analytics makes
the same one.

### 4.6 Example 6 — sessionization on a different key

Sessions are entity-scoped, and the entity is a choice. The same pattern run with
`PARTITION BY device_id` produces device sessions, which is the right unit for a shared tablet or a
kiosk. Run with `PARTITION BY (user_id, app_version)`, it starts a new session when the app updates —
useful for release-cohort analysis. The pattern is identical; only the `PARTITION BY` changes:

```sql
LAG(event_time) OVER (PARTITION BY user_id, device_id
                      ORDER BY event_time, event_id)
```

> **Senior note.** Changing the sessionization key changes every downstream metric, and the change is
> invisible unless you name the key in the metric. `sessions_per_user` on device keys is higher than
> on user keys (a phone plus a laptop is two devices). If your DAU and your session metrics use
> different keys, your DAU can exceed your user-key sessions per user, which is a subtle
> inconsistency auditors love. Publish `sessionization_key` as part of the model's metadata.

### 4.7 Example 7 — session-level metrics that prove the value

**Problem statement.** Produce the session-based KPIs a product dashboard actually needs, and compare
them against the naive non-sessionized aggregates.

```sql
WITH sessions AS ( /* … the session table from §4.2 … */ ),
by_day AS (
    SELECT session_start::date AS d,
           COUNT(*)                                     AS sessions,
           COUNT(DISTINCT user_id)                      AS users,
           COUNT(*) FILTER (WHERE is_bounce)            AS bounces,
           SUM(event_count)                             AS events,
           SUM(duration_seconds)                        AS duration_s
    FROM   sessions GROUP BY 1
)
SELECT d,
       users, sessions, events,
       ROUND(1.0 * sessions / users, 2)                    AS sessions_per_user,
       ROUND(1.0 * events / sessions, 2)                   AS events_per_session,
       ROUND(100.0 * bounces / sessions, 1)                AS bounce_rate_pct,
       ROUND(1.0 * duration_s / sessions, 1)               AS mean_session_s
FROM   by_day
ORDER  BY d;
```

Selected days:

| d | users | sessions | events | sessions_per_user | events_per_session | bounce_rate_pct | mean_session_s |
|---|------:|---------:|-------:|------------------:|-------------------:|----------------:|---------------:|
| 2024-01-05 | 1 | 1 | 5 | 1.00 | 5.00 |   0.0 | 540.0 |
| 2024-01-06 | 1 | 2 | 6 | 2.00 | 3.00 |   0.0 | 360.0 |
| 2024-01-12 | 1 | 1 | 4 | 1.00 | 4.00 |   0.0 | 480.0 |
| 2024-01-26 | 1 | 1 | 1 | 1.00 | 1.00 | 100.0 |   0.0 |
| 2024-02-02 | 2 | 2 | 3 | 1.00 | 1.50 |  50.0 |  90.0 |
| 2024-03-11 | 1 | 1 | 1 | 1.00 | 1.00 | 100.0 |   0.0 |

**Without sessions** you can compute `events / distinct users` = "events per user", which
conflates "one user doing 20 things" with "20 users doing one thing each". With sessions you get
`sessions_per_user` (visit frequency) and `events_per_session` (depth per visit) — two different
product facts that a single events/user ratio cannot separate. This decomposition is the reason
sessionization exists.

## 5. Reusable template

### 5.1 The production sessionization model

```sql
-- Step 1: order with a deterministic tiebreak
WITH ordered AS (
    SELECT <entity_id>                       AS entity_id,
           <extra context cols>              AS ctx,
           <event_id>                        AS event_id,
           <event_time>                      AS event_time,
           LAG(<event_time>) OVER (PARTITION BY <entity_id>
                                   ORDER BY <event_time>, <event_id>) AS prev_time
    FROM   <events_clean>
    WHERE  <filters>
),
-- Step 2: flag boundaries (timeout + first event + optional day break)
flagged AS (
    SELECT *,
           CASE WHEN prev_time IS NULL
                     OR event_time - prev_time > INTERVAL :timeout
                     OR (<optional> date_trunc('day', event_time)
                                     <> date_trunc('day', prev_time))
                THEN 1 ELSE 0 END AS is_new_session
    FROM   ordered
),
-- Step 3: number the runs
numbered AS (
    SELECT *,
           SUM(is_new_session) OVER (PARTITION BY entity_id
                                     ORDER BY event_time, event_id
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
    FROM   flagged
),
-- Step 4 (optional): apply a maximum duration cap
started AS (
    SELECT *, MIN(event_time) OVER (PARTITION BY entity_id, session_seq) AS seq_start FROM numbered
),
capped AS (
    SELECT *, CASE WHEN event_time - seq_start > INTERVAL :max_duration THEN 1 ELSE 0 END AS cap_break
    FROM   started
),
final AS (
    SELECT *,
           SUM(cap_break) OVER (PARTITION BY entity_id
                                ORDER BY event_time, event_id
                                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_key
    FROM   capped
)
-- Step 5: emit the session dimension and the event→session mapping
SELECT entity_id, session_seq, session_key,
       MIN(event_time) AS session_start, MAX(event_time) AS session_end,
       COUNT(*) AS event_count,
       (EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))))::int AS duration_seconds,
       COUNT(*) = 1 AS is_bounce
FROM   final
GROUP  BY entity_id, session_seq, session_key;
```

### 5.2 Parameters and their defaults

| Parameter | Default | Note |
|-----------|---------|------|
| `<entity_id>` | `user_id` | name the key in the model: `sessionization_key = "user_id"` |
| `<event_id>` | the source's monotonic id | **never omit** |
| `:timeout` | `30 minutes` | pick from the plateau |
| `:max_duration` | `2 hours` | or `NULL` if you genuinely want "time on site" |
| day boundary | off for user analytics, **on** for daily reporting | |
| `<filters>` | bot/internal exclusion | get these out *before* sessionizing, or bots become sessions |

### 5.3 Idempotency and incremental strategy

```sql
-- Recompute only the tail that can still change; freeze everything older.
DELETE FROM mat_event_sessions
WHERE  session_start >= :now - INTERVAL :timeout - INTERVAL :lateness;

INSERT INTO mat_event_sessions
SELECT * FROM sessionization
WHERE  session_start >= :now - INTERVAL :timeout - INTERVAL :lateness;
```

Test: run the sessionization twice over the same window and assert identical `session_id`s. If the
test fails, the `ORDER BY` is not a total order.

### 5.4 Reconciliation test (batch vs streaming)

```sql
SELECT COUNT(*) AS disagreements
FROM  (SELECT session_id FROM batch_sessions EXCEPT SELECT session_id FROM stream_sessions) a;
-- Expect 0. Any non-zero value means the tiebreaker or the lateness window differs.
```

### 5.5 Test kit

| Test | SQL / assertion | Catches |
|------|-----------------|---------|
| Every event belongs to exactly one session | `SUM(event_count) = (SELECT COUNT(*) FROM events)` | dropped/fanned-out joins |
| No session exceeds the cap | `MAX(duration_seconds) <= cap + timeout` | missing cap |
| Sessions do not overlap within an entity | `COUNT(*) FILTER (WHERE session_start < LAG(session_end)) = 0` per entity | ordering bug |
| Session ids are unique per entity | `COUNT(*) = COUNT(DISTINCT (entity_id, session_key))` | collision |
| Re-run determinism | two runs produce identical ids | missing tiebreaker |
| Sorted within session | `MIN(event_time) = session_start` and count matches | mis-grouped rows |
| Bounce rate is in `[0, 1]` | `bounce_rate BETWEEN 0 AND 1` | impossible definitions |
| Plateau exists | the session-count series is flat over a 5–60 min range | timeout chosen in the noise regime |

## 6. Real-world use cases and applications

| Domain | Sessionization key | Timeout | Special breaks | Used for |
|--------|-------------------|---------|----------------|----------|
| **Web analytics** | visitor / cookie | 30 min | day boundary, campaign change | sessions, bounce rate, pages/session |
| **Mobile app** | install id or user id | 30–60 min | background timeout, app version | engagement, retention-per-session |
| **GA4 / Firebase** | user pseudo-id | 30 min (default) | **new session on campaign change, at midnight**, 24 h cap | the canonical reference definition |
| **Streaming / media** | user id | 30 min | playback pause vs. stop | watch time, completion rate |
| **E-commerce** | visitor id | 30 min | checkout start forces a boundary | session-scoped funnels |
| **Fintech / banking** | user id | 15–30 min | explicit login/logout events | secure session audit, fraud |
| **IoT / telemetry** | device id | 5 min | alert state transitions | device "sessions" ≈ device uptime |
| **API analytics** | API key + client | 5–10 min | rate-limit window | burst detection, capacity planning |
| **Search** | user id | 30 min | query refinement vs. new search | query chains, task success |
| **Support / chat** | customer id | 15 min | ticket close | handle-time, escalation chains |
| **Gaming** | player id | 30 min | match start/end | play sessions, session length |
| **Log / SIEM analysis** | host or principal | 10 min | privilege change, new source IP | security "sessions", lateral movement |
| **Data engineering** | the stream itself | the batch interval | — | microbatch boundaries, watermarks |

Two production stories:

1. **"Our average session is 47 minutes."** A dashboard used mean session duration with a 30-minute
   timeout but no cap, and the mobile SDK kept "sessions" alive across background transitions, so a
   phone left in a pocket all afternoon was one engagement. Adding a 2-hour cap and switching to the
   **median** produced "the typical session is 90 seconds, and 62 % of sessions are a single event"
   from the same data. The first number had been reported to investors for four quarters.
2. **"Realtime and daily dashboards disagree by 8 %."** A streaming sessionizer ordered events by
   arrival time and emitted sessions at `last_event + 30 min` with **zero** allowed lateness, while
   the batch job ordered by `event_time`. Late mobile events (queued uploads) were being dropped in
   the stream, so the streaming session count was 8 % lower. Setting allowed lateness to 4 hours,
   ordering by `(event_time, event_id)` in both, and adding the reconciliation query from §5.4 closed
   the gap to 0.0 % and caught the bug permanently.

## 7. Performance notes and anti-patterns

### 7.1 Cost model

Sessionization is **three window operations over the same ordering**: `LAG`, then `SUM` of the flag,
then the session aggregates. If the source already provides the order (a clustered index on
`(entity_id, event_time, event_id)`), all three run as a single streaming pass — `O(E)`. Otherwise you
pay one sort: `O(E log E)`.

Practical consequences:

- **Pre-sort physically.** In a lakehouse, cluster/sort the events table by
  `(entity_id, event_time, event_id)`. Delta `ZORDER`, Iceberg `sortOrder`, Snowflake `CLUSTER BY`,
  BigQuery clustering. This turns the whole thing into a streaming pass.
- **Never fold the window into one expression.** Some engineers write
  `SUM(is_new_session) OVER (ORDER BY …)` inline; the engine still needs the sort, so there is no
  saving — and it becomes unreadable.
- **Filter before the window.** Bots, internal users, and test accounts should be excluded *before*
  sessionizing, or they create sessions with absurd durations that pollute the aggregates.
- **Restrict the window.** Sessions cannot cross a long gap, so a query for "this month's sessions"
  only needs events from the last month plus `timeout` — unless a session genuinely spans months
  (which the cap prevents).

### 7.2 Streaming state budget

The dominant cost in a production sessionizer is **state per key**:

```
state ≈ active_keys × (open_sessions_per_key × (session_start + last_event + event_buffer))
```

For 10^7 concurrent users each with one open session and a 4-hour lateness buffer of 50 events, that
is 5 × 10^8 buffered events. Mitigations, in order of impact:

1. **Bound the lateness.** 30–60 minutes is usually enough; 4 hours is generous; unbounded is a bug.
2. **Answer only what you need.** If the downstream metric needs *counts* and *durations* (not the
   full event list), maintain an accumulator per open session (count, min time, max time, flags) and
   discard the events. State drops from `events` to `sessions`.
3. **Use a bounded state backend with TTL** (RocksDB in Flink, `Materialized`/`Persistent`
   KeyValueStore in Kafka Streams) and set a TTL ≥ timeout + lateness.
4. **Regulate key cardinality.** Sessionize on `user_id` (bounded by user count) rather than on a
   cookie or session token that churns.
5. **Two-tier:** sessionize per shard/partition and merge only at the boundary (an event that bridges
   a partition boundary is rare; handle it with a lookback window).

### 7.3 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| `ORDER BY event_time` only | nondeterministic sessions, non-reproducible backfills | add a unique tiebreaker |
| No first-event handling | the first event is never a boundary → `session_seq` starts at 0 and the logic inverts | `prev_time IS NULL OR gap > timeout` |
| `>=` instead of `>` on the timeout | a gap of *exactly* the timeout starts a new session; off-by-one across the whole dataset | decide and document; `>` is conventional |
| No cap | "sessions" of 20+ hours; "time on site" masquerading as engagement | add a 2–4 h cap |
| Sessionizing per query | slow, and definitional drift between dashboards | materialise `dim_session` + an event→session mapping |
| Changing the timeout without versioning | historical numbers become incomparable | version the model / record the timeout |
| Timeout chosen by copying a blog | you may be measuring SDK flush intervals | run the plateau analysis |
| No bot filtering | crawler "sessions" inflate sessions and deflate bounce rate | filter at the clean layer |
| Inconsistent entity key across models | DAU, sessions, and funnels disagree | one `sessionization_key`, declared once |
| Streaming with zero lateness | late events dropped; realtime < batch | allowed lateness ≥ observed max lateness + reconciliation test |
| Recomputing only today's events incrementally | sessions that started yesterday and continued are truncated | recompute the `timeout + lateness` tail |
| Using `COUNT(*)` over the session window for distinct users | double-counts if a user has two sessions | `COUNT(DISTINCT user_id)` |
| Averages in place of medians for duration | the bounce majority makes the mean meaningless | publish median + p90 and the bounce rate |

### 7.4 Scale techniques

- **Single-scan sessionization.** Because `LAG` and the running `SUM` share one ordering, the entire
  sessionization is one ordered pass. If the source is clustered correctly, there is no shuffle at
  all on a columnar warehouse.
- **Sessionize once, join many times.** A `mat_event_sessions` mapping table
  (`event_id → session_key`) plus `dim_session` serves funnels, bounce rate, and engagement metrics
  from the same artifact. This is the highest-leverage model in a product analytics warehouse.
- **Pre-aggregate for dashboards.** Sessions per user per day is a tiny table; build it and let the
  dashboard join nothing.
- **Sketch the big numbers.** For "sessions with ≥ N distinct event types" at scale, use
  `APPROX_COUNT_DISTINCT`/HLL per session, or a bitmap per session if the event-type cardinality is
  small (usually < 100, so a `BITMAP`/`VARBINARY` is perfect).
- **Partition by day, and handle the seams.** Sessions at a partition boundary need the previous
  partition's last event. Store `last_event_time` per entity at each partition boundary, or
  sessionize on a sliding window that overlaps by `timeout`.

## 8. Cross-engine differences

### 8.1 Sessionization primitives

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server 2012+ | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `LAG` over a window | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Running `SUM(flag) OVER (… ROWS UNBOUNDED PRECEDING)` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `timestamp - timestamp` → interval | ✅ → `interval` | ✅ → `INTERVAL` (as a number, use `TIMESTAMPDIFF`) | ✅ → needs `DATEDIFF` | ✅ → **number of days** | ✅ → `INTERVAL` / `TIMESTAMPDIFF` | ❌ → `TIMESTAMP_DIFF` |
| Compare a gap to a threshold | `> INTERVAL '30 minutes'` | `> INTERVAL 30 MINUTE` | `> '00:30:00'` (as `time`) or `DATEDIFF` | `* 1440 > 30` | `> INTERVAL '30 minutes'` | `> INTERVAL 30 MINUTE` |
| `date_trunc('day', …)` | ✅ | `DATE(ts)` | `CAST(ts AS date)` | `TRUNC(ts)` | ✅ | `DATE(ts)` |
| Native session windows (SQL surface) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ `session_window()` in streaming SQL |
| `MATCH_RECOGNIZE` (alternative formulation) | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| `EXCLUDE`/`GROUPS` frames (unrelated but useful) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 8.2 Dialect-specific gotchas

- **Oracle date arithmetic returns days.** `t2 − t1` is a `NUMBER` of days, so a 30-minute timeout is
  `(t2 - t1) * 1440 > 30`. Using `> INTERVAL '30' MINUTE` works for `TIMESTAMP` columns but not for
  `DATE` columns (which have second granularity). Pick your types deliberately.
- **BigQuery has no interval subtraction on timestamps.** Use
  `TIMESTAMP_DIFF(event_time, prev_time, SECOND) > 1800`. This is arguably clearer anyway.
- **MySQL's `TIMESTAMPDIFF` truncates.** `TIMESTAMPDIFF(MINUTE, a, b)` returns whole minutes
  rounded toward zero, so `TIMESTAMPDIFF(MINUTE, a, b) > 30` differs from `b - a > INTERVAL 30 MINUTE`
  for sub-minute differences. Use the `INTERVAL` form, or compare seconds:
  `TIMESTAMPDIFF(SECOND, a, b) > 1800`.
- **SQL Server's `DATEDIFF(SECOND, a, b)` counts *boundary crossings*, not elapsed seconds.** For
  timestamps this is fine, but for dates it means `DATEDIFF(DAY, '2024-01-01 23:59', '2024-01-02 00:01') = 1`
  even though the elapsed time is 2 minutes. Use `DATEDIFF_BIG(SECOND, …)` (or cast to
  `datetime2` and subtract) when the granularity matters.
- **Snowflake's `WEEK_START` and timezone session parameters** change `date_trunc('day', …)` results if
  you do not set them explicitly. For day-boundary breaks, convert to the local timezone first via
  `CONVERT_TIMEZONE`, then truncate.
- **BigQuery's `session_window()`** exists only inside streaming SQL (`FROM ... GROUP BY
  SESSION_WINDOW(...)`); for batch you still write the manual `LAG` + running-sum pattern. Note the
  streaming session window is *per key* and needs `GROUP BY key, session_window(...)`.
- **Flink / Kafka Streams are not SQL engines**, but both expose session windowing as a first-class
  construct (`EventTimeSessionWindows.withGap(Time.minutes(30))`, `SessionWindows.with(...)`) with the
  merging behaviour described in §3.6. If you are sessionizing in a stream processor, use the native
  operator and do not hand-roll the `LAG` pattern — the native operator handles merging; a hand-rolled
  one cannot.

### 8.3 Portability recipe

```sql
-- Portable boundary flag: compare seconds, never intervals
CASE WHEN prev_time IS NULL
          OR <gap_seconds_expr>(event_time, prev_time) > :timeout_seconds
     THEN 1 ELSE 0 END AS is_new_session

-- where <gap_seconds_expr> is:
--   PostgreSQL  : EXTRACT(EPOCH FROM (a - b))
--   MySQL       : TIMESTAMPDIFF(SECOND, b, a)
--   SQL Server  : DATEDIFF_BIG(SECOND, b, a)
--   Oracle      : (a - b) * 86400
--   Snowflake   : DATEDIFF(SECOND, b, a)
--   BigQuery    : TIMESTAMP_DIFF(a, b, SECOND)
```

The rest — `LAG`, the running `SUM`, and the session aggregates — is identical on all six engines.

## 9. Exercises — Intermediate (hints only)

1. **Sessionize the sample.** Produce 35 sessions with a 30-minute timeout. Which user has the most
   sessions, and how many?
   *Hint:* user 1 and user 7 both have 7. Verify `SUM(event_count) = 63`.
2. **The session table.** Produce `session_start`, `session_end`, `event_count`, `duration_seconds`
   and `is_bounce` per session. Confirm the global bounce rate is 65.7 %.
   *Hint:* `GROUP BY user_id, session_seq` after the run-numbering CTE.
3. **Why 1-2 and 1-3 are separate.** Identify the exact gap (4 h 55 min) and the timeout that would
   merge them. What would that do to user 1's session count and mean session duration?
   *Hint:* a 6-hour timeout merges them; session count drops from 7 to 6, and the merged session is
   ~19 h long (unless you cap it).
4. **Timeout plateau.** Run the sensitivity query and tabulate sessions for 1 s, 10 s, 1 m, 5 m, 30 m,
   1 h, 4 h, 24 h. Where is the plateau, and why?
   *Hint:* the sample's intra-session gaps are ≤ 5 min and its inter-session gaps are ≥ 19 h, so the
   plateau is very wide.
5. **Median vs mean duration.** Compute both for the session table. Explain in one sentence why the
   median is 0 and why that is the correct headline number.
   *Hint:* 23 of 35 sessions are bounces.
6. **Add a day boundary.** Re-run the sessionization with a local-day break and count how many
   sessions change. Which sessions in the sample actually span midnight?
   *Hint:* none — no session in the sample crosses midnight, so the answer is 0. Now construct one
   (23:50 / 00:10) and show the flag firing.
7. **Add a cap.** Using the constructed 20-minute-interval stream from §4.4, show that the naive
   session is 3 h 20 min and the capped version is 2 sessions. What should the cap be for a
   "media streaming" product vs a "banking app"?
   *Hint:* compare "content length" and "session cadence" against typical session lengths.
8. **Sessions per user.** Compute `sessions_per_user` and `events_per_session` for the whole sample.
   Why can neither number be derived from `events / distinct_users`?
   *Hint:* 35/7 = 5.0 and 63/35 = 1.80; `63/7 = 9.0` conflates them.
9. **Determinism test.** Add an artificial duplicate timestamp (two events at the same second) and
   show that dropping `event_id` from the `ORDER BY` makes the session assignment ambiguous. What is
   the fix?
   *Hint:* run the query with the tiebreaker and without it and compare session counts.
10. **Different entity key.** Re-run the sessionization grouped by `(user_id, date_trunc('day',
    event_time))` — i.e. one "session run" per user per day. How many sessions does that produce, and
    what product question does it answer?
    *Hint:* this is the day-boundary rule applied as a partition instead of a flag; it produces one
    run per user per active day — **34** in the sample (user 1 has 6 active days, user 2 has 5, user 3
    has 4, user 4 has 5, user 5 has 4, user 6 has 3, user 7 has 7). It answers "how many separate days
    did this user visit on", which is a *frequency* metric, not a *session* metric — a useful reminder
    that "session" and "visit day" are different units and both get called "a session" in meetings.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Exact batch/stream parity.** Build a batch sessionizer and a streaming one (Flink, Kafka Streams,
   or Spark Structured Streaming) over the same synthetic event stream with 5 % late arrivals at up
   to 6 hours, and prove they produce identical session ids.
   *Expected approach:* order **both** by `(event_time, event_id)` (arrival order is not session
   order), set allowed lateness to the observed maximum lateness, and assert equality with an `EXCEPT`
   in both directions. Then state the sensitivity: at what lateness does the gap return to zero, and
   how much state does that cost? Conclude with the design rule — the watermark must dominate the
   source's lateness distribution, and the reconciliation test belongs in CI, not in a one-off
   investigation.
2. **Sessionization under a non-monotone late-arrival regime.** A mobile SDK buffers up to 48 hours and
   can deliver a *burst* of events that conceptually belong to a session already closed by the
   streaming job. Design a sessionizer that is correct in the presence of such bursts without keeping
   unbounded state.
   *Expected approach:* two-layer design — a "provisional" streaming layer with a short lateness
   (fast, may split), and an authoritative batch/`MATERIALIZED VIEW` layer with a 48-hour lookback
   that recomputes sessions whose `session_start >= now − 48 h − timeout`. Publish the provisional
   series for real-time alerting and the authoritative one for reporting, and *label them*. Discuss
   the cost of the recompute window, and how you would bound it if the lookback were 30 days instead
   of 2.
3. **Timeout selection as a measurable decision.** Define an objective function for the timeout (for
   example: maximise the temporal stability of session boundaries under small perturbations of event
   times, or minimise the fraction of sessions that are split by a `± 10 %` jitter in the timeout),
   then compute it over a range of timeouts and pick the optimum. Compare against the plateau method.
   *Expected approach:* inject Gaussian jitter into event times, recompute sessions for each candidate
   timeout, measure the Jensen–Shannon divergence or the simple disagreement rate between the
   perturbed and unperturbed partitions, and plot it against the timeout. Show that the plateau region
   is also the stability region. This is the rigorous version of "pick the middle of the plateau" and
   is the argument you use when a stakeholder asks why not 15 minutes.
4. **Sessions that survive a schema/key change.** The SDK upgrade adds `device_id` to events, and the
   business now wants sessions per *device* while historical data has only `user_id`. Design the model
   so that (a) historical sessions keep their original definition, (b) new sessions use the device key,
   and (c) any metric can be computed consistently across the break.
   *Expected approach:* version the sessionization key (`sessionization_key = 'user_id' | 'device_id'
   | 'hybrid'`), keep both `dim_session_v1` and `dim_session_v2`, and publish a `session_key_version`
   dimension. For cross-version metrics, define them on the *finer* key and roll up, or accept a
   documented discontinuity and annotate the chart. Discuss why "recompute history with the new key"
   is usually the wrong answer (it silently restates published numbers) and what you would do instead
   (publish a restatement with a version, in a changelog the business sees).
5. **State-budgeted sessionizer.** Your streaming sessionizer must hold at most 2 GB of state with 5 ×
   10^7 daily active users, a 30-minute timeout, a 1-hour lateness, and a 4-hour cap. Design the state
   layout, compute the budget, and state what you would drop first if the key space doubled.
   *Expected approach:* sessionize per shard/partition and merge only at the boundary; maintain an
   accumulator (`session_start`, `last_event`, `event_count`, `flags`) rather than buffering events;
   use a hash map keyed by `(shard, entity)` with TTL `timeout + lateness`; estimate
   `active_keys × per_key_bytes` and show the arithmetic. Then discuss the two levers — bound the
   lateness, and reduce the per-key footprint by only tracking the *entities that had an event in the
   last `timeout + lateness`* — and quantify each. Also state the correctness cost of reducing the
   lateness (which events get mis-sessionized, and how you would detect it).
6. **Sessionization as a data-quality gate.** Sessions are an excellent anomaly detector: a sudden
   change in session count, events/session, or bounce rate usually means the SDK changed something.
   Build the monitoring that would have caught each of the three failure modes in §2 automatically.
   *Expected approach:* publish the session metrics as a daily series; add three tests — a
   volume anomaly on session count (z-score against a 28-day moving average with a weekday-aware
   baseline), a *definitional* stability test (the fraction of events within a session whose gap
   exceeds the timeout must be exactly 0), and a determinism test (two runs → identical ids) — plus
   the batch/stream parity test from exercise 1. Then argue why the "definitional" and "determinism"
   tests are more valuable than the volume anomaly, because they fail loudly and precisely instead of
   statistically. This is the observability counterpart to the whole chapter.

## References

- Google Analytics — Measure sessions and user engagement (30-minute default, campaign change, midnight, 24 h cap): <https://developers.google.com/analytics/devguides/collection/ga4/sessions>
- ga4bigquery — How to sessionize your GA4 event data in BigQuery, part 1 (default 30-minute definition): <https://www.ga4bigquery.com/how-to-sessionize-your-ga4-event-data-in-bigquery-part-1-default-session-definition/>
- ga4bigquery — Sessionizing GA4 event data, part 2 (custom session definitions): <https://www.ga4bigquery.com/how-to-sessionize-your-ga4-event-data-in-bigquery-part-2-custom-session-definitions/>
- Google Cloud — BigQuery `SESSION_WINDOW` / window functions: <https://cloud.google.com/bigquery/docs/reference/standard-sql/window-function-calls>
- Google Cloud — BigQuery `TIMESTAMP_DIFF`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/timestamp_functions>
- Apache Flink — Session windows (`EventTimeSessionWindows`, merging): <https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/operators/windows/#session-windows>
- Apache Flink — Generating watermarks (bounded out-of-orderness, allowed lateness): <https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/event-time/generating_watermarks/>
- Kafka Streams — Session windows: <https://kafka.apache.org/documentation/streams/developer-guide/dsl-api/>
- Spark Structured Streaming — `session_window` and watermarking: <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#types-of-time-windows>
- PostgreSQL — Window function calls (`LAG`, frames, `EXCLUDE`): <https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- PostgreSQL — `LAG` / `LEAD` / window function reference: <https://www.postgresql.org/docs/current/functions-window.html>
- MySQL 8.0 — Window functions (`LAG`, frames): <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- Microsoft Learn — `LAG` (Transact-SQL) and `DATEDIFF_BIG`: <https://learn.microsoft.com/en-us/sql/t-sql/functions/lag-transact-sql>
- Oracle — Analytic functions and date arithmetic (`DATE` subtraction returns days): <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/Analytic-Functions.html>
- Snowflake — Window function syntax and usage (`WEEK_START`, session parameters): <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Snowflake — `MATCH_RECOGNIZE`: <https://docs.snowflake.com/en/sql-reference/constructs/match_recognize>
- DuckDB — Window functions (local testing of the portable subset): <https://duckdb.org/docs/stable/sql/functions/window_functions.html>
- Sqlism — Sessionization in SQL: grouping events into sessions: <https://sqlism.com/sessionization-in-sql>
- pipecode — SQL gaps & islands: sessionization, streaks and run-length patterns: <https://pipecode.ai/blogs/sql-gaps-and-islands-sessionization-streaks>
- Amplitude — How time works in a retention analysis (session and period definitions): <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-time>
- dbt Labs — Incremental models (bounded-tail recompute for session models): <https://docs.getdbt.com/docs/build/incremental-models>

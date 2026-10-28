# Funnels

> A funnel is a sequence-matching problem wearing a dashboard's clothes. Everyone can write
> `COUNT(DISTINCT user) WHERE step = k` for each step; almost nobody writes the *ordered* version,
> because the ordered version is a subsequence match and subsequence matching is not expressible as
> independent per-step aggregates. The naive funnel — "each step's first occurrence, compared with
> `>`" — produces **false negatives** on a case you will hit in production (a user who adds to cart,
> browses another product, then adds that one too). This chapter gives you the naive version, shows
> exactly where it lies, and gives you the exact greedy version in one query with `LATERAL`.

## TL;DR

- A funnel is defined by four things: the **step sequence** (ordered event predicates), the
  **entity** (user, session, account), the **scope** (per session, per user, within N days), and
  whether **order is strict** or **lenient**.
- **Session-scoped, order-strict** is the only funnel that means what product teams think it means.
  User-scoped funnels over a lifetime window are *almost always* misleading (user converts in a
  different session three weeks later — that is not the funnel you redesigned).
- The **naive per-step aggregate** is fast and wrong-ish:
  `MIN(event_time) FILTER (WHERE event_type = step_k)` per session, then
  `t1 < t2 < t3 < …`. It is *sufficient* but not *necessary* for the existence of an ordered chain,
  so it **undercounts**. The fix is a sequential (greedy) match: at each step, take the earliest
  event of that type **after** the previously matched step.
- **The exact greedy match in one query** is a chain of `LEFT JOIN LATERAL (… MIN(...) WHERE t >
  prev_t)`, or a recursive CTE. Use `CROSS APPLY` on SQL Server, correlated subqueries on BigQuery.
- **Strict vs lenient ordering** changes the number by 10–40 % on real funnels. Strict requires
  `step_k` *after* `step_{k-1}`; lenient requires only that all steps occurred (in any order).
  Lenient is fine for "did they ever do X and Y", useless for "did the redesign fix the checkout
  flow".
- **Step definitions are event *sets*, not events.** `add_to_cart` is really
  `{add_to_cart, add_to_wishlist, quick_add}`. Model it as a mapping table
  (`funnel_step(step_order, step_name, event_type)`) so the funnel is data, not SQL.
- **Time-to-convert is a separate query** over the same matched steps: `t_{k} − t_{k−1}` per entity,
  then a distribution (median, p90) — not a mean, because funnels are heavy-tailed.
- **Drop-off is a column, not a separate dashboard.** `1 − (step_k / step_{k−1})` per step, plus the
  absolute loss, plus the *conditional* rate. The last one is the only one that tells you where to
  spend engineering time, because the per-step rate is confounded by the population that reaches the
  step.
- **Multi-path funnels** (allow any of several events per step, or skip optional optional steps)
  multiply the definition space; enumerate the paths you actually care about and add them to the
  metric catalogue as separate funnels. Do not build a "flexible funnel builder" in SQL.
- `MATCH_RECOGNIZE` (Snowflake, Oracle, BigQuery, Databricks) and `LISTAGG` + regex (the Oberoi
  technique) are the two idioms that express sequence matching declaratively. Both beat hand-written
  SQL on wide funnels; both are dialect-specific.

## 1. Definitions and core concepts

### 1.1 Vocabulary

| Term | Meaning |
|------|---------|
| **Funnel** | A strictly or loosely ordered sequence of event predicates, measured for a population. |
| **Step** | One predicate in the sequence, usually a set of event types (e.g. `{add_to_cart, quick_add}`). |
| **Entity** | What enters the funnel: a user, a session, an account, a cart, an order attempt. |
| **Scope / window** | The time and grouping constraint within which the sequence must occur (same session, same day, within 7 days). |
| **Strict ordering** | Each step's event must occur *after* the previous step's event. |
| **Lenient ordering** | Steps must all occur; order is not enforced (this is "did they do X and Y", not a funnel). |
| **Step count / conversion** | Entities that reached step `k` (having completed all previous steps). |
| **Step conversion rate** | `step_k / step_{k−1}` (conditional rate between adjacent steps). |
| **Overall conversion rate** | `step_n / step_1`. |
| **Drop-off** | `step_{k−1} − step_k`; the absolute number lost between steps. |
| **Time to convert (TTC)** | `t_k − t_{k−1}` or `t_n − t_1` per matched entity. |
| **Path** | One concrete sequence of event types that satisfies the funnel definition. |
| **Multi-path funnel** | A funnel whose step predicates admit several event types, so several paths satisfy it. |
| **Greedy (earliest-match) subsequence** | Find step 1's earliest event, then step 2's earliest event after it, etc. Optimal for existence testing. |
| **Sessionized events** | Events with a `session_id` assigned by an inactivity-timeout rule (chapter 07). |
| **Funnel population** | The denominator (all entities that reached step 1, not all users). |

### 1.2 The four definitional axes

```
┌── STEP SEQUENCE ─────────────────────────────────────────────────────┐
│ 1 page_view      →  2 view_product  →  3 add_to_cart                 │
│ →  4 checkout_start  →  5 purchase                                   │
└──────────────────────────────────────────────────────────────────────┘
        │                    │                       │
        ▼                    ▼                       ▼
   ENTITY               SCOPE                    ORDER
   user | session |     same session |           strict | lenient
   account | cart       same day |
                        ≤ N days |
                        lifetime
```

Every funnel number you have ever disagreed about differs along one of these four axes. Write all
four down before you write the SQL.

### 1.3 Why session scope beats user scope

Take the ShopStream sample. User 1 viewed a product and abandoned at checkout on **2024-01-12**
(4 events, no purchase) and then purchased on **2024-01-19** (a completely different session, a week
later, 5 events).

| Scope | Does user 1 convert? | What it measures |
|-------|---------------------|------------------|
| Per **session** | ✅ the 2024-01-19 session converts; the 2024-01-12 session does **not** | the flow works |
| Per **user, lifetime** | ✅ user 1 converts (because *some* session converted) | nothing actionable |
| Per **user, 1-day window** | ❌ user 1 does not convert | the flow is broken at checkout, and here is the proof |

The user-scope, lifetime version reports "our funnel converts" for a user who abandoned checkout and
came back a week later. A team shipping a checkout redesign reads that as success. The session-scoped
version correctly shows the abandonment, and the user-scoped 1-day version quantifies it.

**Default for product work: session scope with strict ordering.** Escalate to user scope only when
the product genuinely has multi-session journeys (a mortgage application, a B2B trial).

### 1.4 Strict vs lenient: a concrete difference

```sql
-- STRICT: step k must happen after step k-1
WHERE t_page_view < t_view_product AND t_view_product < t_add_to_cart

-- LENIENT: each step happened; order ignored
WHERE t_page_view IS NOT NULL AND t_view_product IS NOT NULL AND t_add_to_cart IS NOT NULL
```

| Variant | Typical effect on a 4-step checkout funnel |
|---------|-------------------------------------------|
| Strict | the real funnel; lower counts |
| Lenient | higher counts by 5–40 % depending on how often steps occur in unexpected orders |
| Strict, plus "within 1 hour" | removes the "came back next week" tail; another 5–20 % |

A user who adds to cart from a saved link, then browses the product page, satisfies *lenient* but not
*strict*. Enumerating the paths your users actually take (session-scoped, ordered) is how you find out
whether your "linear" funnel ever matched reality — often it doesn't, and the funnel diagram is a
fiction that a path analysis would expose.

## 2. Why it matters / what problem it solves

Funnels drive the most expensive product decisions a company makes:

| Decision | Needs | Because |
|----------|-------|---------|
| Where to spend engineering effort | the step with the largest *absolute* drop-off | a 90 % → 60 % drop costs more users than 10 % → 8 % |
| Did the redesign work? | before/after step conversion, session-scoped | the only fair comparison |
| Which channel converts best? | funnel segmented by `marketing_channel` | traffic quality is a conversion question |
| Is the checkout flow broken on mobile? | funnel by platform | device-specific bugs show as step cliffs |
| How long is the buying journey? | time-to-convert distribution | drives retargeting windows and email cadence |
| What is the CAC ceiling? | funnel × retention × ARPU | conversion is the first multiplier in the LTV chain |
| Is this a sparse-metric anomaly? | step counts vs. the 7-day moving average | a funnel step is a metric like any other |

Failure modes, from real teams:

- **The false-negative funnel.** A team shipped a checkout redesign and *stopped* investing in the
  old flow because the funnel showed add-to-cart → checkout at 71 %. The query used
  `MIN(event_time) FILTER (WHERE event_type = …)` per step with an ascending-time check, which
  undercounts sessions where a user adds to cart, browses another product, and adds again. The true
  rate was 84 %. The team had been optimising a 6-point phantom.
- **The lifetime funnel.** A B2C app measured "signup → first purchase" over a lifetime window and
  reported 42 % conversion. Session-scoped within 1 day it was 6 %. The 42 % number was being used to
  justify a paid acquisition budget; the 6 % number would have justified none of it.
- **The lenient funnel.** A "5-step onboarding funnel" reported 68 % completion. The steps were
  measured leniently, and users routinely did step 4 before step 2 because step 2 was optional and
  buried. The funnel diagram was legal but not real. Path analysis showed three dominant paths, none
  of which matched the drawn funnel.

## 3. The pattern / technique explained

### 3.1 The four implementations, ranked

| # | Approach | Exact? | Portability | Cost | Use when |
|---|----------|--------|-------------|------|----------|
| **1** | Per-step conditional aggregation + ascending-time check | ❌ (false negatives) | all engines | `O(events)` | quick dashboards; accept < 1 % error |
| **2** | Greedy chain with `LEFT JOIN LATERAL` / `APPLY` | ✅ | PG, MySQL 8.0.14+, SQL Server (`APPLY`), Snowflake, Redshift; BigQuery via correlated subqueries | `O(events × steps)` with index | **default for correctness** |
| **3** | Recursive CTE walking the steps | ✅ | PG, MySQL, SQL Server, Oracle, BigQuery (**not Snowflake**) | `O(steps × events)` | wide funnels (> 8 steps) |
| **4** | `MATCH_RECOGNIZE` or `LISTAGG` + regex | ✅ | Snowflake, Oracle, BigQuery, Databricks / regex engines | engine-optimised | wide funnels, path extraction |

### 3.2 Approach 1 — conditional aggregation (the one everybody writes)

```sql
SELECT f.session_id,
       (t1 IS NOT NULL)                                              AS reached_1,
       (t1 IS NOT NULL AND t2 IS NOT NULL AND t2 > t1)                AS reached_2,
       (t1 IS NOT NULL AND t2 IS NOT NULL AND t3 IS NOT NULL
        AND t2 > t1 AND t3 > t2)                                     AS reached_3
FROM (
    SELECT session_id,
           MIN(event_time) FILTER (WHERE event_type = 'page_view')      AS t1,
           MIN(event_time) FILTER (WHERE event_type = 'view_product')   AS t2,
           MIN(event_time) FILTER (WHERE event_type = 'add_to_cart')    AS t3
    FROM   sessionized_events
    GROUP  BY session_id
) f;
```

Fast, one pass, no self-joins. And **wrong** in a specific, constructible case (§4.4).

### 3.3 Approach 2 — the exact greedy match

The greedy (earliest-match) algorithm is the textbook solution for subsequence existence: match step
`k` at the earliest event *after* the position matched for step `k−1`. In SQL that is a chain of
correlated `MIN` subqueries, each depending on the previous one — which is exactly what
`LEFT JOIN LATERAL` (a.k.a. `OUTER APPLY`) provides:

```sql
SELECT f1.session_id,
       f1.t1,
       f2.t2,
       f3.t3,
       f4.t4,
       f5.t5
FROM  (SELECT session_id, MIN(event_time) AS t1
       FROM   sessionized_events WHERE event_type = 'page_view' GROUP BY session_id) AS f1
LEFT JOIN LATERAL (
    SELECT MIN(event_time) AS t2 FROM sessionized_events
    WHERE  session_id = f1.session_id AND event_type = 'view_product'
      AND  event_time > f1.t1
) AS f2 ON true
LEFT JOIN LATERAL (
    SELECT MIN(event_time) AS t3 FROM sessionized_events
    WHERE  session_id = f1.session_id AND event_type = 'add_to_cart'
      AND  event_time > f2.t2
) AS f3 ON true
LEFT JOIN LATERAL (
    SELECT MIN(event_time) AS t4 FROM sessionized_events
    WHERE  session_id = f1.session_id AND event_type = 'checkout_start'
      AND  event_time > f3.t3
) AS f4 ON true
LEFT JOIN LATERAL (
    SELECT MIN(event_time) AS t5 FROM sessionized_events
    WHERE  session_id = f1.session_id AND event_type = 'purchase'
      AND  event_time > f4.t4
) AS f5 ON true;
```

Each lateral sees the previous lateral's alias, so `t3` is guaranteed to be after `t2`, which is
guaranteed to be after `t1`. A `NULL` at level `k` means the session did not reach step `k`, and
every later level is automatically `NULL` because `event_time > NULL` is `NULL` (never true). The
folded funnel is then:

```sql
SELECT COUNT(*) FILTER (WHERE t1 IS NOT NULL) AS step_1,   -- always = session count
       COUNT(t2) AS step_2, COUNT(t3) AS step_3, COUNT(t4) AS step_4, COUNT(t5) AS step_5
FROM   matched;
```

`COUNT(t_k)` counts non-`NULL` values, which is precisely "reached step `k`". That is the whole
funnel aggregate.

### 3.4 Approach 3 — recursive greedy walk (wide funnels)

With a `funnel_step(step_order, event_type)` mapping table you can walk any number of steps:

```sql
WITH RECURSIVE walk AS (
    -- seed: step 1
    SELECT s.session_id, 1::int AS step_order, MIN(s.event_time) AS matched_at
    FROM   sessionized_events s
    JOIN   funnel_step f ON f.step_order = 1 AND f.event_type = s.event_type
    GROUP  BY s.session_id
    UNION ALL
    -- step k+1: earliest event of the next step's type after step k's match
    SELECT w.session_id, w.step_order + 1, nx.matched_at
    FROM   walk w
    JOIN   funnel_step f ON f.step_order = w.step_order + 1
    CROSS  JOIN LATERAL (
        SELECT MIN(s.event_time) AS matched_at
        FROM   sessionized_events s
        WHERE  s.session_id = w.session_id
          AND  s.event_type = f.event_type
          AND  s.event_time > w.matched_at
    ) nx
    WHERE  nx.matched_at IS NOT NULL
)
SELECT step_order, COUNT(*) AS entities_reached
FROM   walk
GROUP  BY step_order
ORDER  BY step_order;
```

This emits one row per session per completed step, so counting rows per `step_order` gives the funnel
directly. Note the recursion depth equals the number of steps, which is fine (funnels are short), but
**Snowflake does not support recursive CTEs** — use approach 2 or 4 there.

### 3.5 Approach 4 — declarative sequence matching

**`MATCH_RECOGNIZE`** (Snowflake, Oracle, BigQuery, Databricks):

```sql
SELECT *
FROM   sessionized_events
MATCH_RECOGNIZE (
    PARTITION BY session_id
    ORDER BY     event_time
    MEASURES
        MATCH_NUMBER()                     AS match_no,
        FIRST(v.event_time)                AS t_start,
        LAST(p.event_time)                 AS t_end
    ONE ROW PER MATCH
    PATTERN (v+ w+ c+ k+ p+)
    DEFINE
        v AS event_type = 'page_view',
        w AS event_type = 'view_product',
        c AS event_type = 'add_to_cart',
        k AS event_type = 'checkout_start',
        p AS event_type = 'purchase'
);
```

The `PATTERN` regex is the funnel definition; `DEFINE` maps symbols to predicates. This is the most
readable funnel in SQL and the engine optimises it — but it exists on four engines and not on
PostgreSQL or MySQL.

**The regex-on-a-string trick** (from Vikram Oberoi's widely cited write-up): collapse each session's
event sequence into a code string and match it with a regular expression.

```sql
WITH coded AS (
    SELECT session_id, event_time,
           CASE event_type
                WHEN 'page_view'      THEN 'v'
                WHEN 'view_product'   THEN 'w'
                WHEN 'add_to_cart'    THEN 'c'
                WHEN 'checkout_start' THEN 'k'
                WHEN 'purchase'       THEN 'p'
                ELSE '.' END AS code
    FROM   sessionized_events
), seq AS (
    SELECT session_id,
           LISTAGG(code, '') WITHIN GROUP (ORDER BY event_time) AS path
    FROM   coded GROUP BY session_id
)
SELECT COUNT(*) FILTER (WHERE path ~ 'v.*w.*c.*k.*p') AS converted,   -- PostgreSQL regex
       COUNT(*)                                       AS sessions
FROM   seq;
```

Two caveats: `LISTAGG` has a length limit (huge sessions truncate, silently if you are not careful),
and the regex is greedy in a way that matches the *existence* of a subsequence — which is exactly
what you want. This is a **tremendous** path-analysis tool: `GROUP BY path` on the same CTE gives you
the top 20 actual paths, which is how you discover that your drawn funnel does not exist.

### 3.6 Time-to-convert

Two shapes, both cheap once you have the matched timestamps:

```sql
-- (a) Total time: step 1 → step 5, per converted entity
SELECT t5 - t1 AS total_ttc FROM matched WHERE t5 IS NOT NULL;

-- (b) Per-hop time, as a long table suitable for a distribution per hop
SELECT step_name, EXTRACT(EPOCH FROM (matched_at - prev_matched_at)) AS seconds
FROM   long_matched_steps;
```

Report the **median and p90**, never the mean. Funnel TTC distributions have a long right tail
(a few users take 28 days), so the mean is dominated by the tail and describes nobody.

### 3.7 Drop-off, the three numbers

```sql
SELECT step_order, step_name, entities_reached,
       LAG(entities_reached) OVER (ORDER BY step_order)                                 AS prev_reached,
       entities_reached - LAG(entities_reached) OVER (ORDER BY step_order)               AS absolute_drop,
       1.0 - (entities_reached::numeric / NULLIF(LAG(entities_reached) OVER (ORDER BY step_order), 0)) AS drop_rate,
       entities_reached::numeric / NULLIF(FIRST_VALUE(entities_reached) OVER (ORDER BY step_order), 0) AS pct_of_top
FROM   funnel;
```

| Number | Answers | Trap |
|--------|---------|------|
| **Absolute drop** | how many users we lose | ranks steps correctly for engineering effort |
| **Step conversion rate** | quality of this hop | confounded: a hop reached by very committed users looks good |
| **% of top** | what fraction of the initial population survives to here | what executives read |

The most common analytical error is optimising the *rate* rather than the *absolute* loss. A step
that converts 95 % → 90 % loses the same number of users as one that converts 10 % → 5 % *only if the
same number of users reach them*. Compute both, and pick the fix with the largest absolute drop that
you can actually move.

## 4. Step-by-step worked example (with sample data and expected output tables)

### 4.1 The input: sessionized events

Funnels are built on sessions (chapter 07). For a self-contained example, here is the sessionization
CTE (30-minute inactivity timeout) applied to the shared `events` table:

```sql
WITH marked AS (
    SELECT user_id, event_id, event_type, event_time,
           CASE WHEN event_time
                     - LAG(event_time) OVER (PARTITION BY user_id
                                             ORDER BY event_time, event_id)
                     > INTERVAL '30 minutes'
                  OR LAG(event_time) OVER (PARTITION BY user_id
                                           ORDER BY event_time, event_id) IS NULL
                THEN 1 ELSE 0 END AS is_new_session
    FROM   events
), sessionized AS (
    SELECT user_id, event_id, event_type, event_time,
           SUM(is_new_session) OVER (PARTITION BY user_id
                                     ORDER BY event_time, event_id
                                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_seq
    FROM   marked
)
SELECT user_id || '-' || session_seq AS session_id,
       user_id, event_type, event_time
FROM   sessionized
ORDER  BY user_id, event_time;
```

That produces **35 sessions** across 7 users. Counted against the strict 5-step funnel
(`page_view → view_product → add_to_cart → checkout_start → purchase`):

| user | sessions | of which complete the 5-step funnel | notes |
|------|---------:|------------------------------------:|-------|
| 1 | 7 | 2 | 2024-01-05 and 2024-01-19 |
| 2 | 5 | 1 | 2024-01-07 |
| 3 | 4 | 0 | never got past `view_product` |
| 4 | 5 | 0 | reached `add_to_cart`, never `checkout_start` |
| 5 | 4 | 0 | never got past `view_product` |
| 6 | 3 | 0 | never got past `view_product` |
| 7 | 7 | 0 | `page_view` only — the "came back seven days running but never browsed" pattern |
| **total** | **35** | **3** | overall conversion **8.6 %** |

And the path table (§4.6) reveals the key asymmetry: **four** sessions contain a `purchase` event,
but only **three** satisfy the 5-step chain. The fourth — user 1's 2024-01-06 14:00 session — is
`page_view → view_product → add_to_cart → purchase`, with **no `checkout_start` at all**. That is a
"Buy Now" path: the user bought directly from the product page and never touched the cart-checkout
flow. It is not a data error, it is a different journey, and every funnel definition decision you make
will either include it or exclude it. **Decide explicitly.**

### 4.2 Example 1 — the session-scoped, strict funnel

```sql
WITH marked AS ( /* … as above … */ )
, sessionized AS ( /* … as above … */ )
, f1 AS (SELECT session_id, MIN(event_time) AS t1
         FROM sessionized WHERE event_type = 'page_view' GROUP BY session_id)
SELECT COUNT(*)                          AS step_1_page_view,
       COUNT(f2.t2)                      AS step_2_view_product,
       COUNT(f3.t3)                      AS step_3_add_to_cart,
       COUNT(f4.t4)                      AS step_4_checkout_start,
       COUNT(f5.t5)                      AS step_5_purchase
FROM   f1
LEFT JOIN LATERAL (SELECT MIN(event_time) AS t2 FROM sessionized
                   WHERE session_id = f1.session_id AND event_type = 'view_product'
                     AND event_time > f1.t1) f2 ON true
LEFT JOIN LATERAL (SELECT MIN(event_time) AS t3 FROM sessionized
                   WHERE session_id = f1.session_id AND event_type = 'add_to_cart'
                     AND event_time > f2.t2) f3 ON true
LEFT JOIN LATERAL (SELECT MIN(event_time) AS t4 FROM sessionized
                   WHERE session_id = f1.session_id AND event_type = 'checkout_start'
                     AND event_time > f3.t3) f4 ON true
LEFT JOIN LATERAL (SELECT MIN(event_time) AS t5 FROM sessionized
                   WHERE session_id = f1.session_id AND event_type = 'purchase'
                     AND event_time > f4.t4) f5 ON true;
```

| step | step name | entities | step conversion | drop-off from previous | % of top |
|-----:|-----------|---------:|----------------:|-----------------------:|---------:|
| 1 | page_view | 35 | — | — | 100.0 % |
| 2 | view_product | 12 | 34.3 % | 23 | 34.3 % |
| 3 | add_to_cart | 7 | 58.3 % | 5 | 20.0 % |
| 4 | checkout_start | 5 | 71.4 % | 2 | 14.3 % |
| 5 | purchase | 3 | 60.0 % | 2 | **8.6 %** |

**Read this table as a senior analyst would:**

- The **biggest absolute loss is step 1 → 2 (23 sessions)**. That is the top of the funnel: users land,
  look at nothing, and leave. A "did you find what you wanted?" search prompt belongs there.
- The **biggest relative loss is step 2 → 3 (41.7 %)**. Twelve sessions viewed a product and only
  seven put anything in a cart. That is a merchandising/price/shipping-cost problem, not a UX-flow
  problem.
- Steps 3 → 4 converts at 71.4 %, and **step 4 → 5 drops to 60.0 %**. Of the five sessions that
  reached `checkout_start`, three completed a purchase in the same session and two abandoned — the
  two abandoned ones are user 1's 2024-01-12 session and user 2's 2024-01-21 session, both of which
  are exactly the "reached checkout and left" pattern a payment-flow fix targets.
- Separately, **four** sessions in the whole dataset contain a `purchase` event, but only three
  satisfy the chain, because user 1's 2024-01-06 session bought via the "Buy Now" path with no
  `checkout_start`. That gap between "contains a purchase" (4) and "completed the funnel" (3) is the
  single most useful diagnostic in the table: it tells you the funnel does not describe every
  purchase path, which is a **definition** problem, not a product problem.
- **Only 3 of 35 sessions (8.6 %) complete the 5-step funnel.** The correct framing is "8.6 % of
  sessions", not "8.6 % of users" — the denominator is the deliverable.

**The 8.6 % is a definitional artefact as much as a product fact.** Drop `checkout_start` from the
funnel (a defensible choice, since the "Buy Now" path skips it) and the answer becomes
`35 → 12 → 7 → 4`, i.e. **11.4 %** — a 33 % relative improvement with no product change at all.
This is why the funnel definition belongs in the metric catalogue: a funnel step list is a *choice*,
and the choice moves the headline number by a third.

**The right resolution is not to remove the step, it is to fork the funnel.** Publish
`checkout_v1` (5 steps, requires `checkout_start`) as the metric for the cart-checkout redesign, and
`buy_path` (4 steps, `checkout_start` optional) as the metric for overall purchase propensity. Two
named funnels beat one ambiguous one.

### 4.3 Example 2 — strict vs lenient, session vs user

```sql
-- LENIENT, session-scoped: every step occurred in the session, order ignored
SELECT COUNT(*)                                          AS sessions,
       COUNT(*) FILTER (WHERE has_pv AND has_vp)         AS lenient_step_2,
       COUNT(*) FILTER (WHERE has_pv AND has_vp AND has_atc) AS lenient_step_3,
       COUNT(*) FILTER (WHERE has_pv AND has_vp AND has_atc AND has_chk) AS lenient_step_4,
       COUNT(*) FILTER (WHERE has_pv AND has_vp AND has_atc AND has_chk AND has_pur) AS lenient_step_5
FROM (
    SELECT session_id,
           BOOL_OR(event_type = 'page_view')      AS has_pv,
           BOOL_OR(event_type = 'view_product')   AS has_vp,
           BOOL_OR(event_type = 'add_to_cart')    AS has_atc,
           BOOL_OR(event_type = 'checkout_start') AS has_chk,
           BOOL_OR(event_type = 'purchase')       AS has_pur
    FROM   sessionized GROUP BY session_id
) x;
```

| variant | step 1 | step 2 | step 3 | step 4 | step 5 | overall |
|---------|-------:|-------:|-------:|-------:|-------:|--------:|
| **strict, session-scoped** | 35 | 12 | 7 | 5 | 3 | 8.6 % |
| **lenient, session-scoped** | 35 | 12 | 7 | 5 | 3 | 8.6 % |
| **strict, user-scoped (lifetime)** | 7 users | 6 | 3 | 2 | 2 | **28.6 %** |

Three things to take from this:

1. **Strict and lenient agree on this sample** (both 3 of 35). That is because the seed data was
   constructed as a linear funnel: no session ever fired a step out of order. Real data is not like
   this — see §4.4, which constructs the counterexample explicitly. Agreement here is *not* evidence
   that the two definitions are equivalent.
2. **The user-scoped number is 3.3× the session-scoped number** (28.6 % vs 8.6 %). It counts user 1
   because user 1 converted in *one* of their seven sessions, and user 2 for the same reason. If your
   funnel's job is to evaluate a flow, the user-scoped lifetime number is not the funnel — it is a
   population-level "did anyone ever do this" statistic.
3. **The rate comparison flips because the denominator collapses.** User scope finds *two* converting
   users (users 1 and 2), session scope finds *three* converting sessions (all belonging to those same
   two users). Yet 2/7 = 28.6 % is triple 3/35 = 8.6 %, because collapsing 35 sessions into 7 users
   shrinks the denominator 5× while shrinking the numerator only 1.5×. **The numerator and denominator
   must always be reported together, in the same sentence.**

The recommended default is the first row, and it should be *named* that way in the metric
catalogue: `checkout_v1_sessions_strict`.

### 4.4 Example 3 — the false negative (why approach 1 lies)

**Problem statement.** Construct a session that *does* satisfy `page_view → view_product →
add_to_cart` in order, and show that the naive per-step aggregate says it does not.

```sql
WITH trap(session_id, event_type, event_time) AS (
    VALUES ('X-1', 'page_view',       TIMESTAMP '2024-05-01 09:00:00'),
           ('X-1', 'add_to_cart',     TIMESTAMP '2024-05-01 09:10:00'),  -- abandoned cart
           ('X-1', 'view_product',    TIMESTAMP '2024-05-01 09:30:00'),  -- browse another product
           ('X-1', 'add_to_cart',     TIMESTAMP '2024-05-01 09:45:00')   -- add that one too
),
naive AS (
    SELECT session_id,
           MIN(event_time) FILTER (WHERE event_type = 'page_view')      AS t1,
           MIN(event_time) FILTER (WHERE event_type = 'view_product')   AS t2,
           MIN(event_time) FILTER (WHERE event_type = 'add_to_cart')    AS t3
    FROM   trap GROUP BY session_id
),
greedy AS (
    SELECT f1.session_id, f1.t1, f2.t2, f3.t3
    FROM  (SELECT session_id, MIN(event_time) AS t1 FROM trap
           WHERE event_type = 'page_view' GROUP BY session_id) f1
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t2 FROM trap
                       WHERE session_id = f1.session_id AND event_type = 'view_product'
                         AND event_time > f1.t1) f2 ON true
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t3 FROM trap
                       WHERE session_id = f1.session_id AND event_type = 'add_to_cart'
                         AND event_time > f2.t2) f3 ON true
)
SELECT n.t1, n.t2, n.t3,
       (n.t1 < n.t2 AND n.t2 < n.t3) AS naive_says_converted,
       g.t1, g.t2, g.t3,
       (g.t3 IS NOT NULL)            AS greedy_says_converted
FROM   naive n JOIN greedy g USING (session_id);
```

| naive t1 | naive t2 | naive t3 | naive says converted | greedy t1 | greedy t2 | greedy t3 | greedy says converted |
|----------|----------|----------|:--------------------:|-----------|-----------|-----------|:---------------------:|
| 09:00:00 | 09:30:00 | 09:10:00 | ❌ false | 09:00:00 | 09:30:00 | **09:45:00** | ✅ **true** |

**The session converted.** The user viewed a product, then added it to the cart. The naive query
looked at the *earliest* add-to-cart (the abandoned 09:10 one), found it before the product view, and
declared failure. The greedy query correctly picked the 09:45 add-to-cart, because it searched for
add-to-cart *after* the product view.

**This is not a contrived edge case.** It happens whenever a step type can fire more than once before
the previous step, and it is most common in exactly the funnels that matter: `add_to_cart` (multiple
products), `search` (multiple queries), `view_product` (browsing), `apply_coupon` (retries). On the
ShopStream sample it does not fire (the data is linear), which is precisely why you must test it with
a constructed case rather than trusting the real-data agreement.

> **Senior note.** If someone tells you "the two approaches agree on our data", ask them to run the
> comparison **per session** and count the sessions where they disagree. A single disagreement in
> 100,000 sessions is still a live bug in the pipeline that produced your number. The correct
> production posture is: ship approach 2 (exact) for the metric that drives decisions, and use
> approach 1 only for exploratory queries where you will re-run with approach 2 before publishing.

### 4.5 Example 4 — time to convert

**Problem statement.** For the four converting sessions, produce total TTC and per-hop TTC.

```sql
WITH /* … sessionized, f1 … f5 as in §4.2 … */
matched AS (
    SELECT f1.session_id, f1.t1, f2.t2, f3.t3, f4.t4, f5.t5
    FROM   f1
    LEFT JOIN LATERAL (…) f2 ON true
    LEFT JOIN LATERAL (…) f3 ON true
    LEFT JOIN LATERAL (…) f4 ON true
    LEFT JOIN LATERAL (…) f5 ON true
),
long_form AS (
    SELECT session_id, 1 AS step_order, 'page_view'      AS step_name, t1 AS matched_at, NULL::timestamp AS prev_at FROM matched WHERE t1 IS NOT NULL
    UNION ALL SELECT session_id, 2, 'view_product',   t2, t1 FROM matched WHERE t2 IS NOT NULL
    UNION ALL SELECT session_id, 3, 'add_to_cart',    t3, t2 FROM matched WHERE t3 IS NOT NULL
    UNION ALL SELECT session_id, 4, 'checkout_start', t4, t3 FROM matched WHERE t4 IS NOT NULL
    UNION ALL SELECT session_id, 5, 'purchase',       t5, t4 FROM matched WHERE t5 IS NOT NULL
)
SELECT step_order, step_name,
       COUNT(*)                                                            AS entities,
       ROUND(AVG(EXTRACT(EPOCH FROM (matched_at - prev_at))), 1)            AS mean_seconds,
       ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (matched_at - prev_at))), 1)
                                                                           AS median_seconds,
       ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (matched_at - prev_at))), 1)
                                                                           AS p90_seconds
FROM   long_form
WHERE  prev_at IS NOT NULL
GROUP  BY step_order, step_name
ORDER  BY step_order;
```

Per-hop results (seconds) — note `entities` equals the number of sessions that completed that hop:

| step_order | step_name | entities | mean | median | p90 |
|-----------:|-----------|---------:|-----:|-------:|----:|
| 2 | view_product | 3 | 140.0 | 120.0 | 180 |
| 3 | add_to_cart | 3 | 100.0 | 120.0 | 120 |
| 4 | checkout_start | 3 |  80.0 |  60.0 | 120 |
| 5 | purchase | 3 | 160.0 | 180.0 | 240 |

Total time to convert (step 1 → step 5) for the three converting sessions:

| session | t1 | t5 | total seconds | total minutes | hops (s) |
|---------|----|----|--------------:|--------------:|----------|
| u1-1 (2024-01-05) | 10:00:00 | 10:09:00 | 540 | 9 | 120, 120, 120, 180 |
| u1-5 (2024-01-19) | 08:30:00 | 08:40:00 | 600 | 10 | 180, 120,  60, 240 |
| u2-1 (2024-01-07) | 09:00:00 | 09:06:00 | 360 | 6 | 120, 120,  60,  60 |

`median = 540 s (9 min)`, `mean = 500 s`, `min = 360`, `max = 600`. In this sample the distribution
is narrow, so mean and median are close — but on production data you will see medians of 4 minutes
against means of 90 minutes, and the mean is the wrong number to put on a slide.

Two derived facts that change product decisions:

- **Every conversion happens within 10 minutes.** Retargeting windows of "7 days" for this funnel are
  chasing users who will never convert in it; the window should be under an hour, and the
  abandoned-cart email should fire in minutes, not hours.
- **The slowest hop is the purchase click (median 180 s, p90 240 s).** That is where a payment step
  feels slow. Adding a wallet / one-click payment targets the 3-minute gap, not the 2-minute ones.

### 4.6 Example 5 — path analysis (does your drawn funnel exist?)

**Problem statement.** Group sessions by their actual event-type sequence and see how many follow the
drawn order.

```sql
WITH coded AS (
    SELECT session_id, event_time,
           CASE event_type
                WHEN 'page_view'      THEN 'v'
                WHEN 'view_product'   THEN 'w'
                WHEN 'add_to_cart'    THEN 'c'
                WHEN 'checkout_start' THEN 'k'
                WHEN 'purchase'       THEN 'p'
                ELSE '.' END AS code
    FROM   sessionized
), seq AS (
    SELECT session_id,
           STRING_AGG(code, '' ORDER BY event_time) AS path,
           COUNT(*)                                 AS events
    FROM   coded GROUP BY session_id
)
SELECT path, COUNT(*) AS sessions,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_of_sessions,
       COUNT(*) FILTER (WHERE path ~ 'v.*w.*c.*k.*p')      AS strict_conversions
FROM   seq
GROUP  BY path
ORDER  BY sessions DESC, path;
```

| path | sessions | pct_of_sessions | strict_conversions |
|------|---------:|----------------:|-------------------:|
| `v` | 23 | 65.7 | 0 |
| `vw` | 5 | 14.3 | 0 |
| `vwckp` | 3 | 8.6 | 3 |
| `vwck` | 2 | 5.7 | 0 |
| `vwc` | 1 | 2.9 | 0 |
| `vwcp` | 1 | 2.9 | 0 |
| **total** | **35** | **100.1** (rounding) | **3** |

This is the most useful table in the chapter, and it is genuinely surprising:

- **The dominant path is a single `v` — one page view — at 65.7 % of sessions.** Two thirds of traffic
  never looks at a product. A step-conversion table hides this: it shows "34.3 % at step 2" rather
  than "65.7 % bounced at step 0". The largest lever in the whole product is invisible in the funnel
  chart and obvious in the path table.
- **Only six distinct paths exist**, and the 5-step linear funnel (`vwckp`) is just one of them. A
  funnel diagram with single arrows between boxes implies every converting session traversed that
  arrow set; here three of them did, **and one session (`vwcp`) purchased without ever reaching
  `checkout_start`** — proving the "linear funnel" is incomplete.
- `vwck` (2 sessions) is where the money is lost: users reached the checkout screen and left. That is
  a payment-flow problem with a name.
- `vwc` (1 session) is a cart-abandonment at the top: user 4 filled a cart and never got to checkout.

**The query to run before drawing any funnel diagram** is the complement: sessions that purchased
without following the drawn order.

```sql
SELECT path, COUNT(*) AS sessions
FROM   seq
WHERE  path LIKE '%p%'                       -- contains a purchase
  AND  path NOT LIKE 'v%w%c%k%p'             -- does not follow the drawn funnel
GROUP  BY path
ORDER  BY sessions DESC;
```

If that returns rows, your drawn funnel does not describe every purchase path and the diagram needs a
"Buy Now" branch (or the definition needs `checkout_start` marked optional, which is a different
funnel — see §4.2).

### 4.7 Example 6 — segmenting the funnel

**Problem statement.** The verifiable version of "which channel converts best" — with an explicit note
about small samples.

```sql
WITH /* … sessionized, f1 … f5 … */ matched,
sess_user AS (
    SELECT DISTINCT session_id, user_id FROM sessionized
)
SELECT u.marketing_channel,
       COUNT(*)            AS sessions,
       COUNT(m.t2)         AS step_2,
       COUNT(m.t5)         AS step_5,
       ROUND(100.0 * COUNT(m.t2) / COUNT(*), 1)        AS step_2_pct,
       ROUND(100.0 * COUNT(m.t5)  / COUNT(*), 1)       AS step_5_pct,
       CASE WHEN COUNT(*) < 100 THEN 'do not read: n < 100' ELSE 'ok' END AS reliability
FROM   sess_user su
JOIN   users u ON u.user_id = su.user_id
LEFT   JOIN matched m ON m.session_id = su.session_id
GROUP  BY u.marketing_channel
ORDER  BY sessions DESC;
```

The `reliability` column is the point. In the ShopStream sample every channel has fewer than 100
sessions, so **every** number in this table is an anecdote. Publishing an unlabelled funnel by channel
on `n = 4` sessions is how teams conclude "referral traffic converts terribly" from one user who
bounced. Make the reliability flag a column, not a footnote.

## 5. Reusable template

### 5.1 The exact funnel (LATERAL chain) — the default

```sql
WITH steps AS (                                     -- funnel definition as DATA
    SELECT * FROM (VALUES
        (1, 'page_view',      'page_view'),
        (2, 'view_product',   'view_product'),
        (3, 'add_to_cart',    'add_to_cart'),
        (4, 'checkout_start', 'checkout_start'),
        (5, 'purchase',       'purchase')
    ) AS t(step_order, step_name, event_type)
),
ev AS (                                             -- sessionized, filtered events
    SELECT session_id, event_type, event_time
    FROM   sessionized_events
    WHERE  event_time >= :start AND event_time < :end
      AND  <entity filters>
),
s1 AS (SELECT session_id, MIN(event_time) AS t01 FROM ev WHERE event_type = 'page_view' GROUP BY 1)
SELECT COUNT(*)   AS step_01_count,
       COUNT(t02) AS step_02_count,
       COUNT(t03) AS step_03_count,
       COUNT(t04) AS step_04_count,
       COUNT(t05) AS step_05_count
FROM (
    SELECT s1.session_id, s1.t01,
           f2.t AS t02, f3.t AS t03, f4.t AS t04, f5.t AS t05
    FROM   s1
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t FROM ev
                       WHERE session_id=s1.session_id AND event_type='view_product'
                         AND event_time > s1.t01) f2 ON TRUE
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t FROM ev
                       WHERE session_id=s1.session_id AND event_type='add_to_cart'
                         AND event_time > f2.t)    f3 ON TRUE
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t FROM ev
                       WHERE session_id=s1.session_id AND event_type='checkout_start'
                         AND event_time > f3.t)    f4 ON TRUE
    LEFT JOIN LATERAL (SELECT MIN(event_time) AS t FROM ev
                       WHERE session_id=s1.session_id AND event_type='purchase'
                         AND event_time > f4.t)    f5 ON TRUE
) m;
```

### 5.2 The generic funnel aggregate wrapper

```sql
WITH funnel AS (
    SELECT step_order, step_name,
           COUNT(...) AS entities      -- per step, from the matched chain
    FROM   steps s LEFT JOIN per_step ps USING (step_order)
    GROUP  BY step_order, step_name
)
SELECT step_order, step_name, entities,
       LAG(entities) OVER (ORDER BY step_order)                                      AS prev_entities,
       entities - LAG(entities) OVER (ORDER BY step_order)                            AS absolute_drop,
       ROUND(100.0 * entities / NULLIF(LAG(entities) OVER (ORDER BY step_order), 0), 1)  AS step_conversion_pct,
       ROUND(100.0 * entities / NULLIF(FIRST_VALUE(entities) OVER (ORDER BY step_order), 0), 1) AS pct_of_top,
       ROUND(100.0 * (1 - entities::numeric / NULLIF(LAG(entities) OVER (ORDER BY step_order), 0)), 1) AS drop_off_pct
FROM   funnel
ORDER  BY step_order;
```

### 5.3 Portability substitutions

| Feature | PostgreSQL | MySQL 8.0.14+ | SQL Server | Oracle | Snowflake | BigQuery |
|---------|-----------|---------------|-----------|--------|-----------|----------|
| Correlated lateral join (exact greedy) | `LEFT JOIN LATERAL (…) ON true` | `LEFT JOIN LATERAL (…) ON true` (derived tables) | `OUTER APPLY (…)` | `OUTER APPLY` | `LATERAL` | correlated scalar subquery in the `SELECT` list |
| Recursive CTE walk | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| `MATCH_RECOGNIZE` | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Regex match | `~`, `regexp_match` | `REGEXP_LIKE` | `LIKE` / limited | `REGEXP_LIKE` | `REGEXP_LIKE`, `RLIKE` | `REGEXP_CONTAINS` |
| String aggregation | `STRING_AGG(x, '' ORDER BY t)` | `GROUP_CONCAT(x ORDER BY t SEPARATOR '')` | `STRING_AGG(x,'') WITHIN GROUP (ORDER BY t)` | `LISTAGG(x,'') WITHIN GROUP (ORDER BY t)` | `LISTAGG(x,'') WITHIN GROUP (ORDER BY t)` | `STRING_AGG(x,'' ORDER BY t)` |
| `FILTER` on aggregates | ✅ | ❌ (`CASE`) | ❌ (`CASE`) | ❌ (`CASE`) | `COUNT_IF` | `COUNTIF` |

### 5.4 Definition checklist (put this in the metric catalogue, not in code comments)

| Item | Example value |
|------|---------------|
| Entity | `session_id` |
| Entry condition | `event_type = 'page_view'` |
| Steps | `[page_view, view_product, add_to_cart, checkout_start, purchase]` |
| Step event sets | step 3 = `{add_to_cart, quick_add, add_from_wishlist}` |
| Scope | same session (30-min inactivity timeout) |
| Ordering | strict |
| Time window | all data in the trailing 90 days |
| Counting | distinct sessions reaching each step |
| Match semantics | greedy earliest-match |
| Exclusions | `is_bot = false`, `is_internal = true`, `properties['test'] IS NULL` |
| Segment dimensions | channel, platform, country, plan |
| Reliability gate | publish only if `n >= 100` sessions at step 1 |

## 6. Real-world use cases and applications

| Domain | Funnel | Scope | Typical insight |
|--------|--------|-------|-----------------|
| **E-commerce** | view → product → cart → checkout → purchase | session | the two real levers are step 1 → 2 and 2 → 3 |
| **SaaS signup** | landing → signup → verify email → workspace → invite | user, 7 days | verification email is the usual cliff |
| **Fintech onboarding** | install → KYC start → document upload → approval → first transfer | user, 30 days | document upload is where identity verification fails |
| **Marketplace supply** | signup → first listing → first sale → repeat listing | user, 30 days | the "first sale" step predicts long-term supply |
| **Media** | impression → click → article read → subscribe | session | paywall timing is a funnel-placement problem |
| **Mobile app** | install → open → permission grant → core action | user, 1 day | permission prompts belong later in the funnel |
| **B2B sales** | lead → MQL → SQL → demo → opportunity → close | account, quarter | stage-slippage analysis; the funnel is per opportunity |
| **Healthcare** | referral → appointment booked → attended → treatment | patient, 90 days | attendance (no-show) is the biggest leak |
| **Education** | enrolment → first lesson → 3 lessons → completion | student, term | the "3rd lesson" is the retention cliff |
| **Payments** | cart → payment method → 3DS challenge → authorisation | session, 1 hour | 3DS failure is a synthetic, fixable step |
| **Data platform** | source registered → model built → model scheduled → dashboard built | team, quarter | adoption funnels work for internal platforms too |
| **Incident response** | alert → ack → triage → mitigation → postmortem | incident, 7 days | the postmortem step is where organisations leak |

Two production stories:

1. **"Our funnel says checkout converts at 71 %, so checkout isn't the problem."** The team had a
   false-negative bug (approach 1) that hid *abandoned-then-completed* sessions, which happened to be
   the most common pattern for repeat buyers. Switching to the greedy match dropped step 3 → 4 from
   71 % to 58 % and moved the roadmap. Same data, same steps; the only change was the algorithm.
2. **"We shipped a 5-step onboarding funnel and completion went up 12 points."** Completion went up
   because the release made step 2 auto-fill, which let users reach step 4 *without* step 2 — and the
   metric was measured leniently. The strict version was flat. The lenient funnel had been measuring
   "did the user visit five pages", not "did they complete onboarding". Re-defining the funnel
   formally (with the step sets and the strictness written down) is what surfaced it.

## 7. Performance notes and anti-patterns

### 7.1 Cost model

Let `E` be the number of events in scope, `S` the number of sessions, and `N` the number of steps.

| Approach | Cost | Dominant term |
|----------|------|---------------|
| 1 — conditional aggregation | `O(E)` | one grouped scan |
| 2 — LATERAL chain | `O(S × N)` index lookups (`O(E)` if the engine materialises the derived tables) | one correlated `MIN` per step per session |
| 3 — recursive CTE | `O(S × N)` recursive iterations | recursion overhead per step |
| 4 — `MATCH_RECOGNIZE` | `O(E)` with engine-specific optimisations | a single pass over the partitioned stream |

Practical guidance:

- **Filter hard before matching.** `WHERE event_time >= :start AND event_time < :end` plus
  `event_type IN (…)` reduces `E` enormously. A funnel query that scans all events to find five types
  is paying for data it will throw away.
- **Start from step 1's distinct entities, not from all events.** `s1` should be a small table
  (`S` rows). Every later level is then `S` correlated lookups.
- **Index for the correlated lookup:**
  `CREATE INDEX ix_events_session_type_time ON events_clean (session_id, event_type, event_time);`
  This turns each `MIN(event_time) WHERE session_id = ? AND event_type = ? AND event_time > ?` into a
  single index seek. Without it you get a nested loop with a scan per step per session, which is the
  classic "my funnel query takes 40 minutes" story.
- **Pre-compute `session_id` once** (chapter 07) and store it physically. Recomputing sessionization
  inside every funnel query is both slow and a source of definitional drift.
- **`MATCH_RECOGNIZE` is usually the fastest on engines that have it**, because the engine can
  compile the pattern into a single deterministic pass.

### 7.2 Anti-patterns

| Anti-pattern | Consequence | Fix |
|--------------|-------------|-----|
| `MIN` per step + ascending-time check | false negatives on repeated step types | greedy `LATERAL` chain or `MATCH_RECOGNIZE` |
| User-scoped, lifetime funnel | converts users who came back weeks later in a different flow | session scope (or an explicit ≤ N-day window) |
| Lenient ordering called a funnel | measures "visited several pages" | strict ordering, or rename the metric |
| No entry filter (denominator = all users) | dilution; the "funnel" starts before the journey | denominator = entities that reached step 1 |
| Counting `COUNT(*)` instead of `COUNT(DISTINCT entity)` | multi-event steps overcount | dedupe to one row per (entity, step) then `COUNT` |
| Hard-coding step names in SQL | changing the funnel requires a deployment | step mapping table |
| Funnel over all history | trends mixed, slow | fixed time window, published with the query |
| Reporting only step conversion rates | hides where the absolute loss is | publish absolute drop and `% of top` too |
| Mean time-to-convert | a tail of 28-day users makes the mean meaningless | median and p90 |
| Funnel by channel with n = 40 | anecdotes published as findings | reliability column and a minimum-N gate |
| No bot/internal filtering | funnels polluted by crawlers and employees | filter at the clean layer, once |
| Different funnels for the same journey in different dashboards | three "checkout conversion" numbers | one governed funnel definition |
| `LISTAGG` without a length guard | silently truncated paths → wrong regex result | `COUNT(events)` guard, or a smarter encoder |

### 7.3 Scale techniques

- **Materialise a `session_funnel_flags` table.** One row per session with `reached_step_1 … reached_step_n`
  and the matched timestamps. Every dashboard then reads a table of `S` rows instead of running the
  matcher. 35 sessions here; 10^8 in production, and still far smaller than `E`.
- **Incremental funnels.** Sessions are append-only per session_id: a session's matched steps only
  change when new events arrive for a session that is still open (say, the last hour). Recompute only
  open sessions incrementally; freeze closed ones. This makes funnels cheap at any scale.
- **Bound the window by session, not by time.** Because sessions are short (30-minute timeout), you
  can restrict the scan to sessions whose `session_start >= :start`; that is a much sharper cut than
  an event-time filter.
- **Approximate funnels for monitoring.** For alerting on step conversion, sample 1 % of users and
  compute the funnel on the sample. The variance is small enough for alerting and the cost drops 100×.
  Use the full funnel for reporting.
- **Columnar claws.** On Snowflake/BigQuery, funnel queries over `sessionized_events` are extremely
  fast because the `event_type` column is highly compressible and the correlated lookups become
  micro-partition / block pruning. On these engines the `MATCH_RECOGNIZE` form is nearly always the
  right choice.

## 8. Cross-engine differences

### 8.1 Feature matrix

| Capability | PostgreSQL | MySQL 8.0+ | SQL Server | Oracle | Snowflake | BigQuery |
|---|---|---|---|---|---|---|
| `LEFT JOIN LATERAL` | ✅ 9.3+ | ✅ 8.0.14+ (derived tables in `FROM`) | ❌ → `OUTER APPLY` | ❌ → `OUTER APPLY` | ✅ `LATERAL` | ❌ → correlated scalar subquery |
| `OUTER APPLY` / `CROSS APPLY` | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| Recursive CTE for the walk | ✅ | ✅ | ✅ | ✅ | ❌ (no recursive CTEs) | ✅ |
| `MATCH_RECOGNIZE` | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Regex predicate on a string | `path ~ 'v.*w.*c'` | `REGEXP_LIKE(path,'v.*w.*c')` | limited (`LIKE` only; no regex) | `REGEXP_LIKE` | `REGEXP_LIKE` / `RLIKE` | `REGEXP_CONTAINS` |
| `STRING_AGG`/`LISTAGG` with `ORDER BY` | `STRING_AGG(x,'' ORDER BY t)` | `GROUP_CONCAT(x ORDER BY t SEPARATOR '')` | `STRING_AGG(x,'') WITHIN GROUP (ORDER BY t)` | `LISTAGG(x,'') WITHIN GROUP (ORDER BY t)` | `LISTAGG(x,'') WITHIN GROUP (ORDER BY t)` | `STRING_AGG(x,'' ORDER BY t)` |
| List-agg size limit | none practical | `group_concat_max_len` (default 1 KB!) | `STRING_AGG` → `nvarchar(max)` | `VARCHAR2` limit | ≤ 16 MB | `STRING_AGG` → `STRING` (no limit) |
| `BOOL_OR` / boolean aggregates | ✅ `BOOL_OR` | ❌ (`MAX(cond)` / `SUM(cond)>0`) | ❌ (`MAX(CAST(… AS int))`) | ❌ (`MAX(CASE …)`) | ✅ `BOOLOR_AGG` | ✅ `LOGICAL_OR` |
| `FILTER (WHERE …)` | ✅ | ❌ | ❌ | ❌ | `COUNT_IF` | `COUNTIF` |
| Interval subtraction on timestamps | `t2 - t1` → `interval` | `TIMESTAMPDIFF(SECOND, t1, t2)` | `DATEDIFF(SECOND, t1, t2)` | `(t2 - t1) * 86400` | `TIMESTAMPDIFF(SECOND, t1, t2)` / `DATEDIFF` | `TIMESTAMP_DIFF(t2, t1, SECOND)` |
| `PERCENTILE_CONT` for TTC | ❌ as a window (ordered-set aggregate restriction); ✅ as an aggregate with `WITHIN GROUP` | ❌ (no `PERCENTILE_CONT`) | ⚠️ `OVER (PARTITION BY …)` only | ✅ aggregate | ✅ aggregate | ✅ aggregate |

### 8.2 Dialect traps that bite funnel SQL

- **MySQL's `GROUP_CONCAT` default cap is 1,024 bytes.** A session with 300 events silently truncates
  and your regex funnel under-reports. Always `SET SESSION group_concat_max_len = 1000000;` and/or
  add a guard column that fails the test when `LENGTH(path) = 1024`.
- **SQL Server has no regex and no `LATERAL`.** The exact greedy funnel becomes a chain of
  `OUTER APPLY (SELECT TOP 1 …)` clauses. This is fine and fast with the right index; just remember
  `TOP 1 … ORDER BY event_time` rather than `MIN()` if you need the row's other columns.
- **Snowflake has no recursive CTEs.** For wide funnels use the `LATERAL` chain (up to a practical
  ~15 steps) or `MATCH_RECOGNIZE`, which is the intended tool and reads best.
- **BigQuery forbids `LATERAL`; use a chain of correlated scalar subqueries**, which works because
  each subquery in the `SELECT` list can reference the *table's* columns but not another `SELECT`
  alias — so nest one level per step, or use `MATCH_RECOGNIZE` (available in GoogleSQL).
- **PostgreSQL's `LATERAL` is the cleanest of the six** and its `LEFT JOIN LATERAL (…) ON true` form
  handles the "step not reached" case naturally, because `MIN()` over an empty set returns `NULL`.
- **`PERCENTILE_CONT` in a window is rejected by PostgreSQL** ("ordered-set aggregates cannot be used
  as window functions"), so time-to-convert distributions must be computed as a grouped
  `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY seconds)` over a long-form step table, not as a window
  per entity. MySQL has no `PERCENTILE_CONT` at all; approximate with `NTILE` or an ordered
  self-join.
- **Timestamp arithmetic differs by a factor of 86,400.** PostgreSQL returns an `interval` from
  `t2 − t1`; Oracle returns a `NUMBER` of *days*; MySQL needs `TIMESTAMPDIFF`;
  BigQuery needs `TIMESTAMP_DIFF`. Extract seconds explicitly and test on a two-row fixture.

### 8.3 Portable funnel recipe

For SQL that runs on all six engines with the same semantics:

1. Pre-compute `session_id` (chapter 07) and filter events **once** into a `sessionized_events` table.
2. Use the **conditional-aggregation** form for exploratory work, and the **`LATERAL`/`APPLY`** form
   for anything published.
3. Replace `FILTER (WHERE …)` with `CASE WHEN … END` inside `COUNT`/`SUM`.
4. Replace regex path matching with an explicit `EXISTS`-style check or the `LATERAL` chain.
5. Compute time-to-convert as `EXTRACT(EPOCH FROM (t2 − t1))` on PostgreSQL and
   `DATEDIFF`/`TIMESTAMP_DIFF` elsewhere — or store a `seconds_since_epoch` column and subtract
   integers, which is the only truly portable option.

## 9. Exercises — Intermediate (hints only)

1. **Sessionize the sample.** Produce the 35 sessions from `events` using a 30-minute inactivity
   timeout. How many sessions does user 1 have, and why is the 2024-01-06 09:00–14:00 gap the boundary?
   *Hint:* `LAG(event_time)` + running `SUM` of the flag (chapter 07).
2. **The 5-step funnel.** Produce the table `35 → 12 → 7 → 5 → 4`. Verify the step conversion rates are
   34.3 %, 58.3 %, 71.4 %, 80.0 %.
   *Hint:* `LEFT JOIN LATERAL (SELECT MIN(event_time) …) ON true` chained five times.
3. **Strict vs lenient.** Compute both session-scoped variants for the sample and explain why they
   agree here. Then construct a session where they differ.
   *Hint:* `BOOL_OR` (or `MAX(CASE …)`) for lenient; a hand-built `VALUES` CTE for the difference.
4. **The false negative.** Reproduce §4.4's trap session and show the naive query returns
   `false` while the greedy query returns `true`. Then explain in one sentence why the naive query
   picks the wrong add-to-cart.
   *Hint:* `MIN` finds the earliest, not the earliest *after* the previous step.
5. **User-scoped vs session-scoped.** Compute the funnel in both scopes and reconcile the difference
   for user 1 specifically (7 sessions, 3 conversions).
   *Hint:* the entry table is `(user_id)` vs `(session_id)`; everything else is identical.
6. **Time to convert.** Produce the per-hop median and p90 in seconds. Which hop is slowest, and what
   product change does that suggest?
   *Hint:* long-form the matched timestamps, then group by step and use `PERCENTILE_CONT` (or `NTILE`).
7. **Path analysis.** Group sessions by their event-type string and produce the top paths. How many
   sessions have the path `v` only?
   *Hint:* `STRING_AGG(code, '' ORDER BY event_time)`.
8. **Drop-off analysis.** For each step produce `absolute_drop`, `step_conversion_pct` and
   `pct_of_top`, and state which step should be prioritised and why the answer is not the lowest
   percentage.
   *Hint:* `LAG` and `FIRST_VALUE` over `step_order`.
9. **Segment the funnel.** Repeat exercise 2 by `marketing_channel` and add a reliability flag. How
   many channels have `n >= 100` sessions?
   *Hint:* join `session_id → user_id → users.marketing_channel`.
10. **Bot filtering.** Add an `is_bot`-style exclusion (say, sessions with more than 100 events, or
    visits on a set of suspicious user agents) and quantify how much the funnel changes.
    *Hint:* a `HAVING` filter on the session aggregate; discuss why filtering must happen at the clean
    layer rather than per query.

## 10. Exercises — Advanced (expected approach, no full answers)

1. **Funnel with step event *sets* and optional steps.** Redesign the funnel so each step is a set of
   event types and some steps are optional (a "skip" path that does not consume the sequence
   position). Implement the matcher in SQL and prove it against a hand-built fixture with at least
   twelve sessions covering every path.
   *Expected approach:* the mapping table becomes `funnel_step(step_order, step_name, event_type,
   is_optional)`; the matcher must decide whether a matched optional step advances the pointer or not.
   The clean formulation is a small state machine with a recursive CTE (`step_order` in the recursion
   is the state) or `MATCH_RECOGNIZE` with a pattern like `v+ w* c+` where `*` denotes the optional
   step. Validate by asserting each fixture session's expected furthest step, and include at least one
   fixture where the *greedy* choice of an optional step gives the wrong answer (to show why
   greediness is not always optimal for optional steps, and what tie-break rule you adopted).
2. **Prove the two matchers differ on production-shaped data.** Build a synthetic session generator
   (a `generate_series` with a `CASE` for the path shape) producing 10^5 sessions with realistic
   repeated-step patterns, run both matchers, and quantify: how many sessions disagree, in which
   direction, and how the funnel's overall conversion rate changes. Then state which matcher you would
   publish and why.
   *Expected approach:* generate sessions from a small path alphabet (e.g. `vwcvkp`, `vwck`, `vwc`,
   `vw`, `v`, plus multi-`c` and multi-`w` variants), run approach 1 and approach 2, and
   `FULL OUTER JOIN` on `session_id` where `reached <> reached`. Report the disagreement rate and
   show that approach 1 is biased *downward* (it never produces false positives) — which means the
   naive funnel systematically understates conversion and therefore understates the size of the win
   available. That asymmetry is the actionable finding.
3. **Funnel attribution by entry path.** For each session that converts, compute the entry path (the
   first three events) and measure step conversion conditioned on it. Then decide whether the funnel
   should be *segmented by entry path* — i.e. whether there are really several funnels wearing one
   name.
   *Expected approach:* encode the path, `GROUP BY` the prefix, and compare step conversion per
   prefix with a minimum-N gate. Conclude with a governance recommendation: either split the protocol
   into N named funnels, or add an entry-path dimension to the single funnel. Discuss why "one funnel
   per journey, with entry-path as a segment" is usually the right answer and when it is not.
4. **Sessionization-aware funnel boundaries.** Sessions are assigned by a 30-minute inactivity
   timeout, but your funnel contains a step that can legitimately take longer than 30 minutes
   (e.g. waiting for a bank transfer). Design a funnel scope that does not depend on the activity
   session at all — an explicit "funnel window" — and compare it against the session-scoped version.
   *Expected approach:* replace the session boundary with a per-entity sliding window: for each
   candidate step-1 event, match the remaining steps within `:window` (using
   `event_time > prev AND event_time <= prev + INTERVAL :window`, chained). Show the two versions
   side by side and discuss which is correct for which question, plus the cost implication (the
   sliding window cannot be sessionized, so it is `O(E)` per step rather than `O(S)`).
5. **Time-to-convert as a survival problem.** Model the probability of converting *by* time `t`
   after step 1, accounting for the censoring of sessions that may still convert later. Give the
   SQL for the conversion curve and its daily hazard, and explain what it changes about a
   retargeting-window decision.
   *Expected approach:* build `(session, ttc_seconds, converted)` rows, bucket into time bins, compute
   the proportion converted by each bin over *eligible* (fully observed within the window) sessions,
   and produce the discrete hazard `h(t) = converted_in_bin / at_risk_at_bin_start`. Censor bins
   beyond `as_of − session_start`. Discuss the difference between "X % converted within 24 h" and the
   hazard shape (front-loaded vs long-tailed) and how each drives an email-timing decision. Note
   `PERCENTILE_CONT` cannot be used as a window function in PostgreSQL — use a grouped aggregate
   over the long form.
6. **A funnel metric with a reliability contract.** Define the funnel as a governed metric with an
   explicit minimum-N rule, an alerting band, and a change-detection policy — then implement it and
   the tests. Specifically: how do you alert when step 3 conversion moves, without firing on
   week-to-week noise, and without silently changing the definition when the funnel's step list
   evolves?
   *Expected approach:* version the funnel definition (`funnel_version_id`), store step counts and
   matched-session counts per version, publish a Wilson/binomial interval around each step rate based
   on the step-level `n`, and alert only when the observed rate leaves the interval for N consecutive
   periods. Add a test that fails if two different `funnel_version_id`s are mixed in one chart. This is
   the metrics-governance exercise the chapter is building toward: a funnel number without its
   definition, scope, strictness and minimum-N gate is not a metric.

## References

- PostgreSQL — `SELECT` (`LATERAL`, `LEFT JOIN LATERAL … ON true`): <https://www.postgresql.org/docs/current/sql-select.html>
- PostgreSQL — Window functions and ordered-set aggregates (why `PERCENTILE_CONT` is not a window function): <https://www.postgresql.org/docs/current/functions-window.html>
- PostgreSQL — Aggregate expressions (`FILTER`, `BOOL_OR`, `STRING_AGG`): <https://www.postgresql.org/docs/current/functions-aggregate.html>
- MySQL 8.0 — Window functions and `LATERAL` derived tables: <https://dev.mysql.com/doc/refman/8.0/en/window-functions.html>
- MySQL 8.0 — `GROUP_CONCAT` (`group_concat_max_len`): <https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_group-concat>
- Microsoft Learn — `OUTER APPLY` / `CROSS APPLY`: <https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql>
- Microsoft Learn — `STRING_AGG` (Transact-SQL): <https://learn.microsoft.com/en-us/sql/t-sql/functions/string-agg-transact-sql>
- Oracle — `MATCH_RECOGNIZE` (row pattern matching): <https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/sql-pattern-matching-data-warehouses.html>
- Oracle — `LISTAGG`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/LISTAGG.html>
- Oracle — `PERCENTILE_CONT`: <https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/PERCENTILE_CONT.html>
- Snowflake — `MATCH_RECOGNIZE`: <https://docs.snowflake.com/en/sql-reference/constructs/match_recognize>
- Snowflake — `LATERAL`: <https://docs.snowflake.com/en/sql-reference/constructs/join-lateral>
- Snowflake — `COUNT_IF` and window frame restrictions: <https://docs.snowflake.com/en/sql-reference/functions-window-syntax>
- Google Cloud — BigQuery `MATCH_RECOGNIZE`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#match_recognize_clause>
- Google Cloud — BigQuery `TIMESTAMP_DIFF` and `STRING_AGG`: <https://cloud.google.com/bigquery/docs/reference/standard-sql/timestamp_functions>
- Vikram Oberoi — Funnel analysis in SQL using window functions, range frames, and regular expressions: <https://vikramoberoi.com/posts/funnel-analysis-in-sql-using-window-functions-range-frames-and-regular-expressions/>
- Amplitude — Funnel analysis: how order and scope change the result: <https://amplitude.com/docs/analytics/charts/funnel-analysis/funnel-analysis-interpret>
- Amplitude — How time works in a retention analysis (the same aligned-index logic applied to funnels): <https://amplitude.com/docs/analytics/charts/retention-analysis/retention-analysis-time>
- Mode / ThoughtSpot — SQL Window Functions tutorial: <https://mode.com/sql-tutorial/sql-window-functions/>
- ga4bigquery — Sessionizing GA4 event data in BigQuery, part 1 (30-minute default used above): <https://www.ga4bigquery.com/how-to-sessionize-your-ga4-event-data-in-bigquery-part-1-default-session-definition/>
- dbt Labs — Metrics and the semantic layer (governing funnel definitions): <https://docs.getdbt.com/docs/build/build-metrics-intro>

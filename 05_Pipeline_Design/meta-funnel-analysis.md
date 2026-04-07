# Meta Design Pattern: Funnel Analysis
**Source:** DataExpert.io Boot Camp — Video 17 (Week 4, Analytics Track)
**Topic:** Funnel pipeline design, conversion tracking, drop-off analysis

---

## Key Concepts

### Funnel Definition
A sequence of steps where users must complete step N to be eligible for step N+1.

Example: Ad funnel
```
Impression → Click → Landing Page → Add to Cart → Purchase
```

### Key Metrics per Funnel Step
| Metric | Formula |
|--------|---------|
| Entry count | Users who reached step N |
| Exit count | Users who reached step N but not step N+1 |
| Conversion rate | step_n+1 / step_n |
| Drop-off rate | 1 - conversion_rate |
| Time to convert | Median time between step N and step N+1 |

### Funnel Table Design
```sql
-- One row per user per funnel entry (session-based)
CREATE TABLE fct_funnel_events (
  user_id BIGINT,
  session_id STRING,
  step_name STRING,        -- 'impression', 'click', 'purchase'
  step_order INT,
  event_ts TIMESTAMP,
  ds DATE
)
PARTITIONED BY (ds);

-- Funnel aggregation
SELECT
  step_name,
  COUNT(DISTINCT user_id) AS users,
  COUNT(DISTINCT user_id) / LAG(COUNT(DISTINCT user_id)) OVER (ORDER BY step_order) AS conversion_rate
FROM fct_funnel_events
WHERE ds BETWEEN '2024-01-01' AND '2024-01-07'
GROUP BY step_name, step_order
ORDER BY step_order;
```

### Attribution Patterns
- **First-touch** — credit the first interaction
- **Last-touch** — credit the last interaction before conversion
- **Linear** — distribute credit equally across all touchpoints
- **Time-decay** — more credit to recent touchpoints

### Ordered Event Sequences in SQL
```sql
-- Find users who completed step A then step B within 7 days
SELECT a.user_id
FROM fct_funnel_events a
JOIN fct_funnel_events b
  ON a.user_id = b.user_id
  AND b.step_name = 'purchase'
  AND b.event_ts > a.event_ts
  AND b.event_ts <= a.event_ts + INTERVAL '7 days'
WHERE a.step_name = 'click';
```

---

## Interview Angles
- How do you define a "session" for funnel analysis? → Typically 30-min inactivity window; can be session_id from frontend or server-side derived
- How do you handle users who enter the funnel multiple times? → Decide: first entry only, last entry only, or all entries (affects denominator)
- What causes a sudden drop at step 3 of a funnel? → Product change (UI), technical bug, targeting shift, external event

---

## Notes (from transcript)

### SQL interview reality check from Zach
- Zach has been asked to rewrite window function solutions without window functions "more than once in my career"
- "Makes me want to throw a chair because I solved the problem"
- Things asked but rarely used on the job: **recursive CTEs**, correlated subqueries
- Things actually used on the job: **COUNT CASE WHEN**, self-joins, CTEs, cumulative table design

### Lead/lag ↔ self-join equivalence (interview trick)
- `LAG(col, 1)` = self-join where `t2.date = t1.date - 1`
- When interviewer says "rewrite without window function" → this is almost always what they want
- Zach: "that's what they're usually looking for when they're asking you to write the query again"

### Self-join vs window function trade-offs

Both approaches can solve the same funnel/sequence problems. The choice depends on shape of the problem and scale.

**Self-join**
```sql
-- Find users who clicked then purchased within 7 days
SELECT a.user_id
FROM fct_funnel_events a
JOIN fct_funnel_events b
  ON a.user_id = b.user_id
  AND b.step_name = 'purchase'
  AND b.event_ts BETWEEN a.event_ts AND a.event_ts + INTERVAL '7 days'
WHERE a.step_name = 'click';
```

**Window function**
```sql
SELECT user_id
FROM (
  SELECT user_id, step_name, event_ts,
    LAG(step_name) OVER (PARTITION BY user_id ORDER BY event_ts) AS prev_step,
    LAG(event_ts)  OVER (PARTITION BY user_id ORDER BY event_ts) AS prev_ts
  FROM fct_funnel_events
)
WHERE step_name = 'purchase'
  AND prev_step = 'click'
  AND event_ts <= prev_ts + INTERVAL '7 days';
```

| Dimension | Self-Join | Window Function |
|---|---|---|
| **Flexibility** | High — can match any two non-adjacent steps freely | Lower — compares only adjacent rows by default |
| **Multi-step funnels** | Gets messy fast (3-step = 3 joins) | Clean — one pass classifies all steps |
| **Performance** | Expensive — O(n²) per user's events | Efficient — single scan with partition |
| **Non-adjacent steps** | Natural — `b.event_ts > a.event_ts` catches any future step | Requires chaining LAGs or subqueries |
| **Time window filtering** | Easy — `BETWEEN` in the JOIN condition | Easy — compare current and lagged timestamp |
| **Readability** | Intuitive for "step A then step B" queries | Requires understanding of window frame semantics |

**When to pick which:**
- Self-join: steps are non-adjacent, complex matching conditions, or 2-step funnel where clarity > performance
- Window function: ordered sequence classification in one pass, large-scale events tables, building aggregation layers

The LAG/self-join equivalence only holds cleanly for **strictly adjacent, ordered rows**. Once you need skips or time windows across arbitrary gaps, the self-join gives more control at the cost of compute.

### COUNT CASE WHEN — the most underrated SQL pattern
```sql
-- Count multiple conditions in one scan (no UNION needed)
SELECT
  COUNT(CASE WHEN user_state = 'new' THEN 1 END) AS new_users,
  COUNT(CASE WHEN user_state = 'retained' THEN 1 END) AS retained_users,
  COUNT(CASE WHEN user_state = 'churned' THEN 1 END) AS churned_users
FROM user_states
WHERE ds = '2024-01-01';
```
- "I've used that so many times to pivot out data — fast and efficient, scans the table once"

### GROUPING SETS / ROLLUP / CUBE — advanced GROUP BY
- Allows multiple aggregations in one query without UNION
- `GROUPING SETS ((a, b), (a), ())` = group by a+b, then just a, then grand total
- `ROLLUP(a, b)` = hierarchical rollup (most useful for time-based rollups)
- Zach: "a way to do multiple aggregations in one query without nasty UNIONs"

Assume a sales table: `(country, product_category, year, month, day, revenue)`

**GROUPING SETS** — explicitly pick exactly which combinations you want:
```sql
SELECT country, product_category, SUM(revenue)
FROM sales
GROUP BY GROUPING SETS (
  (country, product_category),  -- subtotal by country + category
  (country),                    -- subtotal by country only
  ()                            -- grand total
);
```
Equivalent to three queries UNIONed together — you control exactly what appears.

**ROLLUP** — hierarchical collapse from left to right, best for time or geography:
```sql
SELECT year, month, day, SUM(revenue)
FROM sales
GROUP BY ROLLUP(year, month, day);
-- Produces: (year, month, day), (year, month), (year), ()
```
Always goes from most granular to grand total automatically.

**CUBE** — every possible combination, useful for exploratory dashboards:
```sql
SELECT country, product_category, SUM(revenue)
FROM sales
GROUP BY CUBE(country, product_category);
-- Produces: (country, product_category), (country), (product_category), ()
```
With N dimensions, CUBE generates 2^N groupings — gets expensive fast.

| | Controls combinations | Use when |
|---|---|---|
| `GROUPING SETS` | You pick exactly | You know which aggregations you need |
| `ROLLUP` | Hierarchical collapse | Time/geo drilldowns |
| `CUBE` | All combinations | Exploration, small dimension count |

### EXPLAIN keyword — Zach's recommendation for SQL mastery
- Run `EXPLAIN <your query>` to see the abstract syntax tree / query plan
- "Once I started digging deeper into that kind of stuff, the odds that I wrote a bad performing query just went away"
- Order of SQL operations matters: FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

### Funnel analysis at Meta
- "Funnels are a massive part of analytics and data engineering"
- Implementation uses **self-join on event sequences** — join events table to itself to find step A → step B within a time window
- Definition of a "session" is a critical design decision: 30-min inactivity gap is common default

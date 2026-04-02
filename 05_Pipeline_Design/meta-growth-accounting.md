# Meta Design Pattern: Growth Accounting
**Source:** DataExpert.io Boot Camp — Video 16 (Week 4, Analytics Track)
**Topic:** User state transitions, growth accounting framework, retention analytics

---

## Key Concepts

### Growth Accounting Framework
Classifies every user on every day into exactly one state:

| State | Definition |
|-------|-----------|
| New | Active today, no activity before |
| Retained | Active today, active yesterday |
| Resurrected | Active today, NOT active yesterday, but active before |
| Churned | NOT active today, WAS active yesterday |

```
DAU = New + Retained + Resurrected
Net change = New + Resurrected - Churned
```

### Why This Matters
- Breaks DAU trend into its components → reveals the *cause* of growth/decline
- DAU going up but Retained going down → growing by acquisition, losing on retention → unsustainable
- Used at Meta for product health monitoring across all major surfaces (Feed, Stories, Groups)

### SQL Implementation
```sql
-- Step 1: Build yesterday + today activity snapshots
WITH today AS (
  SELECT DISTINCT user_id, ds FROM events WHERE ds = '{{ ds }}'
),
yesterday AS (
  SELECT DISTINCT user_id, ds FROM events WHERE ds = DATE_SUB('{{ ds }}', 1)
),
history AS (
  SELECT DISTINCT user_id FROM events WHERE ds < DATE_SUB('{{ ds }}', 1)
)

-- Step 2: Classify users
SELECT
  COALESCE(t.user_id, y.user_id) AS user_id,
  CASE
    WHEN t.user_id IS NOT NULL AND y.user_id IS NULL AND h.user_id IS NULL THEN 'new'
    WHEN t.user_id IS NOT NULL AND y.user_id IS NOT NULL THEN 'retained'
    WHEN t.user_id IS NOT NULL AND y.user_id IS NULL AND h.user_id IS NOT NULL THEN 'resurrected'
    WHEN t.user_id IS NULL AND y.user_id IS NOT NULL THEN 'churned'
  END AS user_state,
  '{{ ds }}' AS ds
FROM today t
FULL OUTER JOIN yesterday y ON t.user_id = y.user_id
LEFT JOIN history h ON COALESCE(t.user_id, y.user_id) = h.user_id;
```

### Detecting Fake Accounts
- Fake accounts often show "new" every day (recreated to avoid detection)
- Low resurrection rate for fake accounts (never come back after churn)
- Cross-reference with graph network clustering for anomalies

---

## Interview Angles
- How would you investigate a 10% DAU drop? → Decompose via growth accounting: which state drove the drop? Churned spike or Resurrected collapse?
- What does high New but low Retained tell you? → Acquisition is working but product retention is broken
- How does this pattern apply to non-user metrics? → Same framework for any entity: content, pages, groups

---

## Notes (from transcript)

### Repeatable analyses are your best friend
- Zach's core thesis: "if you can see these higher-level patterns then you'll know exactly what type of pipeline to implement"
- There are only **3 analytical pattern buckets** covering 95%+ of pipelines:
  1. **Aggregation-based** (Group By → sum/count/avg)
  2. **Accumulation-based** (Cumulative tables → state tracking)
  3. **Window-based** (Rolling calculations, lead/lag)
- Recognizing the pattern → "the SQL will write itself"

### State change tracking (Growth Accounting) = "opposite of SCD"
- SCD: keeps all values of a dimension over time
- State change tracking: keeps only the **change log** — when the state transitioned
- The four user states (new/retained/resurrected/churned) are a **state machine**

### Growth accounting at Facebook
- "Growth accounting is how Facebook tracks the inflows and outflows of active/inactive users"
- Facebook's internal name for this: tracking the **DAU decomposition**
- Closely related to cumulative table design from Week 1 — requires cumulative tables to work efficiently
- "These patterns don't work very well without cumulative table design"

### Survivorship Analysis (paired with growth accounting)
- Of all users who signed up on day X, what % are still active in 30/60/90 days?
- This is the **retention curve** — one of the most important metrics for any consumer product
- Implementation: `COUNT CASE WHEN` across different day offsets from signup date

### Why Zach wrote a dashboard for Mark Zuckerberg
- "At one point in my career I wrote a dashboard for Mark Zuckerberg and he liked it a lot"
- Point of the story: Zuckerberg doesn't want to write SQL — he wants the picture
- Data engineers must understand the right layer of abstraction for each stakeholder

### LLMs will change SQL but not analytical patterns
- "SQL is going to be different or how we create SQL is going to be different"
- "If you have a firmer grasp on these higher-level analytical patterns then you can be that ChatGPT-using engineer who can write pipelines incredibly quickly"
- Higher-level pattern knowledge is **more durable** than SQL syntax knowledge

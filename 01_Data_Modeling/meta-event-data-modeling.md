# How Meta Models Big Volume Event Data
**Source:** DataExpert.io Boot Camp — Video 9 (4-hour course, Week 2)
**Topic:** Fact data modeling, Kimball methodology, high-scale event data at Meta

---

## Key Concepts

### Fact Table Types
| Type | Grain | Example |
|------|-------|---------|
| Transaction | One row per event | Page views, clicks |
| Snapshot | One row per entity per period | Daily user activity summary |
| Accumulating Snapshot | One row per process instance | Order lifecycle |

### Long-term Analysis Framework (Meta)
- Solves problem: can't store every raw event forever (cost + query speed)
- Strategy: compress historical events into **dateless user-state** rows
- Uses bit manipulation to track activity across days in a single integer/BIGINT field
  - Each bit position = one day of activity
  - Fast population/engagement calculations without full table scans

### `dateless` / Bitset Pattern
```sql
-- Example: track 365 days of activity in a BIGINT
-- Bit 0 = today, Bit 1 = yesterday, etc.
-- User active on day N → set bit N

-- Check if user was active in last 30 days:
SELECT user_id,
  BIT_COUNT(activity_bits & ((1 << 30) - 1)) AS active_days_last_30
FROM user_activity_compressed;
```

### Kimball Methodology at Scale
- Still valid at Meta scale, but adapted:
  - Fact tables can be petabytes — partitioning strategy is critical
  - Dimension tables use SCD 2 for slowly changing attributes
  - Denormalization preferred over joins at query time

---

## Meta-Specific Patterns
- Event data partitioned by `ds` (datestamp) — never scan without partition filter
- User events aggregated to daily rollups before long-term retention
- Separation of "hot" data (last 90 days, full detail) vs. "cold" data (compressed summaries)

---

## Interview Angles
- How does Meta handle petabyte-scale event logs? → Tiered storage, daily rollups, bitset compression
- What is the tradeoff between raw events vs. pre-aggregated rollups? → Raw = flexibility; rollup = query speed + storage cost
- Why partition by date? → Prevent full table scans; partition pruning is the primary cost-control mechanism

---

## Notes (from transcript)

### Scale of fact data at big Tech
- Netflix: Zach worked on fact data that was **2 petabytes per day**
- Facebook: 2B users × 25–30 notifications/day = **50 billion notification events per day**
- Rule of thumb: fact data is **10–100x larger** than dimensional data for the same entity

### What makes a fact "atomic"
- A fact must be something you **cannot break down further**
- Example: a step (not a mile — a mile is an aggregation of steps)
- Facts don't change — they are immutable past events (unlike SCDs)

### The Date List (dateless) Data Structure — Meta's invention
- Problem: storing 365 rows per user per year is expensive at 2B users
- Solution: encode **30 days of activity as one BIGINT** using bit operations
  - Bit position N = whether user was active N days ago
  - MAU calculation = `BIT_COUNT(activity_bits & ((1<<30)-1))`
- Enables **decade-long analyses that used to take weeks to run in hours**
- Zach built this at Facebook as the **Long-Term Analysis Framework (LTAF)**

### The Day 3 concept: Reduced Facts
- Problem: full fact tables cause expensive shuffle in Spark/Trino
- Solution: **Reduced Fact** = pre-aggregated fact that preserves only the most important dimensions
- By building reduced facts you can completely eliminate shuffle for common analytical patterns
- Trade-off: less flexibility, but dramatically cheaper queries

### Blurry line between fact and dimension
- When you **aggregate a fact**, it starts behaving like a dimension
- Example: "number of purchases by user" is a fact aggregation that becomes a user attribute
- Key question: at what grain does it become a dimension? When it describes the entity, not an event

### Important context for labs
- Labs use NBA game details table in PostgreSQL
- Day 2 lab: build the dateless data structure with bit operations in PostgreSQL
- Day 3 lab: build a reduced fact framework to minimize shuffle

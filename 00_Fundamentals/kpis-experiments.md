# DE like a Product Manager: KPIs & Experiments
**Source:** DataExpert.io Boot Camp — Video 19 (Week 5, Analytics Track)
**Topic:** Metric ownership, KPI design, A/B testing infrastructure, experiment analysis

---

## Key Concepts

### DE's Role in Experimentation
- DEs don't just store data — they own the **metric pipelines** that power product decisions
- Bad experiment infrastructure → wrong decisions → wasted product investment
- Data engineers at Meta/Airbnb are responsible for: experiment assignment, logging, metric computation, and result reporting

### KPI Design Principles
| Principle | Description |
|-----------|-------------|
| Measurable | Can be computed from logged events |
| Actionable | Product team can influence it |
| Timely | Available fast enough for decisions |
| Non-gameable | Can't be inflated without real improvement |
| Aligned | Moves in same direction as business goal |

### Experiment Infrastructure Components
```
1. Assignment Service    → randomly assign users to control/treatment
2. Event Logging         → log treatment exposure + user actions
3. Metric Pipeline       → aggregate events into experiment metrics
4. Statistical Analysis  → compute p-values, confidence intervals
5. Reporting Dashboard   → present results to PMs and leadership
```

### A/B Test Metric Pipeline
```sql
-- Step 1: Get experiment assignments
SELECT user_id, variant  -- 'control' or 'treatment'
FROM experiment_assignments
WHERE experiment_id = 'exp_001'
AND ds = '{{ ds }}';

-- Step 2: Join to metric data
SELECT
  a.variant,
  COUNT(DISTINCT a.user_id) AS users,
  SUM(m.revenue) AS total_revenue,
  SUM(m.revenue) / COUNT(DISTINCT a.user_id) AS revenue_per_user
FROM experiment_assignments a
LEFT JOIN fct_purchases m ON a.user_id = m.user_id AND m.ds = a.ds
WHERE a.experiment_id = 'exp_001'
GROUP BY a.variant;
```

### Common Experiment Pitfalls
| Pitfall | Description | Fix |
|---------|-------------|-----|
| SUTVA violation | Treatment affects control group (network effects) | Use cluster-level randomization |
| Novelty effect | Users behave differently just because it's new | Run experiment longer (2+ weeks) |
| Peeking | Stopping early when results look good | Pre-commit to sample size before starting |
| SRM (Sample Ratio Mismatch) | Actual assignment ratio ≠ intended ratio | Check assignment pipeline logs |
| Metric dilution | Including non-exposed users in analysis | Filter to triggered/exposed users only |

### SRM Detection
```sql
-- Expected: 50/50 split. Actual ratio signals a bug.
SELECT
  variant,
  COUNT(*) AS user_count,
  COUNT(*) / SUM(COUNT(*)) OVER () AS actual_ratio
FROM experiment_assignments
WHERE experiment_id = 'exp_001'
GROUP BY variant;
-- If actual_ratio deviates >1% from expected → investigate assignment pipeline
```

---

## Interview Angles
- What is Sample Ratio Mismatch and why does it matter? → Assignment bug that invalidates the experiment; if control/treatment ratio is wrong, all metrics are suspect
- How do you measure the impact of a feature that not all users see? → Use "triggered analysis" — only include users who were actually exposed to the feature
- What's the DE's responsibility in an A/B test? → Ensure correct assignment logging, clean metric pipeline, SRM checks, and timely data availability

---

## Notes (from transcript)

### Thinking like a PM = being a more impactful DE
- "Thinking like a product manager is critical for being a good data engineer"
- Two ways metrics make DEs more impactful:
  1. Build good KPIs/metrics that **change business decision-making**
  2. Build metrics that are **plugged into experiments** (A/B tests)

### Why metrics matter differently at different companies
- **Facebook/Meta**: extremely data-driven, Zuckerberg = "I am a robot" energy. Numbers go up = good. Numbers go down = bad.
- **Airbnb**: more design/storytelling culture (Brian Chesky = "artsy fartsy guy"). Metrics need a narrative, not just numbers.
- Lesson: understand your company's culture before deciding how to present metrics

### Metrics inform data modeling
- Your KPIs tell you what the data table must look like
- If you design the metrics before writing SQL, you'll get the grain and schema right
- "The metrics that you defined in the spec really did inform what the data table should look like"

### Good metrics properties (Zach's criteria)
1. **Visibility** — they help you see what's happening in your business
2. **Actionable** — the product team can actually influence them
3. **Aligned** — they move in the same direction as the business goal
4. Zach adds: **Can avoid icebergs** — "the more clarity and visibility you have, the fewer icebergs you're going to crash into"

### DE's role in experiments (A/B testing)
- DEs own the **metric pipeline**, not just the data storage
- Responsibilities: experiment assignment logging, metric computation pipeline, SRM detection
- Bad experiment infrastructure → wrong product decisions → wasted investment
- "If you can learn how to do both — build good KPIs and understand how experiments work — you will become a much more impactful data engineer"

### Zuckerberg dashboard story
- Zach built a dashboard that Mark Zuckerberg liked
- Key insight: Zuckerberg doesn't want to write SQL — he wants the picture at the right abstraction level
- Each stakeholder has a different "right layer" — DEs must understand all of them

# A/B Testing: Concepts, Statistics & DE Implementation
**Source:** YouTube — 课代表立正 (Cornell PhD, ex-Amazon/Meta/Tencent, Statsig early member)
- Video 1: AB Testing概览 (7 min overview)
- Video 2: AB实验，有哪些重要却不为人知的知识？(28 min deep dive)

---

## Part 1: Why A/B Testing?

### A/B Testing as a Causal Inference Tool
Most analyses only reveal **correlation** — users who use feature X retain better. But did X cause retention, or did already-engaged users just happen to use X?

A/B testing solves this via **randomization**. When you randomly assign users to control and treatment groups:
- Both groups are statistically equivalent on all characteristics (observed and unobserved)
- Any difference in outcome *must* be caused by the treatment
- This is why A/B testing is the gold standard for causal inference

The underlying principle is **exchangeability** — the potential outcome is independent of treatment assignment because of randomization.

### The Core Philosophy: Value Comes from Surprises
The true value of A/B testing is NOT confirming your good ideas work. It's discovering that you're wrong.

- **~80% of highly anticipated product ideas turn out to be ineffective or even harmful** when measured rigorously
- Without experiments, teams ship bad features confidently based on intuition
- With experiments, "surprises" (unexpected negative results) save the product from harm

> If you only run experiments to validate ideas you already believe in, you're missing the point.

### 100% Experiment Coverage via Feature Gates
Top companies (Meta, Statsig customers like OpenAI/Notion/Figma) achieve 100% experiment coverage by combining:
- **Feature Gate** (功能开关) — a toggle that controls which users see a feature
- **Experiment system** — assigns users to variants and logs exposure

Every new feature ships behind a Feature Gate → experiments become the **default**, not a separate project. The marginal cost of running an experiment approaches zero.

### Engineering Culture & Agency
A mature experiment system gives engineers real **agency** (自主权):
- Engineers propose and validate ideas with data, not politics
- Decisions are made by evidence, not by who argues loudest
- Builds a culture of **intellectual honesty** — you have to face the data even when it contradicts your beliefs

---

## Part 2: The Three Key Parameters

### 1. Significance Level (α — Alpha)
- The probability of a **false positive** (Type I error) — declaring a winner when there is no real effect
- Industry standard: α = 0.05 (5% chance of false positive)
- At 95% confidence interval: a result is "statistically significant" if p-value < 0.05

### 2. Statistical Power (1 - β)
- The probability of detecting a real effect when one exists
- β = Type II error (false negative — missing a real effect)
- Industry standard: power = 0.8 (80% chance of catching a real effect)
- Low power = you need a larger sample size to detect the same effect

### 3. Minimum Detectable Effect (MDE) / Sample Size
- The smallest effect size you want to be able to detect
- Smaller MDE → larger sample size required
- Relationship: `n ∝ (σ² × (z_α + z_β)²) / MDE²`
- These three parameters are locked together — fix two, and the third is determined

```
Sample size formula (simplified):
n = 16σ² / MDE²   (for α=0.05, power=0.8)
```

---

## Part 3: Building a Scalable Experiment System

### Foundation: Trusted Data & Metrics
A scalable experiment system requires:
- **Logging integrity** — every exposure must be logged reliably; missed logs invalidate results
- **Stable metric definitions** — metrics must be defined consistently before experiments run
- **Guardrail metrics** — metrics you must NOT harm (e.g. revenue, core retention), checked alongside primary metrics
- **Novelty effect awareness** — new features often get inflated early engagement; run experiments long enough to separate novelty from real effect

### A Dangerous Misconception: Data Scientists Doing Harm
A common mistake: a data scientist sees an experiment is trending positive and "helps" by recommending early shipment. This is wrong because:
- Early stopping inflates false positive rate dramatically
- The experiment hasn't accumulated enough power yet
- Good intentions + premature decisions = systematic bias toward shipping bad features

---

## Part 4: Advanced Statistical Techniques

### Why Textbook Hypothesis Testing Fails in Industry
Textbooks assume you run the test once and check the result at the end. In practice:
- Teams check dashboards daily ("peeking")
- Peeking inflates false positive rate — if you check 20 times at α=0.05, your real false positive rate is much higher
- Sample sizes are rarely exactly as planned
- Multiple metrics are tested simultaneously (multiple comparisons problem)

### Advanced Technique 1: CUPED — Variance Reduction (5x Speedup)
**CUPED** (Controlled-experiment Using Pre-Experiment Data) reduces the variance of your metric by incorporating pre-experiment data.

Core idea: much of the variance in a metric is explained by the user's *pre-experiment* behavior. Remove that explained variance → smaller confidence intervals → detect effects faster.

```
Y_adjusted = Y - θ × X_pre

Where:
  Y     = post-experiment metric (e.g. revenue during experiment)
  X_pre = same metric measured before experiment started
  θ     = covariance(Y, X_pre) / variance(X_pre)
```

Result: **same statistical power with ~5x fewer users**, or equivalently, reach significance much faster with the same user base. This is one of the highest-leverage techniques for teams that are sample-size constrained.

### Advanced Technique 2: Bayesian vs Frequentist
| | Frequentist | Bayesian |
|---|---|---|
| Output | p-value, confidence interval | Posterior probability distribution |
| Interpretation | "If null is true, probability of seeing this data" | "Probability that treatment is better" |
| Peeking problem | Severe — invalidates results | Naturally handles it |
| Prior knowledge | Ignored | Incorporated via prior |
| Industry use | Default (A/A tests, guardrails) | Useful for multi-armed bandits, faster decisions |

**What you actually need to care about:** Neither approach saves you from bad metric design or logging errors. The choice matters less than experiment hygiene.

### Advanced Technique 3: Sequential Testing — Fighting the Peeking Problem
Sequential testing (序贯检验) systematically accounts for repeated looks at the data:
- The valid stopping boundary is **wider early** and narrows over time
- You "pay a cost" for each peek — the threshold for significance is higher
- Result: you can look at results anytime and still maintain the correct false positive rate

This makes daily dashboards statistically valid. Without it, daily monitoring silently inflates your error rate.

---

## Part 5: Data Modeling for A/B Testing (DE Perspective)

As a data engineer, your job is to build the data infrastructure that makes experiments fast, reliable, and cheap to run.

### Core Tables

#### 1. Experiment Assignment Table
```sql
CREATE TABLE fct_experiment_assignments (
  user_id       BIGINT,
  experiment_id STRING,
  variant       STRING,        -- 'control', 'treatment_a', 'treatment_b'
  assigned_at   TIMESTAMP,
  ds            DATE
)
PARTITIONED BY (ds);
```
- One row per user per experiment
- Written at the moment of assignment (when Feature Gate fires)
- Critical: **log on exposure, not on page load** — only count users who actually saw the feature

#### 2. Experiment Metrics Table
```sql
CREATE TABLE fct_experiment_metrics (
  user_id       BIGINT,
  experiment_id STRING,
  variant       STRING,
  metric_name   STRING,
  metric_value  DOUBLE,
  ds            DATE
)
PARTITIONED BY (ds, experiment_id);
```

#### 3. Pre-Experiment Baseline Table (enables CUPED)
```sql
CREATE TABLE fct_user_pre_experiment_metrics (
  user_id          BIGINT,
  experiment_id    STRING,
  metric_name      STRING,
  pre_period_value DOUBLE,  -- same metric measured 7/14/30 days before experiment
  pre_period_days  INT
);
```
This is what enables CUPED — you need the pre-experiment value of the same metric. Build this table as part of experiment setup, not after.

### CUPED Implementation in SQL
```sql
-- Step 1: Calculate theta (regression coefficient)
WITH stats AS (
  SELECT
    experiment_id,
    metric_name,
    COVAR_POP(m.metric_value, p.pre_period_value) /
      VAR_POP(p.pre_period_value) AS theta
  FROM fct_experiment_metrics m
  JOIN fct_user_pre_experiment_metrics p
    ON m.user_id = p.user_id
    AND m.experiment_id = p.experiment_id
    AND m.metric_name = p.metric_name
  GROUP BY experiment_id, metric_name
),

-- Step 2: Compute CUPED-adjusted metric
adjusted AS (
  SELECT
    m.user_id,
    m.experiment_id,
    m.variant,
    m.metric_name,
    m.metric_value - s.theta * p.pre_period_value AS cuped_value
  FROM fct_experiment_metrics m
  JOIN fct_user_pre_experiment_metrics p
    ON m.user_id = p.user_id AND m.experiment_id = p.experiment_id
  JOIN stats s
    ON m.experiment_id = s.experiment_id AND m.metric_name = s.metric_name
)

-- Step 3: Compare variants on adjusted metric
SELECT
  variant,
  AVG(cuped_value) AS mean_adjusted,
  STDDEV(cuped_value) / SQRT(COUNT(*)) AS se
FROM adjusted
GROUP BY variant;
```

### How DE Speeds Up A/B Testing
| Technique | DE Action | Impact |
|---|---|---|
| **CUPED** | Build pre-experiment baseline table automatically at experiment start | 3–5x faster to significance |
| **Correct exposure logging** | Log on feature exposure, not page view | Eliminates dilution bias |
| **Guardrail automation** | Run guardrail metric checks in pipeline after each experiment window | Catches regressions without manual monitoring |
| **Metric pre-computation** | Pre-aggregate daily user-level metrics rather than computing from raw events | Faster experiment queries |
| **Experiment partitioning** | Partition `fct_experiment_metrics` by `experiment_id` | Avoids full table scans per experiment |
| **Sequential testing support** | Compute running statistics daily, not just at end of experiment | Enables valid daily peeking |

### Common DE Pitfalls in A/B Testing
- **Missing exposure logs** — if assignment is logged but exposure isn't, you include users who never saw the treatment → dilutes the effect
- **Clock skew on assignment** — if two services write assignments with different timestamps, joins break
- **Survivor bias in metrics** — only measuring users who converted, not all assigned users
- **Schema drift on event logs** — upstream event schema change mid-experiment corrupts metrics silently
- **Backfilling experiment data** — never backfill into experiment windows; it changes the population and invalidates results

---

## Interview Angles
- Why is A/B testing better than before/after analysis? → Before/after mixes the treatment effect with time-based confounders (seasonality, external events). Randomization removes all confounders.
- What is CUPED and why does it matter? → Variance reduction using pre-experiment data; same power with fewer users or faster results with same users.
- How do you prevent false positives from daily peeking? → Sequential testing — wider boundaries early that narrow over time, maintaining correct α throughout.
- What tables do you build to support experiments? → Assignment table, metrics table, pre-experiment baseline table for CUPED.
- What's the difference between assignment unit and analysis unit? → Assignment unit is what gets randomized (usually user_id); analysis unit is what gets measured. If they differ (e.g. assigned by user, measured by session), you need variance correction (delta method).

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

#### The Core Intuition
Users have very different spending or engagement histories before the experiment even starts. A high-spender will naturally spend more than a low-spender — that difference has nothing to do with your treatment. It's just noise that makes it harder to detect the real effect.

CUPED's insight: **"I already expect Alice to spend more than Bob based on history. So when measuring the treatment effect, I'll subtract out what I already expected and only look at what's surprising."**

The question CUPED answers is not "how much did each user spend?" but:
> **"Did the treatment cause a change BEYOND what history already predicted?"**

CUPED doesn't eliminate the historical difference between users — it removes its *influence* on the experiment result. The treatment effect estimate stays the same; what shrinks is the uncertainty around it.

#### The Math
```
Y_adjusted = Y - θ × X_pre

Where:
  Y         = metric during experiment (e.g. revenue this week)
  X_pre     = same metric before experiment (e.g. revenue last week)
  θ         = Cov(Y, X_pre) / Var(X_pre)   ← how strongly past predicts future
```

The variance reduction is:
```
Var(Y_adjusted) = Var(Y) × (1 - ρ²)

ρ = correlation between Y and X_pre
```
If ρ = 0.8 (strong correlation): variance drops to 36% of original → **~3x fewer users needed**.

#### Concrete Example (θ = 0.7)
| User | Pre-exp revenue | Experiment revenue | Expected (θ × pre) | Adjusted (surprise) |
|---|---|---|---|---|
| Alice | $100 | $110 | $70 | **$40** |
| Bob | $10 | $20 | $7 | **$13** |
| Carol | $50 | $55 | $35 | **$20** |

Alice's raw $110 looks wildly different from Bob's $20 — that spread is noise. After adjustment, the values are much closer together. The treatment effect is the same, but the variance is smaller → tighter confidence intervals → significance reached faster.

#### What CUPED Is NOT
- It doesn't change randomization or experiment design
- It doesn't fix bias — broken assignment means CUPED still fails
- It has no effect if past behavior doesn't predict future behavior (ρ ≈ 0)
- It's a post-processing step applied at analysis time, not during the experiment

Result: **same statistical power with ~5x fewer users**, or equivalently, reach significance much faster with the same user base. This is one of the highest-leverage techniques for teams that are sample-size constrained.

### Advanced Technique 2: Bayesian vs Frequentist

#### The Fundamental Difference in Mindset

**Frequentist asks:** "Assuming the treatment has zero effect, how likely is it that I'd see data this extreme by chance?" → Output: p-value

**Bayesian asks:** "Given the data I've seen, what's the probability that treatment is actually better than control?" → Output: a probability distribution

#### Frequentist in Plain English
Set up a null hypothesis (treatment = control), collect data, compute a p-value.
- p-value = 0.03 means: "If there were truly no effect, there's only a 3% chance of seeing results this extreme"
- If p < 0.05, reject the null and call it significant

**The common trap:** p-value does NOT mean "97% chance the treatment works." It's a statement about the data given the null — not about the null given the data.

#### Bayesian in Plain English
Start with a **prior belief** about the effect size (e.g. "most features move revenue by 0–5%"). As data comes in, update that belief to get a **posterior distribution**:

```
Posterior = Prior × Likelihood of data

"What I believe after seeing data" = "What I believed before" × "How well data fits"
```

Output: "There is an 89% probability that treatment is better than control." — a direct, intuitive answer that decision-makers actually understand.

#### The Peeking Problem — Where the Difference Really Matters
**Frequentist + peeking = inflated false positives.** If you check p-values daily and stop as soon as p < 0.05, your real false positive rate is far higher than 5%. The test was designed to be checked once at the end.

**Bayesian naturally handles peeking.** You're continuously updating a probability distribution, so you can check at any time. The posterior probability at day 3 is a valid statement — no fixed end date required.

#### Side-by-Side Comparison
| | Frequentist | Bayesian |
|---|---|---|
| Question answered | "Is this result unlikely under the null?" | "How probable is it that treatment wins?" |
| Output | p-value, confidence interval | Posterior probability, credible interval |
| Interpretation | Hard to explain to non-statisticians | Intuitive — direct probability |
| Peeking | Breaks the test | Naturally valid |
| Prior knowledge | Ignored | Incorporated — helpful or dangerous |
| Sample size | Must be fixed in advance | Flexible |
| Industry use | Default for guardrails, A/A tests | Multi-armed bandits, faster ship decisions |

#### When Each Wins
**Use Frequentist when:**
- You need a hard, defensible threshold ("only ship if p < 0.05")
- Running A/A tests to validate your experiment platform
- Checking guardrail metrics — you want a fixed false positive rate guarantee

**Use Bayesian when:**
- You want to stop early as soon as you have enough confidence
- Running multi-armed bandits — continuously reallocating traffic to the winning variant
- Communicating to non-technical stakeholders ("87% chance this works" vs "p = 0.04")
- You have strong, reliable prior knowledge to incorporate

**What you actually need to care about:** Neither approach saves you from bad metric design or logging errors. The choice matters less than experiment hygiene — correct exposure logging, stable metric definitions, and not changing decision criteria mid-experiment.

### Advanced Technique 3: Sequential Testing — Fighting the Peeking Problem

#### The Problem
In a standard Frequentist test, you fix your sample size upfront and check the result **once** at the end. But in reality, everyone peeks at the dashboard daily. Here's what that does to your false positive rate:

| Number of peeks | Actual false positive rate (intended 5%) |
|---|---|
| 1 | 5% |
| 5 | ~14% |
| 10 | ~19% |
| 20 | ~25% |

You set α = 0.05 but you're actually running at 25% false positives — shipping bad features 1 in 4 times.

#### Does Passive Peeking (No Decision) Still Cause Problems?
Yes — **peeking inflates false positives even if you make no decision and change nothing.**

The problem operates on two levels:

**Layer 1: Conscious bias** — You peek on day 5, see p = 0.06, decide to wait. You peek on day 8, see p = 0.04, and stop. You've run multiple tests and stopped at the favorable one — even though you told yourself you weren't deciding yet.

**Layer 2: The math doesn't care about your intentions** — Even if you genuinely wait until day 28, if you would have stopped early had you seen p < 0.05, you've implicitly changed your stopping rule. The test is no longer the fixed-horizon test you designed.

**The one case where peeking is truly harmless:** checking purely for operational issues — bugs, logging failures, catastrophic regressions — with a strict rule that you will only stop for those reasons, not for statistical significance. Running guardrail checks daily (is revenue crashing? are errors spiking?) while only looking at the primary metric once at the end is a valid pattern.

#### Do You Need Sequential Testing?
| Situation | Need sequential testing? |
|---|---|
| You peek and would stop early if p < 0.05 | Yes |
| You peek but only check for crashes/bugs | No — but be honest with yourself |
| You want to stop early when effect is clear | Yes |
| You truly wait until the planned end date | No — standard Frequentist is fine |

For most teams: **yes**. Daily dashboards make peeking structural, not just personal — the temptation to act on what you see is nearly impossible to resist.

#### How Sequential Testing Works
Sequential testing (序贯检验) systematically accounts for repeated looks at the data by **raising the bar for significance early and lowering it as more data accumulates**. Every peek "spends" some of your α budget — early peeks cost more because you have less data and more uncertainty.

```
High significance threshold (hard to reach)
│
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
│                                 ← boundary narrows over time
│         ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
│                  ─ ─ ─ ─ ─ ─ ─
│                          ─ ─ ─  ← converges to ~α=0.05 at end
└─────────────────────────────────────────────────
Day 1    Day 7    Day 14   Day 21   Day 28
```

- **Day 1:** You'd need an enormous effect to call it significant
- **Day 28:** The boundary has dropped to roughly the standard threshold
- This also protects against novelty effect — a feature that looks great on day 2 won't cross the boundary until the effect is stable

#### The Two Boundaries
Sequential testing is two-sided:
- **Upper boundary** — declare treatment a winner early
- **Lower boundary** — declare it a loser early (stop for futility)

Stopping for futility is just as important — if the effect is clearly zero or negative by day 10, there's no point running for 28 more days and wasting the experiment slot.

#### The Connection to CUPED
CUPED and sequential testing work well together:
- **CUPED** reduces variance → the experiment reaches the boundary faster
- **Sequential testing** lets you stop as soon as the boundary is crossed

Together they can cut experiment runtime from 4 weeks to 1 week.

This makes daily dashboards statistically valid. Without sequential testing, daily monitoring silently inflates your error rate.

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

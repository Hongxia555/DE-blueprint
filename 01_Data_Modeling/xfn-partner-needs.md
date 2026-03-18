# XFN Partner Needs → Data Model Design

Understanding what each cross-functional team does tells you what data they produce and what they need — which drives robust, purpose-built data models.

---

## Data Scientists (DS)

### What they do
- Run A/B experiments to validate product hypotheses
- Build ML models (ranking, recommendations, fraud, forecasting)
- Explore user behavior to generate product insights
- Define and monitor North Star / leading metrics/ guardrail metrics

### Data they produce
- Experiment configs and assignment tables (`exp_assignments`)
- Model scores and predictions (often written back to a feature store)
- Labeled datasets and ground-truth tables
- Metric definitions (sometimes owned in a metrics layer / dbt)

### Data they need from DE
| Need | Why | Data Model Implication |
|---|---|---|
| Grain-level fact tables | Training data, cohort analysis | Keep raw/atomic facts; don't pre-aggregate |
| Long history (2–3 yrs) | Seasonality, trend modeling | Partition by date; avoid aggressive TTL |
| Experiment join keys | Tie outcomes to assignments | `user_id` + `experiment_id` + `assignment_date` on fact tables |
| Stable column names | Pipelines break on schema drift | Enforce schema contracts; add deprecation notices |
| Feature tables (low latency) | Online inference | Separate OLAP (batch) vs. OLTP (real-time) feature serving |

---

## Software Development Engineers (SDE)

### What they do
- Build product features, APIs, and backend services
- Instrument client/server events (logging schema owners)
- Run infrastructure and define SLAs for data producers
- Conduct launch reviews that require metric readouts

### Data they produce
- Raw event logs (clicks, impressions, errors, API calls)
- System metrics (latency, error rates, availability)
- A/B test treatment assignments (via experimentation SDK)
- Entity state changes (user profile updates, order status transitions)

### Data they need from DE
| Need | Why | Data Model Implication |
|---|---|---|
| Feature adoption dashboards | Measure rollout success | Dim on `feature_flag` / `experiment_arm`; daily active metrics |
| Funnel / conversion metrics | Debug drop-off in new flows | Ordered session-level fact with step sequence |
| Error & latency rollups | Oncall / SLA monitoring | Separate ops fact table; avoid mixing with business facts |
| Self-serve query access | Engineers query logs directly | Expose clean staging layer; document grain clearly |
| Fast feedback post-launch | Ship → measure loop | Near-real-time aggregates (hourly or 15-min rollups) |

---

## Product Managers (PM)

### What they do
- Define product strategy and prioritize roadmap
- Own North Star metrics and OKRs
- Review experiment results and launch decisions
- Communicate health of the product to leadership

### Data they need from DE
| Need | Why | Data Model Implication |
|---|---|---|
| North Star metric (single number) | Weekly exec review | Reliable, documented, single source of truth |
| Cohort retention curves | Understand user stickiness | User-cohort grain fact (`cohort_week`, `week_number`, `retained`) |
| Experiment summary rollups | Ship/no-ship decisions | Pre-aggregated experiment results mart |
| Segment breakdowns | Surface which users drive growth | Flexible dim attributes on `dim_user` (platform, country, tier) |

---

## Data Analysts (DA) / Business Intelligence

### What they do
- Build dashboards and self-serve reports
- Answer ad hoc questions from PMs and leadership
- Define business metrics and KPIs

### Data they need from DE
| Need | Why | Data Model Implication |
|---|---|---|
| Wide, denormalized mart tables | Avoid complex joins in BI tools | Pre-join dims into `fct_` or `mart_` tables |
| Consistent naming conventions | Reuse across dashboards | `fct_`, `dim_`, `mart_` prefixes; snake_case |
| Metric definitions in code | Single source of truth | dbt metrics layer or semantic layer (e.g., MetricFlow) |
| Incremental freshness | Daily/hourly dashboard updates | Incremental dbt models; SLA on partition freshness |

---

## Finance & Marketing

### What they do
- Finance: revenue reconciliation, forecasting, unit economics
- Marketing: campaign attribution, CAC/LTV, channel performance

### Data they need from DE
| Need | Why | Data Model Implication |
|---|---|---|
| Revenue at transaction grain | Reconcile with billing systems | Immutable append-only fact; never update past rows |
| Attribution model inputs | Credit conversions to channels | `dim_attribution_channel`; support multi-touch |
| LTV / cohort spend | Payback period analysis | User-cohort fact joined to spend dim |
| Fiscal calendar support | Reporting periods ≠ calendar months | `dim_date` with `fiscal_week`, `fiscal_quarter` columns |

---

## Design Checklist — Before Building a Data Model

Use this when starting a new model to capture XFN needs upfront:

- [ ] Who are the consumers? (DS, SDE, PM, DA, Finance?)
- [ ] What grain do they need? (user-day, event, session, order?)
- [ ] What history depth is required?
- [ ] Are there experiment join keys needed?
- [ ] Real-time or batch? What freshness SLA?
- [ ] Will this feed ML training, dashboards, or both?
- [ ] What `dim_` attributes do they need to slice by?
- [ ] Are there compliance/PII constraints on any columns?

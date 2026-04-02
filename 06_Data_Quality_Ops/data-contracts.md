# Data Contracts
**Source:** DataExpert.io Boot Camp — Video 14 (Week 3, Analytics Track)
**Topic:** Data contract design, schema agreements, SLA enforcement between producers and consumers

---

## Key Concepts

### What is a Data Contract?
A formal agreement between a data producer and consumer that specifies:
- **Schema** — field names, types, nullability
- **Semantics** — what each field means
- **SLA** — when data is available and freshness guarantees
- **Quality** — acceptable null rates, value ranges, uniqueness constraints
- **Ownership** — who is responsible for breaking changes

### Why Data Contracts?
- Without contracts: schema changes silently break downstream pipelines
- With contracts: breaking changes require negotiation and versioning
- Enables self-service: consumers can trust data without checking with producers daily

### Contract Structure
```yaml
# data-contract.yaml
name: fct_user_events
version: "1.2.0"
owner: data-platform-team
consumers:
  - growth-analytics
  - ml-features-team

schema:
  - name: user_id
    type: BIGINT
    nullable: false
    description: "Unique user identifier"
  - name: event_type
    type: STRING
    nullable: false
    values: ["click", "impression", "conversion"]
  - name: ds
    type: DATE
    nullable: false
    description: "Event date (partition key)"

sla:
  availability: "08:00 UTC daily"
  freshness: "< 2 hours from event time"
  completeness: "> 99.5% of events"

quality_checks:
  - name: no_null_user_ids
    sql: "SELECT COUNT(*) FROM fct_user_events WHERE user_id IS NULL AND ds = '{{ ds }}'"
    threshold: 0
  - name: row_count_anomaly
    description: "Row count within 20% of 7-day average"
```

### Contract Versioning
| Change Type | Action Required |
|-------------|----------------|
| Add nullable column | Minor version bump, backward compatible |
| Rename column | Major version bump, coordinate with consumers |
| Remove column | Major version bump, deprecation period required |
| Change type (widening, e.g. INT→BIGINT) | Minor version bump |
| Change type (narrowing) | Major version bump |

### Enforcement Patterns
- **Pre-commit hooks** — validate contract YAML schema before merge
- **Pipeline validation** — run quality checks from contract at pipeline start
- **Schema registry** — central store for all contracts (e.g., Confluent Schema Registry for Kafka topics)

---

## Interview Angles
- How do you prevent a schema change from breaking 10 downstream pipelines? → Data contracts with versioning + deprecation policy; breaking changes require consumer sign-off
- How do you enforce SLAs on data delivery? → Monitor freshness metrics; alert on-call if data not available by SLA time; track SLA breach rate as a team KPI
- What's the difference between a data contract and documentation? → Contracts are machine-enforceable agreements with versioning; documentation is human-readable and non-binding

---

## Notes (from transcript)

### Video 14 is actually about WAP + data quality (not contracts per se)
- The word "data contract" in this context = "writing to production is a contract"
- A contract has 3 components: Schema + Quality checks + How data appears in production

### WAP pattern details (Video 14 expands on Video 13)
- Netflix and Airbnb use WAP
- Facebook uses the **Signal Table pattern**
- Both achieve the same goal: consumers never see partial/bad data

### Schema contracts in data lake vs RDBMS
- In PostgreSQL: you can enforce `NOT NULL`, `UNIQUE`, foreign key constraints — strong contracts
- In data lake (S3/HDFS): **no constraints** — all of this goes out the window
- "In the data lake you do have to worry about it a little bit more"
- This is why WAP and explicit quality checks are essential in lake-based architectures

### Third-party API data contracts
- "They can change their contract just the same" — and you have no control
- Especially risky with Salesforce, Slack, Stripe, Discord integrations
- "You can't just go and talk to those software engineers" — they may not respond even if you could
- Extra care needed for third-party ingestion pipelines

### Validation best practices (from Airbnb breakout sessions)
Common themes Zach heard from engineers:
- Not enough validation rules was the most cited cause of bad data reaching production
- Process: backfill 1 month → validate all assumptions → document in a "data look" doc → backfill rest
- Rule: **always have someone else check your assumptions** — you cannot validate your own pipeline effectively

### Blocking vs non-blocking checks (decision framework)
```
Quality check fails
    ↓
Is this a blocking check?
    YES → Stop pipeline, page on-call, investigate root cause, manually decide to publish or not
    NO  → Fire alert (Slack/PagerDuty), but continue publishing — treat as data weirdness, not a bug
```

# Kafka + Flink: Real-time Pipeline Architecture
**Source:** DataExpert.io Boot Camp — Video 15 (3-hour course, Week 4 Infrastructure Track)
**Topic:** Kafka fundamentals, Apache Flink, streaming vs. batch, stateful processing

---

## Key Concepts

### Kafka Architecture
| Component | Role |
|-----------|------|
| Producer | Writes events to topics |
| Topic | Ordered, immutable log of events |
| Partition | Unit of parallelism within a topic |
| Consumer Group | Set of consumers that collectively read a topic |
| Broker | Server that stores and serves partitions |
| Offset | Position of a message within a partition |

```
Producer → [Topic: user_events] → Consumer Group (Flink job)
           [Partition 0]
           [Partition 1]
           [Partition 2]
```

### Kafka Key Decisions
- **Partition count** → determines max consumer parallelism (more partitions = more parallel consumers)
- **Retention** → how long messages are kept (default 7 days); for replay capability, set longer
- **Key** → messages with the same key go to the same partition (preserves order per entity)

### Apache Flink Concepts
| Concept | Description |
|---------|-------------|
| DataStream API | Low-level stream processing API |
| Table API / SQL | High-level declarative processing |
| Watermarks | Handle late-arriving events; advance event time |
| Windows | Group events: Tumbling, Sliding, Session |
| State | Maintained per-key across events (e.g., running totals) |
| Checkpointing | Periodic state snapshots for fault tolerance |

### Window Types
```
Tumbling Window (non-overlapping):  [0-60s] [60-120s] [120-180s]
Sliding Window (overlapping):       [0-60s] [30-90s] [60-120s]
Session Window (activity-based):    [burst of events] gap [burst of events]
```

### Streaming vs. Batch
| Dimension | Batch | Streaming |
|-----------|-------|-----------|
| Latency | Minutes–hours | Milliseconds–seconds |
| Complexity | Low | High (state, watermarks, exactly-once) |
| Cost | Lower | Higher |
| Use case | Daily reports, ML training | Fraud detection, live dashboards |

### Exactly-Once Semantics
- **At-most-once** — may lose data
- **At-least-once** — may duplicate data (easiest to achieve)
- **Exactly-once** — hardest; requires idempotent sinks + two-phase commit

---

## Interview Angles
- When would you choose Flink over Spark Streaming? → Flink is true streaming (event-by-event); Spark Streaming is micro-batch. Use Flink for sub-second latency requirements.
- How does Kafka guarantee message ordering? → Only within a single partition, not across partitions
- What happens when a Flink job fails? → Restores from last checkpoint; processes from saved offset in Kafka

---

## Notes (from transcript)

### Streaming vocabulary — get the definitions right first
When a stakeholder says "real time" they almost **never** mean Flink:
- **Streaming / Continuous processing** (Flink) → every event processed immediately as it arrives
- **Near real time / Micro batch** (Spark Structured Streaming) → collect N minutes of events, then process
- **Intraday** → runs more than once per day (could be either)
- "Your stakeholders are not going to know the difference — if they say 'real time' that might not mean use Flink"
- In Zach's career: stakeholder said "real time" and actually meant streaming less than 5% of the time

### Spark Structured Streaming vs Apache Flink
- Spark Streaming is **micro-batch** (not true continuous processing, though Databricks is working on it)
- Flink is **true continuous processing** — processes event-by-event
- "When Databricks announces Spark Streaming as continuous, Flink and Spark Streaming will essentially converge"
- For now: use Flink when you need sub-second/second latency; Spark Streaming for minute-level latency

### Windowing functions in Flink (covered in lab)
- **Count windows** — process every N events
- **Sliding windows** — overlapping time windows (e.g., rolling 5-minute window every 1 minute)
- **Session windows** — gap-based (window closes after N seconds of inactivity per key)
- Lab: connect to a real Kafka cluster in the cloud and process click stream data

### When NOT to use Flink/Streaming
- "If you're the only one on your team that knows Flink then you probably shouldn't use Flink"
- Streaming adds significant complexity: state management, watermarks, exactly-once semantics
- Most business problems that feel like they need streaming can be solved with **5-15 minute micro-batch**
- Use streaming only when the business actually loses value with >5 minute latency (fraud detection, live dashboards)

### Lab setup requirement
- **Docker is required** for Flink and Kafka labs
- Local setup is insufficient for distributed systems like Flink
- Connects to DataExpert.io's cloud Kafka cluster for real click stream data

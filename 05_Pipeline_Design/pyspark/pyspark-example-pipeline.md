# PySpark Example — StreamApp Member Sessions Pipeline

**Related:** [[pyspark-overview]] · [[dbt-and-testing]]

---

## Scenario

Raw member session events arrive as JSON files in S3. Goal: build a clean daily engagement mart following Medallion architecture.

```
S3 raw JSON files  (source)
        ↓
   Bronze layer    — raw events ingested as-is, schema enforced
        ↓
   Silver layer    — sessionized, cleaned, typed
        ↓
   Gold layer      — daily member engagement mart for BI / ML
```

---

## Project Structure

```
streamapp_pipeline/
  ingestion/
    ingest_events.py         ← reads raw JSON, validates with pydantic, writes Bronze
  pipeline/
    bronze_to_silver.py      ← sessionizes events, writes Silver Delta table
    silver_to_gold.py        ← aggregates into daily mart, writes Gold Delta table
  models/
    schemas.py               ← pydantic models for validation
  tests/
    test_sessionize.py       ← pytest unit tests for transformation logic
    test_schemas.py          ← pytest unit tests for pydantic validation
```

---

## Step 1 — Schema Definition (models/schemas.py)

Define expected input shape with pydantic. Validation happens before any data touches the pipeline.

```python
from pydantic import BaseModel, field_validator  # BaseModel: auto type validation; field_validator: custom rules
from datetime import datetime                    # Pydantic auto-parses ISO strings → datetime

class MemberEvent(BaseModel):                   # inheriting BaseModel gives free validation, .model_dump(), .model_json_schema()
    member_id: str                              # required — missing or wrong type raises ValidationError
    event_type: str                             # required — further validated by @field_validator below
    event_ts: datetime                          # required — "2026-06-04T10:00:00" string auto-converted to datetime
    session_id: str | None = None               # optional — str or null; default None means can be omitted entirely
    content_id: str | None = None               # optional — present for content_play events only
    duration_secs: int | None = None            # optional — present for session_end / content_play events

    @field_validator("event_type")              # runs AFTER built-in type check passes, only for this field
    @classmethod                                # required by Pydantic — validates before instance exists, so no self
    def validate_event_type(cls, v):            # v = the field's value after type coercion
        allowed = {"session_start", "session_end", "content_play", "mood_log"}  # set → O(1) lookup
        if v not in allowed:
            raise ValueError(f"Unknown event_type: {v}")  # Pydantic catches ValueError → wraps into ValidationError
        return v                                # return v unchanged; could return v.lower() to normalize casing
```

---

## Step 2 — Ingestion: Raw JSON → Bronze (ingestion/ingest_events.py)

Read raw JSON files line by line, validate each record, route failures to a dead letter queue, write valid records to the Bronze Delta table.

```python
import json                                         # parse JSON strings → Python dicts
from pathlib import Path                            # not used here but useful for cross-platform file paths
from datetime import date                           # run_date type — one date per pipeline run
from pydantic import ValidationError                # the exception Pydantic raises when a record fails validation
from models.schemas import MemberEvent              # our schema class from Step 1
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, TimestampType, IntegerType  # explicit schema defs

spark = SparkSession.builder.appName("streamapp_ingestion").getOrCreate()  # get existing session or create one

# Generator — reads large files without loading everything into memory
def read_json_lines(filepath: str):
    with open(filepath) as f:
        for line in f:
            yield json.loads(line)              # yield: one dict at a time → never loads full file into RAM

def ingest(source_path: str, run_date: date):
    valid_records = []                          # accumulates dicts ready for Spark DataFrame
    dead_letter = []                            # accumulates failed records for investigation

    for raw in read_json_lines(source_path):    # raw = one Python dict per JSON line
        try:
            event = MemberEvent(**raw)          # **raw unpacks dict as kwargs → Pydantic validates all fields
            valid_records.append(event.model_dump())  # .model_dump() converts Pydantic model back to plain dict
        except ValidationError as e:
            dead_letter.append({"record": raw, "errors": str(e)})  # preserve original record + error message

    print(f"Ingested: {len(valid_records)} valid, {len(dead_letter)} failed")

    if dead_letter:
        dlq_df = spark.createDataFrame(dead_letter)         # wrap failed records as Spark DataFrame
        dlq_df.write.format("delta").mode("append") \
            .save(f"s3://bronze/dlq/member_events/date={run_date}/")  # DLQ partitioned by run date for easy lookup

    if valid_records:
        df = spark.createDataFrame(valid_records)           # distribute valid records across Spark executors
        # Bronze: reject unexpected schema changes — mergeSchema=false
        df.write.format("delta") \
            .option("mergeSchema", "false") \              # fail loudly if new columns appear — catches upstream changes
            .mode("append") \                              # append to existing table, never overwrite
            .partitionBy("event_date") \                   # Hive-style folder partitions → prune reads by date downstream
            .save("s3://bronze/member_events/")
```

---

## Step 3 — Bronze → Silver: Sessionize (pipeline/bronze_to_silver.py)

Read Bronze, clean and type-cast columns, assign session IDs using window functions, write Silver.

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F              # all built-in Spark functions: F.col, F.lag, F.sum, etc.
from pyspark.sql.window import Window               # defines the window frame for window functions
from datetime import date

spark = SparkSession.builder.appName("streamapp_bronze_to_silver").getOrCreate()

def bronze_to_silver(run_date: date):
    # Read only today's partition — partition pruning keeps this fast
    bronze = spark.read.format("delta").load("s3://bronze/member_events/") \
        .filter(F.col("event_date") == str(run_date))   # pushes filter into Delta log → scans only today's files

    # Sessionize: new session when gap between events > 30 minutes
    w = Window.partitionBy("member_id").orderBy("event_ts")          # per-member, ordered by time
    w_running = Window.partitionBy("member_id").orderBy("event_ts") \
                      .rowsBetween(Window.unboundedPreceding, Window.currentRow)  # cumulative sum frame: all rows up to current

    sessionized = (
        bronze
        .withColumn("prev_ts", F.lag("event_ts").over(w))             # prev_ts = timestamp of the previous event for this member
        .withColumn("gap_mins",
            (F.col("event_ts").cast("long") - F.col("prev_ts").cast("long")) / 60  # cast to Unix seconds, diff → minutes
        )
        .withColumn("is_new_session",
            F.when(F.col("gap_mins").isNull() | (F.col("gap_mins") > 30), 1).otherwise(0)
            # isNull() catches the first event (no prev_ts); >30 catches long gaps → both start a new session
        )
        .withColumn("session_id",
            F.concat(
                F.col("member_id"),
                F.lit("_"),                                            # F.lit: insert a literal string constant
                F.sum("is_new_session").over(w_running).cast("string") # cumulative count of session boundaries → session number
            )
        )
        .drop("prev_ts", "gap_mins", "is_new_session")                 # remove intermediate columns before writing
    )

    # Silver: allow additive schema evolution — mergeSchema=true
    sessionized.write.format("delta") \
        .option("mergeSchema", "true") \   # new columns added upstream won't break this write
        .mode("append") \
        .partitionBy("event_date") \
        .save("s3://silver/member_sessions/")

    print(f"Silver written: {sessionized.count()} rows for {run_date}")
```

---

## Step 4 — Silver → Gold: Daily Engagement Mart (pipeline/silver_to_gold.py)

Aggregate sessionized events into one row per member per day for BI and ML consumption.

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from datetime import date

spark = SparkSession.builder.appName("streamapp_silver_to_gold").getOrCreate()

def silver_to_gold(run_date: date):
    silver = spark.read.format("delta").load("s3://silver/member_sessions/") \
        .filter(F.col("event_date") == str(run_date))   # read only today's partition

    # One row per member per day
    daily_engagement = (
        silver
        .groupBy("member_id", "event_date")             # collapse all events → one row per member per day
        .agg(
            F.countDistinct("session_id").alias("session_count"),       # unique sessions in the day
            F.sum(F.when(F.col("event_type") == "content_play", 1).otherwise(0))
             .alias("content_plays"),                   # conditional sum: count rows where event_type matches
            F.sum(F.when(F.col("event_type") == "mood_log", 1).otherwise(0))
             .alias("mood_logs"),
            F.sum("duration_secs").alias("total_duration_secs"),        # total time in app (seconds)
            F.max("event_ts").alias("last_active_ts")                   # latest event = when member was last seen
        )
        .withColumn("total_duration_mins", F.round(F.col("total_duration_secs") / 60, 2))  # derived convenience column
    )

    daily_engagement.write.format("delta") \
        .mode("overwrite") \                                            # overwrite, but only today's partition (not full table)
        .option("replaceWhere", f"event_date = '{run_date}'") \         # replaceWhere: safe idempotent daily overwrite — reruns don't duplicate
        .save("s3://gold/mart_member_engagement_daily/")

    print(f"Gold written: {daily_engagement.count()} member-days for {run_date}")
```

---

## Step 5 — Testing with pytest (tests/test_sessionize.py)

Test the transformation logic in pure Python — no Spark needed for unit tests. Keep tests small and deterministic.

```python
from datetime import datetime, timedelta

# Pure Python function — no Spark dependency, so pytest can run it without a cluster
def assign_sessions(events: list[dict], gap_minutes: int = 30) -> list[dict]:
    if not events:
        return []
    result = []
    session_num = 0
    prev_ts = None
    for event in sorted(events, key=lambda e: e["event_ts"]):  # sort ascending by time before processing
        if prev_ts is None:                                     # first event always starts session 1
            session_num = 1
        else:
            gap = (event["event_ts"] - prev_ts).total_seconds() / 60  # timedelta → seconds → minutes
            if gap > gap_minutes:
                session_num += 1                                # gap too large → increment session counter
        result.append({**event, "session_id": f"{event['member_id']}_{session_num}"})  # **event spreads all fields, adds session_id
        prev_ts = event["event_ts"]                             # advance the pointer for next iteration
    return result


# --- Tests ---

def make_event(member_id: str, minutes_offset: int, event_type: str = "session_start") -> dict:
    return {
        "member_id": member_id,
        "event_ts": datetime(2026, 6, 1, 10, 0) + timedelta(minutes=minutes_offset),  # fixed base time + offset → deterministic timestamps
        "event_type": event_type,
    }

def test_single_event_gets_session_1():
    events = [make_event("m_001", 0)]
    result = assign_sessions(events)
    assert result[0]["session_id"] == "m_001_1"         # first event → session 1

def test_events_within_30_min_same_session():
    events = [make_event("m_001", 0), make_event("m_001", 15), make_event("m_001", 29)]
    result = assign_sessions(events)
    session_ids = {r["session_id"] for r in result}     # set comprehension — deduplicates session IDs
    assert session_ids == {"m_001_1"}                   # all three events in one session

def test_gap_over_30_min_starts_new_session():
    events = [make_event("m_001", 0), make_event("m_001", 31)]
    result = assign_sessions(events)
    assert result[0]["session_id"] == "m_001_1"
    assert result[1]["session_id"] == "m_001_2"         # 31-min gap → new session

def test_empty_input_returns_empty():
    assert assign_sessions([]) == []                    # edge case: empty list must not crash

def test_multiple_members_independent_sessions():
    events = [
        make_event("m_001", 0),
        make_event("m_002", 0),   # different member — session count is independent
        make_event("m_001", 60),  # m_001: gap > 30 → new session
    ]
    result = assign_sessions(events)
    m001 = [r for r in result if r["member_id"] == "m_001"]  # filter to just m_001 rows
    assert m001[0]["session_id"] == "m_001_1"
    assert m001[1]["session_id"] == "m_001_2"
```

---

## Step 6 — Testing Schema Validation (tests/test_schemas.py)

```python
import pytest
from datetime import datetime
from pydantic import ValidationError               # the exception raised when any field fails validation
from models.schemas import MemberEvent

def test_valid_event():
    event = MemberEvent(
        member_id="m_001",
        event_type="session_start",
        event_ts=datetime(2026, 6, 1, 10, 0),     # passing a datetime object directly (no parsing needed)
    )
    assert event.member_id == "m_001"

def test_missing_member_id_raises():
    with pytest.raises(ValidationError):           # pytest.raises: asserts that this block throws the exception
        MemberEvent(event_type="session_start", event_ts=datetime(2026, 6, 1, 10, 0))  # member_id omitted → required field missing

def test_invalid_event_type_raises():
    with pytest.raises(ValidationError):
        MemberEvent(member_id="m_001", event_type="UNKNOWN", event_ts=datetime(2026, 6, 1))  # "UNKNOWN" not in allowed set

def test_bad_timestamp_raises():
    with pytest.raises(ValidationError):
        MemberEvent(member_id="m_001", event_type="session_start", event_ts="not-a-date")  # unparseable string → datetime coercion fails

def test_optional_fields_default_to_none():
    event = MemberEvent(
        member_id="m_001",
        event_type="mood_log",
        event_ts=datetime(2026, 6, 1, 10, 0),     # only required fields provided
    )
    assert event.content_id is None                # optional fields default to None when not passed
    assert event.duration_secs is None
```

---

## End-to-End Flow Summary

```
Raw JSON on S3
      │
      ▼
ingest_events.py          ← pydantic validates each record
  ├── valid records   →   Bronze Delta table (partitioned by event_date)
  └── invalid records →   DLQ Delta table (for investigation)
      │
      ▼
bronze_to_silver.py       ← PySpark window functions assign session IDs
      │                      mergeSchema=false at Bronze, true at Silver
      ▼
silver_to_gold.py         ← PySpark groupBy aggregates to daily mart
      │
      ▼
mart_member_engagement_daily  ← one row per member per day
      │
      ▼
dbt (semantic layer)      ← business metrics, lineage, AI agent contracts
```

---

## dbt vs PySpark in This Pipeline

| Step | Tool | Why |
|---|---|---|
| Schema validation | pydantic | Catches bad data before it enters the pipeline |
| Bronze ingestion | PySpark | Reads raw files, partitions, writes Delta |
| Sessionization | PySpark | Window functions across distributed data |
| Daily aggregation | PySpark | GroupBy at scale |
| Business metrics | dbt | SQL transformations, `active_member` definitions, lineage |
| Semantic layer | dbt | Single source of truth for BI and AI agents |
| Unit tests | pytest | Tests pure Python logic without a Spark cluster |
| Pipeline tests | dbt test | Schema tests, not-null, accepted values on final marts |

---

*See also: [[pyspark-overview]] for concepts · [[dbt-example-ecommerce]] for the dbt equivalent*

# Testing Apache Spark Jobs
**Source:** DataExpert.io Boot Camp — Video 12 (Week 3, Infrastructure Track)
**Topic:** PySpark unit testing, CI/CD for data pipelines, `chisa` framework for fake data

---

## Key Concepts

### Why Test Spark Jobs?
- Transformation logic bugs are silent — wrong results, no errors
- Schema drift breaks downstream consumers
- Tests enable confident refactoring and safe CI/CD deploys

### Testing Pyramid for DE
```
         [Integration Tests]     ← slow, test full pipeline end-to-end
        [Component Tests]        ← test one transformation with real Spark
      [Unit Tests]               ← test pure functions without Spark (fastest)
```

### Unit Testing Pattern (PySpark)
```python
# conftest.py — shared Spark session
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder \
        .master("local[2]") \
        .appName("test") \
        .getOrCreate()

# test_transformations.py
def test_growth_accounting(spark):
    # Arrange — create minimal test DataFrames
    today_data = [(1, "2024-01-02"), (2, "2024-01-02")]
    yesterday_data = [(1, "2024-01-01")]  # user 2 is new

    today_df = spark.createDataFrame(today_data, ["user_id", "ds"])
    yesterday_df = spark.createDataFrame(yesterday_data, ["user_id", "ds"])

    # Act
    result = classify_user_states(today_df, yesterday_df, date="2024-01-02")

    # Assert
    states = {row.user_id: row.user_state for row in result.collect()}
    assert states[1] == "retained"
    assert states[2] == "new"
```

### `chisa` Framework (Fake Data Generation)
- Generates realistic fake DataFrames for testing without real data
- Supports schema-based generation with configurable distributions
- Used to test pipelines with realistic cardinality and skew patterns

### CI/CD Integration
```yaml
# GitHub Actions example
- name: Run Spark Tests
  run: |
    pip install pyspark pytest
    pytest tests/ -v --tb=short

- name: Check Schema Compatibility
  run: python scripts/validate_schema.py
```

### Schema Drift Detection
```python
def assert_schema(df, expected_schema):
    """Fail loudly if schema changes unexpectedly"""
    assert df.schema == expected_schema, \
        f"Schema mismatch:\nExpected: {expected_schema}\nActual: {df.schema}"
```

---

## Interview Angles
- How do you test a Spark job that reads from S3? → Mock the read with `spark.createDataFrame()`; test only transformation logic in unit tests
- How do you catch schema drift before it breaks production? → Schema validation step in CI; compare against registered schema in a catalog
- What's the difference between unit tests and integration tests for pipelines? → Unit: test transformation function with synthetic data, no I/O. Integration: test full job against real dev environment data.

---

## Notes (from transcript)

### Three places a quality bug can be caught (ranked best to worst)
1. **In development** — bug fixed before deploy, never surfaces to anyone. "Almost like the bug never existed." Best case.
2. **WAP audit fails in production staging** — data not published, pipeline delays. Better than #3 but onerous.
3. **Bug reaches production tables** — data analyst notices anomaly. Can go **unnoticed for weeks**. Worst case.

### Unit tests vs integration tests for pipelines
- **Unit test**: test individual functions/UDFs without Spark I/O. Fastest, catches logic errors.
- **Integration test**: run full pipeline against real dev environment data. Slower, catches system errors.
- Zach's recommendation: unit test all UDFs especially if they call external libraries (pricing libs, etc.)

### False positive quality checks
- "I hate false positive data quality checks they are common"
- They create alert fatigue and waste engineering time
- If a check fails too often without real issues → recalibrate the threshold, don't disable the check

### Real incident: Ethiopia internet shutdown
- Facebook growth pipeline had a blocking quality check fail
- Reason: Ethiopian government **turned off the internet** → near-zero activity from Ethiopia
- Not a quality error — actual real-world event
- Lesson: when a blocking check fails, investigate the world before assuming pipeline bug

### chisa framework (for fake data in tests)
- Generates realistic fake DataFrames for testing
- Supports schema-based generation with configurable distributions
- Used to test pipelines with realistic cardinality and skew patterns without needing real prod data

### Zach's key testing principle
- "Catch the errors in development before you deploy to production and it causes a lot fewer headaches for you to deal with later on"
- Build your pipelines with a **software engineering mindset** — tests are not optional

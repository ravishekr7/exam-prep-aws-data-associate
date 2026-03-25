# Data Quality

Ensuring data meets expectations for completeness, accuracy, and consistency.

---

## AWS Glue Data Quality

### DQDL (Data Quality Definition Language)
Simple rule syntax:
```
Rules = [
  "IsComplete \"col_a\"",
  "ColumnValues \"age\" > 21",
  "ColumnValues \"email\" matches \".*@.*\\..*\"",
  "RowCount > 100",
  "Uniqueness \"id\" > 0.95"
]
```

### Actions on Failure
- Fail the Glue job
- Write results to CloudWatch
- Route bad records to a separate S3 bucket
- Continue processing (log only)

### Integration
- Embedded in Glue ETL jobs
- Can run as standalone evaluation
- Results visible in Glue console

---

## Glue Data Quality DQDL Deep Dive

DQDL (Data Quality Definition Language) is Glue's declarative rule language. Rules are evaluated per-row or per-dataset and produce PASS/FAIL results with scores.

### Core Rule Types with Syntax

```
Rules = [

  # Completeness: no nulls allowed in this column
  IsComplete "customer_email",

  # Uniqueness: at least 99% of values must be unique
  Uniqueness "order_id" > 0.99,

  # Range check: all values must be between 0 and 120
  ColumnValues "age" between 0 and 120,

  # Minimum row count: dataset must have at least 1000 rows
  RowCount > 1000,

  # Maximum row count (combine with minimum for expected range)
  RowCount between 10000 and 10000000,

  # Custom SQL: count of rows failing a condition must equal 0
  CustomSQL "select count(*) from primary where customer_id is null" = 0,

  # Primary key check: combines IsComplete + Uniqueness in one rule
  IsPrimaryKey "transaction_id",

  # Referential integrity: 99%+ of FK values must exist in the reference table
  ReferentialIntegrity "orders.customer_id" "customers.customer_id" > 0.99,

  # Pattern matching: values must match a regex pattern
  ColumnValues "phone_number" matches "\\+?[0-9]{10,15}",

  # Data type check: column values must be parseable as a date
  ColumnValues "event_date" matches "\\d{4}-\\d{2}-\\d{2}"
]
```

### Ruleset Reuse
Define a ruleset once in the Glue console or as a named resource, then reference it from multiple Glue jobs:
```python
# Reference a named ruleset in a Glue job
response = glue_client.start_data_quality_ruleset_evaluation_run(
    DataSource={
        'GlueTable': {'DatabaseName': 'my_db', 'TableName': 'orders'}
    },
    RulesetNames=['standard_order_rules', 'pii_completeness_rules']
)
```

### Composite Rulesets
Combine rules with logical operators to create nuanced conditions. Glue Data Quality evaluates each rule independently and reports individual scores. You can create composite logic in `CustomSQL` rules that combine multiple conditions:
```
CustomSQL "select count(*) from primary where status = 'ACTIVE' and last_login_date is null" = 0
```

---

## Glue DQ Actions on Failure

When a ruleset evaluation runs as part of a Glue ETL job, you can configure what happens when rules fail:

### FAIL
**Effect:** The Glue job fails immediately. No output data is written to the destination.

**When to use:** Critical data quality gates before loading to a data warehouse. If the data doesn't meet quality standards, it's better to halt the pipeline than to load bad data.

```python
# Job fails if any rule in the ruleset fails
job.commit()  # Only reached if all DQ rules pass
```

### QUARANTINE
**Effect:** Rows failing the quality rules are written to a **separate S3 path** (quarantine bucket). Rows passing all rules continue through the pipeline to the destination. The job does NOT fail.

**Most common in exam scenarios** — allows good data to flow through while bad data is isolated for investigation and reprocessing.

```
Pipeline: S3 source → Glue ETL + DQ check
                         ↓ PASS         ↓ FAIL
                    S3 destination   S3 quarantine bucket
```

### LOG
**Effect:** Quality results are written to CloudWatch Logs. The job continues processing regardless of pass/fail. No data routing occurs.

**When to use:** Monitoring and observability mode. You want to track data quality trends over time without blocking the pipeline.

### Recommended Production Pattern
```
QUARANTINE + CloudWatch Alarm + SNS Notification:
1. Route bad rows to s3://quarantine-bucket/failed/
2. Glue publishes DQ metrics to CloudWatch
3. CloudWatch Alarm: if DQ score < 0.95, trigger alarm
4. SNS sends notification to data engineering team
5. Team reviews quarantine bucket and decides to reprocess or discard
```

---

## AWS Glue DataBrew

- No-code visual data preparation
- 250+ built-in transformations
- Data profiling: statistics, distributions, missing values
- Define data quality rules visually
- Good for analysts who don't write code

---

## Data Quality Patterns

### Validation During Ingestion
1. Lambda on S3 event -> validate header/format -> route to clean/quarantine bucket
2. Firehose -> Lambda transform -> validate and tag records

### Validation During Processing
1. Glue job with embedded DQDL rules
2. Step Functions with quality check step before loading to warehouse

### Data Consistency Checks
- Cross-dataset record count validation
- Referential integrity checks between tables
- Freshness checks (data is not stale)

### Null Check Pattern
Apply `IsComplete` on all required fields before writing to data warehouse. Fields like `customer_id`, `order_date`, and `amount` should never be null in a fact table.
```
Rules = [
  IsComplete "customer_id",
  IsComplete "order_date",
  IsComplete "amount",
  IsComplete "product_id"
]
```
Action: FAIL — if any required field has nulls, stop the load entirely.

### Freshness Check
Ensure the data is not stale (pipeline didn't silently stop producing data):
```
Rules = [
  # All event_date values should be recent (within last 2 days)
  ColumnValues "event_date" > (current_date - 2)
]
```

### Volume Check
Catch pipeline failures that produce empty or near-empty output:
```
Rules = [
  # Normal daily load is between 10K and 1M rows
  RowCount between 10000 and 1000000
]
```
If RowCount = 0, the upstream pipeline likely failed silently. FAIL action prevents an empty load from overwriting yesterday's good data.

### Referential Integrity Check
Before loading a fact table to Redshift, verify that all foreign key values exist in dimension tables:
```
Rules = [
  # 100% of orders must reference valid customers
  ReferentialIntegrity "orders.customer_id" "customers.customer_id" = 1.0,

  # 99%+ of orders must reference valid products (allow for new products)
  ReferentialIntegrity "orders.product_id" "products.product_id" > 0.99
]
```

---

## Data Sampling Techniques

- **Random Sampling:** Uniform random selection of rows
- **Stratified Sampling:** Sample proportionally from each group/category
- **Systematic Sampling:** Every Nth record
- Use for: testing quality rules on large datasets without scanning everything

---

## Data Skew

- One partition has disproportionately more data than others
- Causes: poor partition key choice (e.g., `country` where 90% is US)
- **Fixes:**
  - Salt the key (add random suffix)
  - Choose better partition key
  - Use `groupFiles` in Glue
  - Repartition in Spark

---

## Exam Gotchas

- **"Data quality rules"** = Glue Data Quality with DQDL
- **"No-code data preparation"** = Glue DataBrew
- **"Data skew"** = uneven partition sizes, fix with salting or repartition
- Data quality can be a step in Step Functions workflow (check before proceeding)
- DataBrew also supports data profiling (statistics about your data)

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
